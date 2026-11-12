# 66 — Order Service EKS Deployment

## 1. Objective

This chapter performs the **first complete production-style deployment** of `order-service`.

Until now, we implemented the individual building blocks:

```text
GitHub Repository
Terraform
VPC
IAM
ECR
EKS
EKS Add-ons
GitHub Actions
Helm
GitOps
ArgoCD
```

Now we connect them.

The target architecture is:

```text
                         Internet
                            │
                            ▼
                       Route 53
                            │
                            ▼
                    AWS Load Balancer
                            │
                            ▼
                    Kubernetes Ingress
                            │
                            ▼
                    ClusterIP Service
                            │
                            ▼
                    Order Service Pods
                            │
                            ▼
                     RDS PostgreSQL
```

The deployment pipeline is:

```text
Developer
    │
    ▼
GitHub
    │
    ▼
GitHub Actions
    │
    ├── Build
    ├── Test
    ├── SonarQube
    ├── Security Scan
    ├── Docker Build
    ├── Trivy
    └── ECR Push
            │
            ▼
           ECR
            │
            ▼
        GitOps PR
            │
            ▼
       GitOps main
            │
            ▼
          ArgoCD
            │
            ▼
           EKS
```

---

# 2. What We Are Deploying

The first production-style workload is:

```text
order-service
```

Technology:

```text
Java 21
Spring Boot
Maven
Docker
Helm
Amazon ECR
Amazon EKS
PostgreSQL
Amazon RDS
```

Application port:

```text
8080
```

Health endpoints:

```text
/actuator/health
/actuator/health/readiness
/actuator/health/liveness
/actuator/prometheus
```

---

# 3. Final Kubernetes Architecture

The order service will contain:

```text
ecommerce-dev namespace
│
├── Deployment
│   └── order-service
│       ├── Pod
│       └── Pod
│
├── Service
│   └── order-service
│
├── ServiceAccount
│   └── order-service
│
├── ConfigMap
│
├── HPA
│
├── PDB
│
└── Ingress
    └── AWS ALB
```

Database:

```text
RDS PostgreSQL
```

The database is **not inside the order-service Pod**.

---

# 4. Important Separation

Our architecture intentionally separates:

```text
Application
   │
   └── Kubernetes

Database
   │
   └── Amazon RDS

Container image
   │
   └── Amazon ECR

Deployment configuration
   │
   └── GitOps

Deployment controller
   │
   └── ArgoCD
```

This prevents one component from becoming responsible for everything.

---

# 5. Pre-Deployment Checklist

Before deploying, verify the EKS cluster.

```bash
aws eks update-kubeconfig \
  --region ap-south-1 \
  --name ecommerce-dev
```

Check:

```bash
kubectl get nodes
```

Expected:

```text
NAME                          STATUS
ip-10-0-11-xx.ec2.internal    Ready
ip-10-0-12-xx.ec2.internal    Ready
ip-10-0-13-xx.ec2.internal    Ready
```

---

# 6. Verify EKS Add-ons

Run:

```bash
kubectl get pods -n kube-system
```

You should have the components required by the cluster, including the relevant:

```text
aws-node
coredns
kube-proxy
ebs-csi
aws-load-balancer-controller
```

Check the Load Balancer Controller:

```bash
kubectl get deployment \
  -n kube-system \
  aws-load-balancer-controller
```

Then:

```bash
kubectl get pods \
  -n kube-system \
  -l app.kubernetes.io/name=aws-load-balancer-controller
```

The Pods should eventually be:

```text
Running
```

---

# 7. Verify ArgoCD

```bash
kubectl get pods -n argocd
```

Then:

```bash
argocd app list
```

At this stage, the application may not exist yet.

That's fine.

---

# 8. Verify ECR

Check the repository:

```bash
aws ecr describe-repositories \
  --repository-names order-service \
  --region ap-south-1
```

Check images:

```bash
aws ecr list-images \
  --repository-name order-service \
  --region ap-south-1
```

If the repository is empty, we first need to build and push the application.

---

# 9. Build the Application Locally

From:

```text
order-service/
```

run:

```bash
./mvnw clean test
```

Expected:

```text
BUILD SUCCESS
```

Then:

```bash
./mvnw clean package
```

Expected:

```text
target/order-service-*.jar
```

---

# 10. Build Docker Image

Use an immutable tag.

For example:

```bash
export IMAGE_TAG=$(git rev-parse --short HEAD)
```

Check:

```bash
echo "$IMAGE_TAG"
```

Example:

```text
8f31a2c
```

Build:

```bash
docker build \
  -t order-service:${IMAGE_TAG} \
  .
```

Check:

```bash
docker images | grep order-service
```

---

# 11. Run Trivy Locally

Before pushing:

```bash
trivy image order-service:${IMAGE_TAG}
```

For a CI-style check:

```bash
trivy image \
  --severity HIGH,CRITICAL \
  --exit-code 1 \
  order-service:${IMAGE_TAG}
```

This gives us:

```text
Developer machine
       │
       ▼
Docker image
       │
       ▼
Trivy
       │
       ▼
ECR
```

The CI pipeline will perform the authoritative scan according to the organization's policy.

---

# 12. Authenticate to ECR

Get the ECR registry:

```bash
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)
```

Then:

```bash
export AWS_REGION=ap-south-1
```

Login:

```bash
aws ecr get-login-password \
  --region "$AWS_REGION" \
  | docker login \
      --username AWS \
      --password-stdin \
      "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
```

Expected:

```text
Login Succeeded
```

---

# 13. Tag the Image

```bash
docker tag \
  order-service:${IMAGE_TAG} \
  ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/order-service:${IMAGE_TAG}
```

Check:

```bash
docker images | grep order-service
```

---

# 14. Push to ECR

```bash
docker push \
  ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/order-service:${IMAGE_TAG}
```

Expected output will show layers being uploaded.

Verify:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region "$AWS_REGION"
```

You should see:

```text
imageTag: 8f31a2c
```

---

# 15. Record the Image Digest

The digest is important for traceability.

Run:

```bash
aws ecr describe-images \
  --repository-name order-service \
  --image-ids imageTag="${IMAGE_TAG}" \
  --region "$AWS_REGION" \
  --query 'imageDetails[0].imageDigest' \
  --output text
```

Example:

```text
sha256:abc123...
```

Now we have:

```text
Git commit:
8f31a2c

ECR image:
order-service:8f31a2c

Image digest:
sha256:...
```

This establishes artifact traceability.

---

# 16. Update GitOps

Now modify:

```text
gitops/environments/dev/order-service/values.yaml
```

Change:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "REPLACE_ME"
```

to:

```yaml
image:
  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
  tag: "8f31a2c"
```

Use the actual account ID and actual image tag.

---

# 17. Verify the Git Diff

```bash
git diff
```

You should see only the expected change:

```diff
 image:
   repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service
-  tag: "REPLACE_ME"
+  tag: "8f31a2c"
```

This is an important habit.

Never blindly commit generated deployment changes without checking:

```bash
git diff
```

---

# 18. Commit the GitOps Change

```bash
git add environments/dev/order-service/values.yaml
```

Then:

```bash
git commit \
  -m "deploy(order-service): promote 8f31a2c to dev"
```

Push:

```bash
git push origin main
```

For a stricter GitOps workflow, use a promotion branch and PR instead:

```bash
git checkout -b promote/order-service-8f31a2c
```

then open a PR.

---

# 19. ArgoCD Detects the Change

ArgoCD continuously watches the configured Git repositories.

Conceptually:

```text
GitOps
  │
  │ image changed
  ▼
ArgoCD
  │
  ▼
OutOfSync
```

Check:

```bash
argocd app get order-service-dev
```

You may see:

```text
Sync Status: OutOfSync
```

This means:

```text
Git desired state != cluster state
```

---

# 20. Inspect the Difference

Run:

```bash
argocd app diff order-service-dev
```

You should see a change similar to:

```diff
- image: .../order-service:old-version
+ image: .../order-service:8f31a2c
```

This is one of the most useful ArgoCD commands.

---

# 21. Sync the Application

For the first deployment:

```bash
argocd app sync order-service-dev
```

ArgoCD performs:

```text
Git
 │
 ▼
Helm chart
 │
 +
 ▼
Environment values
 │
 ▼
Rendered Kubernetes manifests
 │
 ▼
Kubernetes API
 │
 ▼
EKS
```

---

# 22. Wait for Deployment

```bash
argocd app wait order-service-dev
```

Then:

```bash
argocd app get order-service-dev
```

Target:

```text
Sync Status: Synced
Health Status: Healthy
```

---

# 23. Verify Kubernetes Resources

```bash
kubectl get all \
  -n ecommerce-dev
```

Expected:

```text
NAME                                  READY   STATUS
pod/order-service-xxxxxxxxx           1/1     Running
pod/order-service-yyyyyyyyy           1/1     Running

NAME                     TYPE        CLUSTER-IP
service/order-service    ClusterIP   10.x.x.x

NAME                            READY
deployment.apps/order-service   2/2

NAME                                       DESIRED
replicaset.apps/order-service-xxxxxxxx    2
```

---

# 24. Verify the Exact Image

```bash
kubectl get deployment order-service \
  -n ecommerce-dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Expected:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f31a2c
```

This confirms:

```text
GitOps
   ↓
ArgoCD
   ↓
Deployment
   ↓
Correct ECR image
```

---

# 25. Verify Pod Image

```bash
kubectl get pods \
  -n ecommerce-dev \
  -o wide
```

Then:

```bash
kubectl describe pod <pod-name> \
  -n ecommerce-dev
```

Look for:

```text
Image:
Image ID:
```

The `Image ID` is particularly useful because it lets you verify the actual pulled image digest.

---

# 26. Verify Application Logs

```bash
kubectl logs \
  deployment/order-service \
  -n ecommerce-dev
```

For live logs:

```bash
kubectl logs \
  -f deployment/order-service \
  -n ecommerce-dev
```

You should see Spring Boot startup information.

If there are errors, check:

```bash
kubectl logs \
  deployment/order-service \
  -n ecommerce-dev \
  --all-containers=true
```

---

# 27. Verify Health Endpoint

Find a Pod:

```bash
kubectl get pods -n ecommerce-dev
```

Then:

```bash
kubectl exec -it <pod-name> \
  -n ecommerce-dev \
  -- curl -s \
  http://localhost:8080/actuator/health
```

Expected:

```json
{
  "status": "UP"
}
```

---

# 28. Verify Readiness

```bash
kubectl exec -it <pod-name> \
  -n ecommerce-dev \
  -- curl -s \
  http://localhost:8080/actuator/health/readiness
```

Expected:

```json
{
  "status": "UP"
}
```

This is important because Kubernetes uses readiness to determine whether the Pod should receive traffic.

---

# 29. Verify Prometheus Endpoint

```bash
kubectl exec -it <pod-name> \
  -n ecommerce-dev \
  -- curl -s \
  http://localhost:8080/actuator/prometheus
```

You should see metrics such as:

```text
jvm_memory_used_bytes
http_server_requests_seconds
process_cpu_usage
```

This connects directly to our later Prometheus configuration.

---

# 30. Test the Service Internally

Port-forward the service:

```bash
kubectl port-forward \
  -n ecommerce-dev \
  svc/order-service \
  8080:8080
```

From your local machine:

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

# 31. Test Create Order

Example request:

```bash
curl -X POST \
  http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 1001,
    "productId": 5001,
    "quantity": 2
  }'
```

The exact response depends on the implementation from Chapter 54.

Conceptually:

```json
{
  "id": 1,
  "customerId": 1001,
  "productId": 5001,
  "quantity": 2
}
```

---

# 32. Now Connect RDS

The application should **not** use:

```text
localhost
```

inside Kubernetes.

Incorrect:

```yaml
DB_URL: jdbc:postgresql://localhost:5432/orders
```

Inside a Pod:

```text
localhost
```

means:

```text
the same container/Pod
```

It does not mean RDS.

---

# 33. Correct Database Architecture

The production path is:

```text
Order Service Pod
       │
       │ TCP 5432
       ▼
RDS PostgreSQL
       │
       ▼
orders database
```

Example:

```text
jdbc:postgresql://orders-db.xxxxx.ap-south-1.rds.amazonaws.com:5432/orders
```

The actual endpoint should come from the deployed RDS instance.

---

# 34. RDS Security Group

The database security group should allow:

```text
Source:
EKS application security group

Protocol:
TCP

Port:
5432
```

Do **not** use:

```text
0.0.0.0/0
```

for PostgreSQL access.

The intended network path is:

```text
Order Pod
   │
   ▼
EKS node/network
   │
   ▼
RDS Security Group
   │
   ▼
PostgreSQL :5432
```

---

# 35. Important EKS Networking Detail

With AWS VPC CNI, Pods receive VPC networking/IP addresses according to the cluster networking configuration.

Therefore database access must be designed around:

```text
Pod networking
Security groups
Routes
Network ACLs
DNS
RDS subnet placement
```

Don't assume:

```text
Pod IP
=
Node IP
```

---

# 36. Store DB Configuration Correctly

Our production application should receive:

```text
DB_URL
DB_USERNAME
DB_PASSWORD
```

without putting the password in GitOps.

Conceptually:

```text
AWS Secrets Manager
        │
        ▼
External Secrets Operator
        │
        ▼
Kubernetes Secret
        │
        ▼
Order Service Pod
```

For example, the Kubernetes Secret might provide:

```text
DB_USERNAME
DB_PASSWORD
DB_URL
```

---

# 37. ExternalSecret Example

A simplified pattern:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-service-db
  namespace: ecommerce-dev

spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secretsmanager
    kind: ClusterSecretStore

  target:
    name: order-service-db

  data:
    - secretKey: DB_USERNAME
      remoteRef:
        key: ecommerce/order-service/database
        property: username

    - secretKey: DB_PASSWORD
      remoteRef:
        key: ecommerce/order-service/database
        property: password

    - secretKey: DB_URL
      remoteRef:
        key: ecommerce/order-service/database
        property: url
```

The actual External Secrets API version should match the version installed in the cluster.

---

# 38. Pod Identity

The External Secrets Operator needs AWS access.

The architecture is:

```text
External Secrets Pod
        │
        ▼
ServiceAccount
        │
        ▼
EKS Pod Identity
        │
        ▼
IAM Role
        │
        ▼
Secrets Manager
```

The IAM policy should be scoped to the specific secret.

Example permission:

```json
{
  "Effect": "Allow",
  "Action": [
    "secretsmanager:GetSecretValue"
  ],
  "Resource": "arn:aws:secretsmanager:ap-south-1:123456789012:secret:ecommerce/order-service/database-*"
}
```

Avoid:

```json
"Resource": "*"
```

when a specific secret ARN can be used.

---

# 39. Order Service Workload IAM

The order-service itself may eventually require:

```text
SQS
S3
Secrets Manager
KMS
```

Its IAM role is separate from:

```text
External Secrets IAM role
ArgoCD IAM
EKS node IAM role
GitHub Actions IAM
```

For example:

```text
order-service
      │
      ▼
ServiceAccount
      │
      ▼
EKS Pod Identity
      │
      ▼
order-service IAM role
```

This is workload identity.

---

# 40. Update Helm Deployment

The Deployment should consume the Kubernetes Secret.

For example:

```yaml
envFrom:
  - secretRef:
      name: order-service-db
```

Then the Spring application receives:

```text
DB_URL
DB_USERNAME
DB_PASSWORD
```

without these values appearing in:

```text
Helm values
Git
Dockerfile
ConfigMap
```

---

# 41. Spring Boot Production Configuration

Our production configuration should use environment variables:

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

  jpa:
    hibernate:
      ddl-auto: validate
```

Important:

```text
Production
ddl-auto: validate
```

not:

```text
ddl-auto: update
```

Database schema changes should be handled using controlled migrations such as:

```text
Flyway
```

or:

```text
Liquibase
```

---

# 42. Database Connectivity Test

After the secret is available:

```bash
kubectl exec -it <pod-name> \
  -n ecommerce-dev \
  -- env | grep '^DB_'
```

Be careful with this command because it can expose secrets in your terminal.

A safer approach is to inspect only non-sensitive configuration or test connectivity from the application itself.

For DNS testing:

```bash
kubectl exec -it <pod-name> \
  -n ecommerce-dev \
  -- getent hosts <RDS-ENDPOINT>
```

Then verify application health/logs.

---

# 43. RDS Connection Failure Troubleshooting

If the application reports:

```text
Connection refused
```

check:

```text
1. RDS status
2. RDS endpoint
3. DNS resolution
4. Route tables
5. Security groups
6. Network ACLs
7. DB port
8. Credentials
9. Database name
10. Connection limits
```

Useful command:

```bash
aws rds describe-db-instances \
  --region ap-south-1
```

Check the DB status:

```text
available
```

---

# 44. Test Database Port

From a debugging Pod with appropriate tools:

```bash
nc -vz <RDS-ENDPOINT> 5432
```

Expected:

```text
succeeded
```

If it fails:

```text
Order Pod
   │
   X
   │
RDS
```

The issue is likely network/security/DNS rather than Spring Boot itself.

---

# 45. Deploy Ingress

Our Helm chart contains:

```yaml
ingress:
  enabled: true
  className: alb
  host: order-dev.example.com
```

The Ingress should look conceptually like:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: order-service
  namespace: ecommerce-dev
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb

  rules:
    - host: order-dev.example.com
      http:
        paths:
          - path: /api/v1/orders
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8080
```

The AWS Load Balancer Controller watches this resource.

---

# 46. What Happens When Ingress Is Created?

The chain is:

```text
Kubernetes Ingress
        │
        ▼
AWS Load Balancer Controller
        │
        ▼
AWS APIs
        │
        ▼
Application Load Balancer
        │
        ▼
Target Group
        │
        ▼
Order Service Pods
```

The controller's IAM permissions come from its own AWS identity configuration.

ArgoCD simply applies the Kubernetes Ingress object.

---

# 47. Check the Ingress

```bash
kubectl get ingress \
  -n ecommerce-dev
```

You may initially see:

```text
NAME            CLASS   HOSTS
order-service   alb     order-dev.example.com
```

The `ADDRESS` field should eventually contain the ALB hostname.

---

# 48. Describe the Ingress

```bash
kubectl describe ingress \
  order-service \
  -n ecommerce-dev
```

Look at:

```text
Events
Address
Rules
Annotations
Backend
```

If the ALB isn't created, the Events section is one of the first places to check.

---

# 49. Check Load Balancer Controller Logs

```bash
kubectl logs \
  -n kube-system \
  deployment/aws-load-balancer-controller
```

For a specific Pod:

```bash
kubectl logs \
  -n kube-system \
  <controller-pod>
```

Search for:

```text
error
failed
reconcile
```

---

# 50. Check AWS ALB

Using AWS CLI:

```bash
aws elbv2 describe-load-balancers \
  --region ap-south-1
```

Find the ALB created for the Ingress.

Then:

```bash
aws elbv2 describe-target-groups \
  --region ap-south-1
```

The target group should contain the expected targets.

---

# 51. Target-Type IP

We configured:

```yaml
alb.ingress.kubernetes.io/target-type: ip
```

This means the ALB can route directly to Pod IPs.

Conceptually:

```text
ALB
 │
 ├── Pod IP 10.x.x.x
 │
 └── Pod IP 10.x.x.x
```

This is different from instance targeting, where the ALB targets EC2 nodes.

---

# 52. Verify Target Health

Use:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <TARGET_GROUP_ARN> \
  --region ap-south-1
```

Expected:

```text
State: healthy
```

If targets are:

```text
unhealthy
```

check:

```text
Pod readiness
Service port
Target port
Security groups
Health check path
Application listener
Network connectivity
```

---

# 53. DNS

Once the ALB works, create DNS:

```text
order-dev.example.com
```

pointing to the ALB.

For Route 53, use an Alias record targeting the ALB rather than hardcoding an ALB IP.

Architecture:

```text
order-dev.example.com
          │
          ▼
      Route 53
          │
          ▼
         ALB
          │
          ▼
        EKS
```

---

# 54. DNS Verification

```bash
nslookup order-dev.example.com
```

or:

```bash
dig order-dev.example.com
```

Then:

```bash
curl -v \
  https://order-dev.example.com/actuator/health
```

The actual TLS configuration depends on the certificate/Ingress configuration implemented for the environment.

---

# 55. Complete Request Path

At this point, the full request looks like:

```text
Client
  │
  │ HTTPS
  ▼
Route 53
  │
  ▼
AWS ALB
  │
  │ HTTP/HTTPS
  ▼
Kubernetes Ingress
  │
  ▼
ClusterIP Service
  │
  ▼
Order Service Pod
  │
  ▼
Spring Boot
  │
  ▼
PostgreSQL
  │
  ▼
RDS
```

This is the first real end-to-end architecture of our project.

---

# 56. Test Real API

Once DNS, ALB, and application are ready:

```bash
curl -X POST \
  https://order-dev.example.com/api/v1/orders \
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
  https://order-dev.example.com/api/v1/orders
```

And:

```bash
curl \
  https://order-dev.example.com/api/v1/orders/1
```

---

# 57. Observe the Request

While sending the request:

```bash
kubectl logs \
  -f deployment/order-service \
  -n ecommerce-dev
```

You should see application activity.

Later, when OpenTelemetry is implemented, the same request can be correlated through:

```text
ALB
 ↓
Order Service
 ↓
Database
```

using trace IDs.

---

# 58. Verify HPA

Check:

```bash
kubectl get hpa \
  -n ecommerce-dev
```

Example:

```text
NAME            TARGETS   MINPODS   MAXPODS
order-service   20%/70%   2         5
```

Check:

```bash
kubectl describe hpa \
  order-service \
  -n ecommerce-dev
```

If metrics aren't available, investigate Metrics Server:

```bash
kubectl get deployment \
  metrics-server \
  -n kube-system
```

Then:

```bash
kubectl top pods \
  -n ecommerce-dev
```

---

# 59. Verify PDB

```bash
kubectl get pdb \
  -n ecommerce-dev
```

Expected:

```text
NAME            MIN AVAILABLE
order-service   1
```

For production we use a stronger configuration such as:

```text
minAvailable: 2
```

when three or more replicas are running.

---

# 60. Test Rolling Deployment

Change the application image:

```text
8f31a2c
```

to:

```text
91ac732
```

Update GitOps.

Then:

```bash
kubectl get pods \
  -n ecommerce-dev \
  -w
```

Watch:

```text
Old Pod
   │
   ▼
New Pod created
   │
   ▼
Startup probe
   │
   ▼
Readiness probe
   │
   ▼
New Pod Ready
   │
   ▼
Old Pod terminated
```

---

# 61. Verify Zero-Downtime Behavior

During deployment, repeatedly run:

```bash
while true; do
  curl -s -o /dev/null \
    -w "%{http_code}\n" \
    https://order-dev.example.com/actuator/health
  sleep 1
done
```

You should avoid unexpected:

```text
500
502
503
504
```

during a healthy rolling deployment.

If errors occur, investigate:

```text
Readiness
ALB target health
Pod termination
Connection draining
Application startup
Resource capacity
```

---

# 62. Test Pod Failure

Delete one Pod:

```bash
kubectl delete pod \
  <pod-name> \
  -n ecommerce-dev
```

Watch:

```bash
kubectl get pods \
  -n ecommerce-dev \
  -w
```

Kubernetes should create a replacement because the Deployment desired state is still:

```text
2 replicas
```

The flow is:

```text
Pod deleted
    │
    ▼
ReplicaSet detects missing replica
    │
    ▼
New Pod created
    │
    ▼
Readiness succeeds
    │
    ▼
Traffic resumes normally
```

---

# 63. Test ArgoCD Self-Healing

Now intentionally change the Deployment:

```bash
kubectl scale deployment order-service \
  -n ecommerce-dev \
  --replicas=1
```

Git says:

```text
2 replicas
```

Cluster says:

```text
1 replica
```

ArgoCD sees:

```text
OutOfSync
```

With `selfHeal` enabled, it should reconcile back toward:

```text
2 replicas
```

Check:

```bash
kubectl get deployment \
  order-service \
  -n ecommerce-dev
```

---

# 64. Test Rollback

Suppose:

```text
Current:
91ac732

Previous:
8f31a2c
```

Revert the GitOps change.

Then:

```bash
git push
```

ArgoCD detects the desired version:

```text
8f31a2c
```

Check:

```bash
argocd app get order-service-dev
```

Then:

```bash
kubectl get deployment \
  order-service \
  -n ecommerce-dev \
  -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Expected:

```text
.../order-service:8f31a2c
```

---

# 65. Common Failure: `ImagePullBackOff`

### Check Pod

```bash
kubectl get pods -n ecommerce-dev
```

### Describe

```bash
kubectl describe pod <pod> \
  -n ecommerce-dev
```

Look for:

```text
Failed to pull image
```

### Verify ECR

```bash
aws ecr describe-images \
  --repository-name order-service \
  --region ap-south-1
```

### Verify node permissions

The node/workload architecture must allow the node to pull from ECR.

Also verify:

```text
ECR repository
Image tag
AWS region
Node networking
ECR permissions
```

---

# 66. Common Failure: `CrashLoopBackOff`

Check:

```bash
kubectl logs \
  <pod> \
  -n ecommerce-dev
```

Then previous logs:

```bash
kubectl logs \
  <pod> \
  -n ecommerce-dev \
  --previous
```

Check:

```bash
kubectl describe pod \
  <pod> \
  -n ecommerce-dev
```

Typical causes:

```text
Bad DB configuration
Application startup failure
Missing environment variable
Invalid configuration
Port mismatch
Java exception
Secret missing
RDS unavailable
```

---

# 67. Common Failure: Readiness Probe Failure

Check:

```bash
kubectl describe pod \
  <pod> \
  -n ecommerce-dev
```

Look for:

```text
Readiness probe failed
```

Test manually:

```bash
kubectl exec -it <pod> \
  -n ecommerce-dev \
  -- curl localhost:8080/actuator/health/readiness
```

Possible causes:

```text
Application not started
Wrong port
Wrong path
Database unavailable
Slow startup
Insufficient resources
```

---

# 68. Common Failure: ALB Not Created

Check:

```bash
kubectl describe ingress \
  order-service \
  -n ecommerce-dev
```

Then:

```bash
kubectl logs \
  -n kube-system \
  deployment/aws-load-balancer-controller
```

Check:

```text
IngressClass
IAM permissions
Subnet tags
Security groups
Annotations
Controller status
```

---

# 69. Common Failure: ALB Targets Unhealthy

Check:

```bash
kubectl get pods \
  -n ecommerce-dev \
  -o wide
```

Then:

```bash
kubectl get endpoints \
  order-service \
  -n ecommerce-dev
```

Check:

```bash
kubectl describe service \
  order-service \
  -n ecommerce-dev
```

Then inspect target health from AWS:

```bash
aws elbv2 describe-target-health \
  --target-group-arn <TARGET_GROUP_ARN> \
  --region ap-south-1
```

---

# 70. Common Failure: RDS Connection Timeout

Follow this order:

```text
1. Is RDS available?
        ↓
2. Does DNS resolve?
        ↓
3. Is port 5432 reachable?
        ↓
4. Does RDS SG allow source?
        ↓
5. Are routes correct?
        ↓
6. Are credentials correct?
        ↓
7. Is database name correct?
        ↓
8. Are connection limits exhausted?
```

Do not immediately modify the application code.

First establish where the connection fails.

---

# 71. Common Failure: ArgoCD Says Synced but Application Doesn't Work

Remember:

```text
Synced != Application healthy
```

Check:

```bash
argocd app get order-service-dev
```

Then:

```bash
kubectl get pods -n ecommerce-dev
```

Then:

```bash
kubectl describe pod <pod> \
  -n ecommerce-dev
```

Then:

```bash
kubectl logs <pod> \
  -n ecommerce-dev
```

Then:

```bash
kubectl get ingress \
  -n ecommerce-dev
```

Then:

```bash
kubectl describe ingress \
  order-service \
  -n ecommerce-dev
```

This gives you a systematic troubleshooting path.

---

# 72. Production Deployment Model

For production, the exact flow becomes:

```text
Developer
    │
    ▼
PR
    │
    ▼
CI
    │
    ├── Build
    ├── Test
    ├── Security
    └── Image
          │
          ▼
         ECR
          │
          ▼
       DEV
          │
          ▼
        QA
          │
          ▼
      STAGING
          │
          ▼
 Production Approval
          │
          ▼
        PROD
```

And every environment uses the **same immutable application artifact**.

---

# 73. Final End-to-End Architecture

At this point the first microservice has a real deployment path:

```text
                         GitHub
                            │
             ┌──────────────┴──────────────┐
             │                             │
             ▼                             ▼
      order-service                      gitops
             │                             │
             ▼                             │
       GitHub Actions                      │
             │                             │
     ┌───────┼────────┐                    │
     │       │        │                    │
   Build   Test    Security                │
     │       │        │                    │
     └───────┼────────┘                    │
             ▼                             │
        Docker Image                       │
             │                             │
             ▼                             │
            ECR                            │
             │                             │
             └─────────────┬───────────────┘
                           │
                           ▼
                         ArgoCD
                           │
                           ▼
                          EKS
                           │
              ┌────────────┼────────────┐
              │            │            │
              ▼            ▼            ▼
             ALB       Order Pods    HPA/PDB
              │            │
              │            ▼
              │        RDS PostgreSQL
              │
              ▼
           Internet
```

---

# 74. Verification Checklist

The first deployment is considered successful when all of these work:

```text
[ ] EKS nodes Ready
[ ] ArgoCD Running
[ ] Git repositories accessible
[ ] ECR repository exists
[ ] Order image exists in ECR
[ ] GitOps contains image version
[ ] ArgoCD Application Synced
[ ] ArgoCD Application Healthy
[ ] Deployment available
[ ] Pods Running
[ ] Service exists
[ ] Service endpoints populated
[ ] Readiness probe passes
[ ] Liveness probe passes
[ ] HPA exists
[ ] PDB exists
[ ] Ingress exists
[ ] ALB created
[ ] ALB targets healthy
[ ] DNS resolves
[ ] HTTPS endpoint works
[ ] Order API works
[ ] RDS connectivity works
[ ] Logs available
[ ] GitOps rollback works
[ ] Pod replacement works
[ ] ArgoCD self-healing works
```

---

# 75. The Most Important Result

We have now moved from:

```text
"Here are some Kubernetes YAML files"
```

to an actual platform workflow:

```text
Code
 ↓
CI
 ↓
Security
 ↓
Docker
 ↓
ECR
 ↓
GitOps
 ↓
ArgoCD
 ↓
EKS
 ↓
ALB
 ↓
Service
 ↓
Pod
 ↓
RDS
```

That is the baseline pattern we can now replicate for the remaining 20 microservices.

---

# Next Chapter

## `67-Order-Service-CI-GitOps-Automation.md`

Next we should remove the remaining manual steps.

We will connect **GitHub Actions directly to the GitOps promotion process**:

```text
Developer PR
      ↓
GitHub Actions
      ↓
Build + Test
      ↓
SonarQube
      ↓
Trivy
      ↓
Docker Build
      ↓
ECR
      ↓
Get Image Digest
      ↓
Update GitOps
      ↓
Create Promotion PR
      ↓
GitOps Merge
      ↓
ArgoCD
      ↓
EKS
```

Then the deployment becomes a genuine **automated CI → ECR → GitOps → ArgoCD → EKS pipeline**, rather than manually changing the GitOps image tag.
