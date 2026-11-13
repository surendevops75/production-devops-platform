# 151 — DNS Failure Production Incident

> **Phase 18 — Incident Management**
>
> This chapter covers a production-grade **DNS failure** in the EKS e-commerce platform.
>
> We will troubleshoot DNS at multiple layers:
>
> `Application → Pod DNS → CoreDNS → Kubernetes Service DNS → AWS/VPC DNS → Route 53`
>
> The most important goal is to distinguish **DNS resolution failure** from a normal network connectivity failure.

---

## 1. What Is a DNS Failure?

DNS converts names into addresses.

For example:

```text
inventory-service
        ↓
10.100.42.20
```

or:

```text
orders-db.xxxxx.ap-south-1.rds.amazonaws.com
        ↓
10.x.x.x
```

If DNS fails, applications may report:

```text
UnknownHostException
Temporary failure in name resolution
Name or service not known
Could not resolve host
DNS lookup timeout
NXDOMAIN
SERVFAIL
```

A DNS failure can affect:

- Kubernetes service discovery
- RDS connectivity
- Redis
- Kafka
- SQS/SNS APIs
- ECR
- External APIs
- ALB/Route 53
- Monitoring
- Application startup

---

# 2. DNS Architecture in Our Platform

We have multiple DNS layers.

```text
                    Internet
                       |
                       v
                   Route 53
                       |
                       v
                 CloudFront / ALB
                       |
                       v
                     EKS
                       |
              +--------+--------+
              |                 |
              v                 v
          CoreDNS          AWS/VPC DNS
              |                 |
              v                 v
     Kubernetes Services    AWS Resources
              |                 |
              v                 |
          Pod IPs              RDS
                              Redis
                              ECR
                              AWS APIs
```

Important distinction:

```text
CoreDNS
=
Kubernetes internal service discovery
```

while:

```text
Route 53 / AWS DNS
=
AWS/public/private DNS and external name resolution
```

---

# 3. Example Kubernetes DNS

Suppose:

```text
order-service
```

needs:

```text
inventory-service
```

Application uses:

```text
http://inventory-service:8080
```

The Pod asks DNS:

```text
inventory-service
        |
        v
CoreDNS
        |
        v
Kubernetes Service
        |
        v
ClusterIP
```

The application then connects to the Service.

---

# 4. Full Kubernetes DNS Resolution

For a Pod in namespace `ecommerce`:

```text
inventory-service
        ↓
inventory-service.ecommerce
        ↓
inventory-service.ecommerce.svc
        ↓
inventory-service.ecommerce.svc.cluster.local
```

The complete service DNS name is:

```text
inventory-service.ecommerce.svc.cluster.local
```

---

# 5. DNS Failure Is a Symptom

Suppose Java reports:

```text
java.net.UnknownHostException:
inventory-service
```

Do not immediately restart CoreDNS.

Possible causes include:

```text
Wrong service name
Wrong namespace
CoreDNS unavailable
Pod cannot reach CoreDNS
NetworkPolicy blocking DNS
DNS configuration incorrect
VPC DNS issue
Node networking problem
External DNS failure
Application configuration error
```

---

# 6. Production Incident Scenario

Suppose customers report:

> "Checkout is failing."

Grafana:

```text
order-service
HTTP 5xx = 25%
```

Application logs:

```text
java.net.UnknownHostException:
inventory-service
```

Jaeger:

```text
order-service
   |
   X
inventory-service
```

Now we have strong evidence that the failure may be DNS-related.

But we still need to prove it.

---

# 7. Production Troubleshooting Flow

Use:

```text
Alert
  ↓
Confirm customer impact
  ↓
Identify hostname
  ↓
Test DNS from affected Pod
  ↓
Determine internal vs external DNS
  ↓
Check CoreDNS
  ↓
Check Pod resolv.conf
  ↓
Check Service existence
  ↓
Check Service namespace/name
  ↓
Check NetworkPolicy
  ↓
Check VPC DNS
  ↓
Check AWS resolver path
  ↓
Check recent changes
  ↓
Mitigate
  ↓
Validate
  ↓
RCA
```

---

# 8. Step 1 — Identify the Exact Hostname

First capture the actual failure.

Example:

```text
inventory-service
```

or:

```text
inventory-service.ecommerce.svc.cluster.local
```

or:

```text
orders-db.xxxxx.ap-south-1.rds.amazonaws.com
```

or:

```text
api.payment-provider.com
```

These are different DNS paths.

---

# 9. DNS Failure Categories

Classify the hostname.

### Type 1 — Kubernetes internal

```text
inventory-service
```

Usually:

```text
CoreDNS → Kubernetes API/service records
```

### Type 2 — AWS private resource

```text
orders-db.xxxxx.ap-south-1.rds.amazonaws.com
```

Usually:

```text
Pod → CoreDNS → upstream/VPC DNS → AWS resolver
```

### Type 3 — Public internet

```text
api.example.com
```

Usually:

```text
Pod → CoreDNS → upstream resolver → Internet DNS
```

### Type 4 — Public application endpoint

```text
api.example.com
```

may eventually resolve through:

```text
Route 53
CloudFront
ALB
```

---

# 10. Step 2 — Check the Affected Pod

```bash
kubectl get pods -n ecommerce -o wide
```

Identify:

```text
order-service Pod
Pod IP
Node
```

Then:

```bash
kubectl describe pod <pod-name> -n ecommerce
```

Check:

```text
Node
DNS Policy
DNS Config
Network
Events
```

---

# 11. Step 3 — Check `/etc/resolv.conf`

Inside the Pod:

```bash
kubectl exec -it <order-pod> -n ecommerce -- cat /etc/resolv.conf
```

Typical output resembles:

```text
search ecommerce.svc.cluster.local svc.cluster.local cluster.local
nameserver 10.100.0.10
options ndots:5
```

The `nameserver` is typically the Kubernetes DNS Service IP.

The exact address varies by cluster.

---

# 12. What Does `search` Mean?

Suppose:

```text
search ecommerce.svc.cluster.local
```

Application asks for:

```text
inventory-service
```

DNS resolution can try names based on the search domains.

Conceptually:

```text
inventory-service
        ↓
inventory-service.ecommerce.svc.cluster.local
```

This is why short Kubernetes service names work.

---

# 13. `ndots` Matters

You may see:

```text
options ndots:5
```

This affects how DNS treats names with dots and search-domain behavior.

It can create extra DNS queries for certain external names.

At large scale, DNS query volume can matter.

Do not change `ndots` casually in production.

---

# 14. Step 4 — Test Kubernetes DNS

From the affected Pod:

```bash
nslookup inventory-service
```

or:

```bash
getent hosts inventory-service
```

Expected:

```text
inventory-service
10.100.42.20
```

Then:

```bash
nslookup inventory-service.ecommerce.svc.cluster.local
```

---

# 15. Test a Known Service

If:

```text
inventory-service
```

fails, test another known service:

```bash
nslookup customer-service
```

Interpretation:

### All services fail

Likely:

```text
CoreDNS
Pod DNS
network path to DNS
```

### Only one service fails

Likely:

```text
Service name
namespace
Service object
```

This is a very useful scope technique.

---

# 16. Step 5 — Check the Service

```bash
kubectl get svc inventory-service -n ecommerce
```

If the result is:

```text
Error from server:
services "inventory-service" not found
```

then the problem may simply be:

```text
wrong service name
```

Check:

```bash
kubectl get svc -A | grep inventory
```

---

# 17. Namespace Error

Suppose the Service exists:

```text
ecommerce/inventory-service
```

but application is configured for:

```text
inventory-service.payments
```

DNS lookup fails because the namespace is wrong.

Correct:

```text
inventory-service.ecommerce
```

or:

```text
inventory-service.ecommerce.svc.cluster.local
```

---

# 18. Step 6 — Check CoreDNS Pods

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Depending on your cluster labels/version, inspect the actual CoreDNS Pods.

Example:

```text
coredns-abc123   1/1 Running
coredns-def456   1/1 Running
```

Check deployment:

```bash
kubectl get deployment coredns -n kube-system
```

---

# 19. Check CoreDNS Replica Count

For production, avoid depending on a single CoreDNS Pod.

Check:

```bash
kubectl get deployment coredns -n kube-system
```

Example:

```text
READY
2/2
```

If:

```text
1/2
```

investigate immediately.

If:

```text
0/2
```

Kubernetes internal DNS is likely severely impacted.

---

# 20. Check CoreDNS Logs

```bash
kubectl logs -n kube-system deployment/coredns
```

Look for:

```text
SERVFAIL
timeout
plugin errors
upstream failures
connection errors
```

If the cluster has multiple CoreDNS Pods, inspect individual Pods when necessary.

---

# 21. Don't Assume CoreDNS Is the Root Cause

Suppose:

```text
CoreDNS Pods:
2/2 Running
```

That doesn't guarantee DNS is healthy.

Possible:

```text
CoreDNS overloaded
Network path broken
Upstream DNS unavailable
Bad Corefile
Resource starvation
NetworkPolicy
Node issue
```

Continue testing.

---

# 22. Step 7 — Check CoreDNS Service

```bash
kubectl get svc kube-dns -n kube-system
```

Example:

```text
NAME       TYPE        CLUSTER-IP
kube-dns   ClusterIP   10.100.0.10
```

This ClusterIP should correspond to the nameserver seen in Pod `/etc/resolv.conf`.

---

# 23. Compare the Addresses

Pod:

```text
nameserver 10.100.0.10
```

Kubernetes:

```text
kube-dns
ClusterIP:
10.100.0.10
```

These should align.

If they don't, investigate DNS configuration.

---

# 24. Step 8 — Check CoreDNS Endpoints

```bash
kubectl get endpointslice \
  -n kube-system \
  -l kubernetes.io/service-name=kube-dns
```

You should see CoreDNS Pod endpoints.

If:

```text
kube-dns
```

has no usable endpoints, DNS traffic cannot reach CoreDNS.

---

# 25. DNS Chain

The internal flow is approximately:

```text
Application
    |
    v
/etc/resolv.conf
    |
    v
kube-dns Service
    |
    v
CoreDNS Pod
    |
    +--> Kubernetes API
    |
    +--> upstream DNS
```

A failure anywhere can produce a DNS error.

---

# 26. Step 9 — Test CoreDNS Directly

You can identify CoreDNS Pod IPs:

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=kube-dns \
  -o wide
```

Then use an approved troubleshooting Pod to test DNS behavior.

The goal is to determine:

```text
Pod → kube-dns Service
```

versus:

```text
Pod → CoreDNS endpoint
```

Do not make arbitrary production changes while testing.

---

# 27. Step 10 — Check NetworkPolicy

DNS traffic usually needs access to the Kubernetes DNS service.

List policies:

```bash
kubectl get networkpolicy -A
```

If strict policies are used, verify that application Pods can reach:

```text
kube-dns
UDP 53
TCP 53
```

Both UDP and TCP DNS behavior matter because DNS queries may use TCP in certain situations.

---

# 28. Example DNS NetworkPolicy Problem

Suppose:

```text
order-service
```

has a NetworkPolicy allowing only:

```text
inventory-service
customer-service
pricing-service
```

but not:

```text
kube-system/kube-dns
```

Then:

```text
order-service
    |
    X
DNS request
    |
kube-dns
```

Application reports:

```text
UnknownHostException
```

The actual root cause is:

```text
NetworkPolicy blocking DNS.
```

---

# 29. Step 11 — Check Node Placement

Find the affected Pod's node:

```bash
kubectl get pod <pod-name> \
  -n ecommerce \
  -o wide
```

Suppose:

```text
order-service → node-A
```

Check whether other Pods on node-A have DNS problems.

This helps identify node-specific failures.

---

# 30. Node-Specific DNS Failure

Example:

```text
Node A:
order-service → DNS fails
payment-service → DNS fails

Node B:
order-service → DNS works

Node C:
order-service → DNS works
```

This strongly suggests:

```text
Node A
```

or its network path.

Investigate:

```text
VPC CNI
node networking
iptables/eBPF
resource pressure
node-level configuration
```

---

# 31. CoreDNS Scheduling

CoreDNS should be resilient across nodes/AZs.

Check:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns -o wide
```

Ideally:

```text
CoreDNS Pod A → Node/AZ A
CoreDNS Pod B → Node/AZ B
```

This reduces single-node failure risk.

The exact topology rules should be verified in the deployed EKS configuration.

---

# 32. CoreDNS Resource Saturation

CoreDNS itself can become CPU/memory constrained.

Check:

```bash
kubectl top pods -n kube-system
```

if metrics are available.

Look for:

```text
coredns
CPU high
Memory high
```

Also inspect:

```bash
kubectl describe pod <coredns-pod> -n kube-system
```

---

# 33. DNS Query Storm

A DNS problem can create a feedback loop.

Example:

```text
Service calls dependency
       ↓
DNS query
       ↓
DNS slow
       ↓
request timeout
       ↓
application retry
       ↓
more DNS queries
       ↓
CoreDNS load increases
       ↓
DNS becomes slower
```

This is a DNS retry storm.

---

# 34. Application-Level DNS Caching

Java applications and the JVM may cache DNS results.

This means:

```text
DNS record changes
```

may not immediately affect every running process depending on JVM DNS caching behavior/configuration.

For production systems, understand:

```text
JVM DNS cache
CoreDNS cache
AWS resolver cache
client behavior
```

before relying on very short DNS TTLs.

---

# 35. Kubernetes Service DNS Stability

Kubernetes Service DNS is designed to abstract away Pod IP changes.

Use:

```text
inventory-service
```

instead of:

```text
10.0.12.31
```

because:

```text
Pod A dies
   ↓
Pod B created
   ↓
Pod IP changes
```

Service DNS remains the stable discovery mechanism.

---

# 36. Don't Hardcode Pod IPs

Bad:

```yaml
ENVIRONMENT:
  INVENTORY_URL: http://10.0.12.31:8080
```

Good:

```yaml
ENVIRONMENT:
  INVENTORY_URL: http://inventory-service:8080
```

This prevents a large class of connectivity failures.

---

# 37. External DNS Failure

Now suppose the failing hostname is:

```text
api.payment-provider.com
```

This is not a Kubernetes Service DNS record.

Test:

```bash
nslookup api.payment-provider.com
```

or:

```bash
getent hosts api.payment-provider.com
```

If it fails:

```text
Pod
 ↓
CoreDNS
 ↓
upstream resolver
 ↓
public DNS
```

Investigate the upstream path.

---

# 38. AWS/VPC DNS

AWS VPC provides DNS resolution through the Amazon-provided DNS resolver.

For a VPC such as:

```text
10.0.0.0/16
```

the resolver is typically reachable at the VPC base-plus-two address:

```text
10.0.0.2
```

However, application Pods normally use Kubernetes/CoreDNS as configured by their Pod DNS settings rather than directly using that address.

---

# 39. VPC DNS Settings

Check that the VPC has DNS support enabled.

Terraform:

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true
}
```

Without correct VPC DNS settings, AWS resource name resolution can fail.

---

# 40. Private RDS DNS

Our RDS endpoint looks conceptually like:

```text
orders-db.xxxxx.ap-south-1.rds.amazonaws.com
```

Application:

```text
order-service
      |
      v
CoreDNS
      |
      v
AWS DNS
      |
      v
RDS private IP
```

If DNS fails:

```text
order-service
      |
      X
RDS hostname
```

The application may report:

```text
UnknownHostException
```

---

# 41. DNS Failure vs RDS Failure

These are different.

### DNS failure

```text
orders-db...
       ↓
cannot resolve
```

### Network failure

```text
orders-db...
       ↓
resolves to IP
       ↓
TCP 5432 fails
```

### Database failure

```text
TCP 5432 works
       ↓
PostgreSQL authentication/query fails
```

Always identify the layer.

---

# 42. Route 53 Public DNS

Our public architecture may use:

```text
api.example.com
```

Route 53:

```text
api.example.com
       ↓
CloudFront
```

or:

```text
api.example.com
       ↓
ALB
```

depending on the chosen architecture.

During a DNS incident, verify the actual architecture before changing records.

---

# 43. Route 53 Failover

For regional DR:

```text
Route 53
    |
    +--> Primary ALB
    |
    +--> DR ALB
```

If primary fails:

```text
Route 53 health evaluation
       ↓
Primary unhealthy
       ↓
DR record selected
```

But DNS failover is not instantaneous.

Caching exists at:

```text
recursive resolver
client
application
```

Therefore:

```text
TTL ≠ guaranteed failover time
```

---

# 44. Route 53 vs CoreDNS

Very important:

### Route 53

Used for:

```text
public DNS
private hosted zones
regional failover
AWS DNS management
```

### CoreDNS

Used for:

```text
Kubernetes internal service discovery
```

Example:

```text
inventory-service
```

is normally CoreDNS.

Example:

```text
api.example.com
```

may involve Route 53.

---

# 45. CoreDNS Configuration

Inspect:

```bash
kubectl get configmap coredns \
  -n kube-system \
  -o yaml
```

You may see configuration resembling:

```text
.:53 {
    errors
    health
    ready
    kubernetes cluster.local ...
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

The exact configuration depends on the EKS/Kubernetes version and your customization.

---

# 46. What Does `forward . /etc/resolv.conf` Mean?

Conceptually:

```text
Kubernetes service query
       ↓
CoreDNS
       ↓
Kubernetes plugin
```

For names it doesn't own:

```text
external.example.com
       ↓
CoreDNS
       ↓
upstream DNS
```

The exact upstream behavior depends on the CoreDNS configuration.

---

# 47. CoreDNS Cache

CoreDNS can cache DNS responses.

This improves:

```text
latency
DNS efficiency
upstream load
```

But caching means changes may not be immediately visible depending on TTL/cache behavior.

Don't blindly flush/restart DNS as a first response.

---

# 48. SERVFAIL vs NXDOMAIN

### NXDOMAIN

Usually means:

> The requested DNS name does not exist.

Example:

```text
inventory-servic
```

instead of:

```text
inventory-service
```

### SERVFAIL

Usually means:

> The resolver failed to successfully answer the query.

Potential:

```text
upstream problem
DNSSEC issue
configuration
resolver failure
```

The exact root cause requires deeper evidence.

---

# 49. Timeout vs NXDOMAIN

These are very different.

### NXDOMAIN

```text
DNS server responded:
name does not exist
```

Investigate:

```text
hostname
namespace
DNS record
service name
```

### Timeout

```text
No usable DNS response
```

Investigate:

```text
network
CoreDNS
upstream resolver
NetworkPolicy
resource saturation
```

---

# 50. DNS + NetworkPolicy

DNS is network traffic.

Usually:

```text
UDP 53
```

and sometimes:

```text
TCP 53
```

If a NetworkPolicy allows:

```text
HTTP 8080
```

but blocks:

```text
DNS 53
```

the application may report:

```text
UnknownHostException
```

even though the actual application-to-application policy is correct.

---

# 51. DNS + VPC CNI

If Pod networking is broken:

```text
Pod
 ↓
CoreDNS
```

may fail.

Check:

```bash
kubectl get pods -n kube-system
```

Look at:

```text
aws-node
coredns
kube-proxy
```

These system components form part of the networking foundation.

---

# 52. DNS + Node Failure

Suppose CoreDNS Pod is running on:

```text
node-A
```

and node-A experiences networking failure.

CoreDNS itself may appear:

```text
Running
```

but be unreachable.

Therefore:

```text
Pod status = Running
```

does not guarantee:

```text
DNS path = healthy
```

---

# 53. DNS + CPU Saturation

If CoreDNS CPU is saturated:

```text
DNS query latency
       ↓
application latency
       ↓
request timeouts
       ↓
5xx
```

This can look like:

```text
ALB incident
```

or:

```text
service connectivity incident
```

until DNS is investigated.

---

# 54. DNS + Memory Pressure

If CoreDNS experiences memory pressure:

```text
CoreDNS
 ↓
OOMKilled
 ↓
restart
 ↓
DNS unavailable/intermittent
```

Check:

```bash
kubectl get pods -n kube-system
```

and:

```bash
kubectl describe pod <coredns-pod> -n kube-system
```

---

# 55. DNS + HPA

CoreDNS can be horizontally scaled when appropriate.

But:

```text
more CoreDNS replicas
```

does not automatically solve:

```text
bad DNS configuration
network policy
broken upstream resolver
```

Scale only after identifying the bottleneck.

---

# 56. Check CoreDNS HPA

Depending on the cluster setup:

```bash
kubectl get hpa -n kube-system
```

If CoreDNS is autoscaled, inspect:

```bash
kubectl describe hpa <coredns-hpa> -n kube-system
```

Actual EKS configuration varies by cluster version and platform choices.

---

# 57. DNS Monitoring

Monitor:

```text
CoreDNS availability
DNS query rate
DNS response latency
SERVFAIL
NXDOMAIN
CoreDNS CPU
CoreDNS memory
CoreDNS restarts
```

For critical services, also monitor:

```text
application DNS errors
```

because infrastructure metrics alone don't capture every customer impact.

---

# 58. Application DNS Metrics

For Java applications, logs may show:

```text
UnknownHostException
```

Search ELK:

```text
service.name: order-service
AND
message: "UnknownHostException"
```

Track the rate over time.

If:

```text
DNS errors = 0
```

normally and suddenly:

```text
DNS errors = 10,000/min
```

that is strong evidence of a DNS incident.

---

# 59. Distributed Tracing

Jaeger can show:

```text
order-service
    |
    +--> inventory-service
```

but DNS resolution errors may occur before a normal downstream span is created.

Therefore combine:

```text
Jaeger
+
application logs
+
CoreDNS metrics
```

rather than relying on traces alone.

---

# 60. Recent Change Investigation

Check recent changes to:

```text
CoreDNS
NetworkPolicy
VPC
EKS
CNI
Service
Ingress
Terraform
Helm
GitOps
application configuration
```

Example:

```text
10:00
NetworkPolicy deployed

10:05
DNS errors begin

10:06
UnknownHostException spikes
```

Strong correlation.

---

# 61. GitOps Investigation

Check platform GitOps:

```text
gitops/
└── platform/
    ├── coredns/
    ├── network-policy/
    └── monitoring/
```

Then:

```text
Git
 ↓
ArgoCD
 ↓
Kubernetes
```

Check:

```bash
argocd app list
```

and:

```bash
argocd app get <platform-app>
```

Identify the exact revision.

---

# 62. Example Incident — Bad NetworkPolicy

### Before

```text
order-service
     |
     +--> DNS
     |
     +--> inventory-service
```

### Change

Security team applies:

```text
default-deny
```

but forgets DNS egress.

### Result

```text
order-service
     |
     X
UDP/TCP 53
     |
CoreDNS
```

### Application

```text
UnknownHostException
```

### Customer

```text
Checkout unavailable
```

---

# 63. Immediate Mitigation

If NetworkPolicy is proven to be the cause:

```text
Restore the known-good policy
```

through:

```text
Git
 ↓
PR
 ↓
merge
 ↓
ArgoCD
```

Avoid manually deleting policies without updating GitOps unless your incident process explicitly authorizes an emergency change.

---

# 64. Example Incident — CoreDNS Failure

Suppose:

```text
CoreDNS Pods:
0/2 healthy
```

Application:

```text
UnknownHostException
```

Immediate priority:

```text
Restore CoreDNS capacity
```

Investigate:

```text
CrashLoopBackOff
OOMKilled
configuration
scheduling
node availability
```

Then validate:

```text
nslookup inventory-service
```

from affected Pods.

---

# 65. Example Incident — Wrong Service Name

Developer changes:

```text
inventory-service
```

to:

```text
inventory-svc
```

but actual Service remains:

```text
inventory-service
```

Result:

```text
NXDOMAIN
```

This is not a CoreDNS outage.

It's an application configuration error.

---

# 66. Example Incident — External DNS Failure

Payment provider changes DNS or becomes unavailable.

Application:

```text
payment-service
      |
      v
api.payment-provider.com
      X
DNS
```

Possible result:

```text
payment failures
```

But Kubernetes internal DNS may be completely healthy.

Therefore scope matters.

---

# 67. Example Incident — RDS DNS Resolution Failure

Suppose:

```text
order-service
      |
      v
orders-db.xxxxx.rds.amazonaws.com
```

fails DNS resolution.

Then:

```text
order-service
      |
      X
RDS
```

All DB queries fail.

The application may produce:

```text
HTTP 500
```

which eventually becomes:

```text
ALB Target 5xx
```

The original failure was DNS.

This illustrates incident chaining:

```text
DNS
 ↓
RDS connectivity
 ↓
Application
 ↓
ALB
 ↓
Customer
```

---

# 68. DNS Failure and Retry Storm

Bad design:

```text
DNS fails
 ↓
request retries 5 times
 ↓
each retry performs DNS lookup
 ↓
more DNS load
 ↓
DNS becomes slower
```

Better:

```text
short timeout
+
bounded retry
+
exponential backoff
+
jitter
+
circuit breaker
```

---

# 69. JVM DNS Considerations

Our services use:

```text
Java 21
Spring Boot
```

Java applications can cache DNS results.

Therefore during a DNS/endpoint change, understand:

```text
JVM DNS cache
```

before expecting every running JVM to immediately observe changes.

For Kubernetes Service discovery, applications should generally use stable Service DNS names rather than Pod IPs.

---

# 70. DNS and Service Discovery Best Practices

Use:

```text
http://inventory-service:8080
```

instead of:

```text
http://10.0.12.31:8080
```

Use:

```text
orders-db.xxxxx.rds.amazonaws.com
```

instead of hardcoding a private IP.

Use:

```text
api.example.com
```

instead of hardcoding an ALB IP.

DNS provides abstraction from changing infrastructure.

---

# 71. Production DNS Dependency Map

```text
                         DNS
                          |
        +-----------------+-----------------+
        |                 |                 |
        v                 v                 v
   Kubernetes          AWS/VPC           Public DNS
    Services             DNS             Route 53
        |                 |                 |
        v                 v                 v
    CoreDNS             RDS              CloudFront
    Services            Redis               ALB
    Pods                ECR
                        SQS
                        SNS
```

---

# 72. DNS Runbook

Create:

```text
runbooks/dns-failure.md
```

Recommended structure:

```markdown
# DNS Failure Runbook

## Symptoms

## Customer Impact

## Identify Failing Hostname

## Internal or External DNS?

## Check /etc/resolv.conf

## Test DNS

## Check Kubernetes Service

## Check CoreDNS

## Check kube-dns Service

## Check EndpointSlices

## Check NetworkPolicy

## Check VPC DNS

## Check VPC CNI

## Check AWS Resolver Path

## Check Route 53

## Check Recent Changes

## Mitigation

## Validation

## RCA
```

---

# 73. DNS Production Checklist

### Kubernetes

- [ ] CoreDNS has sufficient replicas
- [ ] CoreDNS distributed across failure domains
- [ ] kube-dns Service healthy
- [ ] CoreDNS endpoints healthy
- [ ] DNS metrics monitored
- [ ] DNS logs available
- [ ] NetworkPolicy allows DNS
- [ ] Pod DNS configuration correct

### AWS

- [ ] VPC DNS support enabled
- [ ] VPC DNS hostnames enabled
- [ ] Private DNS configuration tested
- [ ] Required VPC endpoints configured
- [ ] Route 53 records monitored
- [ ] Health checks tested
- [ ] DR DNS failover tested

### Applications

- [ ] Use Service DNS names
- [ ] Avoid hardcoded Pod IPs
- [ ] Timeouts configured
- [ ] Retry limits configured
- [ ] DNS errors monitored
- [ ] External DNS dependencies documented

---

# 74. Production Commands Cheat Sheet

### Pod DNS

```bash
kubectl exec -it <pod> -n ecommerce -- cat /etc/resolv.conf
```

```bash
kubectl exec -it <pod> -n ecommerce -- nslookup inventory-service
```

```bash
kubectl exec -it <pod> -n ecommerce -- getent hosts inventory-service
```

---

### CoreDNS

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

```bash
kubectl get deployment coredns -n kube-system
```

```bash
kubectl get svc kube-dns -n kube-system
```

```bash
kubectl get endpointslice \
  -n kube-system \
  -l kubernetes.io/service-name=kube-dns
```

```bash
kubectl logs -n kube-system deployment/coredns
```

---

### Service

```bash
kubectl get svc -A
```

```bash
kubectl describe svc inventory-service -n ecommerce
```

---

### NetworkPolicy

```bash
kubectl get networkpolicy -A
```

```bash
kubectl describe networkpolicy <policy> -n ecommerce
```

---

### VPC CNI

```bash
kubectl get daemonset aws-node -n kube-system
```

```bash
kubectl get pods -n kube-system -o wide
```

---

### Route 53

For DNS records:

```bash
aws route53 list-hosted-zones
```

Then inspect the relevant hosted zone/record according to your environment.

---

# 75. DNS Decision Tree

```text
DNS Failure
    |
    v
What hostname?
    |
    +------------------+
    |                  |
    v                  v
Kubernetes          External/AWS
Service             hostname
    |                  |
    v                  v
Check Service       Check resolver
    |                  |
    v                  v
Check CoreDNS       Check upstream
    |                  |
    +--------+---------+
             |
             v
       /etc/resolv.conf
             |
             v
        DNS query
             |
       +-----+-----+
       |           |
      Works       Fails
       |           |
       v           v
    Network      CoreDNS/
    /HTTP        resolver
       |           |
       v           v
    Continue     Investigate
```

---

# 76. Advanced Decision Tree

```text
UnknownHostException
        |
        v
nslookup hostname
        |
   +----+----+
   |         |
 Works      Fails
   |         |
   v         v
DNS OK     Which name?
   |         |
   |     +---+---+
   |     |       |
   |   Internal External
   |     |       |
   |     v       v
   |   Service  Upstream
   |   CoreDNS  DNS
   |   Policy   Route53/VPC
   |
   v
Test TCP
   |
   +--> refused
   |      |
   |      v
   |   port/listener
   |
   +--> timeout
          |
          v
      network/policy
```

---

# 77. Example RCA

```markdown
# DNS Failure Incident RCA

## Incident

Production order-service DNS failures.

## Severity

SEV-2

## Customer Impact

Checkout requests failed intermittently.

## Detection

UnknownHostException alert and elevated order-service 5xx.

## Symptoms

- order-service could not resolve inventory-service
- CoreDNS error rate increased
- application 5xx increased

## Investigation

The affected Pods had valid DNS configuration.

CoreDNS Pods were running.

NetworkPolicy was reviewed.

A recently deployed egress NetworkPolicy did not allow
DNS traffic to kube-dns on port 53.

## Root Cause

DNS traffic from application Pods was blocked by NetworkPolicy.

## Mitigation

Restored the previous NetworkPolicy through GitOps.

## Permanent Fix

Added automated NetworkPolicy validation ensuring required
DNS access exists for application namespaces.

## Prevention

- DNS synthetic checks
- NetworkPolicy tests
- CoreDNS dashboards
- DNS failure alerts
- dependency documentation
```

---

# 78. Business Validation

After fixing DNS, do not stop at:

```text
nslookup inventory-service
```

Validate:

```text
DNS
 ↓
TCP
 ↓
HTTP
 ↓
Application
 ↓
Business transaction
```

For checkout:

```text
Login
 ↓
Catalog
 ↓
Cart
 ↓
Order
 ↓
Inventory
 ↓
Fraud
 ↓
Payment
 ↓
Invoice
 ↓
Notification
```

---

# 79. Final Production Architecture

```text
                         INTERNET
                            |
                            v
                         Route 53
                            |
                            v
                      CloudFront/WAF
                            |
                            v
                           ALB
                            |
                            v
                     order-service
                            |
                     Kubernetes DNS
                            |
                          CoreDNS
                            |
            +---------------+---------------+
            |               |               |
            v               v               v
      customer-service pricing-service inventory-service
            |               |               |
            v               v               v
           RDS             Redis           Kafka

External DNS:

Application
    |
    v
CoreDNS
    |
    v
VPC/AWS Resolver
    |
    +--> RDS
    +--> Redis
    +--> ECR
    +--> SQS/SNS
    +--> External DNS
```

---

# 80. Final Production Mental Model

When you see:

```text
UnknownHostException
```

think:

```text
WHAT NAME?
    ↓
Kubernetes service?
AWS resource?
External hostname?
    ↓
CAN DNS RESOLVE IT?
    ↓
/etc/resolv.conf
    ↓
CoreDNS
    ↓
Upstream resolver
    ↓
Route 53/VPC DNS
    ↓
CAN NETWORK REACH THE RESULT?
    ↓
TCP
    ↓
TLS
    ↓
HTTP
    ↓
APPLICATION
```

And remember:

> **DNS resolution and network connectivity are separate troubleshooting layers.**

---

# 81. Key Rules to Remember

1. **DNS failure is a symptom, not automatically a CoreDNS problem.**
2. Identify the exact hostname first.
3. Determine whether it is Kubernetes, AWS, or public DNS.
4. Check `/etc/resolv.conf`.
5. Test DNS from the affected Pod.
6. Check the Kubernetes Service and namespace.
7. Check CoreDNS Pods.
8. Check the `kube-dns` Service.
9. Check CoreDNS EndpointSlices.
10. Check NetworkPolicy for UDP/TCP 53.
11. Check VPC DNS settings for AWS resolution.
12. Check VPC CNI when Pod networking is suspected.
13. Check Route 53 for public/private DNS issues.
14. Distinguish NXDOMAIN from timeout/SERVFAIL.
15. Don't restart CoreDNS without evidence.
16. Don't hardcode Pod IPs.
17. Use Kubernetes Service DNS for service discovery.
18. Use correlation IDs, logs, metrics, and traces together.
19. Control retries to prevent DNS storms.
20. Validate DNS → TCP → TLS → HTTP → business transaction after recovery.
21. Fix DNS configuration through GitOps where applicable.
22. Record the actual root cause in the RCA.

---

# 82. Phase 18 Progress

```text
143 — CrashLoopBackOff             ✅
144 — OOMKilled                    ✅
145 — CPU Saturation               ✅
146 — Memory Saturation            ✅
147 — Disk Full                    ✅
148 — ImagePullBackOff             ✅
149 — ALB 5xx                      ✅
150 — Service Connectivity Failure ✅
151 — DNS Failure                  ✅
152 — RDS Connection Exhaustion    ⏭️
153 — Redis Failure                ⏭️
154 — SQS Backlog                  ⏭️
155 — Kafka Consumer Lag           ⏭️
156 — Node Failure                 ⏭️
157 — AZ Failure                   ⏭️
158 — ArgoCD Drift                 ⏭️
159 — Bad Production Deployment    ⏭️
160 — Security Incident            ⏭️
```

**Next: `152 — RDS Connection Exhaustion Production Incident`**.