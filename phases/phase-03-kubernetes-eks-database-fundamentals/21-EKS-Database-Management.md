Absolutely. Before Redis, we should add a dedicated chapter showing **how databases are actually deployed and operated inside EKS**, just like an application workload.

And you're also right about PostgreSQL: **PostgreSQL is only one option.** A real DevOps engineer should understand how the deployment model changes for PostgreSQL, MySQL, MongoDB, and SQL Server.

I would add this as:

# `20-EKS-Database-Management.md`

Then renumber the previous RDS chapter to:

# `21-RDS.md`

and Redis becomes:

# `22-Redis.md`

Below is the complete chapter.

---

# 20-EKS-Database-Management.md

# Running Databases Inside Amazon EKS

> **Project:** Enterprise E-Commerce Platform
> **Platform:** Amazon EKS
> **Purpose:** Understand how stateful databases are deployed, stored, secured, monitored, backed up, upgraded, and operated inside Kubernetes.

---

# 1. Why Learn Databases Inside EKS?

Our primary production architecture uses managed AWS databases such as RDS.

But a DevOps engineer should also understand:

```text
Application in EKS
        │
        ▼
Database in EKS
```

because many organizations run stateful workloads on Kubernetes.

You may encounter:

* PostgreSQL
* MySQL
* MongoDB
* Microsoft SQL Server
* MariaDB
* Cassandra
* Redis
* Kafka
* Elasticsearch/OpenSearch

inside Kubernetes.

The important thing is not memorizing a specific database.

The important thing is understanding:

```text
Application
    ↓
Service
    ↓
Database workload
    ↓
Persistent storage
    ↓
Backup
    ↓
Replication
    ↓
Recovery
    ↓
Monitoring
```

---

# 2. Stateless vs Stateful Workloads

This is the foundation.

### Stateless application

```text
order-service
    │
    ├── Pod 1
    ├── Pod 2
    └── Pod 3
```

If Pod 1 dies:

```text
Pod 1
  X

Kubernetes
   ↓
New Pod
```

The application can continue.

Why?

Because important state is outside the container.

---

### Stateful database

```text
PostgreSQL
    │
    ▼
orders data
```

If the pod dies:

```text
PostgreSQL Pod
      X
      │
      ▼
New PostgreSQL Pod
```

The data **must still exist**.

Therefore:

```text
Database Pod
     │
     ▼
PersistentVolumeClaim
     │
     ▼
PersistentVolume
     │
     ▼
Storage
```

---

# 3. The Kubernetes Database Stack

A database inside EKS typically involves multiple Kubernetes resources.

```text
                    Database
                       │
                  StatefulSet
                       │
                  ┌────┴────┐
                  │         │
                 Pod       Pod
                  │
                PVC
                  │
                 PV
                  │
            StorageClass
                  │
                  ▼
                 EBS
```

And networking:

```text
Application Pod
      │
      ▼
Kubernetes Service
      │
      ▼
Database Pod
```

And security:

```text
NetworkPolicy
     +
RBAC
     +
Secrets
     +
AWS IAM
```

And operations:

```text
Backup
Restore
Monitoring
Replication
Failover
Upgrades
```

---

# 4. Database Deployment Is Not Just a YAML File

A beginner may think:

```text
Create database Deployment
        ↓
Expose Service
        ↓
Done
```

Production doesn't work like that.

For a production database we need to answer:

```text
Where is the data?

What happens when the Pod dies?

What happens when the Node dies?

What happens when the AZ dies?

How is replication handled?

How does failover happen?

How are backups created?

How is PITR implemented?

How are credentials managed?

How are upgrades performed?

How do applications find the primary?

How do we monitor database health?

How do we restore the database?
```

This is why databases are much more complicated than normal applications.

---

# 5. Why StatefulSet?

For a stateful workload, Kubernetes provides:

```text
StatefulSet
```

Example:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
  namespace: database
spec:
  serviceName: mysql
  replicas: 1

  selector:
    matchLabels:
      app: mysql

  template:
    metadata:
      labels:
        app: mysql

    spec:
      containers:
        - name: mysql
          image: mysql:8.4

          ports:
            - containerPort: 3306

          env:
            - name: MYSQL_DATABASE
              value: ecommerce

          volumeMounts:
            - name: mysql-data
              mountPath: /var/lib/mysql

  volumeClaimTemplates:
    - metadata:
        name: mysql-data

      spec:
        accessModes:
          - ReadWriteOnce

        resources:
          requests:
            storage: 100Gi
```

---

# 6. StatefulSet vs Deployment

### Deployment

Usually:

```text
Deployment
    │
    ├── Pod
    ├── Pod
    └── Pod
```

Pods are replaceable.

---

### StatefulSet

```text
StatefulSet
    │
    ├── mysql-0
    ├── mysql-1
    └── mysql-2
```

Each pod has stable identity.

For example:

```text
mysql-0
mysql-1
mysql-2
```

If `mysql-1` dies, Kubernetes recreates:

```text
mysql-1
```

rather than an arbitrary identity.

---

# 7. StatefulSet Does Not Provide Database HA

This deserves repetition.

Suppose:

```text
mysql-0
mysql-1
mysql-2
```

You do **not** automatically have:

```text
HA MySQL
```

Kubernetes knows:

```text
Pod exists
Pod doesn't exist
```

But MySQL needs to know:

```text
Which node is primary?

Which nodes are replicas?

Is replication healthy?

Which replica should become primary?

How do we prevent split brain?

```

Therefore:

```text
StatefulSet
       +
Database replication
       +
Database-aware failover
```

are separate concerns.

---

# 8. PersistentVolumeClaim

The database requests storage using a PVC.

Example:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-data
  namespace: database
spec:
  accessModes:
    - ReadWriteOnce

  resources:
    requests:
      storage: 100Gi
```

The application doesn't directly manage the physical disk.

Instead:

```text
Database
   ↓
PVC
   ↓
PV
   ↓
StorageClass
   ↓
EBS
```

---

# 9. EKS StorageClass

In AWS EKS, EBS volumes can be dynamically provisioned using the **Amazon EBS CSI driver**.

Conceptually:

```text
PVC
 │
 ▼
StorageClass
 │
 ▼
EBS CSI Driver
 │
 ▼
AWS EBS Volume
```

Example:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com

parameters:
  type: gp3
  encrypted: "true"

volumeBindingMode: WaitForFirstConsumer

allowVolumeExpansion: true
```

The exact StorageClass configuration should follow the EKS/EBS CSI driver version and organization's storage standards.

---

# 10. Why `WaitForFirstConsumer`?

This is particularly important with zonal storage such as EBS.

Suppose:

```text
EBS volume
   │
   ▼
AZ-A
```

but Kubernetes schedules the database pod to:

```text
AZ-B
```

You have a placement problem.

With:

```yaml
volumeBindingMode: WaitForFirstConsumer
```

Kubernetes can consider the Pod's scheduling constraints before dynamically provisioning the volume.

Conceptually:

```text
Pod scheduling decision
        ↓
AZ selected
        ↓
EBS volume provisioned
        ↓
Pod starts
```

This helps avoid incorrect AZ/storage placement.

---

# 11. EBS Is AZ-Scoped

This is one of the most important AWS/Kubernetes concepts.

Suppose:

```text
AZ-A
 │
 └── postgres-0
       │
       └── EBS-A
```

If the pod moves to AZ-B, EBS-A cannot simply be treated as a generic multi-AZ filesystem.

Therefore:

```text
Database
   +
EBS
   +
AZ
```

must be considered together.

---

# 12. One Database Pod

The simplest architecture:

```text
             EKS
              │
         StatefulSet
              │
          database-0
              │
             PVC
              │
             EBS
```

Advantages:

* simple
* cheap
* easy to understand

Disadvantages:

* single database instance
* downtime during certain failures
* no database-level HA
* AZ failure can affect availability
* backup/recovery becomes your responsibility

This is suitable for:

* development
* testing
* temporary environments
* learning

but generally isn't enough for a critical production database.

---

# 13. Three Database Pods

Now:

```text
EKS
 │
 ├── db-0
 ├── db-1
 └── db-2
```

You may place them across:

```text
AZ-A
AZ-B
AZ-C
```

But again:

```text
3 Pods
≠
3 database replicas
```

You need database replication.

---

# 14. Database Replication

For a relational database:

```text
Primary
   │
   ├── Replication
   ▼
Replica 1

Primary
   │
   └── Replication
       ▼
     Replica 2
```

For PostgreSQL, this can involve WAL-based replication.

For MySQL, replication mechanisms differ.

For MongoDB, replica sets provide database-level replication.

Therefore, Kubernetes provides the infrastructure primitives, while the **database engine provides its own replication model**.

---

# 15. Database Operators

Manually managing all of this is difficult.

That's where Kubernetes operators come in.

An operator essentially extends Kubernetes with database-specific operational knowledge.

Conceptually:

```text
Kubernetes
    │
    ▼
Database Operator
    │
    ├── Deployment
    ├── Replication
    ├── Failover
    ├── Backup
    ├── Restore
    ├── Upgrade
    └── Recovery
```

Examples:

### PostgreSQL

* CloudNativePG
* Crunchy Postgres for Kubernetes

### MySQL

* MySQL Operator for Kubernetes

### MongoDB

* MongoDB Kubernetes Operator

### Microsoft SQL Server

SQL Server can run as a containerized workload, but its Kubernetes production architecture, licensing, storage, HA, and support model need careful evaluation. Don't assume that a generic StatefulSet is equivalent to an enterprise SQL Server HA deployment.

---

# 16. Operator Mental Model

Without an operator:

```text
DevOps engineer
     │
     ├── create Pod
     ├── configure replication
     ├── detect failure
     ├── promote replica
     ├── repair cluster
     └── manage backup
```

With an operator:

```text
DevOps engineer
       │
       ▼
Desired database configuration
       │
       ▼
Operator
       │
       ├── database lifecycle
       ├── replication
       ├── failover
       └── recovery
```

The operator doesn't eliminate operational responsibility.

It automates database-specific operations.

---

# 17. Database Custom Resource

An operator often gives you a Kubernetes Custom Resource.

Conceptually:

```yaml
apiVersion: database.example.com/v1
kind: DatabaseCluster

metadata:
  name: ecommerce-db

spec:
  replicas: 3

  storage:
    size: 100Gi

  backup:
    enabled: true
```

The operator watches this resource.

```text
CR
 │
 ▼
Operator
 │
 ├── Pods
 ├── Services
 ├── PVCs
 ├── Replication
 └── Backup
```

The exact YAML depends entirely on the chosen operator.

---

# 18. Database Namespace

We should isolate databases.

```bash
kubectl create namespace database
```

Then:

```text
EKS
│
├── production
│   ├── order-service
│   ├── payment-service
│   └── catalog-service
│
├── database
│   ├── PostgreSQL
│   ├── MySQL
│   └── MongoDB
│
├── monitoring
├── logging
└── argocd
```

However, whether all databases share one namespace or each database gets its own namespace depends on organizational isolation requirements.

---

# 19. Database Network Access

Application:

```text
order-service
```

Database:

```text
postgres
```

We create a ClusterIP service:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: postgres
  namespace: database

spec:
  type: ClusterIP

  selector:
    app: postgres

  ports:
    - port: 5432
      targetPort: 5432
```

Now:

```text
order-service
      │
      ▼
postgres.database.svc.cluster.local
      │
      ▼
PostgreSQL
```

---

# 20. NetworkPolicy

We don't want every application to access every database.

Example:

```text
order-service
       │
       ▼
order-db

payment-service
       X
       │
       └── Cannot access order-db
```

NetworkPolicy can enforce this.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:
  name: postgres-ingress
  namespace: database

spec:
  podSelector:
    matchLabels:
      app: postgres

  policyTypes:
    - Ingress

  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: production

          podSelector:
            matchLabels:
              app: order-service

      ports:
        - protocol: TCP
          port: 5432
```

The exact namespace-label strategy must be implemented consistently.

---

# 21. Database Credentials

Never:

```yaml
env:
  - name: DB_PASSWORD
    value: "Password123"
```

inside production manifests.

Instead:

```text
AWS Secrets Manager
        │
        ▼
External Secrets / approved secret integration
        │
        ▼
Kubernetes Secret
        │
        ▼
Database / Application
```

Or use another approved secret-management architecture.

We'll cover the complete implementation later in:

```text
23-Secrets-Management.md
```

---

# 22. Database Configuration

Don't put everything into environment variables.

For example:

```text
DB_HOST
DB_PORT
DB_NAME
DB_USER
DB_PASSWORD
```

may be appropriate for connection information.

But database configuration such as:

```text
max_connections
shared_buffers
work_mem
logging
```

belongs to the database configuration/parameter mechanism appropriate for that engine.

---

# 23. Database Resource Requests

Database workloads need explicit resources.

Example:

```yaml
resources:
  requests:
    cpu: "2"
    memory: "8Gi"

  limits:
    cpu: "4"
    memory: "16Gi"
```

Don't randomly choose these numbers.

Measure:

```text
CPU
Memory
IOPS
Throughput
Connections
Query latency
```

and size accordingly.

---

# 24. Database Nodes

For serious workloads, consider dedicated nodes.

```text
EKS
│
├── application-node-group
│   ├── order-service
│   ├── payment-service
│   └── catalog-service
│
└── database-node-group
    ├── postgres-0
    ├── postgres-1
    └── postgres-2
```

Database nodes can use:

* taints
* tolerations
* node affinity
* topology spread constraints

to keep workloads appropriately separated.

---

# 25. Taint Database Nodes

Example:

```bash
kubectl taint nodes \
  database-node-1 \
  workload=database:NoSchedule
```

Then the database pods need:

```yaml
tolerations:
  - key: "workload"
    operator: "Equal"
    value: "database"
    effect: "NoSchedule"
```

This prevents normal application workloads from being scheduled onto dedicated database nodes.

---

# 26. Topology Spread

We don't want:

```text
AZ-A
 ├── postgres-0
 ├── postgres-1
 └── postgres-2
```

Better:

```text
AZ-A → postgres-0
AZ-B → postgres-1
AZ-C → postgres-2
```

This is critical for failure isolation.

The database operator should also understand topology and replication.

---

# 27. Pod Anti-Affinity

For replicas:

```yaml
affinity:
  podAntiAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: postgres

        topologyKey: topology.kubernetes.io/zone
```

The exact rule should be carefully designed because hard scheduling requirements can also prevent workloads from starting if the cluster lacks capacity.

---

# 28. Database Probes

Application probes:

```text
/actuator/health
```

are not sufficient for databases.

A database needs meaningful health checks.

For example:

```text
Is PostgreSQL process running?
        ↓
Can PostgreSQL accept connections?
        ↓
Is this node primary/replica?
        ↓
Is replication healthy?
        ↓
Is storage available?
```

Operators typically provide database-aware health management.

---

# 29. Liveness vs Readiness

This distinction matters.

### Liveness

> Is the database process fundamentally alive?

### Readiness

> Can this database instance safely serve the role it is supposed to serve?

For a replicated database:

```text
postgres-0
    │
    ├── process alive
    └── replication broken
```

It may be alive but not healthy enough to serve traffic.

Database operators handle much of this logic better than a simple TCP probe.

---

# 30. Backup Architecture

For a Kubernetes database, design backups independently.

Example:

```text
PostgreSQL
    │
    ├── Base backup
    │
    └── WAL
          │
          ▼
          S3
```

Now:

```text
Database
    │
    ▼
Backup system
    │
    ▼
S3
    │
    ├── encryption
    ├── lifecycle
    └── retention
```

Don't depend only on EBS snapshots.

---

# 31. EBS Snapshot vs Database Backup

These are not identical.

### Storage snapshot

```text
EBS
 ↓
Snapshot
```

captures storage state.

### Database backup

```text
PostgreSQL
 ↓
Consistent database backup
 ↓
WAL/PITR
```

A database-aware backup solution understands:

* transactions
* WAL
* consistency
* recovery

Therefore:

> Storage snapshots can be useful, but they should not automatically be considered a complete database backup strategy.

---

# 32. Restore

A production restore process should look like:

```text
Backup
   │
   ▼
Restore temporary database
   │
   ▼
Validate
   │
   ├── tables
   ├── indexes
   ├── data
   └── application queries
   │
   ▼
Recovery approved
```

Don't wait for a disaster to discover that your backup cannot be restored.

---

# 33. Point-in-Time Recovery

A mature database platform should support something like:

```text
10:00
10:30
11:00
11:30
12:00
```

and allow recovery to an appropriate point.

This usually requires:

```text
Base backup
+
WAL/binlogs/oplogs depending on database
```

For example:

### PostgreSQL

```text
Base backup + WAL
```

### MySQL

```text
Backup + binary logs
```

### MongoDB

```text
Backup + oplog
```

The exact mechanism is database-specific.

---

# 34. MySQL Inside EKS

Yes — MySQL can absolutely be used.

Architecture:

```text
Application
     │
     ▼
MySQL Service
     │
     ▼
MySQL StatefulSet / Operator
     │
     ▼
PVC
     │
     ▼
EBS
```

Production HA:

```text
MySQL Primary
      │
      ├── Replica
      └── Replica
```

The replication/failover mechanism is MySQL-specific.

---

# 35. PostgreSQL Inside EKS

PostgreSQL:

```text
Application
     │
     ▼
PostgreSQL Service
     │
     ▼
PostgreSQL Operator
     │
 ┌───┼────┐
 ▼   ▼    ▼
P   R    R
```

Where:

```text
P = Primary
R = Replica
```

The operator handles database-aware lifecycle operations.

---

# 36. MongoDB Inside EKS

MongoDB uses a different architecture.

Instead of PostgreSQL primary/standby concepts, MongoDB commonly uses:

```text
Replica Set
```

Conceptually:

```text
MongoDB Replica Set
│
├── Primary
├── Secondary
└── Secondary
```

Application:

```text
MongoDB client
       │
       ▼
Replica Set
```

MongoDB's replication and election mechanisms are different from PostgreSQL and MySQL.

Therefore, you cannot simply replace:

```text
postgres
```

with:

```text
mongo
```

in the same architecture and assume everything else remains identical.

---

# 37. Microsoft SQL Server Inside EKS

SQL Server can run in containers/Kubernetes, but it needs additional consideration around:

* licensing
* supported deployment architecture
* persistent storage
* HA/DR
* SQL Server-specific clustering technologies
* backup
* maintenance
* resource requirements
* vendor support

For enterprise SQL Server, you should verify the exact Kubernetes architecture against Microsoft's current support matrix and your licensing model.

Don't build production SQL Server HA merely from a generic Kubernetes StatefulSet.

---

# 38. Same Kubernetes Layer, Different Database Layer

This is the key concept.

The Kubernetes layer can look like:

```text
             Kubernetes
                  │
       ┌──────────┼──────────┐
       │          │          │
      PVC       Service    NetworkPolicy
       │          │          │
       └──────────┼──────────┘
                  │
              Database
```

But the database layer differs:

```text
PostgreSQL
 └── WAL replication

MySQL
 └── MySQL replication / InnoDB Cluster patterns

MongoDB
 └── Replica Set

SQL Server
 └── SQL Server-specific HA/DR
```

---

# 39. Database Comparison

| Database      | Type       | Kubernetes State Model           | Replication/HA                              |
| ------------- | ---------- | -------------------------------- | ------------------------------------------- |
| PostgreSQL    | Relational | Stateful                         | PostgreSQL replication/operator             |
| MySQL         | Relational | Stateful                         | MySQL replication/operator                  |
| MongoDB       | Document   | Stateful                         | Replica Set/operator                        |
| SQL Server    | Relational | Stateful                         | SQL Server-specific HA/DR                   |
| Redis         | Key-value  | Stateful/ephemeral depending use | Redis-specific replication/sentinel/cluster |
| Elasticsearch | Search     | Stateful                         | Elasticsearch cluster                       |

---

# 40. Why Did We Start With PostgreSQL?

Not because PostgreSQL is the only database.

We started with PostgreSQL because our e-commerce example needs a relational transactional database.

For example:

```text
Order
Customer
Payment
Invoice
Tax
```

have strong relational characteristics.

Example:

```text
orders
   │
   ├── customer_id
   ├── product_id
   ├── quantity
   └── status
```

A relational database is a natural fit.

PostgreSQL is simply our **reference implementation**.

---

# 41. Could We Use MySQL Instead?

Absolutely.

Architecture could become:

```text
order-service
      │
      ▼
MySQL
```

instead of:

```text
order-service
      │
      ▼
PostgreSQL
```

The application layer would use a different JDBC driver/configuration, and database-specific SQL/migration behavior may differ.

The DevOps architecture remains broadly similar:

```text
EKS
 ↓
Stateful workload/operator
 ↓
PVC
 ↓
EBS
 ↓
Backup
 ↓
Monitoring
```

---

# 42. Could We Use MongoDB?

Yes, if the data model is appropriate.

For example:

```text
product catalog
```

might have highly variable document structures.

MongoDB:

```json
{
  "productId": "P1001",
  "name": "Laptop",
  "attributes": {
    "ram": "32GB",
    "screen": "15.6"
  }
}
```

could be a reasonable fit depending on requirements.

But transactional/order/payment workloads often require careful relational and transactional modeling.

The correct database should be chosen from:

```text
Business requirement
        ↓
Data model
        ↓
Consistency requirements
        ↓
Query patterns
        ↓
Scale
        ↓
Operational requirements
```

not simply:

> "MongoDB is popular."

---

# 43. Could We Use SQL Server?

Yes.

An enterprise organization may already standardize on Microsoft SQL Server because of:

* existing applications
* Microsoft ecosystem
* operational expertise
* licensing agreements
* reporting/BI integration
* vendor requirements

Then the architecture might be:

```text
EKS
 │
 └── application
       │
       ▼
SQL Server
```

The same DevOps principles still apply:

```text
Storage
Backup
Security
HA
Monitoring
Recovery
Upgrade
```

but the implementation is SQL Server-specific.

---

# 44. Database Selection Matrix

For our e-commerce platform:

| Requirement     | Possible Choice                 |
| --------------- | ------------------------------- |
| Orders          | PostgreSQL / MySQL / SQL Server |
| Payments        | PostgreSQL / MySQL / SQL Server |
| Customers       | PostgreSQL / MySQL              |
| Product catalog | PostgreSQL / MongoDB            |
| Search          | OpenSearch/Elasticsearch        |
| Cache           | Redis                           |
| Events          | SQS/SNS/Kafka                   |
| Analytics       | Depends on workload             |
| Reporting       | PostgreSQL / warehouse          |

There is no rule saying:

> "One company must use one database technology."

A microservice architecture can legitimately use **polyglot persistence**.

---

# 45. Polyglot Persistence

Example:

```text
                    E-Commerce
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
     Orders            Catalog           Search
       │                 │                 │
 PostgreSQL          MongoDB          OpenSearch
       │
       │
    Payments
       │
   PostgreSQL

                    Cache
                      │
                    Redis
```

Each system is chosen according to its workload.

But:

> Polyglot persistence increases operational complexity.

More databases mean:

```text
More skills
+
More monitoring
+
More backups
+
More upgrades
+
More security models
+
More failure modes
```

So don't introduce five databases simply to demonstrate technology.

---

# 46. Application-to-Database Ownership

Our microservice rule remains:

```text
order-service
      │
      ▼
order database
```

Not:

```text
order-service
      │
      ├── customer DB
      ├── payment DB
      ├── inventory DB
      └── shipping DB
```

Services should communicate through:

```text
API
Events
Messaging
```

rather than bypassing service boundaries with direct database access.

---

# 47. Kubernetes Database Lifecycle

A proper lifecycle:

```text
Design
  ↓
Provision
  ↓
Storage
  ↓
Security
  ↓
Deploy
  ↓
Initialize
  ↓
Replication
  ↓
Backup
  ↓
Monitor
  ↓
Scale
  ↓
Upgrade
  ↓
Failover test
  ↓
Restore test
  ↓
DR
```

This is what "managing a database in EKS" really means.

---

# 48. What We Would Put in Git

For an operator-based PostgreSQL deployment:

```text
gitops/
└── databases/
    └── postgres/
        ├── namespace.yaml
        ├── operator/
        ├── cluster.yaml
        ├── storage.yaml
        ├── network-policy.yaml
        ├── backup.yaml
        └── monitoring.yaml
```

ArgoCD then manages:

```text
Git
 ↓
ArgoCD
 ↓
EKS
 ↓
Database Operator
 ↓
Database
```

---

# 49. Terraform vs ArgoCD

We need a clear boundary.

### Terraform

Use Terraform for:

```text
VPC
EKS
IAM
KMS
EBS-related AWS infrastructure
S3
AWS networking
```

### ArgoCD

Use ArgoCD/GitOps for:

```text
Namespace
Operator
Database CR
NetworkPolicy
Monitoring
Kubernetes resources
```

Don't have Terraform and ArgoCD continuously fight over the same Kubernetes resources.

---

# 50. Production Database Architecture Inside EKS

A mature architecture could look like:

```text
                         EKS
                          │
                 PostgreSQL Operator
                          │
             ┌────────────┼────────────┐
             │            │            │
           AZ-A         AZ-B         AZ-C
             │            │            │
        postgres-0    postgres-1    postgres-2
          Primary       Replica       Replica
             │            │            │
            EBS          EBS          EBS
             │            │            │
             └────────────┼────────────┘
                          │
                     Backup System
                          │
                          ▼
                         S3
```

Application:

```text
order-service
      │
      ▼
Database Service
      │
      ▼
Current Primary
```

Security:

```text
NetworkPolicy
+
Kubernetes RBAC
+
Secrets Manager
+
KMS
```

Monitoring:

```text
PostgreSQL
    │
    ▼
Exporter / Operator metrics
    │
    ▼
Prometheus
    │
    ▼
Grafana
    │
    ▼
Alertmanager
```

---

# 51. Failure Scenario — Pod Failure

```text
postgres-0
    X
```

Kubernetes:

```text
Detect failure
     ↓
Create replacement
     ↓
Attach persistent storage
     ↓
Start database
```

But if it's a primary/replica architecture:

```text
Operator
   ↓
Determine cluster state
   ↓
Promote healthy replica if necessary
   ↓
Repair failed instance
```

---

# 52. Failure Scenario — Node Failure

```text
Node-A
  │
  └── postgres-0
```

Node fails.

```text
Node-A
   X
```

Kubernetes schedules/restarts according to constraints.

But storage and database topology matter:

```text
Pod
 ↓
PVC
 ↓
EBS
 ↓
AZ
```

This is why stateful workload scheduling is more complicated than normal deployments.

---

# 53. Failure Scenario — AZ Failure

This is where true HA is tested.

```text
AZ-A
 │
 └── postgres-0
```

fails.

Healthy architecture:

```text
AZ-B
 └── postgres-1

AZ-C
 └── postgres-2
```

Database-level replication and failover must determine the new primary.

Kubernetes alone isn't enough.

---

# 54. Failure Scenario — Accidental DELETE

Suppose:

```sql
DELETE FROM orders;
```

was executed accidentally.

Pod restart won't help.

Node replacement won't help.

Kubernetes reconciliation won't restore data.

You need:

```text
Database backup
+
PITR
+
Restore procedure
```

This demonstrates the difference between:

> **Infrastructure recovery**

and:

> **Data recovery**

---

# 55. The Four Layers of Database Reliability

Think about databases in four layers.

### Layer 1 — Kubernetes

```text
Pod
Node
StatefulSet
Service
Scheduling
```

### Layer 2 — Storage

```text
PVC
PV
EBS
Snapshots
```

### Layer 3 — Database

```text
Replication
Transactions
WAL/binlog/oplog
Failover
Indexes
Locks
```

### Layer 4 — Recovery

```text
Backup
PITR
DR
Restore
Validation
```

A production database needs all four.

---

# 56. RDS vs Database Inside EKS

Now the comparison becomes much clearer.

| Area                   | RDS                         | Database in EKS                    |
| ---------------------- | --------------------------- | ---------------------------------- |
| Database server        | AWS manages                 | You manage                         |
| Pod                    | No                          | Yes                                |
| StatefulSet            | No                          | Usually/operator abstraction       |
| PVC                    | AWS-managed storage model   | Yes                                |
| EBS                    | Underlying managed service  | Usually                            |
| Replication            | AWS-managed options         | You/operator                       |
| Failover               | AWS-managed                 | You/operator                       |
| Backup                 | AWS managed capabilities    | You design/manage                  |
| PITR                   | Built-in managed capability | You implement/operator             |
| Patching               | AWS managed                 | You manage                         |
| Monitoring             | CloudWatch                  | Prometheus/Grafana + DB monitoring |
| Upgrades               | Managed workflow            | You plan/manage                    |
| Operational complexity | Lower                       | Higher                             |
| Kubernetes control     | Lower                       | Higher                             |

---

# 57. The Correct Production Decision

For our project, we'll now understand both.

### Managed database path

```text
EKS
 │
 └── Microservices
        │
        ▼
       RDS
```

### Kubernetes database path

```text
EKS
 │
 ├── Microservices
 │
 └── Database Operator
        │
        ├── Database Primary
        ├── Database Replicas
        └── Persistent Storage
```

Neither is universally "correct."

The choice depends on:

```text
Business requirements
+
Team expertise
+
Cost
+
HA requirements
+
Compliance
+
Operational maturity
+
Vendor support
+
Performance
```

---

# 58. What I Want You to Remember

Don't memorize:

> "Use RDS."

Instead remember:

> **A database is a stateful distributed system.**

When you put it inside EKS, you inherit responsibility for:

```text
Storage
Replication
Failover
Backup
Restore
Security
Monitoring
Upgrades
DR
```

Kubernetes provides the orchestration primitives.

The database engine provides the data semantics.

The operator connects those two worlds.

---

# 59. Our Final Architecture Understanding

Now we have three possible database architectures:

### Option A — Recommended for our primary AWS production design

```text
EKS
 │
 └── Applications
        │
        ▼
       RDS
```

### Option B — Database inside EKS

```text
EKS
 │
 ├── Applications
 │
 └── Database Operator
        │
        ├── Primary
        ├── Replica
        └── Replica
              │
              ▼
             EBS
```

### Option C — Polyglot persistence

```text
                    EKS
                     │
       ┌─────────────┼─────────────┐
       │             │             │
    Services       Services      Services
       │             │             │
   PostgreSQL      MongoDB       MySQL
       │             │             │
       └─────────────┼─────────────┘
                     │
                   Redis
```

And all three are valid architectural patterns when the requirements justify them.

---

# 60. Where We Go Next

Now that we've covered both:

```text
RDS
```

and:

```text
Database inside EKS
```

we can move forward to Redis without having a gap in our understanding.

The sequence is now:

```text
20-EKS-Database-Management.md
          ↓
21-RDS.md
          ↓
22-Redis.md
          ↓
23-SQS-SNS.md
          ↓
24-Secrets-Management.md
          ↓
25-GitOps-Repository.md
          ↓
26-Helm.md
          ↓
27-ArgoCD.md
```

And when we reach Redis, we'll apply the **same production thinking**:

```text
Where does it run?
       ↓
Who provisions it?
       ↓
How does the application connect?
       ↓
Where is the data stored?
       ↓
What happens when it fails?
       ↓
How does HA work?
       ↓
How is authentication handled?
       ↓
How is it monitored?
       ↓
How is it backed up?
       ↓
How does it behave under 20+ microservices
and low / medium / high traffic?
```

That way, we're building an actual **production DevOps architecture**, rather than just going tool-by-tool.
