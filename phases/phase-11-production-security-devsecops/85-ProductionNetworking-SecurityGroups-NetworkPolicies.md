# 85 — Production Networking, Security Groups & Network Policies

> Practical implementation of the network-security layer for the 21-microservice EKS platform. This chapter connects the VPC, EKS, ALB, RDS, Redis, Kubernetes Services, and application Pods into one controlled production network.

---

## 1. Objective

We have now completed the deployment-strategy series:

```text
81 — Rolling Deployment
82 — Blue-Green Deployment
83 — Canary Deployment
84 — Argo Rollouts
```

Now we return to the broader platform and implement the **production network-security model**.

Our target architecture is:

```text
Internet
   |
   v
Route 53
   |
   v
CloudFront / WAF
   |
   v
AWS ALB
   |
   v
EKS
   |
   +-------------------------------+
   |                               |
Application Pods              Platform Pods
   |                               |
   +---------------+---------------+
                   |
          +--------+--------+
          |        |        |
         RDS     Redis    Messaging
```

The critical question is:

> Who is allowed to communicate with whom?

---

# 2. Security Has Multiple Layers

Do not think:

```text
Security Group = network security
```

Production security is layered:

```text
AWS Network
      |
      +-- VPC
      +-- Subnets
      +-- Route Tables
      +-- Security Groups
      +-- NACLs
      |
Kubernetes
      |
      +-- Services
      +-- NetworkPolicies
      +-- RBAC
      +-- ServiceAccounts
      |
Application
      |
      +-- TLS
      +-- Authentication
      +-- Authorization
      +-- Secrets
```

Each layer solves a different problem.

---

# 3. Our VPC

From Chapter 57:

```text
VPC
10.0.0.0/16
```

Example:

```text
Public:

10.0.1.0/24
10.0.2.0/24
10.0.3.0/24
```

Private:

```text
10.0.11.0/24
10.0.12.0/24
10.0.13.0/24
```

Architecture:

```text
                    VPC
                     |
       +-------------+-------------+
       |                           |
   Public Subnets             Private Subnets
       |                           |
      ALB                       EKS Nodes
                                   |
                              Application Pods
                                   |
                              RDS / Redis
```

---

# 4. Production Inbound Traffic

The preferred flow is:

```text
Internet
   |
   v
CloudFront / WAF
   |
   v
ALB
   |
   v
EKS Service
   |
   v
Pod
```

We do **not** expose every application Pod directly to the Internet.

For example:

```text
order-service Pod
```

should not have:

```text
Public IP
```

Instead:

```text
ALB
 ↓
Private EKS
 ↓
Pod
```

---

# 5. Security Groups

AWS Security Groups are stateful virtual firewalls.

Think:

```text
Security Group
       |
       +-- Inbound rules
       |
       +-- Outbound rules
```

They operate at the AWS network-interface level.

For our architecture, we can conceptually have:

```text
ALB Security Group
        |
        v
EKS workload/node security boundary
        |
        v
RDS Security Group
```

---

# 6. ALB Security Group

The ALB is Internet-facing.

Example inbound:

```text
HTTPS 443
Source:
0.0.0.0/0
```

Potential HTTP:

```text
HTTP 80
Source:
0.0.0.0/0
```

but normally redirect HTTP to HTTPS.

Conceptually:

```text
ALB-SG

Inbound:
443 → Internet

Outbound:
Application traffic
```

Don't open unnecessary ports.

For example, don't add:

```text
22
3306
5432
6379
9090
```

to an Internet-facing ALB security group simply because those ports exist elsewhere in the architecture.

---

# 7. EKS Security Boundary

The exact EKS security-group architecture depends on the EKS networking mode and AWS Load Balancer Controller configuration.

At a conceptual level:

```text
ALB
 |
 | Application port
 v
EKS workload
```

The application should only accept the traffic it actually needs.

For example:

```text
ALB → order-service → 8080
```

rather than:

```text
ALB → all ports
```

---

# 8. RDS Security Group

RDS should be private.

Example:

```text
RDS-SG
```

Inbound:

```text
PostgreSQL 5432
Source:
approved application/EKS security boundary
```

Not:

```text
0.0.0.0/0
```

The desired path:

```text
Internet
   X
   |
RDS

EKS
 |
 | TCP 5432
 v
RDS
```

---

# 9. Why RDS Should Not Be Public

If RDS has:

```text
PubliclyAccessible = true
```

you have created an unnecessary external attack surface.

Instead:

```text
Internet
   X
   |
RDS
```

while:

```text
EKS
 |
 v
RDS
```

works privately.

Application configuration:

```text
DB_HOST:
private-RDS-endpoint
```

---

# 10. Redis Security Group

For ElastiCache/Redis:

```text
Redis-SG
```

Allow:

```text
EKS workload
    |
    | Redis port
    v
Redis
```

For Redis/Valkey-compatible services, the actual port/configuration depends on the selected service and deployment.

Do not expose Redis publicly.

---

# 11. SQS Is Different

SQS isn't accessed through a security group like RDS.

Instead:

```text
Pod
 |
 v
AWS API
 |
 v
SQS
```

Authorization comes primarily from:

```text
IAM
```

For example:

```text
order-service
   |
EKS Pod Identity
   |
IAM Role
   |
sqs:SendMessage
   |
specific queue
```

This demonstrates why AWS network controls and IAM controls solve different problems.

---

# 12. Security Group vs IAM

### RDS

Network access:

```text
Security Group
```

Authentication/authorization:

```text
Database credentials
```

### SQS

AWS API authorization:

```text
IAM
```

### S3

AWS API authorization:

```text
IAM
```

### EKS API

```text
IAM + Kubernetes RBAC
```

Therefore:

```text
Network ≠ Identity
```

Both matter.

---

# 13. NACLs

Network ACLs operate at the subnet level.

Our architecture should normally use:

```text
Security Groups
+
NACLs
```

but don't turn NACLs into an unnecessarily complicated firewall layer.

A common operational model is:

```text
Security Groups
    ↓
Primary workload-level AWS network control

NACLs
    ↓
Subnet-level guardrail
```

---

# 14. Security Group vs NACL

| Feature        | Security Group           | NACL                    |
| -------------- | ------------------------ | ----------------------- |
| Scope          | ENI/resource             | Subnet                  |
| Stateful       | Yes                      | No                      |
| Return traffic | Automatically allowed    | Explicit rules required |
| Rules          | Allow                    | Allow/Deny              |
| Common use     | Workload/resource access | Subnet guardrail        |
| Evaluation     | All applicable rules     | Ordered rules           |

The practical troubleshooting question is:

```text
Is the traffic blocked at:
Route?
NACL?
Security Group?
Kubernetes NetworkPolicy?
Application?
```

---

# 15. Kubernetes NetworkPolicy

AWS Security Groups don't provide enough application-level isolation inside Kubernetes.

For example:

```text
order-service
```

may need to communicate with:

```text
RDS
Redis
inventory-service
payment-service
```

but shouldn't necessarily communicate freely with every other Pod.

That's where:

```text
NetworkPolicy
```

comes in.

---

# 16. Default Deny

A production namespace can start with a default deny policy.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: default-deny-ingress
  namespace: ecommerce-prod

spec:
  podSelector: {}

  policyTypes:
    - Ingress
```

This means:

> By default, Pods in this namespace don't accept ingress traffic unless another NetworkPolicy permits it.

This is powerful.

But don't apply it blindly to a production namespace before understanding every required traffic path.

---

# 17. Default Deny Egress

You can also restrict outbound traffic:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: default-deny-egress
  namespace: ecommerce-prod

spec:
  podSelector: {}

  policyTypes:
    - Egress
```

Now you must explicitly permit required outbound traffic.

This can include:

```text
DNS
Other microservices
RDS
Redis
Messaging
AWS endpoints
External APIs
```

This is much more complex than ingress-only isolation.

---

# 18. DNS Must Be Allowed

If you apply egress deny:

```text
Pod
  X
  |
CoreDNS
```

then applications may stop resolving:

```text
rds-endpoint
redis-endpoint
order-service
```

Therefore DNS must be explicitly allowed.

Example:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: allow-dns
  namespace: ecommerce-prod

spec:
  podSelector: {}

  policyTypes:
    - Egress

  egress:

    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system

      ports:
        - protocol: UDP
          port: 53

        - protocol: TCP
          port: 53
```

The exact DNS namespace/labels should be verified against your cluster.

---

# 19. Application-to-Application Policy

Suppose:

```text
order-service
```

needs:

```text
inventory-service
```

We can permit only that traffic.

Conceptually:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: order-to-inventory
  namespace: ecommerce-prod

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
              app.kubernetes.io/name: inventory-service

      ports:
        - protocol: TCP
          port: 8080
```

Now:

```text
order-service → inventory-service
```

is permitted.

---

# 20. But What About Namespace Isolation?

Suppose:

```text
order-service
```

is in:

```text
ecommerce-prod
```

and:

```text
inventory-service
```

is also there.

A `podSelector` is namespace-scoped.

For cross-namespace communication, use:

```yaml
namespaceSelector:
```

and:

```yaml
podSelector:
```

together when required.

This is why understanding Kubernetes namespaces matters.

---

# 21. Example RDS Egress

NetworkPolicy can restrict application traffic toward RDS at the Pod network-policy layer.

Conceptually:

```yaml
egress:

  - to:
      - ipBlock:
          cidr: 10.0.20.0/24

    ports:
      - protocol: TCP
        port: 5432
```

However, for production, don't blindly hard-code an RDS subnet CIDR if your infrastructure may change.

A better design should use stable network architecture and complementary AWS controls.

Also remember:

> NetworkPolicy and Security Group rules both need to allow the connection.

---

# 22. Multiple Security Layers

For:

```text
order-service → RDS
```

we can have:

```text
Application Pod
      |
      | NetworkPolicy
      v
Pod network
      |
      | AWS networking
      v
RDS Security Group
      |
      v
RDS
```

The connection succeeds only if the required layers permit it.

---

# 23. Troubleshooting Network Connectivity

Suppose:

```text
order-service
```

cannot connect to:

```text
RDS
```

Don't immediately change the Security Group.

Use a structured flow:

```text
Application
   |
   v
DNS
   |
   v
Route
   |
   v
NACL
   |
   v
Security Group
   |
   v
NetworkPolicy
   |
   v
RDS
```

---

# 24. Step 1 — DNS

From the Pod:

```bash
kubectl exec -it <pod> \
  -n ecommerce-prod \
  -- getent hosts <rds-endpoint>
```

Or if DNS utilities exist:

```bash
nslookup <rds-endpoint>
```

If DNS fails:

```text
Check CoreDNS
Check NetworkPolicy
Check VPC DNS
Check cluster DNS configuration
```

---

# 25. Step 2 — Port Connectivity

From a debugging Pod:

```bash
kubectl run net-debug \
  --rm \
  -it \
  --image=nicolaka/netshoot \
  -n ecommerce-prod \
  -- bash
```

Then:

```bash
nc -vz <rds-endpoint> 5432
```

Possible results:

```text
succeeded
```

or:

```text
connection timed out
```

or:

```text
connection refused
```

These indicate different problems.

---

# 26. Timeout vs Connection Refused

### Timeout

Often investigate:

```text
Route
NACL
Security Group
NetworkPolicy
Network path
```

### Connection refused

Usually indicates the network path reached the destination but the port/application isn't accepting the connection.

Investigate:

```text
Service
Port
Database
Listener
Target
```

Don't treat every connectivity failure as a Security Group problem.

---

# 27. Step 3 — Kubernetes Service

For application-to-application traffic:

```bash
kubectl get svc \
  -n ecommerce-prod
```

Then:

```bash
kubectl get endpointslice \
  -n ecommerce-prod \
  -l kubernetes.io/service-name=inventory-service
```

If there are no endpoints:

```text
Service
   |
   X
   |
No matching Pods
```

Check:

```bash
kubectl get pods \
  -n ecommerce-prod \
  --show-labels
```

---

# 28. Service Selector Problem

Service:

```yaml
selector:
  app: inventory
```

Pod:

```yaml
labels:
  app: inventory-service
```

They don't match.

Result:

```text
Service endpoints = 0
```

The network is fine.

The Kubernetes configuration is wrong.

This distinction is important in production troubleshooting.

---

# 29. ALB Traffic Flow

For:

```text
Internet → order-service
```

the path is:

```text
Internet
   |
Route 53
   |
CloudFront/WAF
   |
ALB
   |
Target
   |
Service
   |
Pod
```

Check each layer separately.

---

# 30. ALB Troubleshooting

Check Ingress:

```bash
kubectl get ingress \
  -n ecommerce-prod
```

Then:

```bash
kubectl describe ingress order-service \
  -n ecommerce-prod
```

Check Load Balancer Controller:

```bash
kubectl get pods \
  -n kube-system
```

Find the controller:

```bash
kubectl logs \
  -n kube-system \
  deployment/aws-load-balancer-controller
```

The deployment name may differ depending on your installation.

---

# 31. Common ALB Problems

### No ALB

Possible:

```text
Ingress annotation issue
AWS Load Balancer Controller failure
IAM problem
Subnet tags
Security Group issue
```

### ALB exists but unhealthy targets

Investigate:

```text
Pod readiness
Service targetPort
Security Group
Health check path
Health check port
Application listener
```

---

# 32. Health Check Path

Suppose ALB expects:

```text
/health
```

but Spring Boot exposes:

```text
/actuator/health
```

The ALB target becomes unhealthy.

Always verify:

```text
Health check path
Health check port
Success code
Readiness
```

---

# 33. Application Port Chain

We need consistency:

```text
ALB
 ↓
Service port 80
 ↓
targetPort 8080
 ↓
Container port 8080
```

Example:

```yaml
ports:
  - port: 80
    targetPort: 8080
```

Container:

```yaml
ports:
  - containerPort: 8080
```

A mismatch can produce:

```text
502
503
connection refused
```

---

# 34. NetworkPolicy + ALB

If you introduce:

```text
default-deny-ingress
```

your application may stop accepting traffic from the ingress path.

You then need an appropriate policy allowing traffic from the relevant ingress/controller/workload path.

The exact source identity depends on how your AWS Load Balancer Controller and Service networking are configured.

Do not blindly create:

```text
allow-all-ingress
```

just to make the ALB work.

---

# 35. Namespace Design

Our production environment can use:

```text
ecommerce-prod
```

for application workloads.

Platform components may use:

```text
argocd
monitoring
logging
external-secrets
argo-rollouts
```

Potentially:

```text
ecommerce-dev
ecommerce-qa
ecommerce-staging
ecommerce-prod
```

This provides logical isolation.

---

# 36. Namespace Labels

Add useful labels:

```yaml
apiVersion: v1
kind: Namespace

metadata:
  name: ecommerce-prod

  labels:
    environment: prod
    team: backend-platform
```

These labels can later be used by:

```text
NetworkPolicy
Admission controls
Monitoring
Cost allocation
Policy engines
```

---

# 37. Pod Labels

Standardize labels across all 21 services.

Example:

```yaml
labels:
  app.kubernetes.io/name: order-service
  app.kubernetes.io/instance: order-service
  app.kubernetes.io/version: "2.4.0"
  app.kubernetes.io/component: backend
  app.kubernetes.io/part-of: ecommerce-platform
  app.kubernetes.io/managed-by: Helm
  environment: prod
```

These labels are valuable for:

```text
Prometheus
Grafana
NetworkPolicy
Troubleshooting
Cost reporting
Log filtering
ArgoCD
```

---

# 38. Standardize Service Names

Use predictable names:

```text
auth-service
user-service
customer-service
catalog-service
product-service
inventory-service
pricing-service
promotion-service
cart-service
order-service
payment-service
fraud-service
shipping-service
delivery-service
notification-service
search-service
recommendation-service
review-service
tax-service
invoice-service
reporting-service
```

Then service discovery becomes:

```text
http://inventory-service.ecommerce-prod.svc.cluster.local
```

or usually simply:

```text
http://inventory-service:8080
```

from the same namespace.

---

# 39. Microservice Communication Matrix

Before implementing NetworkPolicies across 21 services, create a communication matrix.

Example:

| Source          | Destination          |    Port | Purpose           |
| --------------- | -------------------- | ------: | ----------------- |
| order-service   | inventory-service    |    8080 | Reserve inventory |
| order-service   | payment-service      |    8080 | Payment           |
| order-service   | notification-service |    8080 | Notification      |
| order-service   | RDS                  |    5432 | Orders            |
| order-service   | Redis                |    6379 | Cache             |
| order-service   | SQS                  | AWS API | Events            |
| payment-service | fraud-service        |    8080 | Fraud check       |

This becomes the source of truth for network policy design.

---

# 40. Don't Start With 21 Policies

A common mistake:

```text
Create 50 NetworkPolicies
```

before understanding traffic.

Instead:

```text
1. Map traffic
2. Deploy observability
3. Start with default-deny in a controlled environment
4. Add required allows
5. Test
6. Promote to production
```

NetworkPolicy mistakes can break production applications.

---

# 41. Production NetworkPolicy Strategy

Recommended progression:

```text
DEV
 ↓
Observe traffic
 ↓
Define policy
 ↓
Test
 ↓
QA
 ↓
STAGING
 ↓
Production
```

Do not first discover missing dependencies in production.

---

# 42. Egress to AWS Services

Some applications need:

```text
S3
SQS
SNS
Secrets Manager
KMS
CloudWatch APIs
```

These aren't ordinary Kubernetes Services.

The Pod may communicate through:

```text
VPC
 ↓
NAT Gateway
 ↓
AWS service
```

or:

```text
VPC Endpoint
 ↓
AWS service
```

depending on your architecture.

---

# 43. VPC Endpoints

For private EKS workloads, VPC endpoints can reduce dependency on NAT for supported AWS services.

Examples can include:

```text
S3 Gateway Endpoint
ECR API
ECR DKR
STS
Secrets Manager
CloudWatch
```

The exact endpoint set should be based on actual workload requirements.

Don't blindly deploy every available endpoint.

---

# 44. Why This Matters

Without an appropriate path:

```text
Private Pod
    |
    v
NAT
    |
    v
AWS API
```

If NAT is unavailable:

```text
Pod
 X
AWS API
```

Potentially affecting:

```text
ECR
S3
Secrets
SQS
```

depending on the workload.

This is one reason VPC design and application architecture cannot be treated separately.

---

# 45. EKS Private Nodes

Our production architecture keeps worker nodes private:

```text
Internet
   X
   |
EKS Node
```

Instead:

```text
ALB
 |
 v
Private EKS Nodes
```

Outbound:

```text
EKS
 |
 +-- VPC Endpoint
 |
 +-- NAT Gateway
```

depending on destination.

---

# 46. Kubernetes API Access

The EKS API endpoint should also be deliberately configured.

Possible architecture:

```text
Developers
     |
VPN / corporate network
     |
Private EKS API
```

or:

```text
Controlled public endpoint
+
restricted CIDRs
```

Do not expose the Kubernetes API broadly without understanding the security implications.

---

# 47. IAM + Network Together

For:

```text
order-service → S3
```

we need:

```text
Network path
+
IAM permission
```

For example:

```text
Pod
 |
 | Network
 v
AWS S3 endpoint
 |
 | IAM
 v
Bucket
```

Even if network connectivity works:

```text
AccessDenied
```

can still happen.

Conversely, IAM may permit the action but network connectivity can fail.

---

# 48. Troubleshooting AWS Access From Pod

Suppose:

```text
order-service
```

returns:

```text
AccessDenied
```

Check:

```text
1. Pod Identity association
2. ServiceAccount
3. IAM role
4. IAM policy
5. Resource ARN
6. Region
7. Network connectivity
```

Don't modify the Security Group for an IAM error.

---

# 49. Verify ServiceAccount

```bash
kubectl get serviceaccount order-service \
  -n ecommerce-prod \
  -o yaml
```

Then verify the Pod:

```bash
kubectl get pod <POD_NAME> \
  -n ecommerce-prod \
  -o yaml
```

Confirm it uses:

```yaml
serviceAccountName: order-service
```

---

# 50. Verify Pod Identity

Architecture:

```text
Pod
 ↓
ServiceAccount
 ↓
EKS Pod Identity
 ↓
IAM Role
 ↓
AWS API
```

Check the configured association using the AWS CLI or EKS management configuration for your cluster.

The exact command should match the Pod Identity implementation/version used in your environment.

---

# 51. Security Principle

Every microservice should have:

```text
Own ServiceAccount
Own IAM role where needed
Own permissions
```

Avoid:

```text
All 21 services
      ↓
One IAM role
      ↓
AdministratorAccess
```

Instead:

```text
order-service
   ↓
order-service-role

payment-service
   ↓
payment-service-role

inventory-service
   ↓
inventory-service-role
```

---

# 52. Network Segmentation

We can logically divide:

```text
Internet-facing
    |
    v
Ingress layer

Application
    |
    v
Microservices

Data
    |
    v
RDS / Redis
```

A microservice should not automatically have access to:

```text
Every database
Every queue
Every service
```

Least privilege applies to network communication too.

---

# 53. Example: Order Service

Suppose `order-service` needs:

```text
inventory-service
payment-service
notification-service
RDS
Redis
SQS
Secrets Manager
```

Its allowed dependencies are:

```text
order
 |
 +--> inventory
 |
 +--> payment
 |
 +--> notification
 |
 +--> RDS
 |
 +--> Redis
 |
 +--> SQS
 |
 +--> Secrets Manager
```

Not:

```text
order-service
 |
 +--> every microservice
 +--> every database
 +--> every queue
```

---

# 54. Example Communication Diagram

```text
                 ALB
                  |
                  v
            order-service
             /     |      \
            /      |       \
           v       v        v
     inventory   payment   notification
           |
           v
          RDS
           |
         Redis
```

Each arrow should represent an intentional dependency.

---

# 55. Network Security Repository

We can place reusable network policies in:

```text
platform/
├── network-policies/
│   ├── default-deny.yaml
│   ├── allow-dns.yaml
│   ├── order-service.yaml
│   ├── payment-service.yaml
│   └── inventory-service.yaml
```

Or manage them through Helm/platform charts depending on your repository strategy.

The important principle is:

```text
NetworkPolicy
    ↓
Version controlled
    ↓
Reviewed
    ↓
GitOps
```

---

# 56. Don't Store Network Policies Manually on the Cluster

Avoid:

```bash
kubectl apply -f network-policy.yaml
```

as your normal production process.

Use:

```text
Git
 ↓
PR
 ↓
Review
 ↓
ArgoCD
 ↓
EKS
```

This gives us:

```text
Auditability
Rollback
Review
Consistency
```

---

# 57. GitOps Structure

We can extend:

```text
gitops/
```

with:

```text
gitops/
├── environments/
│   ├── dev/
│   ├── qa/
│   ├── staging/
│   └── prod/
│
├── platform/
│   ├── network-policies/
│   ├── ingress/
│   └── security/
│
└── argocd/
```

Platform resources should be separated from individual application configuration where practical.

---

# 58. Security Policy Testing

Before applying a NetworkPolicy:

```text
Expected:
order-service → inventory = ALLOWED

Expected:
order-service → random-service = DENIED
```

Test from a Pod:

```bash
curl http://inventory-service:8080/health
```

Then test an intentionally unauthorized service:

```bash
curl http://some-other-service:8080/health
```

Expected:

```text
inventory → success
unauthorized → blocked
```

---

# 59. NetworkPolicy Doesn't Replace Authentication

Suppose:

```text
order-service
```

is allowed to communicate with:

```text
payment-service
```

That doesn't mean every request should automatically be trusted.

Application-level controls may still require:

```text
TLS
JWT
mTLS
Service authentication
Authorization
```

NetworkPolicy answers:

> Can this network connection occur?

Application authentication answers:

> Is this request authorized?

---

# 60. Production Security Layers

For our platform:

```text
                    Internet
                       |
                     WAF
                       |
                      ALB
                       |
                 Security Group
                       |
                    EKS
                       |
              NetworkPolicy
                       |
                     Pod
                       |
                Application Auth
                       |
                   IAM/Pod ID
                       |
               AWS Resource
```

This is defense in depth.

---

# 61. Network Troubleshooting Decision Tree

When an application cannot connect:

```text
Connection failed
       |
       v
DNS works?
   |
  NO → CoreDNS / DNS policy
   |
  YES
   |
   v
Route exists?
   |
  NO → Route table / subnet
   |
  YES
   |
   v
NACL allows?
   |
  NO → NACL
   |
  YES
   |
   v
Security Group allows?
   |
  NO → SG
   |
  YES
   |
   v
NetworkPolicy allows?
   |
  NO → NetworkPolicy
   |
  YES
   |
   v
Service/Port correct?
   |
  NO → Kubernetes config
   |
  YES
   |
   v
Application healthy?
```

---

# 62. Production Example

Incident:

```text
Order creation is failing.
```

Application log:

```text
Connection timeout to inventory-service
```

Don't immediately restart Pods.

Investigate:

```bash
kubectl get svc inventory-service \
  -n ecommerce-prod
```

Then:

```bash
kubectl get endpointslice \
  -n ecommerce-prod \
  -l kubernetes.io/service-name=inventory-service
```

Then DNS:

```bash
kubectl exec -it <order-pod> \
  -n ecommerce-prod \
  -- getent hosts inventory-service
```

Then connectivity:

```bash
kubectl exec -it <debug-pod> \
  -n ecommerce-prod \
  -- nc -vz inventory-service 8080
```

Then NetworkPolicy:

```bash
kubectl get networkpolicy \
  -n ecommerce-prod
```

Then application logs on Inventory.

This gives you a systematic investigation rather than random changes.

---

# 63. Production Network Checklist

## VPC

```text
[ ] Private EKS nodes
[ ] Correct subnets
[ ] Correct route tables
[ ] NAT design
[ ] VPC endpoints where required
[ ] DNS enabled
[ ] IP capacity planned
```

## AWS Security

```text
[ ] ALB SG restricted appropriately
[ ] EKS network controls
[ ] RDS SG private
[ ] Redis SG private
[ ] No public database
[ ] NACLs reviewed
```

## Kubernetes

```text
[ ] Namespace isolation
[ ] NetworkPolicies
[ ] Default deny where appropriate
[ ] DNS explicitly permitted
[ ] Service-to-service policies
[ ] Egress policies tested
```

## IAM

```text
[ ] Pod Identity
[ ] Service-specific roles
[ ] Least privilege
[ ] No static AWS keys
[ ] Resource-level permissions
```

---

# 64. Final Production Network Architecture

```text
                         INTERNET
                            |
                         Route 53
                            |
                      CloudFront/WAF
                            |
                           ALB
                            |
                    ALB Security Group
                            |
                            v
                    +---------------+
                    |      EKS      |
                    |               |
                    |  Private      |
                    |  Nodes        |
                    +-------+-------+
                            |
                     NetworkPolicy
                            |
             +--------------+--------------+
             |              |              |
       order-service   payment-service  inventory
             |              |              |
             +--------------+--------------+
                            |
             +--------------+--------------+
             |              |              |
            RDS           Redis           SQS
             |              |              |
        Security Group   Security Group    IAM
             |              |              |
             +--------------+--------------+
                            |
                      Secrets Manager
                            |
                       Pod Identity
```

---

# 65. What This Chapter Added

Our project now has a much more realistic network model:

```text
Internet
   ↓
WAF
   ↓
ALB
   ↓
Private EKS
   ↓
NetworkPolicy
   ↓
Microservices
   ↓
Private data layer
```

And security is split correctly:

```text
Security Groups
    ↓
AWS network boundary

NetworkPolicy
    ↓
Pod-to-Pod boundary

IAM
    ↓
AWS resource authorization

RBAC
    ↓
Kubernetes API authorization

Application Auth
    ↓
Business/API authorization
```

---

# 66. Next Chapter

The next logical implementation is:

# **86 — Production Kubernetes Security Hardening**

We'll take the running EKS cluster and harden the actual workloads:

```text
Pod Security Standards
SecurityContext
runAsNonRoot
readOnlyRootFilesystem
capabilities
seccomp
privileged containers
hostNetwork
hostPID
service accounts
RBAC
Secrets
admission policies
image signing
SBOM
Trivy
SonarQube
Gitleaks
ECR scanning
```

and connect those controls to the CI → ECR → GitOps → ArgoCD → EKS pipeline we've already built.
