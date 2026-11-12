# 96 — EKS Upgrades and Cluster Lifecycle

> Production-grade guide for upgrading and operating an Amazon EKS cluster safely across Kubernetes versions, managed node groups, Karpenter nodes, and EKS add-ons.

---

## 1. Why EKS Upgrades Matter

An EKS upgrade is not simply:

```bash
aws eks update-cluster-version
```

A production EKS upgrade affects multiple layers:

```text
                 AWS
                  │
                  ▼
        ┌───────────────────┐
        │ EKS Control Plane │
        └─────────┬─────────┘
                  │
        ┌─────────┴─────────┐
        ▼                   ▼
 Managed Node Groups     Karpenter
        │                   │
        └─────────┬─────────┘
                  ▼
          Kubernetes Workloads
                  │
       ┌──────────┼──────────┐
       ▼          ▼          ▼
     CNI       CoreDNS     kube-proxy
       │
       ├── AWS Load Balancer Controller
       ├── EBS CSI Driver
       ├── Metrics Server
       ├── ArgoCD
       ├── Kyverno
       └── Argo Rollouts
```

A version upgrade can expose:

* deprecated Kubernetes APIs
* incompatible CRDs
* incompatible admission webhooks
* incompatible controllers
* incompatible CSI drivers
* node AMI problems
* CNI issues
* DNS problems
* workload scheduling problems
* PodDisruptionBudget conflicts

Therefore:

> **Treat an EKS upgrade as a production change, not a maintenance command.**

---

# 2. EKS Upgrade Layers

There are several independent components.

```text
Layer 1
EKS Control Plane
        │
        ▼
Layer 2
EKS Add-ons
        │
        ▼
Layer 3
Worker Nodes
        │
        ├── Managed Node Groups
        │
        └── Karpenter
        │
        ▼
Layer 4
Applications
        │
        ▼
Layer 5
Observability / Security / GitOps
```

You must verify compatibility across all of them.

---

# 3. Control Plane vs Data Plane

## Control Plane

AWS manages the EKS control plane.

It includes components such as:

```text
Kubernetes API Server
Controller Manager
Scheduler
etcd
```

You do not SSH into the EKS control plane.

---

## Data Plane

Your worker capacity runs the workloads.

For our architecture:

```text
EKS
│
├── System Managed Node Group
│
├── Application Managed Node Group
│
└── Karpenter Provisioned Nodes
```

Your workloads run on the data plane.

---

# 4. What Actually Changes During an Upgrade?

Suppose:

```text
Current:
Kubernetes 1.X

Target:
Kubernetes 1.Y
```

The upgrade normally involves:

```text
1. Validate compatibility
        ↓
2. Upgrade control plane
        ↓
3. Upgrade EKS add-ons
        ↓
4. Upgrade/replace nodes
        ↓
5. Validate workloads
        ↓
6. Validate observability
        ↓
7. Validate networking/storage/security
```

Do **not** assume upgrading the control plane automatically upgrades everything else.

---

# 5. Production Upgrade Principle

Our environment structure is:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
PRODUCTION
```

We never discover upgrade problems for the first time in production.

Example:

```text
DEV
EKS 1.X
 ↓
Upgrade
 ↓
Test
 ↓
QA
 ↓
Upgrade
 ↓
Test
 ↓
STAGING
 ↓
Upgrade
 ↓
Production-like validation
 ↓
PROD
```

---

# 6. Check Current Cluster Version

Start with:

```bash
aws eks describe-cluster \
  --name ecommerce-prod \
  --region ap-south-1 \
  --query 'cluster.version' \
  --output text
```

Example:

```text
1.xx
```

Check Kubernetes:

```bash
kubectl version --short
```

Depending on your installed `kubectl` version, use:

```bash
kubectl version
```

---

# 7. Check EKS Upgrade Availability

```bash
aws eks describe-cluster \
  --name ecommerce-prod \
  --region ap-south-1 \
  --query 'cluster.version'
```

Then verify the **current AWS EKS Kubernetes version support matrix** before selecting a target version.

Do not hard-code a version in this documentation because EKS support changes over time.

---

# 8. Pre-Upgrade Inventory

Before changing anything, capture the current state.

## Cluster

```bash
aws eks describe-cluster \
  --name ecommerce-prod \
  --region ap-south-1
```

---

## Nodes

```bash
kubectl get nodes -o wide
```

---

## Node groups

```bash
aws eks list-nodegroups \
  --cluster-name ecommerce-prod \
  --region ap-south-1
```

Then:

```bash
aws eks describe-nodegroup \
  --cluster-name ecommerce-prod \
  --nodegroup-name <NODEGROUP_NAME> \
  --region ap-south-1
```

---

# 9. Check EKS Add-ons

```bash
aws eks list-addons \
  --cluster-name ecommerce-prod \
  --region ap-south-1
```

Example:

```text
vpc-cni
coredns
kube-proxy
aws-ebs-csi-driver
eks-pod-identity-agent
```

Get detailed information:

```bash
aws eks describe-addon \
  --cluster-name ecommerce-prod \
  --addon-name vpc-cni \
  --region ap-south-1
```

---

# 10. Check Platform Components

Our cluster also contains:

```text
ArgoCD
Kyverno
Argo Rollouts
AWS Load Balancer Controller
Metrics Server
Prometheus
Grafana
Fluent Bit
Logstash
ECK
OpenTelemetry Collector
Jaeger
```

Check:

```bash
kubectl get pods -A
```

Then specifically:

```bash
kubectl get deployments -A
```

---

# 11. Why Add-ons Are Critical

Imagine:

```text
EKS control plane
       │
       ▼
New Kubernetes version
       │
       ▼
Old incompatible controller
       │
       ▼
Failure
```

For example:

```text
EKS upgraded
     ↓
AWS Load Balancer Controller incompatible
     ↓
Ingress reconciliation fails
     ↓
ALB configuration stops updating
```

Or:

```text
EKS upgraded
     ↓
EBS CSI incompatibility
     ↓
PVC attachment problems
     ↓
Pods remain Pending
```

Therefore add-on compatibility must be part of the upgrade plan.

---

# 12. Kubernetes API Deprecations

One of the most important upgrade checks is:

> Are our manifests using APIs that are removed in the target Kubernetes version?

For example, an old manifest may contain:

```yaml
apiVersion: old.api/v1
```

while the newer Kubernetes version expects:

```yaml
apiVersion: new.api/v1
```

The exact deprecated APIs depend on the Kubernetes version.

---

# 13. Search Your Git Repositories

Search application and platform repositories.

For example:

```bash
grep -R "apiVersion:" .
```

For more targeted checks:

```bash
grep -R "extensions/v1beta1" .
grep -R "apps/v1beta1" .
grep -R "networking.k8s.io/v1beta1" .
```

But don't rely only on manually searching known old API strings.

Use a Kubernetes API deprecation scanner appropriate for your target version.

---

# 14. Why GitOps Helps

Our desired state exists in Git.

```text
GitHub
   │
   ├── Application Helm charts
   │
   ├── GitOps values
   │
   ├── Platform manifests
   │
   └── Kyverno policies
```

Therefore we can audit what Kubernetes resources we intend to deploy before upgrading.

Example:

```bash
helm template ./helm/order-service \
  -f values-prod.yaml
```

Then inspect the generated manifests.

---

# 15. CRDs Are Especially Important

Our platform contains custom resources.

Examples:

```text
Application
ApplicationSet
Rollout
AnalysisTemplate
ExternalSecret
Policy
```

These are backed by CRDs.

Check:

```bash
kubectl get crd
```

Examples:

```text
applications.argoproj.io
applicationsets.argoproj.io
rollouts.argoproj.io
externalsecrets.external-secrets.io
policies.kyverno.io
```

Before an upgrade:

```text
Kubernetes version
        +
CRD version
        +
Controller version
```

must be compatible.

---

# 16. Admission Webhooks

Our cluster has security controls such as Kyverno.

That means:

```text
kubectl apply
      │
      ▼
API Server
      │
      ▼
Admission Webhook
      │
      ▼
Kyverno
      │
      ▼
Allow / Reject
```

During an upgrade, an incompatible admission webhook can cause deployment failures.

Check:

```bash
kubectl get validatingwebhookconfigurations
```

and:

```bash
kubectl get mutatingwebhookconfigurations
```

---

# 17. Backup Before Upgrade

Before production:

```text
Git
+
Terraform state
+
RDS backups
+
Secrets
+
EKS configuration
+
GitOps
```

must already be recoverable.

Check RDS:

```bash
aws rds describe-db-instances
```

Check Terraform state backup/versioning.

Check GitOps:

```bash
git log --oneline
```

The cluster upgrade should never be the first time you discover that your recovery process doesn't work.

---

# 18. Terraform and EKS Upgrades

Our EKS infrastructure is managed through Terraform.

Example:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   └── eks/
    └── environments/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

The version should be controlled by infrastructure code.

Conceptually:

```hcl
module "eks" {
  source = "../../modules/eks"

  cluster_name    = "ecommerce-prod"
  cluster_version = var.cluster_version
}
```

Variable:

```hcl
variable "cluster_version" {
  type = string
}
```

Production:

```hcl
cluster_version = "TARGET_VERSION"
```

Use the currently supported target version after checking AWS's EKS support documentation.

---

# 19. Terraform Upgrade Workflow

Never immediately run:

```bash
terraform apply
```

Production workflow:

```text
Change Terraform
      ↓
terraform fmt
      ↓
terraform validate
      ↓
terraform plan
      ↓
Review plan
      ↓
Jira Change
      ↓
Approval
      ↓
terraform apply
      ↓
Validate EKS
```

---

# 20. Important Terraform Principle

Terraform manages infrastructure.

ArgoCD manages Kubernetes application desired state.

Therefore:

```text
Terraform
   │
   ├── EKS
   ├── VPC
   ├── IAM
   ├── Node Groups
   └── AWS infrastructure
```

while:

```text
ArgoCD
   │
   ├── Deployments
   ├── Services
   ├── Ingress
   ├── Rollouts
   ├── Policies
   └── Applications
```

Do not mix these responsibilities unnecessarily.

---

# 21. Upgrade EKS Control Plane

Once DEV has passed all prechecks, the control plane can be upgraded.

Using AWS CLI:

```bash
aws eks update-cluster-version \
  --name ecommerce-dev \
  --kubernetes-version <TARGET_VERSION> \
  --region ap-south-1
```

Monitor:

```bash
aws eks describe-update \
  --name ecommerce-dev \
  --cluster-name ecommerce-dev \
  --region ap-south-1
```

List updates:

```bash
aws eks list-updates \
  --name ecommerce-dev \
  --region ap-south-1
```

---

# 22. Production Recommendation

In production, Terraform should normally be the controlled mechanism for infrastructure changes if the cluster is Terraform-managed.

Therefore:

```text
Terraform code
      ↓
PR
      ↓
terraform plan
      ↓
Review
      ↓
Jira Change
      ↓
Approval
      ↓
terraform apply
```

The AWS CLI commands are useful for:

* investigation
* validation
* emergency operations
* understanding what Terraform ultimately changes

---

# 23. Control Plane Upgrade Does Not Mean Nodes Are Finished

After the control plane upgrade:

```text
Control plane
      │
      │ new version
      ▼
Worker nodes
      │
      │ old version
      ▼
Temporary version skew
```

This is why the node upgrade must follow the supported EKS/Kubernetes version-skew rules.

Do not leave old nodes indefinitely.

---

# 24. Managed Node Group Upgrade

Check:

```bash
aws eks describe-nodegroup \
  --cluster-name ecommerce-prod \
  --nodegroup-name ecommerce-app \
  --region ap-south-1
```

A managed node group upgrade replaces nodes.

Conceptually:

```text
OLD NODE
   │
   ├── cordon
   ├── drain
   └── terminate
          │
          ▼
NEW NODE
   │
   └── workload scheduled
```

---

# 25. Why PDB Matters During Node Upgrade

Suppose:

```text
order-service
replicas = 3
```

and:

```yaml
minAvailable: 2
```

Kubernetes should maintain the availability requirement while nodes are drained.

Check:

```bash
kubectl get pdb -A
```

For example:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: order-service
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: order-service
```

---

# 26. PDB Can Also Block an Upgrade

This is important.

Suppose:

```text
3 replicas
PDB minAvailable = 3
```

Then Kubernetes cannot voluntarily remove even one pod.

Node drain may become blocked.

Therefore:

```text
PDB
+
replica count
+
node capacity
```

must be designed together.

---

# 27. Topology Spread

Our production workloads should be distributed across AZs.

Example:

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: order-service
```

This helps avoid:

```text
AZ-A
order-1
order-2
order-3

AZ-B
nothing

AZ-C
nothing
```

Instead:

```text
AZ-A → order-1
AZ-B → order-2
AZ-C → order-3
```

This becomes particularly important during node replacement.

---

# 28. Karpenter Node Rotation

Karpenter works differently from managed node groups.

```text
Pending Pod
    ↓
Scheduler
    ↓
Karpenter
    ↓
Provision Node
    ↓
Pod scheduled
```

During an upgrade:

```text
Old Node
    ↓
Disruption / replacement
    ↓
New Node
    ↓
Pod migration
```

The exact Karpenter configuration and API version must match the Karpenter version deployed in the cluster.

---

# 29. Karpenter Pre-Upgrade Checks

Check:

```bash
kubectl get nodepool
```

and:

```bash
kubectl get nodeclass
```

Depending on the Karpenter version, the exact resource/API names can differ.

Check:

```bash
kubectl get pods -n kube-system | grep -i karpenter
```

Then:

```bash
kubectl logs -n kube-system deployment/karpenter
```

if the controller is deployed there.

---

# 30. Check Node Architecture

If the cluster uses both:

```text
AMD64
ARM64
```

verify application images support the architecture.

For example:

```bash
docker buildx imagetools inspect <IMAGE>
```

You might discover:

```text
linux/amd64
```

but not:

```text
linux/arm64
```

A scheduler could place the workload onto an ARM node and the container would fail.

---

# 31. Multi-Architecture Production

If supporting both architectures:

```text
Source
  ↓
CI
  ↓
Build AMD64
  +
Build ARM64
  ↓
Multi-platform image
  ↓
ECR
```

Example:

```bash
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  --push \
  -t <ECR_REPOSITORY>:<TAG> .
```

Only use this if your application dependencies and base images genuinely support both architectures.

---

# 32. EKS Add-on Upgrade

After the control plane:

```text
EKS version
      ↓
VPC CNI
CoreDNS
kube-proxy
EBS CSI
Pod Identity Agent
```

Check available versions:

```bash
aws eks describe-addon-versions \
  --kubernetes-version <TARGET_VERSION> \
  --region ap-south-1
```

Then identify a compatible version for each add-on.

---

# 33. VPC CNI

VPC CNI is particularly important because our nodes use AWS VPC networking.

Check:

```bash
kubectl get daemonset -n kube-system aws-node
```

Pods:

```bash
kubectl get pods -n kube-system -l k8s-app=aws-node
```

Logs:

```bash
kubectl logs -n kube-system \
  -l k8s-app=aws-node \
  --tail=100
```

Problems here can cause:

```text
Pod IP allocation failure
Pod Pending
Network connectivity failure
```

---

# 34. CoreDNS

Check:

```bash
kubectl get deployment coredns -n kube-system
```

Test DNS:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --rm -it \
  --restart=Never \
  -- nslookup kubernetes.default
```

If DNS fails:

```text
Application
    ↓
Service DNS
    ↓
CoreDNS
    ↓
Failure
```

Applications may appear healthy at the container level but fail to communicate.

---

# 35. kube-proxy

Check:

```bash
kubectl get daemonset kube-proxy -n kube-system
```

Its role is related to Kubernetes Service networking.

A kube-proxy issue can affect:

```text
Service
 ↓
Pod
```

connectivity.

---

# 36. EBS CSI Driver

For stateful workloads:

```bash
kubectl get pods -n kube-system | grep ebs
```

Check:

```bash
kubectl get csidrivers
```

PVC status:

```bash
kubectl get pvc -A
```

PV:

```bash
kubectl get pv
```

A driver issue can produce:

```text
Pod Pending
   ↓
PVC
   ↓
Volume attachment failure
```

---

# 37. AWS Load Balancer Controller

This is critical because our external traffic uses:

```text
Internet
 ↓
CloudFront/WAF
 ↓
ALB
 ↓
AWS Load Balancer Controller
 ↓
Kubernetes Service
 ↓
Pod
```

Check:

```bash
kubectl get deployment \
  -n kube-system \
  aws-load-balancer-controller
```

Check logs:

```bash
kubectl logs \
  -n kube-system \
  deployment/aws-load-balancer-controller \
  --tail=100
```

After an upgrade, validate:

```bash
kubectl get ingress -A
```

and:

```bash
kubectl describe ingress <INGRESS_NAME> \
  -n <NAMESPACE>
```

---

# 38. Metrics Server

Check:

```bash
kubectl get deployment metrics-server \
  -n kube-system
```

Then:

```bash
kubectl top nodes
```

and:

```bash
kubectl top pods -A
```

If `kubectl top` fails after an upgrade, investigate Metrics Server before assuming the applications themselves are broken.

---

# 39. ArgoCD Compatibility

Our GitOps control plane is:

```text
GitHub
   ↓
ArgoCD
   ↓
Kubernetes API
```

Check:

```bash
kubectl get pods -n argocd
```

Then:

```bash
argocd app list
```

Every production application should remain:

```text
Synced
Healthy
```

after the upgrade.

---

# 40. Kyverno Compatibility

Check:

```bash
kubectl get pods -n kyverno
```

Check policies:

```bash
kubectl get clusterpolicy
```

and:

```bash
kubectl get policy -A
```

Test deployment admission using a non-production namespace.

For example, intentionally submit a test workload violating:

```text
runAsNonRoot
```

and verify the expected policy behavior.

---

# 41. Argo Rollouts Compatibility

Check:

```bash
kubectl get pods -n argo-rollouts
```

Then:

```bash
kubectl get rollouts -A
```

Do not perform a cluster upgrade while an important production canary is paused unless the change plan explicitly accounts for it.

---

# 42. Prometheus Validation

Check:

```bash
kubectl get pods -n monitoring
```

Then verify targets.

Depending on the Prometheus deployment:

```bash
kubectl port-forward \
  svc/prometheus-operated \
  -n monitoring \
  9090:9090
```

Validate:

```text
Targets
Rules
Alerts
Scrapes
```

---

# 43. Logging Validation

Our logging architecture is:

```text
Pod
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

After upgrade:

```bash
kubectl get pods -A | grep -E 'fluent|logstash|elastic|kibana'
```

Then verify new application logs are arriving.

---

# 44. Upgrade Test: Order Service

After the cluster upgrade, start with our reference application.

Check:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app=order-service
```

Then:

```bash
kubectl get deployment order-service \
  -n ecommerce-prod
```

Service:

```bash
kubectl get svc order-service \
  -n ecommerce-prod
```

---

# 45. Verify Image

```bash
kubectl get deployment order-service \
  -n ecommerce-prod \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Then:

```bash
kubectl get pods \
  -n ecommerce-prod \
  -l app=order-service \
  -o jsonpath='{range .items[*]}{.metadata.name}{" -> "}{.status.containerStatuses[0].imageID}{"\n"}{end}'
```

We want the expected immutable digest.

---

# 46. Application Health

Check:

```bash
kubectl port-forward \
  svc/order-service \
  -n ecommerce-prod \
  8080:8080
```

Then:

```bash
curl http://localhost:8080/actuator/health
```

Expected:

```json
{
  "status": "UP"
}
```

---

# 47. Database Validation

Check application connectivity to PostgreSQL.

Verify:

```text
Pod
 ↓
Service/network
 ↓
RDS
 ↓
PostgreSQL
```

Application logs:

```bash
kubectl logs \
  -n ecommerce-prod \
  deployment/order-service \
  --tail=100
```

Look for:

```text
connection refused
authentication failed
timeout
DNS failure
connection pool exhaustion
```

---

# 48. Redis Validation

If Order Service uses Redis:

```text
Pod
 ↓
Redis endpoint
 ↓
ElastiCache
```

Validate:

* DNS
* security groups
* authentication
* TLS
* application connectivity
* cache hit/miss metrics

---

# 49. SQS Validation

If Order Service publishes events:

```text
Order Service
     ↓
SQS
     ↓
Consumer
```

Verify:

```text
SendMessage
Queue depth
Consumer processing
DLQ
```

Check AWS metrics for abnormal backlog.

---

# 50. Upgrade Validation Matrix

After upgrade:

| Area           | Check              |
| -------------- | ------------------ |
| Control plane  | Kubernetes API     |
| Nodes          | Ready              |
| CNI            | Pod networking     |
| CoreDNS        | DNS                |
| kube-proxy     | Service networking |
| EBS CSI        | PVC/volume         |
| LB Controller  | ALB/Ingress        |
| Metrics Server | `kubectl top`      |
| ArgoCD         | Synced/Healthy     |
| Kyverno        | Admission          |
| Prometheus     | Targets            |
| Grafana        | Dashboards         |
| Fluent Bit     | Logs               |
| Elasticsearch  | Cluster health     |
| Application    | Health endpoint    |
| RDS            | DB connectivity    |
| Redis          | Cache connectivity |
| SQS            | Event processing   |
| HPA            | Scaling            |
| PDB            | Availability       |
| Karpenter      | Node provisioning  |

---

# 51. Application-Level Smoke Test

Our production smoke test should include:

```text
GET /health
       ↓
Authentication
       ↓
Create order
       ↓
Read order
       ↓
Update order
       ↓
Order event
       ↓
Database
       ↓
Messaging
       ↓
Logging
       ↓
Metrics
```

For example:

```bash
curl -X POST \
  https://api.example.com/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 1001,
    "productId": 5001,
    "quantity": 2
  }'
```

Then:

```bash
curl \
  https://api.example.com/api/v1/orders/<ORDER_ID>
```

---

# 52. Check HPA

```bash
kubectl get hpa -A
```

Detailed:

```bash
kubectl describe hpa order-service \
  -n ecommerce-prod
```

Verify:

```text
Current replicas
Desired replicas
CPU
Memory
Custom metrics
```

---

# 53. Check Karpenter

After upgrade:

```bash
kubectl get nodes
```

Then inspect:

```bash
kubectl get nodepool
```

If a workload is Pending:

```bash
kubectl get pods -A \
  --field-selector=status.phase=Pending
```

Then:

```bash
kubectl describe pod <POD_NAME> \
  -n <NAMESPACE>
```

Look for:

```text
FailedScheduling
```

---

# 54. Common Failure: Node Does Not Join

Symptoms:

```text
NodeGroup update
       ↓
New node launched
       ↓
Node never becomes Ready
```

Check:

```bash
kubectl get nodes
```

Then:

```bash
aws ec2 describe-instances ...
```

Potential causes:

```text
IAM
Networking
Security Groups
Bootstrap
AMI
EKS version mismatch
VPC CNI
Subnet IP capacity
```

---

# 55. Common Failure: Pods Cannot Get IP

Symptoms:

```text
FailedCreatePodSandBox
```

or:

```text
failed to assign an IP address
```

Investigate:

```bash
kubectl get pods -A
```

Then:

```bash
kubectl logs \
  -n kube-system \
  -l k8s-app=aws-node \
  --tail=200
```

Also inspect subnet IP availability.

This is especially important in EKS because VPC CNI allocates VPC IP addresses to Pods.

---

# 56. Common Failure: DNS

Symptoms:

```text
Temporary failure in name resolution
```

Test:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --rm -it \
  --restart=Never \
  -- nslookup kubernetes.default
```

Then:

```bash
kubectl get pods -n kube-system \
  -l k8s-app=kube-dns
```

Check CoreDNS logs:

```bash
kubectl logs \
  -n kube-system \
  -l k8s-app=kube-dns \
  --tail=100
```

---

# 57. Common Failure: Admission Webhook

Symptoms:

```text
Internal error occurred
```

or:

```text
failed calling webhook
```

Check:

```bash
kubectl get validatingwebhookconfigurations
```

Then identify the controller behind the webhook.

Check its Pods:

```bash
kubectl get pods -A
```

This is why admission controllers must be included in upgrade testing.

---

# 58. Common Failure: ALB Stops Updating

Symptoms:

```text
Ingress exists
ALB exists
but configuration is stale
```

Check:

```bash
kubectl get ingress -A
```

Then:

```bash
kubectl describe ingress <NAME> \
  -n <NAMESPACE>
```

Controller:

```bash
kubectl logs \
  -n kube-system \
  deployment/aws-load-balancer-controller \
  --tail=200
```

---

# 59. Common Failure: PVC Stuck

Symptoms:

```text
Pod Pending
```

Check:

```bash
kubectl get pvc -A
```

Then:

```bash
kubectl describe pvc <PVC_NAME> \
  -n <NAMESPACE>
```

Check:

```bash
kubectl get pods -n kube-system | grep ebs
```

Potential causes:

```text
EBS CSI version
IAM
AZ constraints
Volume attachment
StorageClass
Node capacity
```

---

# 60. Common Failure: Karpenter Cannot Provision

Symptoms:

```text
Pod Pending
```

Events:

```bash
kubectl describe pod <POD_NAME> \
  -n <NAMESPACE>
```

Check:

```bash
kubectl get nodepool
kubectl get nodeclass
```

Potential causes:

```text
No matching instance type
No matching AZ
IAM problem
Subnet discovery problem
Security group discovery problem
Architecture mismatch
Capacity unavailable
PVC topology constraints
Taints/tolerations
```

---

# 61. Rollback Reality

This is extremely important.

Application rollback:

```text
Git revert
 ↓
ArgoCD
 ↓
Previous image
```

Node rollback:

```text
Replace nodes
 ↓
Previous compatible node image/version
```

But:

> **A Kubernetes control-plane upgrade should not be treated as a normal application rollback.**

Do not assume:

```bash
upgrade
↓
something failed
↓
downgrade control plane
```

is a simple or supported rollback procedure.

This is why:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

and pre-upgrade compatibility testing are critical.

---

# 62. Production Change Management

Our existing Jira process applies.

Example:

```text
CHG-10501
```

Change:

```text
Upgrade EKS
1.xx → target supported version
```

The Jira change should contain:

### Implementation

```text
1. Validate DEV
2. Validate QA
3. Validate STAGING
4. Confirm backups
5. Upgrade EKS control plane
6. Upgrade add-ons
7. Upgrade nodes
8. Validate applications
9. Validate observability
10. Monitor
```

---

# 63. Rollback / Recovery Plan in Jira

Document:

```text
Control plane rollback:
Not treated as normal rollback.

Recovery:
- restore workloads
- restore compatible node capacity
- correct incompatible add-ons
- revert application/platform changes
- use tested recovery procedures
```

For major upgrades, define the recovery architecture **before** executing the production change.

---

# 64. Production Upgrade Window

Example:

```text
22:00
Change begins

22:10
Prechecks complete

22:15
Control plane upgrade

22:30
Add-ons

22:45
Node rotation

23:15
Application validation

23:30
Smoke tests

23:45
Monitoring

00:00
Change completed
```

The exact duration depends on the cluster and upgrade process; do not promise a fixed duration in the runbook.

---

# 65. Monitoring During Upgrade

Watch:

```bash
kubectl get nodes -w
```

Pods:

```bash
kubectl get pods -A -w
```

Deployments:

```bash
kubectl get deployments -A
```

Events:

```bash
kubectl get events -A \
  --sort-by=.lastTimestamp
```

Metrics:

```text
CPU
Memory
Pod restarts
Error rate
Latency
Request rate
Node availability
ALB 4xx/5xx
RDS connections
Redis health
SQS backlog
```

---

# 66. What We Watch in Grafana

Create an:

```text
EKS Upgrade Dashboard
```

Panels:

### Cluster

```text
Ready nodes
NotReady nodes
Pending pods
```

### Workloads

```text
Pod restarts
OOMKilled
CrashLoopBackOff
Deployment availability
```

### Application

```text
Request rate
Error rate
Latency
```

### Infrastructure

```text
CPU
Memory
Disk
Network
```

### AWS

```text
ALB 5xx
RDS connections
Redis metrics
SQS backlog
```

---

# 67. Upgrade Sequence for Our Platform

The complete sequence becomes:

```text
                PRECHECK
                   │
                   ▼
          Deprecated API Audit
                   │
                   ▼
          Backup / Recovery Check
                   │
                   ▼
            DEV UPGRADE
                   │
                   ▼
            Add-ons + Nodes
                   │
                   ▼
          Application Validation
                   │
                   ▼
              QA UPGRADE
                   │
                   ▼
            STAGING UPGRADE
                   │
                   ▼
           Production Approval
                   │
                   ▼
             Jira Change
                   │
                   ▼
             PROD CONTROL
              PLANE UPGRADE
                   │
                   ▼
               ADD-ONS
                   │
                   ▼
             NODE ROTATION
                   │
                   ▼
          APPLICATION VALIDATION
                   │
                   ▼
            SMOKE TESTS
                   │
                   ▼
             MONITORING
                   │
                   ▼
            CHANGE CLOSURE
```

---

# 68. EKS Upgrade Checklist

## Pre-Upgrade

```text
[ ] Target version supported by AWS
[ ] DEV tested
[ ] QA tested
[ ] STAGING tested
[ ] Deprecated APIs checked
[ ] CRDs checked
[ ] Admission webhooks checked
[ ] EKS add-ons checked
[ ] AWS Load Balancer Controller checked
[ ] Karpenter checked
[ ] ArgoCD checked
[ ] Kyverno checked
[ ] Argo Rollouts checked
[ ] Prometheus checked
[ ] Logging checked
[ ] RDS backup verified
[ ] Recovery plan verified
[ ] Jira change approved
```

---

## During Upgrade

```text
[ ] Control plane
[ ] EKS add-ons
[ ] Managed node groups
[ ] Karpenter nodes
[ ] CoreDNS
[ ] VPC CNI
[ ] kube-proxy
[ ] EBS CSI
[ ] Pod Identity
[ ] Load Balancer Controller
[ ] Application workloads
```

---

## Post-Upgrade

```text
[ ] All nodes Ready
[ ] No unexpected Pending Pods
[ ] No CrashLoopBackOff
[ ] No OOMKilled
[ ] DNS works
[ ] Services work
[ ] ALB works
[ ] RDS works
[ ] Redis works
[ ] SQS works
[ ] HPA works
[ ] Karpenter works
[ ] ArgoCD healthy
[ ] Kyverno healthy
[ ] Prometheus healthy
[ ] Grafana healthy
[ ] Logs flowing
[ ] Traces flowing
[ ] Smoke tests passed
[ ] Jira change closed
```

---

# 69. Production EKS Lifecycle

EKS management isn't only upgrading Kubernetes.

The lifecycle is:

```text
         PLAN
           │
           ▼
      PROVISION
           │
           ▼
       CONFIGURE
           │
           ▼
        OPERATE
           │
           ▼
        MONITOR
           │
           ▼
        HARDEN
           │
           ▼
        UPGRADE
           │
           ▼
        VALIDATE
           │
           ▼
        OPTIMIZE
           │
           └──────────────┐
                          ▼
                       UPGRADE
```

---

# 70. Our Production Ownership Model

| Area                | Primary Owner              |
| ------------------- | -------------------------- |
| EKS infrastructure  | DevOps / Platform          |
| Terraform           | DevOps                     |
| Node groups         | Platform                   |
| Karpenter           | Platform                   |
| EKS add-ons         | Platform                   |
| Kubernetes security | Security / Platform        |
| Application         | Backend teams              |
| GitOps              | Platform                   |
| ArgoCD              | Platform                   |
| Monitoring          | SRE                        |
| Logging             | SRE / Platform             |
| Database            | DBA / Platform             |
| Change management   | Engineering / Change Owner |

---

# 71. Final Architecture

After completing this chapter, our EKS lifecycle looks like:

```text
                         AWS
                          │
                 ┌────────┴────────┐
                 │                 │
              Terraform          IAM
                 │                 │
                 ▼                 │
          ┌──────────────┐         │
          │ EKS Cluster  │◄────────┘
          └──────┬───────┘
                 │
       ┌─────────┼─────────┐
       │         │         │
       ▼         ▼         ▼
 Managed      Karpenter   Add-ons
 Nodes         Nodes
       │         │         │
       └─────────┼─────────┘
                 ▼
          Kubernetes Pods
                 │
       ┌─────────┼──────────┐
       ▼         ▼          ▼
     Apps      ArgoCD     Kyverno
       │         │          │
       ▼         ▼          ▼
    RDS/Redis  GitOps    Security
       │
       ▼
 Observability
 Prometheus/Grafana
 ELK/OTel/Jaeger
```

The key production principle is:

> **Upgrade the cluster as a controlled lifecycle process: test compatibility first, upgrade control plane/add-ons/nodes in a supported sequence, validate every platform dependency, and use Jira + Terraform + GitOps for traceability and control.**

---

## Next Chapter

### `97-EKS-Backup-and-Recovery-Architecture.md`

We will go deeper into **what happens if the EKS cluster itself is badly damaged or lost**, including:

* EKS cluster recreation from Terraform
* what Terraform can and cannot restore
* GitOps-based workload recovery
* RDS recovery
* EBS/PVC recovery
* Secrets Manager recovery
* ECR recovery
* S3 recovery
* SQS/SNS recreation
* Route 53 / CloudFront / WAF recovery
* cross-region DR
* Velero and when it is useful
* rebuilding an entire production EKS environment
* **“EKS cluster completely deleted at 2 AM — step-by-step recovery”** scenario
* RTO/RPO validation and DR drills
