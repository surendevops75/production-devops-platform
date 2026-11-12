# 105 — RDS Production Implementation

> **Project:** Enterprise E-Commerce Platform
> **Platform:** AWS EKS + Amazon RDS PostgreSQL
> **Region:** `ap-south-1`
> **Database:** PostgreSQL
> **Primary workload:** `order-service`
> **Management:** Terraform + AWS Secrets Manager + GitOps
> **Access:** Private networking only

---

## 1. Purpose

In the previous chapters we built the EKS compute layer:

```text
100 → Spot Instances
101 → Multi-AZ Workload Placement
102 → Kubernetes Resource Management
103 → HPA / VPA / KEDA
104 → Cluster Autoscaling
```

Now we implement the production database layer.

Our target architecture is:

```text
Internet
   │
   ▼
CloudFront / WAF
   │
   ▼
ALB
   │
   ▼
EKS
   │
   ▼
order-service
   │
   ▼
RDS PostgreSQL
```

The database must **not** be directly accessible from the Internet.

---

# 2. Production Database Architecture

We will use:

```text
Amazon RDS PostgreSQL
        │
        ├── Private Subnet AZ-A
        ├── Private Subnet AZ-B
        └── Private Subnet AZ-C
```

RDS uses a DB subnet group spanning multiple Availability Zones.

For a Multi-AZ deployment, AWS manages the underlying database availability architecture.

Our application architecture:

```text
                    EKS
                     │
              ┌──────┴──────┐
              │             │
        order-service   other services
              │
              │ TCP 5432
              ▼
       RDS PostgreSQL
```

---

# 3. Why RDS Instead of PostgreSQL Inside EKS?

We previously discussed running databases inside Kubernetes.

For the production e-commerce platform, we prefer managed RDS because AWS provides managed capabilities around:

```text
backups
patching
Multi-AZ
monitoring
storage
failover
maintenance
recovery
```

We don't have to operate:

```text
PostgreSQL StatefulSet
PVC
database replication
database failover controller
database backup system
```

ourselves for the primary production database.

---

# 4. Database Responsibility Boundary

This distinction is important.

### Kubernetes manages:

```text
application Pods
Services
Ingress
HPA
PDB
Secrets integration
```

### AWS RDS manages:

```text
PostgreSQL instance
storage
database availability
database backups
database maintenance
database failover
```

### Terraform manages:

```text
RDS infrastructure configuration
DB subnet group
security group
parameter group
option groups where applicable
```

### Application manages:

```text
SQL queries
transactions
connection pool
schema compatibility
migrations
```

---

# 5. Target Production Architecture

```text
                         AWS
                          │
          ┌───────────────┴───────────────┐
          │                               │
        EKS                             RDS
          │                               │
   Private Subnets                  Private Subnets
          │                               │
   order-service                         │
          │                               │
          └──────── TCP 5432 ────────────┘
                          │
                     PostgreSQL
```

Security:

```text
Internet
   X
   │
   X  No direct database access
   │
EKS Security Group
   │
   │ TCP 5432
   ▼
RDS Security Group
```

---

# 6. Database Naming

For our project:

```text
Database:
ecommerce
```

Order service database:

```text
orders
```

Potential production naming:

```text
prod-ecommerce-postgres
```

or:

```text
ecommerce-prod-postgresql
```

Choose one convention and keep it consistent.

---

# 7. Database-per-Service Strategy

Our 21 microservices should not automatically share one schema.

For example:

```text
order-service
    ↓
orders database

payment-service
    ↓
payments database

inventory-service
    ↓
inventory database
```

This supports service ownership and reduces tight coupling.

However, **database-per-service does not necessarily mean 21 separate RDS instances**.

A common approach is:

```text
RDS PostgreSQL cluster/instance
       │
       ├── orders database
       ├── payments database
       ├── inventory database
       └── ...
```

The exact isolation model depends on:

```text
security
performance
failure isolation
cost
team ownership
compliance
```

---

# 8. Important Database Ownership Rule

`order-service` owns:

```text
orders
```

Other services should not directly modify its tables.

Instead:

```text
payment-service
       │
       ▼
API / event
       │
       ▼
order-service
```

or:

```text
order-service
       │
       ▼
event
       │
       ▼
payment-service
```

This keeps service boundaries clear.

---

# 9. VPC Placement

Our VPC from Chapter 57 has:

```text
10.0.0.0/16
```

Private subnets:

```text
10.0.11.0/24
10.0.12.0/24
10.0.13.0/24
```

These are suitable candidates for internal workloads and database placement.

RDS should not be placed in public subnets for this architecture.

---

# 10. DB Subnet Group

RDS needs a DB subnet group.

Conceptually:

```text
DB Subnet Group
   │
   ├── Private subnet AZ-A
   ├── Private subnet AZ-B
   └── Private subnet AZ-C
```

Terraform:

```hcl
resource "aws_db_subnet_group" "postgres" {
  name = "ecommerce-postgres"

  subnet_ids = var.private_subnet_ids

  tags = {
    Name = "ecommerce-postgres"
  }
}
```

---

# 11. RDS Security Group

Create a dedicated security group:

```hcl
resource "aws_security_group" "rds" {
  name        = "ecommerce-rds"
  description = "Security group for RDS PostgreSQL"
  vpc_id      = var.vpc_id

  tags = {
    Name = "ecommerce-rds"
  }
}
```

Do not use the default VPC security group for production database access.

---

# 12. Database Ingress Rule

The key security rule is:

```text
EKS application SG
        │
        │ TCP 5432
        ▼
RDS SG
```

Terraform:

```hcl
resource "aws_vpc_security_group_ingress_rule" "postgres_from_eks" {
  security_group_id            = aws_security_group.rds.id
  referenced_security_group_id = var.eks_application_security_group_id

  from_port = 5432
  to_port   = 5432
  ip_protocol = "tcp"
}
```

This is preferable to:

```text
0.0.0.0/0 → 5432
```

Never expose PostgreSQL directly to the Internet in this architecture.

---

# 13. Egress

The RDS security group does not need unrestricted inbound access.

The application initiates:

```text
EKS → RDS
```

not:

```text
RDS → EKS
```

Security group design should therefore follow actual connection direction and AWS security-group statefulness.

---

# 14. PostgreSQL Port

Default PostgreSQL port:

```text
5432
```

Our connection:

```text
jdbc:postgresql://<rds-endpoint>:5432/orders
```

Example:

```text
jdbc:postgresql://ecommerce-prod.xxxxx.ap-south-1.rds.amazonaws.com:5432/orders
```

The actual endpoint is generated by AWS.

Do not hard-code it in source code.

---

# 15. RDS Terraform

A simplified production-oriented resource:

```hcl
resource "aws_db_instance" "postgres" {
  identifier = "ecommerce-prod-postgres"

  engine         = "postgres"
  engine_version = var.postgres_engine_version

  instance_class = var.db_instance_class

  allocated_storage     = var.allocated_storage
  max_allocated_storage = var.max_allocated_storage
  storage_type          = "gp3"
  storage_encrypted     = true

  kms_key_id = var.kms_key_id

  db_name  = "ecommerce"
  username = var.master_username

  manage_master_user_password = true

  db_subnet_group_name   = aws_db_subnet_group.postgres.name
  vpc_security_group_ids = [aws_security_group.rds.id]

  multi_az = true

  backup_retention_period = 7
  backup_window            = "18:00-19:00"
  maintenance_window       = "sun:19:00-sun:20:00"

  deletion_protection = true
  skip_final_snapshot = false

  auto_minor_version_upgrade = true

  apply_immediately = false

  tags = {
    Environment = "prod"
    Service     = "database"
  }
}
```

The exact Terraform provider arguments and supported RDS features should be verified against the provider version used by the project.

---

# 16. Don't Copy Production Values Blindly

The following are examples:

```text
7-day backup
gp3
specific instance class
maintenance window
```

Actual production values depend on:

```text
RPO
RTO
traffic
database size
IOPS
throughput
SLA
compliance
budget
```

We will tune them during later database-performance chapters.

---

# 17. RDS Storage

For production PostgreSQL we can use:

```text
gp3
```

rather than simply assuming an old default storage type.

Why gp3?

It allows storage and performance characteristics to be managed more independently than older gp2-style configurations.

But:

> Don't choose storage based only on price.

Measure:

```text
IOPS
throughput
latency
queue depth
database workload
```

---

# 18. Storage Encryption

Production database storage should be encrypted.

```hcl
storage_encrypted = true
```

Prefer a customer-managed KMS key where organizational requirements call for it.

Architecture:

```text
RDS
 │
 ▼
KMS
 │
 ▼
Encrypted storage
```

---

# 19. KMS Key

Conceptually:

```hcl
resource "aws_kms_key" "rds" {
  description = "KMS key for ecommerce RDS"

  enable_key_rotation = true

  tags = {
    Name = "ecommerce-rds"
  }
}
```

Then:

```hcl
kms_key_id = aws_kms_key.rds.arn
```

KMS key policy should be carefully designed around the required AWS service and administrative access.

---

# 20. Master Password

Do not place:

```text
DB_PASSWORD
```

inside:

```text
Git
Terraform variables
Helm values
Dockerfile
GitHub Actions YAML
ConfigMap
```

For supported RDS configurations, Terraform can delegate master-password management to AWS:

```hcl
manage_master_user_password = true
```

This allows AWS Secrets Manager to manage the master credentials.

Always verify the exact behavior with the AWS provider version being used.

---

# 21. Application Credentials

The application should not use the RDS master account.

Instead create a dedicated application database user:

```text
orders_app
```

Example:

```text
orders_app
    ↓
orders database
```

Permissions should be limited to what the service requires.

---

# 22. Database Users

Conceptually:

```text
PostgreSQL
│
├── admin / master
│
├── orders_app
│
├── payments_app
│
└── inventory_app
```

The application should use:

```text
orders_app
```

not the master user.

---

# 23. Application Database Permissions

For `orders_app`, grant only required privileges.

For example:

```sql
GRANT CONNECT ON DATABASE orders TO orders_app;
```

Then appropriate schema/table privileges are granted according to the migration/application model.

Avoid:

```sql
GRANT ALL PRIVILEGES ON DATABASE ...
```

unless there is a specific administrative reason.

---

# 24. Secrets Architecture

Our production secret flow is:

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
        │
        ▼
PostgreSQL
```

The source of truth remains:

```text
AWS Secrets Manager
```

---

# 25. Secret Example

Conceptual secret:

```text
prod/ecommerce/order-service/database
```

Data:

```json
{
  "username": "orders_app",
  "password": "********",
  "host": "RDS_ENDPOINT",
  "port": "5432",
  "database": "orders"
}
```

Do not commit the actual value.

---

# 26. ExternalSecret

Example:

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: order-db
  namespace: ecommerce-prod
spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secretsmanager
    kind: SecretStore

  target:
    name: order-db

  data:
    - secretKey: username
      remoteRef:
        key: prod/ecommerce/order-service/database
        property: username

    - secretKey: password
      remoteRef:
        key: prod/ecommerce/order-service/database
        property: password

    - secretKey: host
      remoteRef:
        key: prod/ecommerce/order-service/database
        property: host
```

The exact External Secrets API version should match the installed operator version.

---

# 27. Pod Identity

The External Secrets Operator needs AWS permissions to read the secret.

Architecture:

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

The IAM role should only have:

```text
secretsmanager:GetSecretValue
```

for the required secret resources.

---

# 28. Application Secret Flow

The application itself can receive:

```text
DB_USERNAME
DB_PASSWORD
DB_URL
```

through the Kubernetes Secret.

For example:

```yaml
env:
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: order-db
        key: username

  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: order-db
        key: password
```

---

# 29. Database URL

Our Spring Boot production configuration:

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

Example environment variable:

```text
DB_URL=jdbc:postgresql://RDS_ENDPOINT:5432/orders
```

The endpoint should come from the environment/secret configuration rather than source code.

---

# 30. RDS DNS

RDS provides a DNS endpoint.

Application:

```text
order-service
     │
     ▼
RDS DNS endpoint
     │
     ▼
private IP
     │
     ▼
PostgreSQL
```

Do not hard-code an RDS IP address.

Use the RDS endpoint.

---

# 31. DNS Troubleshooting

From an approved troubleshooting environment or application Pod:

```bash
nslookup <rds-endpoint>
```

or:

```bash
getent hosts <rds-endpoint>
```

Then test TCP connectivity:

```bash
nc -vz <rds-endpoint> 5432
```

Do not install troubleshooting tools into production containers merely for convenience.

A dedicated troubleshooting image or approved debug Pod is preferable.

---

# 32. Connectivity Test

From a controlled Pod:

```bash
nc -vz <rds-endpoint> 5432
```

Expected:

```text
Connection to <rds-endpoint> 5432 port [tcp/postgresql] succeeded!
```

If it fails, investigate:

```text
DNS
route
security groups
NACL
network policy
RDS status
subnet
```

---

# 33. NetworkPolicy

AWS Security Groups protect the network boundary.

Kubernetes NetworkPolicy can add another layer.

For example:

```text
order-service namespace
       │
       │ allowed
       ▼
approved application traffic
```

and:

```text
unapproved Pod
       X
       │
       X
```

NetworkPolicy and AWS Security Groups solve different layers of the problem.

---

# 34. Application → RDS Flow

The complete packet path is approximately:

```text
Order Pod
   │
   ▼
Pod network / VPC CNI
   │
   ▼
EKS node networking
   │
   ▼
VPC routing
   │
   ▼
RDS subnet
   │
   ▼
RDS Security Group
   │
   ▼
PostgreSQL :5432
```

If any required layer blocks the connection, the application fails.

---

# 35. Security Group Troubleshooting

Check RDS SG:

```text
Inbound:
TCP 5432
Source:
EKS application security group
```

Do not use:

```text
0.0.0.0/0
```

for production database access.

---

# 36. RDS Multi-AZ

Production RDS should use Multi-AZ where the workload's availability requirements call for it.

Conceptually:

```text
Primary DB
   │
   │ managed replication
   ▼
Standby
```

AWS manages the failover architecture.

Applications continue using the RDS endpoint rather than connecting to a fixed database IP.

---

# 37. Important Multi-AZ Understanding

Multi-AZ is primarily for:

```text
high availability
failover
```

It is not automatically:

```text
read scaling
```

For read scaling, use an appropriate read-replica architecture.

---

# 38. Read Replica

Conceptually:

```text
Primary
   │
   ├──────► Read Replica A
   │
   └──────► Read Replica B
```

Application traffic can potentially separate:

```text
writes → primary
reads  → replica
```

but this requires application/data-access architecture that understands replication lag and consistency.

We will cover this more deeply in performance optimization.

---

# 39. Database Connection Pool

Our Spring Boot application should not create a new TCP/database connection for every request.

It normally uses a connection pool such as HikariCP.

Conceptually:

```text
100 HTTP requests
       │
       ▼
Connection Pool
       │
       ├── connection 1
       ├── connection 2
       ├── connection 3
       └── ...
```

---

# 40. Example Hikari Configuration

Example:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

These are starting examples.

Do not automatically configure every service with:

```text
maximum-pool-size = 50
```

---

# 41. Connection Multiplication Problem

Suppose:

```text
20 Pods
```

and:

```text
maximum-pool-size = 10
```

Potential connection demand:

```text
20 × 10 = 200
```

If we scale:

```text
20 → 50 Pods
```

potential connections:

```text
50 × 10 = 500
```

This is why:

```text
HPA
+
KEDA
+
connection pool
+
RDS
```

must be designed together.

---

# 42. Production Pooling Strategy

We should calculate:

```text
Total possible DB connections
=
maximum application replicas
×
maximum pool per Pod
```

Then reserve capacity for:

```text
administrative connections
monitoring
migrations
other services
failover
```

The result must remain within the database's safe operating capacity.

---

# 43. RDS Proxy

For some architectures, Amazon RDS Proxy can help manage database connections.

Architecture:

```text
Pods
 │
 ├── connection
 ├── connection
 ├── connection
 │
 ▼
RDS Proxy
 │
 ▼
RDS PostgreSQL
```

Potential benefits include:

```text
connection pooling
connection management
better behavior during rapid scaling
```

But RDS Proxy is not automatically required for every service.

We will evaluate it based on:

```text
connection churn
workload
scaling behavior
database limits
cost
```

---

# 44. Database Migrations

Do not use:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update
```

for production schema management.

Instead:

```text
Flyway
or
Liquibase
```

should control schema changes.

Production:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

This means Hibernate validates the schema instead of changing it automatically.

---

# 45. Why This Matters

Suppose version 1 has:

```text
orders
├── id
├── customer_id
└── status
```

Version 2 needs:

```text
payment_reference
```

We should execute:

```text
migration
   ↓
database schema
   ↓
application deployment
```

through controlled GitOps/release processes.

Not:

```text
application startup
   ↓
random schema modification
```

---

# 46. Database Migration Flow

Our future flow:

```text
Developer
   ↓
Migration file
   ↓
PR
   ↓
CI
   ↓
Test database
   ↓
QA
   ↓
Staging
   ↓
Production change approval
   ↓
Migration
   ↓
Application deployment
```

This will be implemented in:

```text
106 — Database Migration with Flyway/Liquibase
```

---

# 47. Expand-and-Contract Pattern

For zero-downtime schema changes:

### Phase 1

Add new column:

```text
new_column
```

without removing the old one.

### Phase 2

Deploy application capable of using both.

### Phase 3

Backfill data.

### Phase 4

Switch reads/writes.

### Phase 5

Remove old schema later.

Conceptually:

```text
Old App
   ↓
Old + New Schema
   ↓
New App
   ↓
Cleanup
```

This prevents application/database incompatibility during rolling deployments.

---

# 48. RDS Parameter Group

PostgreSQL parameters can be customized through an RDS parameter group.

Example Terraform:

```hcl
resource "aws_db_parameter_group" "postgres" {
  name   = "ecommerce-postgres"
  family = var.postgres_parameter_family

  parameter {
    name  = "log_min_duration_statement"
    value = "1000"
  }

  tags = {
    Name = "ecommerce-postgres"
  }
}
```

The exact parameter family must match the PostgreSQL engine version.

---

# 49. Don't Change Parameters Randomly

Before changing PostgreSQL parameters:

```text
1. Identify the problem.
2. Check AWS/PostgreSQL documentation.
3. Understand static vs dynamic parameters.
4. Test in DEV.
5. Validate in QA/STAGING.
6. Schedule production change if required.
7. Monitor after change.
```

Avoid tuning PostgreSQL based on arbitrary internet "best settings."

---

# 50. RDS Monitoring

Important signals include:

```text
CPUUtilization
FreeableMemory
DatabaseConnections
ReadIOPS
WriteIOPS
ReadLatency
WriteLatency
DiskQueueDepth
FreeStorageSpace
NetworkThroughput
```

Also monitor PostgreSQL-level behavior.

---

# 51. Connection Monitoring

One of the most important production metrics:

```text
DatabaseConnections
```

Example:

```text
RDS max connections = 500

Current:
420
```

This should trigger investigation before:

```text
500
```

is reached.

---

# 52. Connection Exhaustion Incident

Symptoms:

```text
order-service
connection timeout
```

Application logs:

```text
HikariPool - Connection is not available
```

Investigation:

```text
Pods = 30
Pool size = 15
Potential connections = 450
```

RDS:

```text
available safe connections = 400
```

Root cause:

```text
application scaled beyond DB connection capacity
```

Resolution may involve:

```text
reduce pool size
limit HPA
use RDS Proxy where appropriate
optimize DB usage
increase DB capacity if justified
```

This will later become:

```text
152 — RDS Connection Exhaustion
```

in our incident simulation phase.

---

# 53. RDS Backups

Production should have automated backups configured.

Example:

```hcl
backup_retention_period = 7
```

But the correct value depends on:

```text
RPO
compliance
data retention
cost
recovery requirements
```

---

# 54. Point-in-Time Recovery

RDS automated backups support recovery to a point within the available backup retention window.

Conceptually:

```text
Day 1
 │
 ├── 10:00
 ├── 11:00
 ├── 12:00
 ├── 13:00
 └── ...
```

If accidental deletion happens at:

```text
13:37
```

we may restore to an appropriate earlier point.

The exact recovery procedure and supported window depend on the RDS configuration.

---

# 55. Final Snapshot

Before destructive deletion:

```hcl
skip_final_snapshot = false
```

This helps ensure a final snapshot is retained according to the configured deletion process.

Production also uses:

```hcl
deletion_protection = true
```

to prevent accidental deletion.

---

# 56. RDS Maintenance

RDS requires maintenance operations.

Configure an appropriate window:

```hcl
maintenance_window = "sun:19:00-sun:20:00"
```

This is only an example.

Choose a window based on:

```text
traffic pattern
business hours
deployment schedule
on-call availability
regional operations
```

---

# 57. RDS Endpoint

Terraform can expose:

```hcl
output "rds_endpoint" {
  value = aws_db_instance.postgres.address
}
```

However, don't expose sensitive database credentials as Terraform outputs.

The endpoint itself is generally not a secret, but it should still be treated as internal infrastructure information.

---

# 58. Terraform State Warning

Terraform state can contain sensitive infrastructure information.

Therefore:

```text
terraform.tfstate
```

must not be committed to Git.

Our backend:

```text
S3
+
encryption
+
versioning
+
locking
```

must be protected.

This connects directly to:

```text
56 — Terraform State Bootstrap
```

---

# 59. Production Terraform Structure

Our infrastructure repository can contain:

```text
terraform/
├── modules/
│   └── rds/
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       └── README.md
│
└── environments/
    ├── dev/
    │   └── rds/
    ├── qa/
    │   └── rds/
    ├── staging/
    │   └── rds/
    └── prod/
        └── rds/
```

---

# 60. RDS Module Variables

Example:

```hcl
variable "identifier" {
  type = string
}

variable "engine_version" {
  type = string
}

variable "instance_class" {
  type = string
}

variable "allocated_storage" {
  type = number
}

variable "max_allocated_storage" {
  type = number
}

variable "multi_az" {
  type    = bool
  default = true
}

variable "private_subnet_ids" {
  type = list(string)
}

variable "security_group_ids" {
  type = list(string)
}
```

---

# 61. Production Environment

Example:

```hcl
module "postgres" {
  source = "../../modules/rds"

  identifier = "ecommerce-prod-postgres"

  engine_version = var.postgres_engine_version
  instance_class = var.db_instance_class

  allocated_storage     = 100
  max_allocated_storage = 500

  multi_az = true

  private_subnet_ids = module.vpc.private_subnet_ids

  security_group_ids = [
    module.rds_security_group.id
  ]
}
```

Exact sizes are examples and should be performance-tested.

---

# 62. Terraform Workflow

Production:

```text
Developer
   ↓
Terraform change
   ↓
PR
   ↓
terraform fmt
   ↓
terraform validate
   ↓
terraform plan
   ↓
review
   ↓
approval
   ↓
apply
```

Avoid:

```bash
terraform apply -auto-approve
```

as an uncontrolled production workflow.

---

# 63. RDS Deployment Order

Our infrastructure dependency is:

```text
VPC
 ↓
Private Subnets
 ↓
Security Groups
 ↓
DB Subnet Group
 ↓
KMS
 ↓
RDS
 ↓
Secrets
 ↓
External Secrets
 ↓
Application
```

Terraform should manage infrastructure dependencies.

Kubernetes/GitOps should manage application deployment.

---

# 64. Application Deployment Order

For the Order Service:

```text
1. VPC
2. RDS
3. database/user
4. secret
5. ExternalSecret
6. Kubernetes Secret
7. Order Service
8. connection test
9. health test
10. application API test
```

---

# 65. RDS Connectivity From Order Service

Final path:

```text
Order Service Pod
       │
       │ DB_URL
       ▼
RDS DNS
       │
       ▼
VPC
       │
       ▼
RDS Security Group
       │
       ▼
PostgreSQL :5432
```

---

# 66. Spring Boot Configuration

Production:

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

  jpa:
    hibernate:
      ddl-auto: validate

    properties:
      hibernate:
        format_sql: false

  datasource:
    hikari:
      maximum-pool-size: ${DB_POOL_MAX:10}
      minimum-idle: ${DB_POOL_MIN:2}
      connection-timeout: ${DB_CONNECTION_TIMEOUT:30000}
```

Avoid logging SQL or credentials in production unless there is a controlled troubleshooting reason.

---

# 67. Health Checks

Spring Boot health endpoint:

```text
/actuator/health
```

We should distinguish:

```text
liveness
readiness
```

A database outage should not necessarily make the application process itself "dead."

Readiness may be used to remove a Pod from service when it cannot safely serve requests, depending on the application's dependency strategy.

---

# 68. Database Health

Example:

```text
Pod
 │
 ├── process alive
 │
 └── DB connectivity
```

We should avoid creating a liveness probe that restarts the application continuously because RDS is temporarily unavailable.

That can create:

```text
restart storm
```

A dependency failure should be handled intentionally.

---

# 69. RDS Security Checklist

```text
[ ] RDS private
[ ] No public database access
[ ] Dedicated RDS security group
[ ] Only required EKS SG allowed
[ ] TCP 5432 restricted
[ ] Storage encrypted
[ ] KMS configured
[ ] Master credentials managed securely
[ ] Application user separate from master
[ ] Least privilege
[ ] Secrets Manager
[ ] External Secrets
[ ] Pod Identity
[ ] NetworkPolicy where appropriate
```

---

# 70. RDS Availability Checklist

```text
[ ] Multi-AZ where required
[ ] DB subnet group spans AZs
[ ] Automated backups
[ ] PITR tested
[ ] Final snapshot protection
[ ] Deletion protection
[ ] Maintenance window
[ ] Monitoring
[ ] Alerting
[ ] Failover procedure
```

---

# 71. RDS Performance Checklist

```text
[ ] CPU monitored
[ ] Memory monitored
[ ] Storage monitored
[ ] IOPS monitored
[ ] latency monitored
[ ] connection count monitored
[ ] connection pool tuned
[ ] slow queries investigated
[ ] indexes reviewed
[ ] query plans reviewed
[ ] read replicas considered if required
```

---

# 72. Production Troubleshooting — Connection Failure

### Symptom

```text
Order API:
500 Internal Server Error
```

Logs:

```text
Connection refused
```

Check:

```bash
kubectl logs <order-pod> -n ecommerce-prod
```

Then test DNS:

```bash
kubectl exec -it <debug-pod> -n ecommerce-prod -- \
  getent hosts <rds-endpoint>
```

Then:

```bash
kubectl exec -it <debug-pod> -n ecommerce-prod -- \
  nc -vz <rds-endpoint> 5432
```

---

# 73. If DNS Fails

Investigate:

```text
CoreDNS
VPC DNS
RDS endpoint
network configuration
```

If DNS resolves but TCP fails:

```text
security group
NACL
route
NetworkPolicy
RDS status
```

If TCP works but login fails:

```text
username
password
database name
authentication
PostgreSQL configuration
```

---

# 74. Troubleshooting Decision Tree

```text
Application cannot connect to RDS
              │
              ▼
         DNS resolves?
          /          \
        No            Yes
        │              │
     DNS issue       TCP 5432?
                       /    \
                     No      Yes
                     │        │
             Network/Security  │
                              │
                         Authentication?
                           /       \
                         No         Yes
                         │           │
                    Credentials     DB/app
                    /permissions    issue
```

---

# 75. RDS Failover

During a controlled failover test:

```text
Application
    │
    ▼
RDS endpoint
    │
    ▼
Primary
    X
    │
    ▼
AWS promotes standby
    │
    ▼
Endpoint points to new primary
    │
    ▼
Application reconnects
```

Application should use the endpoint rather than hard-coded IP.

This will be tested in:

```text
109 — RDS Failover Testing
```

---

# 76. What We Should Not Do

### Don't:

```text
store DB password in Git
```

### Don't:

```text
allow 0.0.0.0/0 to port 5432
```

### Don't:

```text
use RDS master account from application
```

### Don't:

```text
use ddl-auto=update in production
```

### Don't:

```text
create huge connection pools blindly
```

### Don't:

```text
put RDS credentials into ConfigMaps
```

### Don't:

```text
hard-code RDS IP addresses
```

### Don't:

```text
manually modify production infrastructure without Git/approval
```

---

# 77. Complete Production Architecture

```text
                           INTERNET
                              │
                              ▼
                       CloudFront / WAF
                              │
                              ▼
                             ALB
                              │
                              ▼
                         EKS Cluster
                              │
                     ┌────────┴────────┐
                     │                 │
                 HPA/KEDA           Pods
                                       │
                              order-service
                                       │
                              ServiceAccount
                                       │
                              EKS Pod Identity
                                       │
                                       ▼
                              AWS Secrets Manager
                                       │
                                       ▼
                                  DB Secret
                                       │
                                       ▼
                              PostgreSQL Connection
                                       │
                                       ▼
                                VPC Networking
                                       │
                                       ▼
                             RDS Security Group
                                       │
                                       ▼
                              Amazon RDS PostgreSQL
                                       │
                     ┌─────────────────┼─────────────────┐
                     │                 │                 │
                   AZ-A              AZ-B              AZ-C
                     │                 │                 │
                     └──────── Multi-AZ ────────────────┘
```

---

# 78. Code-to-Production Flow

Our complete path is now:

```text
Developer
   │
   ▼
GitHub
   │
   ▼
CI
   │
   ├── Test
   ├── SonarQube
   ├── Trivy
   └── Build
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
Order Service
   │
   ▼
AWS Secrets Manager
   │
   ▼
RDS PostgreSQL
```

---

# 79. Production Readiness Checklist

### Infrastructure

```text
[ ] VPC ready
[ ] private subnets ready
[ ] DB subnet group ready
[ ] RDS SG ready
[ ] KMS ready
[ ] RDS provisioned
```

### Security

```text
[ ] no public access
[ ] least-privilege SG
[ ] encrypted storage
[ ] Secrets Manager
[ ] Pod Identity
[ ] application DB user
```

### Application

```text
[ ] DB URL externalized
[ ] credentials externalized
[ ] connection pool configured
[ ] ddl-auto=validate
[ ] migration strategy defined
[ ] graceful DB failure handling
```

### Availability

```text
[ ] Multi-AZ
[ ] automated backups
[ ] PITR
[ ] deletion protection
[ ] failover runbook
```

### Operations

```text
[ ] CloudWatch/RDS metrics
[ ] Prometheus integration where appropriate
[ ] Grafana dashboard
[ ] connection alerts
[ ] storage alerts
[ ] CPU alerts
[ ] backup monitoring
```

---

# 80. Current Architecture Progress

We have now implemented the EKS compute/capacity layer:

```text
100 — Spot Instances
101 — Multi-AZ Workload Placement
102 — Kubernetes Resource Management
103 — HPA / VPA / KEDA
104 — Cluster Autoscaling
```

And now:

```text
105 — RDS Production Implementation
```

The database architecture is:

```text
                    EKS
                     │
                     ▼
              Order Service
                     │
              Pod Identity
                     │
                     ▼
             Secrets Manager
                     │
                     ▼
               DB credentials
                     │
                     ▼
                RDS PostgreSQL
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
      AZ-A         AZ-B         AZ-C
        │            │            │
        └────── Multi-AZ ────────┘
```

---

## Key Takeaways

1. **RDS is the managed production database layer for our platform.**
2. **RDS remains private inside the VPC.**
3. **Only approved EKS application security groups can reach PostgreSQL.**
4. **The application never uses the RDS master account.**
5. **Secrets Manager is the source of truth for database credentials.**
6. **EKS Pod Identity provides AWS access without static AWS keys.**
7. **External Secrets synchronizes required credentials into Kubernetes.**
8. **Database schema changes belong to Flyway/Liquibase, not Hibernate `ddl-auto=update`.**
9. **Connection-pool size must be designed together with HPA/KEDA maximum replicas.**
10. **Multi-AZ provides database high availability; read replicas solve a different problem.**
11. **RDS backups/PITR must be tested, not merely enabled.**
12. **RDS failover must be exercised before calling the production architecture ready.**

### Next chapter

**106 — Database Migration with Flyway/Liquibase**

We will implement the actual database schema-management pipeline:

```text
Developer
   ↓
Migration SQL
   ↓
Flyway/Liquibase
   ↓
CI
   ↓
Test PostgreSQL
   ↓
QA
   ↓
STAGING
   ↓
Production Change
   ↓
RDS PostgreSQL
   ↓
Application Deployment
```

including **versioned migrations, checksum validation, baseline, rollback strategy, expand-and-contract migrations, GitOps integration, CI validation, production ordering, and failure recovery**.
