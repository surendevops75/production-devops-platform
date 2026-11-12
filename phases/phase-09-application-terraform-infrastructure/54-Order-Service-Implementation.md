# 54 — Order Service Implementation

> **Project:** Enterprise E-Commerce Platform
> **Service:** `order-service`
> **Language:** Java 21
> **Framework:** Spring Boot
> **Database:** PostgreSQL
> **Build:** Maven
> **Container:** Docker
> **Port:** `8080`

This chapter is the **actual application implementation**. We will build the first production-style microservice before deploying anything to EKS.

---

# 1. What We Are Building

The Order Service will initially provide:

```text
Client
  │
  ▼
REST API
  │
  ▼
OrderController
  │
  ▼
OrderService
  │
  ▼
OrderRepository
  │
  ▼
PostgreSQL
```

The first version supports:

```text
POST   /api/v1/orders
GET    /api/v1/orders/{id}
GET    /api/v1/orders
PUT    /api/v1/orders/{id}
DELETE /api/v1/orders/{id}
```

Later this service will be extended with:

```text
Redis
SQS
Secrets Manager
Prometheus
OpenTelemetry
ELK
```

---

# 2. Final Repository Structure

Our repository will become:

```text
order-service/
│
├── .github/
│   ├── CODEOWNERS
│   ├── PULL_REQUEST_TEMPLATE.md
│   └── workflows/
│
├── .mvn/
│   └── wrapper/
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── company/
│   │   │           └── order/
│   │   │               ├── OrderServiceApplication.java
│   │   │               │
│   │   │               ├── controller/
│   │   │               │   └── OrderController.java
│   │   │               │
│   │   │               ├── service/
│   │   │               │   └── OrderService.java
│   │   │               │
│   │   │               ├── repository/
│   │   │               │   └── OrderRepository.java
│   │   │               │
│   │   │               ├── entity/
│   │   │               │   └── Order.java
│   │   │               │
│   │   │               ├── dto/
│   │   │               │   ├── CreateOrderRequest.java
│   │   │               │   └── OrderResponse.java
│   │   │               │
│   │   │               └── exception/
│   │   │                   ├── OrderNotFoundException.java
│   │   │                   └── GlobalExceptionHandler.java
│   │   │
│   │   └── resources/
│   │       ├── application.yaml
│   │       └── application-prod.yaml
│   │
│   └── test/
│       └── java/
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

---

# 3. Generate the Spring Boot Project

You can generate the initial project using Spring Initializr.

Use:

```text
Project: Maven
Language: Java
Spring Boot: current stable version
Group: com.company
Artifact: order-service
Name: order-service
Package: com.company.order
Packaging: Jar
Java: 21
```

Dependencies:

```text
Spring Web
Spring Data JPA
PostgreSQL Driver
Validation
Spring Boot Actuator
Prometheus
Lombok
```

The generated project should contain:

```text
pom.xml
src/
mvnw
mvnw.cmd
.mvn/
```

---

# 4. `pom.xml`

Use a dependency structure similar to:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
         http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>REPLACE_WITH_CURRENT_APPROVED_VERSION</version>
        <relativePath/>
    </parent>

    <groupId>com.company</groupId>
    <artifactId>order-service</artifactId>
    <version>1.0.0</version>

    <name>order-service</name>
    <description>Order management microservice</description>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>

        <!-- REST API -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- JPA -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- PostgreSQL -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Health / Metrics -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Prometheus -->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Tests -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <build>
        <plugins>

            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>

        </plugins>
    </build>

</project>
```

**Important:** don't blindly copy an old Spring Boot version into a new production project. Use the version approved by your organization and verify compatibility with Java 21 and the PostgreSQL driver.

---

# 5. Main Application

Create:

```text
src/main/java/com/company/order/OrderServiceApplication.java
```

```java
package com.company.order;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }
}
```

This is the entry point.

Run:

```bash
./mvnw spring-boot:run
```

Windows:

```powershell
.\mvnw.cmd spring-boot:run
```

---

# 6. Order Entity

Create:

```text
src/main/java/com/company/order/entity/Order.java
```

```java
package com.company.order.entity;

import jakarta.persistence.*;

@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private Long customerId;

    @Column(nullable = false)
    private Long productId;

    @Column(nullable = false)
    private Integer quantity;

    @Column(nullable = false)
    private String status;

    public Order() {
    }

    public Order(
            Long customerId,
            Long productId,
            Integer quantity,
            String status) {

        this.customerId = customerId;
        this.productId = productId;
        this.quantity = quantity;
        this.status = status;
    }

    public Long getId() {
        return id;
    }

    public Long getCustomerId() {
        return customerId;
    }

    public void setCustomerId(Long customerId) {
        this.customerId = customerId;
    }

    public Long getProductId() {
        return productId;
    }

    public void setProductId(Long productId) {
        this.productId = productId;
    }

    public Integer getQuantity() {
        return quantity;
    }

    public void setQuantity(Integer quantity) {
        this.quantity = quantity;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(String status) {
        this.status = status;
    }
}
```

---

# 7. Why `orders` Instead of `order`

Don't use:

```sql
order
```

as the table name because `ORDER` is associated with SQL syntax.

Use:

```sql
orders
```

instead.

---

# 8. Repository

Create:

```text
src/main/java/com/company/order/repository/OrderRepository.java
```

```java
package com.company.order.repository;

import com.company.order.entity.Order;
import org.springframework.data.jpa.repository.JpaRepository;

public interface OrderRepository extends JpaRepository<Order, Long> {
}
```

Spring Data JPA provides:

```text
save()
findById()
findAll()
deleteById()
existsById()
count()
```

without us writing SQL for these basic operations.

---

# 9. Create Order DTO

Don't expose the entity directly as the API contract.

Create:

```text
src/main/java/com/company/order/dto/CreateOrderRequest.java
```

```java
package com.company.order.dto;

import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotNull;

public record CreateOrderRequest(

        @NotNull
        Long customerId,

        @NotNull
        Long productId,

        @NotNull
        @Min(1)
        Integer quantity

) {
}
```

This validates:

```text
customerId -> required
productId  -> required
quantity   -> required
quantity   -> minimum 1
```

---

# 10. Response DTO

Create:

```text
src/main/java/com/company/order/dto/OrderResponse.java
```

```java
package com.company.order.dto;

public record OrderResponse(
        Long id,
        Long customerId,
        Long productId,
        Integer quantity,
        String status
) {
}
```

---

# 11. Exception

Create:

```text
src/main/java/com/company/order/exception/OrderNotFoundException.java
```

```java
package com.company.order.exception;

public class OrderNotFoundException extends RuntimeException {

    public OrderNotFoundException(Long id) {
        super("Order not found: " + id);
    }
}
```

---

# 12. Service Layer

Create:

```text
src/main/java/com/company/order/service/OrderService.java
```

```java
package com.company.order.service;

import com.company.order.dto.CreateOrderRequest;
import com.company.order.dto.OrderResponse;
import com.company.order.entity.Order;
import com.company.order.exception.OrderNotFoundException;
import com.company.order.repository.OrderRepository;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class OrderService {

    private final OrderRepository orderRepository;

    public OrderService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public OrderResponse createOrder(CreateOrderRequest request) {

        Order order = new Order(
                request.customerId(),
                request.productId(),
                request.quantity(),
                "CREATED"
        );

        Order savedOrder = orderRepository.save(order);

        return toResponse(savedOrder);
    }

    public OrderResponse getOrder(Long id) {

        Order order = orderRepository.findById(id)
                .orElseThrow(() -> new OrderNotFoundException(id));

        return toResponse(order);
    }

    public List<OrderResponse> getAllOrders() {

        return orderRepository.findAll()
                .stream()
                .map(this::toResponse)
                .toList();
    }

    public OrderResponse updateOrder(
            Long id,
            CreateOrderRequest request) {

        Order order = orderRepository.findById(id)
                .orElseThrow(() -> new OrderNotFoundException(id));

        order.setCustomerId(request.customerId());
        order.setProductId(request.productId());
        order.setQuantity(request.quantity());

        Order updatedOrder = orderRepository.save(order);

        return toResponse(updatedOrder);
    }

    public void deleteOrder(Long id) {

        if (!orderRepository.existsById(id)) {
            throw new OrderNotFoundException(id);
        }

        orderRepository.deleteById(id);
    }

    private OrderResponse toResponse(Order order) {

        return new OrderResponse(
                order.getId(),
                order.getCustomerId(),
                order.getProductId(),
                order.getQuantity(),
                order.getStatus()
        );
    }
}
```

---

# 13. REST Controller

Create:

```text
src/main/java/com/company/order/controller/OrderController.java
```

```java
package com.company.order.controller;

import com.company.order.dto.CreateOrderRequest;
import com.company.order.dto.OrderResponse;
import com.company.order.service.OrderService;

import jakarta.validation.Valid;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(OrderService orderService) {
        this.orderService = orderService;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public OrderResponse createOrder(
            @Valid @RequestBody CreateOrderRequest request) {

        return orderService.createOrder(request);
    }

    @GetMapping("/{id}")
    public OrderResponse getOrder(@PathVariable Long id) {

        return orderService.getOrder(id);
    }

    @GetMapping
    public List<OrderResponse> getOrders() {

        return orderService.getAllOrders();
    }

    @PutMapping("/{id}")
    public OrderResponse updateOrder(
            @PathVariable Long id,
            @Valid @RequestBody CreateOrderRequest request) {

        return orderService.updateOrder(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteOrder(@PathVariable Long id) {

        orderService.deleteOrder(id);
    }
}
```

---

# 14. Global Exception Handler

Create:

```text
src/main/java/com/company/order/exception/GlobalExceptionHandler.java
```

```java
package com.company.order.exception;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public Map<String, String> handleOrderNotFound(
            OrderNotFoundException exception) {

        return Map.of(
                "error", "ORDER_NOT_FOUND",
                "message", exception.getMessage()
        );
    }
}
```

Now:

```text
GET /api/v1/orders/999
```

can return:

```json
{
  "error": "ORDER_NOT_FOUND",
  "message": "Order not found: 999"
}
```

instead of an uncontrolled stack trace.

---

# 15. Application Configuration

Create:

```text
src/main/resources/application.yaml
```

```yaml
server:
  port: 8080

spring:

  application:
    name: order-service

  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/orders}
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}

  jpa:
    hibernate:
      ddl-auto: update
    show-sql: false

    properties:
      hibernate:
        format_sql: true

logging:
  level:
    root: INFO
    com.company.order: INFO

management:

  endpoints:
    web:
      exposure:
        include: health,info,prometheus

  endpoint:
    health:
      probes:
        enabled: true
```

---

# 16. Why Environment Variables?

Notice:

```yaml
username: ${DB_USERNAME:postgres}
password: ${DB_PASSWORD:postgres}
```

Locally:

```text
DB_USERNAME=postgres
DB_PASSWORD=postgres
```

Production:

```text
DB_USERNAME=<secret>
DB_PASSWORD=<secret>
```

Later:

```text
AWS Secrets Manager
        ↓
External Secrets Operator
        ↓
Kubernetes Secret
        ↓
Environment Variables
        ↓
Spring Boot
```

We don't hardcode production credentials.

---

# 17. Important Production Difference

For local development:

```yaml
ddl-auto: update
```

is convenient.

For production:

```yaml
ddl-auto: validate
```

should be used with a migration framework such as:

```text
Flyway
```

or:

```text
Liquibase
```

Eventually:

```text
Developer
   ↓
Flyway migration
   ↓
CI validation
   ↓
Controlled migration
   ↓
RDS
   ↓
Application deployment
```

We will implement this properly before production deployment.

---

# 18. Production Configuration

Create:

```text
src/main/resources/application-prod.yaml
```

```yaml
spring:

  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}

  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false

logging:
  level:
    root: INFO
    com.company.order: INFO

management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus

  endpoint:
    health:
      probes:
        enabled: true
```

---

# 19. Build the Application

Run:

```bash
./mvnw clean verify
```

Expected:

```text
BUILD SUCCESS
```

The JAR should appear under:

```text
target/
```

For example:

```text
target/order-service-1.0.0.jar
```

---

# 20. Start PostgreSQL Locally

We will use Docker instead of installing PostgreSQL directly on your machine.

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

volumes:

  postgres-data:
```

Start:

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Expected:

```text
order-postgres
running
healthy
```

---

# 21. Run the Application

Start Spring Boot:

```bash
./mvnw spring-boot:run
```

Expected:

```text
Tomcat started on port 8080
Started OrderServiceApplication
```

Now:

```text
Application
     │
     ▼
localhost:8080
     │
     ▼
PostgreSQL
localhost:5432
```

---

# 22. Health Check

Run:

```bash
curl http://localhost:8080/actuator/health
```

Expected:

```json
{
  "status": "UP"
}
```

This endpoint becomes extremely important later.

Kubernetes will use:

```text
startupProbe
readinessProbe
livenessProbe
```

based on application health endpoints.

---

# 23. Create an Order

Run:

```bash
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 1001,
    "productId": 501,
    "quantity": 2
  }'
```

Expected:

```json
{
  "id": 1,
  "customerId": 1001,
  "productId": 501,
  "quantity": 2,
  "status": "CREATED"
}
```

---

# 24. Get the Order

```bash
curl http://localhost:8080/api/v1/orders/1
```

Expected:

```json
{
  "id": 1,
  "customerId": 1001,
  "productId": 501,
  "quantity": 2,
  "status": "CREATED"
}
```

---

# 25. Get All Orders

```bash
curl http://localhost:8080/api/v1/orders
```

Expected:

```json
[
  {
    "id": 1,
    "customerId": 1001,
    "productId": 501,
    "quantity": 2,
    "status": "CREATED"
  }
]
```

---

# 26. Update the Order

```bash
curl -X PUT http://localhost:8080/api/v1/orders/1 \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 1001,
    "productId": 501,
    "quantity": 5
  }'
```

Expected:

```json
{
  "id": 1,
  "customerId": 1001,
  "productId": 501,
  "quantity": 5,
  "status": "CREATED"
}
```

---

# 27. Delete the Order

```bash
curl -X DELETE \
  http://localhost:8080/api/v1/orders/1
```

Expected HTTP status:

```text
204 No Content
```

---

# 28. Test Not Found

```bash
curl http://localhost:8080/api/v1/orders/999
```

Expected:

```json
{
  "error": "ORDER_NOT_FOUND",
  "message": "Order not found: 999"
}
```

HTTP status:

```text
404
```

---

# 29. Test Validation

Try:

```bash
curl -X POST http://localhost:8080/api/v1/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 1001,
    "productId": 501,
    "quantity": 0
  }'
```

The request should be rejected because:

```java
@Min(1)
```

requires quantity to be at least 1.

---

# 30. Verify PostgreSQL

Connect:

```bash
docker exec -it order-postgres \
  psql -U postgres -d orders
```

Then:

```sql
\dt
```

You should see:

```text
orders
```

Query:

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

# 31. Prometheus Endpoint

Because we added:

```xml
micrometer-registry-prometheus
```

Spring Boot exposes:

```text
/actuator/prometheus
```

Run:

```bash
curl http://localhost:8080/actuator/prometheus
```

You'll see metrics such as:

```text
jvm_memory_used_bytes
jvm_threads_live_threads
http_server_requests_seconds_count
process_cpu_usage
```

Later:

```text
Order Service
     │
     ▼
/actuator/prometheus
     │
     ▼
Prometheus
     │
     ▼
Grafana
```

---

# 32. Dockerfile

Create:

```text
Dockerfile
```

```dockerfile
FROM eclipse-temurin:21-jdk AS builder

WORKDIR /build

COPY .mvn/ .mvn/
COPY mvnw pom.xml ./

RUN chmod +x mvnw

RUN ./mvnw dependency:go-offline

COPY src ./src

RUN ./mvnw clean package -DskipTests


FROM eclipse-temurin:21-jre

WORKDIR /app

COPY --from=builder /build/target/*.jar app.jar

EXPOSE 8080

USER 10001

ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

# 33. Why Multi-Stage Docker Build?

Instead of:

```text
JDK
Maven
source code
dependencies
application
```

inside the production image, we use:

```text
Builder Image
     │
     ├── JDK
     ├── Maven
     └── source
           │
           ▼
          JAR
           │
           ▼
Runtime Image
     │
     ├── JRE
     └── JAR
```

The final image doesn't need the source code or Maven tooling.

---

# 34. Run Docker Build

```bash
docker build \
  -t order-service:1.0.0 .
```

Check:

```bash
docker images | grep order-service
```

---

# 35. Run Container

Stop the locally running Spring Boot process first.

Then:

```bash
docker run -d \
  --name order-service \
  -p 8080:8080 \
  -e DB_URL=jdbc:postgresql://host.docker.internal:5432/orders \
  -e DB_USERNAME=postgres \
  -e DB_PASSWORD=postgres \
  order-service:1.0.0
```

Check:

```bash
docker ps
```

Logs:

```bash
docker logs order-service
```

Health:

```bash
curl http://localhost:8080/actuator/health
```

---

# 36. Important Container Networking Difference

When Spring Boot runs directly on your laptop:

```text
Application
    ↓
localhost:5432
    ↓
PostgreSQL
```

When application runs inside Docker:

```text
Application Container
        ↓
Docker network
        ↓
PostgreSQL Container
```

Therefore:

```text
localhost
```

inside a container means:

> **the container itself**

It does not mean your laptop or another container.

With Compose, we'll use:

```text
postgres:5432
```

instead of:

```text
localhost:5432
```

---

# 37. Better Docker Compose

Now combine the application and database.

Update `docker-compose.yaml`:

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

Start:

```bash
docker compose up -d --build
```

Check:

```bash
docker compose ps
```

Expected:

```text
order-postgres
order-service
```

---

# 38. Test Complete Local Architecture

Now:

```text
curl
 │
 ▼
localhost:8080
 │
 ▼
order-service container
 │
 ▼
postgres:5432
 │
 ▼
PostgreSQL container
```

Test:

```bash
curl -X POST http://localhost:8080/api/v1/orders \
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

# 39. Docker Troubleshooting

### Application container exits

```bash
docker logs order-service
```

---

### PostgreSQL isn't ready

```bash
docker compose ps
```

Then:

```bash
docker logs order-postgres
```

---

### Application can't connect to DB

Check:

```bash
docker exec order-service \
  getent hosts postgres
```

The hostname should resolve.

Then check:

```text
DB_URL
```

It should be:

```text
jdbc:postgresql://postgres:5432/orders
```

Not:

```text
jdbc:postgresql://localhost:5432/orders
```

---

### Port 8080 already occupied

Linux:

```bash
sudo ss -lntp | grep :8080
```

Windows PowerShell:

```powershell
Get-NetTCPConnection -LocalPort 8080
```

---

# 40. Add Tests

Create:

```text
src/test/java/com/company/order/
```

Start with a service/controller test suite.

At minimum we want:

```text
Order creation
Order retrieval
Order update
Order deletion
Order not found
Validation
```

The CI pipeline later will enforce:

```text
Code
 ↓
Compile
 ↓
Unit Tests
 ↓
Integration Tests
 ↓
SonarQube
```

---

# 41. Git Commit

Once local testing works:

```bash
git status
```

Then:

```bash
git add .
```

Commit:

```bash
git commit -m "feat: implement order service"
```

Push:

```bash
git push origin main
```

**If `main` is already protected**, don't push directly. Create:

```bash
git checkout -b feature/implement-order-service
```

Then:

```bash
git push -u origin feature/implement-order-service
```

and create the PR.

---

# 42. What We Have Now

We have built:

```text
                 LOCAL ENVIRONMENT

                curl / Postman
                      │
                      ▼
                REST Controller
                      │
                      ▼
                 Service Layer
                      │
                      ▼
                JPA Repository
                      │
                      ▼
                  PostgreSQL
```

And containerized it:

```text
Docker Compose
│
├── order-service
│       │
│       └── Spring Boot
│
└── postgres
        │
        └── PostgreSQL
```

---

# 43. What This Is NOT Yet

We have **not** yet implemented:

```text
❌ GitHub Actions CI
❌ SonarQube
❌ Gitleaks
❌ Trivy
❌ ECR
❌ Terraform
❌ EKS
❌ Helm
❌ ArgoCD
❌ RDS
❌ Redis
❌ SQS/SNS
❌ Secrets Manager
❌ Prometheus server
❌ Grafana
❌ ELK
❌ OpenTelemetry
❌ Jaeger
```

That's intentional.

We are building the system **layer by layer**.

---

# 44. Current Progress

```text
[✓] Repository architecture
[✓] GitHub organization structure
[✓] Order Service repository
[✓] Spring Boot application
[✓] REST APIs
[✓] PostgreSQL integration
[✓] Validation
[✓] Exception handling
[✓] Actuator
[✓] Prometheus endpoint
[✓] Dockerfile
[✓] Docker Compose
[✓] Local application testing

[ ] CI
[ ] Security pipeline
[ ] ECR
[ ] AWS infrastructure
[ ] EKS
[ ] Helm
[ ] GitOps
[ ] ArgoCD
[ ] Production database
...
```

---

# 45. The Critical Transition

The next stage is:

```text
LOCAL
────────────────────────

Developer
   ↓
Spring Boot
   ↓
Docker Compose
   ↓
PostgreSQL


                    ↓


AWS
────────────────────────

GitHub
   ↓
GitHub Actions
   ↓
SonarQube
   ↓
Trivy
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
RDS
```

We **do not jump directly from local Docker to EKS**.

The next chapter establishes the local container workflow properly first.

## Next: `55-Local-Docker-Environment.md`

We will make the local environment closer to the eventual production architecture, including **PostgreSQL, application networking, environment variables, health checks, persistent volumes, container logs, rebuilds, failure testing, and troubleshooting** before moving to Terraform/AWS.
