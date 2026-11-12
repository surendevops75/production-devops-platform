# 57 — Terraform VPC Implementation

> **Project:** Enterprise E-Commerce Platform
> **AWS Region:** `ap-south-1`
> **VPC CIDR:** `10.0.0.0/16`
> **Availability Zones:** 3
> **IaC:** Terraform
> **Target:** Production-style EKS networking

This is the first major AWS infrastructure layer.

---

# 1. What We Are Building

Our target architecture:

```text
                         INTERNET
                             │
                             ▼
                    Internet Gateway
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
   Public Subnet A      Public Subnet B      Public Subnet C
    10.0.1.0/24          10.0.2.0/24          10.0.3.0/24
        │                    │                    │
      NAT A                NAT B                NAT C
        │                    │                    │
        ▼                    ▼                    ▼
  Private Subnet A     Private Subnet B     Private Subnet C
   10.0.11.0/24         10.0.12.0/24         10.0.13.0/24
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                             ▼
                           EKS
```

Later:

```text
Internet
   │
   ▼
Route 53
   │
   ▼
CloudFront/WAF
   │
   ▼
ALB
   │
   ▼
EKS Pods
```

And private workloads:

```text
EKS
 │
 ├── RDS
 ├── Redis
 ├── SQS/SNS
 ├── ECR
 ├── Secrets Manager
 └── S3
```

---

# 2. Why VPC Comes Before EKS

EKS needs networking.

The EKS nodes and Pods need:

```text
IP addresses
subnets
routing
security groups
DNS
internet egress
AWS service connectivity
```

Therefore:

```text
Terraform State
      ↓
VPC
      ↓
IAM
      ↓
ECR
      ↓
EKS
```

We don't create EKS first and figure out networking afterward.

---

# 3. VPC CIDR

Use:

```text
10.0.0.0/16
```

This gives:

```text
65,536 IPv4 addresses
```

We divide it into:

### Public

```text
10.0.1.0/24
10.0.2.0/24
10.0.3.0/24
```

### Private

```text
10.0.11.0/24
10.0.12.0/24
10.0.13.0/24
```

Later, if required, additional subnet groups can be added.

---

# 4. Why Private Subnets for EKS Nodes?

We don't want worker nodes directly exposed to the Internet.

Bad:

```text
Internet
   │
   ▼
Public EC2/EKS nodes
```

Preferred:

```text
Internet
   │
   ▼
ALB
   │
   ▼
Private EKS nodes
   │
   ▼
Pods
```

Outbound access:

```text
Private Node
     │
     ▼
NAT Gateway
     │
     ▼
Internet Gateway
     │
     ▼
Internet
```

---

# 5. Three Availability Zones

Use:

```text
ap-south-1a
ap-south-1b
ap-south-1c
```

Architecture:

```text
                  VPC
                   │
       ┌───────────┼───────────┐
       │           │           │
       ▼           ▼           ▼
      AZ-A        AZ-B        AZ-C
       │           │           │
       ▼           ▼           ▼
   Public A     Public B     Public C
   Private A    Private B    Private C
```

This allows workloads to spread across AZs.

---

# 6. Repository Structure

In:

```text
company-ecommerce/infrastructure
```

we'll use:

```text
terraform/
│
├── bootstrap/
│   └── state/
│
├── modules/
│   └── vpc/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── versions.tf
│
└── environments/
    └── dev/
        ├── backend.tf
        ├── provider.tf
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        └── terraform.tfvars
```

Later:

```text
environments/
├── dev/
├── qa/
└── prod/
```

---

# 7. VPC Module

Create:

```text
terraform/modules/vpc/
```

This module should contain reusable networking logic.

The environment decides:

```text
CIDR
AZs
environment name
NAT strategy
```

The module creates:

```text
VPC
Subnets
Route tables
Internet Gateway
NAT Gateway
Elastic IPs
VPC endpoints
```

---

# 8. Module `versions.tf`

Create:

```hcl
terraform/modules/vpc/versions.tf
```

```hcl
terraform {
  required_version = ">= 1.10.0"

  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}
```

Provider version selection remains controlled by the root environment's lock file.

---

# 9. VPC Variables

Create:

```hcl
terraform/modules/vpc/variables.tf
```

```hcl
variable "name" {
  description = "Name prefix for VPC resources"
  type        = string
}

variable "vpc_cidr" {
  description = "CIDR block for the VPC"
  type        = string
}

variable "availability_zones" {
  description = "Availability zones used by the VPC"
  type        = list(string)
}

variable "public_subnet_cidrs" {
  description = "CIDRs for public subnets"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "CIDRs for private subnets"
  type        = list(string)
}

variable "enable_nat_gateway" {
  description = "Whether NAT gateways should be created"
  type        = bool
  default     = true
}

variable "single_nat_gateway" {
  description = "Use one NAT gateway instead of one per AZ"
  type        = bool
  default     = false
}

variable "enable_vpc_endpoints" {
  description = "Whether interface/gateway VPC endpoints should be created"
  type        = bool
  default     = true
}
```

---

# 10. VPC Resource

Create:

```text
terraform/modules/vpc/main.tf
```

```hcl
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = var.name
  }
}
```

Two important settings:

```text
enable_dns_support
enable_dns_hostnames
```

Both should be enabled for our EKS architecture.

---

# 11. Internet Gateway

```hcl
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${var.name}-igw"
  }
}
```

Traffic path:

```text
Public subnet
     │
     ▼
Route table
     │
     ▼
0.0.0.0/0
     │
     ▼
Internet Gateway
     │
     ▼
Internet
```

---

# 12. Public Subnets

```hcl
resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id = aws_vpc.this.id

  cidr_block = var.public_subnet_cidrs[count.index]

  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name = "${var.name}-public-${var.availability_zones[count.index]}"

    "kubernetes.io/role/elb" = "1"
  }
}
```

The Kubernetes tag:

```text
kubernetes.io/role/elb
```

helps AWS Load Balancer Controller identify subnets suitable for internet-facing load balancers.

---

# 13. Private Subnets

```hcl
resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id = aws_vpc.this.id

  cidr_block = var.private_subnet_cidrs[count.index]

  availability_zone = var.availability_zones[count.index]

  map_public_ip_on_launch = false

  tags = {
    Name = "${var.name}-private-${var.availability_zones[count.index]}"

    "kubernetes.io/role/internal-elb" = "1"
  }
}
```

For EKS, we'll also add the Kubernetes cluster discovery tag later.

---

# 14. Why `map_public_ip_on_launch = false`?

Private subnets:

```text
Private subnet
      │
      ▼
EC2/EKS node
      │
      X
Public IPv4
```

Nodes should not automatically receive public IP addresses.

Instead:

```text
Node
 │
 ▼
NAT Gateway
 │
 ▼
Internet
```

for outbound internet access.

---

# 15. Public Route Table

```hcl
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${var.name}-public-rt"
  }
}
```

Add default route:

```hcl
resource "aws_route" "public_internet" {
  route_table_id         = aws_route_table.public.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.this.id
}
```

---

# 16. Associate Public Subnets

```hcl
resource "aws_route_table_association" "public" {
  count = length(var.availability_zones)

  subnet_id = aws_subnet.public[count.index].id

  route_table_id = aws_route_table.public.id
}
```

Now:

```text
Public A
   │
   ├── Public Route Table
   │
   └── IGW
```

and similarly for B/C.

---

# 17. Elastic IPs for NAT

For production, we want one NAT Gateway per AZ.

```hcl
resource "aws_eip" "nat" {
  count = var.enable_nat_gateway ? (
    var.single_nat_gateway ? 1 : length(var.availability_zones)
  ) : 0

  domain = "vpc"

  tags = {
    Name = "${var.name}-nat-eip-${count.index + 1}"
  }
}
```

---

# 18. NAT Gateways

```hcl
resource "aws_nat_gateway" "this" {
  count = var.enable_nat_gateway ? (
    var.single_nat_gateway ? 1 : length(var.availability_zones)
  ) : 0

  allocation_id = aws_eip.nat[count.index].id

  subnet_id = aws_subnet.public[
    var.single_nat_gateway ? 0 : count.index
  ].id

  tags = {
    Name = "${var.name}-nat-${count.index + 1}"
  }

  depends_on = [
    aws_internet_gateway.this
  ]
}
```

Architecture:

```text
AZ-A                         AZ-B                         AZ-C

Private A                    Private B                    Private C
   │                            │                            │
   ▼                            ▼                            ▼
NAT A                         NAT B                         NAT C
   │                            │                            │
   ▼                            ▼                            ▼
Public A                     Public B                     Public C
   │                            │                            │
   └─────────────── Internet Gateway ──────────────────────┘
```

---

# 19. Why One NAT Per AZ?

Suppose:

```text
Private A → NAT A
Private B → NAT B
Private C → NAT C
```

If AZ-A has a problem:

```text
AZ-A failure
    │
    X
NAT A unavailable
```

B and C remain independent.

This reduces cross-AZ dependency.

The trade-off is cost.

---

# 20. Development NAT Strategy

For development, you might use:

```hcl
single_nat_gateway = true
```

Architecture:

```text
Private A ─┐
Private B ─┼──> NAT A
Private C ─┘
```

Cheaper, but less resilient.

Production:

```hcl
single_nat_gateway = false
```

Target:

```text
NAT A
NAT B
NAT C
```

---

# 21. Private Route Tables

Create one route table per private subnet/AZ:

```hcl
resource "aws_route_table" "private" {
  count = length(var.availability_zones)

  vpc_id = aws_vpc.this.id

  tags = {
    Name = "${var.name}-private-rt-${var.availability_zones[count.index]}"
  }
}
```

---

# 22. Private Default Routes

```hcl
resource "aws_route" "private_nat" {
  count = var.enable_nat_gateway ? length(var.availability_zones) : 0

  route_table_id = aws_route_table.private[count.index].id

  destination_cidr_block = "0.0.0.0/0"

  nat_gateway_id = aws_nat_gateway.this[
    var.single_nat_gateway ? 0 : count.index
  ].id
}
```

---

# 23. Associate Private Subnets

```hcl
resource "aws_route_table_association" "private" {
  count = length(var.availability_zones)

  subnet_id = aws_subnet.private[count.index].id

  route_table_id = aws_route_table.private[count.index].id
}
```

Now:

```text
Private subnet
      │
      ▼
Private route table
      │
      ▼
NAT Gateway
      │
      ▼
Internet Gateway
      │
      ▼
Internet
```

---

# 24. Why Private Workloads Need NAT

Private EKS nodes may need outbound access for things such as:

```text
OS package repositories
container registries
external APIs
software downloads
AWS endpoints not reached through VPC endpoints
```

Without NAT:

```text
Private Node
    │
    X
Internet
```

With NAT:

```text
Private Node
    │
    ▼
NAT
    │
    ▼
IGW
    │
    ▼
Internet
```

---

# 25. VPC Endpoints

NAT shouldn't necessarily carry every AWS service request.

We can use VPC endpoints.

Common services:

```text
S3
ECR API
ECR DKR
CloudWatch Logs
STS
Secrets Manager
SSM
EC2
```

The exact endpoint set should be based on which AWS APIs the workloads actually use.

---

# 26. S3 Gateway Endpoint

S3 is a common example.

```hcl
resource "aws_vpc_endpoint" "s3" {
  count = var.enable_vpc_endpoints ? 1 : 0

  vpc_id = aws_vpc.this.id

  service_name = "com.amazonaws.${data.aws_region.current.name}.s3"

  vpc_endpoint_type = "Gateway"

  route_table_ids = aws_route_table.private[*].id

  tags = {
    Name = "${var.name}-s3-endpoint"
  }
}
```

We need the region data source:

```hcl
data "aws_region" "current" {}
```

---

# 27. Why S3 Gateway Endpoint?

Instead of:

```text
EKS
 ↓
NAT
 ↓
Internet
 ↓
S3
```

we can have:

```text
EKS
 ↓
VPC
 ↓
S3 Gateway Endpoint
 ↓
S3
```

This can reduce NAT traffic/cost and keeps the path within AWS networking.

---

# 28. ECR Connectivity

EKS nodes pulling images from ECR may need access to:

```text
ECR API
ECR DKR
S3
```

depending on the exact image-pull path and configuration.

For a production private EKS design, evaluate VPC endpoints for the AWS services required by your nodes and workloads.

Don't blindly add every AWS endpoint.

Each endpoint has cost/operational implications, particularly interface endpoints.

---

# 29. Interface Endpoint Pattern

An interface endpoint generally uses:

```text
Private subnet
     │
     ▼
ENI
     │
     ▼
AWS service
```

Example services may include:

```text
sts
secretsmanager
ecr.api
ecr.dkr
logs
```

We'll add the endpoints needed by our EKS architecture once the IAM/EKS requirements are finalized.

---

# 30. VPC Module Data Sources

At the top of `main.tf`:

```hcl
data "aws_region" "current" {}
```

You can then build service names dynamically:

```hcl
service_name = "com.amazonaws.${data.aws_region.current.name}.s3"
```

This avoids hardcoding the AWS region inside the module.

---

# 31. Outputs

Create:

```text
terraform/modules/vpc/outputs.tf
```

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.this.id
}

output "vpc_cidr" {
  description = "VPC CIDR"
  value       = aws_vpc.this.cidr_block
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "availability_zones" {
  description = "Availability zones"
  value       = var.availability_zones
}

output "nat_gateway_ids" {
  description = "NAT gateway IDs"
  value       = aws_nat_gateway.this[*].id
}
```

---

# 32. Development Environment

Go to:

```text
terraform/environments/dev/
```

Create:

```text
provider.tf
```

```hcl
provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = "enterprise-ecommerce"
      Environment = "dev"
      ManagedBy   = "terraform"
    }
  }
}
```

---

# 33. Backend

Create:

```text
backend.tf
```

```hcl
terraform {
  backend "s3" {
    bucket       = "YOUR-TERRAFORM-STATE-BUCKET"
    key          = "dev/vpc/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

This state is now remote.

---

# 34. Dev Variables

Create:

```text
variables.tf
```

```hcl
variable "aws_region" {
  type    = string
  default = "ap-south-1"
}

variable "vpc_cidr" {
  type    = string
  default = "10.0.0.0/16"
}

variable "availability_zones" {
  type = list(string)

  default = [
    "ap-south-1a",
    "ap-south-1b",
    "ap-south-1c"
  ]
}

variable "public_subnet_cidrs" {
  type = list(string)

  default = [
    "10.0.1.0/24",
    "10.0.2.0/24",
    "10.0.3.0/24"
  ]
}

variable "private_subnet_cidrs" {
  type = list(string)

  default = [
    "10.0.11.0/24",
    "10.0.12.0/24",
    "10.0.13.0/24"
  ]
}
```

---

# 35. Development VPC

Create:

```text
main.tf
```

```hcl
module "vpc" {
  source = "../../modules/vpc"

  name = "ecommerce-dev"

  vpc_cidr = var.vpc_cidr

  availability_zones = var.availability_zones

  public_subnet_cidrs = var.public_subnet_cidrs

  private_subnet_cidrs = var.private_subnet_cidrs

  enable_nat_gateway = true

  single_nat_gateway = true

  enable_vpc_endpoints = true
}
```

For a cost-conscious development environment:

```text
single_nat_gateway = true
```

For production later:

```text
single_nat_gateway = false
```

---

# 36. Environment Outputs

Create:

```text
outputs.tf
```

```hcl
output "vpc_id" {
  value = module.vpc.vpc_id
}

output "public_subnet_ids" {
  value = module.vpc.public_subnet_ids
}

output "private_subnet_ids" {
  value = module.vpc.private_subnet_ids
}
```

---

# 37. Initialize Dev Terraform

```bash
cd terraform/environments/dev
```

Then:

```bash
terraform init
```

You should see Terraform configure the S3 backend.

Check:

```bash
terraform validate
```

Then:

```bash
terraform fmt -recursive
```

---

# 38. Terraform Plan

Run:

```bash
terraform plan
```

You should see resources such as:

```text
aws_vpc
aws_internet_gateway
aws_subnet
aws_route_table
aws_route
aws_route_table_association
aws_eip
aws_nat_gateway
aws_vpc_endpoint
```

Review the plan carefully.

---

# 39. Apply

For the development environment:

```bash
terraform apply
```

Review:

```text
Plan: X to add, 0 to change, 0 to destroy.
```

Then:

```text
yes
```

Terraform creates the VPC.

---

# 40. Verify VPC

Get Terraform output:

```bash
terraform output
```

Example:

```text
vpc_id = "vpc-xxxxxxxx"
private_subnet_ids = [
  "subnet-aaaa",
  "subnet-bbbb",
  "subnet-cccc"
]
```

AWS CLI:

```bash
aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=ecommerce-dev"
```

---

# 41. Verify Subnets

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=YOUR_VPC_ID"
```

You should find:

```text
10.0.1.0/24
10.0.2.0/24
10.0.3.0/24

10.0.11.0/24
10.0.12.0/24
10.0.13.0/24
```

---

# 42. Verify Availability Zones

```bash
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=YOUR_VPC_ID" \
  --query 'Subnets[].{CIDR:CidrBlock,AZ:AvailabilityZone,Subnet:SubnetId}'
```

Expected conceptually:

```text
CIDR             AZ
------------------------
10.0.1.0/24      ap-south-1a
10.0.2.0/24      ap-south-1b
10.0.3.0/24      ap-south-1c

10.0.11.0/24     ap-south-1a
10.0.12.0/24     ap-south-1b
10.0.13.0/24     ap-south-1c
```

---

# 43. Verify Route Tables

```bash
aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=YOUR_VPC_ID"
```

Public route tables should contain:

```text
0.0.0.0/0 → Internet Gateway
```

Private route tables should contain:

```text
0.0.0.0/0 → NAT Gateway
```

---

# 44. Understand the Traffic Path

### Internet-facing traffic

Eventually:

```text
Internet
   │
   ▼
ALB
   │
   ▼
Private EKS
```

The ALB will live using public subnet networking, while its targets can be private workloads.

---

### Private outbound traffic

```text
Pod
 │
 ▼
Private subnet
 │
 ▼
Private route table
 │
 ▼
NAT Gateway
 │
 ▼
Public subnet
 │
 ▼
Internet Gateway
 │
 ▼
Internet
```

---

# 45. S3 Endpoint Traffic

For S3 traffic:

```text
Private workload
      │
      ▼
VPC route
      │
      ▼
S3 Gateway Endpoint
      │
      ▼
S3
```

No NAT is needed for that S3 path.

---

# 46. Security Groups

At this stage we have built the network foundation.

We should **not** put every security rule into the VPC module.

Later, security groups will be created according to workload boundaries.

For example:

```text
ALB SG
  ↓
EKS workload SG
  ↓
RDS SG
```

Desired concept:

```text
Internet
   │
   ▼
ALB :443
   │
   ▼
Application :8080
   │
   ▼
RDS :5432
```

But:

```text
Internet
   X
RDS :5432
```

must not be allowed.

---

# 47. EKS Subnet Tags

When we create EKS, we will need tags such as:

```text
kubernetes.io/cluster/<cluster-name>
```

and load-balancer role tags.

For example:

```text
kubernetes.io/role/elb
kubernetes.io/role/internal-elb
```

The exact tagging requirements should be aligned with the AWS Load Balancer Controller and EKS version we deploy.

Don't hardcode cluster-specific tags into a generic VPC module unless the module is explicitly designed to receive the cluster name.

---

# 48. IP Address Planning

EKS uses the AWS VPC CNI.

Therefore Pods can consume VPC IP addresses.

This means:

```text
10.0.0.0/16
```

is not simply:

```text
nodes only
```

It supports the broader EKS networking model.

As the number of Pods grows:

```text
21 services
   ↓
multiple replicas
   ↓
many Pod IPs
   ↓
VPC subnet capacity matters
```

This becomes particularly important when we reach:

```text
HPA
Karpenter
high traffic
large node groups
```

---

# 49. Why `/24` Might Eventually Be Too Small

Each `/24` contains roughly:

```text
256 addresses
```

before AWS subnet reservations and other consumption.

For a learning environment, this can be sufficient.

For a production EKS environment with:

```text
many nodes
+
many Pods
+
AWS VPC CNI
```

you may choose larger private subnets.

For example:

```text
10.0.0.0/18
```

or a larger CIDR allocation strategy.

The correct size depends on:

```text
number of nodes
number of Pods per node
ENI/IP allocation
Pod density
growth
other AWS resources
```

Do not choose subnet size solely because `/24` looks tidy.

---

# 50. Production VPC CIDR Planning

Before production, consider:

```text
Current capacity
+
3–5 year growth
+
future clusters
+
future services
+
network peering
+
Transit Gateway
+
on-prem connectivity
```

Also avoid CIDR overlap with:

```text
other AWS accounts
on-prem networks
VPN networks
partner networks
```

For example:

```text
Ecommerce VPC
10.0.0.0/16

Corporate network
10.0.0.0/8
```

would create potential overlap problems.

CIDR planning needs to happen before large-scale network integration.

---

# 51. NAT Cost Awareness

A production architecture with:

```text
3 NAT Gateways
```

has ongoing AWS charges.

Development:

```text
1 NAT Gateway
```

can reduce cost.

Alternative designs may use:

```text
VPC endpoints
NAT instances
centralized egress
```

depending on enterprise requirements.

We should not optimize production by simply removing NAT without understanding what private workloads need.

---

# 52. Troubleshooting

### No internet from private subnet

Check:

```text
Private subnet
   ↓
Route table
   ↓
NAT Gateway
   ↓
Public subnet
   ↓
IGW
```

Commands:

```bash
aws ec2 describe-route-tables
```

and:

```bash
aws ec2 describe-nat-gateways
```

---

### NAT Gateway unavailable

Check:

```bash
aws ec2 describe-nat-gateways \
  --filter Name=vpc-id,Values=YOUR_VPC_ID
```

Check state:

```text
available
```

---

### Subnet has wrong AZ

```bash
aws ec2 describe-subnets \
  --subnet-ids SUBNET_ID
```

---

### S3 access unexpectedly uses NAT

Check route tables and the S3 gateway endpoint:

```bash
aws ec2 describe-vpc-endpoints \
  --filters Name=vpc-id,Values=YOUR_VPC_ID
```

---

# 53. Terraform Destroy Warning

For development, you may eventually clean everything:

```bash
terraform destroy
```

But **do not casually run this against production**.

Especially once we have:

```text
EKS
RDS
S3
ElastiCache
```

A destroy can have catastrophic consequences.

Before destructive operations:

```text
Plan
 ↓
Review
 ↓
Understand dependencies
 ↓
Backup if necessary
 ↓
Approval
 ↓
Execute
```

---

# 54. Important Terraform Dependency Model

Terraform understands:

```text
VPC
 ↓
Subnet
 ↓
NAT
 ↓
EKS
```

and:

```text
VPC
 ↓
Private subnet
 ↓
RDS
```

The architecture becomes:

```text
                 VPC
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
      EKS        RDS      Redis
        │
        ▼
      Pods
        │
   ┌────┼────┐
   ▼    ▼    ▼
  SQS  S3   Secrets
```

---

# 55. What We Have Built

After this chapter:

```text
                    AWS
                     │
                     ▼
             ecommerce-dev VPC
              10.0.0.0/16
                     │
        ┌────────────┼────────────┐
        │            │            │
        ▼            ▼            ▼
       AZ-A         AZ-B         AZ-C
        │            │            │
     Public A     Public B     Public C
        │            │            │
      NAT A        NAT B        NAT C
        │            │            │
     Private A    Private B    Private C
        │            │            │
        └────────────┼────────────┘
                     │
                     ▼
              Future EKS cluster
```

For development, we deliberately configure:

```text
3 AZs
1 NAT Gateway
```

For production:

```text
3 AZs
3 NAT Gateways
```

unless the final architecture chooses a different controlled egress design.

---

# 56. Current Implementation Progress

```text
[✓] GitHub repositories
[✓] Order Service
[✓] Local Docker
[✓] PostgreSQL
[✓] Terraform repository
[✓] Terraform remote state
[✓] S3 versioning
[✓] S3 encryption
[✓] S3 public access protection
[✓] S3 native state locking
[✓] VPC
[✓] Internet Gateway
[✓] Public subnets
[✓] Private subnets
[✓] Route tables
[✓] NAT Gateway
[✓] S3 VPC endpoint
[ ] IAM
[ ] ECR
[ ] EKS
[ ] EKS Add-ons
[ ] CI
[ ] Helm
[ ] GitOps
[ ] ArgoCD
```

---

# 57. Definition of Done

Verify:

```text
[ ] VPC created
[ ] CIDR = 10.0.0.0/16
[ ] 3 AZs configured
[ ] 3 public subnets
[ ] 3 private subnets
[ ] Internet Gateway created
[ ] NAT Gateway created
[ ] Public route tables configured
[ ] Private route tables configured
[ ] Private subnet has NAT route
[ ] DNS support enabled
[ ] DNS hostnames enabled
[ ] S3 gateway endpoint configured
[ ] Terraform state stored remotely
[ ] terraform validate succeeds
[ ] terraform plan succeeds
[ ] Terraform outputs verified
```

---

# 58. The Architecture We Are Building

We have now crossed the boundary from local development into AWS:

```text
LOCAL
─────────────────────────────

Order Service
     ↓
Docker
     ↓
PostgreSQL


AWS
─────────────────────────────

Terraform
     ↓
VPC
     ↓
IAM
     ↓
ECR
     ↓
EKS
     ↓
RDS
```

The next step is **IAM**, because EKS, GitHub Actions, ECR, workloads, and AWS services all need controlled identities.

## Next: `58-Terraform-IAM.md`

We will implement the real identity chain:

```text
GitHub Actions
      │
      │ OIDC
      ▼
AWS IAM Role
      │
      ▼
ECR

EKS Pod
      │
      │ EKS Pod Identity
      ▼
IAM Role
      │
      ├── SQS
      ├── S3
      └── Secrets Manager
```

and establish **least-privilege trust policies**, rather than putting AWS access keys into GitHub Actions or Kubernetes Secrets.
