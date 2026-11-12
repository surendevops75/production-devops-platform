# 140 — DNS FAILOVER

## 1. Objective

In this chapter, we will implement **production-grade DNS failover** for the e-commerce platform.

The objective is not simply to change a DNS record.

The real objective is:

> **When the primary AWS region becomes unavailable, safely redirect customer traffic to the DR region while protecting data consistency and validating that the DR environment can actually serve business traffic.**

We will cover:

* Route 53 architecture
* Public and private DNS
* Route 53 health checks
* Primary/secondary failover routing
* ALB health checks
* EKS regional failover
* CloudFront considerations
* RDS regional recovery
* DNS TTL
* DNS propagation
* Manual vs automated failover
* Failover and rollback
* RTO/RPO
* DNS failover game day

---

# 2. Production Architecture

Our primary region:

```text
AWS Primary Region
ap-south-1
```

DR region:

```text
AWS DR Region
ap-southeast-1
```

Example architecture:

```text
                         Internet
                            |
                            |
                    Route 53 DNS
                            |
                  api.example.com
                            |
                 Failover Routing
                    /            \
                   /              \
              PRIMARY           SECONDARY
                 |                  |
                 v                  v
          Primary ALB           DR ALB
          ap-south-1         ap-southeast-1
                 |                  |
                 v                  v
             EKS PROD             EKS DR
                 |                  |
        21 Microservices     21 Microservices
                 |                  |
          Primary Data         DR Data
```

The important point is:

```text
DNS failover
     !=
Data replication
```

Route 53 can redirect traffic, but it does **not** replicate:

* PostgreSQL
* Redis
* Kafka
* S3
* secrets
* container images
* application state

Those must have their own DR strategy.

---

# 3. Route 53 Responsibilities

Route 53 can provide:

### DNS hosting

```text
example.com
api.example.com
www.example.com
```

### Health checks

```text
Is primary endpoint healthy?
```

### Routing policies

For this project we will primarily use:

```text
Failover routing
```

Other policies include:

```text
Simple
Weighted
Latency-based
Failover
Geolocation
Geoproximity
Multivalue
```

For regional disaster recovery:

```text
PRIMARY
SECONDARY
```

is the important model.

---

# 4. Public DNS Architecture

Example:

```text
api.example.com
```

Route 53 contains two failover records.

```text
api.example.com
       |
       +---- PRIMARY
       |       |
       |       v
       |   Primary ALB
       |   ap-south-1
       |
       +---- SECONDARY
               |
               v
           DR ALB
        ap-southeast-1
```

Normally:

```text
Client
  |
  v
Route 53
  |
  v
Primary ALB
```

During regional failure:

```text
Client
  |
  v
Route 53
  |
  v
DR ALB
```

---

# 5. Public vs Private DNS

We need to distinguish three DNS layers.

## Public DNS

Used by customers:

```text
api.example.com
www.example.com
checkout.example.com
```

Managed through:

```text
Route 53 Public Hosted Zone
```

---

## Kubernetes DNS

Used inside EKS:

```text
order-service
inventory-service
payment-service
```

Example:

```text
http://inventory-service:8080
```

or:

```text
inventory-service.ecommerce.svc.cluster.local
```

This is handled by:

```text
CoreDNS
```

not Route 53 directly.

---

## Private AWS DNS

Used for internal AWS resources:

```text
RDS
internal ALB
ElastiCache
private endpoints
```

We can use:

```text
Route 53 Private Hosted Zones
```

associated with the appropriate VPCs.

---

# 6. ALB Health Checks vs Route 53 Health Checks

These are different.

## ALB health check

ALB determines whether individual Kubernetes targets are healthy.

```text
ALB
 |
 +-- Pod A -> healthy
 |
 +-- Pod B -> healthy
 |
 +-- Pod C -> unhealthy
```

The ALB removes unhealthy targets from service.

---

## Route 53 health check

Route 53 determines whether the **regional endpoint** should receive DNS traffic.

```text
Route 53
 |
 +-- Primary ALB -> healthy
 |
 +-- DR ALB      -> standby
```

If primary becomes unhealthy:

```text
Route 53
 |
 +-- Primary ALB -> unhealthy
 |
 +-- DR ALB      -> healthy
```

Traffic moves to DR.

Therefore:

```text
ALB health check
        |
        v
Target health inside region


Route 53 health check
        |
        v
Regional endpoint health
```

---

# 7. Dedicated Health Endpoint

Do not blindly expose:

```text
/actuator/health
```

to the public internet just for DNS checking.

Instead, we can create a dedicated endpoint such as:

```text
/health/route53
```

The endpoint should represent the minimum condition required for the region to safely receive traffic.

For example:

```text
GET /health/route53
```

Response:

```http
HTTP/1.1 200 OK
```

when the service is operational.

---

# 8. Health Check Philosophy

There are two extremes.

### Too shallow

```text
ALB is reachable
```

but:

```text
Database is unavailable
```

The DNS check thinks the region is healthy.

---

### Too deep

Health endpoint checks:

```text
Database
Redis
Kafka
SQS
Payment provider
Inventory
every dependency
```

This can create another problem.

A temporary dependency failure could cause the entire region to disappear from DNS.

Therefore the health check should represent:

> **Can this regional endpoint safely serve customer traffic?**

Not:

> "Is every component in the entire region perfect?"

---

# 9. Example Health Endpoint

Conceptually:

```text
GET /health/route53
```

Checks:

```text
Application process
        |
        +-- Database connectivity
        |
        +-- critical application dependency
```

Avoid making it dependent on every non-critical component.

For example:

```text
Notification service unavailable
```

should not necessarily make:

```text
api.example.com
```

fail DNS health.

---

# 10. Route 53 Health Check Types

Route 53 health checks can be based on:

```text
HTTP
HTTPS
TCP
```

Health checks can also be combined using calculated health checks.

For example:

```text
Primary Region
      |
      +---- ALB health
      |
      +---- Application health
      |
      +---- CloudWatch alarm
```

and derive an overall health state.

---

# 11. Important Private Endpoint Limitation

A public Route 53 health checker cannot simply connect to an arbitrary private-only endpoint inside your VPC.

For example:

```text
Internal ALB
10.x.x.x
```

is not directly reachable from public Route 53 health-check infrastructure.

For private health signals, use an appropriate design such as:

```text
CloudWatch alarm
       |
       v
Calculated health / failover mechanism
```

or another externally reachable synthetic endpoint.

This needs to be designed deliberately.

---

# 12. Primary and Secondary Regions

Our DR architecture:

```text
                     Route 53
                         |
                 Failover Routing
                    /          \
                   /            \
                  v              v
          ap-south-1       ap-southeast-1
             PRIMARY            DR
                |                |
              ALB              ALB
                |                |
              EKS              EKS
                |                |
              Apps             Apps
```

Primary:

```text
ap-south-1
```

Secondary:

```text
ap-southeast-1
```

The actual DR region can be different depending on:

* business requirements
* compliance
* latency
* AWS service availability
* cost
* data residency

---

# 13. Route 53 Terraform

Conceptual primary health check:

```hcl
resource "aws_route53_health_check" "primary" {
  fqdn              = "api-primary.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health/route53"
  request_interval  = 30
  failure_threshold = 3

  tags = {
    Name = "ecommerce-primary-health"
  }
}
```

Secondary:

```hcl
resource "aws_route53_health_check" "secondary" {
  fqdn              = "api-dr.example.com"
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health/route53"
  request_interval  = 30
  failure_threshold = 3

  tags = {
    Name = "ecommerce-dr-health"
  }
}
```

The actual endpoints should be designed so that the health checker can reach them securely.

---

# 14. Primary Failover Record

Conceptual Terraform:

```hcl
resource "aws_route53_record" "api_primary" {
  zone_id = var.zone_id
  name    = "api.example.com"
  type    = "A"

  set_identifier = "primary"

  failover_routing_policy {
    type = "PRIMARY"
  }

  alias {
    name                   = var.primary_alb_dns
    zone_id                = var.primary_alb_zone_id
    evaluate_target_health = true
  }

  health_check_id = aws_route53_health_check.primary.id
}
```

---

# 15. Secondary Failover Record

```hcl
resource "aws_route53_record" "api_secondary" {
  zone_id = var.zone_id
  name    = "api.example.com"
  type    = "A"

  set_identifier = "secondary"

  failover_routing_policy {
    type = "SECONDARY"
  }

  alias {
    name                   = var.dr_alb_dns
    zone_id                = var.dr_alb_zone_id
    evaluate_target_health = true
  }

  health_check_id = aws_route53_health_check.secondary.id
}
```

The exact combination of custom health checks and `evaluate_target_health` should be validated against the chosen Route 53 record design and current Terraform AWS provider behavior.

Do not blindly copy a production DNS configuration without testing it.

---

# 16. Normal Traffic

Normally:

```text
Customer
   |
   | HTTPS
   v
api.example.com
   |
   v
Route 53
   |
   | PRIMARY healthy
   v
Primary ALB
   |
   v
EKS ap-south-1
```

DR remains ready but does not normally receive customer traffic.

---

# 17. Regional Failure

Suppose:

```text
ap-south-1
```

experiences a major outage.

Potential symptoms:

```text
ALB unavailable
EKS API unavailable
Nodes unavailable
Pods unavailable
NAT problems
AZ problems
regional AWS service problems
```

Route 53 health evaluation becomes:

```text
PRIMARY = unhealthy
SECONDARY = healthy
```

Traffic becomes:

```text
Customer
   |
   v
Route 53
   |
   | PRIMARY unhealthy
   |
   | SECONDARY
   v
DR ALB
   |
   v
EKS ap-southeast-1
```

---

# 18. DNS Failover Is Not Instantaneous

This is extremely important.

Suppose:

```text
TTL = 60 seconds
```

That does **not** mean:

```text
Failure at 10:00:00
Everything switches exactly at 10:01:00
```

Real-world behavior depends on:

* Route 53 health detection
* recursive DNS resolvers
* client-side DNS caching
* operating-system caching
* browser/application caching
* network behavior
* TTL
* health-check intervals

Therefore:

```text
DNS TTL ≠ guaranteed failover time
```

---

# 19. TTL Strategy

For important production endpoints, an example could be:

```text
TTL = 30–60 seconds
```

depending on architecture.

But lowering TTL has trade-offs.

Lower TTL:

```text
faster potential DNS changes
```

but potentially:

```text
more DNS queries
```

Higher TTL:

```text
less DNS churn
```

but potentially:

```text
slower client-side convergence
```

Use measured values based on your RTO requirement.

---

# 20. CloudFront Changes the Architecture

If CloudFront is directly in front of the application:

```text
Customer
   |
   v
Route 53
   |
   v
CloudFront
   |
   v
ALB
```

Route 53 normally points:

```text
api.example.com
       |
       v
CloudFront distribution
```

not directly to the regional ALB.

Therefore regional failover can instead be handled through:

```text
CloudFront
   |
   +---- Primary origin
   |
   +---- DR origin
```

CloudFront origin failover/origin groups can be used where appropriate.

---

# 21. Two Possible Public Architectures

## Architecture A — Route 53 Regional Failover

```text
Route 53
   |
   +---- Primary ALB
   |
   +---- DR ALB
```

Useful when regional ALBs are directly exposed as the public application endpoints.

---

## Architecture B — CloudFront Origin Failover

```text
Route 53
   |
   v
CloudFront
   |
   +---- Primary ALB
   |
   +---- DR ALB
```

Here CloudFront manages origin failover behavior.

The correct architecture depends on:

* caching requirements
* API behavior
* TLS
* WAF
* origin architecture
* regional failover requirements

---

# 22. DNS Failover Does Not Recover RDS

This is one of the most important DR concepts.

Suppose:

```text
Primary EKS
    |
    v
Primary RDS
```

fails.

Changing DNS:

```text
api.example.com
       |
       v
DR EKS
```

does not magically create:

```text
DR PostgreSQL
```

We need an independent database DR strategy.

For example:

```text
Primary RDS
     |
     | replication / backup strategy
     v
DR RDS
```

Then:

```text
DR EKS
   |
   v
DR RDS
```

---

# 23. Application Configuration During DR

Primary:

```text
DB endpoint:
primary-db.xxxxxx.ap-south-1.rds.amazonaws.com
```

DR:

```text
DB endpoint:
dr-db.xxxxxx.ap-southeast-1.rds.amazonaws.com
```

Do not hard-code these into images.

Use:

```text
Secrets Manager
      |
      v
External Secrets Operator
      |
      v
Kubernetes Secret
      |
      v
Spring Boot
```

This allows DR configuration to be managed independently.

---

# 24. ECR During Regional Failover

DR EKS needs application images.

Therefore:

```text
Primary ECR
      |
      | replication
      v
DR ECR
```

or another controlled artifact replication strategy.

The DR GitOps configuration should reference the appropriate DR artifact location if required by the architecture.

The goal remains:

> Use the same tested immutable application artifact.

---

# 25. S3 During Regional DR

For critical objects:

```text
Primary S3
    |
    | replication
    v
DR S3
```

Examples:

```text
invoice PDFs
product images
reports
exports
documents
```

DNS failover does not copy those objects.

---

# 26. Secrets During Regional DR

Secrets also need regional availability.

Example:

```text
Primary Secrets Manager
          |
          |
      DR strategy
          |
          v
DR Secrets Manager
```

KMS must also be available.

Remember:

```text
Secret
  |
  v
KMS
```

can fail even if the secret exists if:

```text
KMS key disabled
IAM denied
key unavailable
wrong region
```

---

# 27. Kafka During Regional DR

If Kafka is part of the architecture:

```text
Primary Kafka
       |
       | replication / DR strategy
       v
DR Kafka
```

You must determine:

```text
What events must be replicated?
What offsets must be preserved?
Can events be replayed?
What is acceptable data loss?
```

Kafka DR is not solved by DNS.

---

# 28. SQS During DR

For SQS:

```text
Primary SQS
```

and:

```text
DR SQS
```

are separate regional resources.

Application design should define:

```text
Can messages be recreated?
Can events be replayed?
Are DLQs recoverable?
What happens to messages in primary during regional outage?
```

For critical workflows, the DR plan must explicitly define message recovery.

---

# 29. DNS Failover Sequence

Production failover should follow a controlled sequence.

```text
1. Detect incident
       |
       v
2. Confirm regional impact
       |
       v
3. Declare DR event
       |
       v
4. Freeze risky changes
       |
       v
5. Validate DR infrastructure
       |
       v
6. Validate DR database/data
       |
       v
7. Validate secrets/KMS
       |
       v
8. Validate ECR/images
       |
       v
9. Validate EKS
       |
       v
10. Activate DR application
       |
       v
11. Perform DNS failover
       |
       v
12. Monitor traffic
       |
       v
13. Business validation
       |
       v
14. Continue incident management
```

---

# 30. Do Not Fail Over Too Early

A common mistake is:

```text
One service is down
      |
      v
Immediately fail over entire region
```

That can make the incident worse.

First determine:

```text
Isolated service failure?
AZ failure?
ALB issue?
EKS issue?
Networking issue?
AWS regional issue?
Database issue?
Application deployment issue?
```

Use:

```text
Prometheus
Grafana
ELK
Jaeger
AWS health/events
Route 53 health
ALB metrics
RDS metrics
```

before declaring regional failure.

---

# 31. Automated vs Manual Failover

## Automated

Route 53 automatically detects health state.

Advantages:

```text
Fast response
Less manual intervention
```

Risks:

```text
False positives
Unexpected failover
Data consistency problems
Application-level partial failures
```

---

## Manual

Incident commander approves:

```text
FAILOVER TO DR
```

and the DNS state is changed.

Advantages:

```text
Human validation
Better control for stateful systems
```

Disadvantage:

```text
Slower
```

For many critical systems, a **controlled automated health mechanism plus explicit operational failover criteria** is safer than allowing a single shallow health check to trigger a complete regional disaster recovery event.

---

# 32. Failover Decision Matrix

Example:

| Situation                         | DNS Failover                    |
| --------------------------------- | ------------------------------- |
| Single pod failure                | No                              |
| Single service failure            | No                              |
| Single AZ failure                 | Usually no, if capacity remains |
| ALB target failures               | No, investigate                 |
| EKS cluster failure               | Potentially                     |
| Major regional outage             | Yes                             |
| Primary application unrecoverable | Potentially                     |
| Primary RDS logical corruption    | Not by itself                   |
| Security compromise               | Depends on incident             |
| Planned DR test                   | Yes                             |

The exact decision criteria should be defined in the organization's DR runbook.

---

# 33. DNS Failover and Security Incidents

Suppose the primary environment is compromised.

Failover alone is not enough.

Bad:

```text
Compromised application
        |
        v
DNS failover
        |
        v
DR using same compromised artifact/credential
```

Instead:

```text
Contain primary
      |
      v
Validate DR security state
      |
      v
Rotate credentials
      |
      v
Validate trusted images
      |
      v
Validate IAM/KMS
      |
      v
Activate DR
```

---

# 34. GitOps and DNS

DNS configuration should normally be managed through:

```text
Git
 |
 v
Terraform
 |
 v
Route 53
```

Example:

```text
infrastructure/
└── terraform/
    └── modules/
        └── route53/
```

Possible structure:

```text
modules/
└── route53-failover/

environments/
├── dev/
├── qa/
├── prod/
└── dr/
```

---

# 35. Emergency DNS Change

During a major incident, an emergency change may be required.

The operational process should be:

```text
Incident Commander
       |
       v
Change approval
       |
       v
DNS failover
       |
       v
Validation
       |
       v
Incident timeline
```

The emergency change should still be recorded.

Do not allow:

```text
Engineer changes DNS
        |
        v
Nobody knows what changed
```

---

# 36. DNS Rollback

Suppose DR is active:

```text
Customer
   |
   v
DR
```

Primary becomes healthy again.

Do **not** immediately switch traffic back.

First validate:

```text
Primary EKS
Primary database
Primary Redis
Primary messaging
Primary secrets
Primary application
Primary integrations
```

Then:

```text
Data consistency
      |
      v
Application validation
      |
      v
Controlled traffic return
```

---

# 37. Why Immediate Failback Is Dangerous

Imagine:

```text
Primary DB
```

was behind DR by:

```text
30 minutes
```

If you immediately route customers back:

```text
Customer
   |
   v
Primary
   |
   v
Old database state
```

you could create:

```text
lost orders
duplicate orders
inventory inconsistency
payment mismatch
```

Therefore:

> **Failback is a separate recovery operation, not simply the reverse of failover.**

---

# 38. RTO

Recovery Time Objective:

> How long can the business/application remain unavailable before recovery must be completed?

Example:

```text
RTO = 30 minutes
```

This means the DR design should target recovery within approximately that business-defined window.

Actual measured RTO must come from testing.

---

# 39. RPO

Recovery Point Objective:

> How much data loss is acceptable?

Example:

```text
RPO = 5 minutes
```

That means the organization accepts up to approximately five minutes of recoverable data loss under the defined DR scenario.

Again:

```text
RPO is a business requirement.
```

It is not automatically provided by Route 53.

---

# 40. Complete DR Dependency Chain

Our regional DR dependency model:

```text
                 Route 53
                     |
                     v
                  DR ALB
                     |
                     v
                  DR EKS
                     |
       +-------------+-------------+
       |             |             |
       v             v             v
    Secrets         ECR          Config
       |
       v
      KMS

DR EKS
  |
  +----> DR RDS
  |
  +----> DR Redis
  |
  +----> DR SQS/Kafka
  |
  +----> DR S3
```

Every dependency needs a recovery strategy.

---

# 41. DNS Failover Game Day

We should test this in a dedicated DR environment.

Do not begin by shutting down production.

Scenario:

```text
Primary:
ap-south-1

DR:
ap-southeast-1
```

Test:

```text
Primary regional application failure
```

---

# 42. Game Day Step 1 — Baseline

Before failure:

```bash
dig api.example.com
```

Record:

```text
Resolved IP
TTL
Primary ALB
HTTP response
latency
```

Test:

```bash
curl -i https://api.example.com/health/route53
```

---

# 43. Game Day Step 2 — Verify DR

Check:

```bash
kubectl get nodes
kubectl get pods -A
```

in DR.

Check:

```bash
kubectl get ingress -A
kubectl get svc -A
```

Verify:

```text
ALB
EKS
21 services
Secrets
RDS
Redis
messaging
observability
```

---

# 44. Game Day Step 3 — Simulate Primary Failure

Use an approved DR test mechanism.

For example, disable the primary application endpoint in the test environment so that the Route 53 health check becomes unhealthy.

Do not perform destructive production commands simply to test DNS.

---

# 45. Game Day Step 4 — Observe Route 53

Monitor:

```text
Primary health
Secondary health
DNS resolution
ALB traffic
HTTP traffic
```

Use:

```bash
dig api.example.com
```

from multiple networks/resolvers.

---

# 46. Game Day Step 5 — Validate DR

Test:

```text
Login
Product browsing
Search
Cart
Order creation
Inventory
Payment
Notification
Invoice
```

Example:

```text
Customer
   |
   v
Login
   |
   v
Product
   |
   v
Cart
   |
   v
Order
   |
   v
Payment
   |
   v
Inventory
```

This is more important than simply seeing:

```text
HTTP 200
```

---

# 47. Game Day Step 6 — Measure RTO

Record timestamps:

```text
T0 = failure begins

T1 = failure detected

T2 = incident declared

T3 = DR activated

T4 = DNS failover

T5 = first successful DR request

T6 = business workflow validated
```

Then:

```text
Measured RTO = T6 - T0
```

Do not claim:

```text
RTO = 15 minutes
```

until the system has actually been tested.

---

# 48. Game Day Step 7 — Measure RPO

Determine:

```text
Last successfully replicated data
```

and compare with:

```text
Failure timestamp
```

Example:

```text
Failure:
10:00

Latest recoverable transaction:
09:57
```

Measured RPO:

```text
3 minutes
```

This should be recorded for the tested scenario.

---

# 49. DNS Troubleshooting Commands

Check DNS:

```bash
dig api.example.com
```

More detail:

```bash
dig +trace api.example.com
```

Query a specific resolver:

```bash
dig @1.1.1.1 api.example.com
```

Another resolver:

```bash
dig @8.8.8.8 api.example.com
```

Check HTTPS:

```bash
curl -I https://api.example.com
```

Check TLS:

```bash
openssl s_client \
  -connect api.example.com:443 \
  -servername api.example.com
```

---

# 50. DNS Failure Troubleshooting

If customers still reach primary after failover:

Check:

```text
Route 53 health status
        |
        v
Failover record configuration
        |
        v
TTL
        |
        v
Resolver caching
        |
        v
Client/application DNS cache
```

Do not immediately assume Route 53 failed.

---

# 51. If DR Receives Traffic but Returns 5xx

Investigate:

```text
Route 53
   |
   v
DR ALB
   |
   v
EKS
   |
   +---- Pods
   |
   +---- Secrets
   |
   +---- RDS
   |
   +---- Redis
   |
   +---- SQS/Kafka
```

Use:

```text
Prometheus
Grafana
ELK
Jaeger
kubectl
AWS service metrics
```

---

# 52. If DNS Does Not Fail Over

Check:

### 1. Health check

```text
Is primary actually unhealthy according to Route 53?
```

### 2. Endpoint

```text
Is the health-check endpoint reachable?
```

### 3. HTTP status

```text
200?
301?
403?
500?
```

### 4. Record type

```text
PRIMARY
SECONDARY
```

### 5. Record name

```text
api.example.com
```

### 6. Hosted zone

```text
Correct hosted zone?
```

### 7. TTL/caching

```text
Are clients still using cached DNS?
```

---

# 53. Common Production Mistakes

### Mistake 1

Only configuring Route 53.

```text
Route 53
   |
   v
DR
```

but no DR data.

---

### Mistake 2

DR EKS exists but has no secrets.

---

### Mistake 3

DR EKS exists but images are only in primary ECR.

---

### Mistake 4

DNS points to DR but DR database is stale.

---

### Mistake 5

Health check only checks ALB availability.

---

### Mistake 6

Health check checks every dependency.

This can cause unnecessary regional failovers.

---

### Mistake 7

Failback without data synchronization.

---

### Mistake 8

Never testing the DR environment.

---

### Mistake 9

Assuming:

```text
TTL 30 seconds = failover in 30 seconds
```

---

### Mistake 10

Allowing DNS changes without incident/change records.

---

# 54. Production DNS Architecture

Final architecture:

```text
                         Internet
                            |
                            v
                       Route 53
                            |
                   Failover Routing
                    /             \
                   /               \
                  v                 v
          PRIMARY REGION        DR REGION
          ap-south-1          ap-southeast-1
               |                    |
              ALB                  ALB
               |                    |
              EKS                  EKS
               |                    |
          21 Services          21 Services
               |                    |
       +-------+------+       +-----+------+
       |       |      |       |     |      |
      RDS    Redis   SQS      RDS  Redis   SQS
       |              |       |            |
       +------ DR ----+-------+------------+
```

Supporting infrastructure:

```text
ECR
S3
Secrets Manager
KMS
Terraform
GitHub
GitOps
ArgoCD
Prometheus
Grafana
ELK
OpenTelemetry
Jaeger
```

must also have an appropriate DR/recovery strategy.

---

# 55. DNS Failover Runbook

## Trigger

```text
Primary region unavailable
```

## Step 1

Confirm customer impact.

## Step 2

Check AWS regional/service status.

## Step 3

Check:

```text
ALB
EKS
RDS
Redis
SQS/Kafka
```

## Step 4

Declare DR incident if criteria are met.

## Step 5

Freeze risky production changes.

## Step 6

Validate DR.

## Step 7

Validate data.

## Step 8

Validate secrets/KMS.

## Step 9

Validate application artifacts.

## Step 10

Activate DR application if required.

## Step 11

Execute DNS failover.

## Step 12

Monitor:

```text
HTTP 5xx
latency
RPS
orders
payments
database
queues
Kafka lag
```

## Step 13

Perform business validation.

## Step 14

Measure RTO/RPO.

## Step 15

Maintain incident timeline.

## Step 16

Plan controlled failback.

---

# 56. Final Checklist

### Route 53

* [ ] Public hosted zone configured
* [ ] Primary record configured
* [ ] Secondary record configured
* [ ] Health checks configured
* [ ] TTL documented
* [ ] Failover tested
* [ ] DNS resolution tested from multiple resolvers

### Primary

* [ ] ALB healthy
* [ ] EKS healthy
* [ ] Applications healthy
* [ ] Database healthy
* [ ] Redis healthy
* [ ] Messaging healthy

### DR

* [ ] EKS available
* [ ] Nodes available
* [ ] 21 services deployable
* [ ] ECR artifacts available
* [ ] Secrets available
* [ ] KMS available
* [ ] RDS recovery available
* [ ] Redis recovery available
* [ ] SQS/Kafka strategy defined
* [ ] S3 recovery available

### Operations

* [ ] DR runbook
* [ ] Incident commander
* [ ] Change approval
* [ ] Communication process
* [ ] RTO defined
* [ ] RPO defined
* [ ] Game day performed
* [ ] RTO measured
* [ ] RPO measured
* [ ] Failback tested

---

# 57. Key Production Lessons

The most important concepts from this chapter are:

```text
Route 53
   =
Traffic steering
```

not:

```text
Route 53
   =
Disaster recovery
```

True regional DR is:

```text
DNS
 +
Compute
 +
Networking
 +
Database
 +
Cache
 +
Messaging
 +
Storage
 +
Secrets
 +
KMS
 +
Container Images
 +
Observability
 +
Runbooks
```

And the complete recovery process is:

```text
Detect
   ↓
Assess
   ↓
Declare
   ↓
Protect
   ↓
Recover DR dependencies
   ↓
Activate DR
   ↓
DNS Failover
   ↓
Validate business workflows
   ↓
Monitor
   ↓
Recover primary
   ↓
Synchronize data
   ↓
Controlled failback
```

**The goal is not simply to make `api.example.com` resolve to another ALB. The goal is to make the entire business workflow operate safely in the secondary region.**
