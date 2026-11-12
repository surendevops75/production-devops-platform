# 98 --- Complete DevOps Connectivity and Troubleshooting Guide: Code to Production

> A production-focused runbook showing **where each tool runs, how
> engineers connect to it, what credentials are used, and how to
> troubleshoot problems from source code through Terraform, AWS, EKS,
> Kubernetes, GitOps, ArgoCD, applications, databases, messaging,
> observability, and production traffic**.

------------------------------------------------------------------------

## 1. Purpose

In a real DevOps environment, troubleshooting is not:

``` text
SSH into something
run random commands
restart things
```

The production approach is:

``` text
Developer
   |
   v
GitHub
   |
   v
CI
   |
   v
ECR
   |
   v
GitOps
   |
   v
ArgoCD
   |
   v
EKS API
   |
   v
Kubernetes
   |
   +---- Nodes
   |
   +---- Pods
   |
   +---- Services
   |
   +---- Ingress / ALB
   |
   +---- RDS
   |
   +---- Redis
   |
   +---- SQS/SNS
   |
   +---- Observability
   |
   +---- AWS infrastructure
```

At every layer, ask:

1.  **Where does this component run?**
2.  **How do I authenticate to it?**
3.  **What interface do I use to troubleshoot it?**
4.  **What logs/metrics/events exist?**
5.  **What is the next layer below it?**

------------------------------------------------------------------------

# 2. Complete Environment Map

Our project contains:

``` text
company-ecommerce/
│
├── Application repositories
│   ├── order-service
│   ├── payment-service
│   ├── catalog-service
│   └── ... 21 services
│
├── infrastructure
│   └── Terraform
│
├── gitops
│   └── ArgoCD / Helm environment configuration
│
└── platform
    └── shared platform policies/configuration
```

AWS:

``` text
AWS Account
│
├── IAM
├── KMS
├── VPC
│   ├── Public subnets
│   └── Private subnets
│
├── ECR
├── EKS
│   ├── Control plane
│   └── Data plane
│
├── RDS
├── ElastiCache
├── SQS
├── SNS
├── S3
├── Route 53
├── CloudFront
└── WAF
```

------------------------------------------------------------------------

# 3. The Most Important Concept: Control Plane vs Data Plane

## EKS Control Plane

AWS manages:

``` text
Kubernetes API Server
Scheduler
Controller Manager
etcd
```

You do **not** SSH into the AWS-managed EKS control plane.

You communicate through:

``` text
kubectl
AWS CLI
EKS APIs
CloudWatch control-plane logs
```

------------------------------------------------------------------------

## EKS Data Plane

Your workloads run here:

``` text
Managed Node Groups
Karpenter nodes
Pods
DaemonSets
Deployments
```

Node-level access can be through an approved mechanism such as AWS
Systems Manager Session Manager.

------------------------------------------------------------------------

# 4. Complete Connectivity Map

``` text
                           ENGINEER
                              |
                 +------------+------------+
                 |                         |
                 v                         v
             AWS CLI                    kubectl
                 |                         |
                 v                         v
               AWS                     EKS API
                 |                         |
        +--------+--------+               |
        |        |        |               |
       IAM      EKS      EC2              |
                 |                         |
                 +------------+------------+
                              |
                              v
                       Kubernetes API
                              |
             +----------------+----------------+
             |                |                |
             v                v                v
          Scheduler        Controller       Objects
             |                                 |
             v                                 v
           Nodes                              Pods
             |
      +------+------+
      |             |
      v             v
    Kubelet       CNI
      |
      v
 Container Runtime
      |
      v
    Container
```

------------------------------------------------------------------------

# 5. Engineer Access

A production engineer normally does not receive one giant credential.

Instead:

``` text
Engineer
   |
   +-- AWS IAM / Identity Center
   |
   +-- GitHub access
   |
   +-- Jira access
   |
   +-- Kubernetes access
   |
   +-- Observability access
```

Access should be least privilege.

------------------------------------------------------------------------

# 6. Verify AWS Identity

Before troubleshooting AWS:

``` bash
aws sts get-caller-identity
```

Expected:

``` json
{
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:role/DevOpsRole"
}
```

If this fails:

``` text
STOP
```

Do not continue with EKS commands.

First fix AWS authentication.

------------------------------------------------------------------------

# 7. Check AWS Region

``` bash
aws configure get region
```

Or explicitly:

``` bash
export AWS_REGION=ap-south-1
```

Then:

``` bash
aws sts get-caller-identity
```

------------------------------------------------------------------------

# 8. Check EKS Cluster

``` bash
aws eks describe-cluster \
  --name ecommerce-prod \
  --region ap-south-1
```

Check only the endpoint:

``` bash
aws eks describe-cluster \
  --name ecommerce-prod \
  --region ap-south-1 \
  --query 'cluster.endpoint' \
  --output text
```

Check version:

``` bash
aws eks describe-cluster \
  --name ecommerce-prod \
  --region ap-south-1 \
  --query 'cluster.version' \
  --output text
```

------------------------------------------------------------------------

# 9. Configure kubectl

``` bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name ecommerce-prod
```

Then:

``` bash
kubectl config current-context
```

Verify:

``` bash
kubectl get nodes
```

If this works:

``` text
AWS authentication
        |
        v
EKS authentication
        |
        v
Kubernetes authorization
        |
        v
Kubernetes API
```

is working.

------------------------------------------------------------------------

# 10. If kubectl Fails

Troubleshoot in this order:

``` text
1. AWS credentials
2. Correct region
3. Correct cluster
4. kubeconfig
5. Network reachability
6. EKS authentication
7. Kubernetes authorization
```

Commands:

``` bash
aws sts get-caller-identity
```

``` bash
aws eks describe-cluster \
  --name ecommerce-prod \
  --region ap-south-1
```

``` bash
kubectl config get-contexts
```

``` bash
kubectl cluster-info
```

Then:

``` bash
kubectl auth can-i get pods -A
```

------------------------------------------------------------------------

# 11. Authentication vs Authorization

These are different.

### Authentication

``` text
Who are you?
```

Examples:

``` text
AWS IAM
IAM Identity Center
OIDC
```

### Authorization

``` text
What are you allowed to do?
```

Examples:

``` text
Kubernetes RBAC
EKS access configuration
IAM policies
```

------------------------------------------------------------------------

# 12. Kubernetes RBAC Troubleshooting

Check:

``` bash
kubectl auth can-i get pods \
  -n ecommerce-prod
```

Check:

``` bash
kubectl auth can-i create deployments \
  -n ecommerce-prod
```

Check:

``` bash
kubectl auth can-i '*' '*' \
  -A
```

Do not grant broad permissions merely to make an incident disappear.

------------------------------------------------------------------------

# 13. Where Terraform Runs

Terraform usually runs from:

``` text
Developer workstation
        OR
CI/CD runner
        OR
approved automation environment
```

Terraform does not need to run inside EKS.

Architecture:

``` text
Engineer / CI
      |
      v
Terraform
      |
      v
AWS APIs
      |
      +-- VPC
      +-- IAM
      +-- EKS
      +-- ECR
      +-- RDS
      +-- S3
      +-- SQS
      +-- SNS
      +-- KMS
```

------------------------------------------------------------------------

# 14. Terraform Authentication

Verify:

``` bash
aws sts get-caller-identity
```

Terraform then uses the AWS provider.

Example:

``` hcl
provider "aws" {
  region = var.aws_region
}
```

Do not put long-lived AWS access keys into Terraform files.

------------------------------------------------------------------------

# 15. Terraform State

Our state is stored in S3.

Example:

``` hcl
terraform {
  backend "s3" {
    bucket       = "company-ecommerce-prod-terraform-state"
    key          = "prod/terraform.tfstate"
    region       = "ap-south-1"
    encrypt      = true
    use_lockfile = true
  }
}
```

------------------------------------------------------------------------

# 16. Terraform Troubleshooting Flow

``` text
terraform init
      |
      v
Backend reachable?
      |
      v
AWS authentication?
      |
      v
Provider authentication?
      |
      v
terraform validate
      |
      v
terraform plan
      |
      v
AWS API error?
      |
      v
IAM / resource / dependency
```

------------------------------------------------------------------------

# 17. Terraform Init Problem

Run:

``` bash
terraform init
```

If backend access fails:

``` text
Check:
- AWS identity
- S3 bucket
- region
- bucket permissions
- state key
- network
```

Verify:

``` bash
aws s3api head-bucket \
  --bucket company-ecommerce-prod-terraform-state
```

------------------------------------------------------------------------

# 18. Terraform Validate

``` bash
terraform fmt -check
terraform validate
```

These catch local configuration problems before AWS is contacted.

------------------------------------------------------------------------

# 19. Terraform Plan

``` bash
terraform plan
```

Always inspect:

``` text
Plan to add
Plan to change
Plan to destroy
```

A production engineer should pay special attention to:

``` text
destroy
replace
```

especially for:

``` text
RDS
EKS
KMS
S3
networking
```

------------------------------------------------------------------------

# 20. Terraform State Inspection

Use:

``` bash
terraform state list
```

Then:

``` bash
terraform state show <RESOURCE>
```

Example:

``` bash
terraform state show module.vpc.aws_vpc.this
```

Do not casually modify state with `terraform state rm`.

State manipulation is an advanced recovery operation.

------------------------------------------------------------------------

# 21. Terraform AWS Permission Failure

Example:

``` text
AccessDenied
```

Check:

``` bash
aws sts get-caller-identity
```

Then identify the IAM role.

Review:

``` text
Identity policy
Permission boundary
SCP
Resource policy
Explicit deny
```

The key question is:

> Which AWS API call did Terraform attempt, and which policy denied it?

------------------------------------------------------------------------

# 22. Terraform Dependency Failure

Example:

``` text
RDS creation failed because subnet group is wrong
```

Trace:

``` text
RDS
 |
 +-- DB subnet group
       |
       +-- VPC
       +-- private subnets
```

Check:

``` bash
aws ec2 describe-subnets
```

Then:

``` bash
aws rds describe-db-subnet-groups
```

------------------------------------------------------------------------

# 23. Terraform Drift

Terraform drift means:

``` text
Actual AWS
    !=
Terraform desired configuration
```

Run:

``` bash
terraform plan
```

If Terraform wants to change something unexpectedly:

``` text
Investigate first.
```

Do not automatically apply.

------------------------------------------------------------------------

# 24. AWS VPC Troubleshooting

Our architecture:

``` text
Internet
   |
   v
CloudFront/WAF
   |
   v
ALB
   |
   v
Private EKS nodes
   |
   +---- RDS
   +---- Redis
   +---- AWS APIs
```

Troubleshooting path:

``` text
DNS
 |
Route
 |
NACL
 |
Security Group
 |
NetworkPolicy
 |
Service
 |
Pod
 |
Application
```

------------------------------------------------------------------------

# 25. VPC Inspection

List VPCs:

``` bash
aws ec2 describe-vpcs
```

Subnets:

``` bash
aws ec2 describe-subnets
```

Route tables:

``` bash
aws ec2 describe-route-tables
```

Security groups:

``` bash
aws ec2 describe-security-groups
```

------------------------------------------------------------------------

# 26. NAT Gateway Troubleshooting

Private nodes often need outbound access.

Flow:

``` text
Private subnet
     |
     v
Route table
     |
     v
NAT Gateway
     |
     v
Internet Gateway
     |
     v
Internet
```

If Pods cannot download something:

``` text
Check:
- private route
- NAT
- NAT route
- NACL
- DNS
- security group
```

------------------------------------------------------------------------

# 27. VPC Endpoint Troubleshooting

AWS services may use VPC endpoints.

Example:

``` text
Private workload
      |
      v
VPC Endpoint
      |
      v
S3 / ECR / STS / Secrets Manager
```

If AWS API access fails from private workloads:

``` text
Check:
- endpoint exists
- endpoint policy
- route/DNS
- security groups
- IAM
```

------------------------------------------------------------------------

# 28. ECR Troubleshooting

Check repository:

``` bash
aws ecr describe-repositories \
  --repository-names order-service \
  --region ap-south-1
```

Check images:

``` bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

------------------------------------------------------------------------

# 29. Docker to ECR

Developer/CI:

``` bash
docker build \
  -t order-service:${GIT_SHA} .
```

Authenticate:

``` bash
aws ecr get-login-password \
  --region ap-south-1 |
docker login \
  --username AWS \
  --password-stdin \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com
```

Push:

``` bash
docker push \
  123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:${GIT_SHA}
```

------------------------------------------------------------------------

# 30. ECR Image Troubleshooting

If Kubernetes reports:

``` text
ImagePullBackOff
```

check:

``` bash
kubectl describe pod <POD> -n <NAMESPACE>
```

Look for:

``` text
pull access denied
repository does not exist
manifest unknown
authorization failure
```

Then verify:

``` text
ECR repository
image tag
node/pod IAM permissions
network access
region
```

------------------------------------------------------------------------

# 31. CI Authentication

Our GitHub Actions architecture:

``` text
GitHub
   |
   v
GitHub OIDC
   |
   v
AWS IAM Role
   |
   v
ECR
```

No long-lived AWS access key should be required.

Workflow permissions:

``` yaml
permissions:
  contents: read
  id-token: write
```

------------------------------------------------------------------------

# 32. CI Troubleshooting

If AWS authentication fails:

``` bash
aws sts get-caller-identity
```

inside the CI job.

Check:

``` text
OIDC provider
trust policy
repository
branch
audience
IAM role ARN
```

The trust policy should be restrictive.

------------------------------------------------------------------------

# 33. GitHub to GitOps Troubleshooting

Our preferred flow:

``` text
CI
 |
 v
ECR
 |
 v
GitHub App
 |
 v
GitOps PR
 |
 v
Merge
 |
 v
ArgoCD
```

CI should not receive production EKS administrator credentials merely to
deploy an image.

------------------------------------------------------------------------

# 34. GitOps Troubleshooting

Check repository:

``` bash
git status
git log --oneline
```

Check expected values:

``` bash
cat environments/prod/order-service/values.yaml
```

Verify:

``` yaml
image:
  repository: ...
  tag: ...
  digest: ...
```

------------------------------------------------------------------------

# 35. Helm Troubleshooting

Render locally:

``` bash
helm template order-service \
  ./helm/order-service \
  -f values-prod.yaml
```

Lint:

``` bash
helm lint ./helm/order-service \
  -f values-prod.yaml
```

Check generated resources before ArgoCD receives them.

------------------------------------------------------------------------

# 36. ArgoCD Troubleshooting

ArgoCD is installed inside EKS.

Namespace:

``` bash
kubectl get pods -n argocd
```

Applications:

``` bash
argocd app list
```

Specific application:

``` bash
argocd app get order-service-prod
```

------------------------------------------------------------------------

# 37. ArgoCD Application States

Important states:

``` text
Synced
OutOfSync
Healthy
Degraded
Missing
Unknown
```

For example:

``` text
OutOfSync + Healthy
```

means the live cluster is healthy but differs from Git desired state.

------------------------------------------------------------------------

# 38. ArgoCD OutOfSync

Check:

``` bash
argocd app diff order-service-prod
```

Possible causes:

``` text
Git changed
Manual kubectl change
Helm rendering difference
Controller mutation
Admission mutation
```

Do not immediately sync.

First understand the difference.

------------------------------------------------------------------------

# 39. ArgoCD Degraded

Check:

``` bash
argocd app get order-service-prod
```

Then:

``` bash
kubectl get pods -n ecommerce-prod
```

Then:

``` bash
kubectl describe pod <POD>
kubectl logs <POD>
```

ArgoCD may be reporting a Kubernetes workload problem rather than an
ArgoCD problem.

------------------------------------------------------------------------

# 40. Kubernetes Troubleshooting Master Flow

Always begin:

``` bash
kubectl get pods -A
```

Then:

``` bash
kubectl get nodes
```

Then:

``` bash
kubectl get events -A \
  --sort-by=.lastTimestamp
```

For a specific workload:

``` bash
kubectl describe deployment order-service \
  -n ecommerce-prod
```

``` bash
kubectl get pods \
  -n ecommerce-prod \
  -l app=order-service
```

------------------------------------------------------------------------

# 41. CrashLoopBackOff

Flow:

``` text
CrashLoopBackOff
       |
       v
describe pod
       |
       v
logs
       |
       v
previous logs
       |
       v
events
       |
       v
config/secrets/dependencies
```

Commands:

``` bash
kubectl describe pod <POD> -n ecommerce-prod
```

``` bash
kubectl logs <POD> -n ecommerce-prod
```

``` bash
kubectl logs <POD> -n ecommerce-prod --previous
```

------------------------------------------------------------------------

# 42. OOMKilled

Check:

``` bash
kubectl describe pod <POD> -n ecommerce-prod
```

Look for:

``` text
Reason: OOMKilled
```

Then:

``` bash
kubectl top pod <POD> -n ecommerce-prod
```

Check resource configuration:

``` yaml
resources:
  requests:
    cpu: "250m"
    memory: "512Mi"
  limits:
    cpu: "1"
    memory: "1Gi"
```

Then investigate whether the application actually needs more memory or
has a memory leak.

------------------------------------------------------------------------

# 43. Pending Pod

Run:

``` bash
kubectl describe pod <POD> -n ecommerce-prod
```

Look for:

``` text
FailedScheduling
```

Common causes:

``` text
Insufficient CPU
Insufficient memory
Taints
Tolerations
Node selectors
Affinity
Topology constraints
PVC
Karpenter
```

------------------------------------------------------------------------

# 44. ImagePullBackOff

Run:

``` bash
kubectl describe pod <POD> -n ecommerce-prod
```

Then check:

``` text
ECR repository
tag
digest
IAM
network
image architecture
```

Verify ECR:

``` bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

------------------------------------------------------------------------

# 45. Service Troubleshooting

Check:

``` bash
kubectl get svc -n ecommerce-prod
```

Then:

``` bash
kubectl describe svc order-service \
  -n ecommerce-prod
```

Check EndpointSlices:

``` bash
kubectl get endpointslice \
  -n ecommerce-prod
```

If there are no endpoints:

``` text
Service selector
       |
       v
Pod labels
```

are likely mismatched or Pods are not Ready.

------------------------------------------------------------------------

# 46. NetworkPolicy Troubleshooting

Check:

``` bash
kubectl get networkpolicy -A
```

If:

``` text
Pod A
```

cannot reach:

``` text
Pod B
```

check:

``` text
DNS
Service
NetworkPolicy
Pod labels
Namespace labels
Port
Application
```

Remember that a default-deny egress policy can also block DNS.

------------------------------------------------------------------------

# 47. Ingress / ALB Troubleshooting

Check:

``` bash
kubectl get ingress -A
```

Then:

``` bash
kubectl describe ingress <NAME> \
  -n <NAMESPACE>
```

Check controller:

``` bash
kubectl get deployment \
  -n kube-system \
  aws-load-balancer-controller
```

Then logs:

``` bash
kubectl logs \
  -n kube-system \
  deployment/aws-load-balancer-controller \
  --tail=200
```

------------------------------------------------------------------------

# 48. ALB Traffic Flow

``` text
Client
  |
  v
Route 53
  |
  v
CloudFront
  |
  v
WAF
  |
  v
ALB
  |
  v
Ingress
  |
  v
Service
  |
  v
Pod
```

Troubleshoot from left to right.

------------------------------------------------------------------------

# 49. DNS Troubleshooting

External DNS:

``` bash
nslookup api.example.com
```

or:

``` bash
dig api.example.com
```

Inside Kubernetes:

``` bash
kubectl run dns-test \
  --image=busybox:1.36 \
  --rm -it \
  --restart=Never \
  -- nslookup order-service.ecommerce-prod.svc.cluster.local
```

------------------------------------------------------------------------

# 50. CoreDNS Troubleshooting

``` bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=kube-dns
```

Logs:

``` bash
kubectl logs \
  -n kube-system \
  -l k8s-app=kube-dns \
  --tail=100
```

------------------------------------------------------------------------

# 51. Node Troubleshooting

First:

``` bash
kubectl get nodes
```

Then:

``` bash
kubectl describe node <NODE>
```

Check:

``` text
Ready
MemoryPressure
DiskPressure
PIDPressure
NetworkUnavailable
```

Then:

``` bash
kubectl top node <NODE>
```

------------------------------------------------------------------------

# 52. Worker Node Host-Level Troubleshooting

If the Kubernetes layer isn't enough, use an approved node access
mechanism.

For EC2 nodes, Systems Manager Session Manager is often preferred over
opening SSH.

Conceptually:

``` text
Engineer
   |
   v
AWS IAM
   |
   v
SSM
   |
   v
Private EC2 node
```

Host checks may include:

``` bash
uptime
df -h
free -m
top
ps aux
journalctl
```

Use node-level access carefully and do not make manual host changes that
fight the immutable infrastructure model.

------------------------------------------------------------------------

# 53. Kubelet Troubleshooting

On a worker node:

``` text
kubelet
```

is responsible for managing Pods assigned to that node.

If host access is authorized:

``` bash
systemctl status kubelet
```

and:

``` bash
journalctl -u kubelet --since "30 minutes ago"
```

The exact host OS and service layout depends on the EKS node image.

------------------------------------------------------------------------

# 54. Container Runtime Troubleshooting

Modern EKS worker nodes use a container runtime appropriate to the
supported node image.

If host-level investigation is required, inspect the runtime/service
according to the node image documentation.

Do not assume every EKS node has the same runtime commands.

------------------------------------------------------------------------

# 55. VPC CNI Troubleshooting

Check:

``` bash
kubectl get daemonset aws-node \
  -n kube-system
```

Pods:

``` bash
kubectl get pods \
  -n kube-system \
  -l k8s-app=aws-node
```

Logs:

``` bash
kubectl logs \
  -n kube-system \
  -l k8s-app=aws-node \
  --tail=200
```

Common symptoms:

``` text
No Pod IP
Network unavailable
Pod sandbox creation failure
```

------------------------------------------------------------------------

# 56. EBS CSI Troubleshooting

Check:

``` bash
kubectl get pods \
  -n kube-system | grep ebs
```

Then:

``` bash
kubectl get pvc -A
```

``` bash
kubectl describe pvc <PVC> \
  -n <NAMESPACE>
```

Then:

``` bash
kubectl get pv
```

Investigate:

``` text
IAM
EBS volume
AZ
StorageClass
CSI driver
node
```

------------------------------------------------------------------------

# 57. Karpenter Troubleshooting

Check:

``` bash
kubectl get nodepool
```

``` bash
kubectl get nodeclass
```

Then:

``` bash
kubectl get pods -A | grep -i karpenter
```

For Pending workloads:

``` bash
kubectl describe pod <POD> -n <NAMESPACE>
```

Check:

``` text
instance type
capacity type
AZ
architecture
taints
tolerations
subnets
security groups
IAM
```

------------------------------------------------------------------------

# 58. HPA Troubleshooting

``` bash
kubectl get hpa -A
```

Then:

``` bash
kubectl describe hpa order-service \
  -n ecommerce-prod
```

Check:

``` text
current replicas
desired replicas
CPU
memory
custom metric
target
```

Then:

``` bash
kubectl top pods -n ecommerce-prod
```

If metrics are unavailable:

``` bash
kubectl get deployment metrics-server \
  -n kube-system
```

------------------------------------------------------------------------

# 59. Database Troubleshooting

For RDS:

``` text
Pod
 |
 +-- DNS
 |
 +-- Route
 |
 +-- NetworkPolicy
 |
 +-- Security Group
 |
 +-- RDS
 |
 +-- PostgreSQL
```

Check RDS:

``` bash
aws rds describe-db-instances \
  --region ap-south-1
```

Application logs:

``` bash
kubectl logs deployment/order-service \
  -n ecommerce-prod
```

------------------------------------------------------------------------

# 60. RDS Security Group

Never troubleshoot database access by immediately opening:

``` text
0.0.0.0/0
```

Instead verify:

``` text
approved workload/node security boundary
       |
       v
RDS security group
       |
       v
TCP 5432
```

------------------------------------------------------------------------

# 61. Database Connection Pool

If:

``` text
20 Pods
```

and each has:

``` text
20 DB connections
```

potential maximum:

``` text
20 × 20 = 400
```

connections.

Therefore high DB connections may be an application
scaling/configuration issue rather than an RDS failure.

------------------------------------------------------------------------

# 62. Redis Troubleshooting

Flow:

``` text
Pod
 |
 v
DNS
 |
 v
ElastiCache endpoint
 |
 v
Security Group
 |
 v
Redis
```

Investigate:

``` text
connection timeout
authentication
TLS
memory
evictions
connection count
latency
```

------------------------------------------------------------------------

# 63. SQS Troubleshooting

Application:

``` text
order-service
    |
    v
SQS
```

Check AWS-side metrics such as:

``` text
ApproximateNumberOfMessagesVisible
ApproximateNumberOfMessagesNotVisible
```

Investigate:

``` text
IAM
queue URL
region
visibility timeout
consumer health
DLQ
```

------------------------------------------------------------------------

# 64. Secrets Troubleshooting

Our architecture:

``` text
AWS Secrets Manager
        |
        v
External Secrets Operator
        |
        v
Kubernetes Secret
        |
        v
Pod
```

Check:

``` bash
kubectl get externalsecret -A
```

Then:

``` bash
kubectl describe externalsecret \
  <NAME> \
  -n <NAMESPACE>
```

Do not print secret values.

------------------------------------------------------------------------

# 65. Prometheus Troubleshooting

Check:

``` bash
kubectl get pods -n monitoring
```

Verify targets in Prometheus.

If a service's metrics disappeared:

``` text
Application endpoint
       |
       v
Service / PodMonitor / ServiceMonitor
       |
       v
Prometheus discovery
       |
       v
Scrape
```

Check the ServiceMonitor:

``` bash
kubectl get servicemonitor -A
```

------------------------------------------------------------------------

# 66. Grafana Troubleshooting

Grafana usually consumes data from:

``` text
Prometheus
Elasticsearch
CloudWatch
other approved data sources
```

If a dashboard is empty:

``` text
Check datasource
       |
       v
Query
       |
       v
Prometheus/Elasticsearch
       |
       v
Metric/log existence
```

Don't assume Grafana is the source of the problem.

------------------------------------------------------------------------

# 67. Logging Troubleshooting

Our flow:

``` text
Application
   |
stdout/stderr
   |
container log
   |
Fluent Bit
   |
Logstash
   |
Elasticsearch
   |
Kibana
```

If logs are missing:

``` text
1. Is application writing logs?
2. Does node contain container logs?
3. Is Fluent Bit running?
4. Is Fluent Bit parsing?
5. Is Logstash receiving?
6. Is Elasticsearch healthy?
7. Is Kibana querying correct data?
```

------------------------------------------------------------------------

# 68. Tracing Troubleshooting

Flow:

``` text
Application
   |
OpenTelemetry SDK/agent
   |
OTel Collector
   |
Jaeger
```

If traces are missing:

``` text
Application instrumentation
       |
OTLP endpoint
       |
Collector
       |
Exporter
       |
Jaeger
```

Check each hop.

------------------------------------------------------------------------

# 69. Security Troubleshooting

When access is denied, determine which layer denied it.

``` text
IAM
 |
EKS authentication
 |
RBAC
 |
NetworkPolicy
 |
Security Group
 |
NACL
 |
Application authentication
 |
Application authorization
```

Do not change all layers simultaneously.

------------------------------------------------------------------------

# 70. CloudTrail

Use CloudTrail to investigate AWS API activity.

Useful questions:

``` text
Who changed the resource?
When?
Which API?
Which identity?
Was it successful?
```

This is especially useful for:

``` text
IAM
EKS
Security Groups
RDS
S3
KMS
```

------------------------------------------------------------------------

# 71. VPC Flow Logs

For network investigation:

``` text
Source
 |
v
VPC Flow Logs
 |
v
Destination
```

Useful when you need evidence around network traffic being accepted or
rejected.

Flow logs complement, but do not replace, Kubernetes and security-group
investigation.

------------------------------------------------------------------------

# 72. Incident Troubleshooting Order

Use this sequence:

``` text
1. What changed?
2. What is the blast radius?
3. Is the problem application or platform?
4. Is the Pod healthy?
5. Is the Service healthy?
6. Is DNS healthy?
7. Is network connectivity healthy?
8. Is the node healthy?
9. Is AWS infrastructure healthy?
10. Is a dependency unhealthy?
11. What is the safest mitigation?
12. How do we verify recovery?
```

------------------------------------------------------------------------

# 73. Production Troubleshooting Decision Tree

``` text
                 INCIDENT
                     |
                     v
               What changed?
                     |
                     v
              Check blast radius
                     |
        +------------+------------+
        |                         |
        v                         v
   One service               Many services
        |                         |
        v                         v
     Pod?                    Cluster?
        |                         |
   +----+----+               +----+----+
   |         |               |         |
 Healthy   Unhealthy        Nodes     AWS
   |         |               |         |
 Service    Logs           CNI/DNS   VPC/EKS
   |         |
 Network    Config
   |
 Dependency
```

------------------------------------------------------------------------

# 74. Golden Commands

## AWS

``` bash
aws sts get-caller-identity
aws eks describe-cluster --name ecommerce-prod --region ap-south-1
aws eks list-nodegroups --cluster-name ecommerce-prod --region ap-south-1
aws ecr describe-repositories --region ap-south-1
aws rds describe-db-instances --region ap-south-1
```

## Kubernetes

``` bash
kubectl get nodes
kubectl get pods -A
kubectl get events -A --sort-by=.lastTimestamp
kubectl describe pod <POD> -n <NS>
kubectl logs <POD> -n <NS>
kubectl logs <POD> -n <NS> --previous
kubectl describe node <NODE>
kubectl top nodes
kubectl top pods -A
```

## Networking

``` bash
kubectl get svc -A
kubectl get endpointslice -A
kubectl get ingress -A
kubectl get networkpolicy -A
```

## GitOps

``` bash
argocd app list
argocd app get <APP>
argocd app diff <APP>
```

## Terraform

``` bash
terraform fmt
terraform validate
terraform plan
terraform state list
terraform state show <RESOURCE>
```

------------------------------------------------------------------------

# 75. Never Run These Blindly in Production

Be extremely careful with:

``` bash
terraform apply
terraform destroy
kubectl delete pod
kubectl delete deployment
kubectl delete namespace
kubectl delete pvc
kubectl rollout restart
kubectl scale
aws rds delete-db-instance
aws eks delete-cluster
```

The command itself may be valid.

The question is:

> **Is this the correct mitigation for the current incident?**

------------------------------------------------------------------------

# 76. Production Change Boundary

For normal production changes:

``` text
Jira Change
    |
    v
GitHub PR
    |
    v
CI
    |
    v
ECR
    |
    v
GitOps PR
    |
    v
ArgoCD
    |
    v
EKS
```

For infrastructure:

``` text
Jira Change
    |
    v
Terraform PR
    |
    v
Terraform plan
    |
    v
Approval
    |
    v
Terraform apply
    |
    v
AWS
```

------------------------------------------------------------------------

# 77. Who Connects to What?

  ----------------------------------------------------------------------------
  Component         Where it runs      How we connect    Primary
                                                         troubleshooting
  ----------------- ------------------ ----------------- ---------------------
  GitHub            SaaS               Browser/Git/CLI   PR/actions/repo

  Terraform         Laptop/CI          AWS APIs          plan/state/provider

  AWS IAM           AWS                AWS CLI/Console   policies/access

  VPC               AWS                AWS CLI/Console   routes/SG/NACL

  EKS control plane AWS managed        Kubernetes        kubectl/EKS logs
                                       API/EKS API       

  Worker node       EC2                kubectl + SSM if  node/kubelet/OS
                                       needed            

  ArgoCD            EKS Pods           kubectl/argocd    app/controller/repo

  Helm              Laptop/CI/GitOps   Kubernetes API    template/lint
                    tooling                              

  Application Pods  EKS                kubectl           logs/exec/events

  RDS               AWS                application DB    DB metrics/logs
                                       protocol/AWS API  

  Redis             AWS                Redis             connectivity/cache
                                       protocol/AWS API  

  SQS               AWS                AWS SDK/CLI       queue metrics/DLQ

  Prometheus        EKS                Web/API/kubectl   targets/rules

  Grafana           EKS                Browser/API       datasource/query

  Fluent Bit        EKS nodes          kubectl/log       DaemonSet/logs
                                       pipeline          

  Elasticsearch     EKS/managed        API/Kibana        cluster/shards

  Jaeger            EKS/managed        UI/API            traces/collector

  CloudTrail        AWS                Console/API       audit events

  CloudWatch        AWS                Console/API       AWS metrics/logs
  ----------------------------------------------------------------------------

------------------------------------------------------------------------

# 78. The Complete Code-to-Production Path

This is the architecture you should memorize conceptually:

``` text
Developer
   |
   | git push
   v
GitHub
   |
   | Pull Request
   v
CI
   |
   +-- Maven tests
   +-- SonarQube
   +-- Gitleaks
   +-- Dependency scan
   +-- Trivy
   +-- Docker build
   |
   v
ECR
   |
   | immutable image digest
   v
GitOps PR
   |
   v
ArgoCD
   |
   | Kubernetes API
   v
EKS Control Plane
   |
   | scheduler/controller
   v
Worker Nodes
   |
   v
Application Pods
   |
   +---- RDS
   +---- Redis
   +---- SQS/SNS
   +---- S3
   +---- Secrets Manager
```

------------------------------------------------------------------------

# 79. Complete Troubleshooting Path

When production fails:

``` text
USER REPORTS FAILURE
        |
        v
CloudFront/WAF/ALB?
        |
        v
Ingress?
        |
        v
Service?
        |
        v
Pod?
        |
        +---- logs
        +---- events
        +---- probes
        +---- resources
        |
        v
Node?
        |
        +---- CPU
        +---- memory
        +---- disk
        +---- kubelet
        +---- CNI
        |
        v
Network?
        |
        +---- DNS
        +---- route
        +---- SG
        +---- NACL
        +---- NetworkPolicy
        |
        v
Dependency?
        |
        +---- RDS
        +---- Redis
        +---- SQS
        +---- S3
        |
        v
AWS?
        |
        +---- EKS
        +---- EC2
        +---- VPC
        +---- IAM
        |
        v
RECOVER
        |
        v
VERIFY
        |
        v
MONITOR
        |
        v
DOCUMENT
```

------------------------------------------------------------------------

# 80. Example: Complete Order-Service Incident

Suppose users report:

``` text
"Cannot create orders."
```

### Step 1 --- Check application

``` bash
kubectl get pods \
  -n ecommerce-prod \
  -l app=order-service
```

### Step 2 --- Check logs

``` bash
kubectl logs \
  -n ecommerce-prod \
  deployment/order-service \
  --tail=200
```

Suppose:

``` text
connection timeout to PostgreSQL
```

Now don't restart the Pod immediately.

------------------------------------------------------------------------

## Step 3 --- Test DNS

``` bash
kubectl exec -it <POD> \
  -n ecommerce-prod \
  -- nslookup <RDS_ENDPOINT>
```

If DNS fails:

``` text
CoreDNS/network problem
```

If DNS succeeds:

``` text
continue
```

------------------------------------------------------------------------

## Step 4 --- Network

Check:

``` text
NetworkPolicy
Security Group
route
NACL
```

------------------------------------------------------------------------

## Step 5 --- RDS

Check:

``` bash
aws rds describe-db-instances \
  --region ap-south-1
```

Then RDS metrics/health.

------------------------------------------------------------------------

## Step 6 --- Credentials

Check:

``` text
Secrets Manager
ExternalSecret
Kubernetes Secret
```

Do not print the secret.

------------------------------------------------------------------------

## Step 7 --- Connection Pool

If RDS is healthy but connections are exhausted:

``` text
Pods × connection pool
```

may be too high.

------------------------------------------------------------------------

## Step 8 --- Mitigation

Depending on evidence:

``` text
rollback application
reduce load
scale database appropriately
correct configuration
restore dependency
```

Then verify:

``` bash
curl ...
```

and business smoke tests.

------------------------------------------------------------------------

# 81. Practical Rule: Troubleshoot From the Outside In

For an HTTP production incident:

``` text
Internet
 ↓
DNS
 ↓
CloudFront
 ↓
WAF
 ↓
ALB
 ↓
Ingress
 ↓
Service
 ↓
Pod
 ↓
Application
 ↓
Database / Redis / Messaging
```

For an infrastructure incident:

``` text
Terraform
 ↓
AWS IAM
 ↓
VPC
 ↓
EKS
 ↓
Node
 ↓
Kubernetes
 ↓
Pod
```

For a GitOps incident:

``` text
Git
 ↓
GitOps PR
 ↓
ArgoCD
 ↓
Kubernetes API
 ↓
Resource
 ↓
Pod
```

------------------------------------------------------------------------

# 82. Final Mental Model

You should now be able to answer:

### "Where do I connect?"

``` text
GitHub
 → Git/Browser/CLI

Terraform
 → AWS APIs

EKS control plane
 → kubectl/EKS API

Worker node
 → kubectl first, SSM if host access is required

ArgoCD
 → kubectl/argocd CLI

Pod
 → kubectl logs/describe/exec

RDS
 → AWS APIs + PostgreSQL connectivity

Redis
 → AWS APIs + Redis connectivity

SQS
 → AWS APIs/SDK

Prometheus
 → Prometheus UI/API/kubectl

Grafana
 → Browser/API

Logs
 → Kibana/Elasticsearch/Logstash/Fluent Bit

AWS audit
 → CloudTrail

AWS metrics/logs
 → CloudWatch
```

------------------------------------------------------------------------

# 83. Final Production Architecture

``` text
                           DEVELOPER
                               |
                               v
                            GitHub
                               |
                    +----------+----------+
                    |                     |
                    v                     v
                   CI                 Pull Request
                    |
       +------------+-------------+
       |            |             |
     Tests       Security      Docker
       |            |             |
       +------------+-------------+
                    |
                    v
                   ECR
                    |
                 Image Digest
                    |
                    v
                 GitOps PR
                    |
                    v
                  ArgoCD
                    |
                    v
             Kubernetes API
                    |
           +--------+--------+
           |                 |
           v                 v
      Scheduler         Controllers
           |                 |
           +--------+--------+
                    |
                    v
              EKS Nodes
                    |
        +-----------+-----------+
        |           |           |
       Pod         Pod         Pod
        |           |           |
        +-----------+-----------+
                    |
        +-----------+-----------+
        |           |           |
       RDS        Redis       SQS/SNS
        |
        v
      Business Data


Troubleshooting access:

Engineer
   |
   +---- AWS CLI ---------> AWS APIs
   |
   +---- kubectl ----------> EKS API
   |                            |
   |                            +--> Pods
   |                            +--> Services
   |                            +--> Nodes
   |                            +--> Events
   |
   +---- argocd CLI --------> ArgoCD
   |
   +---- terraform ---------> AWS APIs / State
   |
   +---- SSM ---------------> Worker node
   |
   +---- Grafana -----------> Metrics
   |
   +---- Kibana ------------> Logs
   |
   +---- CloudTrail --------> AWS audit
```

------------------------------------------------------------------------

# 84. Golden Rule for Real Production Troubleshooting

Never start with:

``` text
"What command should I run?"
```

Start with:

``` text
"Which layer is failing?"
```

Then determine:

``` text
Who owns that layer?
Where does it run?
How do I authenticate?
What interface exposes its state?
What evidence do I have?
What dependency is below it?
```

That is the difference between **running DevOps commands** and actually
**operating a production platform**.

------------------------------------------------------------------------

# 85. One-Page Incident Cheat Sheet

``` text
========================================================
              PRODUCTION TROUBLESHOOTING
========================================================

1. IDENTIFY
   What changed?
   What is the blast radius?

2. AWS
   aws sts get-caller-identity
   aws eks describe-cluster ...

3. KUBERNETES
   kubectl get nodes
   kubectl get pods -A
   kubectl get events -A --sort-by=.lastTimestamp

4. POD
   kubectl describe pod <pod> -n <ns>
   kubectl logs <pod> -n <ns>
   kubectl logs <pod> -n <ns> --previous

5. SERVICE
   kubectl get svc -n <ns>
   kubectl get endpointslice -n <ns>

6. NETWORK
   DNS
   NetworkPolicy
   Security Group
   NACL
   Routes

7. NODE
   kubectl describe node <node>
   kubectl top node <node>
   SSM if host investigation is required

8. DEPENDENCIES
   RDS
   Redis
   SQS/SNS
   S3
   Secrets Manager

9. GITOPS
   argocd app get <app>
   argocd app diff <app>

10. TERRAFORM
    terraform validate
    terraform plan
    terraform state list

11. OBSERVABILITY
    Prometheus
    Grafana
    Kibana
    CloudWatch
    CloudTrail

12. MITIGATE
    Rollback / scale / repair / failover

13. VERIFY
    Smoke test
    Metrics
    Logs
    Traces

14. DOCUMENT
    Incident
    Timeline
    Root cause
    Corrective action
========================================================
```

------------------------------------------------------------------------

# 86. Final Architecture Principle

The entire project can be understood as four control paths:

``` text
             CODE PATH
Developer
   ↓
GitHub
   ↓
CI
   ↓
ECR


        INFRASTRUCTURE PATH
Terraform
   ↓
AWS
   ↓
VPC / IAM / EKS / RDS / etc.


             DEPLOYMENT PATH
GitOps
   ↓
ArgoCD
   ↓
Kubernetes API
   ↓
EKS workloads


         OBSERVABILITY PATH
Applications
   ↓
Metrics / Logs / Traces
   ↓
Prometheus / ELK / Jaeger
   ↓
Grafana / Kibana
   ↓
Engineer
```

And the engineer connects to each layer through the **appropriate
interface**, rather than trying to SSH into everything.

------------------------------------------------------------------------

## Recommended next chapter

After this, the logical next chapter is:

**`99-EKS-Multi-Region-Disaster-Recovery.md`**

That chapter can build directly on this connectivity model and show
exactly how the same engineer operates **Primary EKS + DR EKS**,
including failover, ArgoCD multi-cluster, Route 53, RDS cross-region
recovery, ECR replication, Secrets Manager, and failback.
