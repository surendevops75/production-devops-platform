# 19-EKS.md

# Amazon EKS — Production Kubernetes Architecture

Until now, we discussed Kubernetes as a platform.

Now we move into the **actual AWS production implementation**.

Our enterprise e-commerce platform will run on Amazon EKS.

The important goal of this chapter is not just to learn:

```bash
eksctl create cluster
```

Instead, we want to understand:

> **If I joined a real DevOps team operating an EKS platform, how was the cluster built, what components are running, who manages each component, how networking works, how workloads reach AWS services, and how do I troubleshoot the cluster when production fails?**

---

# 1. Our Target Production Architecture

The high-level architecture is:

```text
                              AWS REGION
                             ap-south-1
                                  │
                                  ▼
                              VPC
                         10.0.0.0/16
                                  │
             ┌────────────────────┴────────────────────┐
             │                                         │
        Public Subnets                            Private Subnets
             │                                         │
       ┌─────┼─────┐                         ┌─────────┼─────────┐
       ▼     ▼     ▼                         ▼         ▼         ▼
      AZ-A  AZ-B  AZ-C                     AZ-A      AZ-B      AZ-C
       │     │     │                         │         │         │
       │     │     │                         └─────────┼─────────┘
       │     │     │                                   │
       │     │     │                              EKS Nodes
       │     │     │                                   │
       │     │     │                    ┌──────────────┼──────────────┐
       │     │     │                    ▼              ▼              ▼
       │     │     │                  Node           Node           Node
       │     │     │                    │              │              │
       │     │     │                  Pods           Pods           Pods
       │     │     │
       │     │     └─────────────── AWS Load Balancer
       │     │
       └─────┴────────────────────── Internet Gateway
```

Production workloads run primarily in **private subnets**.

---

# 2. What Is EKS?

Amazon EKS is AWS's managed Kubernetes service.

The key word is:

> **Managed control plane**

AWS operates the Kubernetes control-plane infrastructure for the EKS cluster.

We still manage the Kubernetes workloads and much of the surrounding platform architecture.

---

# 3. What AWS Manages vs What We Manage

This distinction is extremely important in production.

## AWS manages

Conceptually:

```text
EKS Control Plane
├── Kubernetes API Server
├── etcd
├── Control-plane availability
└── Control-plane infrastructure
```

## Our DevOps/Platform team manages

```text
EKS
├── VPC
├── Subnets
├── Security
├── Worker capacity
├── Node groups / Karpenter
├── Kubernetes workloads
├── Services
├── Ingress
├── Configurations
├── RBAC
├── NetworkPolicies
├── Observability
├── Application deployments
└── GitOps
```

The exact boundary evolves as AWS introduces managed capabilities.

---

# 4. EKS Is Not Just "A Kubernetes Cluster"

In production, think of EKS as:

```text
AWS
│
├── VPC
│
├── EKS Control Plane
│
├── Worker Capacity
│
├── IAM
│
├── Load Balancers
│
├── ECR
│
├── CloudWatch
│
├── KMS
│
├── Route 53
│
└── Other AWS Services
```

EKS connects Kubernetes with AWS infrastructure.

---

# 5. Production AWS Account

Our target architecture eventually uses separate AWS accounts:

```text
AWS Organization
│
├── Management
├── Security
├── Logging
├── Development
├── QA
└── Production
```

Our EKS production cluster belongs in:

```text
Production AWS Account
```

The actual account IDs are environment-specific.

---

# 6. Production Region

For this project we use:

```text
ap-south-1
```

Mumbai.

But Terraform should make the region configurable:

```hcl
variable "aws_region" {
  type    = string
  default = "ap-south-1"
}
```

Don't hardcode the region throughout every Terraform file.

---

# 7. Availability Zones

Our production cluster should span multiple AZs.

For example:

```text
ap-south-1a
ap-south-1b
ap-south-1c
```

Architecture:

```text
                  EKS
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
     AZ-A        AZ-B        AZ-C
       │           │           │
     Nodes        Nodes       Nodes
       │           │           │
      Pods        Pods        Pods
```

This protects against a single-AZ failure.

---

# 8. VPC Design

Our target production VPC:

```text
10.0.0.0/16
```

Public:

```text
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

Conceptually:

```text
VPC 10.0.0.0/16
│
├── Public-A 10.0.1.0/24
├── Public-B 10.0.2.0/24
├── Public-C 10.0.3.0/24
│
├── Private-A 10.0.11.0/24
├── Private-B 10.0.12.0/24
└── Private-C 10.0.13.0/24
```

---

# 9. Why EKS Nodes Are Usually Private

We don't want:

```text
Internet
   │
   ▼
Worker Node
   │
   ▼
Pod
```

Instead:

```text
Internet
   │
   ▼
WAF / Load Balancer
   │
   ▼
Private EKS workload
```

Worker nodes remain inaccessible directly from the public Internet.

---

# 10. Internet Traffic

Our production request path:

```text
User
 │
 ▼
Route 53
 │
 ▼
CloudFront
 │
 ▼
AWS WAF
 │
 ▼
ALB
 │
 ▼
EKS Ingress
 │
 ▼
Kubernetes Service
 │
 ▼
Pod
```

The worker nodes remain in private subnets.

---

# 11. Private Node Internet Access

Pods sometimes need outbound Internet access.

Examples:

```text
Docker image pulls
External APIs
Package repositories
AWS endpoints
```

Private subnet traffic can use:

```text
Private subnet
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

# 12. NAT Gateway Architecture

For production resilience:

```text
AZ-A
Private subnet
    │
    ▼
NAT Gateway A

AZ-B
Private subnet
    │
    ▼
NAT Gateway B

AZ-C
Private subnet
    │
    ▼
NAT Gateway C
```

This provides AZ-local outbound paths.

The tradeoff:

> **More NAT Gateways = higher cost, but better resilience.**

For production, resilience usually matters more than minimizing NAT cost.

---

# 13. Reduce NAT Dependency

A mature AWS architecture can reduce NAT traffic by using VPC endpoints where appropriate.

Examples:

```text
EKS
 │
 ├── ECR
 ├── S3
 ├── STS
 └── CloudWatch
```

Private connectivity can reduce unnecessary Internet/NAT traversal.

This is also useful for security and cost optimization.

---

# 14. EKS Control Plane

Conceptually:

```text
                 EKS
                  │
        ┌─────────┴─────────┐
        │                   │
 Control Plane          Data Plane
        │                   │
        ▼                   ▼
 API Server              Nodes
 etcd                    Pods
 Scheduler
 Controllers
```

AWS operates the control-plane infrastructure.

Our applications run on the data plane.

---

# 15. Kubernetes API Server

The API server is the central interface.

For example:

```bash
kubectl get pods -n production
```

works through the Kubernetes API.

Architecture:

```text
Engineer
   │
   ▼
kubectl
   │
   ▼
EKS API Server
   │
   ▼
Kubernetes resources
```

---

# 16. EKS API Endpoint

EKS provides an API endpoint configuration.

Broadly, you can configure endpoint access to support:

```text
Public
Private
Public + Private
```

For production, endpoint exposure must be carefully designed.

A private endpoint can keep Kubernetes API traffic inside private networking.

A public endpoint can be useful for controlled administrative access but must be restricted appropriately.

---

# 17. Don't Expose Kubernetes API to Everyone

Avoid:

```text
0.0.0.0/0
```

for administrative Kubernetes API access unless there is a very specific, reviewed reason.

Prefer:

```text
Approved corporate network
        │
        ▼
VPN / Direct Connect / secure access
        │
        ▼
EKS API
```

or another controlled administrative access architecture.

---

# 18. Kubernetes Access

Authentication and authorization are separate concepts.

### Authentication

> Who are you?

### Authorization

> What are you allowed to do?

Architecture:

```text
Engineer
   │
   ▼
AWS Identity
   │
   ▼
EKS Authentication
   │
   ▼
Kubernetes RBAC
   │
   ▼
Allowed actions
```

---

# 19. EKS Access Entries

Modern EKS environments can use EKS access entries to manage access to the Kubernetes API.

The exact configuration should follow the current EKS access model selected by the platform team.

Conceptually:

```text
IAM Principal
      │
      ▼
EKS Access Entry
      │
      ▼
Kubernetes permissions
```

Avoid giving every engineer unrestricted cluster-admin access.

---

# 20. Cluster Admin Is Dangerous

Avoid:

```text
Developer
   ↓
cluster-admin
   ↓
Everything
```

Instead:

```text
Developer
   ↓
namespace-specific permissions

Platform Engineer
   ↓
broader cluster permissions

SRE
   ↓
operational permissions

Security
   ↓
security/audit permissions
```

Use least privilege.

---

# 21. Worker Nodes

Worker nodes are the compute machines that run application Pods.

Example:

```text
EKS
│
├── Node 1
├── Node 2
└── Node 3
```

Each node has:

```text
OS
container runtime
kubelet
networking components
storage components
```

and runs application containers.

---

# 22. Node Groups

EKS supports managed node groups.

Example:

```text
EKS Cluster
│
├── system-node-group
│
└── application-node-group
```

We might separate:

```text
system workloads
```

from:

```text
business workloads
```

---

# 23. System Nodes

Some workloads are required for the cluster itself.

Examples:

```text
CoreDNS
kube-proxy
AWS VPC CNI
monitoring agents
logging agents
```

We don't want critical platform workloads fighting for capacity with application workloads.

Therefore dedicated system capacity can be useful.

---

# 24. Application Nodes

Application nodes run:

```text
order-service
payment-service
catalog-service
inventory-service
...
```

Example:

```text
application-node-group
│
├── order-service
├── catalog-service
└── inventory-service
```

---

# 25. Instance Selection

Don't simply choose:

```text
t3.medium
```

because it is cheap.

Production node sizing depends on:

```text
Pod requests
Pod limits
CPU utilization
Memory utilization
Network bandwidth
EBS requirements
Traffic
Workload type
Scaling behavior
```

We should derive capacity from actual workloads.

---

# 26. Capacity Planning

Suppose:

```text
Node capacity:
8 CPU
32 GiB memory
```

Our workloads request:

```text
order-service:
250m CPU
512Mi

payment-service:
500m CPU
1Gi
```

We calculate how many Pods can fit while reserving capacity for:

```text
Kubernetes system
DaemonSets
eviction thresholds
operational headroom
```

You should not schedule nodes at 100% theoretical capacity.

---

# 27. DaemonSets Consume Node Capacity

This is an important production detail.

Suppose we run:

```text
Fluent Bit
Node monitoring agent
Security agent
```

as DaemonSets.

Then every node gets an instance:

```text
Node 1
├── system pods
├── Fluent Bit
├── monitoring agent
└── application pods
```

Therefore node sizing must account for DaemonSet overhead.

---

# 28. Kubelet

Every worker node has a kubelet.

Conceptually:

```text
Kubernetes API
      │
      ▼
   kubelet
      │
      ▼
Container runtime
      │
      ▼
Containers
```

The kubelet ensures Pods assigned to the node are running.

---

# 29. Container Runtime

The container runtime executes containers.

Modern Kubernetes environments use supported runtimes such as:

```text
containerd
```

Architecture:

```text
Pod specification
      │
      ▼
kubelet
      │
      ▼
containerd
      │
      ▼
Container
```

---

# 30. AWS VPC CNI

One of the most important EKS components is the AWS VPC CNI.

It provides Kubernetes Pod networking integrated with AWS VPC networking.

Conceptually:

```text
Pod
 │
 ▼
AWS VPC CNI
 │
 ▼
VPC networking
 │
 ▼
AWS network
```

This is different from simply treating Pod networking as an isolated virtual network unrelated to AWS.

---

# 31. Pod IP Addresses

Depending on the networking configuration, Pods receive IP addresses associated with the VPC networking model.

Example:

```text
Node
 │
 ├── Pod A → 10.0.x.x
 ├── Pod B → 10.0.x.x
 └── Pod C → 10.0.x.x
```

This has major implications for:

* subnet capacity
* ENIs
* IP exhaustion
* security groups
* routing

---

# 32. Subnet IP Exhaustion

A production EKS cluster can have a surprising problem:

```text
Pods cannot start
```

even though CPU and memory are available.

Why?

```text
Subnet
   ↓
No available IP addresses
```

Then:

```text
Pod
 ↓
CNI
 ↓
Unable to allocate IP
 ↓
Pod remains pending/failed
```

This is why VPC CIDR planning is extremely important.

---

# 33. EKS Networking Troubleshooting

If Pods fail to obtain networking:

Check:

```bash
kubectl get pods -n kube-system
```

Look at the AWS VPC CNI Pods.

Then inspect:

```bash
kubectl describe pod <pod-name> -n production
```

Events may indicate networking/IP allocation problems.

---

# 34. CoreDNS

Kubernetes service discovery depends heavily on DNS.

EKS runs CoreDNS Pods.

Conceptually:

```text
order-service
     │
     ▼
DNS query
     │
     ▼
CoreDNS
     │
     ▼
Service IP
```

Check:

```bash
kubectl get pods -n kube-system
```

You should see CoreDNS Pods.

---

# 35. CoreDNS Failure

Suppose:

```text
payment-service
```

cannot resolve:

```text
order-service
```

but the Service exists.

Possible causes include:

```text
CoreDNS issue
Network issue
Service configuration
DNS policy
NetworkPolicy
```

Don't immediately assume the application is broken.

---

# 36. kube-proxy

`kube-proxy` participates in implementing Kubernetes Service networking.

Conceptually:

```text
Client Pod
   │
   ▼
Service
   │
   ▼
Networking rules
   │
   ▼
Backend Pod
```

The exact implementation depends on the cluster networking configuration.

---

# 37. EKS Add-ons

A production EKS cluster commonly uses AWS-supported add-ons for important components.

Examples include:

```text
VPC CNI
CoreDNS
kube-proxy
EBS CSI driver
```

Manage them deliberately.

Don't manually install random versions into production and forget about them.

---

# 38. EBS CSI Driver

Applications may require persistent storage.

The EBS CSI driver allows Kubernetes to provision/use Amazon EBS volumes.

Architecture:

```text
Pod
 │
 ▼
PersistentVolumeClaim
 │
 ▼
StorageClass
 │
 ▼
EBS CSI Driver
 │
 ▼
Amazon EBS
```

We'll discuss persistent storage more deeply when we cover databases and stateful workloads.

---

# 39. Do We Run PostgreSQL Inside EKS?

For our enterprise architecture:

```text
Order Service
      │
      ▼
Amazon RDS PostgreSQL
```

rather than:

```text
Order Service
      │
      ▼
PostgreSQL Pod
```

For production, managed databases often reduce operational burden.

We will cover RDS separately.

---

# 40. EKS + RDS

Our architecture:

```text
                 EKS
                  │
             order-service
                  │
                  ▼
             Private network
                  │
                  ▼
            RDS PostgreSQL
```

Both should be privately reachable according to security-group and routing policy.

---

# 41. EKS + Redis

Similarly:

```text
order-service
      │
      ▼
Amazon ElastiCache / Redis
```

rather than automatically running Redis inside Kubernetes.

Again, managed services reduce the operational responsibility for certain stateful systems.

---

# 42. EKS + SQS

For asynchronous processing:

```text
order-service
      │
      ▼
SQS
      │
      ▼
consumer-service
```

The Pod uses an AWS identity rather than static AWS credentials.

---

# 43. EKS Pod Identity

Modern EKS architecture can use EKS Pod Identity for workload AWS permissions.

Conceptually:

```text
Order Pod
   │
   ▼
ServiceAccount
   │
   ▼
EKS Pod Identity Association
   │
   ▼
IAM Role
   │
   ├── SQS
   ├── Secrets Manager
   └── S3
```

No AWS access key is stored inside the Pod.

---

# 44. Example Least-Privilege Role

Order service might need:

```text
sqs:SendMessage
```

on:

```text
order-events queue
```

and:

```text
secretsmanager:GetSecretValue
```

on:

```text
specific order-service secret
```

Not:

```text
Action: *
Resource: *
```

---

# 45. Security Groups

AWS Security Groups protect network interfaces.

Conceptually:

```text
ALB
 │
 └── Security Group
          │
          ▼
       EKS workload
          │
          ▼
       RDS SG
```

Security groups should allow only necessary traffic.

For example:

```text
ALB SG
  ↓ TCP 443
EKS workload

EKS workload
  ↓ PostgreSQL 5432
RDS
```

---

# 46. NetworkPolicy vs Security Groups

These operate at different layers.

### Security Group

AWS networking boundary.

```text
AWS ENI / network interface
```

### Kubernetes NetworkPolicy

Kubernetes workload networking boundary.

```text
Pod → Pod
```

A mature architecture can use both.

---

# 47. Production Layered Security

```text
Internet
   │
   ▼
CloudFront
   │
   ▼
WAF
   │
   ▼
ALB Security Group
   │
   ▼
EKS networking
   │
   ▼
NetworkPolicy
   │
   ▼
Service
   │
   ▼
Pod
   │
   ▼
Application
```

This is defense in depth.

---

# 48. EKS Load Balancer Integration

For our architecture, AWS Load Balancer Controller can integrate Kubernetes resources with AWS load-balancing services.

Conceptually:

```text
Kubernetes
     │
     ▼
Ingress
     │
     ▼
AWS Load Balancer Controller
     │
     ▼
AWS ALB
```

The controller watches Kubernetes resources and reconciles corresponding AWS infrastructure.

We'll cover its production deployment separately.

---

# 49. Where Does the Load Balancer Controller Run?

Unlike Trivy:

```text
Trivy
↓
CI
```

the AWS Load Balancer Controller is a **cluster/platform component**.

It runs inside EKS.

Conceptually:

```text
EKS
│
├── kube-system
│    ├── CoreDNS
│    ├── VPC CNI
│    ├── kube-proxy
│    └── AWS Load Balancer Controller
│
└── production
     ├── order-service
     ├── payment-service
     └── ...
```

This is exactly the distinction you asked me to maintain.

---

# 50. GitOps Controls EKS

We don't want engineers manually installing every Kubernetes resource.

Our architecture:

```text
GitOps
   │
   ▼
ArgoCD
   │
   ▼
Kubernetes API
   │
   ▼
EKS
```

Platform components can also be managed through GitOps, Terraform, Helm, or a combination depending on ownership.

---

# 51. Terraform vs Helm vs ArgoCD

This distinction is critical.

### Terraform

Best suited to infrastructure:

```text
VPC
IAM
EKS
ECR
RDS
Redis
SQS
KMS
```

### Helm

Packages Kubernetes applications/configuration:

```text
Deployment
Service
HPA
ConfigMap
ServiceAccount
```

### ArgoCD

Deploys/synchronizes Kubernetes desired state from Git.

```text
Git
 ↓
ArgoCD
 ↓
EKS
```

---

# 52. Who Creates the EKS Cluster?

Our infrastructure pipeline:

```text
Infrastructure Git
        │
        ▼
Terraform CI
        │
        ▼
Terraform Plan
        │
        ▼
Approval
        │
        ▼
Terraform Apply
        │
        ▼
AWS
        │
        ▼
EKS Cluster
```

We should not create production EKS manually from an engineer's laptop.

---

# 53. Terraform EKS Module

Our repository:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   ├── vpc/
    │   ├── eks/
    │   ├── iam/
    │   └── ...
    │
    └── environments/
        ├── dev/
        ├── qa/
        └── prod/
```

The production EKS configuration consumes the EKS module.

---

# 54. EKS Terraform Concept

A production EKS module needs to manage concepts such as:

```text
EKS Cluster
IAM roles
Cluster networking
Node groups
Security groups
Add-ons
Access
Logging
Encryption
```

Rather than writing a giant Terraform file, we encapsulate these into modules.

---

# 55. EKS Cluster Encryption

Sensitive Kubernetes data should be protected appropriately.

EKS supports encryption options for Kubernetes API data using AWS KMS.

Conceptually:

```text
Kubernetes data
      │
      ▼
EKS encryption
      │
      ▼
AWS KMS
```

The exact resources and encryption scope should follow the organization's security requirements.

---

# 56. EKS Control Plane Logging

For production troubleshooting and auditing, enable appropriate EKS control-plane logs.

Conceptually:

```text
EKS Control Plane
       │
       ├── API logs
       ├── Audit logs
       ├── Authenticator logs
       └── Other supported control-plane logs
       │
       ▼
CloudWatch Logs
```

This gives operators visibility into control-plane activity.

---

# 57. Production Logging Architecture

Our application logs eventually flow:

```text
Application
    │
    ▼
stdout/stderr
    │
    ▼
Node log files
    │
    ▼
Fluent Bit
    │
    ▼
Logstash
    │
    ▼
Elasticsearch
    │
    ▼
Kibana
```

Control-plane logs are a separate AWS/EKS logging stream.

---

# 58. Monitoring EKS

Our monitoring architecture:

```text
EKS
 │
 ├── kube-state-metrics
 ├── node exporter
 ├── Prometheus
 ├── Grafana
 └── Alertmanager
```

We monitor:

```text
Node CPU
Node memory
Pod CPU
Pod memory
Pod restarts
Pending Pods
API server health
Cluster capacity
HPA
Deployment availability
```

---

# 59. Node Monitoring

Suppose:

```text
Node 2
CPU = 95%
```

You might see:

```text
Pods pending
```

because the node doesn't have enough capacity.

Monitoring should expose:

```text
CPU
Memory
Disk
Network
Pod density
```

---

# 60. EKS Autoscaling

There are two different scaling levels.

### Pod scaling

```text
HPA
 ↓
Pods
```

### Node scaling

```text
Karpenter / Node Groups
 ↓
EC2 capacity
```

Architecture:

```text
Traffic
  ↓
HPA
  ↓
More Pods
  ↓
Insufficient Node Capacity
  ↓
Node Autoscaling
  ↓
New EC2 Nodes
  ↓
Pending Pods scheduled
```

This relationship is extremely important.

---

# 61. Karpenter

Karpenter is a Kubernetes node provisioning solution that can dynamically provision suitable compute capacity based on pending workloads.

Conceptually:

```text
Pending Pod
    │
    ▼
Karpenter
    │
    ├── CPU requirement
    ├── Memory requirement
    ├── Architecture
    ├── Availability Zone
    ├── Capacity type
    └── constraints
    │
    ▼
EC2 capacity
```

We'll dedicate a later chapter to autoscaling.

---

# 62. Managed Node Groups vs Karpenter

### Managed Node Groups

Good for:

```text
Predictable baseline capacity
System workloads
Simple operational model
```

### Karpenter

Useful for:

```text
Dynamic capacity
Diverse instance types
Fast provisioning
Workload-driven scaling
```

A production platform may use both.

---

# 63. Production Node Architecture

A possible architecture:

```text
EKS
│
├── System Capacity
│     └── Managed Node Group
│
├── Baseline Application Capacity
│     └── Managed Node Group
│
└── Dynamic Application Capacity
      └── Karpenter
```

The actual design should be based on workload requirements.

---

# 64. EKS Upgrade

Kubernetes clusters require upgrades.

A production upgrade is not:

```bash
upgrade cluster
```

and hope.

Instead:

```text
Current version
      │
      ▼
Check AWS/EKS compatibility
      │
      ▼
Check add-ons
      │
      ▼
Check deprecated APIs
      │
      ▼
Test in DEV
      │
      ▼
QA
      │
      ▼
STAGING
      │
      ▼
Production
```

---

# 65. What Must Be Checked Before Upgrade?

At minimum:

```text
Kubernetes version
EKS add-ons
AWS VPC CNI
CoreDNS
kube-proxy
EBS CSI
Ingress controller
Admission controllers
Helm charts
Deprecated APIs
Custom resources
PodDisruptionBudgets
Node AMIs
```

---

# 66. Node Upgrade

Node upgrades should also be controlled.

Conceptually:

```text
Old Node
   │
   ▼
Cordon
   │
   ▼
Drain
   │
   ▼
Pods rescheduled
   │
   ▼
Terminate / replace
   │
   ▼
New Node
```

PDBs and workload topology determine how safely this can happen.

---

# 67. Cordon

```bash
kubectl cordon <node-name>
```

This prevents new Pods from being scheduled onto the node.

Existing Pods can continue running.

---

# 68. Drain

```bash
kubectl drain <node-name>
```

This attempts to evict eligible Pods so they can run elsewhere.

Production draining must consider:

```text
PDB
DaemonSets
local storage
stateful workloads
critical system Pods
```

Never blindly run:

```bash
kubectl drain --force --ignore-daemonsets --delete-emptydir-data
```

on a production node without understanding the consequences.

---

# 69. EKS Failure Scenario — Node Dies

Suppose:

```text
AZ-A
Node 1
   ↓
FAILURE
```

Kubernetes sees:

```text
Pods unavailable
```

If enough capacity exists:

```text
Replacement Pods
      ↓
Node 2 / Node 3
```

If capacity is insufficient:

```text
Pods
 ↓
Pending
 ↓
Node autoscaling
 ↓
New node
 ↓
Pods scheduled
```

This demonstrates why node-level and pod-level resilience work together.

---

# 70. EKS Failure Scenario — AZ Problem

Suppose:

```text
AZ-A
   ↓
Unavailable
```

Our architecture should have:

```text
AZ-B
AZ-C
```

with application replicas distributed appropriately.

Example:

```text
order-service
├── Pod A → AZ-A
├── Pod B → AZ-B
└── Pod C → AZ-C
```

Now:

```text
AZ-A failure
      ↓
Pod A lost
      ↓
Pod B + Pod C remain
```

The service can continue if capacity and dependencies remain healthy.

---

# 71. EKS Failure Scenario — Subnet Exhaustion

Suppose:

```text
Private-A
   ↓
No available IPs
```

Then new Pods may fail to obtain networking.

Symptoms:

```text
Pod pending
CNI errors
IP allocation errors
```

Troubleshooting path:

```text
Pod
 ↓
CNI
 ↓
ENI/IP allocation
 ↓
Subnet
 ↓
VPC
```

---

# 72. EKS Failure Scenario — ECR Pull Failure

Suppose a new Pod reports:

```text
ImagePullBackOff
```

Troubleshooting:

```text
Pod
 ↓
Image name
 ↓
Image tag/digest
 ↓
ECR repository
 ↓
IAM permissions
 ↓
Network
 ↓
ECR availability
```

Check:

```bash
kubectl describe pod <pod-name> -n production
```

The Events section is often the first place to look.

---

# 73. EKS Failure Scenario — CoreDNS

Application:

```text
payment-service
```

tries:

```text
order-service.production.svc.cluster.local
```

and DNS fails.

Troubleshooting:

```text
Application
 ↓
DNS query
 ↓
CoreDNS
 ↓
Service
 ↓
Endpoints
```

Check:

```bash
kubectl get pods -n kube-system
```

Then CoreDNS logs if necessary.

---

# 74. EKS Failure Scenario — API Access

Engineer says:

> "kubectl can't connect."

Don't assume EKS is down.

Check:

```text
Engineer network
 ↓
VPN / corporate network
 ↓
DNS
 ↓
EKS endpoint
 ↓
Authentication
 ↓
Authorization
```

Potential causes:

```text
VPN issue
DNS issue
endpoint access configuration
IAM authentication
EKS access entry
RBAC
```

---

# 75. EKS Production Security Checklist

```text
[ ] Private worker nodes
[ ] Restricted API access
[ ] Least-privilege IAM
[ ] EKS access controls
[ ] NetworkPolicies
[ ] Security groups
[ ] Pod security controls
[ ] Encryption
[ ] Secrets Manager integration
[ ] No static AWS keys in Pods
[ ] ECR image scanning
[ ] Image signing
[ ] Audit/control-plane logging
[ ] Vulnerability management
[ ] Regular cluster upgrades
```

---

# 76. EKS Production Observability

Monitor:

```text
CLUSTER
├── API health
├── Control-plane logs
└── Cluster events

NODES
├── CPU
├── Memory
├── Disk
├── Network
└── Capacity

PODS
├── CPU
├── Memory
├── Restarts
├── Readiness
└── OOMKilled

WORKLOADS
├── Deployment availability
├── HPA
├── PDB
└── rollout status
```

---

# 77. Daily DevOps Operational Commands

### Nodes

```bash
kubectl get nodes
```

```bash
kubectl describe node <node-name>
```

### Pods

```bash
kubectl get pods -A
```

```bash
kubectl get pods -n production
```

### Deployments

```bash
kubectl get deployments -n production
```

### Services

```bash
kubectl get svc -n production
```

### Events

```bash
kubectl get events -n production --sort-by=.lastTimestamp
```

### Rollout

```bash
kubectl rollout status deployment/order-service \
  -n production
```

---

# 78. Production Troubleshooting Method

Don't randomly execute commands.

Use layers:

```text
1. User
   ↓
2. DNS
   ↓
3. CloudFront/WAF
   ↓
4. ALB
   ↓
5. Ingress
   ↓
6. Service
   ↓
7. Pod
   ↓
8. Container
   ↓
9. Application
   ↓
10. Database / Redis / SQS
```

If the problem is:

```text
Pod Pending
```

don't start troubleshooting the database.

The failure is likely earlier in the runtime lifecycle.

---

# 79. EKS Mental Model

Remember:

```text
AWS
│
├── VPC
│
├── EKS Control Plane
│
└── Worker Capacity
      │
      ├── kubelet
      ├── container runtime
      ├── VPC CNI
      └── Pods
```

Then:

```text
Pods
 │
 ├── Services
 ├── Ingress
 ├── IAM
 ├── Config
 ├── Secrets
 └── AWS services
```

---

# 80. Complete Production EKS Architecture

```text
                              INTERNET
                                  │
                                  ▼
                              Route 53
                                  │
                                  ▼
                              CloudFront
                                  │
                                  ▼
                                WAF
                                  │
                                  ▼
                                ALB
                                  │
                                  ▼
                           EKS Ingress
                                  │
                                  ▼
                           Kubernetes
                              Service
                                  │
                 ┌────────────────┼────────────────┐
                 ▼                ▼                ▼
               Pod              Pod              Pod
                 │                │                │
                 └────────────────┼────────────────┘
                                  │
                     ┌────────────┼────────────┐
                     ▼            ▼            ▼
                    RDS         Redis          SQS
```

Infrastructure:

```text
AWS
│
└── VPC
    │
    ├── Public Subnets
    │     └── ALB/NAT-related infrastructure
    │
    └── Private Subnets
          │
          └── EKS Nodes
                │
                ├── System Pods
                └── Application Pods
```

---

# 81. Where Each Tool Runs

This directly addresses your earlier question.

| Component                    | Production location                              |
| ---------------------------- | ------------------------------------------------ |
| GitHub Actions               | GitHub runner / enterprise runner infrastructure |
| Jenkins                      | CI infrastructure                                |
| GitLab Runner                | CI infrastructure                                |
| Trivy                        | CI runner/security pipeline                      |
| Sonar Scanner                | CI runner                                        |
| SonarQube                    | Centralized platform service                     |
| Docker Build                 | CI/build environment                             |
| ECR                          | AWS managed registry                             |
| ArgoCD                       | EKS/platform namespace                           |
| Helm                         | CI/GitOps tooling; charts stored in Git          |
| AWS Load Balancer Controller | EKS                                              |
| CoreDNS                      | EKS                                              |
| VPC CNI                      | EKS nodes / system components                    |
| kube-proxy                   | EKS nodes                                        |
| EBS CSI Driver               | EKS                                              |
| Prometheus                   | EKS/platform                                     |
| Grafana                      | EKS/platform                                     |
| Fluent Bit                   | EKS nodes as DaemonSet                           |
| Logstash                     | EKS/platform                                     |
| Elasticsearch                | EKS/platform or managed alternative              |
| Kibana                       | EKS/platform                                     |
| OpenTelemetry                | EKS/platform/agents depending on architecture    |
| Jaeger                       | EKS/platform or managed alternative              |

This distinction will continue throughout the project.

---

# 82. Production Ownership

A realistic ownership model:

```text
AWS
 │
 └── Managed infrastructure
      │
      ├── EKS control plane
      ├── ECR
      └── managed services

Platform Team
 │
 ├── EKS
 ├── ArgoCD
 ├── Helm
 ├── ingress
 └── cluster add-ons

DevOps
 │
 ├── Terraform
 ├── CI/CD
 ├── ECR lifecycle
 └── release automation

Security
 │
 ├── IAM standards
 ├── vulnerability policy
 ├── image policy
 └── audit

SRE
 │
 ├── monitoring
 ├── alerting
 ├── reliability
 └── incidents

Backend Team
 │
 └── application code
```

---

# 83. What Terraform Should Provision

For our infrastructure repository:

```text
Terraform
│
├── VPC
├── Subnets
├── NAT
├── Route Tables
├── Security Groups
├── EKS
├── IAM
├── ECR
├── RDS
├── Redis
├── SQS
├── SNS
├── S3
└── KMS
```

Terraform establishes the infrastructure foundation.

---

# 84. What ArgoCD Should Deploy

ArgoCD should manage Kubernetes application/platform desired state such as:

```text
ArgoCD
│
├── order-service
├── payment-service
├── catalog-service
├── inventory-service
├── monitoring
├── logging
├── tracing
└── other Kubernetes workloads
```

The exact split between Terraform and ArgoCD for cluster add-ons should be standardized; don't have both tools fighting over the same resource.

---

# 85. Important Ownership Rule

Avoid:

```text
Terraform
   ↓
Deployment/order-service

AND

ArgoCD
   ↓
Deployment/order-service
```

Both tools managing the same Kubernetes Deployment creates ownership conflicts.

Instead:

```text
Terraform
   ↓
Infrastructure

ArgoCD
   ↓
Kubernetes application state
```

Clear ownership prevents drift and conflicts.

---

# 86. Real Production Deployment Chain

Now our entire architecture is connected:

```text
Developer
    │
    ▼
GitHub
    │
    ▼
CI
    │
    ├── Build
    ├── Test
    ├── SonarQube
    ├── Gitleaks
    ├── Trivy
    ├── SBOM
    └── Sign
    │
    ▼
ECR
    │
    ▼
GitOps
    │
    ▼
ArgoCD
    │
    ▼
EKS
    │
    ▼
Pods
    │
    ├── RDS
    ├── Redis
    ├── SQS
    └── AWS services
```

This is now our core production platform.

---

# 87. EKS Production Checklist

Before calling our EKS platform production-ready:

```text
NETWORK
[ ] Multi-AZ VPC
[ ] Private worker subnets
[ ] Public ingress architecture
[ ] NAT strategy
[ ] VPC endpoints where appropriate
[ ] IP capacity planned

EKS
[ ] Supported Kubernetes version
[ ] EKS control-plane configuration
[ ] Restricted API access
[ ] Access management
[ ] Encryption
[ ] Control-plane logging

NODES
[ ] Managed node groups / Karpenter
[ ] Appropriate instance types
[ ] Multi-AZ capacity
[ ] System capacity
[ ] Resource headroom
[ ] Upgrade strategy

ADD-ONS
[ ] VPC CNI
[ ] CoreDNS
[ ] kube-proxy
[ ] EBS CSI
[ ] Load Balancer Controller

SECURITY
[ ] IAM least privilege
[ ] Pod identity
[ ] Security groups
[ ] NetworkPolicies
[ ] Pod security controls
[ ] Secrets Manager
[ ] ECR security

OBSERVABILITY
[ ] Prometheus
[ ] Grafana
[ ] Alertmanager
[ ] Logging
[ ] Tracing
[ ] Control-plane logs

OPERATIONS
[ ] Upgrade process
[ ] Backup/recovery
[ ] Incident response
[ ] Capacity planning
[ ] Cost monitoring
[ ] Troubleshooting runbooks
```

---

# 88. The Production Mental Model

Don't think:

> "EKS is where my Docker containers run."

Think:

> **"EKS is the Kubernetes control-plane and workload platform integrated with AWS networking, IAM, load balancing, storage, and managed services."**

And always know which layer owns the problem:

```text
AWS
 ↓
VPC
 ↓
EKS
 ↓
Node
 ↓
Kubernetes
 ↓
Service
 ↓
Pod
 ↓
Container
 ↓
Application
 ↓
Database / AWS service
```

---

# Next Chapter

## `20-RDS.md`

Now we'll build the **production database architecture** for the 20+ microservices.

We'll cover why we are **not putting PostgreSQL inside an EKS Pod**, and instead use Amazon RDS:

```text
                    EKS
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
 order-service  payment-service  user-service
        │            │            │
        ▼            ▼            ▼
     PostgreSQL   PostgreSQL   PostgreSQL
        │
        ▼
      RDS
```

We'll go deeply into:

* RDS production architecture
* Multi-AZ
* subnet groups
* security groups
* PostgreSQL
* database-per-service strategy
* connection pooling
* credentials
* Secrets Manager
* backups
* point-in-time recovery
* read replicas
* storage
* encryption/KMS
* monitoring
* CloudWatch
* Performance Insights
* failover
* maintenance
* upgrades
* connection exhaustion
* slow queries
* production troubleshooting
* and exactly how an EKS Pod connects securely to RDS in a real production environment.
