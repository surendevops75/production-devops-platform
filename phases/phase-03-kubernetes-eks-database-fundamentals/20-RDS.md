# 20-RDS.md

# AWS RDS PostgreSQL — Production Database Architecture

> **Project:** Enterprise E-Commerce Platform
> **Cloud:** AWS
> **Region:** `ap-south-1`
> **Database:** PostgreSQL
> **Platform:** Amazon RDS
> **Consumers:** EKS Microservices

---

## 1. Why RDS Instead of PostgreSQL Inside EKS?

For a production e-commerce platform, we generally **do not run the primary PostgreSQL database as a normal StatefulSet inside EKS**.

Instead:

```text
                    AWS
                     │
        ┌────────────┴────────────┐
        │                         │
       EKS                       RDS
        │                         │
 Microservices              PostgreSQL
        │                         │
        └──────────► Database ◄──┘
```

RDS gives us managed capabilities such as:

* automated backups
* point-in-time recovery
* Multi-AZ failover
* managed patching
* storage management
* monitoring
* encryption
* CloudWatch integration
* database maintenance
* snapshots
* easier operational recovery

If PostgreSQL were deployed ourselves inside EKS, the platform team would have to manage:

```text
PostgreSQL
    ↓
Persistent Volumes
    ↓
Replication
    ↓
Failover
    ↓
Backups
    ↓
Restore
    ↓
Database upgrades
    ↓
Storage expansion
    ↓
Monitoring
    ↓
Operational recovery
```

For this project, we want the DevOps team to focus on the **platform around the database**, while AWS manages much of the underlying database infrastructure.

---

# 2. Our Microservice Database Architecture

We have 21 microservices.

We should avoid creating one giant shared database such as:

```text
                 PostgreSQL
                     │
       ┌─────────────┼─────────────┐
       │             │             │
     Orders       Payments      Users
```

Instead, we follow the **database-per-service ownership model** where practical.

For example:

```text
order-service
      │
      ▼
order database

payment-service
      │
      ▼
payment database

inventory-service
      │
      ▼
inventory database

customer-service
      │
      ▼
customer database
```

The important principle is:

> A microservice owns its data and other services should not directly modify that service's database.

For example:

```text
payment-service
      │
      X
      │
      └──── Cannot directly UPDATE order database
```

Instead:

```text
order-service
      │
      │ event/API
      ▼
payment-service
```

---

# 3. Important Database Design Decision

Database-per-service does **not necessarily mean 21 RDS instances**.

That would often be unnecessarily expensive.

Possible production models include:

### Model 1 — Separate RDS instance

```text
order-service
     │
     ▼
RDS PostgreSQL #1
```

Useful when the service requires:

* strong isolation
* independent scaling
* independent maintenance
* sensitive workload isolation

---

### Model 2 — Shared RDS instance, separate databases

```text
                 RDS PostgreSQL
                       │
        ┌──────────────┼──────────────┐
        │              │              │
     orders         payments       customers
      DB               DB             DB
```

This can be more cost-efficient.

However, teams must maintain strict ownership.

---

### Model 3 — Shared RDS instance + separate schemas

```text
RDS PostgreSQL
      │
      ├── order schema
      ├── payment schema
      ├── customer schema
      └── inventory schema
```

This provides weaker isolation.

For our learning project, we'll demonstrate **independent logical database ownership**, while the exact RDS-instance topology can vary by environment and workload.

---

# 4. Production Architecture

Our target architecture:

```text
                         Internet
                            │
                       CloudFront
                            │
                          WAF
                            │
                          ALB
                            │
                     AWS Load Balancer
                            │
                          EKS
                            │
              ┌─────────────┼─────────────┐
              │             │             │
         order-service payment-service inventory
              │             │             │
              └─────────────┼─────────────┘
                            │
                       Private VPC
                            │
                     RDS PostgreSQL
                       Multi-AZ
                    ┌───────┴───────┐
                    │               │
                  AZ-A            AZ-B
                    │               │
                    └───────┬───────┘
                            │
                         Storage
```

The RDS instance is **not publicly accessible**.

---

# 5. VPC Placement

Our VPC:

```text
10.0.0.0/16
```

Subnets:

```text
Public
├── 10.0.1.0/24   AZ-A
├── 10.0.2.0/24   AZ-B
└── 10.0.3.0/24   AZ-C

Private
├── 10.0.11.0/24  AZ-A
├── 10.0.12.0/24  AZ-B
└── 10.0.13.0/24  AZ-C
```

RDS goes into private subnets.

```text
                    VPC
                     │
       ┌─────────────┴──────────────┐
       │                            │
   Public Subnets              Private Subnets
       │                            │
      ALB                           │
                                    ├── EKS
                                    │
                                    └── RDS
```

---

# 6. DB Subnet Group

RDS needs a DB subnet group containing subnets in multiple Availability Zones.

Example:

```text
RDS DB Subnet Group

AZ-A → 10.0.11.0/24
AZ-B → 10.0.12.0/24
AZ-C → 10.0.13.0/24
```

This gives RDS multiple subnet/AZ placement options.

---

# 7. RDS Multi-AZ

Production should normally use Multi-AZ for critical databases.

Conceptually:

```text
                 RDS
                  │
          ┌───────┴────────┐
          │                │
       AZ-A               AZ-B
     Primary             Standby
          │                │
          └──── failover ──┘
```

Application:

```text
order-service
      │
      ▼
RDS endpoint
      │
      ▼
Primary DB
```

If the primary database becomes unavailable:

```text
Failure
   ↓
RDS detects issue
   ↓
Standby becomes primary
   ↓
RDS endpoint points to new primary
   ↓
Application reconnects
```

The application should therefore use the **RDS endpoint**, not a hardcoded database IP.

---

# 8. Multi-AZ vs Read Replica

These solve different problems.

| Feature                 | Multi-AZ          | Read Replica        |
| ----------------------- | ----------------- | ------------------- |
| Primary purpose         | High availability | Read scaling        |
| Automatic failover      | Yes               | Not primarily       |
| Handles primary failure | Yes               | Can be promoted     |
| Read scaling            | No                | Yes                 |
| Typical use             | Production HA     | Heavy read workload |

Example:

```text
                Application
                    │
             ┌──────┴──────┐
             │             │
           Writes         Reads
             │             │
          Primary      Read Replica
```

For example, `catalog-service` may eventually have substantial read traffic.

---

# 9. RDS Storage

For this project we can use:

```text
Storage type: gp3
```

Why gp3?

* configurable performance
* predictable performance characteristics
* commonly suitable for general-purpose workloads
* storage can be adjusted independently from compute within supported limits

Example:

```text
Allocated storage: 100 GiB
Storage type: gp3
```

Production sizing should come from actual workload measurements.

Do not blindly use:

```text
100 GiB
```

for every service.

---

# 10. Encryption

Production RDS should be encrypted.

Architecture:

```text
Application
    │
    ▼
RDS
    │
    ▼
AWS KMS
```

Encryption should cover:

* storage
* automated backups
* snapshots
* replicas where supported by the chosen architecture

Use a customer-managed KMS key where organizational requirements justify it.

---

# 11. Security Group

RDS should not accept PostgreSQL connections from the entire VPC.

Bad:

```text
0.0.0.0/0 → TCP 5432
```

Never expose PostgreSQL publicly just because an application needs database access.

Better:

```text
EKS workload
     │
     │ TCP 5432
     ▼
RDS Security Group
```

Example:

```text
RDS-SG

Inbound:
TCP 5432
Source: application workload security group
```

The exact source security group design depends on whether we use node-level security groups or AWS Security Groups for Pods.

---

# 12. Database Security Model

Our security chain:

```text
Internet
   │
   X
   │
RDS
```

There should be no direct Internet path.

Instead:

```text
EKS Pod
   │
   │ private VPC
   ▼
RDS
```

Only required workloads should be allowed to connect.

---

# 13. RDS DNS

Suppose AWS provides an endpoint similar to:

```text
order-db.xxxxxxxxxxxx.ap-south-1.rds.amazonaws.com
```

The application connects using the endpoint:

```text
jdbc:postgresql://order-db.xxxxxxxxxxxx.ap-south-1.rds.amazonaws.com:5432/orders
```

Not:

```text
10.0.x.x
```

Why?

Because during failover the underlying infrastructure can change while the managed endpoint remains the application connection target.

---

# 14. Terraform RDS Module

Infrastructure repository:

```text
infrastructure/
└── terraform/
    ├── modules/
    │   └── rds/
    │       ├── main.tf
    │       ├── variables.tf
    │       └── outputs.tf
    │
    └── environments/
        ├── dev/
        ├── qa/
        └── prod/
```

---

# 15. RDS Variables

`modules/rds/variables.tf`

```hcl
variable "identifier" {
  description = "RDS instance identifier"
  type        = string
}

variable "engine" {
  description = "Database engine"
  type        = string
  default     = "postgres"
}

variable "engine_version" {
  description = "PostgreSQL engine version"
  type        = string
}

variable "instance_class" {
  description = "RDS instance class"
  type        = string
}

variable "allocated_storage" {
  description = "Allocated storage in GiB"
  type        = number
}

variable "db_name" {
  description = "Database name"
  type        = string
}

variable "username" {
  description = "Master username"
  type        = string
}

variable "subnet_ids" {
  description = "Private subnet IDs"
  type        = list(string)
}

variable "security_group_ids" {
  description = "Security groups attached to RDS"
  type        = list(string)
}

variable "kms_key_id" {
  description = "KMS key for RDS encryption"
  type        = string
}

variable "environment" {
  description = "Environment name"
  type        = string
}
```

---

# 16. DB Subnet Group

```hcl
resource "aws_db_subnet_group" "this" {
  name = "${var.identifier}-subnet-group"

  subnet_ids = var.subnet_ids

  tags = {
    Name        = "${var.identifier}-subnet-group"
    Environment = var.environment
  }
}
```

Example:

```text
order-db-subnet-group
```

contains:

```text
private-subnet-a
private-subnet-b
private-subnet-c
```

---

# 17. RDS Parameter Group

```hcl
resource "aws_db_parameter_group" "this" {
  name   = "${var.identifier}-params"
  family = "postgresXX"

  tags = {
    Name        = "${var.identifier}-params"
    Environment = var.environment
  }
}
```

Replace `postgresXX` with the actual supported parameter-group family matching the PostgreSQL version selected for the environment.

Parameter groups allow controlled database configuration.

Examples include:

* connection-related settings
* logging
* query behavior
* performance settings

Do not blindly copy random PostgreSQL tuning values from the Internet.

---

# 18. Production RDS Instance

A production-oriented example:

```hcl
resource "aws_db_instance" "this" {
  identifier = var.identifier

  engine         = var.engine
  engine_version = var.engine_version

  instance_class    = var.instance_class
  allocated_storage = var.allocated_storage
  storage_type      = "gp3"

  db_name  = var.db_name
  username = var.username

  manage_master_user_password = true

  db_subnet_group_name   = aws_db_subnet_group.this.name
  vpc_security_group_ids = var.security_group_ids

  parameter_group_name = aws_db_parameter_group.this.name

  multi_az = true

  storage_encrypted = true
  kms_key_id        = var.kms_key_id

  backup_retention_period = 7

  backup_window = "18:00-19:00"

  maintenance_window = "sun:19:00-sun:20:00"

  auto_minor_version_upgrade = true

  deletion_protection = true

  skip_final_snapshot = false

  copy_tags_to_snapshot = true

  publicly_accessible = false

  apply_immediately = false

  tags = {
    Name        = var.identifier
    Environment = var.environment
  }
}
```

### Important

`manage_master_user_password = true` allows AWS to manage the RDS master password through AWS Secrets Manager where supported.

This is preferable to casually putting database passwords into Terraform variables.

---

# 19. Why `manage_master_user_password` Matters

Avoid:

```hcl
password = "MyPassword123"
```

Also be careful with:

```hcl
variable "db_password" {
  type = string
}
```

because sensitive values can still end up in Terraform state depending on how they are used.

A better production pattern is:

```text
RDS
 │
 └── AWS-managed master password
             │
             ▼
      Secrets Manager
```

Then:

```text
Application
    │
    ▼
Secrets integration
    │
    ▼
Secrets Manager
    │
    ▼
Database credentials
```

---

# 20. RDS Outputs

`modules/rds/outputs.tf`

```hcl
output "endpoint" {
  description = "RDS endpoint"
  value       = aws_db_instance.this.address
}

output "port" {
  description = "RDS port"
  value       = aws_db_instance.this.port
}

output "identifier" {
  description = "RDS identifier"
  value       = aws_db_instance.this.identifier
}
```

We can use the endpoint in downstream infrastructure configuration.

---

# 21. Calling the RDS Module

Development:

```hcl
module "order_db" {
  source = "../../modules/rds"

  identifier = "ecommerce-order-db"

  engine         = "postgres"
  engine_version = "16"

  instance_class    = "db.t4g.medium"
  allocated_storage = 100

  db_name  = "orders"
  username = "order_admin"

  subnet_ids = module.vpc.private_subnet_ids

  security_group_ids = [
    aws_security_group.order_db.id
  ]

  kms_key_id = aws_kms_key.rds.arn

  environment = "dev"
}
```

Production sizing should be independently determined.

Do not assume the development instance size is suitable for production.

---

# 22. RDS Security Group

Example:

```hcl
resource "aws_security_group" "order_db" {
  name        = "order-db-sg"
  description = "Security group for order PostgreSQL database"
  vpc_id      = module.vpc.vpc_id

  ingress {
    description     = "PostgreSQL from order-service workload"
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.order_service.id]
  }

  egress {
    description = "Outbound traffic"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "order-db-sg"
  }
}
```

The production implementation should align with the chosen EKS security-group model.

---

# 23. Application → RDS

Our application:

```text
order-service
      │
      │ JDBC
      ▼
RDS PostgreSQL
```

Spring Boot:

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

Example:

```text
DB_URL=
jdbc:postgresql://ecommerce-order-db.xxxxx.ap-south-1.rds.amazonaws.com:5432/orders
```

---

# 24. Never Use `ddl-auto=update` in Production

Earlier local development configuration used:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: update
```

That's acceptable for a local learning environment.

For production:

```yaml
spring:
  jpa:
    hibernate:
      ddl-auto: validate
```

or an equivalent controlled strategy.

Why?

Imagine three production pods:

```text
Pod 1
Pod 2
Pod 3
```

If every application instance tries to modify the database schema automatically during startup, deployments can become unpredictable.

Instead use:

```text
Developer
   │
   ▼
Migration script
   │
   ▼
CI validation
   │
   ▼
Controlled migration
   │
   ▼
RDS
   │
   ▼
Application rollout
```

---

# 25. Flyway / Liquibase

For schema migrations we can use:

* Flyway
* Liquibase

For this project we'll use **Flyway** as the example.

Repository:

```text
src/
└── main/
    └── resources/
        └── db/
            └── migration/
                ├── V1__create_orders_table.sql
                ├── V2__add_order_status.sql
                └── V3__add_customer_index.sql
```

Example:

```sql
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INTEGER NOT NULL,
    status VARCHAR(50) NOT NULL
);
```

---

# 26. Migration Principle

Never treat production database schema as something developers manually edit.

Bad:

```text
Developer laptop
      │
      ▼
Production DB
```

Better:

```text
Git
 │
 ▼
Migration
 │
 ▼
CI validation
 │
 ▼
Approved release
 │
 ▼
Production migration
```

This creates an audit trail.

---

# 27. Migration With Multiple Pods

Suppose we deploy:

```text
order-service
├── pod-1
├── pod-2
├── pod-3
└── pod-4
```

Do not blindly allow all four pods to execute migrations independently.

Better:

```text
Release
  │
  ▼
Migration Job
  │
  ▼
RDS
  │
  ▼
Migration successful
  │
  ▼
Application rollout
```

Or use a migration mechanism with proper locking/coordination.

The exact pattern depends on the migration tool and deployment architecture.

---

# 28. Connection Pooling

Spring Boot uses HikariCP by default in typical Spring Boot JDBC setups.

Instead of every request creating a new database connection:

```text
Request
   ↓
Create DB connection
   ↓
Query
   ↓
Close connection
```

we use a connection pool:

```text
             HikariCP
          ┌─────────────┐
Request → │ Connection  │
Request → │ Connection  │ → RDS
Request → │ Connection  │
Request → │ Connection  │
          └─────────────┘
```

This reduces connection creation overhead.

---

# 29. Hikari Configuration

Example:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
      connection-timeout: 30000
      idle-timeout: 600000
      max-lifetime: 1800000
```

These are **example values**, not universal production settings.

---

# 30. Why Connection Pool Size Is Critical

Suppose:

```text
10 Pods
```

and each pod has:

```text
maximumPoolSize = 20
```

Potential connections:

```text
10 × 20 = 200
```

Now imagine:

```text
20 Pods × 30 connections
```

That's:

```text
600 connections
```

The database may not support that safely.

Therefore:

> Connection-pool sizing must be considered at the **cluster level**, not only at the application-pod level.

---

# 31. Hikari + HPA Interaction

This is a very important production problem.

Suppose:

```text
Current pods = 3
Pool per pod = 20
```

Maximum connections:

```text
3 × 20 = 60
```

HPA scales to:

```text
10 pods
```

Now:

```text
10 × 20 = 200
```

The application scaled successfully.

But the database may now become overloaded.

```text
                    HPA
                     │
                     ▼
              More application pods
                     │
                     ▼
              More DB connections
                     │
                     ▼
                RDS overload
```

This is why autoscaling must consider downstream dependencies.

---

# 32. RDS Proxy

For some architectures, AWS RDS Proxy can help manage database connections.

Conceptually:

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
RDS
```

Instead of every application connection mapping directly to a database connection, the proxy can help manage connection multiplexing/pooling and improve resilience for certain workloads.

However:

> Do not add RDS Proxy automatically just because it exists.

For a normal EKS + Spring Boot workload, start with properly sized HikariCP and measure.

Consider RDS Proxy when the workload has characteristics such as:

* very bursty connection patterns
* large numbers of clients
* serverless workloads
* frequent connection establishment
* specific failover/connection-management requirements

---

# 33. RDS Monitoring

RDS integrates with CloudWatch.

Important metrics include:

```text
CPUUtilization
FreeStorageSpace
FreeableMemory
DatabaseConnections
ReadIOPS
WriteIOPS
ReadLatency
WriteLatency
NetworkReceiveThroughput
NetworkTransmitThroughput
```

Depending on workload, additional database-specific monitoring is required.

---

# 34. Production Dashboard

Our monitoring architecture eventually becomes:

```text
                       RDS
                        │
                 CloudWatch Metrics
                        │
                        ▼
                  Monitoring Layer
                        │
                 ┌──────┴──────┐
                 │             │
              Grafana      Alerting
```

For AWS-native infrastructure metrics, CloudWatch remains important even if we use Prometheus/Grafana for Kubernetes observability.

---

# 35. Database Alerts

Example alerts:

### CPU

```text
RDS CPU > 80%
for 10 minutes
```

### Storage

```text
FreeStorageSpace below threshold
```

### Connections

```text
DatabaseConnections approaching safe maximum
```

### Memory

```text
FreeableMemory continuously low
```

### Latency

```text
ReadLatency / WriteLatency elevated
```

Thresholds should be based on the actual workload baseline rather than arbitrary numbers.

---

# 36. PostgreSQL Logs

RDS PostgreSQL logs can be integrated with CloudWatch Logs.

Conceptually:

```text
PostgreSQL
    │
    ▼
RDS
    │
    ▼
CloudWatch Logs
```

This helps investigate:

* connection failures
* authentication errors
* slow queries
* database warnings
* restart events
* operational problems

We will later connect the broader observability architecture.

---

# 37. RDS Backup

Production database:

```text
RDS
 │
 ├── Automated backups
 │
 ├── Point-in-time recovery
 │
 └── Snapshots
```

Example retention:

```text
7 days
```

But production may require:

```text
7
14
30
35
```

days or another policy depending on:

* business requirements
* compliance
* RPO
* recovery strategy.

---

# 38. Point-in-Time Recovery

Suppose:

```text
10:00 → normal
11:00 → normal
12:00 → accidental DELETE
```

We don't necessarily want only the last snapshot.

With point-in-time recovery:

```text
Backup history
      │
      ├── 10:00
      ├── 10:30
      ├── 11:00
      ├── 11:30
      └── 12:00
```

We can restore to an appropriate supported point in time.

---

# 39. Backup Is Not Enough

This is a major production lesson:

> A backup that has never been restored is only an assumption.

We need restore testing.

Process:

```text
Production RDS
      │
      ▼
Snapshot / PITR
      │
      ▼
Restore temporary database
      │
      ▼
Run validation
      │
      ▼
Verify application connectivity
      │
      ▼
Record recovery time
```

This validates the actual recovery procedure.

---

# 40. RPO and RTO

Two important concepts:

### RPO

Recovery Point Objective:

> How much data loss can the business tolerate?

Example:

```text
RPO = 5 minutes
```

means the organization cannot accept losing more than roughly five minutes of data.

### RTO

Recovery Time Objective:

> How quickly must the service be restored?

Example:

```text
RTO = 30 minutes
```

These values drive the backup/DR architecture.

---

# 41. RDS Deletion Protection

Production:

```hcl
deletion_protection = true
```

This prevents accidental deletion of the database through normal deletion operations until protection is disabled.

Development may use:

```hcl
deletion_protection = false
```

depending on lifecycle requirements.

---

# 42. Terraform Production Safety

Our infrastructure pipeline should be:

```text
Developer
   │
   ▼
Terraform code
   │
   ▼
Pull Request
   │
   ▼
terraform fmt
   │
   ▼
terraform validate
   │
   ▼
terraform plan
   │
   ▼
Review
   │
   ▼
Approval
   │
   ▼
Terraform apply
```

Production should not be:

```bash
terraform apply -auto-approve
```

from someone's laptop.

---

# 43. RDS Terraform State

Remember:

```text
Terraform
    │
    ▼
S3 State
    │
    ├── Encryption
    ├── Versioning
    └── Locking
```

The state file is sensitive infrastructure data.

Protect it using:

* restrictive IAM
* encryption
* controlled access
* versioning
* state locking
* separate environment/account boundaries where appropriate.

---

# 44. Production Database Credentials

We want:

```text
GitHub
   X
   │
   └── no DB password

Git repository
   X
   │
   └── no DB password

Docker image
   X
   │
   └── no DB password

Helm values
   X
   │
   └── no plaintext DB password
```

Instead:

```text
AWS Secrets Manager
        │
        ▼
EKS workload identity
        │
        ▼
order-service
```

We will implement this properly in:

```text
23-Secrets-Management.md
```

---

# 45. Local vs Production Database

### Local

```text
Docker Compose
     │
     ▼
PostgreSQL container
     │
     ▼
order-service
```

### Production

```text
EKS
 │
 ▼
order-service
 │
 ▼
Private VPC
 │
 ▼
RDS PostgreSQL
```

This is an important distinction.

We don't need to install PostgreSQL manually on every EKS node.

---

# 46. Where Does `psql` Run?

`psql` is the PostgreSQL client.

It can be used from:

```text
Developer laptop
```

or:

```text
Controlled admin/operations environment
```

or:

```text
Temporary troubleshooting container
```

depending on the organization's security model.

We should **not** install `psql permanently into every application container**.

Application image:

```text
Spring Boot
Java
Application dependencies
```

not:

```text
Spring Boot
Java
psql
kubectl
aws-cli
terraform
docker
```

Keep containers focused.

---

# 47. Production Access Path

A controlled operational path could look like:

```text
Engineer
   │
   ▼
Approved access
   │
   ▼
Private network / bastion / SSM / controlled admin path
   │
   ▼
psql
   │
   ▼
RDS
```

The exact access mechanism depends on the organization's security architecture.

Never make RDS public just to make troubleshooting easier.

---

# 48. RDS Troubleshooting — Connection Timeout

Application:

```text
Connection timed out
```

Do not immediately assume PostgreSQL is down.

Check layers.

### Layer 1 — DNS

```bash
nslookup <rds-endpoint>
```

### Layer 2 — Network

Can the workload reach the RDS subnet?

### Layer 3 — Security Group

Is TCP:

```text
5432
```

allowed?

### Layer 4 — Route

Does the VPC routing support the connection?

### Layer 5 — RDS

Is the DB instance available?

### Layer 6 — Application

Is the JDBC URL correct?

---

# 49. Authentication Failure

Example:

```text
password authentication failed
```

Check:

```text
DB_USERNAME
DB_PASSWORD
DB_URL
Secret version
Secret rotation
Database user
```

Don't immediately change the database password manually.

Trace:

```text
Pod
 ↓
Secret
 ↓
Environment/configuration
 ↓
JDBC
 ↓
RDS
```

---

# 50. Too Many Connections

Error:

```text
too many connections for role
```

Investigate:

```text
Number of pods
       ↓
Hikari max pool
       ↓
Total possible connections
       ↓
RDS max connections
```

Calculate:

```text
Pods × pool-size
```

Example:

```text
10 × 20 = 200
```

Then compare with the actual safe database capacity.

---

# 51. Slow Database Query

Application symptom:

```text
API response = 5 seconds
```

Don't immediately increase application replicas.

Investigate:

```text
API
 ↓
Application
 ↓
JDBC
 ↓
PostgreSQL
 ↓
Query
```

Look at:

* query execution time
* locks
* indexes
* execution plans
* CPU
* I/O
* connection pool wait
* database load

PostgreSQL tools such as:

```sql
SELECT *
FROM pg_stat_activity;
```

can help inspect active sessions.

---

# 52. `EXPLAIN ANALYZE`

For controlled troubleshooting:

```sql
EXPLAIN ANALYZE
SELECT *
FROM orders
WHERE customer_id = 1001;
```

This can reveal whether PostgreSQL is:

```text
Index Scan
```

or:

```text
Sequential Scan
```

But use `EXPLAIN ANALYZE` carefully on production queries because it actually executes the query.

---

# 53. Database Locks

A query may appear slow because another transaction is holding a lock.

Conceptually:

```text
Transaction A
      │
      ▼
Lock
      │
      X
Transaction B
```

The application may report:

```text
timeout
```

while the real root cause is:

```text
database lock contention
```

This is why production troubleshooting requires understanding PostgreSQL itself, not just AWS.

---

# 54. Deadlocks

Example:

```text
Transaction A
  locks Row 1
  waits for Row 2

Transaction B
  locks Row 2
  waits for Row 1
```

Result:

```text
Deadlock
```

PostgreSQL can detect and abort one transaction.

The application must be designed to handle appropriate transient failures and retry safely where appropriate.

---

# 55. RDS Failover

During a Multi-AZ failover:

```text
Application
     │
     ▼
RDS endpoint
     │
     X
Primary failure
     │
     ▼
Standby
     │
     ▼
New primary
```

Applications may see transient connection failures.

Therefore:

* connection pools need sensible settings
* applications should handle transient DB failures appropriately
* timeouts must be configured
* retry logic must be safe
* transactions should not be blindly retried if doing so can duplicate business operations

For an order/payment system, **idempotency** is especially important.

---

# 56. Database and Payment Operations

Consider:

```text
Create Order
    ↓
Charge Payment
```

Suppose payment succeeds but the application loses the connection before receiving the response.

If we blindly retry:

```text
Charge Payment
Charge Payment
```

we could charge the customer twice.

This is why database reliability is connected to:

* idempotency
* transactions
* event-driven architecture
* payment design
* distributed systems

We'll address these more deeply when we build SQS/SNS and the application architecture.

---

# 57. RDS Maintenance

Production RDS requires maintenance planning.

Examples:

* minor engine upgrades
* OS maintenance
* database parameter changes
* storage changes
* instance resizing

We should define:

```text
Maintenance Window
```

Example:

```text
Sunday
19:00–20:00
```

The actual production window should be chosen based on business traffic.

---

# 58. Major PostgreSQL Upgrades

Major upgrades require much more planning than minor upgrades.

Typical process:

```text
Current PostgreSQL
       │
       ▼
Compatibility analysis
       │
       ▼
Application testing
       │
       ▼
Migration testing
       │
       ▼
Performance testing
       │
       ▼
Backup verification
       │
       ▼
Production upgrade
       │
       ▼
Validation
```

Never treat a major database upgrade as:

```bash
terraform apply
```

and hope everything works.

---

# 59. RDS Production Architecture Summary

```text
                    AWS VPC
                       │
        ┌──────────────┴──────────────┐
        │                             │
   Public Subnets                Private Subnets
        │                             │
       ALB                            │
                                      │
                                ┌─────┴─────┐
                                │           │
                               EKS         RDS
                                │        Multi-AZ
                                │           │
                                │       PostgreSQL
                                │           │
                                └───────────┘
```

Security:

```text
Internet
   │
   X
 RDS
```

Only:

```text
Authorized EKS workload
          │
          ▼
       TCP 5432
          │
          ▼
          RDS
```

---

# 60. Complete Order-Service Database Flow

The complete production request path eventually becomes:

```text
Customer
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
order-service
   │
   ├── Redis
   │
   ├── SQS
   │
   └── PostgreSQL
          │
          ▼
         RDS
```

And credentials:

```text
order-service
      │
      ▼
EKS workload identity
      │
      ▼
Secrets Manager
      │
      ▼
Database credentials
      │
      ▼
RDS
```

---

# 61. What Runs Where?

This directly connects to your earlier question about **where tools/services are actually installed in production**.

| Component         | Production Location                               |
| ----------------- | ------------------------------------------------- |
| PostgreSQL        | AWS RDS                                           |
| PostgreSQL server | Managed by AWS                                    |
| `psql` client     | Controlled admin/ops environment when needed      |
| RDS monitoring    | AWS CloudWatch + observability stack              |
| Terraform         | CI runner / controlled infrastructure environment |
| Trivy             | CI runner                                         |
| Sonar Scanner     | CI runner                                         |
| SonarQube         | Centralized platform service                      |
| Docker build      | CI/build infrastructure                           |
| ECR               | AWS managed service                               |
| ArgoCD            | EKS platform namespace                            |
| Helm              | CI/GitOps tooling                                 |
| Prometheus        | EKS monitoring namespace                          |
| Grafana           | EKS monitoring namespace                          |
| Fluent Bit        | EKS nodes as DaemonSet                            |
| Logstash          | EKS logging namespace                             |
| Elasticsearch     | EKS logging platform or managed service           |
| OpenTelemetry     | EKS/application observability layer               |
| Jaeger            | EKS tracing platform or managed service           |

**The important production mindset is:**

> We don't install every DevOps tool inside the production application cluster.

Instead, each component lives where its responsibility belongs.

---

# 62. Production RDS Checklist

Before calling the database production-ready:

### Networking

* [ ] Private subnets
* [ ] Multi-AZ subnet group
* [ ] `publicly_accessible = false`
* [ ] Security group restricted to required workloads
* [ ] No `0.0.0.0/0` on port 5432

### Security

* [ ] Encryption enabled
* [ ] KMS configured
* [ ] Secrets Manager used
* [ ] No database passwords in Git
* [ ] No passwords in Docker images
* [ ] Least-privilege IAM

### Availability

* [ ] Multi-AZ
* [ ] Failover tested
* [ ] Connection retry strategy
* [ ] Appropriate connection pool sizing

### Backup

* [ ] Automated backups enabled
* [ ] Backup retention configured
* [ ] PITR understood
* [ ] Snapshot strategy defined
* [ ] Restore test performed

### Database operations

* [ ] Flyway/Liquibase migrations
* [ ] `ddl-auto=update` disabled
* [ ] Maintenance window defined
* [ ] Upgrade strategy defined
* [ ] Query monitoring enabled

### Monitoring

* [ ] CPU
* [ ] Storage
* [ ] Memory
* [ ] Connections
* [ ] IOPS
* [ ] Latency
* [ ] Logs
* [ ] Alerts

---

# 63. Final Mental Model

When an engineer says:

> "Order service cannot connect to PostgreSQL."

Don't immediately restart the pod.

Think:

```text
Is the Pod healthy?
        ↓
Is DNS resolving?
        ↓
Can the Pod reach RDS?
        ↓
Does the Security Group allow 5432?
        ↓
Is the route correct?
        ↓
Is RDS available?
        ↓
Are credentials correct?
        ↓
Is the connection pool exhausted?
        ↓
Are RDS connections exhausted?
        ↓
Are there locks?
        ↓
Is PostgreSQL overloaded?
```

That is the **production DevOps mindset**.

You are not just learning:

```text
Terraform syntax
RDS syntax
Kubernetes syntax
```

You are learning how:

```text
Application
     ↓
Network
     ↓
Security
     ↓
AWS
     ↓
Database
     ↓
Monitoring
     ↓
Recovery
```

all work together in production.

---

## Next Chapter

### `21-Redis.md`

We'll add Redis to the same production architecture and cover:

```text
EKS
 │
 ├── order-service
 ├── catalog-service
 ├── product-service
 └── ...
          │
          ▼
       Redis
          │
          ├── Cache
          ├── Session/state use cases
          ├── Hot data
          ├── Rate limiting
          └── Performance optimization
```

including **ElastiCache/Redis architecture, private networking, replication, Multi-AZ, failover, Terraform, security groups, authentication, Spring Boot integration, cache strategies, TTL, eviction, cache stampede, HPA implications, monitoring, and production troubleshooting**.
