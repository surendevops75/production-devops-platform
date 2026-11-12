# 61-EKS-Addons.md

# EKS Add-ons — Production Platform Components

This chapter installs the core components required before we deploy the Order Service.

Our EKS cluster currently exists:

```text
AWS VPC
   |
   v
EKS Cluster
   |
   +--> Control Plane
   |
   +--> Managed Node Group
```

Now we turn the basic cluster into a usable Kubernetes platform:

```text
EKS
 |
 +--> VPC CNI
 +--> CoreDNS
 +--> kube-proxy
 +--> EBS CSI Driver
 +--> EKS Pod Identity
 +--> AWS Load Balancer Controller
 +--> Metrics Server
```

Later we will add:

```text
ArgoCD
External Secrets Operator
Prometheus
Grafana
Fluent Bit
Logstash
Elasticsearch
Kibana
OpenTelemetry
```

---

# 1. Why Add-ons Matter

A newly created EKS cluster is not the complete application platform.

For example, our application eventually needs:

```text
Order Service
     |
     +--> Database
     +--> Secrets
     +--> Load Balancer
     +--> Storage
     +--> AWS APIs
     +--> Metrics
```

Those capabilities depend on platform components.

Therefore:

```text
EKS Cluster
      ↓
EKS Add-ons
      ↓
Platform Services
      ↓
Applications
```

---

# 2. Production Architecture

Our target architecture:

```text
                         AWS
                          |
                   +------+------+
                   |             |
                  VPC           IAM
                   |             |
                   v             |
                  EKS <----------+
                   |
       +-----------+-----------+
       |           |           |
       v           v           v
     CNI        CoreDNS    kube-proxy
       |
       +-------------------+
       |                   |
       v                   v
   EBS CSI          Load Balancer
       |             Controller
       |                   |
       v                   v
     EBS                  ALB
       |
       v
     PVC
```

---

# 3. Add-on Categories

We should distinguish between **AWS-managed EKS add-ons** and **additional platform controllers**.

### Core EKS add-ons

```text
VPC CNI
CoreDNS
kube-proxy
EBS CSI Driver
```

### AWS/Kubernetes controllers

```text
EKS Pod Identity
AWS Load Balancer Controller
```

### Metrics

```text
Metrics Server
```

### Later platform services

```text
ArgoCD
External Secrets Operator
Prometheus
Grafana
Fluent Bit
```

This distinction matters because installation and lifecycle management differ.

---

# 4. Terraform Directory

We will extend:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   ├── vpc/
    │   ├── iam/
    │   ├── ecr/
    │   ├── eks/
    │   └── eks-addons/
    │
    └── environments/
        ├── dev/
        ├── qa/
        └── prod/
```

Create:

```bash
mkdir -p terraform/modules/eks-addons
cd terraform/modules/eks-addons
```

---

# 5. EKS Managed Add-ons

For supported AWS-managed components, prefer:

```text
Terraform
    |
    v
aws_eks_addon
```

instead of manually applying arbitrary YAML.

Benefits:

* version management
* AWS integration
* lifecycle management
* Terraform visibility
* upgrade control

---

# 6. VPC CNI

The VPC CNI provides networking for EKS pods.

Architecture:

```text
Pod
 |
 v
VPC CNI
 |
 v
AWS VPC
 |
 v
ENI / IP
```

This is one of the most important EKS components.

---

# 7. VPC CNI Add-on

Terraform:

```hcl
resource "aws_eks_addon" "vpc_cni" {
  cluster_name = var.cluster_name

  addon_name = "vpc-cni"

  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  tags = var.tags
}
```

The exact supported arguments can vary with the AWS provider version, so use the schema associated with the provider version pinned by this project.

---

# 8. VPC CNI IAM Permissions

A modern security architecture avoids unnecessarily putting all CNI permissions on the EC2 node role.

Conceptually:

```text
aws-node
   |
   v
EKS Pod Identity
   |
   v
CNI IAM Role
   |
   v
AWS EC2 networking APIs
```

This follows least privilege better than giving every workload or every node excessive AWS permissions.

The exact recommended AWS identity mechanism should be verified against the EKS version and current AWS guidance when implementing.

---

# 9. Why VPC CNI Is Important

Suppose:

```text
Node
10.0.11.100
```

and pods consume VPC addresses:

```text
Pod A -> 10.0.11.x
Pod B -> 10.0.11.x
Pod C -> 10.0.11.x
```

Eventually:

```text
Subnet
   |
   +--> IP addresses
   |
   +--> ENIs
   |
   +--> Pod capacity
```

becomes a capacity constraint.

Therefore, EKS scaling requires monitoring:

```text
CPU
Memory
Pod count
IP availability
ENI capacity
```

---

# 10. CoreDNS

Kubernetes service discovery depends heavily on DNS.

Example:

```text
order-service
     |
     v
http://payment-service
```

Kubernetes resolves:

```text
payment-service
```

through CoreDNS.

Architecture:

```text
Application Pod
      |
      v
Kubernetes DNS
      |
      v
CoreDNS
      |
      v
Service
```

---

# 11. CoreDNS Add-on

Terraform:

```hcl
resource "aws_eks_addon" "coredns" {
  cluster_name = var.cluster_name

  addon_name = "coredns"

  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  tags = var.tags
}
```

---

# 12. Why CoreDNS Matters to Microservices

Our 21 microservices communicate using Kubernetes Services.

Example:

```text
order-service
     |
     | HTTP
     v
inventory-service
```

The application uses:

```text
http://inventory-service:8080
```

instead of:

```text
http://10.0.11.123:8080
```

Because pod IPs can change.

Service discovery provides stable naming.

---

# 13. CoreDNS Failure

If CoreDNS is unhealthy:

```text
order-service
     |
     v
DNS lookup
     |
     X
CoreDNS unavailable
```

Symptoms can look like:

```text
Connection failure
Unknown host
Service unavailable
Timeouts
```

This is why DNS health is a platform-level concern.

---

# 14. kube-proxy

`kube-proxy` participates in implementing Kubernetes Service networking on nodes.

Conceptually:

```text
Pod
 |
 v
Service
 |
 v
kube-proxy/networking
 |
 v
Backend Pod
```

It is normally deployed as a DaemonSet.

---

# 15. kube-proxy Add-on

Terraform:

```hcl
resource "aws_eks_addon" "kube_proxy" {
  cluster_name = var.cluster_name

  addon_name = "kube-proxy"

  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  tags = var.tags
}
```

---

# 16. Version Compatibility

Do not choose arbitrary versions for:

```text
vpc-cni
coredns
kube-proxy
```

The versions should be compatible with:

```text
EKS Kubernetes version
AWS-supported add-on versions
```

A practical workflow is:

```text
Check EKS Kubernetes version
        ↓
Check supported add-on versions
        ↓
Select versions
        ↓
Terraform
        ↓
Apply
        ↓
Validate
```

---

# 17. EBS CSI Driver

Our applications may eventually need persistent storage.

Architecture:

```text
Pod
 |
 v
PVC
 |
 v
StorageClass
 |
 v
EBS CSI Driver
 |
 v
Amazon EBS
```

For example:

```text
PostgreSQL StatefulSet
        |
        v
PVC
        |
        v
gp3 EBS volume
```

However, remember our production database strategy:

```text
Production PostgreSQL
        |
        v
Amazon RDS
```

rather than automatically running PostgreSQL inside EKS.

EBS CSI is still required for workloads that genuinely need Kubernetes persistent volumes.

---

# 18. EBS CSI IAM

The CSI driver needs AWS permissions to manage EBS resources.

Modern EKS deployments should associate those permissions with the CSI controller's AWS identity rather than unnecessarily giving the EC2 nodes broad permissions.

Conceptually:

```text
EBS CSI Controller
       |
       v
Pod Identity
       |
       v
EBS CSI IAM Role
       |
       v
EC2/EBS APIs
```

---

# 19. EBS CSI Add-on

Terraform:

```hcl
resource "aws_eks_addon" "ebs_csi" {
  cluster_name = var.cluster_name

  addon_name = "aws-ebs-csi-driver"

  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  tags = var.tags
}
```

The controller's IAM association will be configured separately.

---

# 20. Why EBS CSI Uses `gp3`

For general-purpose persistent workloads:

```text
gp3
```

is often preferred over older defaults because performance and capacity can be configured independently.

Example architecture:

```text
PVC
 |
 v
StorageClass
 |
 v
gp3
 |
 v
EBS
```

We will create the StorageClass as part of the storage implementation.

---

# 21. EKS Pod Identity

This is a major part of our AWS security architecture.

Instead of:

```text
Pod
 |
 +--> AWS access key
 +--> AWS secret key
```

we use:

```text
Pod
 |
 v
EKS Pod Identity
 |
 v
IAM Role
 |
 v
AWS API
```

Example:

```text
order-service
      |
      v
SQS
```

The order service gets permission only to the required queue.

---

# 22. Why Pod Identity?

Without workload identity, developers sometimes create:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
```

inside Kubernetes Secrets.

We do **not** want that.

Problems include:

* credential rotation
* accidental leakage
* Git exposure
* container compromise
* excessive permissions

Instead:

```text
Kubernetes ServiceAccount
          |
          v
EKS Pod Identity
          |
          v
IAM Role
```

---

# 23. Pod Identity Agent

EKS provides a Pod Identity Agent add-on.

Conceptually:

```text
Application Pod
      |
      v
Pod Identity Agent
      |
      v
AWS credential vending
      |
      v
STS/IAM
```

Install:

```hcl
resource "aws_eks_addon" "pod_identity_agent" {
  cluster_name = var.cluster_name

  addon_name = "eks-pod-identity-agent"

  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "PRESERVE"

  tags = var.tags
}
```

---

# 24. Pod Identity Example

Later:

```text
order-service
```

will have:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service
  namespace: production
```

Then an AWS association maps it to:

```text
OrderServiceIAMRole
```

That IAM role may contain:

```text
sqs:SendMessage
secretsmanager:GetSecretValue
s3:GetObject
```

only for approved resources.

---

# 25. AWS Load Balancer Controller

Our users access the application through:

```text
Internet
   |
   v
ALB
   |
   v
Kubernetes Service
   |
   v
Pod
```

The AWS Load Balancer Controller manages AWS load balancers from Kubernetes resources.

For example:

```yaml
kind: Ingress
```

can result in an AWS Application Load Balancer.

---

# 26. Why We Need It

Without the controller, Kubernetes does not automatically understand all AWS-specific load-balancer behavior.

The controller manages AWS resources based on Kubernetes objects.

Conceptually:

```text
Ingress
   |
   v
AWS Load Balancer Controller
   |
   v
AWS ALB
```

---

# 27. Controller Architecture

```text
Kubernetes API
       |
       v
AWS Load Balancer Controller
       |
       +--> ALB
       +--> Target Groups
       +--> Listener
       +--> Listener Rules
```

The controller watches:

```text
Ingress
Service
TargetGroupBinding
```

and other relevant resources.

---

# 28. Controller Installation

Unlike:

```text
vpc-cni
coredns
kube-proxy
ebs-csi
```

the AWS Load Balancer Controller is generally installed as a Kubernetes controller, commonly through Helm, with its AWS permissions attached using a supported workload identity mechanism.

We will manage it through:

```text
Helm
+
GitOps
```

rather than manually installing it from a laptop.

That implementation belongs in our platform/GitOps stage.

---

# 29. Why Not Install It Manually?

Avoid:

```bash
kubectl apply -f random-url.yaml
```

for production platform components.

Instead:

```text
Git
 |
 v
Helm
 |
 v
ArgoCD
 |
 v
EKS
```

This gives:

* version control
* review
* rollback
* reproducibility
* audit trail

---

# 30. Metrics Server

Metrics Server provides resource metrics used by Kubernetes components.

Architecture:

```text
Kubelet
  |
  v
Metrics Server
  |
  v
Kubernetes Metrics API
  |
  +--> kubectl top
  |
  +--> HPA
```

Install it after the basic cluster is healthy.

---

# 31. Why Metrics Server?

Run:

```bash
kubectl top nodes
```

and:

```bash
kubectl top pods -A
```

You can then see resource usage such as:

```text
CPU
Memory
```

This is different from Prometheus.

---

# 32. Metrics Server vs Prometheus

Important distinction:

| Component      | Purpose                             |
| -------------- | ----------------------------------- |
| Metrics Server | Kubernetes resource metrics         |
| Prometheus     | Full observability/metrics platform |
| Grafana        | Visualization                       |
| Alertmanager   | Alert routing                       |

Example:

```text
Metrics Server
     |
     +--> HPA
     +--> kubectl top
```

while:

```text
Prometheus
     |
     +--> application metrics
     +--> node metrics
     +--> Kubernetes metrics
     +--> AWS metrics
     +--> alerts
```

---

# 33. Add-on Dependency Order

We should install the platform in an intentional sequence:

```text
1. EKS
   |
2. VPC CNI
   |
3. CoreDNS
   |
4. kube-proxy
   |
5. Pod Identity Agent
   |
6. EBS CSI
   |
7. Metrics Server
   |
8. Load Balancer Controller
```

Then:

```text
9. External Secrets
10. ArgoCD
11. Prometheus
12. Grafana
13. Fluent Bit
```

---

# 34. Terraform Variables

Create:

```hcl
variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
}

variable "tags" {
  description = "Tags for EKS add-ons"
  type        = map(string)
  default     = {}
}
```

---

# 35. Module Outputs

Create:

```hcl
output "vpc_cni_addon_name" {
  value = aws_eks_addon.vpc_cni.addon_name
}

output "coredns_addon_name" {
  value = aws_eks_addon.coredns.addon_name
}

output "kube_proxy_addon_name" {
  value = aws_eks_addon.kube_proxy.addon_name
}

output "ebs_csi_addon_name" {
  value = aws_eks_addon.ebs_csi.addon_name
}

output "pod_identity_addon_name" {
  value = aws_eks_addon.pod_identity_agent.addon_name
}
```

---

# 36. Root Environment

In:

```text
terraform/environments/dev/main.tf
```

add:

```hcl
module "eks_addons" {
  source = "../../modules/eks-addons"

  cluster_name = module.eks.cluster_name

  tags = {
    Environment = "dev"
    Project     = "enterprise-ecommerce"
    ManagedBy   = "terraform"
  }
}
```

---

# 37. Terraform Dependency

Terraform understands:

```text
module.eks_addons
```

depends on:

```text
module.eks
```

because:

```hcl
cluster_name = module.eks.cluster_name
```

references the EKS cluster output.

Therefore:

```text
VPC
 ↓
EKS
 ↓
Add-ons
```

Terraform can construct the dependency graph.

---

# 38. Apply

Run:

```bash
terraform fmt -recursive
```

Then:

```bash
terraform validate
```

Then:

```bash
terraform plan
```

Review carefully.

Then:

```bash
terraform apply
```

---

# 39. Verify EKS Add-ons

Run:

```bash
aws eks list-addons \
  --cluster-name ecommerce-dev \
  --region ap-south-1
```

You should see add-ons such as:

```text
vpc-cni
coredns
kube-proxy
aws-ebs-csi-driver
eks-pod-identity-agent
```

The exact list depends on what you installed.

---

# 40. Check Add-on Status

For example:

```bash
aws eks describe-addon \
  --cluster-name ecommerce-dev \
  --addon-name vpc-cni \
  --region ap-south-1 \
  --query 'addon.status'
```

Expected:

```text
ACTIVE
```

Repeat:

```bash
aws eks describe-addon \
  --cluster-name ecommerce-dev \
  --addon-name coredns \
  --region ap-south-1 \
  --query 'addon.status'
```

and:

```bash
aws eks describe-addon \
  --cluster-name ecommerce-dev \
  --addon-name kube-proxy \
  --region ap-south-1 \
  --query 'addon.status'
```

---

# 41. Kubernetes Verification

Run:

```bash
kubectl get pods -n kube-system
```

Then:

```bash
kubectl get daemonsets -n kube-system
```

Then:

```bash
kubectl get deployments -n kube-system
```

Then:

```bash
kubectl get nodes
```

---

# 42. Verify VPC CNI

```bash
kubectl get daemonset \
  -n kube-system \
  aws-node
```

Then:

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=aws-node
```

Every node should have an appropriate `aws-node` pod.

---

# 43. Verify CoreDNS

```bash
kubectl get deployment \
  -n kube-system \
  coredns
```

Then:

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=kube-dns
```

---

# 44. Verify kube-proxy

```bash
kubectl get daemonset \
  -n kube-system \
  kube-proxy
```

Then:

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=kube-proxy
```

---

# 45. Verify EBS CSI

Check:

```bash
kubectl get pods \
  -n kube-system \
  -l app.kubernetes.io/name=aws-ebs-csi-driver
```

Also:

```bash
kubectl get daemonset -n kube-system
```

and:

```bash
kubectl get deployment -n kube-system
```

The controller and node components have different responsibilities.

---

# 46. Verify Pod Identity Agent

```bash
kubectl get pods \
  -n kube-system \
  -l app.kubernetes.io/instance=eks-pod-identity-agent
```

Also:

```bash
kubectl get daemonset -n kube-system
```

You should see the agent deployed across appropriate nodes.

---

# 47. Verify Metrics Server

Once installed:

```bash
kubectl get deployment \
  -n kube-system \
  metrics-server
```

Then:

```bash
kubectl top nodes
```

Example:

```text
NAME       CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
node-1     250m         12%    2Gi             30%
node-2     300m         15%    2.1Gi           32%
```

Numbers will vary.

---

# 48. Test Kubernetes DNS

Create a temporary test pod:

```bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --restart=Never \
  -- sleep 3600
```

Then:

```bash
kubectl exec -it dns-test -- nslookup kubernetes.default
```

Expected:

```text
Server: ...
Name: kubernetes.default
Address: ...
```

Clean it up:

```bash
kubectl delete pod dns-test
```

---

# 49. Test Service Discovery

Later, when services exist:

```text
order-service
   |
   v
inventory-service
```

we'll test:

```bash
nslookup inventory-service
```

from an application pod.

This verifies:

```text
Pod
 ↓
CoreDNS
 ↓
Kubernetes Service
```

---

# 50. Test EBS Storage

We will eventually create:

```text
StorageClass
      |
      v
PVC
      |
      v
EBS CSI
      |
      v
EBS Volume
```

Do not test this by putting a production database into EKS.

Use a small temporary PVC workload first.

---

# 51. Production Storage Rule

Our architecture remains:

```text
Application state
      |
      +--> RDS
      +--> ElastiCache
      +--> S3
      +--> SQS
```

not:

```text
Everything
   |
   v
EKS StatefulSets
```

EBS CSI is infrastructure capability, not a reason to move every database into Kubernetes.

---

# 52. AWS Load Balancer Controller — Production Installation

For the AWS Load Balancer Controller, the recommended flow is:

```text
IAM Role
   |
   v
ServiceAccount
   |
   v
Helm Chart
   |
   v
AWS Load Balancer Controller
   |
   v
Ingress
   |
   v
ALB
```

We will put the Helm configuration under the platform/GitOps repository.

---

# 53. Controller IAM

The controller needs AWS permissions to manage resources such as:

```text
ALB
Target Groups
Listeners
Security Group rules
```

Do not attach:

```text
AdministratorAccess
```

to the controller.

Instead:

```text
AWS Load Balancer Controller
        |
        v
Dedicated IAM Role
        |
        v
Only required AWS permissions
```

The controller's IAM policy should be based on the AWS-published policy for the version being installed, because AWS updates those requirements over time.

---

# 54. Why Controller Version Matters

Suppose:

```text
EKS version
      +
Controller version
      +
Kubernetes version
      +
AWS APIs
```

must all be compatible.

Therefore, production installation should record:

```text
EKS version
Controller version
Helm chart version
AWS provider version
```

in Git.

---

# 55. Add-on Version Management

Do not blindly accept whatever version AWS happens to choose.

A production process is:

```text
Current EKS version
        |
        v
List compatible add-ons
        |
        v
Select approved versions
        |
        v
Terraform
        |
        v
Test DEV
        |
        v
Promote QA
        |
        v
Promote PROD
```

---

# 56. Upgrade Process

Example:

```text
Current:
EKS 1.X
CNI A
CoreDNS B
kube-proxy C
```

Upgrade:

```text
DEV
 ↓
Upgrade EKS/add-ons
 ↓
Test
 ↓
QA
 ↓
Test applications
 ↓
STAGING
 ↓
Production
```

Check:

```text
Pod networking
DNS
Load balancing
Storage
HPA
Ingress
Application startup
```

---

# 57. Troubleshooting — CNI

If:

```text
FailedCreatePodSandbox
```

check:

```bash
kubectl get pods -n kube-system
```

Then:

```bash
kubectl logs \
  -n kube-system \
  <aws-node-pod>
```

Also inspect:

```text
Subnet IP availability
ENI limits
IAM permissions
Security groups
VPC configuration
```

---

# 58. Troubleshooting — CoreDNS

Check:

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

Then:

```bash
kubectl logs \
  -n kube-system \
  <coredns-pod>
```

Check:

```bash
kubectl describe deployment \
  -n kube-system \
  coredns
```

If DNS fails:

```text
Service communication
      |
      v
DNS
      |
      v
CoreDNS
```

investigate DNS before assuming the application itself is broken.

---

# 59. Troubleshooting — EBS CSI

If a PVC stays:

```text
Pending
```

check:

```bash
kubectl describe pvc <pvc-name>
```

Then:

```bash
kubectl get storageclass
```

Then:

```bash
kubectl get pods -n kube-system
```

Check the EBS CSI controller logs.

Possible causes:

```text
IAM permissions
StorageClass
Availability Zone constraints
EBS API errors
CSI driver issue
PVC specification
```

---

# 60. Troubleshooting — Load Balancer Controller

If an Ingress doesn't create an ALB:

```bash
kubectl describe ingress <ingress-name>
```

Then:

```bash
kubectl logs \
  -n kube-system \
  deployment/aws-load-balancer-controller
```

Check:

```text
IAM role
subnet tags
security groups
Ingress annotations
controller version
AWS API access
```

---

# 61. Troubleshooting — Metrics Server

If:

```bash
kubectl top nodes
```

returns an error:

check:

```bash
kubectl get pods -n kube-system
```

Then:

```bash
kubectl logs \
  -n kube-system \
  deployment/metrics-server
```

Also verify the Metrics API:

```bash
kubectl get apiservice
```

---

# 62. Add-on Health Checklist

Before continuing:

### EKS

```bash
aws eks describe-cluster \
  --name ecommerce-dev \
  --region ap-south-1 \
  --query 'cluster.status'
```

Expected:

```text
ACTIVE
```

### Nodes

```bash
kubectl get nodes
```

Expected:

```text
Ready
```

### CNI

```bash
kubectl get pods -n kube-system -l k8s-app=aws-node
```

### CoreDNS

```bash
kubectl get pods -n kube-system -l k8s-app=kube-dns
```

### kube-proxy

```bash
kubectl get pods -n kube-system -l k8s-app=kube-proxy
```

### EBS CSI

```bash
kubectl get pods -n kube-system
```

### Pod Identity

```bash
kubectl get pods -n kube-system
```

### Metrics

```bash
kubectl top nodes
```

---

# 63. What We Have Now

The EKS platform foundation is:

```text
                         EKS
                          |
        +-----------------+-----------------+
        |                 |                 |
        v                 v                 v
      CNI              CoreDNS          kube-proxy
        |
        +-----------------+
        |
        v
    Pod Networking

        EKS
         |
         +--> EBS CSI
         |       |
         |       v
         |      EBS
         |
         +--> Pod Identity
         |       |
         |       v
         |      IAM
         |
         +--> Load Balancer Controller
         |       |
         |       v
         |      ALB
         |
         +--> Metrics Server
                 |
                 v
             HPA Metrics
```

---

# 64. Production Platform Principle

At this point, we have intentionally separated:

### Infrastructure

```text
Terraform
```

### AWS resources

```text
VPC
IAM
ECR
EKS
```

### Kubernetes platform

```text
CNI
CoreDNS
CSI
Pod Identity
Load Balancer Controller
Metrics Server
```

### Applications

```text
order-service
payment-service
...
```

This separation becomes very important when we introduce GitOps.

---

# 65. Current Project Progress

```text
53 Repository Setup
        ↓
54 Order Service
        ↓
55 Local Docker
        ↓
56 Terraform State
        ↓
57 Terraform VPC
        ↓
58 Terraform IAM
        ↓
59 Terraform ECR
        ↓
60 Terraform EKS
        ↓
61 EKS Add-ons
```

We now have:

```text
GitHub
   |
   v
Terraform
   |
   +--> VPC
   +--> IAM
   +--> ECR
   +--> EKS
          |
          +--> CNI
          +--> DNS
          +--> kube-proxy
          +--> Storage
          +--> Pod Identity
          +--> Metrics
```

---

# 66. Next Chapter

## `62-GitHub-Actions-CI.md`

Now we move from **infrastructure provisioning** to the real application delivery pipeline.

We will build the actual Order Service CI pipeline:

```text
Developer
    |
    v
Feature Branch
    |
    v
Pull Request
    |
    v
GitHub Actions
    |
    +--> Checkout
    |
    +--> Java Setup
    |
    +--> Maven Dependency Cache
    |
    +--> Compile
    |
    +--> Unit Tests
    |
    +--> SonarQube
    |
    +--> Dependency Scan
    |
    +--> Secret Scan
    |
    +--> Docker Build
    |
    +--> Trivy Image Scan
    |
    +--> AWS OIDC
    |
    +--> ECR Login
    |
    +--> Push Image
    |
    v
ECR
```

And, importantly, we will implement the **real production separation**:

```text
CI
GitHub Actions
    |
    v
ECR

        X

CD
GitOps
    |
    v
ArgoCD
    |
    v
EKS
```

The CI pipeline will **not** run:

```bash
kubectl apply
```

against production.
