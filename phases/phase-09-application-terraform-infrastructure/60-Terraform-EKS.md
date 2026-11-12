# 60-Terraform-EKS.md

# Terraform EKS — Production-Grade Amazon EKS Cluster

This chapter creates the **Amazon EKS cluster** for the Enterprise E-Commerce Platform.

At the end of this chapter, the infrastructure path will be:

```text
GitHub
   |
   v
Terraform
   |
   +------------------+
   |                  |
   v                  v
VPC                IAM
   |                  |
   +--------+---------+
            |
            v
          EKS
            |
            v
     Managed Node Groups
            |
            v
     Kubernetes Workloads
```

We are **not deploying the 21 microservices yet**.

First we build the Kubernetes platform correctly.

---

# 1. Target EKS Architecture

Our production-style architecture:

```text
                         Internet
                            |
                            v
                    Route 53 / CloudFront
                            |
                            v
                       AWS WAF
                            |
                            v
                          ALB
                            |
                    Public Subnets
                            |
                            v
                 +---------------------+
                 |      EKS Cluster    |
                 |                     |
                 |  Private Subnets    |
                 |                     |
                 | +-----------------+ |
                 | | System Nodes    | |
                 | +-----------------+ |
                 |                     |
                 | +-----------------+ |
                 | | App Nodes       | |
                 | +-----------------+ |
                 |                     |
                 |  21 Microservices  |
                 +---------------------+
                            |
          +-----------------+------------------+
          |                 |                  |
          v                 v                  v
         RDS              Redis              SQS/SNS
```

The EKS worker nodes are private.

We don't expose individual Kubernetes nodes directly to the Internet.

---

# 2. Why EKS?

Instead of manually creating Kubernetes control-plane infrastructure:

```text
EC2
  |
  +--> etcd
  +--> API Server
  +--> Scheduler
  +--> Controller Manager
```

AWS manages the EKS control plane.

We manage:

```text
VPC
IAM
Node Groups
Add-ons
Workloads
Networking
Security
Scaling
Observability
Deployments
```

Conceptually:

```text
AWS Managed
----------------
EKS Control Plane


Our Responsibility
------------------
Worker Nodes
Applications
Kubernetes Objects
Networking Configuration
Security Configuration
Observability
GitOps
```

---

# 3. EKS Components

An EKS cluster contains:

```text
EKS
│
├── Control Plane
│   ├── Kubernetes API Server
│   ├── Scheduler
│   ├── Controller Manager
│   └── etcd
│
├── Worker Nodes
│   ├── kubelet
│   ├── container runtime
│   └── kube-proxy
│
└── Add-ons
    ├── VPC CNI
    ├── CoreDNS
    ├── kube-proxy
    ├── EBS CSI Driver
    └── AWS Load Balancer Controller
```

Later:

```text
Prometheus
Grafana
Fluent Bit
External Secrets
ArgoCD
KEDA
```

will run as platform workloads.

---

# 4. Terraform Repository

Our repository:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   ├── vpc/
    │   ├── iam/
    │   ├── ecr/
    │   └── eks/
    │
    └── environments/
        ├── dev/
        ├── qa/
        └── prod/
```

Create:

```bash
cd infrastructure/terraform/modules

mkdir -p eks
cd eks
```

Files:

```text
eks/
├── main.tf
├── variables.tf
├── outputs.tf
└── versions.tf
```

---

# 5. EKS Module — `versions.tf`

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}
```

The root environment should pin the provider version and commit:

```text
.terraform.lock.hcl
```

---

# 6. EKS Variables

Create:

```text
variables.tf
```

```hcl
variable "cluster_name" {
  description = "EKS cluster name"
  type        = string
}

variable "kubernetes_version" {
  description = "Kubernetes version for EKS"
  type        = string
}

variable "subnet_ids" {
  description = "Private subnet IDs for EKS"
  type        = list(string)
}

variable "cluster_security_group_ids" {
  description = "Security groups attached to the EKS cluster"
  type        = list(string)
  default     = []
}

variable "node_instance_types" {
  description = "EC2 instance types for worker nodes"
  type        = list(string)

  default = [
    "m6i.large"
  ]
}

variable "node_min_size" {
  type    = number
  default = 2
}

variable "node_max_size" {
  type    = number
  default = 6
}

variable "node_desired_size" {
  type    = number
  default = 3
}

variable "tags" {
  description = "Tags for EKS resources"
  type        = map(string)
  default     = {}
}
```

---

# 7. EKS Cluster IAM Role

We previously discussed IAM.

The EKS control plane needs an IAM role.

The architecture is:

```text
EKS
 |
 v
Cluster IAM Role
 |
 v
AWS APIs
```

We can create it inside the EKS module.

```hcl
resource "aws_iam_role" "cluster" {
  name = "${var.cluster_name}-cluster-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"

    Statement = [
      {
        Effect = "Allow"

        Principal = {
          Service = "eks.amazonaws.com"
        }

        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = var.tags
}
```

---

# 8. Attach Cluster Policy

```hcl
resource "aws_iam_role_policy_attachment" "cluster_policy" {
  role = aws_iam_role.cluster.name

  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
}
```

AWS periodically evolves EKS IAM patterns, so when implementing against a specific current AWS environment, verify the current EKS documentation and provider behavior.

---

# 9. Create EKS Cluster

Now:

```hcl
resource "aws_eks_cluster" "this" {
  name     = var.cluster_name
  role_arn = aws_iam_role.cluster.arn

  version = var.kubernetes_version

  vpc_config {
    subnet_ids              = var.subnet_ids
    security_group_ids      = var.cluster_security_group_ids
    endpoint_private_access = true
    endpoint_public_access  = true
  }

  tags = var.tags

  depends_on = [
    aws_iam_role_policy_attachment.cluster_policy
  ]
}
```

---

# 10. API Endpoint Configuration

We have:

```hcl
endpoint_private_access = true
endpoint_public_access  = true
```

This means the Kubernetes API endpoint is reachable through:

```text
Private networking
+
Controlled public access
```

For a production environment, public API access should be tightly restricted to approved CIDRs or avoided where the operational model allows.

Do **not** expose the Kubernetes API broadly to:

```text
0.0.0.0/0
```

without a deliberate security design.

---

# 11. Why Private Access?

Worker nodes are in private subnets:

```text
Private Subnet
     |
     v
EKS Node
     |
     v
EKS API
```

This avoids requiring nodes to communicate with the control plane through an unnecessarily public path.

---

# 12. EKS Subnets

From Chapter 57:

```text
VPC
10.0.0.0/16

Public:
10.0.1.0/24
10.0.2.0/24
10.0.3.0/24

Private:
10.0.11.0/24
10.0.12.0/24
10.0.13.0/24
```

EKS nodes:

```text
Private-A
Private-B
Private-C
```

not:

```text
Public-A
Public-B
Public-C
```

---

# 13. Why Three AZs?

Production:

```text
AZ-A
 |
 +--> Nodes

AZ-B
 |
 +--> Nodes

AZ-C
 |
 +--> Nodes
```

If one AZ has a failure:

```text
AZ-A DOWN

AZ-B -> workloads
AZ-C -> workloads
```

This requires enough capacity in the remaining AZs and correct topology configuration.

---

# 14. EKS Cluster Security Group

We can create a dedicated security group.

Example:

```hcl
resource "aws_security_group" "cluster" {
  name        = "${var.cluster_name}-cluster-sg"
  description = "Security group for EKS cluster"
  vpc_id      = var.vpc_id

  tags = merge(
    var.tags,
    {
      Name = "${var.cluster_name}-cluster-sg"
    }
  )
}
```

Add variable:

```hcl
variable "vpc_id" {
  description = "VPC ID for the EKS cluster"
  type        = string
}
```

The exact ingress/egress rules should be designed around EKS requirements and the chosen networking/add-on architecture rather than blindly opening all traffic.

---

# 15. EKS Cluster Logging

For production troubleshooting, control-plane logs are valuable.

We can enable:

```hcl
enabled_cluster_log_types = [
  "api",
  "audit",
  "authenticator",
  "controllerManager",
  "scheduler"
]
```

Example:

```hcl
resource "aws_eks_cluster" "this" {
  name     = var.cluster_name
  role_arn = aws_iam_role.cluster.arn

  version = var.kubernetes_version

  enabled_cluster_log_types = [
    "api",
    "audit",
    "authenticator",
    "controllerManager",
    "scheduler"
  ]

  vpc_config {
    subnet_ids              = var.subnet_ids
    endpoint_private_access = true
    endpoint_public_access  = true
  }

  depends_on = [
    aws_iam_role_policy_attachment.cluster_policy
  ]
}
```

These logs can be useful for:

```text
authentication failures
API errors
RBAC issues
control-plane behavior
Kubernetes API investigation
```

There is a CloudWatch Logs cost, so production retention should be designed intentionally.

---

# 16. Worker Node IAM Role

The control plane role and node role are different.

Architecture:

```text
EKS Control Plane
       |
       v
Cluster IAM Role


EC2 Worker Node
       |
       v
Node IAM Role
```

Create:

```hcl
resource "aws_iam_role" "node" {
  name = "${var.cluster_name}-node-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"

    Statement = [
      {
        Effect = "Allow"

        Principal = {
          Service = "ec2.amazonaws.com"
        }

        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = var.tags
}
```

---

# 17. Node Policies

A common managed-node configuration uses policies such as:

```text
AmazonEKSWorkerNodePolicy
AmazonEC2ContainerRegistryPullOnly
```

and appropriate networking permissions.

Historically, `AmazonEKS_CNI_Policy` was commonly attached to the node role.

For a more secure modern architecture, VPC CNI permissions can be separated from the node role using the supported AWS identity mechanism for the CNI.

That gives:

```text
Node Role
    |
    +--> node functionality

VPC CNI identity
    |
    +--> ENI/IP management
```

rather than unnecessarily giving every node broad networking permissions.

We will implement the exact add-on identity configuration in Chapter 61.

---

# 18. Node Group

Now create a managed node group.

```hcl
resource "aws_eks_node_group" "system" {
  cluster_name = aws_eks_cluster.this.name

  node_group_name = "${var.cluster_name}-system"

  node_role_arn = aws_iam_role.node.arn

  subnet_ids = var.subnet_ids

  instance_types = var.node_instance_types

  capacity_type = "ON_DEMAND"

  scaling_config {
    desired_size = var.node_desired_size
    min_size     = var.node_min_size
    max_size     = var.node_max_size
  }

  update_config {
    max_unavailable = 1
  }

  labels = {
    workload = "system"
  }

  tags = merge(
    var.tags,
    {
      Name = "${var.cluster_name}-system"
    }
  )

  depends_on = [
    aws_iam_role_policy_attachment.node_worker,
    aws_iam_role_policy_attachment.node_ecr
  ]
}
```

---

# 19. Node Policies

Example:

```hcl
resource "aws_iam_role_policy_attachment" "node_worker" {
  role = aws_iam_role.node.name

  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
}

resource "aws_iam_role_policy_attachment" "node_ecr" {
  role = aws_iam_role.node.name

  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryPullOnly"
}
```

The CNI identity will be addressed with the EKS add-on implementation.

---

# 20. Why Managed Node Groups?

Instead of manually managing:

```text
EC2
AMI
kubelet
bootstrap
Auto Scaling Group
```

AWS manages important node-group lifecycle operations.

We still control:

```text
instance type
capacity
scaling
labels
taints
subnets
update behavior
```

---

# 21. System Nodes vs Application Nodes

Our production design should eventually separate workloads.

```text
EKS
│
├── System Node Group
│   ├── CoreDNS
│   ├── kube-proxy
│   ├── VPC CNI
│   └── platform components
│
└── Application Node Group
    ├── auth-service
    ├── order-service
    ├── payment-service
    └── ...
```

Why?

If application workloads consume all CPU/memory:

```text
Application Pods
       |
       v
Node capacity exhausted
       |
       v
System components affected
```

Separating capacity improves isolation.

---

# 22. System Node Taints

Later we can use:

```text
taint:
workload=system:NoSchedule
```

and platform workloads use:

```yaml
tolerations:
  - key: workload
    operator: Equal
    value: system
    effect: NoSchedule
```

This prevents normal application pods from consuming system-only nodes.

We don't need to add this immediately while building the initial cluster.

---

# 23. Application Node Group

Later:

```hcl
resource "aws_eks_node_group" "application" {
  cluster_name = aws_eks_cluster.this.name

  node_group_name = "${var.cluster_name}-application"

  node_role_arn = aws_iam_role.node.arn

  subnet_ids = var.subnet_ids

  instance_types = [
    "m6i.large",
    "m6i.xlarge"
  ]

  capacity_type = "ON_DEMAND"

  scaling_config {
    desired_size = 3
    min_size     = 3
    max_size     = 10
  }

  labels = {
    workload = "application"
  }
}
```

Later, Karpenter can dynamically provision nodes instead.

---

# 24. Why Not Start With Karpenter?

Because we want to understand the fundamental architecture first.

Initial:

```text
EKS
 |
 v
Managed Node Group
 |
 v
Application
```

Later:

```text
EKS
 |
 v
Karpenter
 |
 +--> NodePool
 |
 +--> NodeClass
 |
 +--> Dynamic EC2 capacity
```

Karpenter is an optimization/scaling layer, not a prerequisite for understanding EKS.

---

# 25. EKS Outputs

Create:

```text
outputs.tf
```

```hcl
output "cluster_name" {
  description = "EKS cluster name"
  value       = aws_eks_cluster.this.name
}

output "cluster_endpoint" {
  description = "EKS API endpoint"
  value       = aws_eks_cluster.this.endpoint
}

output "cluster_arn" {
  description = "EKS cluster ARN"
  value       = aws_eks_cluster.this.arn
}

output "cluster_certificate_authority" {
  description = "EKS cluster certificate authority"
  value       = aws_eks_cluster.this.certificate_authority[0].data
}

output "node_group_name" {
  description = "System node group name"
  value       = aws_eks_node_group.system.node_group_name
}
```

---

# 26. Complete Initial EKS Module

Conceptually:

```text
modules/eks/
│
├── versions.tf
├── variables.tf
├── main.tf
└── outputs.tf
```

Architecture:

```text
main.tf
│
├── Cluster IAM Role
│
├── Cluster IAM Policy
│
├── EKS Cluster
│
├── Node IAM Role
│
├── Node Policies
│
└── Managed Node Group
```

---

# 27. Root Environment

Now:

```text
terraform/environments/dev/
```

will call the module.

Example:

```hcl
module "eks" {
  source = "../../modules/eks"

  cluster_name = "ecommerce-dev"

  kubernetes_version = var.kubernetes_version

  vpc_id = module.vpc.vpc_id

  subnet_ids = module.vpc.private_subnet_ids

  node_instance_types = [
    "m6i.large"
  ]

  node_min_size     = 2
  node_max_size     = 6
  node_desired_size = 3

  tags = {
    Environment = "dev"
    Project     = "enterprise-ecommerce"
    ManagedBy   = "terraform"
  }
}
```

The exact VPC output names must match the VPC module we implemented in Chapter 57.

---

# 28. Kubernetes Version

Do not blindly hardcode a version forever.

Use:

```hcl
variable "kubernetes_version" {
  type    = string
  default = "CURRENT_SUPPORTED_VERSION"
}
```

Before applying, verify which Kubernetes versions the selected EKS region currently supports.

EKS versions have a lifecycle.

Therefore:

```text
Current supported version
        |
        v
EKS
```

should be treated as an operational decision.

Do not copy an old version from a tutorial and assume it is still supported.

---

# 29. Kubernetes Version Upgrade Strategy

Never approach production upgrades as:

```text
terraform apply
```

and hope everything works.

Instead:

```text
DEV
 ↓
Validate
 ↓
QA
 ↓
STAGING
 ↓
PROD
```

Check:

```text
Kubernetes compatibility
EKS add-ons
CNI
CoreDNS
kube-proxy
CSI
Ingress controller
Helm charts
Admission controllers
Applications
```

---

# 30. Terraform Plan

From:

```bash
cd infrastructure/terraform/environments/dev
```

run:

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

Expected high-level resources:

```text
EKS cluster
Cluster IAM role
Cluster policy attachment
Node IAM role
Node policy attachments
Managed node group
```

Plus the VPC/IAM/ECR resources already defined in the environment.

---

# 31. Apply

```bash
terraform apply
```

Terraform will create:

```text
VPC
 |
 +--> subnets
 +--> route tables
 +--> NAT
 |
 v
EKS
 |
 +--> control plane
 |
 +--> node group
```

EKS cluster creation can take several minutes.

Do not terminate Terraform midway just because cluster creation appears slow.

---

# 32. Verify Cluster

AWS CLI:

```bash
aws eks describe-cluster \
  --name ecommerce-dev \
  --region ap-south-1
```

Check status:

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

---

# 33. Configure `kubectl`

Run:

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name ecommerce-dev
```

Expected behavior:

```text
Updated context ...
```

Check:

```bash
kubectl config current-context
```

Then:

```bash
kubectl get nodes
```

---

# 34. Expected Node Output

Example:

```text
NAME                                           STATUS   ROLES    AGE   VERSION
ip-10-0-11-100.ap-south-1.compute.internal   Ready    <none>   5m    v1.xx
ip-10-0-12-101.ap-south-1.compute.internal   Ready    <none>   5m    v1.xx
ip-10-0-13-102.ap-south-1.compute.internal   Ready    <none>   5m    v1.xx
```

The exact names and Kubernetes version will differ.

---

# 35. Verify Nodes Across AZs

Get node information:

```bash
kubectl get nodes -o wide
```

Then:

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

You want to see nodes distributed across multiple AZs.

Example:

```text
NODE        ZONE
node-1      ap-south-1a
node-2      ap-south-1b
node-3      ap-south-1c
```

---

# 36. Verify System Pods

```bash
kubectl get pods -A
```

Initially you'll see components such as:

```text
kube-system
```

including:

```text
CoreDNS
kube-proxy
aws-node
```

Exact pods depend on the EKS version and add-on configuration.

---

# 37. Check CoreDNS

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=kube-dns
```

Expected:

```text
Running
```

Check:

```bash
kubectl get deployment \
  -n kube-system \
  coredns
```

---

# 38. Check VPC CNI

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=aws-node
```

Expected:

```text
Running
```

VPC CNI is critical because it provides Kubernetes networking integration with AWS VPC networking.

---

# 39. Check kube-proxy

```bash
kubectl get daemonset \
  -n kube-system \
  kube-proxy
```

Check:

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=kube-proxy
```

---

# 40. What VPC CNI Does

Conceptually:

```text
Pod
 |
 v
AWS VPC CNI
 |
 v
ENI / secondary IP
 |
 v
VPC
```

Therefore, unlike some overlay networking models, Kubernetes pod networking can consume actual VPC IP capacity.

This is why our earlier subnet/IP planning matters.

---

# 41. IP Exhaustion

Suppose:

```text
Private subnet
10.0.11.0/24
```

has limited IP addresses.

Then:

```text
Many Pods
    |
    v
Many IP addresses
    |
    v
Subnet exhaustion
```

Symptoms can include:

```text
Pod Pending
CNI errors
FailedCreatePodSandbox
IP allocation failures
```

This is why EKS capacity planning is not only about EC2 CPU/memory.

It is also:

```text
CPU
Memory
IP addresses
ENIs
Pod density
```

---

# 42. Node Capacity

Check:

```bash
kubectl describe node <node-name>
```

Look for:

```text
Capacity
Allocatable
```

Example:

```text
Capacity:
  cpu: 2
  memory: ...
  pods: 29

Allocatable:
  cpu: ...
  memory: ...
  pods: ...
```

Do not schedule based solely on raw EC2 vCPU.

Kubernetes reserves resources for system processes and platform components.

---

# 43. Node Labels

Check:

```bash
kubectl get nodes --show-labels
```

We should eventually use labels such as:

```text
workload=system
workload=application
```

Then workloads can target appropriate nodes.

---

# 44. Node Taints

Check:

```bash
kubectl get nodes -o custom-columns=NAME:.metadata.name,TAINTS:.spec.taints
```

Taints allow us to control scheduling.

Example:

```text
workload=system:NoSchedule
```

Application pods without the corresponding toleration cannot schedule there.

---

# 45. EKS Access

There are two related concepts:

```text
AWS IAM
   |
   v
Who can access EKS
   |
   v
Kubernetes authorization
```

A user may authenticate to AWS but still not have the Kubernetes permissions required to perform an operation.

This distinction is critical:

```text
Authentication
      ≠
Authorization
```

---

# 46. Don't Give Everyone `cluster-admin`

Avoid:

```text
cluster-admin
```

for every engineer.

Instead use:

```text
Developer
   |
   +--> namespace read/deploy permissions

DevOps
   |
   +--> infrastructure/platform permissions

SRE
   |
   +--> operational permissions

Platform Admin
   |
   +--> cluster administration
```

RBAC will be implemented in the platform/security chapters.

---

# 47. EKS Node Security

Nodes are private.

Inbound access should be minimized.

Do not build a workflow around:

```text
Internet
   |
   v
SSH
   |
   v
EKS Node
```

For operational access, prefer managed AWS access mechanisms such as Systems Manager where appropriate.

---

# 48. No SSH Keys in Git

Never commit:

```text
*.pem
private-key.pem
aws-access-key
```

to:

```text
infrastructure/
```

or:

```text
gitops/
```

AWS authentication should use:

```text
IAM
OIDC
STS
Pod Identity
```

rather than static credentials.

---

# 49. EKS + ECR

Now we have:

```text
GitHub Actions
      |
      | OIDC
      v
AWS IAM
      |
      v
ECR
      |
      | image
      v
EKS
```

The EKS node/workload side needs permission to pull:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:<tag>
```

This will be tested when we deploy the Order Service.

---

# 50. EKS + RDS

Our future architecture:

```text
Order Service Pod
       |
       v
PostgreSQL
       |
       v
Amazon RDS
```

The application pod should **not** receive broad access to the entire VPC.

Security groups and database access will restrict:

```text
EKS workload
      |
      | TCP 5432
      v
RDS PostgreSQL
```

only where required.

---

# 51. EKS + Redis

Similarly:

```text
Order Service
      |
      | Redis
      v
ElastiCache
```

Access is controlled using:

```text
Security Groups
IAM where applicable
Secrets
Network controls
```

---

# 52. EKS + SQS

The application should not receive AWS access keys.

Instead:

```text
Order Pod
    |
    v
EKS Pod Identity
    |
    v
IAM Role
    |
    v
SQS
```

For example:

```text
order-service
      |
      +--> sqs:SendMessage
              |
              v
        order-events queue
```

Only the required queue should be permitted.

---

# 53. EKS + Secrets Manager

Similarly:

```text
Order Pod
    |
    v
EKS Pod Identity
    |
    v
IAM Role
    |
    v
Secrets Manager
```

The pod retrieves:

```text
DB credentials
API credentials
application secrets
```

without putting AWS credentials inside the container.

---

# 54. Why We Don't Deploy Applications Yet

We could now run:

```bash
kubectl apply
```

but we deliberately won't.

We first need:

```text
EKS
 |
 +--> VPC CNI
 +--> CoreDNS
 +--> kube-proxy
 +--> EBS CSI
 +--> Load Balancer Controller
 +--> Pod Identity
```

Then:

```text
External Secrets
ArgoCD
Prometheus
Grafana
Fluent Bit
```

and other platform components.

Only then should we deploy:

```text
order-service
```

through Helm + GitOps.

---

# 55. EKS Creation Troubleshooting

## Problem: Cluster Stuck Creating

Check:

```bash
aws eks describe-cluster \
  --name ecommerce-dev \
  --region ap-south-1 \
  --query 'cluster.status'
```

Then inspect Terraform:

```bash
terraform state list
```

Check:

```bash
terraform show
```

Possible causes include:

```text
IAM permissions
subnet configuration
security groups
AWS service limits
incorrect VPC configuration
unsupported Kubernetes version
```

---

# 56. Problem: Nodes Not Joining

Check:

```bash
aws eks describe-nodegroup \
  --cluster-name ecommerce-dev \
  --nodegroup-name ecommerce-dev-system \
  --region ap-south-1
```

Check:

```bash
kubectl get nodes
```

If no nodes appear, investigate:

```text
Node IAM role
Subnet routing
Security groups
EKS bootstrap
AMI compatibility
VPC CNI
EC2 capacity
```

---

# 57. Problem: Nodes `NotReady`

Run:

```bash
kubectl describe node <node-name>
```

Then:

```bash
kubectl get pods -n kube-system
```

Check:

```text
aws-node
kube-proxy
CoreDNS
```

Look at events:

```bash
kubectl get events -A --sort-by=.lastTimestamp
```

---

# 58. Problem: Pod Networking Failure

Check:

```bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=aws-node
```

Then:

```bash
kubectl logs \
  -n kube-system \
  <aws-node-pod>
```

Check subnet IP availability:

```text
AWS VPC
  |
  +--> Subnet available IPs
```

Also inspect EC2 ENIs.

---

# 59. Problem: `kubectl` Cannot Connect

Run:

```bash
kubectl config current-context
```

Then:

```bash
aws sts get-caller-identity
```

Then:

```bash
aws eks describe-cluster \
  --name ecommerce-dev \
  --region ap-south-1
```

Possible causes:

```text
Wrong AWS account
Wrong region
Wrong kubeconfig
IAM access not configured
API endpoint inaccessible
CIDR restrictions
Expired/invalid AWS credentials
```

---

# 60. Verify Everything

Run:

```bash
aws eks describe-cluster \
  --name ecommerce-dev \
  --region ap-south-1 \
  --query 'cluster.status'
```

Then:

```bash
kubectl get nodes
```

Then:

```bash
kubectl get pods -A
```

Then:

```bash
kubectl get nodes \
  -L topology.kubernetes.io/zone
```

Then:

```bash
kubectl get daemonsets -n kube-system
```

Then:

```bash
kubectl get deployments -n kube-system
```

---

# 61. Current Architecture

We now have:

```text
                         AWS
                          |
                 +--------+--------+
                 |                 |
                VPC               IAM
                 |                 |
                 +--------+--------+
                          |
                          v
                       EKS
                          |
              +-----------+-----------+
              |                       |
       Control Plane             Worker Nodes
              |                       |
              |              +--------+--------+
              |              |                 |
              |          System Nodes      App Nodes
              |              |                 |
              |              |                 |
              +--------------+-----------------+
                             |
                    Kubernetes Platform
```

---

# 62. Infrastructure Progress

Our implementation is now:

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
```

The infrastructure path is now:

```text
GitHub
   ↓
Terraform
   ↓
S3 Terraform State
   ↓
VPC
   ↓
IAM
   ↓
ECR
   ↓
EKS
```

---

# 63. Next Chapter

## `61-EKS-Addons.md`

This is the next critical platform chapter.

We will install and configure the components that make EKS production-ready:

```text
EKS
 |
 +--> VPC CNI
 |
 +--> CoreDNS
 |
 +--> kube-proxy
 |
 +--> EBS CSI Driver
 |
 +--> EKS Pod Identity
 |
 +--> AWS Load Balancer Controller
 |
 +--> Metrics Server
```

Then we will establish:

```text
EKS
 |
 +--> Networking
 +--> DNS
 +--> Storage
 +--> IAM for Pods
 +--> Load Balancing
 +--> Metrics
```

**Only after these are working will we move to `62-GitHub-Actions-CI.md` and build the real CI pipeline that takes the Order Service from GitHub → tests → SonarQube → Trivy → Docker → ECR.**
