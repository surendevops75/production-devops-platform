# 55 — Local Docker Environment

> **Project:** Enterprise E-Commerce Platform
> **Service:** `order-service`
> **Purpose:** Build a reliable local environment that closely matches the networking and configuration model we will later use in AWS/EKS.

---

# 1. Objective

We already have:

```text
Spring Boot
    ↓
PostgreSQL
```

Now we make the local environment reproducible:

```text
Developer Machine
        │
        ▼
Docker Compose
        │
        ├── order-service
        │       │
        │       └── Spring Boot :8080
        │
        └── postgres
                │
                └── PostgreSQL :5432
```

We will implement:

* dedicated Docker network
* persistent PostgreSQL volume
* application environment variables
* health checks
* startup dependency handling
* container logs
* database persistence
* application restart behavior
* failure testing
* connectivity troubleshooting
* clean shutdown
* local configuration separation

---

# 2. Why We Need This Before AWS

A common mistake is:

```text
Write application
    ↓
Immediately deploy to EKS
    ↓
Debug everything simultaneously
```

That makes troubleshooting difficult.

Instead:

```text
Local
  ↓
Docker
  ↓
Docker Compose
  ↓
CI
  ↓
ECR
  ↓
EKS
```

Each layer should work before moving to the next.

---

# 3. Target Local Architecture

Our local environment should look like:

```text
                    Developer
                        │
                        │ HTTP
                        ▼
                localhost:8080
                        │
                        ▼
              ┌─────────────────┐
              │  order-service   │
              │  Spring Boot     │
              │      :8080       │
              └────────┬────────┘
                       │
                       │ PostgreSQL
                       │ :5432
                       ▼
              ┌─────────────────┐
              │    postgres      │
              │  PostgreSQL 16  │
              │      :5432       │
              └────────┬────────┘
                       │
                       ▼
                 Docker Volume
```

---

# 4. Project Structure

Update the repository:

```text
order-service/
│
├── .github/
│
├── .mvn/
│
├── src/
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── pom.xml
├── mvnw
├── mvnw.cmd
├── README.md
├── CONTRIBUTING.md
└── SECURITY.md
```

For local-only configuration, we will **not** commit actual passwords or `.env` files.

---

# 5. Docker Network

Create a dedicated Compose network:

```yaml
networks:
  order-network:
    driver: bridge
```

Then:

```yaml
services:

  postgres:
    networks:
      - order-network

  order-service:
    networks:
      - order-network
```

Now the containers communicate through:

```text
order-network
```

---

# 6. Why We Don't Use `localhost`

This is one of the most important Docker concepts.

From your laptop:

```text
localhost:8080
```

means:

```text
Your laptop
```

Inside `order-service`:

```text
localhost
```

means:

```text
order-service container itself
```

Inside PostgreSQL:

```text
localhost
```

means:

```text
postgres container itself
```

Therefore:

```text
order-service → postgres
```

must use the Compose service name:

```text
postgres
```

So:

```text
jdbc:postgresql://postgres:5432/orders
```

is correct.

---

# 7. Complete Docker Compose

Use:

```yaml
services:

  postgres:

    image: postgres:16

    container_name: order-postgres

    restart: unless-stopped

    environment:
      POSTGRES_DB: orders
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres

    ports:
      - "5432:5432"

    volumes:
      - postgres-data:/var/lib/postgresql/data

    healthcheck:
      test:
        [
          "CMD-SHELL",
          "pg_isready -U postgres -d orders"
        ]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 10s

    networks:
      - order-network


  order-service:

    build:
      context: .
      dockerfile: Dockerfile

    container_name: order-service

    restart: unless-stopped

    depends_on:
      postgres:
        condition: service_healthy

    environment:

      SPRING_PROFILES_ACTIVE: local

      DB_URL: jdbc:postgresql://postgres:5432/orders
      DB_USERNAME: postgres
      DB_PASSWORD: postgres

    ports:
      - "8080:8080"

    networks:
      - order-network


volumes:

  postgres-data:


networks:

  order-network:
    driver: bridge
```

---

# 8. Start the Environment

From the repository:

```bash
docker compose up -d --build
```

Check:

```bash
docker compose ps
```

Expected:

```text
NAME             STATUS
order-postgres   Up (healthy)
order-service    Up
```

---

# 9. Understand the Startup Sequence

Docker Compose starts:

```text
postgres
    │
    ▼
healthcheck
    │
    ▼
healthy
    │
    ▼
order-service
```

This is better than simply:

```yaml
depends_on:
  - postgres
```

because container startup does **not** necessarily mean PostgreSQL is ready to accept connections.

---

# 10. Check PostgreSQL Health

Run:

```bash
docker inspect order-postgres \
  --format='{{json .State.Health}}'
```

You should see health information.

A simpler check:

```bash
docker compose ps
```

should show:

```text
healthy
```

---

# 11. Check Application Health

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

# 12. Check Prometheus Metrics

```bash
curl http://localhost:8080/actuator/prometheus
```

You should receive Prometheus-formatted metrics.

For example:

```text
jvm_memory_used_bytes
process_cpu_usage
http_server_requests_seconds_count
```

---

# 13. Test the API

Create an order:

```bash
curl -X POST \
  http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 1001,
    "productId": 501,
    "quantity": 2
  }'
```

Then:

```bash
curl http://localhost:8080/api/v1/orders
```

---

# 14. Verify Database Data

Connect:

```bash
docker exec -it order-postgres \
  psql -U postgres -d orders
```

Run:

```sql
SELECT * FROM orders;
```

Example:

```text
 id | customer_id | product_id | quantity | status
----+-------------+------------+----------+---------
  1 |        1001 |        501 |        2 | CREATED
```

Exit:

```sql
\q
```

---

# 15. Test Persistence

This is important.

Stop the containers:

```bash
docker compose down
```

Start again:

```bash
docker compose up -d
```

Check:

```bash
docker exec -it order-postgres \
  psql -U postgres -d orders \
  -c "SELECT * FROM orders;"
```

The order should still exist.

Why?

Because:

```yaml
volumes:
  - postgres-data:/var/lib/postgresql/data
```

stores PostgreSQL data outside the container filesystem.

---

# 16. Container vs Volume

Without a volume:

```text
PostgreSQL container
        │
        └── database data
```

Delete container:

```text
data disappears
```

With volume:

```text
PostgreSQL container
        │
        ▼
Docker volume
        │
        ▼
database data
```

Container can be recreated while the volume remains.

---

# 17. Important Difference From Production

Our local PostgreSQL volume is:

```text
Docker volume
```

Production database will be:

```text
Amazon RDS PostgreSQL
```

So:

```text
LOCAL
order-service → PostgreSQL container


PRODUCTION
EKS Pod → RDS PostgreSQL
```

We deliberately don't try to make local Docker PostgreSQL pretend to be production RDS.

---

# 18. Environment Configuration

Our application currently has:

```yaml
spring:
  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/orders}
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}
```

This gives us:

```text
Environment variable exists
        ↓
Use environment value

Environment variable doesn't exist
        ↓
Use local default
```

For production:

```text
DB_URL
DB_USERNAME
DB_PASSWORD
```

will come from Kubernetes/Secrets Manager rather than source code.

---

# 19. Don't Commit `.env`

If you use a local `.env` file:

```text
.env
```

add it to `.gitignore`:

```gitignore
.env
.env.*
!.env.example
```

Then create:

```text
.env.example
```

with placeholders only:

```env
DB_URL=jdbc:postgresql://localhost:5432/orders
DB_USERNAME=postgres
DB_PASSWORD=CHANGE_ME
```

Never put a real production credential into `.env.example`.

---

# 20. Why We Prefer Compose Environment Variables Here

For this learning environment:

```yaml
environment:
  DB_URL: jdbc:postgresql://postgres:5432/orders
  DB_USERNAME: postgres
  DB_PASSWORD: postgres
```

is acceptable because these are local development credentials.

Production will be completely different:

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
Pod
        │
        ▼
Spring Boot
```

---

# 21. Container Logs

View application logs:

```bash
docker logs order-service
```

Follow logs:

```bash
docker logs -f order-service
```

PostgreSQL:

```bash
docker logs order-postgres
```

Compose:

```bash
docker compose logs
```

Only application:

```bash
docker compose logs order-service
```

Only database:

```bash
docker compose logs postgres
```

---

# 22. Why We Use stdout/stderr

Our application should log to:

```text
stdout
stderr
```

rather than depending on application-managed log files inside the container.

Production flow:

```text
Spring Boot
     │
     ▼
stdout/stderr
     │
     ▼
container runtime
     │
     ▼
Kubernetes node logs
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

This is why our later ELK architecture works naturally with Kubernetes.

---

# 23. Test Application Restart

Kill the application container:

```bash
docker kill order-service
```

Check:

```bash
docker ps
```

Because we configured:

```yaml
restart: unless-stopped
```

Docker should restart it.

Check:

```bash
docker compose ps
```

Then:

```bash
curl http://localhost:8080/actuator/health
```

---

# 24. Test Database Failure

Stop PostgreSQL:

```bash
docker stop order-postgres
```

Now check:

```bash
curl http://localhost:8080/actuator/health
```

The application's database health may become unhealthy depending on the Actuator configuration and dependency health indicators.

Check logs:

```bash
docker logs order-service
```

You may see database connection errors.

This is useful because we're deliberately creating a failure.

---

# 25. Recover PostgreSQL

Start it again:

```bash
docker start order-postgres
```

Check:

```bash
docker inspect order-postgres \
  --format='{{.State.Health.Status}}'
```

Wait until:

```text
healthy
```

Then:

```bash
curl http://localhost:8080/actuator/health
```

---

# 26. Test Network Failure

Inspect the network:

```bash
docker network inspect order-network
```

You should see:

```text
order-service
order-postgres
```

The important relationship is:

```text
order-service
      │
      │ order-network
      ▼
postgres
```

---

# 27. DNS Test

From the application container:

```bash
docker exec order-service \
  getent hosts postgres
```

Expected output resembles:

```text
172.x.x.x postgres
```

This demonstrates Docker's internal DNS.

The application doesn't need to know PostgreSQL's IP address.

It only needs:

```text
postgres
```

This is conceptually similar to Kubernetes:

```text
postgres.production.svc.cluster.local
```

---

# 28. Test PostgreSQL Connectivity

If PostgreSQL client tools are available inside the application image, you could test directly.

Our minimal runtime image doesn't include them, which is intentional.

Instead, test through the application:

```bash
curl http://localhost:8080/actuator/health
```

or connect from the PostgreSQL container itself:

```bash
docker exec -it order-postgres \
  psql -U postgres -d orders
```

In production, don't install debugging tools into every application container just for troubleshooting.

Use controlled debugging environments instead.

---

# 29. Inspect Container Environment

You can inspect environment variables:

```bash
docker inspect order-service
```

Be careful: this may expose credentials.

For local development that's manageable, but in production you should avoid casually dumping container configuration into tickets, logs, screenshots, or chat channels.

---

# 30. Resource Limits

We eventually need to understand:

```text
CPU
Memory
```

For local testing you can add limits.

Example:

```yaml
services:

  order-service:

    deploy:
      resources:
        limits:
          cpus: "1.0"
          memory: 768M
        reservations:
          cpus: "0.25"
          memory: 256M
```

However, don't assume every `deploy.resources` behavior is identical across all Docker Compose/runtime versions.

For production Kubernetes, the equivalent is explicitly configured with:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: "1"
    memory: 768Mi
```

We'll use those properly in the Helm deployment.

---

# 31. Why Requests Matter Later

The local environment introduces a concept that becomes critical in EKS:

```text
Application
     │
     ├── CPU
     └── Memory
```

In Kubernetes:

```text
Pod
 │
 ├── requests
 │
 └── limits
```

Then:

```text
requests
   ↓
scheduler placement

usage
   ↓
HPA decisions

limits
   ↓
container enforcement
```

So resource configuration isn't just an optimization detail.

---

# 32. Clean Shutdown

Stop everything:

```bash
docker compose down
```

This removes:

```text
containers
network
```

but normally keeps:

```text
postgres-data volume
```

Verify:

```bash
docker volume ls
```

You should see the PostgreSQL volume.

---

# 33. Complete Cleanup

If you intentionally want to delete the database too:

```bash
docker compose down -v
```

**Warning:** this deletes the Compose-managed PostgreSQL volume and therefore local database data.

Use this when you deliberately want a clean database.

---

# 34. Fresh Environment Test

A very useful test is:

```bash
docker compose down -v
docker compose up -d --build
```

Now the database starts empty.

Check:

```bash
curl http://localhost:8080/actuator/health
```

Then create an order again.

This verifies that another developer can reproduce the environment from the repository.

---

# 35. Local Failure Scenarios

We should deliberately test these:

| Failure                    | Expected investigation         |
| -------------------------- | ------------------------------ |
| Application doesn't start  | `docker logs order-service`    |
| PostgreSQL doesn't start   | `docker logs order-postgres`   |
| DB connection refused      | Check service name/network     |
| Wrong DB credentials       | Check environment              |
| Port 8080 occupied         | Check host process/container   |
| Port 5432 occupied         | Check PostgreSQL/other service |
| Database disappears        | Check volume                   |
| Application crashes        | Check logs/exit code           |
| Container keeps restarting | Inspect logs/restart policy    |
| API returns 500            | Application + DB logs          |
| Health becomes unhealthy   | Check dependency               |
| DNS doesn't resolve        | Inspect Docker network         |

---

# 36. Application Failure Example

Stop PostgreSQL:

```bash
docker stop order-postgres
```

Then watch:

```bash
docker logs -f order-service
```

You may see errors similar to:

```text
Connection refused
```

This teaches an important production troubleshooting pattern:

```text
Application symptom
       ↓
Database connection failure
       ↓
Check network
       ↓
Check endpoint
       ↓
Check credentials
       ↓
Check database health
```

Don't immediately restart everything.

---

# 37. Database Failure vs Application Failure

Suppose:

```text
GET /api/v1/orders/1
```

returns an error.

Possible causes:

```text
Application bug
       OR
Database unavailable
       OR
Network problem
       OR
Authentication problem
       OR
Connection pool exhaustion
       OR
Slow database
```

This is why production troubleshooting must follow layers instead of guessing.

---

# 38. Local Connection Pool

Spring Boot uses HikariCP by default with Spring Data/JPA.

Conceptually:

```text
Order Service
      │
      ▼
HikariCP
 ┌────┼────┐
 │    │    │
conn conn conn
 │    │    │
 └────┼────┘
      ▼
 PostgreSQL
```

Later, when we have:

```text
10 Pods
```

and each Pod has:

```text
20 DB connections
```

we could potentially create:

```text
10 × 20 = 200 connections
```

to the database.

This becomes extremely important with:

```text
HPA
+
RDS
```

We will tune connection pools before high-traffic testing.

---

# 39. Don't Over-Engineer Local Docker

We don't need to run:

```text
Kubernetes
Redis
Kafka
RabbitMQ
Elasticsearch
Logstash
Prometheus
Grafana
Jaeger
21 microservices
```

on your laptop just because production contains them.

A practical local environment should start small:

```text
order-service
+
PostgreSQL
```

Then add dependencies when the service actually needs them.

For example, when Redis is implemented:

```text
order-service
+
PostgreSQL
+
Redis
```

When messaging is implemented:

```text
order-service
+
PostgreSQL
+
Redis
+
SQS/local messaging equivalent
```

---

# 40. Local vs Production

This distinction is critical.

| Local                | Production                |
| -------------------- | ------------------------- |
| Docker Compose       | Kubernetes/EKS            |
| PostgreSQL container | RDS PostgreSQL            |
| Docker volume        | RDS managed storage       |
| `.env`/local env     | Secrets Manager           |
| Docker network       | VPC/Kubernetes networking |
| localhost            | ALB/Service DNS           |
| Docker logs          | Fluent Bit → ELK          |
| Manual startup       | GitOps/ArgoCD             |
| Local image          | ECR image                 |
| Docker healthcheck   | Kubernetes probes         |
| Compose restart      | Kubernetes controller     |

The **application behavior** should be similar, while the **platform implementation** changes.

---

# 41. Add a Local Development Section to README

Add:

````markdown
## Local Development

### Prerequisites

- Docker
- Docker Compose
- Java 21
- Git

### Start

```bash
docker compose up -d --build
````

### Check

```bash
docker compose ps
```

### Health

```bash
curl http://localhost:8080/actuator/health
```

### Create Order

```bash
curl -X POST \
  http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 1001,
    "productId": 501,
    "quantity": 2
  }'
```

### Logs

```bash
docker compose logs -f order-service
```

### Stop

```bash
docker compose down
```

### Reset Database

```bash
docker compose down -v
docker compose up -d --build
```

````

---

# 42. Git Changes

Check:

```bash
git status
````

You should have changes such as:

```text
modified:
    docker-compose.yaml

modified:
    README.md
```

Then:

```bash
git add .
```

Commit:

```bash
git commit -m "build: improve local docker environment"
```

Push through your normal feature branch/PR workflow.

---

# 43. Definition of Done

This chapter is complete when all of these work:

```text
[✓] Docker Compose starts
[✓] PostgreSQL becomes healthy
[✓] Order Service starts
[✓] Application connects to PostgreSQL
[✓] Health endpoint works
[✓] Prometheus endpoint works
[✓] POST order works
[✓] GET order works
[✓] PUT order works
[✓] DELETE order works
[✓] PostgreSQL data persists across container restart
[✓] Application logs accessible
[✓] PostgreSQL logs accessible
[✓] Docker DNS resolves postgres
[✓] Application restart works
[✓] Database failure can be diagnosed
[✓] Full cleanup works
[✓] Fresh environment can be recreated
```

---

# 44. Current Architecture

We now have:

```text
                         LOCAL DEVELOPMENT

                       Developer Machine
                              │
                              │ HTTP
                              ▼
                         localhost:8080
                              │
                     ┌────────┴────────┐
                     │                 │
                     ▼                 │
              order-service            │
              Spring Boot              │
                     │                 │
                     │ Docker DNS      │
                     │                 │
                     ▼                 │
                  postgres             │
                  PostgreSQL           │
                     │                 │
                     ▼                 │
               postgres-data           │
                  volume               │
                                       │
                     Docker Network ────┘
```

And the eventual production architecture remains:

```text
Developer
   │
   ▼
GitHub
   │
   ▼
CI
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
   ├── order-service
   │       │
   │       ├── RDS
   │       ├── Redis
   │       └── SQS/SNS
   │
   └── Platform
           ├── Prometheus
           ├── Grafana
           ├── ELK
           ├── OpenTelemetry
           └── Jaeger
```

---

# 45. Next Step

The local application is now reproducible.

The next major dependency is **Terraform state**.

We need to establish this **before creating VPC/EKS/RDS/etc.**, because production Terraform must have a safe remote state design.

## Next: `56-Terraform-State-Bootstrap.md`

We will build:

```text
Terraform
    │
    ▼
S3 State Bucket
    │
    ├── Encryption
    ├── Versioning
    ├── Access Control
    └── Native State Locking
```

and then establish the Terraform workflow:

```text
Terraform Code
      ↓
fmt
      ↓
validate
      ↓
plan
      ↓
PR Review
      ↓
approved apply
      ↓
AWS
```

After that we can build the actual **VPC**.
