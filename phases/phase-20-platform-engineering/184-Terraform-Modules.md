# Chapter 184 — Terraform Modules

Phase 20 — Platform Engineering | Enterprise E-commerce Platform | AWS EKS | 21 Microservices

This chapter builds the reusable Terraform module layer for our enterprise e-commerce platform. The objective is to avoid writing separate infrastructure code for every microservice or environment while maintaining production controls for security, reliability, cost, and change management.

We will connect the infrastructure layer to the reusable Helm templates from Chapter 182 and reusable GitHub Actions from Chapter 183.

## Chapter roadmap

Part 1 — Architecture and module design

Define repository boundaries, module ownership, versioning, naming, environment composition, and remote Terraform state.

Part 2 — AWS infrastructure modules

Build reusable VPC, EKS, ECR, IAM, and supporting infrastructure modules.

Part 3 — Environment composition and CI/CD

Compose DEV, QA, STAGING, and PROD stacks; run validation and plans; control applies through approvals.

Part 4 — Production operations

Test modules, publish versions, detect drift, troubleshoot state, recover from mistakes, and manage infrastructure changes.

# Part 1 — Architecture and module design

## 1. What are we building?

Our application has 21 microservices:

- Auth, User, Customer
- Catalog, Product, Inventory, Pricing, Promotion
- Cart, Order, Payment, Fraud
- Shipping, Delivery, Notification
- Search, Recommendation, Review
- Tax, Invoice, Reporting

These services share infrastructure. For example, they may run in the same EKS cluster, use a common network foundation, and publish images to ECR. Some services may also need separate databases, queues, IAM permissions, or other resources.

We should not copy and paste the same Terraform configuration into 21 repositories.

Instead, we create reusable modules that describe standard infrastructure patterns. Environment-specific root configurations call those modules with approved inputs.

The important separation is:

- Modules define reusable infrastructure patterns.
- Root stacks compose modules into a complete environment.
- CI workflows validate and plan changes.
- Protected deployment controls determine who can apply changes.
- Remote state tracks Terraform-managed resources.

## 2. Production ownership: who does what?

| Responsibility                       | Primary owner                                              | Example                                       |
| ------------------------------------ | ---------------------------------------------------------- | --------------------------------------------- |
| VPC and subnet architecture          | Cloud/platform team                                        | CIDR ranges, AZ layout, routing               |
| EKS cluster and node groups          | Platform team                                              | Cluster version, node scaling, access         |
| ECR repository standards             | Platform team                                              | Image immutability and lifecycle              |
| Application deployment configuration | Application/platform team                                  | Helm values, resource requests                |
| Application-specific IAM access      | Platform and service owner                                 | Permission to read one required secret        |
| Database architecture                | DBA/cloud platform, depending on organization              | Engine, availability, backups                 |
| Infrastructure module changes        | Platform engineering                                       | Module code and release version               |
| Production Terraform apply           | Authorized infrastructure operators or approved automation | Controlled change after review                |
| Security policy                      | Security/platform engineering                              | Encryption, tagging, scanning, IAM boundaries |

The exact ownership model differs between organizations. The principle is that application teams should consume approved infrastructure interfaces rather than independently reinventing network, IAM, and cluster configurations.

## 3. Repository architecture

We will use a dedicated repository for reusable Terraform modules and a separate repository for environment root configurations.

```
company-ecommerce/
├── platform-terraform-modules/
│   ├── modules/
│   │   ├── vpc/
│   │   ├── eks/
│   │   ├── ecr/
│   │   ├── iam-github-oidc/
│   │   ├── iam-eks-workload/
│   │   ├── security-groups/
│   │   └── kms/
│   ├── examples/
│   ├── tests/
│   ├── .github/
│   │   └── workflows/
│   ├── versions.tf
│   └── README.md
│
├── platform-infrastructure/
│   ├── bootstrap/
│   │   └── terraform-state/
│   ├── environments/
│   │   ├── dev/
│   │   ├── qa/
│   │   ├── staging/
│   │   └── prod/
│   ├── policies/
│   ├── scripts/
│   ├── .github/
│   │   └── workflows/
│   └── README.md
│
├── platform-helm/
├── platform-github-actions/
├── platform-gitops/
└── services/
    ├── auth-service/
    ├── order-service/
    ├── payment-service/
    └── ... remaining application repositories
```

### Why two Terraform repositories?

`platform-terraform-modules` contains the reusable building blocks. It should change when a reusable capability changes.

`platform-infrastructure` contains the configurations that instantiate those building blocks for specific environments. It should change when we need to deploy or modify infrastructure in DEV, QA, STAGING, or PROD.

This separation allows us to release a module without automatically applying that release to every environment.

Production rule: publishing a new module version must not itself trigger a production infrastructure change. The environment configuration must explicitly adopt the version, review the resulting plan, and pass the required approval controls.

## 4. Module versus root configuration

This distinction is essential for real-world Terraform work.

A module defines a reusable capability. A root configuration selects and connects modules to create an environment.

For example, the VPC module should not hardcode `prod` or the e-commerce production CIDR. It should accept inputs such as VPC CIDR, availability zones, and subnet definitions.

The DEV root configuration supplies DEV values. The PROD root configuration supplies PROD values.

### Example: same module, different environments

DEV

Illustrative configuration

- Smaller node capacity
- Lower non-production scale
- DEV CIDR ranges

PROD

Illustrative configuration

- Production capacity
- Multi-AZ design
- Separate production CIDR ranges

Both environments can consume the same reviewed module version. Their values and operational controls differ.

Example module call:

```
module "network" {
  source  = "git::https://github.com/company-ecommerce/platform-terraform-modules.git//modules/vpc?ref=v1.0.0"

  name               = "ecommerce-dev"
  vpc_cidr           = var.vpc_cidr
  availability_zones = var.availability_zones

  public_subnet_cidrs  = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs

  environment = var.environment
  tags        = var.tags
}
```

The `ref=v1.0.0` pins the module source to a version tag. In a production organization, protect release tags and prefer immutable, reviewed releases. A Git tag alone is not a complete security boundary if unauthorized users can move tags.

Important: module source addresses, inputs, and outputs must match the actual module implementation. The call above becomes valid only after we implement those interfaces.

## 5. Define the infrastructure boundaries

For our project, we will group infrastructure by lifecycle and ownership.

| Infrastructure stack | Resources managed                                                                      |
| -------------------- | -------------------------------------------------------------------------------------- |
| State bootstrap      | Terraform state S3 bucket and related access controls                                  |
| Network foundation   | VPC, subnets, route tables, gateways, endpoints                                        |
| EKS platform         | EKS cluster, node groups, cluster access, required add-ons                             |
| Container registry   | ECR repositories and lifecycle policies                                                |
| Identity             | GitHub Actions OIDC roles and workload IAM roles                                       |
| Data infrastructure  | Database resources and supporting access, coordinated with the database implementation |
| Shared platform      | KMS keys, security groups, common policies where appropriate                           |

Do not put every resource into one enormous state file simply because it belongs to the same business application.

Likewise, do not create a separate Terraform state for every tiny resource without a reason. Every state boundary introduces its own permissions, dependencies, locking, plan, apply, and recovery responsibilities.

A reasonable starting point is to separate resources where lifecycle, ownership, risk, or access requirements materially differ.

### Example dependency flow

```
State Bootstrap
      |
      v
Network Foundation
      |
      +------------------+
      |                  |
      v                  v
EKS Platform         Data Infrastructure
      |
      v
ECR / IAM integration
      |
      v
Helm and GitOps deployment
```

This diagram illustrates logical dependencies, not a universal requirement to apply the stacks in exactly this order. ECR repositories can often be provisioned independently of EKS, for example.

## 6. Establish a Terraform version policy

Use a tested Terraform version range and commit the dependency lock file.

Root configuration example:

```
terraform {
  required_version = ">= 1.10.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}
```

These version constraints are an illustrative starting point, not a claim that every module or organization should use these exact versions. Before adopting them, verify that the selected Terraform binary, AWS provider, module versions, and all referenced resources are compatible.

For each environment:

1. Pin the approved Terraform CLI version in CI.
2. Commit `.terraform.lock.hcl` for root configurations.
3. Review provider upgrades in pull requests.
4. Run module and environment tests before promoting upgrades.
5. Avoid unreviewed provider upgrades during production applies.

The provider lock file records selected provider versions and checksums. It does not pin remote module source code in the same way, so module versions must be controlled separately.

## 7. Standard naming and tagging

A production platform needs consistent names and tags so that teams can find resources, attribute costs, identify ownership, and investigate incidents.

Example resource name:

```
ecommerce-prod-ap-south-1-eks
```

Example tags:

```
locals {
  common_tags = {
    Application = "ecommerce"
    ManagedBy   = "terraform"
    Environment = var.environment
    Owner       = "platform-engineering"
    CostCenter  = "engineering"
  }
}
```

Pass these tags to resources and modules that support them. Avoid assuming that every AWS resource supports tags in the same way.

Use explicit variables for environment-specific values. Avoid deriving critical production names from a developer's local shell environment, an unvalidated branch name, or a default that could accidentally point at PROD.

## 8. Input validation is part of the module contract

A reusable module should reject invalid inputs early rather than allow an unsafe configuration to reach AWS.

For example, a VPC module should validate the environment name and CIDR input.

```
variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition = contains(
      ["dev", "qa", "staging", "prod"],
      var.environment
    )

    error_message = "Environment must be dev, qa, staging, or prod."
  }
}

variable "vpc_cidr" {
  description = "IPv4 CIDR for the VPC"
  type        = string

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "vpc_cidr must be a valid IPv4 CIDR block."
  }
}
```

CIDR syntax validation is not sufficient to guarantee that a network design is safe. The platform team must also check that ranges do not overlap with connected VPCs, on-premises networks, VPNs, or other routed environments.

Other useful validations include:

- Availability-zone and subnet list lengths.
- Required tags and naming rules.
- Supported instance families.
- Valid cluster version values.
- Minimum and maximum scaling limits.
- Whether public IP assignment is allowed for a given subnet class.

## 9. Remote Terraform state: why it matters

Terraform state records the relationship between Terraform resource addresses and real infrastructure objects. It can contain sensitive values, resource identifiers, and other information that should not be exposed.

For a team environment, local state files are not an appropriate shared source of truth.

We will use an S3 backend for remote state, with access controls, encryption, versioning, and controlled access. The state bucket is bootstrapped separately because Terraform cannot depend on a backend resource that has not yet been created.

### Example state key layout

```
s3://<terraform-state-bucket>/
    ecommerce/
        dev/
            network/terraform.tfstate
            eks/terraform.tfstate
            registry/terraform.tfstate
        qa/
            network/terraform.tfstate
            eks/terraform.tfstate
            registry/terraform.tfstate
        staging/
            network/terraform.tfstate
            eks/terraform.tfstate
            registry/terraform.tfstate
        prod/
            network/terraform.tfstate
            eks/terraform.tfstate
            registry/terraform.tfstate
```

These are illustrative logical paths. In practice, each root configuration's `key` determines its state object path. S3 does not need actual directories; the apparent folders are prefixes.

Never commit a state file, plan file containing sensitive values, AWS credentials, or state-backup files to Git. Access to the state bucket should be limited to the identities that need it.

## 10. Configure the S3 backend

Create a backend configuration in the root stack, for example `environments/dev/network/backend.tf`.

```
terraform {
  backend "s3" {
    bucket       = "REPLACE_WITH_APPROVED_STATE_BUCKET"
    key          = "ecommerce/dev/network/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

The bucket name is a placeholder. Replace it with the approved bucket name for your AWS account; do not create a shared bucket with a guessed or copied name.

### What does `use_lockfile = true` do?

For Terraform versions that support S3 native lock files, `use_lockfile = true` enables locking through an S3 lock object. This helps prevent concurrent Terraform operations from modifying the same state at the same time.

It does not mean all Terraform state operations across all states are globally serialized. Separate state keys can be operated on independently.

An older setup may use a DynamoDB table for state locking. Do not configure both approaches automatically just because an older tutorial uses DynamoDB. Confirm the locking mechanism supported by your selected Terraform version and the organization's existing backend before migrating.

### Why do we encrypt state?

State can reveal resource configuration and sensitive outputs. Encryption at rest is one protection, but it does not replace authorization controls.

The platform team should define:

- S3 bucket-level access restrictions.
- Encryption requirements and KMS permissions, where customer-managed keys are used.
- Versioning and recovery expectations.
- Public-access blocking.
- Transport security.
- Audit logging and access monitoring.
- Retention rules consistent with recovery and compliance requirements.

### Why is the backend bucket created separately?

The backend is initialized before Terraform can read or write that root configuration's state. If the same state depends on creating its own backend bucket, the bootstrap dependency becomes circular.

A practical setup is:

1. A restricted bootstrap process creates the state bucket and any required KMS key or access configuration.
2. The platform team verifies encryption, versioning, public-access blocking, and permissions.
3. The network root configuration is initialized against its dedicated state key.
4. Subsequent infrastructure changes use the remote backend.
5. The bootstrap stack is operated with separate access and documented recovery procedures.

## 11. Bootstrap repository and files

Example bootstrap directory:

```
platform-infrastructure/
└── bootstrap/
    └── terraform-state/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        ├── versions.tf
        └── README.md
```

A bootstrap configuration can create the bucket and related security controls. The exact settings must align with your organization's retention, encryption, logging, and recovery requirements.

For example, the bucket resource can enforce versioning and block public access:

```
resource "aws_s3_bucket" "terraform_state" {
  bucket = var.state_bucket_name

  tags = {
    Name      = var.state_bucket_name
    ManagedBy = "terraform"
    Purpose   = "terraform-state"
  }

  lifecycle {
    prevent_destroy = true
  }
}

resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  versioning_configuration {
    status = "Enabled"
  }
}

resource "aws_s3_bucket_public_access_block" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id

  block_public_acls       = true
  ignore_public_acls      = true
  block_public_policy     = true
  restrict_public_buckets = true
}
```

This is only part of the complete configuration. A production implementation should also define encryption, the bucket policy, and any required logging controls. If using a customer-managed KMS key, configure key policy and IAM permissions together; enabling encryption without ensuring the right principals can use the key can make state unavailable.

`prevent_destroy` is a useful safeguard, but it is not a substitute for restrictive IAM permissions, protected plans, backups, and recovery procedures. It can also complicate intentional replacement or decommissioning, so document the controlled removal process.

## 12. Secure the state bucket access

A typical access model separates these identities:

- Bootstrap operator: creates or updates the backend and its security configuration.
- Terraform plan role: reads state and inspects AWS resources needed to calculate a plan.
- Terraform apply role: has the approved permissions required to make the planned infrastructure changes.
- Read-only auditor: can inspect permitted configuration or audit evidence without applying changes.

Do not automatically grant the same broad permissions to every developer, CI workflow, and auditor.

The plan role may still need read access to many AWS resources. The apply role needs a carefully scoped set of permissions for the resources managed by the selected stack. Both roles also need the appropriate backend access.

For GitHub Actions, prefer short-lived AWS credentials obtained through OpenID Connect (OIDC), with trust conditions restricting which repository and workflow context can assume each role. Avoid long-lived AWS access keys stored as repository secrets.

## 13. Separate state files by environment and lifecycle

A simple initial layout could be:

```
environments/
├── dev/
│   ├── network/
│   ├── eks/
│   └── registry/
├── qa/
│   ├── network/
│   ├── eks/
│   └── registry/
├── staging/
│   ├── network/
│   ├── eks/
│   └── registry/
└── prod/
    ├── network/
    ├── eks/
    └── registry/
```

Each directory is a root configuration with its own backend key and explicit module versions.

This gives us several benefits:

1. A DEV plan cannot accidentally reuse the PROD state key.
2. Different environments can have different IAM permissions and approval requirements.
3. A network change can be reviewed separately from a routine application release.
4. Recovery can focus on the affected state rather than a single enormous state file.

It also introduces a trade-off: modules with cross-stack dependencies require a clear interface. Avoid creating hidden dependencies through hardcoded resource names or manual console changes.

When one stack needs a value from another stack, choose an explicit method—such as a controlled remote-state output, a published configuration value, or an existing infrastructure inventory. Remote-state access can expose more information than a consumer needs, so prefer narrowly scoped outputs and permissions.

## 14. Use environment-specific values without copying modules

Example DEV variables:

```
environment = "dev"
aws_region  = "ap-south-1"

vpc_cidr = "10.40.0.0/16"

availability_zones = [
  "ap-south-1a",
  "ap-south-1b",
  "ap-south-1c"
]

tags = {
  Application = "ecommerce"
  Environment = "dev"
  ManagedBy   = "terraform"
  Owner       = "platform-engineering"
}
```

The CIDR and AZ choices are illustrative. Verify that the selected AZs are available to your account and that the CIDR does not overlap with other networks.

A production values file should be deliberately reviewed rather than copied blindly from DEV. The same module can support different capacity, routing, resilience, and access requirements without embedding those decisions in the reusable module itself.

## 15. Avoid unsafe Terraform patterns

Common patterns that make infrastructure harder to operate include:

- Using `latest` or other floating versions for production module sources.
- Committing state or sensitive plan artifacts to Git.
- Running `terraform apply -auto-approve` against production from every pull request.
- Allowing application repositories to assume unrestricted infrastructure roles.
- Using one state file for unrelated lifecycles without a reason.
- Duplicating the same VPC or IAM logic across every environment.
- Hardcoding credentials or account IDs into module code.
- Making manual AWS console changes without reconciling them with Terraform.
- Treating a successful `terraform validate` as proof that a production plan is safe.

The goal is not to make every operation complicated. It is to make routine infrastructure changes repeatable and high-risk changes deliberate, reviewable, and recoverable.

# Part 2 — AWS infrastructure modules

## 16. Build the VPC module

Our first reusable infrastructure module will define the network foundation for the EKS platform.

The module should accept network design inputs and create the corresponding AWS resources. It should not contain application-specific settings such as the Order Service's container port or the Payment Service's database password.

### Network architecture

[AWS NAT Gateway and Private/Public Subnets](https://images.openai.com/static-rsc-4/pOYWv2BRCB7XQ94CMxfcjaHcEoxTyaXrWIJAnF5g8R8eRSHtjTNCBw2odBP445rNJspLEftNszJDf_4YO4ErorzyPXC0lbJR8zWOzPy-prx4fSmB_iqk3arXRvZnAs1EvUk1HbO7BpIk8b-Gj5J4KuCXYXTD8wxBgzOIzcDCzKU?purpose=inline)

[AWS EKS - Satori Documentation](https://images.openai.com/static-rsc-4/H0Kj_VhR6a0lZCnQLVVdp59dZLpQMNlLEurIYZtYQwOtXS4_vhh3vInVD5hDmH4GCvk_fjrg7JGOaG_cVxRPKR_AvN9fS3zzHhzagkOZsck4YyV_Oqlf0fFYYyh1XD2nHf9WuaCVAY5mrzSyt3-6Rhjr8vMLKEkJEPHx0Yefrx8?purpose=inline)

[MULTI-TIER ARCHITECTURE ON AWS-EKS | by Kamalesh kumar | Medium](https://images.openai.com/static-rsc-4/yAYE1lL98iyVt2C2PWH6vKghoM-kYhXUy2WDhfGrnYgQal2Y7p_7GDhXHVkN2oGlrc8nR0K6L_MkinWhcv1OXyTvqzkAZkXPS2aTweL3sIaBOlzkwI6yYOyqMgiE1gr6PTFkmx7mun2tue8tI8RTVgP492TFg9DVy7WlvBhIid8?purpose=inline)

9

A typical e-commerce network includes:

- A VPC with a planned IPv4 address range.
- Public subnets for resources that genuinely need public routing, such as public load balancers or NAT gateways depending on the design.
- Private subnets for EKS worker nodes and internal workloads.
- Route tables and gateways.
- Security groups and network ACLs where appropriate.
- Optional VPC endpoints for AWS services to reduce public egress dependency.
- Adequate IP capacity for nodes, pods, load balancers, and future growth.

Important: a private subnet does not automatically make every workload private or secure. Routing, security groups, Kubernetes networking, IAM, and load-balancer exposure all affect the effective security boundary.

### Why use multiple Availability Zones?

A multi-AZ design helps the platform tolerate certain zonal failures and allows workloads to be distributed. It does not by itself guarantee application availability: replicas, scheduling constraints, load-balancer configuration, database design, and capacity must also support the intended resilience.

For production, plan enough address space for current and expected EKS node and pod networking requirements. An undersized subnet can prevent scaling or replacing nodes during an incident.

## 17. VPC module directory structure

```
platform-terraform-modules/
└── modules/
    └── vpc/
        ├── main.tf
        ├── variables.tf
        ├── outputs.tf
        ├── versions.tf
        ├── locals.tf
        ├── README.md
        └── examples/
            └── basic/
                ├── main.tf
                └── terraform.tfvars.example
```

Responsibilities of each file:

| File              | Purpose                                                   |
| ----------------- | --------------------------------------------------------- |
| `main.tf`         | VPC, subnet, route, and gateway resources                 |
| `variables.tf`    | Module input definitions and validation                   |
| `outputs.tf`      | Values consumed by other stacks                           |
| `versions.tf`     | Terraform and provider constraints                        |
| `locals.tf`       | Derived names, tags, and local calculations               |
| `README.md`       | Usage, assumptions, limitations, and examples             |
| `examples/basic/` | A minimal configuration showing how to consume the module |

Keep the module small enough to understand and test. If routing, endpoints, or subnet behavior become complicated, separate those concerns into clear submodules or optional capabilities rather than adding dozens of unrelated switches to one file.

## 18. Define the VPC module inputs

`modules/vpc/variables.tf`:

```
variable "name" {
  description = "Name prefix for network resources"
  type        = string

  validation {
    condition     = length(trimspace(var.name)) > 0
    error_message = "name must not be empty."
  }
}

variable "vpc_cidr" {
  description = "IPv4 CIDR range for the VPC"
  type        = string

  validation {
    condition     = can(cidrnetmask(var.vpc_cidr))
    error_message = "vpc_cidr must be a valid IPv4 CIDR."
  }
}

variable "availability_zones" {
  description = "Availability zones used by the network"
  type        = list(string)

  validation {
    condition     = length(var.availability_zones) >= 2
    error_message = "At least two availability zones are required."
  }

  validation {
    condition = (
      length(var.availability_zones) ==
      length(distinct(var.availability_zones))
    )
    error_message = "availability_zones must not contain duplicates."
  }
}

variable "public_subnet_cidrs" {
  description = "One public subnet CIDR per configured AZ"
  type        = list(string)

  validation {
    condition = (
      length(var.public_subnet_cidrs) ==
      length(var.availability_zones)
    )
    error_message = "Provide one public subnet CIDR per AZ."
  }
}

variable "private_subnet_cidrs" {
  description = "One private subnet CIDR per configured AZ"
  type        = list(string)

  validation {
    condition = (
      length(var.private_subnet_cidrs) ==
      length(var.availability_zones)
    )
    error_message = "Provide one private subnet CIDR per AZ."
  }
}

variable "environment" {
  description = "Environment name"
  type        = string

  validation {
    condition = contains(
      ["dev", "qa", "staging", "prod"],
      var.environment
    )
    error_message = "environment must be dev, qa, staging, or prod."
  }
}

variable "tags" {
  description = "Additional resource tags"
  type        = map(string)
  default     = {}
}
```

These validations check basic input consistency. They do not prove that subnet CIDRs are inside the VPC CIDR, non-overlapping, or compatible with existing connected networks. Add those checks to module tests and the environment design review.

Also remember that availability-zone names can map to different physical zones across AWS accounts. For multi-account architectures, confirm the intended AZ mapping instead of assuming that `ap-south-1a` identifies the same physical zone everywhere.

## 19. Define common tags

`modules/vpc/locals.tf`:

```
locals {
  common_tags = merge(
    var.tags,
    {
      Name        = var.name
      Environment = var.environment
      ManagedBy   = "terraform"
      Component   = "network"
    }
  )
}
```

Here, `merge` combines user-supplied tags with the module's standard tags. Because later maps override earlier maps, the standard tags in this example take precedence over duplicate keys supplied through `var.tags`.

That is intentional for protected attributes such as `ManagedBy` and `Component`. Document this behavior so callers understand which tags they may customize.

## 20. Implement the VPC and subnets

The following is a teaching implementation of the network foundation. It deliberately shows the essential VPC and subnet resources rather than pretending to be a complete production network module.

`modules/vpc/main.tf`:

```
resource "aws_vpc" "this" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-vpc"
    }
  )
}

resource "aws_subnet" "public" {
  for_each = {
    for index, cidr in var.public_subnet_cidrs :
    tostring(index) => cidr
  }

  vpc_id                  = aws_vpc.this.id
  cidr_block              = each.value
  availability_zone       = var.availability_zones[tonumber(each.key)]
  map_public_ip_on_launch = false

  tags = merge(
    local.common_tags,
    {
      Name                     = "${var.name}-public-${each.key}"
      "kubernetes.io/role/elb" = "1"
    }
  )
}

resource "aws_subnet" "private" {
  for_each = {
    for index, cidr in var.private_subnet_cidrs :
    tostring(index) => cidr
  }

  vpc_id                  = aws_vpc.this.id
  cidr_block              = each.value
  availability_zone       = var.availability_zones[tonumber(each.key)]
  map_public_ip_on_launch = false

  tags = merge(
    local.common_tags,
    {
      Name                              = "${var.name}-private-${each.key}"
      "kubernetes.io/role/internal-elb" = "1"
    }
  )
}
```

### What does `for_each` do here?

Instead of writing three public subnet resources and three private subnet resources manually, the module iterates through the supplied CIDR lists.

For example, with three public subnet CIDRs, Terraform creates three subnet instances.

The example uses stringified list indices as keys. For a long-lived production module, consider using stable, meaningful keys such as `az1`, `az2`, and `az3` or AZ identifiers. Terraform resource addresses can change when the key scheme changes, potentially causing replacement unless state addresses are migrated carefully.

The Kubernetes subnet tags help AWS load-balancer integrations identify eligible subnets, but tags alone do not configure routes or make a subnet public. The route tables and associated gateways must be implemented separately.

## 21. Add the internet gateway and routing

The public subnet name is not what makes it public. A public subnet normally has a route to an internet gateway, and the resources using that route must have suitable network and security configuration.

Add the internet gateway and public route table:

```
resource "aws_internet_gateway" "this" {
  vpc_id = aws_vpc.this.id

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-igw"
    }
  )
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.this.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.this.id
  }

  tags = merge(
    local.common_tags,
    {
      Name = "${var.name}-public-rt"
    }
  )
}

resource "aws_route_table_association" "public" {
  for_each = aws_subnet.public

  subnet_id      = each.value.id
  route_table_id = aws_route_table.public.id
}
```

This gives the public subnets an IPv4 default route through the internet gateway.

It does not create NAT gateways, private-subnet routes, IPv6 routing, or VPC endpoints. Those should be designed explicitly.

## 22. Design private subnet egress

Private EKS nodes may need outbound access to container registries, operating-system repositories, external APIs, or AWS services. There are several approaches.

| Option                  | Benefits                                                   | Trade-offs                                                                    |
| ----------------------- | ---------------------------------------------------------- | ----------------------------------------------------------------------------- |
| NAT gateway per AZ      | Better AZ-level independence and simpler routing isolation | Higher hourly and data-processing costs                                       |
| Shared NAT gateway      | Lower baseline cost                                        | Cross-AZ traffic, dependency concentration, possible AZ-related egress impact |
| VPC endpoints           | Private connectivity to supported AWS services             | Additional resources, policies, and service-specific design                   |
| Restricted egress proxy | Centralized outbound control and inspection                | Additional operational complexity                                             |

For a production EKS platform, decide which destinations require internet access and which can be reached privately. Do not create unrestricted outbound paths merely because a deployment initially fails to pull an image.

A typical architecture may use private subnets for worker nodes, NAT for necessary external destinations, and endpoints for services such as S3 and ECR where appropriate. ECR image pulls can involve both ECR API/registry endpoints and S3 access to image layers; endpoint policies, DNS, and security-group rules must all be correct.

### NAT design example

For a three-AZ production network, a resilient pattern is often:

```
AZ-a private subnets -> NAT gateway in AZ-a
AZ-b private subnets -> NAT gateway in AZ-b
AZ-c private subnets -> NAT gateway in AZ-c
```

A cost-sensitive non-production environment may use fewer NAT gateways if the team accepts the availability and cross-AZ routing trade-offs.

These are design options, not a universal requirement. Measure cost, expected traffic, availability objectives, and the impact of a zonal outage before selecting the pattern.

## 23. Define VPC outputs

`modules/vpc/outputs.tf`:

```
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.this.id
}

output "public_subnet_ids" {
  description = "Public subnet IDs"
  value       = [for subnet in aws_subnet.public : subnet.id]
}

output "private_subnet_ids" {
  description = "Private subnet IDs"
  value       = [for subnet in aws_subnet.private : subnet.id]
}

output "availability_zones" {
  description = "Configured availability zones"
  value       = var.availability_zones
}
```

The EKS stack can consume the private subnet IDs when configuring the cluster and node groups. Other stacks can consume the VPC ID for supported network integrations.

Outputs should be intentional. Avoid exposing secrets or dumping large collections of internal details when a consumer needs only a few identifiers.

## 24. Compose the network root configuration

Now consume the reusable module from `environments/dev/network/main.tf`.

```
terraform {
  required_version = ">= 1.10.0, < 2.0.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = var.tags
  }
}

module "network" {
  source = "git::https://github.com/company-ecommerce/platform-terraform-modules.git//modules/vpc?ref=v1.0.0"

  name               = "ecommerce-${var.environment}"
  vpc_cidr           = var.vpc_cidr
  availability_zones = var.availability_zones

  public_subnet_cidrs  = var.public_subnet_cidrs
  private_subnet_cidrs = var.private_subnet_cidrs

  environment = var.environment
  tags        = var.tags
}
```

The `default_tags` provider configuration and module-level tags overlap deliberately in this example. Provider default tags apply to supported resources, while the module also defines specific resource tags such as `Name`. In a production module, establish a consistent policy so that duplicate tag values do not cause confusion.

The `provider` configuration should use a controlled execution identity. It should not contain access keys or secret credentials.

## 25. Add DEV variable definitions

`environments/dev/network/variables.tf`:

```
variable "aws_region" {
  type    = string
  default = "ap-south-1"
}

variable "environment" {
  type = string

  validation {
    condition     = var.environment == "dev"
    error_message = "This root configuration is for DEV only."
  }
}

variable "vpc_cidr" {
  type = string
}

variable "availability_zones" {
  type = list(string)
}

variable "public_subnet_cidrs" {
  type = list(string)
}

variable "private_subnet_cidrs" {
  type = list(string)
}

variable "tags" {
  type = map(string)
}
```

Example `terraform.tfvars`:

```
environment = "dev"

vpc_cidr = "10.40.0.0/16"

availability_zones = [
  "ap-south-1a",
  "ap-south-1b",
  "ap-south-1c"
]

public_subnet_cidrs = [
  "10.40.0.0/20",
  "10.40.16.0/20",
  "10.40.32.0/20"
]

private_subnet_cidrs = [
  "10.40.64.0/20",
  "10.40.80.0/20",
  "10.40.96.0/20"
]

tags = {
  Application = "ecommerce"
  Environment = "dev"
  Owner       = "platform-engineering"
  CostCenter  = "engineering"
}
```

These CIDR ranges are examples only. They need to be checked for overlap and sized against expected EKS networking requirements before use.

The corresponding root configuration should also include `outputs.tf` to expose the module outputs to the root stack's consumers or tooling.

## 26. Run the first Terraform checks

From the DEV network root directory, the local validation sequence is:

```
terraform fmt -recursive
terraform init
terraform validate
terraform plan -out=tfplan
```

Before running `plan`, verify the backend configuration, AWS identity, region, module source, and environment inputs.

Inspect the plan before applying. Do not treat the command sequence as permission to apply infrastructure automatically.

For CI, keep the generated plan associated with the exact reviewed commit and execution context. Plan files may contain sensitive data; store them only in a controlled location with limited access and appropriate retention. A production workflow must also ensure that the plan is applied under the expected state and authorization conditions.

## 27. VPC production acceptance criteria

Before the network module is considered ready, verify the following.

CIDR ranges are valid, non-overlapping, and documented.

Subnet capacity is sufficient for EKS nodes, pods, and load balancers.

Subnets are distributed across the intended Availability Zones.

Public and private route tables match the approved network design.

Private workloads do not receive public IP addresses by default.

NAT gateways or VPC endpoints meet the approved egress requirements.

Security groups and network ACLs follow the network security design.

Kubernetes subnet tags match the chosen AWS load-balancer integration.

State is remote, encrypted, versioned, and access-restricted.

The module has a tested example, documented inputs, and documented outputs.

Passing these checks does not replace a security review or a real plan review. It establishes that the module's core network behavior has been considered and can be tested.

# Part 3 — EKS and supporting infrastructure modules

## 28. Design the EKS module boundary

The EKS module provisions the Kubernetes control plane and its required AWS-side configuration. Depending on the organization's module boundaries, managed node groups and cluster add-ons may be included in the same module or managed through separate modules.

For our platform, the EKS stack needs to coordinate:

- The VPC and private subnets from Chapter 184's network module.
- The cluster version and supported upgrade path.
- EKS control-plane logging configuration where required by policy.
- Cluster access and authentication.
- Managed node groups and scaling bounds.
- Cluster add-ons and their version compatibility.
- IAM roles and permissions.
- Security groups and endpoint access.
- Kubernetes and AWS networking compatibility.
- Outputs needed by GitOps and deployment automation.

The EKS module must not deploy all 21 microservices. Application releases belong to the Helm and GitOps layer.

### Architecture

This is a logical dependency diagram. Actual node placement, endpoint reachability, and GitOps access must be designed and tested for the chosen cluster architecture.

## 29. Create the EKS module structure

```
modules/
└── eks/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── versions.tf
    ├── locals.tf
    ├── cluster.tf
    ├── node-groups.tf
    ├── addons.tf
    ├── access.tf
    ├── README.md
    └── examples/
        └── basic/
```

These file names are organizational conventions, not special Terraform filenames. Terraform loads `.tf` files in the same directory together.

The split is intended to make the module easier to maintain. If a file becomes too large, split it by resource responsibility while keeping the module's public interface coherent.

## 30. Define EKS module inputs

`modules/eks/variables.tf`:

```
variable "cluster_name" {
  description = "Name of the EKS cluster"
  type        = string
}

variable "cluster_version" {
  description = "Approved Kubernetes version for EKS"
  type        = string
}

variable "vpc_id" {
  description = "VPC ID for the cluster"
  type        = string
}

variable "private_subnet_ids" {
  description = "Private subnet IDs for EKS resources"
  type        = list(string)

  validation {
    condition     = length(var.private_subnet_ids) >= 2
    error_message = "At least two private subnet IDs are required."
  }
}

variable "endpoint_private_access" {
  description = "Whether the cluster API endpoint has private access"
  type        = bool
  default     = true
}

variable "endpoint_public_access" {
  description = "Whether the cluster API endpoint has public access"
  type        = bool
  default     = false
}

variable "tags" {
  description = "Tags applied to supported resources"
  type        = map(string)
  default     = {}
}
```

The endpoint defaults represent a security-oriented starting point, not a complete connectivity design. A private-only cluster endpoint requires administrators and deployment systems to have network access to the private API endpoint—for example, through a controlled VPC runner, VPN, Direct Connect, or other approved private connectivity.

If the endpoint is inaccessible from the environment where GitHub Actions runs, deployment workflows will fail to reach the Kubernetes API. Do not solve this by casually exposing the endpoint publicly; first determine the required connectivity model.

## 31. Create the EKS control plane

A simplified `modules/eks/cluster.tf`:

```
resource "aws_eks_cluster" "this" {
  name     = var.cluster_name
  role_arn = aws_iam_role.cluster.arn
  version  = var.cluster_version

  vpc_config {
    subnet_ids              = var.private_subnet_ids
    endpoint_private_access = var.endpoint_private_access
    endpoint_public_access  = var.endpoint_public_access
  }

  tags = merge(
    var.tags,
    {
      Name      = var.cluster_name
      ManagedBy = "terraform"
    }
  )

  depends_on = [
    aws_iam_role_policy_attachment.cluster_policy
  ]
}
```

This snippet assumes that the module defines `aws_iam_role.cluster` and `aws_iam_role_policy_attachment.cluster_policy` elsewhere. Those resources must exist before the module can validate and plan successfully.

For a complete production module, explicitly design the cluster IAM trust policy, the supported control-plane permissions, encryption requirements where applicable, and any logging or access controls required by organizational policy.

Do not attach arbitrary administrator policies to the EKS cluster role. The cluster role is not the same as a human administrator role or an application's workload role.

## 32. Understand the three different identities

One of the most common EKS architecture mistakes is treating every IAM role as interchangeable.

Cluster IAM role

Used by the EKS control plane for the AWS operations it is authorized to perform.

Node IAM role

Used by worker nodes for necessary node-level AWS operations. Avoid granting application permissions to all nodes when workload-level identity can provide narrower access.

Workload IAM role

Used by an individual Kubernetes workload to access specific AWS resources, through the approved EKS workload identity mechanism.

For example, the Order Service might need permission to read a particular secret. That permission should normally be granted to its dedicated workload identity rather than every EC2 node in the cluster.

The exact implementation may use EKS Pod Identity or IAM Roles for Service Accounts (IRSA). Choose and document the approved mechanism, then standardize it through the platform templates. Do not configure both approaches arbitrarily for the same workload.

## 33. Node groups: capacity and isolation

Managed node groups provide EC2 worker capacity for Kubernetes workloads. Their configuration affects availability, performance, and cost.

A production platform should define:

- Approved instance families and architectures.
- Minimum, desired, and maximum capacity.
- Subnet and AZ placement.
- Node update and replacement strategy.
- Labels and taints for workload isolation.
- Capacity planning and scaling behavior.
- The relationship between cluster autoscaling components and node groups.
- IAM permissions for node bootstrap and image pulls.

Example input contract:

```
variable "node_groups" {
  description = "Configuration for EKS managed node groups"

  type = map(object({
    instance_types = list(string)
    min_size       = number
    desired_size   = number
    max_size       = number
    capacity_type  = string
  }))

  validation {
    condition = alltrue([
      for group in values(var.node_groups) :
      group.min_size <= group.desired_size &&
      group.desired_size <= group.max_size
    ])

    error_message = "Each node group must satisfy min_size <= desired_size <= max_size."
  }
}
```

This validation checks the scaling bounds. It does not determine whether the requested capacity is affordable or sufficient for the application's workload.

A useful starting separation is to use dedicated capacity for workloads with different operational requirements, such as critical production services and non-critical batch jobs. Whether that requires separate node groups, separate clusters, or both depends on isolation, security, and cost objectives.

## 34. Configure cluster access deliberately

Creating an EKS cluster does not automatically mean every developer, CI workflow, or Argo CD installation has the right access.

Define who can:

- Discover cluster metadata.
- Authenticate to the Kubernetes API.
- Read resources in a namespace.
- Deploy application workloads.
- Change cluster-wide resources.
- Modify security-sensitive objects such as service accounts and network policies.

Use the EKS access configuration supported by the chosen cluster setup and maintain Kubernetes RBAC boundaries.

For CI/CD, separate infrastructure provisioning access from application deployment access:

- The Terraform role provisions AWS infrastructure.
- The application release workflow builds and publishes artifacts.
- The GitOps controller reconciles approved Kubernetes configuration.
- Human operators use approved administrative access for troubleshooting and emergency actions.

Avoid using one all-powerful IAM role for all four activities.

## 35. Plan the cluster add-ons

An EKS platform typically requires a deliberate add-on strategy. Depending on the design, this may include:

- Amazon VPC CNI.
- CoreDNS.
- `kube-proxy`.
- EBS CSI driver when EBS-backed persistent volumes are required.
- AWS Load Balancer Controller for supported load-balancer integration.
- Metrics and observability agents required by the platform.
- Cluster autoscaling components, if used.

Some add-ons are managed through EKS add-on resources; others are installed and upgraded through Helm or GitOps. Record the ownership of each add-on so that Terraform and Argo CD do not compete to manage the same resource.

### CoreDNS example

CoreDNS provides Kubernetes service-name resolution. If a pod cannot resolve an internal service name, troubleshoot the pod's DNS configuration, CoreDNS health, service records, and network policies before assuming the AWS VPC DNS configuration is the only cause.

CoreDNS is a cluster-level component, so its resource requirements, replicas, scheduling, and upgrade compatibility must be considered during node-group changes.

## 36. Define EKS outputs

`modules/eks/outputs.tf`:

```
output "cluster_name" {
  description = "EKS cluster name"
  value       = aws_eks_cluster.this.name
}

output "cluster_arn" {
  description = "EKS cluster ARN"
  value       = aws_eks_cluster.this.arn
}

output "cluster_endpoint" {
  description = "Kubernetes API endpoint"
  value       = aws_eks_cluster.this.endpoint
}

output "cluster_certificate_authority_data" {
  description = "Base64-encoded cluster certificate authority data"
  value       = aws_eks_cluster.this.certificate_authority[0].data
}
```

Treat the endpoint and certificate authority data as configuration outputs, not credentials. They still should be shared only where needed.

Do not output access tokens, static credentials, or application secrets. A CI workflow should obtain short-lived authentication through the approved AWS and Kubernetes access model.

## 37. Create the ECR module

Amazon Elastic Container Registry (ECR) stores the container images used by our 21 microservices.

Our image flow is:

```
Application Repository
        |
        v
Reusable GitHub Actions Workflow
        |
        v
Build and Test
        |
        v
Security Scanning
        |
        v
Publish Image to ECR
        |
        v
Capture Immutable Image Digest
        |
        v
Update GitOps Configuration
        |
        v
Argo CD Deploys the Approved Digest
```

The ECR module should standardize repository creation, tag immutability, scanning configuration where supported, encryption, lifecycle policies, and access controls.

It should not decide whether an application release is approved for production. That decision belongs to the release and GitOps promotion process described in Chapter 183.

### Suggested module structure

```
modules/
└── ecr/
    ├── main.tf
    ├── variables.tf
    ├── outputs.tf
    ├── versions.tf
    ├── README.md
    └── examples/
        └── basic/
```

## 38. Define ECR module inputs

`modules/ecr/variables.tf`:

```
variable "repository_name" {
  description = "ECR repository name"
  type        = string

  validation {
    condition     = length(trimspace(var.repository_name)) > 0
    error_message = "repository_name must not be empty."
  }
}

variable "image_tag_mutability" {
  description = "ECR image tag mutability setting"
  type        = string
  default     = "IMMUTABLE"

  validation {
    condition = contains(
      ["IMMUTABLE", "MUTABLE"],
      var.image_tag_mutability
    )

    error_message = "image_tag_mutability must be IMMUTABLE or MUTABLE."
  }
}

variable "scan_on_push" {
  description = "Whether ECR basic scanning is enabled on image push"
  type        = bool
  default     = true
}

variable "tags" {
  description = "Resource tags"
  type        = map(string)
  default     = {}
}
```

For production images, immutable tags are a useful default because they prevent a published tag from silently being overwritten under normal repository tag-immutability enforcement.

However, immutability of a tag does not itself prove that an image is safe, authentic, or approved. We still need image scanning, provenance or signing controls where required, access restrictions, and digest-based deployment.

Also, do not confuse ECR basic scanning configuration with every available enhanced scanning capability. The appropriate scanning mode and coverage depend on the AWS service configuration and the organization's security requirements.

## 39. Create the repository

`modules/ecr/main.tf`:

```
resource "aws_ecr_repository" "this" {
  name                 = var.repository_name
  image_tag_mutability = var.image_tag_mutability

  image_scanning_configuration {
    scan_on_push = var.scan_on_push
  }

  encryption_configuration {
    encryption_type = "AES256"
  }

  tags = merge(
    var.tags,
    {
      Name      = var.repository_name
      ManagedBy = "terraform"
    }
  )
}
```

This example uses ECR-managed AES-256 encryption. If policy requires a customer-managed KMS key, implement that option deliberately and ensure the key policy and lifecycle are compatible with ECR and the organization's recovery requirements.

### Add a lifecycle policy

Container registries can grow quickly as developers publish feature branches, pull-request builds, and release candidates.

A lifecycle policy can remove eligible images according to an approved retention policy.

Example policy JSON:

```
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Expire old untagged images",
      "selection": {
        "tagStatus": "untagged",
        "countType": "imageCountMoreThan",
        "countNumber": 30
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
```

This example is a starting point, not a complete enterprise retention policy. A real policy must account for tagged releases, rollback windows, active deployments, compliance retention, and image replication. ECR lifecycle policies are evaluated by AWS asynchronously, and an image being deleted from a repository can break future rollbacks if the deployment still depends on it.

A more advanced policy can use tag prefixes and age-based rules for disposable build artifacts, but do not expire production rollback images without a defined recovery alternative.

## 40. Output the ECR identifiers

`modules/ecr/outputs.tf`:

```
output "repository_name" {
  description = "ECR repository name"
  value       = aws_ecr_repository.this.name
}

output "repository_url" {
  description = "ECR repository URL"
  value       = aws_ecr_repository.this.repository_url
}

output "repository_arn" {
  description = "ECR repository ARN"
  value       = aws_ecr_repository.this.arn
}
```

The repository URL can be passed to a build workflow through approved environment configuration. The workflow should still authenticate using its authorized role and publish the exact image it has built and verified.

For the 21 services, the infrastructure configuration can instantiate the same module repeatedly using a map of approved service names. This is more maintainable than manually duplicating 21 ECR resource definitions.

## 41. Manage 21 ECR repositories consistently

A root configuration can use `for_each` to instantiate the module.

```
locals {
  services = toset([
    "auth",
    "user",
    "customer",
    "catalog",
    "product",
    "inventory",
    "pricing",
    "promotion",
    "cart",
    "order",
    "payment",
    "fraud",
    "shipping",
    "delivery",
    "notification",
    "search",
    "recommendation",
    "review",
    "tax",
    "invoice",
    "reporting"
  ])
}

module "service_ecr" {
  for_each = local.services

  source = "git::https://github.com/company-ecommerce/platform-terraform-modules.git//modules/ecr?ref=v1.0.0"

  repository_name      = "ecommerce/${each.value}"
  image_tag_mutability = "IMMUTABLE"
  scan_on_push         = true

  tags = {
    Application = "ecommerce"
    Service     = each.value
    ManagedBy   = "terraform"
  }
}
```

This is an illustrative configuration. Confirm that the repository naming scheme meets your organization's ECR naming standards and that the chosen module version contains the inputs shown.

If a service is renamed, Terraform may interpret the change as removing one repository and creating another. Before changing a key or repository name, determine how the state address and existing images will be handled. Do not casually destroy a repository that contains production images or rollback artifacts.

## 42. Implement the GitHub Actions OIDC IAM module

Our CI workflows need to assume AWS roles without storing permanent AWS access keys in GitHub secrets.

The preferred pattern is generally:

1. GitHub Actions requests an OIDC token.
2. AWS STS validates the token against a configured identity provider and role trust policy.
3. The workflow receives temporary credentials.
4. AWS IAM authorizes only the actions permitted by that role.

For this project, use different roles or trust boundaries for different responsibilities—for example, publishing images, reading state and planning infrastructure, and applying infrastructure.

Do not create one unrestricted `github-actions-admin` role for every repository and workflow.

### Trust-policy considerations

A GitHub OIDC role trust policy should restrict the expected audience and repository/workflow identity. A broad subject pattern can allow unintended repositories or branches to assume the role.

Illustrative trust-policy fragment:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:company-ecommerce/platform-infrastructure:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

Replace `<ACCOUNT_ID>` with the actual AWS account ID in the approved deployment configuration. This fragment assumes the identity provider already exists and that the workflow's OIDC subject matches the configured trust condition.

The exact `sub` value changes with the GitHub event and identity context. Pull-request workflows, environment-bound jobs, and branch-based workflows can produce different subject claims. Test the trust conditions against the actual workflow before deployment.

For high-risk production applies, use a protected GitHub environment with required reviewers and restrict the role trust to the intended environment identity where that is the selected approach.

## 43. Separate IAM permissions by workflow purpose

| Role             | Expected capability                                   | Avoid granting                                         |
| ---------------- | ----------------------------------------------------- | ------------------------------------------------------ |
| Image publisher  | Push to approved ECR repositories                     | Broad EC2, IAM, or EKS administration                  |
| Terraform plan   | Read state and inspect resources needed for planning  | Unrestricted infrastructure mutation                   |
| Terraform apply  | Manage resources owned by the approved stack          | Organization-wide administrator access                 |
| GitOps promotion | Update approved GitOps configuration                  | Direct, unrestricted production cluster administration |
| Runtime workload | Access specific AWS resources required by one service | Permissions for every microservice                     |

IAM policies should be designed against the actual operations performed by each workflow. For example, an image publisher generally needs ECR authorization and upload actions scoped to its permitted repositories, plus the appropriate identity and token operations.

Some AWS APIs require resource-level permissions, while others require `"Resource": "*"` for particular actions. Where a wildcard is required, restrict it with conditions when supported and document the reason. Do not use `Resource = "*"` as a shortcut for an entire role.

## 44. Workload identity for the microservices

Consider the Payment Service. It may need to read a payment-provider configuration from Secrets Manager, but it should not receive the same permissions as the Reporting Service.

A suitable access design is:

```
payment-service Kubernetes ServiceAccount
        |
        v
Payment workload IAM identity
        |
        v
Approved Secrets Manager resource
```

The implementation must include all parts of the identity chain:

- The selected EKS workload identity mechanism.
- The IAM role and narrowly scoped permissions.
- The Kubernetes ServiceAccount configuration.
- The workload's pod or service-account configuration.
- Any required trust relationship, association, or cluster add-on.
- A runtime test confirming that the intended access succeeds and unrelated access is denied.

Creating an IAM role alone does not attach that role to a Kubernetes workload. Chapter 190 will standardize reusable ServiceAccount templates; the current chapter establishes the underlying AWS infrastructure and identity boundary.

## 45. Keep Terraform and Kubernetes responsibilities separate

A common ownership mistake is to let Terraform, Helm, Argo CD, and manual `kubectl` commands all manage the same object.

For our platform, use an explicit division:

| Resource or configuration                                     | Preferred owner                                         |
| ------------------------------------------------------------- | ------------------------------------------------------- |
| VPC, subnets, routing, NAT, endpoints                         | Terraform                                               |
| EKS control plane and managed node groups                     | Terraform                                               |
| ECR repositories and IAM roles                                | Terraform                                               |
| Cluster bootstrap resources, depending on chosen architecture | Terraform or a documented bootstrap process             |
| Service deployments and application configuration             | Helm and GitOps                                         |
| ServiceAccount manifests                                      | GitOps, using approved platform templates               |
| Application runtime secrets                                   | Approved secret-management integration                  |
| Emergency manual changes                                      | Authorized incident process, followed by reconciliation |

The table is a proposed project ownership contract. Some organizations manage selected Kubernetes resources through Terraform, but the same resource should not be independently reconciled by two controllers without a deliberate ownership design.

# Part 4 — Environment composition and infrastructure CI/CD

## 46. Compose the EKS stack from the network stack

The EKS root configuration needs the VPC and subnet IDs produced by the network stack.

A practical dependency sequence is:

1. Apply the network foundation.
2. Capture and verify the network outputs.
3. Pass the approved VPC and subnet IDs to the EKS root configuration.
4. Review and apply the EKS plan.
5. Configure required add-ons and cluster access.
6. Verify the cluster is healthy before allowing application deployment.

There are two common approaches for passing outputs between independently managed stacks.

Option A — Explicit configuration inputs

A pipeline retrieves the approved outputs and supplies them to the EKS configuration through controlled variables or a configuration store.

Option B — Remote-state data source

The EKS configuration reads selected outputs from the network state.

Illustrative example:

```
data "terraform_remote_state" "network" {
  backend = "s3"

  config = {
    bucket = var.state_bucket_name
    key    = "ecommerce/dev/network/terraform.tfstate"
    region = var.aws_region
  }
}
```

Then:

```
module "eks" {
  source = "git::https://github.com/company-ecommerce/platform-terraform-modules.git//modules/eks?ref=v1.0.0"

  cluster_name     = "ecommerce-dev"
  cluster_version  = var.cluster_version
  vpc_id           = data.terraform_remote_state.network.outputs.vpc_id

  private_subnet_ids = (
    data.terraform_remote_state.network.outputs.private_subnet_ids
  )

  tags = var.tags
}
```

This assumes the network state exposes those output names. The EKS execution identity also needs the required state access and, if applicable, KMS decrypt permissions.

Security consideration: the remote-state data source can expose the entire state snapshot to the consuming configuration, not just the output names referenced in the code. If the source state contains sensitive values, prefer a narrower configuration publication mechanism or separate the states so that sensitive data is not exposed to unrelated consumers.

## 47. Environment-specific root stacks

A recommended starting model is four independently controlled environment directories:

```
environments/
├── dev/
│   ├── network/
│   ├── eks/
│   ├── ecr/
│   └── iam/
├── qa/
│   ├── network/
│   ├── eks/
│   ├── ecr/
│   └── iam/
├── staging/
│   ├── network/
│   ├── eks/
│   ├── ecr/
│   └── iam/
└── prod/
    ├── network/
    ├── eks/
    ├── ecr/
    └── iam/
```

The `iam` directory is illustrative; you can split identity configuration further where the permissions or lifecycle require it.

### Should DEV and PROD use separate AWS accounts?

For a production-grade enterprise design, separate AWS accounts are often preferable for stronger isolation between non-production and production environments. They also help isolate permissions, billing, quotas, and incident blast radius.

The decision depends on the organization's account structure and governance. If the project uses separate accounts, configure the provider and role-assumption process to target the correct account for each environment.

Do not assume that changing `environment = "prod"` is enough to protect production. Enforce the account, role, state key, repository permissions, and deployment approval boundaries.

## 48. Root-level variables and outputs

Every environment root configuration should declare its expected inputs and outputs explicitly.

Example output file:

```
output "vpc_id" {
  description = "VPC ID for the environment"
  value       = module.network.vpc_id
}

output "private_subnet_ids" {
  description = "Private subnet IDs for the environment"
  value       = module.network.private_subnet_ids
}
```

Example input validation:

```
variable "aws_region" {
  type = string

  validation {
    condition     = var.aws_region == "ap-south-1"
    error_message = "This stack is configured for ap-south-1."
  }
}
```

The region validation above is deliberately restrictive for a single-region example. A multi-region platform should validate against an approved region list instead.

Do not place credentials, passwords, private keys, or application secrets into `.tfvars` files committed to Git. Marking a Terraform variable `sensitive = true` helps redact it from normal output, but does not automatically prevent the value from being stored in state.

## 49. Terraform CI/CD pipeline design

Infrastructure CI should be predictable and repeatable.

This flow describes the desired control sequence. Exact triggers may differ by repository, environment, and risk classification.

### Pull-request workflow

The PR workflow should:

- Check formatting.
- Initialize the intended root configuration.
- Validate Terraform syntax and provider schemas.
- Run linting and security policy checks.
- Produce a plan for the changed stack.
- Publish a concise plan summary and controlled review artifact.
- Fail when required checks fail.

It should not apply infrastructure to PROD just because a pull request has been opened.

### Apply workflow

The apply workflow should:

- Run from a protected branch or approved release context.
- Target the intended environment and state key.
- Assume the approved role through OIDC.
- Enforce environment protection and required reviewers.
- Apply the reviewed change under controlled conditions.
- Capture execution evidence and verify the resulting infrastructure.

A production pipeline should not blindly apply an old plan after unrelated infrastructure changes. Terraform plans are tied to the configuration, provider context, and state assumptions used when they were created. If those assumptions have changed, regenerate and review the plan.

## 50. Reusable Terraform validation workflow

The following is an illustrative GitHub Actions workflow for a root configuration. It assumes the repository contains the specified Terraform directory and uses a pinned, approved Terraform CLI version.

```
name: Terraform Validation

on:
  pull_request:
    paths:
      - "environments/**"
      - "modules/**"
      - ".github/workflows/**"

permissions:
  contents: read

jobs:
  validate:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: "1.10.5"

      - name: Check formatting
        run: terraform fmt -check -recursive

      - name: Initialize DEV network
        working-directory: environments/dev/network
        run: terraform init -backend=false

      - name: Validate DEV network
        working-directory: environments/dev/network
        run: terraform validate
```

The pinned CLI version is an example; select the version approved by your platform team and update it through a reviewed change.

`terraform init -backend=false` initializes the configuration without configuring the remote backend. It is useful for syntax and validation checks that should not access live state, but it does not produce a real infrastructure plan.

For actual planning, the workflow needs the appropriate backend configuration and credentials, plus any necessary read permissions. Keep untrusted pull-request code away from privileged production credentials.

## 51. Validate only the affected root stacks

As the infrastructure repository grows, running every plan for every PR can become expensive and confusing.

A more efficient workflow determines which root stacks are affected and runs validation for the relevant directories.

For example:

| Changed path                  | Typical checks                                          |
| ----------------------------- | ------------------------------------------------------- |
| `modules/vpc/**`              | VPC module tests and every relevant consumer validation |
| `modules/eks/**`              | EKS module tests and relevant EKS root configurations   |
| `modules/ecr/**`              | ECR module tests and repository root validation         |
| `environments/dev/network/**` | DEV network validation and plan                         |
| `environments/prod/eks/**`    | PROD EKS validation and protected plan review           |
| `policies/**`                 | Policy tests and applicable configuration checks        |

The dependency map must be maintained. A change to a shared module can affect multiple environments even when their directories have not changed.

Avoid a workflow that silently skips all checks because its path filter failed to recognize a newly added module directory.

## 52. Security scanning and policy enforcement

Terraform syntax validation cannot detect every insecure or operationally dangerous configuration.

A mature CI pipeline combines multiple types of checks.

| Check                                         | What it can help detect                                           |
| --------------------------------------------- | ----------------------------------------------------------------- |
| `terraform fmt -check`                        | Formatting inconsistencies                                        |
| `terraform validate`                          | Invalid Terraform configuration and incompatible arguments        |
| TFLint                                        | Configurable Terraform and provider-specific lint rules           |
| Checkov, Trivy config, or another IaC scanner | Known configuration risks and security misconfigurations          |
| Policy as code                                | Organization-specific rules that must be enforced                 |
| Plan review                                   | The actual proposed resource changes, replacements, and deletions |
| Module tests                                  | Expected behavior of reusable module inputs and outputs           |

A scanner passing does not prove that an infrastructure design is secure. Configure the tools, select relevant rules, review exceptions, and test the policy behavior against known-good and known-bad configurations.

### Example policy requirements

- Production resources must use approved regions.
- Production data resources must meet encryption requirements.
- Required ownership and environment tags must be present.
- Public ingress must be restricted to approved use cases.
- Production EKS endpoint access must match the approved connectivity model.
- IAM permissions must not grant unnecessary administrative access.
- Destructive changes to protected resources require explicit review.

The policy engine should distinguish a real violation from an approved exception. Exceptions should identify the owner, reason, approval, and expiry or review date where applicable.

## 53. Plan output and destructive changes

The most important part of infrastructure review is understanding what Terraform intends to change.

For example, a plan might contain:

```
Plan: 2 to add, 1 to change, 1 to destroy.
```

That summary is not sufficient for approval.

Reviewers should inspect:

- Which resources will be created.
- Which resource attributes will change in place.
- Which resources will be replaced.
- Which resources will be destroyed.
- Whether replacements affect availability.
- Whether an IAM or security-group change expands access.
- Whether a subnet, database, cluster, or repository change affects dependent systems.
- Whether the plan contains unexpected drift or manual changes.

A replacement of an EKS cluster, a VPC, or a data-bearing resource should receive much more scrutiny than a routine tag update.

A saved plan can be generated with:

```
terraform plan -out=tfplan
```

It can be applied with:

```
terraform apply tfplan
```

These commands are examples of the plan/apply mechanism, not an instruction to apply an unreviewed plan. Use a controlled pipeline, ensure the plan belongs to the intended environment and commit, and protect the artifact because it may contain sensitive values.

## 54. Production approval and account protection

For PROD, configure multiple layers of protection:

1. A dedicated production AWS account or equivalent strong boundary.
2. A production-specific backend key and restricted state access.
3. A separate apply role with scoped permissions.
4. A protected GitHub environment with required reviewers.
5. A plan review showing the proposed changes.
6. An approved change record where organizational process requires it.
7. Post-apply verification and a documented recovery plan.

An approval should not be a substitute for understanding the change. Reviewers should know which resources will be modified, the expected impact, and how to recover if the change causes an incident.

# Part 5 — Module testing, versioning, and releases

## 55. Why reusable modules need tests

A reusable module can be consumed by multiple environments and teams. A defect in the module can therefore affect more than one application or account.

For example, a faulty VPC module could create incorrect subnet routing in DEV, QA, STAGING, and PROD. A faulty IAM module could grant excessive permissions to every service using it.

Testing should cover the module contract and the resulting infrastructure behavior—not just whether Terraform syntax is valid.

Use several layers of testing:

- Static validation.
- Linting and security scanning.
- Input validation tests.
- Unit-style tests for supported module behavior.
- Plan-based integration tests.
- Isolated deployment tests where practical.
- Upgrade tests for existing consumers.
- Destructive-change review.

## 56. Module test scenarios

For the VPC module, useful test scenarios include:

| Scenario                                         | Expected result                                                  |
| ------------------------------------------------ | ---------------------------------------------------------------- |
| Valid VPC CIDR and subnet inputs                 | Validation succeeds                                              |
| Invalid IPv4 CIDR                                | Validation fails                                                 |
| Duplicate AZ values                              | Validation fails                                                 |
| Fewer than two AZs                               | Validation fails                                                 |
| Number of subnet CIDRs differs from AZ count     | Validation fails                                                 |
| Overlapping subnet ranges                        | A module test or additional validation rejects the configuration |
| Private subnet accidentally given public routing | A policy or integration test identifies the risk                 |
| Required tags missing                            | Policy validation fails                                          |
| Module output contract changes unexpectedly      | Consumer tests fail                                              |

The tests should cover supported configurations and negative cases. A module that only passes its happy-path test may still fail when an input list is empty, an AZ changes, or an environment uses a different scale.

## 57. Use Terraform's test framework

Terraform includes a native test framework in supported versions. It can validate module behavior through test files and assertions.

An illustrative `modules/vpc/tests/inputs.tftest.hcl`:

```
run "reject_invalid_environment" {
  command = plan

  variables {
    name                 = "ecommerce-test"
    vpc_cidr             = "10.40.0.0/16"
    availability_zones   = ["ap-south-1a", "ap-south-1b"]
    public_subnet_cidrs  = ["10.40.0.0/20", "10.40.16.0/20"]
    private_subnet_cidrs = ["10.40.64.0/20", "10.40.80.0/20"]
    environment          = "production"
    tags                 = {}
  }

  expect_failures = [
    var.environment
  ]
}
```

This test assumes the module's variables match the interface shown earlier. It expects validation to reject `production`, because the allowed value is `prod`.

Run tests from the module directory with:

```
terraform init -backend=false
terraform test
```

Native Terraform tests can use mock providers in supported configurations, which helps test module logic without creating real infrastructure. Tests that exercise actual AWS behavior need a controlled AWS test environment, appropriate permissions, cost controls, and cleanup procedures.

Check the test syntax against the selected Terraform CLI version. Do not run infrastructure-creating integration tests against PROD.

## 58. When to use Terratest

Terratest is an additional option for teams that want integration tests written in Go.

It can help test behaviors such as:

- Whether an isolated VPC has the expected subnet topology.
- Whether an EKS cluster becomes accessible through the approved network path.
- Whether an ECR repository enforces the intended configuration.
- Whether a deployed resource behaves as expected from an authorized test identity.

The trade-off is that real integration tests can create resources and incur costs. They may also take longer than static validation.

Use an isolated test account, unique resource names, time-bounded execution, and a reliable cleanup strategy. Cleanup should never blindly destroy shared resources based only on a broad name pattern.

## 59. Release modules with explicit versions

A reusable module release is a contract to its consumers.

Example release progression:

```
v1.0.0  Initial reviewed implementation
v1.0.1  Backward-compatible bug fix
v1.1.0  Backward-compatible feature addition
v2.0.0  Breaking input, output, or behavior change
```

This follows semantic versioning as a convention. A team should document what it considers breaking, especially when a Terraform change can replace real resources even if the HCL interface appears unchanged.

For example:

- Adding an optional variable with a safe default may be backward-compatible.
- Renaming a required input can break consumers.
- Changing a resource's `for_each` keys can change Terraform addresses.
- Changing subnet CIDRs may require subnet replacement.
- Changing a resource name may cause replacement or recreation.
- Changing IAM defaults may alter the effective security boundary.

Before releasing a version, test all relevant consumers and inspect plans for representative existing environments.

## 60. Publish and consume module releases

The workflow for a module release should be:

1. Merge the reviewed module change.
2. Run validation, tests, linting, and security checks.
3. Review the expected behavior and upgrade implications.
4. Publish the approved version.
5. Update the release notes and module documentation.
6. Open a separate PR in `platform-infrastructure` to adopt the version.
7. Generate plans for the affected environments.
8. Promote through the organization's environment approval process.

A new module release should not be automatically adopted by every environment merely because a Git tag exists.

Example consumer reference:

```
module "network" {
  source = "git::https://github.com/company-ecommerce/platform-terraform-modules.git//modules/vpc?ref=v1.1.0"

  # Reviewed inputs go here.
}
```

This is an illustrative reference; the specified version must exist and include the module interface expected by the consumer.

## 61. Upgrade one environment at a time

A reasonable promotion strategy is:

```
Module Release
      |
      v
Isolated Module Tests
      |
      v
DEV Consumer Upgrade
      |
      v
QA Consumer Upgrade
      |
      v
STAGING Consumer Upgrade
      |
      v
PROD Consumer Upgrade
```

This is a common sequence, but not every module must follow identical timing. A security-critical change may require a faster controlled promotion, while a risky network refactor may need additional testing.

For each environment:

- Update the module version in a PR.
- Review the exact plan.
- Apply through the approved workflow.
- Run verification checks.
- Record any unexpected behavior.
- Proceed only when the environment meets its acceptance criteria.

Do not assume that a module is safe to promote because it worked in a clean test environment. Existing infrastructure may contain state, drift, or dependencies that a fresh environment does not reproduce.

## 62. Handling breaking changes safely

Suppose version `v1.0.0` exposes:

```
output "private_subnet_ids" {
  value = [for subnet in aws_subnet.private : subnet.id]
}
```

A new module version changes the output name to `private_subnets`.

This is a breaking interface change for consumers that reference `module.network.private_subnet_ids`.

A safe migration includes:

1. Identify every consumer.
2. Publish the new contract and migration instructions.
3. Update consumers in reviewed PRs.
4. Check plans for unintended infrastructure replacement.
5. Apply the changes in the required order.
6. Remove the old interface only when the migration is complete.

Sometimes the Terraform state resource addresses themselves must be migrated. A source-code rename is not proof that the underlying infrastructure should be destroyed and recreated.

## 63. Avoid unreviewed module source changes

These module source patterns have different operational risks:

```
# Versioned source
source = "git::https://github.com/company-ecommerce/platform-terraform-modules.git//modules/vpc?ref=v1.0.0"
```

```
# Floating branch source: unsuitable for controlled production releases
source = "git::https://github.com/company-ecommerce/platform-terraform-modules.git//modules/vpc?ref=main"
```

A branch reference can change as new commits are added. A production plan could therefore use different module content from a previous run even though the environment configuration looks unchanged.

Prefer reviewed releases with protected tags or another immutable artifact mechanism. Also secure the source repository and release process: version pinning does not protect against an attacker who can alter the source artifact or the trust chain.

# Part 6 — Drift detection and infrastructure operations

## 64. What is Terraform drift?

Drift occurs when real infrastructure no longer matches the configuration and state Terraform expects.

Common causes include:

- A human changes an AWS resource through the console.
- An automation tool changes a resource outside Terraform.
- A resource is modified by another IaC system.
- A configuration update is merged without being applied.
- An external change affects an attribute managed by Terraform.

For example, someone may manually widen an EKS security group's inbound rule to allow a troubleshooting connection. Terraform might later detect that the rule differs from the expected configuration.

Drift is not automatically malicious. Some differences are expected because AWS services manage certain fields or because providers normalize values. The team must distinguish meaningful unmanaged changes from harmless differences.

## 65. Run a controlled drift check

From the relevant root configuration:

```
terraform init
terraform plan -detailed-exitcode
```

For the standard detailed exit-code behavior:

- `0` indicates a successful plan with no changes.
- `1` indicates an error.
- `2` indicates a successful plan with changes.

This command can help detect changes, but a plan with changes is not necessarily drift. It may simply represent a legitimate configuration change that has not yet been applied.

A drift job should therefore compare the plan with the intended configuration and recent approved changes, rather than automatically applying every proposed correction.

### Why not run `terraform apply` automatically after drift detection?

Because an automatically applied plan could:

- Undo a legitimate emergency change.
- Replace a production resource.
- Remove an access path required for recovery.
- Change a resource before an incident commander has approved the remediation.
- Apply a stale or incorrectly targeted plan.

Drift detection should create an actionable report or ticket. Remediation should follow the approved change process unless a separately authorized, narrowly scoped automated remediation policy exists.

## 66. Investigate drift before fixing it

When drift appears, follow a structured investigation.

1. Confirm the correct AWS account, region, root directory, and state key.
2. Capture the plan output and identify each affected resource.
3. Determine whether the change came from an approved PR, a manual operation, an AWS service, or another automation system.
4. Check whether the change was an emergency response to an incident.
5. Compare the actual resource configuration with the declared Terraform configuration.
6. Decide whether the desired outcome is to keep the change or restore the intended configuration.
7. Update the configuration or restore the resource through an approved plan.
8. Record the root cause and add preventive controls where appropriate.

Do not simply overwrite the real resource or change the Terraform code until the reason for the drift is understood.

## 67. Import existing AWS resources

Sometimes the platform team needs to bring an existing resource under Terraform management.

For example, an ECR repository may already exist because it was created manually before the module was introduced.

The safe process is:

1. Write the Terraform resource or module configuration.
2. Confirm the real resource's settings.
3. Define the correct Terraform resource address.
4. Import the resource into the intended state.
5. Run a plan and inspect the result.
6. Reconcile the configuration with the real resource.
7. Apply only after the plan shows the intended behavior.

For an individual resource, Terraform supports commands such as:

```
terraform import aws_ecr_repository.example existing-repository-name
```

This example applies only when the root configuration actually contains the resource address `aws_ecr_repository.example` and the provider supports that import format.

For resources managed inside modules or with `for_each`, the address can be more complex. Confirm the exact address before importing.

An import does not automatically create a complete, accurate Terraform configuration. The code must still represent the desired resource settings.

## 68. Rename Terraform resources without accidental recreation

Consider a module that originally defines:

```
resource "aws_subnet" "private" {
  # Resource configuration
}
```

Later, an engineer renames the resource block to `aws_subnet.application_private`.

Terraform may interpret this as one resource being removed and another being created unless the address migration is represented.

A `moved` block can document a state-address migration:

```
moved {
  from = aws_subnet.private
  to   = aws_subnet.application_private
}
```

This is illustrative. The correct addresses depend on the actual configuration and module boundaries. For resources inside modules or `for_each` collections, the old and new addresses may include module paths or instance keys.

A moved block changes how Terraform maps addresses; it does not make an incompatible infrastructure change safe. Inspect the plan to verify that Terraform recognizes the intended migration.

## 69. Recover from a damaged or incorrect state

Terraform state recovery is a high-risk operation. Do not experiment with state modification against production.

A recovery procedure should include:

1. Stop concurrent Terraform operations for the affected state.
2. Identify the correct backend bucket and state key.
3. Confirm the intended AWS account and environment.
4. Inspect available S3 object versions and the most recent known-good state.
5. Preserve the current state version and relevant audit evidence.
6. Determine whether the problem is a state issue, a real infrastructure change, or a backend access problem.
7. Follow the organization's approved state-recovery procedure.
8. Run a plan before any apply.
9. Verify that Terraform's resource mapping matches real infrastructure.
10. Record the recovery and any follow-up preventive action.

S3 versioning can help recover an earlier object version, but restoring an older state snapshot may omit infrastructure changes that occurred after that snapshot. A blind restore can cause Terraform to lose track of real resources or attempt unintended recreation.

Never use `terraform state push -force` as a routine troubleshooting command. State surgery should be reserved for authorized operators who understand the consequences and have preserved recovery evidence.

## 70. State-lock troubleshooting

If Terraform reports that the state is locked, do not immediately force-unlock it.

First verify:

- Is another Terraform plan or apply running?
- Is a CI job still active or waiting for an environment approval?
- Did a previous process terminate unexpectedly?
- Does the lock belong to the same backend and state key?
- Is the reported lock ID current?
- Has the authorized operator confirmed that the original operation is no longer active?

If the lock is stale and the team has verified that no operation is running, follow the documented recovery process for the configured backend.

Forcing an unlock while another operation is active can allow concurrent state changes and corrupt the intended workflow. Use the backend's supported locking mechanism for the selected Terraform version and avoid mixing older DynamoDB instructions with an S3 native-lockfile deployment without a planned migration.

# Part 7 — Production troubleshooting runbooks

## 71. Troubleshooting: Terraform initialization fails

### Symptoms

```
Error: Failed to get existing workspaces
Error: AccessDenied
Error: Failed to load state
```

### Possible causes

- Incorrect backend bucket or state key.
- Incorrect AWS account or region.
- Missing S3 permissions.
- Missing KMS permissions for a customer-managed encryption key.
- Network connectivity or endpoint policy restrictions.
- Unsupported backend configuration for the installed Terraform version.
- Incorrect AWS credentials or expired temporary credentials.

### Investigation

```
aws sts get-caller-identity
terraform version
terraform init
```

`aws sts get-caller-identity` confirms which AWS identity is active. It does not prove that the identity can access the Terraform backend or manage the target resources.

Then verify the backend settings and the permissions for the exact bucket and key. If the state is encrypted with a customer-managed KMS key, verify the applicable IAM and key policies.

### Recovery

1. Confirm the target environment and backend configuration.
2. Verify the active identity.
3. Correct the configuration or permissions through the approved process.
4. Retry initialization.
5. Run a plan to confirm that the state was loaded correctly.

Do not delete the state object or reinitialize against a new key merely to make an error disappear.

## 72. Troubleshooting: Terraform wants to replace a resource

### Symptoms

The plan contains an action similar to:

```
-/+ resource will be replaced
```

This indicates that Terraform plans to destroy and recreate the resource rather than change it in place.

### Investigation questions

- Which argument forces replacement?
- Did a name, CIDR, identifier, or immutable property change?
- Was the module version changed?
- Did a `for_each` key or resource address change?
- Is the plan detecting unexpected drift?
- Does another resource depend on this resource?
- Could replacement interrupt production traffic or delete data?

### Recovery

Do not approve the plan solely because the code change appears small.

Identify the provider's replacement behavior for the changed argument. Determine whether a migration, parallel resource, or controlled cutover is needed. If the change is unintended, correct the configuration or address migration and generate a new plan.

For a production VPC, subnet, EKS cluster, or stateful data resource, a replacement can be an architectural migration—not a routine update.

## 73. Troubleshooting: EKS nodes fail to join the cluster

### Symptoms

- Managed node groups report degraded health.
- EC2 instances launch but do not become Kubernetes nodes.
- Workloads remain pending because no usable nodes are available.

### Check the infrastructure layers

1. EKS cluster: verify cluster status and endpoint configuration.
2. Subnets: confirm the nodes are launched in the intended subnets and have sufficient IP capacity.
3. Routing: verify required connectivity to the EKS endpoint and AWS dependencies.
4. Node IAM role: verify the node role and required policies.
5. Security groups: verify the necessary cluster-to-node communication.
6. DNS: confirm the nodes can resolve required names.
7. Bootstrap and AMI: verify compatibility with the selected Kubernetes version and node configuration.
8. Capacity: check whether the requested instance types are available in the chosen AZs.
9. Cluster access: confirm the node identity is permitted to join the cluster under the selected access model.

Do not grant broad administrator permissions to the node role as a quick fix. Identify the failed operation and correct the specific missing permission or configuration.

## 74. Troubleshooting: ECR image pulls fail

### Symptoms

Pods show image-pull errors, such as:

```
ImagePullBackOff
ErrImagePull
```

### Check the relevant layers

- Does the repository exist in the intended AWS account and region?
- Does the image tag or digest exist?
- Is the image architecture compatible with the node?
- Can the nodes reach ECR and the endpoints required for image-layer downloads?
- Are the endpoint policies, security groups, and DNS configuration correct?
- Does the node or workload identity have the required permissions for the selected image-pull architecture?
- Was the image removed by an overly aggressive lifecycle policy?

Do not assume every image-pull error is an IAM problem. A missing tag, wrong account, inaccessible endpoint, incompatible architecture, or deleted image can produce similar symptoms.

The platform team should test image pulls from a representative node environment and verify the actual image digest rather than relying only on the existence of a tag.

## 75. Troubleshooting: Terraform plan shows unexpected changes in every environment

When a shared module change causes widespread plans, do not immediately apply each plan.

Investigate whether:

- A provider version changed.
- Default tags or provider behavior changed.
- A module version was updated.
- A resource naming or `for_each` key changed.
- A computed attribute is now handled differently.
- AWS has changed a managed value.
- Environment-specific configuration has diverged.
- The root stacks are using different versions of a shared module.

Start with a representative non-production environment. Determine whether the plan represents intended changes, harmless normalization, or an actual regression.

If the module change is responsible, pause promotion and release a correction or a documented migration. Avoid applying the same unexplained plan across all environments.

## 76. Troubleshooting: GitHub Actions cannot assume the AWS role

Typical causes include:

- The OIDC provider is missing or incorrectly configured.
- The token audience does not match the trust policy.
- The token subject does not match the expected repository, branch, or environment.
- The workflow lacks the permission to request an OIDC token.
- The role ARN points to the wrong account.
- A protected environment or branch condition prevents the expected identity context.
- The role's trust policy has been tightened without updating the workflow.

The workflow generally needs:

```
permissions:
  contents: read
  id-token: write
```

Only grant the OIDC token permission to jobs that need it. Then verify the actual role trust conditions and the job's identity context.

A workflow that can assume the role still might not have permission to read state or modify AWS resources. Authentication and authorization are separate steps.

## 77. Troubleshooting: Terraform succeeds, but the platform is unhealthy

Terraform's successful completion means that the provider operations it performed succeeded. It does not prove that all applications, Kubernetes add-ons, or business workflows are healthy.

For example, Terraform may create a cluster successfully while:

- The node group has insufficient capacity.
- CoreDNS is not ready.
- A load-balancer controller is misconfigured.
- Argo CD cannot reach the API endpoint.
- The application cannot access its database.
- The GitOps repository points to an unavailable image.

Post-apply verification should therefore include infrastructure checks and the appropriate platform health checks.

Use the right owner for the next action:

- AWS resource issue → cloud/platform team.
- Kubernetes scheduling or networking issue → platform/Kubernetes team.
- Application health issue → service owner with platform support.
- Access or policy issue → platform/security team.
- Data integrity issue → application/data owner and DBA as appropriate.

# Part 8 — Production change management

## 78. Classify infrastructure changes by risk

Not every Terraform change requires the same approval path. A mature platform classifies changes according to their impact.

| Risk class | Example                                                 | Typical controls                                                                           |
| ---------- | ------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| Low        | Non-sensitive tag correction                            | Automated validation, review, approved apply                                               |
| Medium     | Add a non-production ECR repository                     | Plan review and normal environment controls                                                |
| High       | Change production security-group rules                  | Security review, impact assessment, production approval                                    |
| Critical   | Replace production networking or cluster infrastructure | Formal change plan, dependency analysis, recovery plan, scheduled execution where required |

The exact classification is organization-specific. A change that looks small in code can have a large impact if it modifies a shared resource.

## 79. Example: changing production subnet capacity

Suppose the team discovers that EKS workloads are running low on available IP addresses.

The wrong approach is to widen a subnet or replace it without analyzing the dependencies.

A controlled approach is:

1. Confirm the actual IP exhaustion symptoms.
2. Inspect current subnet allocations and growth forecasts.
3. Determine whether additional subnets, secondary CIDRs, or a broader networking change is appropriate.
4. Review AWS EKS and VPC CNI compatibility for the selected approach.
5. Update the Terraform configuration in a PR.
6. Validate CIDR overlap and routing.
7. Generate a plan and identify any replacement operations.
8. Test the design in a representative non-production environment.
9. Obtain production approval.
10. Apply the change in the planned sequence.
11. Verify available IP capacity and workload behavior.
12. Document the resulting design and capacity thresholds.

Adding a subnet does not automatically guarantee that all existing pods will move to it or that the networking configuration will start using it. The node and pod networking behavior must be understood and tested.

## 80. Example: upgrading the EKS cluster version

A Kubernetes version upgrade can affect the control plane, add-ons, node groups, APIs, and application workloads.

A responsible process includes:

- Reviewing the supported EKS upgrade path.
- Identifying deprecated APIs and removed features.
- Checking add-on and node AMI compatibility.
- Verifying application compatibility.
- Reviewing disruption budgets and workload replica counts.
- Planning node replacement and workload rescheduling.
- Testing the upgrade in a non-production environment.
- Preparing monitoring and rollback or recovery actions.
- Applying the production change through the approved workflow.
- Verifying cluster and application health afterward.

Terraform can manage parts of the upgrade, but it cannot guarantee that every application remains compatible with the new Kubernetes version. That requires workload-level testing and operational readiness.

## 81. Example: removing an ECR repository

Deleting a repository can remove access to images needed for current deployments or future rollback.

Before authorizing repository deletion:

1. Identify all images and tags in the repository.
2. Check whether running workloads use images from that repository.
3. Check whether any approved rollback version depends on those images.
4. Confirm that images are retained or replicated elsewhere if required.
5. Verify the owning service has migrated to the replacement repository.
6. Review the Terraform plan for deletion.
7. Obtain the required approval.
8. Remove the repository only after the dependencies are resolved.

A repository should not be destroyed just because its Terraform module instance was removed from a `for_each` collection.

## 82. Keep an auditable change record

For a high-risk infrastructure change, record:

```
Change ID:
Environment:
AWS account:
Terraform root configuration:
Module version changes:
Git commit:
Plan artifact reference:
Risk classification:
Expected impact:
Reviewer / approver:
Execution identity:
Apply time:
Post-apply verification:
Recovery plan:
Outcome:
Follow-up actions:
```

These fields are a template. Do not put secrets or unrestricted state contents into a ticket.

The change record connects the PR, reviewed plan, approved execution, and post-change verification. It also helps the team reconstruct events during an incident.

# Part 9 — Module documentation and developer experience

## 83. Every module needs a useful README

The README is part of the module's interface. A developer should not need to inspect every `.tf` file to understand the supported usage.

Each module README should include:

1. Purpose and supported use cases.
2. Architecture and resource ownership.
3. Prerequisites.
4. Terraform and provider compatibility.
5. Required and optional inputs.
6. Outputs and their meanings.
7. A minimal usage example.
8. Security assumptions.
9. Known limitations.
10. Testing instructions.
11. Versioning and migration notes.
12. Troubleshooting guidance.
13. Resource replacement and destruction risks.

Example README section:

```
# VPC Module

## Purpose

Creates the approved VPC and subnet foundation for an environment.

## Inputs

- name
- vpc_cidr
- availability_zones
- public_subnet_cidrs
- private_subnet_cidrs
- environment
- tags

## Outputs

- vpc_id
- public_subnet_ids
- private_subnet_ids
- availability_zones

## Important notes

- Validate CIDR overlap before deployment.
- Review subnet capacity for EKS workloads.
- Public subnet tags do not create public routes.
- Routing and egress behavior must match the approved network design.
- Review replacement operations before applying.
```

The README must match the released module version. Do not document inputs or outputs that do not actually exist.

## 84. Build a module catalog

As the platform repository grows, maintain a catalog of approved modules and versions.

| Module             | Responsibility                                | Typical consumers               |
| ------------------ | --------------------------------------------- | ------------------------------- |
| `vpc`              | Network foundation                            | Environment network stacks      |
| `eks`              | Kubernetes control plane and node integration | EKS environment stacks          |
| `ecr`              | Container image repositories                  | Shared registry stack           |
| `iam-github-oidc`  | CI identity federation                        | CI and infrastructure workflows |
| `iam-eks-workload` | Workload-specific AWS access                  | Kubernetes service identities   |
| `security-groups`  | Approved network access patterns              | Network and platform stacks     |
| `kms`              | Encryption key patterns                       | Selected encrypted resources    |

This catalog is a starting design. If a capability is too complex to expose safely as a generic module, provide a narrower approved pattern instead of an overly flexible interface.

## 85. Avoid a module with too many switches

A module can become difficult to operate when it has dozens of booleans:

```
enable_nat               = true
enable_endpoint_s3       = true
enable_endpoint_ecr      = true
enable_endpoint_logs     = true
enable_public_api        = false
enable_special_route     = true
enable_custom_dns        = false
```

Some optional inputs are useful, but a module should not become an undocumented collection of unrelated architecture decisions.

Prefer a small number of clear, validated configuration options. For substantially different architectures, separate modules or documented profiles can be easier to test and govern.

For example, a cost-optimized development network and a highly available production network may share subnet primitives but use different approved compositions.

## 86. Make module consumers predictable

A good module interface should make expected behavior clear:

- Required inputs have no misleading defaults.
- Optional inputs have safe, documented defaults.
- Names and tags are predictable.
- Outputs have stable meanings.
- Resource replacement behavior is documented.
- Dependencies are explicit.
- Permissions are not broader than necessary.
- Examples use the same interface as the released module.

The module should not silently select PROD, invent a production CIDR, assume an AWS account, or create public exposure merely because the caller omitted a value.

# Part 10 — Practical implementation exercise

## 87. Exercise objective

Build the first usable version of the Terraform platform layer in a controlled AWS development environment.

The exercise is not complete merely because the files exist. It is complete when the configuration validates, the plan is understood, the intended resources are provisioned, and the resulting infrastructure is verified.

## 88. Implementation order

Step 1 — Bootstrap state

Create the restricted state bucket, enable versioning and encryption, configure access, and test initialization.

Step 2 — Build the VPC module

Implement inputs, outputs, subnet resources, routing, security assumptions, and tests.

Step 3 — Build the EKS module

Add the control plane, node groups, cluster access, and required add-on strategy.

Step 4 — Build the ECR module

Create service repositories with reviewed immutability, scanning, and lifecycle settings.

Step 5 — Add CI controls

Run formatting, validation, linting, policy checks, tests, and controlled plans through GitHub Actions.

Step 6 — Verify and document

Test the deployed resources, publish module documentation, and record the release and recovery procedures.

## 89. Validation commands

Run these commands in the appropriate module or root directory:

```
terraform fmt -check -recursive
terraform init -backend=false
terraform validate
terraform test
```

These commands cover formatting, initialization without the remote backend, configuration validation, and native Terraform tests. They do not by themselves verify live AWS behavior.

For an environment plan, initialize against the correct backend and use the authorized AWS identity:

```
terraform init
terraform plan -out=tfplan
```

Review the plan and run the controlled apply workflow only after the required approval.

Do not use a local test configuration with production backend settings by accident. Always confirm the directory, backend key, account, region, and active identity before performing live infrastructure operations.

## 90. What evidence should the engineer produce?

For the exercise, retain evidence of:

- Module source and version.
- Successful static checks and test results.
- The reviewed infrastructure plan.
- The identity and environment used for execution.
- The created AWS resources and their key settings.
- EKS cluster and node-group health, where deployed.
- ECR repository configuration.
- The outcome of security and policy checks.
- The state backup and recovery procedure.
- Known limitations and follow-up actions.

Evidence should be sanitized. Do not attach credentials, sensitive state files, or unrestricted plan artifacts to a public repository.

# Part 11 — Production readiness checklist

## 91. Terraform module readiness

Modules have documented inputs, outputs, prerequisites, and supported versions.

Module interfaces use explicit inputs rather than hidden environment assumptions.

Resource names and tags follow platform standards.

Resource replacement behavior is documented.

Module tests include invalid-input and failure scenarios.

Shared module changes are tested against existing consumers.

Releases use reviewed, controlled module versions.

Breaking changes include migration instructions.

## 92. AWS infrastructure readiness

Network CIDRs, routing, AZ placement, and subnet capacity are reviewed.

Private and public subnet behavior matches the security design.

EKS control-plane and node IAM roles are separated.

Cluster access and endpoint connectivity are tested.

Node groups have reviewed scaling limits and upgrade procedures.

Required cluster add-ons have clear ownership.

ECR repositories use approved encryption, scanning, and retention settings.

GitHub Actions uses restricted OIDC roles and short-lived credentials.

Workload identities grant only the permissions each service requires.

## 93. State and CI/CD readiness

Remote state is encrypted, versioned, and access-restricted.

Backend locking is configured for the selected Terraform version.

DEV, QA, STAGING, and PROD use deliberate state boundaries.

State and sensitive plan artifacts are excluded from source control.

CI performs formatting, validation, testing, and policy checks.

Production plans receive meaningful review before application.

Production applies use protected environments and restricted roles.

Drift detection reports unexpected changes without blindly applying them.

State recovery and lock troubleshooting procedures are documented.

Post-apply verification confirms infrastructure and platform health.

## 94. Definition of done

Chapter 184 is complete when the platform team can demonstrate all of the following:

1. A reusable Terraform module can be consumed by multiple environment configurations without copying its implementation.
2. The DEV infrastructure stack can be planned and applied through the approved workflow.
3. Module and environment changes are validated and tested.
4. AWS identity and Terraform state access are restricted appropriately.
5. Module releases can be adopted independently by each environment.
6. Infrastructure changes have a reviewable plan and an approved execution path.
7. Drift and state problems have documented investigation and recovery procedures.
8. EKS, ECR, and network resources have clear ownership boundaries.
9. The platform can identify the exact infrastructure configuration and module versions used by an environment.
10. A tested procedure exists for responding to failed or destructive infrastructure changes.

These are acceptance criteria, not claims that the infrastructure has already been deployed or verified in your AWS account.

## 95. How Chapter 184 connects to the platform

Chapter 182 — Reusable Helm Templates

Standard Kubernetes deployment resources and application configuration.

Chapter 183 — Reusable GitHub Actions

Reusable CI workflows for testing, scanning, building, and publishing application artifacts.

Chapter 184 — Terraform Modules

Reusable AWS infrastructure, environment composition, IAM, state, and infrastructure change controls.

The combined platform now has three distinct reusable layers: infrastructure provisioning, application deployment, and CI/CD execution. They must share clear interfaces without competing to manage the same resources.

## Next: Chapter 185 — GitHub Actions Reusable Workflows

Chapter 183 established reusable actions and CI building blocks. Chapter 185 will define the reusable workflow layer that orchestrates those building blocks across the 21 service repositories, including:

- Centralized workflow contracts and input/output definitions.
- Java, Node.js, Python, and Go workflow patterns.
- Build, test, scan, and publish job boundaries.
- GitHub Actions permissions and OIDC trust.
- Workflow versioning and release promotion.
- Pull-request validation versus release workflows.
- Secure artifact and image-digest handoff.
- Environment protection and production approval.
- Failure handling, concurrency, caching, and troubleshooting.
- Integration with Terraform, ECR, Helm, and GitOps.

The key distinction: a reusable action performs a focused task; a reusable workflow orchestrates a complete job or pipeline.