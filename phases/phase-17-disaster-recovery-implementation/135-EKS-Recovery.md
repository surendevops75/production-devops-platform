# 135 — EKS RECOVERY

## 1. Objective

This chapter covers how to recover the **production EKS platform** when the Kubernetes environment suffers a major failure.

The goal is not simply:

> "Create another EKS cluster."

The real production recovery process is:

```text
Failure
   ↓
Assess impact
   ↓
Identify failure scope
   ↓
Protect data and dependencies
   ↓
Recover EKS infrastructure
   ↓
Recover Kubernetes platform components
   ↓
Bootstrap ArgoCD
   ↓
Restore GitOps-managed platform
   ↓
Restore 21 microservices
   ↓
Reconnect databases/messaging/cache
   ↓
Validate traffic
   ↓
Validate business transactions
   ↓
Declare recovery complete
```

The key principle is:

> **EKS is disposable infrastructure; application configuration should be reproducible from Terraform + GitOps + container images + managed data services.**

---

# 2. EKS Recovery Is Different From VM Recovery

With a traditional VM:

```text
VM backup
   ↓
Restore VM
   ↓
Start application
```

With EKS:

```text
Terraform
   ↓
VPC
   ↓
EKS
   ↓
Node infrastructure
   ↓
EKS add-ons
   ↓
ArgoCD
   ↓
Platform components
   ↓
Applications
```

The Kubernetes control plane of EKS is AWS-managed.

You don't normally take an etcd snapshot of the EKS control plane and restore it yourself like you would with a self-managed Kubernetes cluster.

Therefore, our recovery strategy is based on **reconstruction**, not traditional control-plane backup restoration.

---

# 3. Production Architecture

Our application platform:

```text
                         Internet
                            |
                     Route 53 / DNS
                            |
                     CloudFront / WAF
                            |
                           ALB
                            |
                     AWS Load Balancer
                        Controller
                            |
                    +-------+-------+
                    |               |
                 EKS AZ-A        EKS AZ-B
                    |               |
                 Nodes            Nodes
                    |               |
                    +-------+-------+
                            |
                    EKS AZ-C
                            |
                   21 Microservices
                            |
        +-------------------+-------------------+
        |                   |                   |
       RDS                Redis             Messaging
    PostgreSQL          ElastiCache       SQS/SNS/Kafka
        |                   |                   |
        +-------------------+-------------------+
                            |
                       Observability
                            |
             Prometheus / Grafana / ELK
                            |
                   OpenTelemetry / Jaeger
```

Infrastructure:

```text
Terraform
   |
   +--> VPC
   +--> EKS
   +--> IAM
   +--> ECR
   +--> RDS
   +--> Redis
   +--> SQS/SNS
   +--> S3
   +--> KMS
```

Application deployment:

```text
GitHub
   |
   +--> CI
   |
   +--> ECR
   |
GitOps Repository
   |
   +--> ArgoCD
          |
          +--> EKS
```

---

# 4. What Can Fail?

EKS recovery must distinguish between different failure scopes.

## Scenario 1 — Pod failure

```text
Pod failure
   ↓
Kubernetes restarts Pod
```

This is not an EKS recovery event.

---

## Scenario 2 — Node failure

```text
EC2 node fails
   ↓
Kubernetes reschedules Pods
   ↓
Karpenter / Auto Scaling replaces capacity
```

Usually no cluster recovery is required.

---

## Scenario 3 — AZ failure

```text
AZ-A unavailable
   ↓
Workloads in AZ-A lost
   ↓
Pods rescheduled to AZ-B/AZ-C
```

Multi-AZ architecture handles much of this.

---

## Scenario 4 — EKS add-on failure

Examples:

```text
VPC CNI failure
CoreDNS failure
kube-proxy failure
EBS CSI failure
AWS Load Balancer Controller failure
```

The cluster exists, but platform functionality is degraded.

---

## Scenario 5 — EKS cluster configuration corruption

Examples:

```text
Incorrect RBAC
Broken admission configuration
NetworkPolicy mistakes
Broken add-on configuration
Bad platform deployment
```

The cluster may still be running, but recovery may require configuration rollback or reconstruction.

---

## Scenario 6 — Complete cluster loss

```text
EKS cluster unavailable
   ↓
Create replacement cluster
   ↓
Bootstrap platform
   ↓
ArgoCD
   ↓
Applications
```

This is the primary scenario for this chapter.

---

# 5. Recovery Philosophy

Our recovery design follows:

```text
Git = desired state
Terraform = infrastructure state
ECR = application artifacts
RDS = persistent database
Redis = recoverable cache
SQS/Kafka = messaging state
Secrets Manager = secrets
S3 = object data
ArgoCD = Kubernetes reconciliation
```

Therefore:

```text
New EKS cluster
       |
       v
Same Terraform
       |
       v
Same GitOps repository
       |
       v
Same ECR images
       |
       v
Same managed data services
       |
       v
Same application architecture
```

This is why infrastructure-as-code and GitOps are extremely important for disaster recovery.

---

# 6. What Should NOT Be the Source of Truth?

Do not depend on:

```text
kubectl edit
kubectl apply from laptop
Manual Helm installation
Manually created namespaces
Manually created ServiceAccounts
Manually created ConfigMaps
Manually configured applications
```

For example:

```bash
kubectl apply -f production.yaml
```

may work temporarily.

But if the cluster disappears:

```text
Where is production.yaml?
```

If the answer is:

> "It is on an engineer's laptop."

the recovery process is weak.

Instead:

```text
Git
 |
 +--> Kubernetes manifests
 +--> Helm values
 +--> ArgoCD Applications
 +--> NetworkPolicies
 +--> ServiceAccounts
 +--> monitoring
 +--> platform configuration
```

---

# 7. Recovery Dependencies

Before recovering EKS, identify external dependencies.

```text
                    EKS Recovery
                         |
       +-----------------+------------------+
       |                 |                  |
      VPC               IAM               ECR
       |                 |                  |
       +-----------------+------------------+
                         |
                       EKS
                         |
       +-----------------+------------------+
       |                 |                  |
      RDS              Redis             Messaging
       |                 |                  |
       +-----------------+------------------+
                         |
                       Apps
```

Important dependencies:

### Infrastructure

```text
AWS account
Terraform
Terraform state
VPC
Subnets
Route tables
NAT
Security Groups
IAM
KMS
```

### Application

```text
ECR
GitHub
GitOps repository
Helm charts
ArgoCD
```

### Data

```text
RDS
Redis
S3
SQS
SNS
Kafka
```

### Security

```text
IAM
KMS
Secrets Manager
Pod Identity
NetworkPolicy
```

---

# 8. Terraform State Is Critical

Our Terraform state contains the information Terraform needs to manage infrastructure.

Example:

```text
S3
 |
 +-- company-ecommerce-prod-terraform-state
       |
       +-- eks/
       +-- vpc/
       +-- rds/
       +-- redis/
       +-- iam/
```

Example backend:

```hcl
terraform {
  backend "s3" {
    bucket       = "company-ecommerce-prod-terraform-state"
    key          = "prod/eks/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

The state bucket itself must be protected.

Recommended controls:

```text
S3 Versioning
Encryption
Public Access Block
Restricted IAM
CloudTrail
Deletion protection through organizational controls where applicable
Backup/replication strategy
```

---

# 9. EKS Recovery Levels

We can define four recovery levels.

## Level 1 — Pod Recovery

```text
Pod failed
↓
Deployment recreates Pod
```

---

## Level 2 — Node Recovery

```text
Node failed
↓
New node
↓
Pods rescheduled
```

---

## Level 3 — Platform Recovery

```text
Add-on/platform component failed
↓
Restore GitOps configuration
↓
ArgoCD reconciles
```

---

## Level 4 — Cluster Recovery

```text
Cluster unavailable
↓
Terraform creates new EKS
↓
Bootstrap
↓
ArgoCD
↓
Platform
↓
Applications
```

---

# 10. Complete Cluster Recovery Flow

The recovery order should be:

```text
1. Incident declaration
        ↓
2. Determine failure scope
        ↓
3. Verify AWS infrastructure
        ↓
4. Verify data services
        ↓
5. Verify Terraform state
        ↓
6. Recover VPC if necessary
        ↓
7. Create/recover EKS
        ↓
8. Install/enable EKS add-ons
        ↓
9. Verify node capacity
        ↓
10. Bootstrap ArgoCD
        ↓
11. Restore platform services
        ↓
12. Restore namespaces
        ↓
13. Restore secrets integration
        ↓
14. Restore ingress
        ↓
15. Restore microservices
        ↓
16. Restore autoscaling
        ↓
17. Restore observability
        ↓
18. Validate dependencies
        ↓
19. Validate application
        ↓
20. Validate business transactions
        ↓
21. Restore normal traffic
```

---

# 11. Step 1 — Declare Incident

Example:

```text
INCIDENT: Production EKS Cluster Failure

Severity: SEV-1

Impact:
Production API unavailable

Start:
10:15 IST

Incident Commander:
Platform Engineering

Application Owner:
Backend Team

Infrastructure Owner:
DevOps Team

Communication:
Incident channel
```

Immediately establish:

```text
Who is incident commander?
Who owns infrastructure?
Who owns application?
Who owns database?
Who communicates status?
```

---

# 12. Step 2 — Determine Failure Scope

Do not immediately destroy and recreate infrastructure.

First determine:

```text
Is EKS API reachable?
Are nodes healthy?
Are Pods running?
Is ALB healthy?
Is Route 53 healthy?
Is RDS healthy?
Is Redis healthy?
Is messaging healthy?
```

Example:

```bash
aws eks describe-cluster \
  --name ecommerce-prod \
  --region ap-south-1
```

Then:

```bash
kubectl get nodes
```

```bash
kubectl get pods -A
```

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

---

# 13. If EKS API Is Reachable

Do not recreate the cluster automatically.

Investigate:

```text
EKS API
   |
   +--> Nodes?
   |
   +--> CNI?
   |
   +--> CoreDNS?
   |
   +--> kube-proxy?
   |
   +--> CSI?
   |
   +--> ALB Controller?
   |
   +--> ArgoCD?
```

Example:

```bash
kubectl get nodes
```

```bash
kubectl get pods -A
```

```bash
kubectl get daemonsets -A
```

```bash
kubectl get deployments -A
```

---

# 14. EKS System Components

Important components include:

```text
VPC CNI
CoreDNS
kube-proxy
EBS CSI Driver
AWS Load Balancer Controller
```

Check:

```bash
kubectl get pods -n kube-system
```

Example:

```text
aws-node-xxxxx
coredns-xxxxx
kube-proxy-xxxxx
ebs-csi-node-xxxxx
```

If these are unhealthy, application symptoms can appear even though the control plane itself is healthy.

---

# 15. VPC CNI Failure

The VPC CNI provides networking for Pods.

Typical symptoms:

```text
Pod stuck ContainerCreating
Pod cannot receive IP
Pod-to-Pod connectivity failures
IP exhaustion
```

Check:

```bash
kubectl get pods -n kube-system -l k8s-app=aws-node
```

Logs:

```bash
kubectl logs -n kube-system \
  -l k8s-app=aws-node \
  --tail=200
```

Investigate:

```text
Subnet IP capacity
ENI capacity
Security groups
IAM permissions
CNI configuration
Node instance type
```

This is especially important for a 21-service architecture because Pod density can become an IP-capacity problem.

---

# 16. CoreDNS Failure

Symptoms:

```text
Service-to-service communication fails
External DNS resolution fails
Applications show connection errors
```

Check:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Test:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --rm -it \
  --restart=Never \
  -- nslookup kubernetes.default
```

Also check:

```bash
kubectl get svc -n kube-system kube-dns
```

---

# 17. kube-proxy Failure

Symptoms can include:

```text
Service networking problems
Node-level service routing problems
Intermittent connectivity
```

Check:

```bash
kubectl get daemonset -n kube-system kube-proxy
```

Then inspect logs for unhealthy Pods.

---

# 18. EBS CSI Failure

If workloads use EBS-backed persistent volumes:

```text
Pod
 |
 PVC
 |
 PV
 |
 EBS
```

Check:

```bash
kubectl get pods -n kube-system | grep ebs
```

```bash
kubectl get pvc -A
```

```bash
kubectl get pv
```

Important:

> Recreating the EKS cluster does not mean deleting EBS volumes.

Persistent storage must be handled deliberately.

---

# 19. AWS Load Balancer Controller Failure

Symptoms:

```text
Ingress exists
but ALB missing/unhealthy
```

Check:

```bash
kubectl get deployment \
  -n kube-system \
  aws-load-balancer-controller
```

Then:

```bash
kubectl logs \
  -n kube-system \
  deployment/aws-load-balancer-controller \
  --tail=200
```

Check:

```bash
kubectl get ingress -A
```

The controller uses AWS APIs to create/manage ALBs and target groups.

Therefore verify:

```text
IAM
Pod Identity
Subnet tags
Security groups
Ingress configuration
```

---

# 20. When Should We Recreate the Cluster?

A cluster reconstruction may be appropriate when:

```text
Control plane/platform state is unrecoverable
Cluster configuration is severely corrupted
Recovery time of repair exceeds rebuild time
Security incident requires clean-cluster reconstruction
Major infrastructure drift exists
Cluster must be replaced as part of DR
```

Decision:

```text
Repair existing cluster
        OR
Rebuild cluster
```

This should be based on the incident runbook rather than instinct.

---

# 21. Cluster Rebuild Architecture

New cluster:

```text
Terraform
    |
    +--> EKS
    |
    +--> Node Groups / Karpenter
    |
    +--> IAM
    |
    +--> Security Groups
    |
    +--> Add-ons
            |
            +--> VPC CNI
            +--> CoreDNS
            +--> kube-proxy
            +--> EBS CSI
```

Then:

```text
ArgoCD
   |
   +--> platform
   |
   +--> observability
   |
   +--> ingress
   |
   +--> security
   |
   +--> applications
```

---

# 22. Terraform Recovery

Move to the infrastructure repository:

```bash
cd infrastructure/terraform/environments/prod
```

Initialize:

```bash
terraform init
```

Verify:

```bash
terraform validate
```

Plan:

```bash
terraform plan
```

Do not immediately run:

```bash
terraform apply -auto-approve
```

in a production recovery unless that behavior is explicitly part of your approved recovery procedure.

First review:

```text
What will be created?
What will be modified?
What will be destroyed?
Are RDS resources affected?
Are security groups affected?
Are existing dependencies referenced?
```

---

# 23. Important Terraform Recovery Rule

If RDS already exists:

```text
DO NOT recreate RDS
```

unless the recovery plan specifically requires it.

Similarly:

```text
DO NOT recreate:
RDS
Redis
S3
SQS
SNS
Kafka
KMS keys
Secrets
```

just because EKS is being rebuilt.

The application cluster and persistent services have different lifecycles.

---

# 24. Recreate EKS

Conceptually:

```text
VPC
 |
 +--> Private subnets
 |
 +--> IAM
 |
 +--> EKS
 |
 +--> Node capacity
```

After Terraform:

```bash
aws eks describe-cluster \
  --name ecommerce-prod \
  --region ap-south-1
```

Then configure access:

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name ecommerce-prod
```

Verify:

```bash
kubectl get nodes
```

---

# 25. Verify Node Capacity

Example:

```bash
kubectl get nodes -o wide
```

Check:

```text
Node status
Instance type
AZ
CPU
Memory
Pod capacity
```

Then:

```bash
kubectl describe nodes
```

Look at:

```text
Allocatable
Capacity
Conditions
Taints
PodCIDR
```

---

# 26. Karpenter Recovery

If Karpenter is part of the platform:

```text
Pending Pod
     |
     v
Karpenter
     |
     v
Select instance
     |
     v
Launch node
     |
     v
Schedule Pod
```

After rebuilding the cluster:

```bash
kubectl get pods -n kube-system
```

Verify Karpenter.

Then check:

```bash
kubectl get nodeclaims
```

or the equivalent resources for the installed Karpenter version.

> Karpenter APIs and resource names can change between releases, so always use the version-specific documentation for the version deployed in the cluster.

---

# 27. Bootstrap ArgoCD

ArgoCD is critical because it becomes the deployment engine.

Architecture:

```text
GitOps Repository
       |
       v
     ArgoCD
       |
       v
      EKS
```

After the cluster is ready:

```text
Install ArgoCD
       |
       v
Configure repository access
       |
       v
Deploy root Application / ApplicationSet
       |
       v
Platform components
       |
       v
Applications
```

---

# 28. Why ArgoCD Comes Early

Without ArgoCD:

```text
Engineer
   |
   +--> manually apply 100s of manifests
```

With ArgoCD:

```text
Git
 |
 v
ArgoCD
 |
 +--> namespaces
 +--> service accounts
 +--> network policies
 +--> deployments
 +--> services
 +--> ingress
 +--> HPA
 +--> monitoring
```

Therefore ArgoCD dramatically reduces recovery effort.

---

# 29. GitOps Recovery Order

Our GitOps hierarchy:

```text
gitops/
├── platform/
│   ├── argocd/
│   ├── ingress/
│   ├── external-secrets/
│   ├── monitoring/
│   ├── logging/
│   ├── opentelemetry/
│   └── jaeger/
│
├── environments/
│   ├── dev/
│   ├── qa/
│   ├── staging/
│   └── prod/
│
└── applications/
    ├── auth-service/
    ├── user-service/
    ├── customer-service/
    └── ...
```

Recovery:

```text
ArgoCD
   ↓
Platform
   ↓
Security
   ↓
Ingress
   ↓
Observability
   ↓
Applications
```

---

# 30. Namespace Recovery

Example:

```text
ecommerce-prod
monitoring
logging
tracing
argocd
external-secrets
ingress
```

Check:

```bash
kubectl get namespaces
```

Namespaces should preferably be defined through GitOps.

Example:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ecommerce-prod
```

---

# 31. Secrets Recovery

Application secrets should not be stored directly in Git.

Architecture:

```text
AWS Secrets Manager
        |
        v
External Secrets Operator
        |
        v
Kubernetes Secret
        |
        v
Application Pod
```

After cluster recovery:

```text
ESO
 |
 +--> authenticate to AWS
 |
 +--> read Secrets Manager
 |
 +--> create Kubernetes Secret
```

Check:

```bash
kubectl get pods -n external-secrets
```

Then:

```bash
kubectl get externalsecrets -A
```

And:

```bash
kubectl get secrets -n ecommerce-prod
```

Do not print secret values during incident investigation.

---

# 32. IAM Recovery

Check workload identity:

```text
Pod
 |
 v
EKS Pod Identity
 |
 v
IAM Role
 |
 +--> RDS-related secret
 +--> SQS
 +--> S3
 +--> KMS
```

Typical recovery problem:

```text
Pod running
but AWS API returns AccessDenied
```

Investigate:

```text
ServiceAccount
Pod Identity association
IAM role
IAM policy
resource ARN
AWS region
```

---

# 33. NetworkPolicy Recovery

NetworkPolicies may initially block application traffic.

Example:

```text
order-service
      |
      +--> customer-service
      +--> payment-service
      +--> inventory-service
      +--> PostgreSQL
```

If NetworkPolicy is restored incorrectly:

```text
Pod = Running
Service = Running
Application = Broken
```

Therefore test:

```text
DNS
Service connectivity
Database connectivity
Messaging connectivity
External APIs
```

---

# 34. RDS Recovery Dependency

During EKS recovery:

```text
EKS
 |
 +--> order-service
 |
 +--> RDS endpoint
```

RDS is outside the EKS cluster.

The new Pods should connect to the same RDS endpoint.

Example:

```text
order-service
    |
    v
orders-prod.xxxxxx.ap-south-1.rds.amazonaws.com
```

The endpoint is provided through configuration/secrets rather than hard-coded Pod IP addresses.

---

# 35. Redis Recovery Dependency

Same principle:

```text
EKS
 |
 +--> applications
          |
          v
     ElastiCache
```

Redis should not become a hidden requirement for application startup unless the application actually requires it.

For cache-aside:

```text
Request
  |
  v
Redis
  |
  +-- HIT --> Response
  |
  +-- MISS
       |
       v
      RDS
       |
       v
      Redis
```

If Redis is unavailable, applications should follow their designed fallback behavior.

---

# 36. SQS Recovery

SQS exists outside EKS.

Therefore:

```text
EKS destroyed
      |
      X
      |
SQS continues existing
```

After the new consumers start:

```text
SQS
 |
 v
notification-service
```

Messages may still be available depending on retention and visibility state.

Recovery should therefore verify:

```text
Queue exists
Messages exist
DLQ exists
IAM works
Consumers connect
Backlog drains
```

---

# 37. Kafka Recovery

Kafka/MSK is similarly an external dependency if using Amazon MSK.

```text
EKS
 |
 +--> Kafka consumers/producers
             |
             v
            MSK
```

Recreated Pods must restore:

```text
Bootstrap servers
Authentication
TLS
Topic configuration
Consumer groups
IAM/credentials
```

Important:

> Consumer offsets are messaging-system state, not Kubernetes Pod state.

---

# 38. Restore Ingress

After the application is deployed:

```text
Ingress
   |
   v
AWS Load Balancer Controller
   |
   v
ALB
   |
   v
Services
```

Check:

```bash
kubectl get ingress -A
```

Then verify:

```text
ALB exists
Listeners exist
Target groups exist
Targets healthy
Security groups correct
TLS certificate correct
DNS points correctly
```

---

# 39. Restore the 21 Microservices

Applications:

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

ArgoCD should progressively deploy them.

---

# 40. Application Recovery Order

Not every service needs to start simultaneously.

A dependency-aware order can be:

```text
1. auth-service
2. user-service
3. customer-service
4. catalog-service
5. product-service
6. pricing-service
7. promotion-service
8. inventory-service
9. cart-service
10. order-service
11. payment-service
12. fraud-service
13. shipping-service
14. delivery-service
15. notification-service
16. search-service
17. recommendation-service
18. review-service
19. tax-service
20. invoice-service
21. reporting-service
```

However, because many services are independently deployable, the actual order should be driven by **real dependency relationships**, not simply this list.

---

# 41. ArgoCD Application Health

Check:

```bash
argocd app list
```

Expected:

```text
NAME
STATUS
HEALTH
SYNC
```

Example:

```text
order-service
Synced
Healthy
```

If:

```text
Synced
Degraded
```

Git state matches the cluster, but the application itself is unhealthy.

---

# 42. Recovery Verification Matrix

| Layer            | Verification                |
| ---------------- | --------------------------- |
| AWS              | Account/resources available |
| VPC              | Subnets/routes/NAT          |
| IAM              | Roles and Pod Identity      |
| EKS              | Control plane healthy       |
| Nodes            | Ready                       |
| CNI              | Healthy                     |
| CoreDNS          | Healthy                     |
| kube-proxy       | Healthy                     |
| EBS CSI          | Healthy                     |
| Karpenter        | Healthy                     |
| ALB Controller   | Healthy                     |
| ArgoCD           | Healthy                     |
| External Secrets | Healthy                     |
| Ingress          | Healthy                     |
| Applications     | Healthy                     |
| RDS              | Reachable                   |
| Redis            | Reachable                   |
| SQS              | Consumers active            |
| Kafka            | Consumers active            |
| Prometheus       | Scraping                    |
| Grafana          | Dashboards                  |
| ELK              | Logs arriving               |
| OTel             | Telemetry flowing           |
| Jaeger           | Traces visible              |

---

# 43. Application Health Checks

Example:

```bash
kubectl get pods -n ecommerce-prod
```

Then:

```bash
kubectl get svc -n ecommerce-prod
```

Then:

```bash
kubectl get ingress -n ecommerce-prod
```

Check deployment:

```bash
kubectl rollout status \
  deployment/order-service \
  -n ecommerce-prod
```

---

# 44. Business Validation

Infrastructure being green does **not** mean recovery is complete.

We need a business transaction.

Example:

```text
Customer
   |
   v
Login
   |
   v
Browse product
   |
   v
Add to cart
   |
   v
Create order
   |
   v
Inventory reservation
   |
   v
Payment authorization
   |
   v
Order confirmation
   |
   v
Notification
```

This is the actual recovery test.

---

# 45. Example Order Recovery Test

Create test order:

```http
POST /api/v1/orders
```

Verify:

```text
HTTP response
Database record
Inventory event
Payment event
Order event
Notification
Logs
Metrics
Trace
```

Then search:

```text
correlationId
```

in:

```text
Grafana
Kibana
Jaeger
```

This verifies all three observability pillars.

---

# 46. End-to-End Recovery Trace

Expected:

```text
Client
 |
 v
ALB
 |
 v
order-service
 |
 +--> customer-service
 |
 +--> product-service
 |
 +--> pricing-service
 |
 +--> tax-service
 |
 v
PostgreSQL
 |
 v
Outbox
 |
 v
Kafka
 |
 +--> inventory-service
 |
 +--> payment-service
 |
 +--> fraud-service
 |
 +--> reporting-service
 |
 +--> notification-service
```

The same `correlationId` should allow investigation across the workflow.

---

# 47. RTO

Recovery Time Objective answers:

> How long can the service remain unavailable?

Example target:

```text
RTO = 60 minutes
```

This means the recovery exercise should demonstrate whether the platform can restore the required service within approximately that objective.

Do not simply write:

```text
RTO = 60 minutes
```

and assume it is true.

Measure it.

---

# 48. RPO

Recovery Point Objective answers:

> How much data loss is acceptable?

Example:

```text
RPO = 5 minutes
```

For EKS itself, most Kubernetes application configuration is reconstructed from Git.

For persistent data:

```text
RDS
S3
Kafka
SQS
```

their individual recovery characteristics determine the effective data recovery point.

---

# 49. Recovery Timeline

Record timestamps.

Example:

```text
10:15 - Incident detected
10:18 - Incident declared
10:22 - Failure scope identified
10:27 - EKS rebuild started
10:39 - EKS available
10:44 - Nodes ready
10:49 - ArgoCD ready
10:55 - Platform components healthy
11:05 - Applications deployed
11:12 - Ingress healthy
11:18 - Database connectivity verified
11:22 - Business transaction successful
11:25 - Traffic restored
```

Calculate:

```text
Total recovery time
```

Then compare:

```text
Actual RTO
vs
Target RTO
```

---

# 50. Controlled EKS Recovery Exercise

Do this only in a dedicated DR/test environment unless an approved production game day explicitly permits it.

Example:

```text
Production-like DR environment
        |
        v
Create failure
        |
        v
Recover EKS
        |
        v
Measure RTO
        |
        v
Document gaps
```

Never use destructive commands against production simply to "practice recovery."

---

# 51. EKS Recovery Game Day

Scenario:

```text
Simulated EKS cluster loss
```

Teams:

```text
Incident Commander
DevOps
Platform Engineering
Backend
Database
Security
SRE
```

Objective:

```text
Restore customer checkout
```

Success criteria:

```text
EKS available
21 services healthy
RDS connected
Redis functional
Messaging functional
ALB healthy
DNS healthy
Observability functional
Checkout successful
RTO achieved
```

---

# 52. Failure Injection

Possible controlled failures:

### Node failure

```text
Terminate test node
```

Observe:

```text
Pod rescheduling
Karpenter
HPA
PDB
```

### Application failure

```text
Deploy known-bad image
```

Observe:

```text
CrashLoopBackOff
Alert
Rollback
```

### Platform failure

```text
Scale test controller deployment down
```

Observe:

```text
Alert
Recovery
```

### Cluster rebuild

```text
Destroy/recreate dedicated DR EKS
```

Observe complete recovery.

---

# 53. EKS Recovery Dependency Graph

```text
Terraform State
       |
       v
      VPC
       |
       v
      IAM
       |
       v
      EKS
       |
       +-------------------+
       |                   |
       v                   v
  Node Capacity        EKS Add-ons
                           |
                           +--> CNI
                           +--> CoreDNS
                           +--> kube-proxy
                           +--> CSI
                           +--> ALB Controller
                           |
                           v
                         ArgoCD
                           |
            +--------------+--------------+
            |              |              |
            v              v              v
        Security      Observability    Ingress
            |              |              |
            +--------------+--------------+
                           |
                           v
                     21 Microservices
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
         RDS             Redis          Messaging
```

---

# 54. What EKS Recovery Does NOT Recover

Rebuilding EKS does not automatically recover:

```text
RDS data
Redis data
S3 objects
Kafka data
SQS messages
Secrets
DNS configuration
KMS keys
ECR images
```

These are separate AWS resources and must have their own recovery strategies.

That is why our next recovery chapters are:

```text
135 EKS Recovery
136 RDS Recovery
137 S3 Recovery
138 ECR Recovery
139 Secrets/KMS Recovery
140 DNS Failover
141 Region Failure Simulation
142 Full DR Game Day
```

---

# 55. Recovery Checklist

## Infrastructure

```text
[ ] Terraform state available
[ ] VPC available
[ ] Subnets available
[ ] Routes available
[ ] NAT available
[ ] Security groups available
[ ] IAM available
[ ] KMS available
```

## EKS

```text
[ ] Cluster healthy
[ ] Nodes Ready
[ ] VPC CNI healthy
[ ] CoreDNS healthy
[ ] kube-proxy healthy
[ ] EBS CSI healthy
[ ] Karpenter healthy
[ ] ALB Controller healthy
```

## GitOps

```text
[ ] ArgoCD healthy
[ ] Git repository reachable
[ ] Applications synced
[ ] No unexpected OutOfSync resources
```

## Security

```text
[ ] Pod Identity working
[ ] External Secrets working
[ ] Secrets available
[ ] NetworkPolicies correct
```

## Application

```text
[ ] 21 services deployed
[ ] Pods healthy
[ ] Services healthy
[ ] Ingress healthy
[ ] HPA healthy
[ ] KEDA healthy
```

## Dependencies

```text
[ ] RDS reachable
[ ] Redis reachable
[ ] SQS reachable
[ ] SNS reachable
[ ] Kafka reachable
[ ] S3 reachable
```

## Observability

```text
[ ] Prometheus healthy
[ ] Grafana healthy
[ ] Alertmanager healthy
[ ] Fluent Bit healthy
[ ] Logstash healthy
[ ] Elasticsearch/OpenSearch healthy
[ ] Kibana healthy
[ ] OTel Collector healthy
[ ] Jaeger healthy
```

## Business

```text
[ ] Login works
[ ] Product browsing works
[ ] Cart works
[ ] Order creation works
[ ] Inventory workflow works
[ ] Payment workflow works
[ ] Notification works
[ ] End-to-end trace exists
```

---

# 56. Production Recovery Runbook

```text
EKS FAILURE
     |
     v
Declare Incident
     |
     v
Check EKS API
     |
     +---- YES ----> Investigate Nodes/Add-ons/Platform
     |
     NO
     |
     v
Check AWS / Terraform / Dependencies
     |
     v
Rebuild EKS
     |
     v
Verify Nodes
     |
     v
Verify CNI/CoreDNS/kube-proxy/CSI
     |
     v
Install/Recover ALB Controller
     |
     v
Bootstrap ArgoCD
     |
     v
Restore Platform
     |
     v
Restore Secrets
     |
     v
Restore Ingress
     |
     v
Deploy 21 Services
     |
     v
Connect RDS/Redis/Messaging
     |
     v
Restore Autoscaling
     |
     v
Restore Observability
     |
     v
Run Business Validation
     |
     v
Restore Traffic
     |
     v
Monitor
     |
     v
RCA + Corrective Actions
```

---

# 57. Key Production Lessons

### Lesson 1

**EKS control plane recovery is not the same as VM restoration.**

EKS is managed by AWS, so the recovery strategy focuses on infrastructure reconstruction and workload reconciliation.

### Lesson 2

**GitOps is a recovery mechanism.**

ArgoCD allows the Kubernetes desired state to be reconstructed from Git.

### Lesson 3

**Terraform is a recovery mechanism.**

Infrastructure can be rebuilt consistently.

### Lesson 4

**ECR provides immutable application artifacts.**

The replacement cluster does not need a new application build.

```text
Same Git SHA
      ↓
Same image
      ↓
New EKS
```

### Lesson 5

**Databases must have independent recovery plans.**

EKS recovery does not equal RDS recovery.

### Lesson 6

**Recovery isn't complete when Pods become Running.**

The real test is:

```text
Customer
   ↓
Application
   ↓
Database
   ↓
Messaging
   ↓
External dependencies
   ↓
Business transaction
```

### Lesson 7

**RTO must be measured.**

A documented RTO is only a target until a recovery exercise proves it.

---

# 58. Final Architecture

After recovery:

```text
                       Route 53
                          |
                     CloudFront/WAF
                          |
                         ALB
                          |
              +-----------+-----------+
              |           |           |
             AZ-A        AZ-B        AZ-C
              |           |           |
            Nodes       Nodes       Nodes
              |           |           |
              +-----------+-----------+
                          |
                         EKS
                          |
        +-----------------+-----------------+
        |                 |                 |
      HPA/KEDA         ArgoCD          Observability
        |                 |                 |
        +--------+--------+-----------------+
                 |
           21 Microservices
                 |
     +-----------+-----------+
     |           |           |
    RDS        Redis      Messaging
     |           |           |
     +-----------+-----------+
                 |
            Business Flow
```

The recovery architecture is therefore:

```text
Terraform
    ↓
AWS Infrastructure
    ↓
EKS
    ↓
ArgoCD
    ↓
Platform
    ↓
21 Microservices
    ↓
Managed Data Services
    ↓
Observability
    ↓
Business Validation
```

**Next chapter: `136-RDS-Recovery.md`**

That chapter will focus specifically on **PostgreSQL/RDS recovery**, including snapshot restore, point-in-time recovery, accidental deletion, corruption, Multi-AZ failure, restoring into a new VPC, reconnecting the 21 services, Flyway validation, RTO/RPO measurement, and a controlled RDS recovery game day.
