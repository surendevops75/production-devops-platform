# 05 — Dockerization of `order-service`

Now we take the application we created and turn it into a **production-ready container image**.

The flow is:

```text
Java Source Code
      |
      v
Maven Build
      |
      v
order-service.jar
      |
      v
Docker Build
      |
      v
Docker Image
      |
      +----> Local Testing
      |
      +----> Trivy Scan
      |
      v
Amazon ECR
      |
      v
EKS
```

---

# 1. Why Docker?

Without Docker:

```text
Developer machine
    |
    +-- Java 21
    +-- Maven
    +-- PostgreSQL
    +-- dependencies
```

Another developer might have:

```text
Developer machine
    |
    +-- Java 17
    +-- different Maven
    +-- different libraries
```

Then:

> "It works on my machine."

Docker solves the runtime consistency problem.

```text
Application
    +
Runtime
    +
Dependencies
    |
    v
Container Image
```

The same image can run in:

```text
Developer laptop
       ↓
CI
       ↓
DEV EKS
       ↓
QA EKS
       ↓
PROD EKS
```

---

# 2. Dockerfile

Create:

```text
order-service/Dockerfile
```

We'll use a **multi-stage build**.

```dockerfile
# =========================
# Stage 1 - Build
# =========================

FROM eclipse-temurin:21-jdk AS builder

WORKDIR /build

COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

RUN chmod +x mvnw

RUN ./mvnw dependency:go-offline

COPY src ./src

RUN ./mvnw clean package -DskipTests


# =========================
# Stage 2 - Runtime
# =========================

FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /build/target/*.jar app.jar

EXPOSE 8080

USER 10001

ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

# 3. Why Multi-Stage Build?

A common beginner Dockerfile is:

```dockerfile
FROM maven
COPY .
RUN mvn package
CMD ...
```

The final image contains:

```text
Maven
JDK
source code
build files
dependencies
application
```

That's unnecessary.

With multi-stage:

```text
Build stage
    |
    +-- JDK
    +-- Maven
    +-- source
    |
    v
JAR
    |
    v
Runtime stage
    |
    +-- JRE
    +-- JAR
```

So production gets only what it needs.

---

# 4. Build Stage

This:

```dockerfile
FROM eclipse-temurin:21-jdk AS builder
```

creates the build environment.

Then:

```dockerfile
WORKDIR /build
```

sets:

```text
/build
```

as the working directory.

---

# 5. Copy Maven Wrapper First

```dockerfile
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
```

Why copy these before the source?

Docker layer caching.

If source code changes but dependencies don't:

```text
pom.xml
     |
     v
dependency layer
     |
     v
cached
```

Docker doesn't need to download everything again.

---

# 6. Dependency Cache

```dockerfile
RUN ./mvnw dependency:go-offline
```

This downloads dependencies.

For example:

```text
Spring Boot
PostgreSQL
Hibernate
Actuator
Micrometer
```

---

# 7. Copy Source

```dockerfile
COPY src ./src
```

Now source code enters the build layer.

If:

```text
src/
```

changes, only later layers need rebuilding.

---

# 8. Build Application

```dockerfile
RUN ./mvnw clean package -DskipTests
```

This generates:

```text
target/order-service.jar
```

We skip tests here because our CI pipeline will run tests **before** building the production image.

That gives us:

```text
CI
 |
 +--> Unit Tests
 |
 +--> Security
 |
 +--> Docker Build
```

rather than hiding tests inside Docker.

---

# 9. Runtime Image

```dockerfile
FROM eclipse-temurin:21-jre
```

We don't need the full JDK to run the application.

We need the Java runtime.

---

# 10. Non-Root Container

This is very important.

```dockerfile
USER 10001
```

The application doesn't run as:

```text
root
```

Instead:

```text
UID 10001
```

If an attacker compromises the application, this reduces the privileges available inside the container.

Later Kubernetes security settings will reinforce this.

---

# 11. Why `EXPOSE` Doesn't Publish the Port

This:

```dockerfile
EXPOSE 8080
```

doesn't make the application accessible from the Internet.

It simply documents that:

```text
container -> 8080
```

Later Kubernetes handles networking:

```text
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
 |
 v
8080
```

---

# 12. ENTRYPOINT

```dockerfile
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Container starts:

```text
java -jar app.jar
```

---

# 13. `.dockerignore`

Create:

```text
.dockerignore
```

with:

```text
.git
.github
.gitignore

.idea
.vscode

target

*.log

.env
.env.*

README.md

node_modules
```

This prevents unnecessary files from entering the Docker build context.

---

# 14. Why `.dockerignore` Matters

Without it:

```text
docker build .
```

could send:

```text
.git/
target/
IDE files
logs
local configuration
```

to the Docker daemon/build system.

That's unnecessary and potentially dangerous.

---

# 15. Build the Image

First build the application:

```bash
./mvnw clean package -DskipTests
```

Then:

```bash
docker build -t order-service:1.0.0 .
```

Check:

```bash
docker images
```

You should see something similar to:

```text
REPOSITORY       TAG       IMAGE ID
order-service    1.0.0     abc123
```

---

# 16. Run the Container

We need PostgreSQL first.

```bash
docker network create ecommerce-network
```

Run PostgreSQL:

```bash
docker run -d \
  --name order-postgres \
  --network ecommerce-network \
  -e POSTGRES_DB=orders \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  postgres:16
```

Then run our application:

```bash
docker run -d \
  --name order-service \
  --network ecommerce-network \
  -p 8080:8080 \
  -e DB_URL=jdbc:postgresql://order-postgres:5432/orders \
  -e DB_USERNAME=postgres \
  -e DB_PASSWORD=postgres \
  order-service:1.0.0
```

---

# 17. Check Containers

```bash
docker ps
```

Expected:

```text
CONTAINER
----------------
order-service
order-postgres
```

Check logs:

```bash
docker logs order-service
```

You should see Spring Boot starting.

---

# 18. Test API

```bash
curl \
  -X POST \
  http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "cust-001",
    "productId": "prod-001",
    "quantity": 2
  }'
```

Expected:

```json
{
  "id": 1,
  "customerId": "cust-001",
  "productId": "prod-001",
  "quantity": 2,
  "status": "CREATED"
}
```

Now we've proven:

```text
Docker
  |
  +-- order-service
  |
  +-- PostgreSQL
  |
  v
Working application
```

---

# 19. Docker Compose

Running multiple `docker run` commands becomes inconvenient.

Create:

```text
docker-compose.yaml
```

```yaml
services:

  postgres:
    image: postgres:16

    container_name: order-postgres

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


  order-service:

    build:
      context: .

    container_name: order-service

    depends_on:
      postgres:
        condition: service_healthy

    environment:

      DB_URL: jdbc:postgresql://postgres:5432/orders

      DB_USERNAME: postgres

      DB_PASSWORD: postgres

    ports:
      - "8080:8080"


volumes:

  postgres-data:
```

---

# 20. Start Everything

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Expected:

```text
NAME              STATUS
--------------------------------
order-postgres    healthy
order-service     running
```

Logs:

```bash
docker compose logs -f order-service
```

---

# 21. Test Again

```bash
curl http://localhost:8080/actuator/health
```

Expected:

```json
{
  "status": "UP"
}
```

Then:

```bash
curl \
  -X POST \
  http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "cust-100",
    "productId": "product-500",
    "quantity": 3
  }'
```

---

# 22. What We've Achieved

Our application is now:

```text
Java
 |
 v
Spring Boot
 |
 v
PostgreSQL
 |
 v
Docker
 |
 v
Docker Compose
```

But this is still local.

Next we need:

```text
Developer
   |
   v
GitHub
   |
   v
GitHub Actions
   |
   v
Docker Build
   |
   v
Trivy
   |
   v
Amazon ECR
```

---

# 23. Image Naming Strategy

We don't want:

```text
order-service:latest
```

as our production deployment identifier.

Use:

```text
order-service:1.0.0
```

or preferably an immutable Git-based identifier:

```text
order-service:8f72a91
```

A production image might eventually be:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:8f72a91
```

This tells us:

```text
AWS Account
     |
     v
ECR
     |
     v
order-service
     |
     v
Git commit 8f72a91
```

---

# 24. Image Lifecycle

Our future ECR repository:

```text
ECR
 |
 +-- order-service:8f72a91
 +-- order-service:9a72bc2
 +-- order-service:1.0.0
 +-- order-service:1.1.0
```

We can retain images according to a lifecycle policy.

For example:

```text
Production releases
     ↓
Keep important versions

Old development images
     ↓
Expire automatically
```

This prevents ECR storage from growing forever.

---

# 25. Docker Security

Our image should eventually satisfy:

```text
Non-root user
Minimal runtime image
No secrets
Pinned/controlled base image strategy
Regular vulnerability scanning
No unnecessary packages
Read-only filesystem where practical
Drop unnecessary Linux capabilities
```

Kubernetes will later add:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001
  allowPrivilegeEscalation: false
```

---

# 26. Important Production Principle

Don't think:

> "Docker image is secure because Trivy passed."

Security is layered:

```text
Source Code
    |
    v
SAST / SonarQube
    |
    v
Dependencies
    |
    v
Secret Scanning
    |
    v
Docker Image
    |
    v
Trivy
    |
    v
ECR
    |
    v
Kubernetes Security
    |
    v
AWS IAM
    |
    v
Network Controls
```

No single tool makes the application secure.

---

# 27. Our Git History Now

We can commit:

```bash
git add .
git commit -m "build: containerize order service"
git push
```

Our repository now looks like:

```text
order-service/
│
├── src/
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── pom.xml
├── README.md
│
├── deploy/
├── helm/
│
└── .github/
    ├── CODEOWNERS
    └── workflows/
```

---

# 28. The Big Picture

We've now completed:

```text
                 GitHub
                    |
                    v
              order-service
                    |
                    v
              Java/Spring Boot
                    |
                    v
                 Maven
                    |
                    v
                Docker
                    |
                    v
             Local Container
                    |
                    v
              PostgreSQL
```

The next step is where **AWS starts**.

---

# 29. Next — AWS Account & Infrastructure Repository

We'll create the second major repository:

```text
infrastructure
```

and build it properly instead of manually clicking through the AWS console.

Structure:

```text
infrastructure/
│
├── terraform/
│   │
│   ├── modules/
│   │   ├── vpc/
│   │   ├── eks/
│   │   ├── iam/
│   │   ├── ecr/
│   │   ├── rds/
│   │   ├── redis/
│   │   ├── dynamodb/
│   │   ├── s3/
│   │   ├── sqs/
│   │   ├── sns/
│   │   └── kms/
│   │
│   └── environments/
│       ├── dev/
│       ├── qa/
│       └── prod/
│
└── .github/
    └── workflows/
        └── terraform.yaml
```

Then we'll actually build:

```text
AWS
 |
 v
Terraform State
 |
 v
VPC
 |
 +---- Public Subnets
 |
 +---- Private Subnets
 |
 +---- NAT
 |
 +---- Route Tables
 |
 v
EKS
 |
 +---- Managed Node Groups
 |
 +---- IAM
 |
 +---- ECR
```

And we'll learn **why every subnet, route table, NAT gateway, security group, IAM role, and Terraform module exists**, rather than just copying Terraform code.

**Next: `06-AWS-Account-and-Infrastructure-Repository.md`.**
