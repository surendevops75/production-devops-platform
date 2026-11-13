# Chapter 189 — NetworkPolicy Templates

Phase 20 — Platform Engineering

## Part 1 — Production Network Security Architecture and Implementation Foundation

Expansion Part 1 of 8

This part establishes the implementation foundation for the complete NetworkPolicy platform. We will build on the earlier chapter rather than repeat its basic YAML examples.

The target is a reusable platform capability that can be applied consistently across all 21 microservices.

## 189.29. Production network security objectives

In a production e-commerce platform, network security must protect communication between workloads without breaking legitimate application functionality.

Consider a compromised Cart Service.

Without network restrictions, it might attempt to communicate with:

- Payment Service.
- User Service.
- Reporting Service.
- Database endpoints.
- Messaging brokers.
- Other workloads in the cluster.

But Cart Service should only access its approved dependencies.

Our objective is to reduce unnecessary communication.

### Security objectives

| Objective              | Implementation                      |
| ---------------------- | ----------------------------------- |
| Workload isolation     | Default-deny policies               |
| Service access         | Explicit ingress and egress         |
| Database access        | Approved database destinations      |
| Messaging access       | Approved broker endpoints           |
| DNS access             | Controlled DNS permissions          |
| Monitoring access      | Prometheus scrape permissions       |
| Logging access         | Approved logging destinations       |
| Environment separation | Namespace and environment selectors |
| Policy governance      | Git reviews and approvals           |
| Continuous validation  | CI and connectivity tests           |

### Production security model

```
                   INTERNET
                      |
                      |
                AWS LOAD BALANCER
                      |
                      |
              INGRESS CONTROLLER
                      |
                      |
             NETWORK POLICY CHECK
                      |
          +-----------+-----------+
          |           |           |
          v           v           v
       PRODUCT      CART        ORDER
       SERVICE     SERVICE     SERVICE
          |           |           |
          |           |           |
          v           v           v
      INVENTORY    PRICING     PAYMENT
       SERVICE     SERVICE     SERVICE
                      |
                      v
                  DATABASE
                      |
                      v
                    RDS
```

The diagram represents an illustrative application flow. NetworkPolicy does not replace authentication, authorization, encryption, or application-level access control.

## 189.30. NetworkPolicy implementation layers

We should not create one large YAML file containing every permission.

Instead, divide the policy implementation into logical layers.

Layer 1 — Namespace isolation

Controls which workloads are isolated from unrestricted communication.

Layer 2 — Workload identity

Uses namespace and pod labels to identify the source and destination.

Layer 3 — Application communication

Allows only approved microservice-to-microservice traffic.

Layer 4 — Infrastructure communication

Allows DNS, databases, messaging, ingress and external dependencies.

Layer 5 — Observability communication

Preserves Prometheus scraping and log forwarding.

Layer 6 — Governance and validation

Validates, reviews, deploys and audits policies.

Each layer should have a clear owner and purpose.

### Why separate these layers?

Suppose an application cannot connect to PostgreSQL.

If all permissions are in one large policy, troubleshooting becomes difficult.

With separate policies, we can inspect:

```
default-deny-egress
allow-dns
allow-order-database
allow-order-payment
allow-prometheus
```

We can quickly identify the intended permission and check whether its selectors and ports are correct.

Remember that policies are additive. Separating them improves maintainability, but does not create independent security boundaries between policies.

## 189.31. Define the platform policy contract

Before building reusable Helm templates, establish a standard contract.

Every microservice should provide its network requirements in a consistent format.

Example:

`order-service/network-policy.yaml`

```
apiVersion: platform.company.io/v1
kind: ServiceNetworkPolicy

metadata:
  name: order-service-network
  namespace: ecommerce

spec:
  service:
    name: order-service

    labels:
      app.kubernetes.io/name: order-service
      app.kubernetes.io/part-of: ecommerce
      team: commerce

  ingress:
    - source:
        namespace: ingress-system
        service: ingress-controller

      port: 8080
      protocol: TCP

  egress:
    - destination:
        namespace: ecommerce
        service: inventory-service

      port: 8080
      protocol: TCP

    - destination:
        namespace: ecommerce
        service: payment-service

      port: 8080
      protocol: TCP

    - destination:
        namespace: ecommerce
        service: tax-service

      port: 8080
      protocol: TCP

    - destination:
        type: database
        name: order-postgresql

      port: 5432
      protocol: TCP
```

Important: `ServiceNetworkPolicy` is our proposed internal platform configuration format. It is not a built-in Kubernetes resource. Kubernetes will not understand it unless we create a CRD and controller or write a generator that converts it into standard NetworkPolicy resources.

For this chapter, we will use it as a documented input contract for the platform team.

### Contract responsibilities

| Field                   | Responsibility                 |
| ----------------------- | ------------------------------ |
| `service.name`          | Application owner              |
| `service.labels`        | Platform team                  |
| `ingress`               | Application and platform teams |
| `egress`                | Application team               |
| `destination.namespace` | Application/platform teams     |
| `destination.service`   | Application team               |
| `port`                  | Application owner              |
| `protocol`              | Application owner              |
| `database` destination  | Database/platform teams        |

### Contract validation rules

We should reject invalid definitions before generating policies.

For example:

1. Every service must have a unique name.
2. Every application must have a valid namespace.
3. Every destination must be approved.
4. Ports must be within the valid TCP/UDP port range.
5. Protocol must be explicitly supported.
6. Production policies must not allow unrestricted access without an approved exception.
7. Database destinations must have documented network ranges or a supported egress-control implementation.
8. Every exception must have an owner and review date.

## 189.32. Define standard labels across all microservices

The platform team should define the labels that all deployments must use.

Recommended standard:

```
app.kubernetes.io/name: order-service
app.kubernetes.io/part-of: ecommerce
app.kubernetes.io/component: backend
app.kubernetes.io/managed-by: helm
team: commerce
environment: production
```

We should not depend on arbitrary labels created independently by individual developers.

### Why use Kubernetes recommended labels?

For example:

```
app.kubernetes.io/name: order-service
```

This identifies the application consistently.

```
app.kubernetes.io/part-of: ecommerce
```

This identifies the larger product or system.

```
app.kubernetes.io/component: backend
```

This identifies the application component.

```
team: commerce
```

This identifies the owning team.

```
environment: production
```

This identifies the deployment environment.

### Standard labels in Helm

In an application Helm chart:

`templates/deployment.yaml`

```
apiVersion: apps/v1
kind: Deployment

metadata:
  name: {{ include "order-service.fullname" . }}

  labels:
    app.kubernetes.io/name: order-service
    app.kubernetes.io/part-of: ecommerce
    app.kubernetes.io/component: backend
    app.kubernetes.io/managed-by: helm
    team: commerce
    environment: {{ .Values.environment | quote }}

spec:
  replicas: {{ .Values.replicaCount }}

  selector:
    matchLabels:
      app.kubernetes.io/name: order-service

  template:
    metadata:
      labels:
        app.kubernetes.io/name: order-service
        app.kubernetes.io/part-of: ecommerce
        app.kubernetes.io/component: backend
        team: commerce
        environment: {{ .Values.environment | quote }}

    spec:
      containers:
        - name: order-service
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}

          ports:
            - name: http
              containerPort: 8080
              protocol: TCP
```

The Deployment selector and pod-template labels must remain compatible.

### Label verification

```
kubectl get deployments -n ecommerce \
  -o custom-columns=\
NAME:.metadata.name,\
SELECTOR:.spec.selector.matchLabels
```

Check actual pod labels:

```
kubectl get pods -n ecommerce --show-labels
```

Find Order Service pods:

```
kubectl get pods -n ecommerce \
  -l app.kubernetes.io/name=order-service
```

Find all e-commerce backend pods:

```
kubectl get pods -n ecommerce \
  -l app.kubernetes.io/component=backend
```

This consistent labeling makes reusable NetworkPolicy selectors possible.

## 189.33. Namespace architecture

Our enterprise platform should separate workloads according to operational responsibility.

Example:

```
EKS CLUSTER
│
├── ecommerce
│   ├── auth-service
│   ├── user-service
│   ├── customer-service
│   ├── catalog-service
│   ├── product-service
│   ├── inventory-service
│   ├── pricing-service
│   ├── promotion-service
│   ├── cart-service
│   ├── order-service
│   ├── payment-service
│   ├── fraud-service
│   ├── shipping-service
│   ├── delivery-service
│   ├── notification-service
│   ├── search-service
│   ├── recommendation-service
│   ├── review-service
│   ├── tax-service
│   ├── invoice-service
│   └── reporting-service
│
├── ingress-system
│   └── ingress-controller
│
├── monitoring
│   ├── prometheus
│   ├── grafana
│   ├── node-exporter
│   └── kube-state-metrics
│
├── logging
│   ├── fluent-bit
│   └── logstash
│
├── argocd
│   └── argocd-server
│
└── kube-system
    ├── coredns
    ├── aws-node
    └── kube-proxy
```

This is an example namespace design, not a requirement to run all these components in separate namespaces.

### Namespace creation

```
kubectl create namespace ecommerce

kubectl create namespace monitoring

kubectl create namespace logging

kubectl create namespace ingress-system
```

### Namespace labels

```
kubectl label namespace ecommerce \
  environment=production \
  platform.company.io/managed=true

kubectl label namespace monitoring \
  environment=production \
  platform.company.io/managed=true

kubectl label namespace logging \
  environment=production \
  platform.company.io/managed=true
```

Check:

```
kubectl get namespaces --show-labels
```

### Namespace selector behavior

Consider:

```
namespaceSelector:
  matchLabels:
    kubernetes.io/metadata.name: monitoring
```

This selects the namespace named `monitoring` through its standard namespace-name label.

Now consider:

```
podSelector:
  matchLabels:
    app.kubernetes.io/name: prometheus
```

When both selectors are in the same `from` or `to` entry, the selected pod must satisfy both conditions:

```
Namespace = monitoring
        AND
Pod label = prometheus
```

This is important.

A common configuration mistake is writing two separate list entries when the intention is to require both selectors together.

## 189.34. Design a namespace-level default-deny policy

Let's build the first reusable security template.

`templates/default-deny.yaml`

```
{{- if .Values.defaultDeny.enabled }}
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: default-deny
  namespace: {{ .Values.namespace }}

  labels:
    app.kubernetes.io/managed-by: helm
    app.kubernetes.io/part-of: ecommerce
    platform.company.io/policy-type: baseline

spec:
  podSelector: {}

  policyTypes:
    - Ingress
    - Egress

  ingress: []

  egress: []
{{- end }}
```

Corresponding values:

`values.yaml`

```
namespace: ecommerce

defaultDeny:
  enabled: true
```

This creates one policy that isolates all pods in the selected namespace for both ingress and egress.

### Render the policy

```
helm template network-policy \
  ./charts/network-policy \
  -f ./environments/dev/values.yaml
```

Expected important output:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: ecommerce
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  ingress: []
  egress: []
```

### Validate the YAML

```
helm lint ./charts/network-policy
```

If a Kubernetes API schema validation tool is configured:

```
kubeconform -strict rendered-dev.yaml
```

The exact schema tool version and Kubernetes version must match the cluster's supported API versions.

### Apply only in a test environment

```
kubectl apply -f rendered-dev.yaml
```

Check:

```
kubectl get networkpolicy -n ecommerce
```

Do not apply this policy in production until all required ingress and egress paths have been identified and validated.

## 189.35. Understand policy isolation in detail

This section is important for interviews and real production troubleshooting.

Suppose we have:

```
Pod A: Order Service
Pod B: Payment Service
```

Order Service sends a request to Payment Service.

For that connection to succeed:

```
Order Service
    |
    | Egress must allow
    v
Payment Service
    |
    | Ingress must allow
    v
Application
```

### Case 1: Source egress allowed, destination ingress unrestricted

The connection can succeed.

### Case 2: Source egress restricted and does not allow the destination

The connection is blocked.

### Case 3: Destination ingress restricted and does not allow the source

The connection is blocked.

### Case 4: Both source and destination are restricted

Both sides must allow the connection.

### Case 5: Another policy allows traffic

Because policies are additive, a separate matching policy can allow traffic that another policy does not allow.

Example:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-order-to-payment
  namespace: ecommerce

spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: order-service

  policyTypes:
    - Egress

  egress:
    - to:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: payment-service

      ports:
        - protocol: TCP
          port: 8080
```

This allows Order Service egress to Payment Service pods on TCP 8080.

It does not automatically allow the reverse direction.

It also does not provide authentication or verify whether the user placing the request is authorized to make a payment.

## 189.36. Design a policy ownership model

A platform engineering team should own the reusable policy framework, but service teams should own the declaration of their application dependencies.

Example ownership:

| Artifact                      | Owner              | Reviewer            |
| ----------------------------- | ------------------ | ------------------- |
| Default-deny template         | Platform           | Security            |
| DNS template                  | Platform           | Platform operations |
| Order-to-Payment dependency   | Application        | Payment team        |
| Order-to-Inventory dependency | Application        | Inventory team      |
| RDS egress requirements       | Platform/DBA       | Security            |
| Prometheus scrape access      | Observability team | Platform            |
| Logstash access               | Logging team       | Platform            |
| Production exceptions         | Requesting team    | Security            |

### Pull request workflow

```
Developer identifies dependency
             |
             v
Creates dependency change
             |
             v
Platform team reviews labels and selectors
             |
             v
Application owner confirms port
             |
             v
Security team reviews access
             |
             v
CI validates rendered policies
             |
             v
Approved Git merge
             |
             v
ArgoCD deploys policy
             |
             v
Connectivity tests
             |
             v
Production monitoring
```

### Example pull request description

```
## NetworkPolicy Change

Service:
order-service

Environment:
staging

Source namespace:
ecommerce

Destination:
payment-service

Protocol:
TCP

Destination port:
8080

Business reason:
Order Service must communicate with Payment Service
to initiate payment processing.

Security impact:
Adds narrowly scoped egress access from Order Service
to Payment Service.

Testing:
- DNS resolution tested
- Approved connection tested
- Unapproved connection tested
- Application integration tests passed

Rollback:
Revert the Git commit and allow ArgoCD to reconcile
the previous approved policy configuration.

Approvals:
- Application owner
- Platform team
- Security team
```

This creates an auditable history of why network access was granted.

## 189.37. Production policy discovery before enforcement

Before enabling restrictive policies, we need to understand the actual traffic.

Do not assume the architecture diagram contains every dependency.

Some services may use:

- External payment gateways.
- Third-party fraud APIs.
- Email delivery services.
- Object storage.
- Managed databases.
- Internal DNS.
- Container registry endpoints.
- Messaging brokers.
- Monitoring agents.
- Other platform endpoints.

### Discovery checklist

| Discovery area     | Information to collect               |
| ------------------ | ------------------------------------ |
| Service-to-service | Source, destination and port         |
| Databases          | Endpoint, protocol and port          |
| Messaging          | Broker endpoints and listeners       |
| DNS                | DNS server and query behavior        |
| Ingress            | Actual source workload               |
| Monitoring         | Scrape source and metrics port       |
| Logging            | Agent and Logstash endpoints         |
| External APIs      | Destination and egress architecture  |
| AWS services       | VPC endpoints or other network paths |
| Platform services  | Required cluster components          |

### Example dependency inventory

```
service: order-service

namespace: ecommerce

dependencies:
  services:
    - name: inventory-service
      port: 8080
      protocol: TCP
      required: true

    - name: payment-service
      port: 8080
      protocol: TCP
      required: true

    - name: tax-service
      port: 8080
      protocol: TCP
      required: true

  databases:
    - name: order-postgresql
      port: 5432
      protocol: TCP
      required: true

  dns:
    enabled: true

  monitoring:
    enabled: true
    metricsPort: 8081

  logging:
    enabled: true
```

This inventory is useful for documentation, code generation and review. It is not itself an enforceable Kubernetes policy.

### Discovery questions for application developers

Ask each service owner:

1. Which services does your application call?
2. Which services call your application?
3. Which port does your application listen on?
4. Does your application connect to RDS?
5. Does it publish or consume messaging events?
6. Does it call external APIs?
7. Does it require object storage?
8. Does it expose a metrics endpoint?
9. Does it use a health-check endpoint?
10. Does it depend on any platform service?

The platform team should not invent dependencies to complete the matrix.

## 189.38. Establish a policy exception process

Sometimes a workload needs temporary access for debugging or an unusual integration.

Do not create permanent broad permissions just because a temporary requirement exists.

Example exception record:

```
exception:
  id: NET-EXC-1042

  service: order-service

  namespace: ecommerce

  destination:
    name: external-tax-provider

  port: 443
  protocol: TCP

  reason: External tax calculation API

  owner: commerce-platform-team

  requestedBy: order-service-team

  approvedBy: security-team

  environment: production

  reviewDate: "2026-11-30"

  status: approved
```

This is an example governance record, not a Kubernetes resource.

The policy implementation must still translate the approved exception into a supported network rule.

### Exception requirements

- Business justification.
- Named owner.
- Approved destination.
- Approved protocol and port.
- Environment scope.
- Expiry or review date.
- Testing evidence.
- Rollback procedure.

### Exception review

At regular intervals, review:

```
Approved exceptions
       |
       v
Still required?
       |
   +---+---+
   |       |
  Yes      No
   |       |
   v       v
Renew    Remove
```

This helps prevent temporary permissions from becoming permanent.

## 189.39. Part 1 completion checklist

- Network security objectives defined.
- EKS CNI enforcement requirement documented.
- Platform policy layers established.
- Network policy contract documented.
- Standard pod labels defined.
- Namespace architecture documented.
- Namespace selectors understood.
- Default-deny Helm template created.
- Policy isolation behavior understood.
- Ownership and approval process defined.
- Application dependency discovery process documented.
- Exception governance established.

## 189.40. Complete 21-microservice inventory

Before generating policies, we need to understand every service and its network responsibilities.

Our 21 services are:

| ID  | Microservice           | Primary responsibility              |
| --- | ---------------------- | ----------------------------------- |
| 161 | Auth Service           | Authentication and token validation |
| 162 | User Service           | User account management             |
| 163 | Customer Service       | Customer profiles                   |
| 164 | Catalog Service        | Product catalog organization        |
| 165 | Product Service        | Product information                 |
| 166 | Inventory Service      | Stock management                    |
| 167 | Pricing Service        | Product pricing                     |
| 168 | Promotion Service      | Discounts and promotions            |
| 169 | Cart Service           | Shopping cart management            |
| 170 | Order Service          | Order orchestration                 |
| 171 | Payment Service        | Payment processing                  |
| 172 | Fraud Service          | Fraud checks                        |
| 173 | Shipping Service       | Shipment coordination               |
| 174 | Delivery Service       | Delivery tracking                   |
| 175 | Notification Service   | Email and notification processing   |
| 176 | Search Service         | Product search                      |
| 177 | Recommendation Service | Product recommendations             |
| 178 | Review Service         | Product reviews                     |
| 179 | Tax Service            | Tax calculation                     |
| 180 | Invoice Service        | Invoice generation                  |
| 181 | Reporting Service      | Business reporting                  |

The service descriptions above define their broad responsibilities. They do not establish every network dependency. The actual dependency graph must be confirmed with application owners.

### 189.40.1. Define service ports

For this implementation example, we will use TCP 8080 as the application port and TCP 8081 as the metrics port.

These are project conventions for the examples, not verified ports for every application.

| Service        | Application port | Metrics port |
| -------------- | ---------------- | ------------ |
| Auth           | 8080             | 8081         |
| User           | 8080             | 8081         |
| Customer       | 8080             | 8081         |
| Catalog        | 8080             | 8081         |
| Product        | 8080             | 8081         |
| Inventory      | 8080             | 8081         |
| Pricing        | 8080             | 8081         |
| Promotion      | 8080             | 8081         |
| Cart           | 8080             | 8081         |
| Order          | 8080             | 8081         |
| Payment        | 8080             | 8081         |
| Fraud          | 8080             | 8081         |
| Shipping       | 8080             | 8081         |
| Delivery       | 8080             | 8081         |
| Notification   | 8080             | 8081         |
| Search         | 8080             | 8081         |
| Recommendation | 8080             | 8081         |
| Review         | 8080             | 8081         |
| Tax            | 8080             | 8081         |
| Invoice        | 8080             | 8081         |
| Reporting      | 8080             | 8081         |

If one service listens on port 8090, its policy must use 8090. A NetworkPolicy does not automatically discover the application's listening port.

### 189.40.2. Define dependency categories

We classify dependencies into five categories.

```
DEPENDENCY CATEGORIES
|
+-- Application
|   +-- Microservice to microservice
|
+-- Data
|   +-- RDS
|   +-- Database endpoints
|
+-- Messaging
|   +-- RabbitMQ or Kafka
|
+-- Platform
|   +-- DNS
|   +-- Ingress controller
|   +-- Kubernetes services
|
+-- Observability
    +-- Prometheus
    +-- Node Exporter
    +-- Logstash
```

This classification helps us create reusable policies without granting unrelated access.

## 189.41. Create the complete communication matrix

The communication matrix is the source of truth for application network access.

We will distinguish between:

- Confirmed project dependencies.
- Proposed example dependencies.
- Dependencies still requiring application-owner confirmation.

### 189.41.1. Example dependency matrix

The following is a proposed e-commerce communication design. It must be checked against the implemented service-to-service calls before deployment.

| Source         | Destination | Port | Reason                |
| -------------- | ----------- | ---- | --------------------- |
| Customer       | User        | 8080 | User information      |
| Catalog        | Product     | 8080 | Product information   |
| Product        | Inventory   | 8080 | Stock information     |
| Product        | Pricing     | 8080 | Product pricing       |
| Cart           | Product     | 8080 | Product details       |
| Cart           | Pricing     | 8080 | Price calculation     |
| Cart           | Promotion   | 8080 | Discount calculation  |
| Order          | Customer    | 8080 | Customer validation   |
| Order          | Inventory   | 8080 | Stock validation      |
| Order          | Payment     | 8080 | Payment processing    |
| Order          | Fraud       | 8080 | Fraud validation      |
| Order          | Tax         | 8080 | Tax calculation       |
| Order          | Shipping    | 8080 | Shipment creation     |
| Payment        | Fraud       | 8080 | Payment risk check    |
| Shipping       | Delivery    | 8080 | Delivery coordination |
| Notification   | Order       | 8080 | Order event details   |
| Search         | Product     | 8080 | Product indexing      |
| Recommendation | Product     | 8080 | Product information   |
| Review         | Product     | 8080 | Product validation    |
| Invoice        | Order       | 8080 | Invoice information   |
| Reporting      | Order       | 8080 | Order reporting       |
| Reporting      | Payment     | 8080 | Payment reporting     |

### 189.41.2. Validate each dependency

For every row, the application owner should answer:

1. Is this communication synchronous HTTP or another protocol?
2. Is the destination accessed through a Kubernetes Service?
3. Does the destination use the same namespace?
4. What is the actual destination port?
5. Is TLS used?
6. Is the communication required in production?
7. Is the dependency optional or mandatory?
8. What is the expected behavior if the destination is unavailable?

Example dependency declaration:

```
source:
  service: order-service
  namespace: ecommerce

destination:
  service: payment-service
  namespace: ecommerce

connection:
  protocol: TCP
  port: 8080

purpose: payment-processing

required: true

owner:
  source: commerce-team
  destination: payments-team

environment:
  - dev
  - qa
  - staging
  - production
```

This is an internal dependency record. It must be translated into a Kubernetes NetworkPolicy before enforcement.

### 189.41.3. Dependencies that must not be assumed

Do not automatically add all these connections to every microservice:

```
Every service -> Every database
Every service -> RabbitMQ
Every service -> Kafka
Every service -> Internet
Every service -> Every other service
```

That would defeat the purpose of network isolation.

Each application should receive only its required permissions.

## 189.42. Complete Helm chart structure

We will now define the reusable chart.

Repository:

```
company-ecommerce-platform/
|
+-- network-policies/
|   |
|   +-- charts/
|   |   |
|   |   +-- network-policy/
|   |       |
|   |       +-- Chart.yaml
|   |       |
|   |       +-- values.yaml
|   |       |
|   |       +-- values.schema.json
|   |       |
|   |       +-- templates/
|   |           |
|   |           +-- _helpers.tpl
|   |           |
|   |           +-- default-deny.yaml
|   |           |
|   |           +-- allow-dns.yaml
|   |           |
|   |           +-- allow-service-ingress.yaml
|   |           |
|   |           +-- allow-service-egress.yaml
|   |           |
|   |           +-- allow-database.yaml
|   |           |
|   |           +-- allow-monitoring.yaml
|   |           |
|   |           +-- allow-ingress-controller.yaml
|   |           |
|   |           +-- allow-logging.yaml
|   |           |
|   |           +-- allow-messaging.yaml
|   |           |
|   |           +-- allow-external-egress.yaml
|   |
|   +-- environments/
|   |   |
|   |   +-- dev/
|   |   +-- qa/
|   |   +-- staging/
|   |   +-- production/
|   |
|   +-- tests/
|   |   |
|   |   +-- connectivity-matrix.yaml
|   |   +-- policy-test-cases.md
|   |   +-- expected-policies.yaml
|   |
|   +-- scripts/
|   |   |
|   |   +-- validate-chart.sh
|   |   +-- render-policies.sh
|   |   +-- check-policy-resources.sh
|   |
|   +-- README.md
```

### 189.42.1. Chart metadata

`charts/network-policy/Chart.yaml`

```
apiVersion: v2

name: network-policy

description: Reusable network security policies for ecommerce microservices

type: application

version: 1.0.0

appVersion: "1.0"
```

### 189.42.2. Template helper functions

`templates/_helpers.tpl`

```
{{/*
Return the chart name.
*/}}

{{- define "network-policy.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" -}}
{{- end -}}


{{/*
Return the release fullname.
*/}}

{{- define "network-policy.fullname" -}}
{{- printf "%s-%s" .Release.Name (include "network-policy.name" .) | trunc 63 | trimSuffix "-" -}}
{{- end -}}


{{/*
Return common labels.
*/}}

{{- define "network-policy.labels" -}}

app.kubernetes.io/name: {{ include "network-policy.name" . }}

app.kubernetes.io/instance: {{ .Release.Name }}

app.kubernetes.io/managed-by: {{ .Release.Service }}

app.kubernetes.io/part-of: ecommerce

{{- end -}}


{{/*
Return service selector labels.
*/}}

{{- define "network-policy.serviceSelector" -}}

app.kubernetes.io/name: {{ .serviceName }}

{{- end -}}
```

### 189.42.3. Default chart values

`charts/network-policy/values.yaml`

```
nameOverride: ""

namespace: ecommerce

environment: dev

defaultDeny:
  enabled: true

dns:
  enabled: true

  namespace: kube-system

  podLabels:
    k8s-app: kube-dns

  ports:
    - protocol: UDP
      port: 53

    - protocol: TCP
      port: 53

ingressController:
  enabled: true

  namespace: ingress-system

  podLabels:
    app.kubernetes.io/name: ingress-controller

  applicationPort: 8080

monitoring:
  enabled: true

  namespace: monitoring

  podLabels:
    app.kubernetes.io/name: prometheus

  metricsPort: 8081

logging:
  enabled: false

  namespace: logging

  logstashLabels:
    app.kubernetes.io/name: logstash

  fluentBitLabels:
    app.kubernetes.io/name: fluent-bit

  port: 5044

services: []

databases: []

messaging: []

externalEgress: []

policyLabels:
  app.kubernetes.io/part-of: ecommerce
  platform.company.io/managed: "true"
```

This is a baseline. The environment-specific files will provide the actual service dependencies and approved database destinations.

### 189.42.4. Default-deny template

`templates/default-deny.yaml`

```
{{- if .Values.defaultDeny.enabled }}

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: default-deny
  namespace: {{ .Values.namespace }}

  labels:
    {{- include "network-policy.labels" . | nindent 4 }}

spec:
  podSelector: {}

  policyTypes:
    - Ingress
    - Egress

  ingress: []

  egress: []

{{- end }}
```

This policy isolates all pods in the configured namespace for ingress and egress.

It does not independently guarantee that all traffic is blocked if another NetworkPolicy selects the same pods and allows that traffic.

### 189.42.5. DNS template

`templates/allow-dns.yaml`

```
{{- if .Values.dns.enabled }}

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-dns
  namespace: {{ .Values.namespace }}

  labels:
    {{- include "network-policy.labels" . | nindent 4 }}

spec:
  podSelector: {}

  policyTypes:
    - Egress

  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: {{ .Values.dns.namespace }}

          podSelector:
            matchLabels:
              {{- toYaml .Values.dns.podLabels | nindent 14 }}

      ports:
        {{- toYaml .Values.dns.ports | nindent 8 }}

{{- end }}
```

### 189.42.6. Service egress template

We need to allow a service to call only its approved destinations.

`templates/allow-service-egress.yaml`

```
{{- range $service := .Values.services }}

{{- range $dependency := $service.egress }}

---

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-{{ $service.name }}-to-{{ $dependency.name }}
  namespace: {{ $.Values.namespace }}

  labels:
    {{- include "network-policy.labels" $ | nindent 4 }}

spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: {{ $service.name }}

  policyTypes:
    - Egress

  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: {{ default $.Values.namespace $dependency.namespace }}

          podSelector:
            matchLabels:
              app.kubernetes.io/name: {{ $dependency.name }}

      ports:
        - protocol: {{ default "TCP" $dependency.protocol }}
          port: {{ $dependency.port }}

{{- end }}
{{- end }}
```

This template assumes the source service and destination service are represented in the same chart's values.

It creates a separate NetworkPolicy for each declared dependency. It also supports a destination namespace different from the chart's configured namespace.

The namespace and pod selectors are intentionally combined in one `to` entry.

### 189.42.7. Service ingress template

The destination service must also allow traffic from approved sources.

`templates/allow-service-ingress.yaml`

```
{{- range $service := .Values.services }}

{{- range $rule := $service.ingress }}

---

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-{{ $rule.source }}-to-{{ $service.name }}
  namespace: {{ $.Values.namespace }}

  labels:
    {{- include "network-policy.labels" $ | nindent 4 }}

spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: {{ $service.name }}

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: {{ default $.Values.namespace $rule.namespace }}

          podSelector:
            matchLabels:
              app.kubernetes.io/name: {{ $rule.source }}

      ports:
        - protocol: {{ default "TCP" $rule.protocol }}
          port: {{ $rule.port }}

{{- end }}
{{- end }}
```

This template supports source workloads in another namespace.

For example, Payment Service can allow Order Service traffic without allowing every pod in the cluster.

### 189.42.8. Database egress template

For standard Kubernetes NetworkPolicy, external database destinations are represented by IP blocks rather than DNS names.

`templates/allow-database.yaml`

```
{{- range $database := .Values.databases }}

---

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-{{ $database.source }}-to-{{ $database.name }}
  namespace: {{ $.Values.namespace }}

  labels:
    {{- include "network-policy.labels" $ | nindent 4 }}

spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: {{ $database.source }}

  policyTypes:
    - Egress

  egress:
    - to:
        - ipBlock:
            cidr: {{ $database.cidr | quote }}

      ports:
        - protocol: {{ default "TCP" $database.protocol }}
          port: {{ $database.port }}

{{- end }}
```

Example values:

```
databases:
  - name: order-postgresql

    source: order-service

    cidr: 10.20.30.15/32

    port: 5432

    protocol: TCP

  - name: payment-postgresql

    source: payment-service

    cidr: 10.20.30.16/32

    port: 5432

    protocol: TCP
```

These addresses are examples only.

Before using `/32` destinations, verify that the database endpoint resolves to a stable address and that the CNI enforces the expected policy on pod-to-database traffic.

If an RDS endpoint resolves to changing IP addresses, a static IP-based rule may stop working. Standard Kubernetes NetworkPolicy cannot directly match an FQDN.

### 189.42.9. Monitoring ingress template

`templates/allow-monitoring.yaml`

```
{{- if .Values.monitoring.enabled }}

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-prometheus-scraping
  namespace: {{ .Values.namespace }}

  labels:
    {{- include "network-policy.labels" . | nindent 4 }}

spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: ecommerce

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: {{ .Values.monitoring.namespace }}

          podSelector:
            matchLabels:
              {{- toYaml .Values.monitoring.podLabels | nindent 14 }}

      ports:
        - protocol: TCP
          port: {{ .Values.monitoring.metricsPort }}

{{- end }}
```

This assumes all selected application pods expose metrics on the configured port. If some services use different ports, split the policy by workload or generate per-service rules.

### 189.42.10. Ingress controller template

`templates/allow-ingress-controller.yaml`

```
{{- if .Values.ingressController.enabled }}

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-ingress-controller
  namespace: {{ .Values.namespace }}

  labels:
    {{- include "network-policy.labels" . | nindent 4 }}

spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/part-of: ecommerce

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: {{ .Values.ingressController.namespace }}

          podSelector:
            matchLabels:
              {{- toYaml .Values.ingressController.podLabels | nindent 14 }}

      ports:
        - protocol: TCP
          port: {{ .Values.ingressController.applicationPort }}

{{- end }}
```

This policy is appropriate only if the actual traffic reaches application pods from the selected ingress-controller pods. Verify the packet path for the specific load balancer and ingress architecture.

### 189.42.11. Messaging template

`templates/allow-messaging.yaml`

```
{{- range $broker := .Values.messaging }}

---

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-{{ $broker.source }}-to-{{ $broker.name }}
  namespace: {{ $.Values.namespace }}

  labels:
    {{- include "network-policy.labels" $ | nindent 4 }}

spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: {{ $broker.source }}

  policyTypes:
    - Egress

  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: {{ default $.Values.namespace $broker.namespace }}

          podSelector:
            matchLabels:
              app.kubernetes.io/name: {{ $broker.name }}

      ports:
        - protocol: TCP
          port: {{ $broker.port }}

{{- end }}
```

Example RabbitMQ values:

```
messaging:
  - name: rabbitmq

    source: order-service

    namespace: messaging

    port: 5672
```

Example Kafka values:

```
messaging:
  - name: kafka

    source: order-service

    namespace: messaging

    port: 9092
```

Use the actual listener port and namespace. Do not enable both broker examples unless the architecture actually uses both.

### 189.42.12. External egress template

A standard NetworkPolicy can allow an external destination through `ipBlock`, but it cannot select a DNS hostname.

`templates/allow-external-egress.yaml`

```
{{- range $rule := .Values.externalEgress }}

---

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-{{ $rule.source }}-external
  namespace: {{ $.Values.namespace }}

  labels:
    {{- include "network-policy.labels" $ | nindent 4 }}

spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: {{ $rule.source }}

  policyTypes:
    - Egress

  egress:
    - to:
        - ipBlock:
            cidr: {{ $rule.cidr | quote }}

      ports:
        - protocol: {{ default "TCP" $rule.protocol }}
          port: {{ $rule.port }}

{{- end }}
```

Example:

```
externalEgress:
  - source: payment-service

    cidr: 198.51.100.10/32

    port: 443

    protocol: TCP
```

`198.51.100.10` belongs to a documentation-only address range. Replace it with a real, approved destination range before deployment.

For external APIs with changing IP addresses, use a supported FQDN-aware CNI feature, controlled egress proxy, or firewall architecture rather than trying to maintain unreliable static IP lists.

## 189.43. Configure service-specific Helm values

Now we can configure the policies for individual services.

`environments/dev/values.yaml`

```
namespace: ecommerce

environment: dev

defaultDeny:
  enabled: true

dns:
  enabled: true

ingressController:
  enabled: true

monitoring:
  enabled: true

logging:
  enabled: false

services:

  - name: customer-service

    ingress:
      - source: ingress-controller
        namespace: ingress-system
        port: 8080
        protocol: TCP

    egress:
      - name: user-service
        port: 8080
        protocol: TCP

  - name: catalog-service

    ingress:
      - source: ingress-controller
        namespace: ingress-system
        port: 8080
        protocol: TCP

    egress:
      - name: product-service
        port: 8080
        protocol: TCP

  - name: product-service

    egress:
      - name: inventory-service
        port: 8080
        protocol: TCP

      - name: pricing-service
        port: 8080
        protocol: TCP

  - name: cart-service

    ingress:
      - source: ingress-controller
        namespace: ingress-system
        port: 8080
        protocol: TCP

    egress:
      - name: product-service
        port: 8080
        protocol: TCP

      - name: pricing-service
        port: 8080
        protocol: TCP

      - name: promotion-service
        port: 8080
        protocol: TCP

  - name: order-service

    ingress:
      - source: ingress-controller
        namespace: ingress-system
        port: 8080
        protocol: TCP

    egress:
      - name: customer-service
        port: 8080
        protocol: TCP

      - name: inventory-service
        port: 8080
        protocol: TCP

      - name: payment-service
        port: 8080
        protocol: TCP

      - name: fraud-service
        port: 8080
        protocol: TCP

      - name: tax-service
        port: 8080
        protocol: TCP

      - name: shipping-service
        port: 8080
        protocol: TCP
```

This is a sample DEV configuration, not a declaration that these are the final dependencies of the project.

The same pattern can be extended to the remaining services after dependency confirmation.

### 189.43.1. Add database configuration

```
databases:

  - name: order-postgresql

    source: order-service

    cidr: 10.20.30.15/32

    port: 5432

    protocol: TCP

  - name: payment-postgresql

    source: payment-service

    cidr: 10.20.30.16/32

    port: 5432

    protocol: TCP
```

### 189.43.2. Add messaging configuration

```
messaging:

  - name: rabbitmq

    source: order-service

    namespace: messaging

    port: 5672
```

Only include this if the Order Service actually connects to RabbitMQ.

### 189.43.3. Add external API configuration

```
externalEgress:

  - source: payment-service

    cidr: 198.51.100.10/32

    port: 443

    protocol: TCP
```

The destination CIDR must be replaced with an approved, verified range. A changing third-party API address requires an appropriate egress-control architecture.

## 189.44. Validate the Helm chart

Before deployment, render the chart.

```
helm lint \
  ./charts/network-policy
```

Render DEV:

```
helm template ecommerce-network-policy \
  ./charts/network-policy \
  --namespace ecommerce \
  -f ./environments/dev/values.yaml \
  > rendered-dev.yaml
```

Render QA:

```
helm template ecommerce-network-policy \
  ./charts/network-policy \
  --namespace ecommerce \
  -f ./environments/qa/values.yaml \
  > rendered-qa.yaml
```

Render STAGING:

```
helm template ecommerce-network-policy \
  ./charts/network-policy \
  --namespace ecommerce \
  -f ./environments/staging/values.yaml \
  > rendered-staging.yaml
```

Render PRODUCTION:

```
helm template ecommerce-network-policy \
  ./charts/network-policy \
  --namespace ecommerce \
  -f ./environments/production/values.yaml \
  > rendered-production.yaml
```

Validate that the files exist and contain policies:

```
for file in rendered-dev.yaml \
            rendered-qa.yaml \
            rendered-staging.yaml \
            rendered-production.yaml
do
    test -s "$file" || {
        echo "ERROR: Missing or empty output: $file"
        exit 1
    }

    grep -q "kind: NetworkPolicy" "$file" || {
        echo "ERROR: No NetworkPolicy found in $file"
        exit 1
    }

    echo "Validated: $file"
done
```

### 189.44.1. Check generated policy names

```
grep -E "name:|namespace:|kind:" rendered-dev.yaml
```

### 189.44.2. Inspect generated policies

```
kubectl apply \
  --dry-run=server \
  -f rendered-dev.yaml
```

Server-side dry-run validates the resources against the API server, but it does not prove that the CNI will enforce the desired traffic behavior.

### 189.44.3. Check Helm template output

```
helm template ecommerce-network-policy \
  ./charts/network-policy \
  -f ./environments/dev/values.yaml \
  --debug
```

Review:

- Correct namespace.
- Correct pod selectors.
- Correct namespace selectors.
- Correct destination labels.
- Correct ports.
- Correct policy types.
- No unintended broad permissions.

## 189.45. Production-specific EKS networking considerations

NetworkPolicy is one layer of network security. On EKS, it must work alongside AWS networking controls.

### 189.45.1. Understand the network path

For a pod connecting to RDS, the path may look like:

```
Order Service Pod
       |
       v
Kubernetes NetworkPolicy
       |
       v
EKS Pod Network
       |
       v
AWS VPC Routing
       |
       v
Network ACL
       |
       v
RDS Security Group
       |
       v
RDS PostgreSQL
```

The exact enforcement path depends on the CNI configuration, pod networking mode, security groups and routing architecture.

### 189.45.2. Amazon VPC CNI

Inspect the installed VPC CNI:

```
kubectl get daemonset aws-node -n kube-system

kubectl describe daemonset aws-node -n kube-system

kubectl get pods -n kube-system -l k8s-app=aws-node
```

Inspect the installed add-on:

```
aws eks describe-addon \
  --cluster-name ecommerce-production \
  --addon-name vpc-cni \
  --region ap-south-1
```

Check the installed version and its supported NetworkPolicy configuration before enabling enforcement.

Do not assume that the presence of the `aws-node` DaemonSet means NetworkPolicy is enabled.

### 189.45.3. AWS security groups

Security groups control traffic at the AWS network interface level.

For example, the RDS security group might allow PostgreSQL traffic from an approved EKS security group.

Example conceptual rule:

```
RDS Security Group
|
+-- Inbound
    |
    +-- Protocol: TCP
    +-- Port: 5432
    +-- Source: Approved EKS workload security group
```

NetworkPolicy and security groups solve different problems.

A Kubernetes NetworkPolicy can select application pods. An AWS security group controls traffic associated with AWS network interfaces.

Both need to be configured correctly.

### 189.45.4. Security Groups for Pods

AWS Security Groups for Pods can provide security-group-level controls for selected pod workloads when the EKS configuration supports the required features.

A conceptual architecture:

```
Order Service Pod
       |
       v
Pod Network Interface
       |
       v
Order Service Security Group
       |
       v
RDS Security Group
```

This can provide more granular AWS network controls, but it does not replace application-level NetworkPolicy requirements.

Verify compatibility with the selected VPC CNI mode and cluster configuration.

### 189.45.5. External AWS services

Some workloads may need access to AWS services such as:

- S3
- ECR
- Secrets Manager
- Systems Manager
- Other approved AWS APIs

Do not give all application pods unrestricted internet access for these requirements.

Consider:

- VPC endpoints where supported.
- Private DNS configuration.
- Security-group rules.
- Route-table restrictions.
- Controlled egress architecture.
- IAM permissions for the AWS service itself.

Network reachability and IAM authorization are separate controls.

## 189.46. Cross-namespace service communication

Some platform components are in separate namespaces.

Example:

```
ecommerce namespace
       |
       v
monitoring namespace

ecommerce namespace
       |
       v
messaging namespace

ecommerce namespace
       |
       v
ingress-system namespace
```

A policy allowing traffic from another namespace needs a namespace selector.

Example:

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-order-to-payment-cross-namespace
  namespace: ecommerce

spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: order-service

  policyTypes:
    - Egress

  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: payments

          podSelector:
            matchLabels:
              app.kubernetes.io/name: payment-service

      ports:
        - protocol: TCP
          port: 8080
```

This example assumes Payment Service runs in the `payments` namespace.

If the destination is in another namespace, the destination's ingress policy must also allow the source.

### 189.46.1. Cross-namespace test

```
kubectl get pods -n ecommerce --show-labels

kubectl get pods -n payments --show-labels

kubectl get networkpolicy -n ecommerce

kubectl get networkpolicy -n payments
```

Test the connection using a workload that has the same relevant labels as the real source application.

### 189.46.2. Prevent broad namespace access

Avoid policies such as:

```
from:
  - namespaceSelector: {}
```

This selects pods from all namespaces, subject to the other rule conditions.

If broad access is not explicitly required, use a specific namespace label and pod label.

## 189.47. NetworkPolicy and Kubernetes Services

A common troubleshooting question is whether a policy should select a Kubernetes Service or a pod.

NetworkPolicy selects pods, not Kubernetes Service objects.

Example:

```
apiVersion: v1
kind: Service

metadata:
  name: payment-service
  namespace: ecommerce

spec:
  selector:
    app.kubernetes.io/name: payment-service

  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

The NetworkPolicy selects the destination pods:

```
podSelector:
  matchLabels:
    app.kubernetes.io/name: payment-service
```

It does not select the Service resource by name.

### 189.47.1. Service port versus target port

Consider:

```
ports:
  - port: 80
    targetPort: 8080
```

The Service exposes port 80, while the application container receives traffic on port 8080.

When writing NetworkPolicy rules, use the port as observed by the policy enforcement implementation on the relevant pod traffic path. For ordinary pod-to-Service traffic this is commonly the destination pod port, but verify the behavior of the selected CNI and networking path.

Do not blindly copy the Service's `port` value into a policy.

## 189.48. NetworkPolicy and health checks

Applications commonly have:

- Startup probes.
- Readiness probes.
- Liveness probes.

Example:

```
startupProbe:
  httpGet:
    path: /actuator/health
    port: 8080

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080

livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
```

Kubelet-originated traffic and probe behavior need special attention when restrictive NetworkPolicies are introduced.

Depending on the networking implementation and probe configuration, the source observed by the policy may not match the pod-to-pod traffic model.

Before enforcing policies:

1. Inspect the probe configuration.
2. Determine how kubelet probes reach the pod.
3. Verify whether the CNI applies policy enforcement to that path.
4. Test readiness and liveness after applying the policy.
5. Check whether the load balancer health checks are affected.

Do not automatically open every node CIDR to every application pod. Determine the actual required source and port.

## 189.49. NetworkPolicy and monitoring architecture

Our existing monitoring stack is Prometheus, Grafana, Node Exporter and kube-state-metrics.

Example architecture:

```
                PROMETHEUS
                    |
        +-----------+-----------+
        |           |           |
        v           v           v
   Application   Node Exporter  kube-state-metrics
     Metrics        Metrics          Metrics
        |           |                 |
        +-----------+-----------------+
                    |
                    v
                  Grafana
```

NetworkPolicy requirements:

| Source     | Destination         | Port                    |
| ---------- | ------------------- | ----------------------- |
| Prometheus | Application metrics | 8081                    |
| Prometheus | Node Exporter       | 9100                    |
| Prometheus | kube-state-metrics  | 8080 or configured port |
| Grafana    | Prometheus          | 9090                    |

These are common example ports. Inspect the actual Service and container configurations.

### 189.49.1. Inspect monitoring ports

```
kubectl get services -n monitoring

kubectl get pods -n monitoring

kubectl describe service prometheus -n monitoring

kubectl describe service grafana -n monitoring
```

### 189.49.2. Inspect application metrics services

```
kubectl get services -n ecommerce

kubectl get endpointslices -n ecommerce
```

### 189.49.3. Validate scrape connectivity

Check:

- Prometheus can resolve the target.
- Prometheus can connect to the target port.
- The metrics endpoint returns a valid response.
- The scrape configuration points to the right endpoint.
- NetworkPolicy allows the actual Prometheus source pods.

### 189.49.4. Do not expose metrics unnecessarily

Metrics endpoints may expose:

- Application versions.
- Runtime details.
- Request counts.
- Internal endpoint names.
- Database pool statistics.
- Application performance information.

Only approved monitoring workloads should be able to scrape protected application metrics endpoints.

## 189.50. NetworkPolicy and logging architecture

Our logging implementation uses ELK with Logstash.

A logging agent such as Fluent Bit can collect container logs from the node and forward them to Logstash.

Example:

```
Application Containers
        |
        v
Container stdout/stderr
        |
        v
Fluent Bit
        |
        | TCP 5044 (example)
        v
Logstash
        |
        v
Elasticsearch
        |
        v
Kibana
```

NetworkPolicy requirements depend on where the logging components run and how they communicate.

### 189.50.1. Inspect Logstash service

```
kubectl get services -n logging

kubectl describe service logstash -n logging
```

### 189.50.2. Inspect Fluent Bit

```
kubectl get daemonset -n logging

kubectl get pods -n logging
```

### 189.50.3. Check Fluent Bit logs

```
kubectl logs daemonset/fluent-bit -n logging \
  --tail=100
```

The actual DaemonSet name may differ.

### 189.50.4. Check Logstash logs

```
kubectl logs deployment/logstash -n logging \
  --tail=100
```

The actual workload may be a StatefulSet or another controller.

### 189.50.5. Important node-level distinction

Fluent Bit often runs as a DaemonSet and reads container logs from the node filesystem.

NetworkPolicy controls pod network traffic. It does not replace host filesystem permissions or node-level security.

You must separately configure:

- HostPath access.
- Service account permissions.
- Container security context.
- Log file access.
- Logstash network access.

## 189.51. NetworkPolicy testing framework

A production platform should test both permitted and forbidden communication.

We will define test categories.

| Test                               | Expected result |
| ---------------------------------- | --------------- |
| Order → Payment                    | Allowed         |
| Order → Inventory                  | Allowed         |
| Order → Unapproved service         | Denied          |
| Application → CoreDNS              | Allowed         |
| Application → Unapproved namespace | Denied          |
| Prometheus → Metrics endpoint      | Allowed         |
| Unapproved pod → Metrics endpoint  | Denied          |
| Order → Approved RDS endpoint      | Allowed         |
| Order → Unapproved database        | Denied          |

The test must run with realistic source labels and network paths.

### 189.51.1. Connectivity test pod

Create a temporary pod:

```
apiVersion: v1
kind: Pod

metadata:
  name: order-network-test
  namespace: ecommerce

  labels:
    app.kubernetes.io/name: order-service
    app.kubernetes.io/part-of: ecommerce
    environment: dev

spec:
  restartPolicy: Never

  containers:
    - name: network-test

      image: busybox:1.36

      command:
        - sleep
        - "3600"
```

This is a test pod, not an additional production application replica.

Use a reviewed and pinned image version or digest in a controlled environment.

Apply:

```
kubectl apply -f order-network-test.yaml
```

Wait:

```
kubectl wait \
  --for=condition=Ready \
  pod/order-network-test \
  -n ecommerce \
  --timeout=120s
```

### 189.51.2. Test allowed service

```
kubectl exec -n ecommerce order-network-test -- \
  wget -T 5 -qO- http://payment-service:8080/health
```

Expected: successful response if the service, endpoint, health path and policy are configured correctly.

### 189.51.3. Test denied service

```
kubectl exec -n ecommerce order-network-test -- \
  wget -T 5 -qO- http://unapproved-service:8080/health
```

Expected: failure if the destination is reachable in the absence of policy restrictions but is not permitted by the effective policy.

A failed request alone is not sufficient proof of policy enforcement. Correlate the test with the CNI's policy diagnostics or a controlled baseline test.

### 189.51.4. Test DNS

```
kubectl exec -n ecommerce order-network-test -- \
  nslookup payment-service.ecommerce.svc.cluster.local
```

### 189.51.5. Clean up

```
kubectl delete pod order-network-test -n ecommerce
```

## 189.52. Connectivity test matrix

`tests/connectivity-matrix.yaml`

```
apiVersion: v1
kind: ConfigMap

metadata:
  name: network-policy-test-matrix
  namespace: ecommerce

data:
  matrix.yaml: |
    tests:

      - name: order-to-payment
        source: order-service
        destination: payment-service
        port: 8080
        protocol: TCP
        expected: allow

      - name: order-to-inventory
        source: order-service
        destination: inventory-service
        port: 8080
        protocol: TCP
        expected: allow

      - name: order-to-unapproved-service
        source: order-service
        destination: unapproved-service
        port: 8080
        protocol: TCP
        expected: deny

      - name: application-dns
        source: order-service
        destination: coredns
        port: 53
        protocol: UDP
        expected: allow

      - name: prometheus-to-order-metrics
        source: prometheus
        destination: order-service
        port: 8081
        protocol: TCP
        expected: allow
```

This ConfigMap is a test specification. It does not execute tests by itself.

A test runner or CI job must read the matrix and execute the checks.

## 189.53. Production rollout procedure

We should introduce NetworkPolicy in controlled stages.

### Stage 1 — Baseline discovery

Collect existing traffic requirements.

Do not immediately isolate all workloads.

### Stage 2 — DEV deployment

Deploy default-deny policies and the approved allow rules.

Verify:

- DNS.
- Ingress.
- Service-to-service calls.
- Database access.
- Messaging.
- Monitoring.
- Logging.

### Stage 3 — QA deployment

Run integration and negative connectivity tests.

### Stage 4 — STAGING deployment

Run production-like traffic simulations.

Verify that health checks and monitoring remain operational.

### Stage 5 — Production change approval

Require:

- Platform approval.
- Application-owner approval.
- Security approval.
- Rollback procedure.
- Test evidence.

### Stage 6 — Production deployment

Deploy to a controlled workload group or namespace first.

Observe application behavior before expanding.

### Stage 7 — Post-deployment verification

Monitor:

- HTTP 5xx rate.
- Application latency.
- Pod readiness.
- Database connectivity.
- Payment failures.
- Prometheus target health.
- Log delivery.
- Network-related application errors.

## 189.54. GitHub Actions production validation workflow

The CI pipeline should validate the chart and rendered manifests.

`.github/workflows/network-policy-validation.yaml`

```
name: NetworkPolicy Validation

on:
  pull_request:
    paths:
      - "network-policies/**"
      - ".github/workflows/network-policy-validation.yaml"

  push:
    branches:
      - main
    paths:
      - "network-policies/**"

permissions:
  contents: read

jobs:
  validate:
    name: Validate NetworkPolicy Templates

    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Install Helm
        uses: azure/setup-helm@v4
        with:
          version: v3.16.4

      - name: Helm lint
        run: |
          helm lint \
            network-policies/charts/network-policy

      - name: Render DEV
        run: |
          helm template ecommerce-network-policy \
            network-policies/charts/network-policy \
            -f network-policies/environments/dev/values.yaml \
            > rendered-dev.yaml

      - name: Render QA
        run: |
          helm template ecommerce-network-policy \
            network-policies/charts/network-policy \
            -f network-policies/environments/qa/values.yaml \
            > rendered-qa.yaml

      - name: Render STAGING
        run: |
          helm template ecommerce-network-policy \
            network-policies/charts/network-policy \
            -f network-policies/environments/staging/values.yaml \
            > rendered-staging.yaml

      - name: Render PRODUCTION
        run: |
          helm template ecommerce-network-policy \
            network-policies/charts/network-policy \
            -f network-policies/environments/production/values.yaml \
            > rendered-production.yaml

      - name: Verify rendered files
        run: |
          for file in rendered-*.yaml; do
            test -s "$file"
            grep -q "kind: NetworkPolicy" "$file"
          done

      - name: Upload rendered manifests
        uses: actions/upload-artifact@v4
        with:
          name: rendered-network-policies
          path: rendered-*.yaml
```

This is an illustrative validation workflow.

For production:

- Pin third-party actions to reviewed commit SHAs.
- Pin and review tool versions.
- Add Kubernetes schema validation.
- Add security policy checks.
- Add tests for unexpected broad selectors and CIDRs.
- Add required reviews for production configuration.
- Do not give an untrusted pull-request workflow production cluster credentials.

## 189.55. GitOps promotion

The platform team merges an approved policy change into the GitOps repository.

Example:

```
company-ecommerce-gitops/
|
+-- applications/
|   |
|   +-- network-policy-dev.yaml
|   +-- network-policy-qa.yaml
|   +-- network-policy-staging.yaml
|   +-- network-policy-production.yaml
|
+-- network-policies/
    |
    +-- dev/
    |   +-- values.yaml
    |
    +-- qa/
    |   +-- values.yaml
    |
    +-- staging/
    |   +-- values.yaml
    |
    +-- production/
        +-- values.yaml
```

Example ArgoCD Application:

```
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:
  name: network-policy-dev
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/company-ecommerce/company-ecommerce-platform.git

    targetRevision: main

    path: network-policies/charts/network-policy

    helm:
      valueFiles:
        - ../../environments/dev/values.yaml

  destination:
    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

The chart path and values-file path must be verified against the actual repository structure. For ArgoCD, Helm values files normally need to be accessible through the configured source or an explicitly configured multi-source arrangement. Do not assume arbitrary relative paths outside the chart source will work.

For production, use a separate approved Application configuration and a controlled promotion process.

## 189.56. Rollback procedure

If a policy deployment blocks a critical application flow, the team should restore the last known-good configuration through GitOps.

### Step 1 — Identify the affected policy

```
kubectl get networkpolicy -n ecommerce
```

### Step 2 — Inspect the policy

```
kubectl describe networkpolicy \
  allow-order-service-to-payment-service \
  -n ecommerce
```

### Step 3 — Review the recent Git change

Identify the commit that introduced the policy.

### Step 4 — Revert the change

```
git revert <commit-sha>
```

### Step 5 — Submit and approve the rollback PR

Follow the repository's emergency change process.

### Step 6 — Allow ArgoCD to reconcile

```
argocd app get network-policy-production
```

### Step 7 — Verify connectivity

Retest the previously failing connection.

### Step 8 — Verify application recovery

Check:

- Application readiness.
- HTTP 5xx.
- Request latency.
- Business transaction success.
- Monitoring targets.
- Logging.

Do not delete all NetworkPolicies as a generic emergency fix. That may remove protections unrelated to the incident.

## 189.57. Production incident simulation

Incident: Checkout requests begin failing after NetworkPolicy deployment.

### Customer impact

Customers can add products to the cart, but checkout fails.

Example error:

```
Order Service

PaymentClientException:
Unable to connect to payment-service:8080

Connection timed out
```

### Investigation

Check application logs:

```
kubectl logs deployment/order-service \
  -n ecommerce \
  --tail=100
```

Check Payment Service:

```
kubectl get pods -n ecommerce \
  -l app.kubernetes.io/name=payment-service
```

Check the Order Service egress policy:

```
kubectl get networkpolicy -n ecommerce

kubectl describe networkpolicy \
  allow-order-service-to-payment-service \
  -n ecommerce
```

Check Payment Service ingress policy:

```
kubectl describe networkpolicy \
  allow-order-service-to-payment-service-ingress \
  -n ecommerce
```

The example resource names should be adjusted to match the rendered chart.

### Root cause

The source egress permission was deployed, but the destination ingress permission was missing.

### Resolution

Add the approved Order Service source selector to Payment Service ingress.

Validate the generated manifest.

Deploy through GitOps.

Retest the checkout workflow.

### Preventive action

Add an automated connectivity test that checks both directions of the intended connection and validates the actual policy behavior.

## 189.58. Production runbook

`runbooks/networkpolicy-connectivity-failure.md`

```
# NetworkPolicy Connectivity Failure

## Purpose

Troubleshoot application connectivity problems after
NetworkPolicy deployment.

## Symptoms

- Connection timeout
- Connection refused
- DNS resolution failure
- HTTP 503
- Application dependency unavailable
- Prometheus target DOWN
- Log delivery failure

## Step 1: Identify affected workload

kubectl get pods -n ecommerce -o wide

## Step 2: Check service endpoints

kubectl get services -n ecommerce

kubectl get endpointslices -n ecommerce

## Step 3: Check NetworkPolicies

kubectl get networkpolicy -n ecommerce

## Step 4: Inspect relevant policy

kubectl describe networkpolicy <policy-name> \
  -n ecommerce

## Step 5: Verify labels

kubectl get pods -n ecommerce --show-labels

## Step 6: Verify DNS

nslookup <service-name>.ecommerce.svc.cluster.local

## Step 7: Verify the destination port

Inspect the Kubernetes Service and target pod configuration.

## Step 8: Test approved connectivity

Use a test workload with the same relevant labels and policy selection.

## Step 9: Check CNI enforcement

Inspect the installed CNI components and supported policy diagnostics.

## Step 10: Escalate

Contact the platform team, application owner and security team
when the cause cannot be confirmed.

## Rollback

Revert the approved Git change and allow GitOps to reconcile.
Do not remove unrelated security policies.
```

## 189.59. Security governance checklist

| Control             | Requirement                         |
| ------------------- | ----------------------------------- |
| Default deny        | Required for protected namespaces   |
| Service selectors   | Specific workload labels            |
| Namespace selectors | Specific namespace labels           |
| Database CIDRs      | Approved and reviewed               |
| External access     | Explicit business justification     |
| Monitoring          | Restricted to monitoring workloads  |
| Logging             | Required forwarding path preserved  |
| Production changes  | Reviewed and approved               |
| Exceptions          | Owner and review date               |
| Testing             | Positive and negative connectivity  |
| Rollback            | Documented                          |
| Audit               | Git history and deployment evidence |

## 189.60. Final definition of done

Chapter acceptance criteria

- EKS CNI NetworkPolicy enforcement verified.
- Network architecture documented.
- All 21 microservices inventoried.
- Application dependency matrix reviewed.
- Standard pod labels implemented.
- Namespace labels implemented.
- Default-deny ingress and egress templates implemented.
- DNS policy implemented and tested.
- Service ingress and egress templates implemented.
- Database policy template implemented.
- Messaging policy template implemented where applicable.
- External egress design reviewed.
- Ingress-controller access restricted.
- Prometheus scraping preserved.
- Node Exporter and kube-state-metrics connectivity checked.
- Logstash forwarding preserved.
- DEV, QA, STAGING and PRODUCTION values created.
- Helm chart validation configured.
- Kubernetes schema validation configured.
- GitHub Actions workflow implemented.
- GitOps deployment configured.
- Positive connectivity tests passed.
- Negative connectivity tests passed.
- Production rollout approved.
- Rollback procedure documented.
- Incident runbook created.
- Security exceptions governed.

## 189.61. What the platform team delivers

At the end of this chapter, the platform team should deliver:

```
NETWORK POLICY PLATFORM
|
+-- Reusable Helm Chart
|
+-- Namespace Isolation
|
+-- Default-Deny Policies
|
+-- DNS Policies
|
+-- Service Communication Policies
|
+-- Database Policies
|
+-- Messaging Policies
|
+-- Monitoring Policies
|
+-- Logging Policies
|
+-- External Egress Policies
|
+-- Environment Configurations
|
+-- CI Validation
|
+-- GitOps Deployment
|
+-- Connectivity Test Framework
|
+-- Production Runbooks
|
+-- Security Governance
```

Final principle: Network access must be intentional, documented, tested and reviewed. A successful deployment is not enough; we must also prove that approved traffic works and unapproved traffic is blocked.

Next chapter — Chapter 190: ServiceAccount Templates

We will build reusable Kubernetes ServiceAccounts, Roles, ClusterRoles, RoleBindings, least-privilege RBAC, EKS IAM Roles for Service Accounts (IRSA), workload identity, and secure access to AWS services.

Length note: This is a consolidated chapter continuation, not a verified 2,000-line file. I won't claim the 2,000-line target has been met without an actual line-count check.