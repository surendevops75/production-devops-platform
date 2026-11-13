# 150 — Service Connectivity Failure Production Incident

> **Phase 18 — Incident Management**
>
> This chapter covers a production-grade **service-to-service connectivity failure** in the 21-microservice EKS platform.
>
> The goal is to troubleshoot connectivity systematically across:
>
> `Application → Kubernetes Service → EndpointSlice → Pod → Network → Dependency`
>
> without immediately restarting Pods or blaming Kubernetes.

---

## 1. What Is a Service Connectivity Failure?

A service connectivity failure occurs when one component cannot communicate with another component that it depends on.

Examples:

```text
order-service
      |
      X
      |
inventory-service
```

or:

```text
payment-service
      |
      X
      |
Redis
```

or:

```text
order-service
      |
      X
      |
PostgreSQL
```

or:

```text
notification-service
      |
      X
      |
SQS
```

The failure can occur at many layers.

---

# 2. Our Production Architecture

For our e-commerce platform:

```text
                         ALB
                          |
                          v
                   order-service
                          |
          +---------------+---------------+
          |               |               |
          v               v               v
   customer-service  pricing-service  inventory-service
          |               |               |
          v               v               v
         RDS             Redis           Kafka
```

Other communication paths include:

```text
payment-service
      |
      +--> payment provider

notification-service
      |
      +--> SQS
      |
      +--> email/SMS provider

search-service
      |
      +--> OpenSearch/Elasticsearch

reporting-service
      |
      +--> S3
      |
      +--> RDS
```

---

# 3. Connectivity Is More Than "Ping"

In Kubernetes, connectivity involves:

```text
Application
   ↓
DNS
   ↓
Service
   ↓
ClusterIP
   ↓
Service routing
   ↓
EndpointSlice
   ↓
Pod IP
   ↓
NetworkPolicy
   ↓
VPC CNI
   ↓
Node/network
   ↓
Destination
```

A failure anywhere in this chain can appear as:

```text
Connection refused
Connection timed out
Unknown host
No route to host
503
502
504
```

---

# 4. Connectivity Failure Is a Symptom

Suppose:

```text
order-service
```

reports:

```text
Connection refused
```

That does **not** immediately mean:

```text
inventory-service is down
```

Possible causes:

```text
inventory Pod not running
inventory Pod not Ready
Service selector incorrect
Service has no endpoints
Wrong Service port
Wrong targetPort
DNS failure
NetworkPolicy
Security Group
VPC CNI problem
Application not listening
Application bound to localhost
Pod IP changed
Node networking problem
Resource exhaustion
```

---

# 5. Production Incident Scenario

Suppose customers report:

> "Checkout is intermittently failing."

Grafana shows:

```text
order-service
HTTP 5xx: 12%
```

Jaeger shows:

```text
order-service
    |
    +--> inventory-service
             |
             +--> timeout
```

Application logs:

```text
Connection timed out
```

Now the incident is:

```text
order-service
      |
      X
      |
inventory-service
```

We need to identify exactly where the communication fails.

---

# 6. Production Troubleshooting Flow

Use:

```text
Alert
  ↓
Confirm customer impact
  ↓
Identify source service
  ↓
Identify destination
  ↓
Check destination Pods
  ↓
Check Service
  ↓
Check EndpointSlices
  ↓
Check DNS
  ↓
Check port
  ↓
Test connectivity
  ↓
Check NetworkPolicy
  ↓
Check VPC/networking
  ↓
Check application listener
  ↓
Check dependencies
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

# 7. Step 1 — Identify Source and Destination

Do not start with random Kubernetes commands.

First determine:

```text
SOURCE:
order-service

DESTINATION:
inventory-service

PORT:
8080

PROTOCOL:
HTTP

ERROR:
timeout
```

Example:

```text
order-service
  |
  | HTTP :8080
  v
inventory-service
```

This gives you a specific troubleshooting target.

---

# 8. Step 2 — Check Destination Pods

```bash
kubectl get pods \
  -n ecommerce \
  -l app=inventory-service \
  -o wide
```

Example:

```text
NAME                              READY   STATUS    IP
inventory-service-7d8c9f7b-x1a   1/1     Running   10.0.12.31
inventory-service-7d8c9f7b-y2b   1/1     Running   10.0.13.45
```

This looks healthy.

But continue.

---

# 9. Running Does Not Mean Reachable

A Pod can be:

```text
Running
```

but still have:

```text
Application not listening
Readiness failure
NetworkPolicy block
Wrong port
Wrong Service selector
```

Therefore:

```text
Running
```

is only one piece of evidence.

---

# 10. Step 3 — Check Service

```bash
kubectl get svc inventory-service -n ecommerce
```

Example:

```text
NAME               TYPE        CLUSTER-IP
inventory-service  ClusterIP   10.100.42.20
```

Then:

```bash
kubectl describe svc inventory-service -n ecommerce
```

Check:

```text
Selector
Port
TargetPort
Endpoints
```

Example:

```text
Port:       80/TCP
TargetPort: 8080/TCP
```

The application must actually listen on the target port.

---

# 11. Service Port vs TargetPort

This is a very common production issue.

Example:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

Means:

```text
Client
  |
  | :80
  v
Service
  |
  | forwards to :8080
  v
Pod
```

If the application listens on:

```text
9090
```

instead of:

```text
8080
```

traffic fails.

---

# 12. Step 4 — Check EndpointSlices

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=inventory-service
```

Example:

```text
ENDPOINTS
10.0.12.31
10.0.13.45
```

If there are no endpoints:

```text
inventory-service
      |
      X
No ready endpoints
```

The Service has nothing to send traffic to.

---

# 13. Service Selector Problem

Suppose Service:

```yaml
selector:
  app: inventory
```

but Pods use:

```yaml
labels:
  app: inventory-service
```

Then:

```text
Service selector
       |
       X
Pod labels
```

No match.

Result:

```text
Endpoints = none
```

The Pods can be perfectly healthy while the Service cannot reach them.

---

# 14. Step 5 — Check Endpoint Readiness

EndpointSlices contain readiness information.

A Pod can exist:

```text
Pod = Running
```

but not be considered ready.

Therefore:

```text
Pods
 ↓
Readiness
 ↓
EndpointSlice
 ↓
Service
```

must be checked together.

---

# 15. Step 6 — Check DNS

Our services should communicate using Kubernetes Service DNS.

Example:

```text
inventory-service
```

or:

```text
inventory-service.ecommerce.svc.cluster.local
```

From `order-service`:

```text
http://inventory-service:8080
```

is preferable to hardcoding:

```text
http://10.0.12.31:8080
```

because Pod IPs are ephemeral.

---

# 16. Kubernetes DNS Architecture

```text
order-service Pod
       |
       | DNS query
       v
    CoreDNS
       |
       v
Kubernetes API/service information
       |
       v
inventory-service
       |
       v
ClusterIP
```

CoreDNS provides Kubernetes service discovery.

---

# 17. Test DNS From a Pod

Enter a troubleshooting container or appropriate application Pod:

```bash
kubectl exec -it <order-pod> -n ecommerce -- sh
```

Then:

```bash
nslookup inventory-service
```

or, if available:

```bash
getent hosts inventory-service
```

Expected:

```text
inventory-service
10.100.42.20
```

For full DNS:

```bash
nslookup inventory-service.ecommerce.svc.cluster.local
```

---

# 18. DNS Failure vs Connectivity Failure

Suppose:

```text
nslookup inventory-service
```

returns:

```text
server can't find inventory-service
```

This points toward:

```text
CoreDNS
Pod DNS configuration
Network path to CoreDNS
Service name
Namespace
```

But if DNS resolves:

```text
inventory-service → 10.100.42.20
```

and HTTP fails:

```text
curl: connection refused
```

then DNS is working.

Now investigate:

```text
Service
Endpoints
Port
Application
NetworkPolicy
```

---

# 19. Step 7 — Test TCP Connectivity

From the source Pod:

```bash
nc -vz inventory-service 8080
```

If `nc` is available.

Possible result:

```text
Connection to inventory-service 8080 succeeded
```

or:

```text
Connection refused
```

or:

```text
Connection timed out
```

These errors mean different things.

---

# 20. Connection Refused

Typical meaning:

```text
Network path reached destination
but nothing accepted the connection
```

Potential causes:

```text
Wrong port
Application not listening
Application crashed
Application bound to wrong interface
Service targetPort wrong
Target process stopped
```

Example:

```text
inventory-service
      |
      v
Pod :8080
      X
Nothing listening
```

---

# 21. Connection Timeout

Typical investigation:

```text
Packet/request isn't getting a usable response
```

Potential causes:

```text
NetworkPolicy
Security Group
VPC routing
CNI issue
Node networking
Firewall
Destination overloaded
Dropped traffic
```

A timeout generally pushes us deeper into the network path than a simple connection-refused error.

---

# 22. No Route to Host

This suggests a networking/routing problem.

Investigate:

```text
VPC routing
node networking
CNI
subnets
routes
security controls
```

---

# 23. Step 8 — Test HTTP

DNS and TCP may work while HTTP is still broken.

Test:

```bash
curl -v http://inventory-service:8080/actuator/health
```

or the service's internal health endpoint.

Possible result:

```text
HTTP/1.1 200 OK
```

Good.

Or:

```text
HTTP/1.1 503
```

Network works, but application/service health is failing.

---

# 24. Layered Connectivity Testing

Always test in layers:

```text
Layer 1
DNS
 ↓
Layer 2
TCP
 ↓
Layer 3
HTTP
 ↓
Layer 4
Application behavior
```

Example:

```text
DNS       ✅
TCP       ✅
HTTP      ❌
Business  ❌
```

This tells us:

> Network connectivity exists; application-level behavior is failing.

---

# 25. Step 9 — Check Application Listener

For Java/Spring Boot:

```yaml
server:
  port: 8080
```

The application must listen on the expected interface and port.

Inside the Pod, where tools are available:

```bash
ss -lntp
```

or:

```bash
netstat -lntp
```

Look for:

```text
0.0.0.0:8080
```

---

# 26. `localhost` vs `0.0.0.0`

This is a classic container issue.

Bad:

```text
127.0.0.1:8080
```

The application is only listening inside its own network namespace.

Better for containerized applications:

```text
0.0.0.0:8080
```

Then traffic arriving through the Pod network can reach it.

---

# 27. Spring Boot Example

Configuration:

```yaml
server:
  port: 8080
```

Normally Spring Boot listens appropriately for container networking.

If custom configuration forces:

```text
127.0.0.1
```

other Pods may not be able to connect.

---

# 28. Step 10 — Test Pod IP Directly

This is an excellent diagnostic technique.

Get the Pod IP:

```bash
kubectl get pod \
  -n ecommerce \
  -l app=inventory-service \
  -o wide
```

Suppose:

```text
10.0.12.31
```

Then from the source Pod:

```bash
curl -v http://10.0.12.31:8080/actuator/health
```

Compare:

```text
Service DNS
```

with:

```text
Pod IP
```

---

# 29. Interpret the Result

### Service fails, Pod IP works

Potential:

```text
Service configuration
targetPort
selector
Service routing
ClusterIP
```

### Service works, Pod IP fails

Potential:

```text
Pod/application
network path
NetworkPolicy
```

### Both fail

Investigate:

```text
application
Pod
network policy
node networking
```

### Both work

The problem may be:

```text
application-specific request
dependency
timeout
intermittent network behavior
```

---

# 30. Step 11 — Check NetworkPolicy

List policies:

```bash
kubectl get networkpolicy -A
```

For namespace:

```bash
kubectl get networkpolicy -n ecommerce
```

Describe:

```bash
kubectl describe networkpolicy <policy> -n ecommerce
```

---

# 31. Example NetworkPolicy

Suppose:

```yaml
podSelector:
  matchLabels:
    app: inventory-service
```

allows ingress only from:

```yaml
app: cart-service
```

But `order-service` now calls inventory directly.

Result:

```text
order-service
      |
      X
NetworkPolicy
      |
inventory-service
```

Connectivity fails.

---

# 32. NetworkPolicy Principle

NetworkPolicy should be based on explicit communication requirements.

Example:

```text
order-service → inventory-service
order-service → pricing-service
cart-service → product-service
```

should be intentionally allowed.

Do not solve an incident by opening:

```text
from:
  0.0.0.0/0
```

inside the cluster.

---

# 33. Check Namespace

A common mistake:

```text
inventory-service
```

exists in:

```text
ecommerce
```

but source application uses:

```text
inventory-service.other-namespace
```

Check:

```bash
kubectl get svc -A | grep inventory-service
```

Then verify:

```text
namespace
service name
port
```

---

# 34. DNS Name by Namespace

Within the same namespace:

```text
inventory-service
```

Cross-namespace:

```text
inventory-service.ecommerce
```

Full DNS:

```text
inventory-service.ecommerce.svc.cluster.local
```

This is why namespaces matter.

---

# 35. Step 12 — Check CoreDNS

Check:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Depending on your cluster labels/version, inspect the actual CoreDNS Pods.

Then:

```bash
kubectl get deployment coredns -n kube-system
```

Check logs:

```bash
kubectl logs -n kube-system deployment/coredns
```

Look for:

```text
DNS errors
timeouts
plugin failures
upstream issues
```

---

# 36. Don't Restart CoreDNS Immediately

If one application cannot reach another:

```text
order-service → inventory-service
```

don't immediately execute:

```bash
kubectl rollout restart deployment/coredns -n kube-system
```

First prove DNS is the problem.

If:

```bash
nslookup inventory-service
```

works consistently:

```text
DNS is probably not the problem.
```

---

# 37. Step 13 — Check VPC CNI

EKS uses the Amazon VPC CNI for Pod networking in the standard architecture.

Check:

```bash
kubectl get pods -n kube-system
```

Look for:

```text
aws-node
```

Check:

```bash
kubectl get daemonset aws-node -n kube-system
```

If Pods have networking problems, inspect:

```bash
kubectl logs -n kube-system daemonset/aws-node
```

Use the appropriate Pod selection for your installed CNI version.

---

# 38. VPC CNI Architecture

Simplified:

```text
Pod
 |
 v
VPC CNI
 |
 v
ENI/IP allocation
 |
 v
AWS VPC
 |
 v
Other Pod / AWS resource
```

This is why EKS Pod networking is closely connected to:

```text
VPC
Subnets
ENIs
Security Groups
Routes
IP capacity
```

---

# 39. Pod IP Exhaustion

A connectivity issue can occur when the cluster has insufficient available Pod IPs.

This becomes especially important with:

```text
many Pods
HPA scaling
Karpenter
large 21-service deployment
```

Check:

```text
subnet available IPs
ENI capacity
Pod IP allocation
```

This can also produce scheduling or networking symptoms.

---

# 40. Step 14 — Security Groups

For AWS-level connectivity, investigate security groups where applicable.

Example:

```text
EKS workload
     |
     v
RDS
```

RDS security group should allow:

```text
TCP 5432
```

from the approved application/network security group.

For service-to-service Pod traffic, the exact security-group path depends on the EKS networking/target architecture.

Do not blindly modify security groups.

---

# 41. Service-to-RDS Connectivity

Example:

```text
order-service
      |
      v
orders-db.xxxxx.rds.amazonaws.com:5432
```

Test DNS:

```bash
nslookup orders-db.xxxxx.rds.amazonaws.com
```

Test TCP:

```bash
nc -vz orders-db.xxxxx.rds.amazonaws.com 5432
```

Then application authentication/query behavior.

---

# 42. Three-Layer Database Test

```text
DNS
 ↓
TCP 5432
 ↓
PostgreSQL authentication
 ↓
SQL query
```

Example:

```text
DNS       ✅
TCP       ✅
Auth      ❌
```

This is **not a network connectivity problem**.

It is likely:

```text
credentials
database user
secret
authentication configuration
```

---

# 43. Service-to-Redis Connectivity

Example:

```text
cart-service
     |
     v
Redis
```

Test:

```text
DNS
 ↓
TCP 6379
 ↓
TLS/authentication
 ↓
Redis command
```

If:

```text
TCP works
Redis AUTH fails
```

investigate credentials/security rather than Kubernetes routing.

---

# 44. Service-to-Kafka Connectivity

For Kafka:

```text
notification-service
       |
       v
Kafka broker
```

Investigate:

```text
DNS
broker address
port
TLS
authentication
network policy
security group
consumer group
```

A Kafka connectivity problem can manifest as:

```text
consumer lag
timeouts
connection retries
```

---

# 45. Service-to-SQS Is Different

SQS is an AWS API, not a Kubernetes Service.

Architecture:

```text
notification-service
       |
       v
AWS SDK
       |
       v
AWS API
       |
       v
SQS
```

Troubleshooting:

```text
DNS/network
 ↓
AWS API connectivity
 ↓
IAM/Pod Identity
 ↓
SQS permissions
 ↓
queue behavior
```

If IAM fails:

```text
AccessDenied
```

that's different from:

```text
Connection timeout
```

---

# 46. Service Connectivity Matrix

For our platform, maintain a dependency matrix.

Example:

| Source | Destination | Protocol | Port |
|---|---|---:|---:|
| order-service | customer-service | HTTP | 8080 |
| order-service | pricing-service | HTTP | 8080 |
| order-service | inventory-service | HTTP | 8080 |
| order-service | RDS | PostgreSQL | 5432 |
| cart-service | Redis | Redis | 6379 |
| payment-service | payment provider | HTTPS | 443 |
| notification-service | SQS | HTTPS | 443 |
| reporting-service | S3 | HTTPS | 443 |

This becomes extremely useful during incidents.

---

# 47. Check Recent Deployment

Connectivity can break after a deployment.

Check:

```bash
kubectl rollout history deployment/order-service -n ecommerce
```

Look for changes to:

```text
environment variables
Service URLs
ports
timeouts
TLS
credentials
NetworkPolicy
sidecars
Service names
```

---

# 48. Common Deployment Error

Old configuration:

```text
http://inventory-service:8080
```

New configuration:

```text
http://inventory-service:9090
```

But inventory still listens on:

```text
8080
```

Result:

```text
Connection refused
```

The infrastructure is healthy.

The application configuration is wrong.

---

# 49. Another Common Error — Wrong Service Name

Correct:

```text
inventory-service
```

Incorrect:

```text
inventory
```

Application:

```text
http://inventory:8080
```

DNS fails.

---

# 50. Another Common Error — HTTP vs HTTPS

Suppose destination expects:

```text
HTTPS :8443
```

but source calls:

```text
HTTP :8080
```

TCP may succeed but application protocol negotiation fails.

This is why:

```text
DNS
TCP
HTTP
```

must be tested separately.

---

# 51. Another Common Error — TLS Certificate

Suppose:

```text
TCP = works
```

but:

```text
TLS = fails
```

Possible:

```text
certificate
hostname
CA trust
TLS version
SNI
```

The network is functioning, but secure application communication is failing.

---

# 52. Check Service Mesh / Sidecar If Present

Our baseline project does not require a service mesh.

If one is introduced later:

```text
Application
   |
   v
Sidecar proxy
   |
   v
Network
   |
   v
Destination sidecar
   |
   v
Application
```

Connectivity troubleshooting becomes more complex.

You would investigate:

```text
application
proxy
mTLS
policy
routing
destination
```

Don't introduce service-mesh complexity unless there is a real requirement.

---

# 53. Intermittent Connectivity

The hardest case:

```text
request 1 → success
request 2 → timeout
request 3 → success
request 4 → timeout
```

Potential causes:

```text
one unhealthy Pod
one unhealthy node
connection pool exhaustion
network packet loss
DNS inconsistency
load distribution
resource saturation
specific endpoint failure
```

---

# 54. One Bad Pod

Suppose:

```text
inventory-service

Pod A → healthy
Pod B → healthy
Pod C → broken
```

Service load balancing sends requests to all eligible endpoints.

Result:

```text
80% success
20% failure
```

This can create intermittent errors.

Check:

```bash
kubectl get pods -n ecommerce -o wide
```

and:

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=inventory-service
```

Then correlate failures with Pod IPs.

---

# 55. One Bad Node

Suppose:

```text
Node A → healthy
Node B → networking issue
Node C → healthy
```

Pods on Node B may experience connectivity failures.

Compare:

```text
Pod
 ↓
Node
 ↓
Failure percentage
```

This is a powerful production diagnostic technique.

---

# 56. Resource Saturation Can Look Like Connectivity Failure

Suppose inventory-service CPU reaches:

```text
100%
```

Requests start timing out.

The source service reports:

```text
connection timeout
```

It may appear to be a network problem.

But actual chain:

```text
CPU saturation
 ↓
slow application
 ↓
requests queue
 ↓
timeout
 ↓
source sees connectivity timeout
```

Always correlate:

```text
network
+
CPU
+
memory
+
latency
```

---

# 57. Connection Pool Exhaustion

A service can have healthy network connectivity but no available connections.

For example:

```text
Hikari pool
max = 20

20 connections active
0 available
```

New requests wait.

Eventually:

```text
timeout
```

Application reports:

```text
connection timeout
```

This can be mistaken for a network issue.

Check:

```text
Hikari active
Hikari idle
Hikari pending
connection timeout
RDS connections
```

---

# 58. Retry Storm During Connectivity Failure

Suppose:

```text
order-service
      |
      X
inventory-service
```

Application retries:

```text
3 times
```

At 2,000 RPS:

```text
2,000 × 3
```

can create huge additional traffic.

Use:

```text
timeouts
exponential backoff
jitter
retry limits
circuit breakers
```

---

# 59. Circuit Breaker

Architecture:

```text
order-service
     |
     v
inventory-service
     |
     X
failure
```

Without circuit breaker:

```text
keep sending requests
```

With circuit breaker:

```text
failure threshold
       ↓
circuit opens
       ↓
stop sending requests temporarily
       ↓
protect source + destination
```

This prevents cascading failures.

---

# 60. Graceful Degradation

Not every dependency has the same criticality.

For example:

```text
Order
 |
 +--> Inventory       CRITICAL
 +--> Payment         CRITICAL
 +--> Fraud           CRITICAL
 +--> Recommendation  NON-CRITICAL
```

If recommendation-service fails:

```text
checkout should continue
```

if business requirements permit.

But if payment fails:

```text
order completion should not proceed
```

Design connectivity handling according to business criticality.

---

# 61. Distributed Tracing

Our OpenTelemetry architecture:

```text
Service
   |
   v
OpenTelemetry Java Agent
   |
   v
OpenTelemetry Collector
   |
   v
Jaeger
```

A connectivity failure should produce spans showing:

```text
order-service
     |
     +--> inventory-service
              |
              +--> timeout
```

Look for:

```text
duration
status
error
exception
peer.service
network attributes
```

---

# 62. Log Correlation

Example:

```text
correlationId=abc123
traceId=xyz456
```

Search:

```text
ELK
```

for:

```text
correlationId: abc123
```

You may find:

```text
order-service
inventory-service
RDS-related errors
```

This provides evidence across services.

---

# 63. Metrics

Useful metrics:

```text
Request rate
Error rate
Latency
Connection failures
Timeout count
Retry count
Circuit breaker state
Active connections
Pool saturation
Pod restarts
CPU
Memory
Network errors
```

Don't build an alert only around:

```text
network packet count
```

Tie infrastructure symptoms to customer impact.

---

# 64. Example Incident

### 10:00

Normal traffic:

```text
1,000 RPS
```

### 10:05

Inventory deployment:

```text
v3.4.0
```

### 10:07

One Pod becomes unhealthy.

### 10:08

Order API begins showing:

```text
3% timeout
```

### 10:10

Retries increase traffic.

### 10:12

Order-service CPU rises.

### 10:14

5xx reaches:

```text
15%
```

### 10:15

Incident declared.

---

# 65. Investigation

Engineer checks:

```bash
kubectl get pods -n ecommerce -o wide
```

Finds:

```text
inventory Pod C
```

has:

```text
Ready = 0/1
```

But EndpointSlice still appears to contain unexpected endpoints.

Investigation discovers a readiness configuration issue.

---

# 66. Root Cause

Deployment `v3.4.0` changed:

```text
readiness behavior
```

and allowed a partially initialized Pod to receive traffic.

The Pod accepted requests but could not access its dependency.

Result:

```text
bad Pod
 ↓
inventory failures
 ↓
order retries
 ↓
order CPU increase
 ↓
5xx
```

This is a realistic cascading incident.

---

# 67. Immediate Mitigation

Possible actions:

```text
stop rollout
rollback inventory-service
```

GitOps:

```text
revert commit
   ↓
ArgoCD
   ↓
known-good inventory version
```

Then monitor:

```text
inventory health
order 5xx
latency
retry rate
```

---

# 68. Validation

First:

```bash
kubectl get pods -n ecommerce
```

Then:

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=inventory-service
```

Then test:

```bash
curl http://inventory-service:8080/actuator/health
```

Finally test:

```text
Create order
 ↓
Reserve inventory
 ↓
Payment
 ↓
Confirmation
```

---

# 69. Do Not Test Only From Inside One Pod

A connectivity test from:

```text
order-service Pod A
```

can succeed while:

```text
order-service Pod B
```

fails.

For intermittent incidents, test multiple source Pods/nodes.

Example:

```text
Pod A → inventory = success
Pod B → inventory = timeout
Pod C → inventory = success
```

Now investigate:

```text
Pod B
Node hosting Pod B
Network path
```

---

# 70. Production Connectivity Test Pod

For debugging, organizations often maintain a small approved network-debug image/toolbox.

It may contain:

```text
curl
nslookup/dig
getent
nc
wget
ip
ss
```

Example concept:

```text
debug Pod
   |
   +--> DNS
   +--> TCP
   +--> HTTP
```

Do not install random debugging packages into production application containers merely for an incident.

---

# 71. Network Troubleshooting Hierarchy

Use this exact hierarchy:

```text
1. DNS
     ↓
2. TCP
     ↓
3. TLS
     ↓
4. HTTP
     ↓
5. Application
     ↓
6. Dependency
```

Example:

```text
DNS       ✅
TCP       ❌
```

Stop investigating HTTP.

Example:

```text
DNS       ✅
TCP       ✅
TLS       ❌
```

Investigate certificate/trust/configuration.

Example:

```text
DNS       ✅
TCP       ✅
TLS       ✅
HTTP      500
```

Investigate application/dependency behavior.

---

# 72. Common Error Mapping

| Error | First investigation |
|---|---|
| `Unknown host` | DNS |
| `Connection refused` | Port/listener/Service |
| `Connection timed out` | NetworkPolicy/network/resource |
| `No route to host` | Routing/network |
| `Connection reset` | Target/application/network |
| `TLS handshake failed` | TLS/certificates |
| `401/403` | Authentication/authorization |
| `404` | Application routing |
| `500` | Application/dependency |
| `502` | Proxy/target response path |
| `503` | Service/target availability |
| `504` | Timeout/dependency |

These are starting points, not absolute diagnoses.

---

# 73. Service Connectivity Monitoring

Create dashboards for:

```text
Service A → Service B
```

with:

```text
request rate
success rate
timeout rate
P95/P99 latency
connection errors
retry count
circuit state
```

For critical paths:

```text
order → inventory
order → payment
order → fraud
```

these metrics are especially valuable.

---

# 74. Dependency Map

Maintain a dependency map:

```text
order-service
 |
 +--> customer-service
 +--> pricing-service
 +--> promotion-service
 +--> inventory-service
 +--> payment-service
 +--> fraud-service
 +--> tax-service
```

Then:

```text
inventory-service
 |
 +--> RDS
 +--> Redis
 +--> Kafka
```

This makes incident triage much faster.

---

# 75. Production Prevention

### Kubernetes

- [ ] Correct Service selectors
- [ ] Correct `port`
- [ ] Correct `targetPort`
- [ ] Readiness probes
- [ ] EndpointSlice monitoring
- [ ] NetworkPolicy reviewed
- [ ] DNS monitoring
- [ ] CNI monitoring

### Application

- [ ] Timeouts
- [ ] Retries
- [ ] Backoff
- [ ] Circuit breakers
- [ ] Correlation IDs
- [ ] Distributed tracing
- [ ] Connection pool limits
- [ ] Graceful degradation

### AWS

- [ ] VPC routes
- [ ] Subnet capacity
- [ ] Security Groups
- [ ] VPC CNI
- [ ] DNS
- [ ] Required VPC endpoints/NAT

### Operations

- [ ] Dependency map
- [ ] Connectivity runbook
- [ ] Synthetic tests
- [ ] Service-level dashboards
- [ ] Incident game days

---

# 76. Connectivity Runbook

Create:

```text
runbooks/service-connectivity.md
```

Recommended structure:

```markdown
# Service Connectivity Failure Runbook

## Symptoms

## Customer Impact

## Identify Source Service

## Identify Destination

## Check Destination Pods

## Check Service

## Check EndpointSlices

## Check DNS

## Check TCP

## Check TLS

## Check HTTP

## Check NetworkPolicy

## Check VPC CNI

## Check Security Groups

## Check Resource Saturation

## Check Connection Pools

## Check Dependencies

## Check Recent Deployment

## Mitigation

## Validation

## RCA
```

---

# 77. RCA Example

```markdown
# Service Connectivity Incident RCA

## Incident

order-service could not reliably reach inventory-service.

## Severity

SEV-2

## Customer Impact

Checkout requests intermittently failed.

## Detection

HTTP timeout and 5xx alerts.

## Symptoms

- order-service timeout rate increased
- inventory-service request failures increased
- retry rate increased

## Investigation

DNS was healthy.

TCP connectivity worked from most Pods.

One inventory Pod was accepting traffic but was not fully initialized.

## Root Cause

Incorrect readiness configuration allowed a partially
initialized Pod to receive traffic.

## Contributing Factors

- Missing synthetic dependency test
- Retry policy amplified failures
- Insufficient readiness validation

## Mitigation

Rolled back inventory-service.

## Permanent Fix

Corrected readiness behavior and added startup integration tests.

## Prevention

- readiness contract tests
- service connectivity monitoring
- retry budget
- circuit breaker
- dependency dashboard
```

---

# 78. Final Decision Tree

```text
Service Connectivity Failure
            |
            v
Who is calling whom?
            |
            v
Destination Pods healthy?
       |
   +---+---+
   |       |
  No      Yes
   |       |
   v       v
Fix Pod   Service exists?
             |
         +---+---+
         |       |
        No      Yes
         |       |
         v       v
       Fix     Endpoints?
                 |
             +---+---+
             |       |
            No      Yes
             |       |
             v       v
         Selector/   DNS?
         readiness     |
                    +--+--+
                    |     |
                   No    Yes
                    |     |
                    v     v
                  DNS    TCP?
                         |
                     +---+---+
                     |       |
                    No      Yes
                     |       |
                     v       v
                  Network  TLS?
                           |
                       +---+---+
                       |       |
                      No      Yes
                       |       |
                       v       v
                     TLS     HTTP?
                              |
                          +---+---+
                          |       |
                         No      Yes
                          |       |
                          v       v
                       App/TLS  Dependency?
```

---

# 79. Final Production Mental Model

When someone says:

> **"order-service cannot connect to inventory-service."**

Think:

```text
SOURCE
order-service
   |
   v
DNS
   |
   v
Service
   |
   v
EndpointSlice
   |
   v
Pod IP
   |
   v
NetworkPolicy
   |
   v
VPC CNI / Network
   |
   v
Port
   |
   v
Application listener
   |
   v
HTTP
   |
   v
Business logic
   |
   v
Dependency
```

Troubleshoot from the **lowest-cost, highest-signal layer first**.

---

# 80. Key Rules to Remember

1. **A connectivity error is not automatically a network problem.**
2. Identify the **source, destination, protocol, and port** first.
3. Check Pods.
4. Check Service.
5. Check EndpointSlices.
6. Check DNS.
7. Check TCP.
8. Check TLS.
9. Check HTTP.
10. Check application behavior.
11. Check NetworkPolicies.
12. Check VPC CNI and AWS networking when required.
13. Check security groups for AWS-resource connectivity.
14. Check connection pools and resource saturation.
15. Do not confuse timeout with connection refusal.
16. Do not restart CoreDNS without proving DNS is the problem.
17. Do not open NetworkPolicy or Security Groups broadly just to restore traffic.
18. Use correlation IDs and Jaeger to trace cross-service failures.
19. Control retries to prevent cascading failures.
20. Use circuit breakers for appropriate dependencies.
21. Design graceful degradation for non-critical dependencies.
22. Validate the complete business workflow after recovery.
23. Fix GitOps desired state rather than leaving manual drift.
24. Record the actual root cause, not simply "network issue."

---

# 81. Complete Connectivity Architecture

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
             +--------------+--------------+
             |              |              |
             v              v              v
       customer-service pricing-service inventory-service
             |              |              |
             v              v              v
            RDS           Redis           Kafka
             |
             v
       PostgreSQL

Kubernetes internal communication:

Service DNS
     |
     v
CoreDNS
     |
     v
ClusterIP
     |
     v
EndpointSlice
     |
     v
Pod IP
     |
     v
VPC CNI
     |
     v
NetworkPolicy / AWS networking
     |
     v
Destination Pod
```

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
151 — DNS Failure                  ⏭️
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

**Next: `151 — DNS Failure Production Incident`**.