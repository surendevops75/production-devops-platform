# 04 — Microservice Foundation (Order Service)

Now we stop designing and start building a **real microservice**.

Our goal:

```text
Developer
   |
   v
Code
   |
   v
Local Run
   |
   v
Docker
   |
   v
ECR
   |
   v
EKS
```

Before Kubernetes, Terraform, ArgoCD, Prometheus, ELK, etc., we need a proper application.

---

# 1. What Are We Building?

Service:

```text
order-service
```

Responsibilities:

```text
Create Order
Get Order
Update Order
Cancel Order
List Orders
```

API:

```http
POST   /api/v1/orders

GET    /api/v1/orders/{id}

GET    /api/v1/orders

PUT    /api/v1/orders/{id}

DELETE /api/v1/orders/{id}
```

---

# 2. Real Production Architecture

```text
                    Client
                       |
                       v
                OrderController
                       |
                       v
                  OrderService
                       |
                       v
                OrderRepository
                       |
                       v
                  PostgreSQL
```

Later:

```text
                    Order Service
                           |
      +--------------------+-------------------+
      |                    |                   |
      v                    v                   v
 PostgreSQL            Redis Cache          SQS
      |
      v
 OpenTelemetry
      |
      v
 Jaeger

 Logs
      |
      v
 FluentBit
      |
      v
 Logstash
      |
      v
 Elasticsearch
      |
      v
 Kibana

 Metrics
      |
      v
 Prometheus
      |
      v
 Grafana
```

---

# 3. Repository Structure

```text
order-service/
│
├── src/
│
│   ├── main/
│   │
│   ├── java/
│   │
│   └── resources/
│
├── test/
│
├── Dockerfile
├── pom.xml
├── README.md
│
├── deploy/
│
├── helm/
│
└── .github/
```

---

# 4. Java Package Structure

```text
src/main/java/com/company/order/
│
├── OrderServiceApplication.java
│
├── controller/
│   └── OrderController.java
│
├── service/
│   └── OrderService.java
│
├── repository/
│   └── OrderRepository.java
│
├── entity/
│   └── Order.java
│
├── dto/
│   ├── CreateOrderRequest.java
│   └── OrderResponse.java
│
├── exception/
│   ├── OrderNotFoundException.java
│   └── GlobalExceptionHandler.java
│
└── config/
```

This is a standard Spring Boot structure.

---

# 5. pom.xml

Dependencies:

```xml
<dependencies>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-actuator</artifactId>
    </dependency>

    <dependency>
        <groupId>io.micrometer</groupId>
        <artifactId>micrometer-registry-prometheus</artifactId>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>

</dependencies>
```

Why Actuator?

Because later:

```text
Prometheus
      |
      v
/actuator/prometheus
```

---

# 6. Main Class

```java
@SpringBootApplication
public class OrderServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(OrderServiceApplication.class, args);
    }

}
```

---

# 7. Order Entity

```java
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String customerId;

    private String productId;

    private Integer quantity;

    private String status;

}
```

Database table:

```sql
CREATE TABLE orders(
    id BIGSERIAL PRIMARY KEY,
    customer_id VARCHAR(100),
    product_id VARCHAR(100),
    quantity INT,
    status VARCHAR(50)
);
```

---

# 8. DTO

Request:

```java
public class CreateOrderRequest {

    private String customerId;

    private String productId;

    private Integer quantity;

}
```

Response:

```java
public class OrderResponse {

    private Long id;

    private String customerId;

    private String productId;

    private Integer quantity;

    private String status;

}
```

---

# 9. Repository

```java
@Repository
public interface OrderRepository
extends JpaRepository<Order, Long> {

}
```

Spring automatically generates CRUD operations.

---

# 10. Service Layer

```java
@Service
public class OrderService {

    private final OrderRepository repository;

    public OrderService(OrderRepository repository) {
        this.repository = repository;
    }

}
```

Business logic belongs here.

Not inside controller.

---

# 11. Create Order

```java
public Order createOrder(CreateOrderRequest request){

    Order order = new Order();

    order.setCustomerId(request.getCustomerId());
    order.setProductId(request.getProductId());
    order.setQuantity(request.getQuantity());
    order.setStatus("CREATED");

    return repository.save(order);
}
```

---

# 12. Get Order

```java
public Order getOrder(Long id){

    return repository.findById(id)
            .orElseThrow(
                 () -> new OrderNotFoundException(id)
            );
}
```

---

# 13. Update Order

```java
public Order updateOrder(
        Long id,
        CreateOrderRequest request){

    Order order = getOrder(id);

    order.setProductId(request.getProductId());
    order.setQuantity(request.getQuantity());

    return repository.save(order);
}
```

---

# 14. Delete Order

```java
public void deleteOrder(Long id){

    Order order = getOrder(id);

    repository.delete(order);
}
```

---

# 15. Controller

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

}
```

---

# 16. Create Endpoint

```java
@PostMapping
public ResponseEntity<Order> createOrder(
        @RequestBody CreateOrderRequest request){

    return ResponseEntity.ok(
        service.createOrder(request)
    );
}
```

---

# 17. Get Endpoint

```java
@GetMapping("/{id}")
public ResponseEntity<Order> getOrder(
        @PathVariable Long id){

    return ResponseEntity.ok(
        service.getOrder(id)
    );
}
```

---

# 18. Delete Endpoint

```java
@DeleteMapping("/{id}")
public ResponseEntity<Void> deleteOrder(
        @PathVariable Long id){

    service.deleteOrder(id);

    return ResponseEntity.noContent().build();
}
```

---

# 19. Global Exception Handler

Production services should never dump stack traces to users.

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

}
```

---

# 20. Handle Not Found

```java
@ExceptionHandler(OrderNotFoundException.class)
public ResponseEntity<String> handleNotFound(
        OrderNotFoundException ex){

    return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(ex.getMessage());
}
```

---

# 21. application.yaml

Local configuration:

```yaml
server:
  port: 8080

spring:

  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: postgres
    password: postgres

  jpa:
    hibernate:
      ddl-auto: update

    show-sql: true
```

---

# 22. Production Configuration

Never hardcode production values.

```yaml
spring:

  datasource:

    url: ${DB_URL}

    username: ${DB_USERNAME}

    password: ${DB_PASSWORD}
```

These will later come from:

```text
AWS Secrets Manager
        |
        v
External Secrets
        |
        v
Kubernetes Secret
        |
        v
Pod
```

---

# 23. Logging Configuration

Structured logging is critical.

```yaml
logging:

  level:
    root: INFO

    com.company.order: INFO
```

Later:

```text
Application
    |
    v
stdout
    |
    v
Fluent Bit
    |
    v
Logstash
    |
    v
Elasticsearch
    |
    v
Kibana
```

---

# 24. Health Checks

Enable Actuator.

```yaml
management:

  endpoints:
    web:
      exposure:
        include: health,prometheus,info

  endpoint:
    health:
      probes:
        enabled: true
```

Now:

```text
/actuator/health
```

works.

---

# 25. Kubernetes Readiness

Later EKS uses:

```yaml
readinessProbe:

  httpGet:
    path: /actuator/health/readiness
    port: 8080
```

---

# 26. Kubernetes Liveness

```yaml
livenessProbe:

  httpGet:
    path: /actuator/health/liveness
    port: 8080
```

Without these, production outages become harder to manage.

---

# 27. Prometheus Metrics

Expose:

```text
/actuator/prometheus
```

Prometheus later scrapes:

```yaml
- targets:
    - order-service:8080
```

---

# 28. OpenTelemetry Preparation

Later we add:

```xml
opentelemetry-javaagent
```

Then traces flow:

```text
Order Service
      |
      v
OpenTelemetry
      |
      v
OTEL Collector
      |
      v
Jaeger
```

---

# 29. Local PostgreSQL

Run locally:

```bash
docker run -d \
--name postgres \
-e POSTGRES_DB=orders \
-e POSTGRES_USER=postgres \
-e POSTGRES_PASSWORD=postgres \
-p 5432:5432 \
postgres:16
```

Verify:

```bash
docker ps
```

---

# 30. Start Application

```bash
./mvnw spring-boot:run
```

Expected:

```text
Tomcat started on port 8080
```

---

# 31. Create Order Test

```bash
curl -X POST \
http://localhost:8080/api/v1/orders \
-H "Content-Type: application/json" \
-d '
{
 "customerId":"cust1",
 "productId":"prod1",
 "quantity":2
}'
```

Response:

```json
{
 "id":1,
 "customerId":"cust1",
 "productId":"prod1",
 "quantity":2,
 "status":"CREATED"
}
```

---

# 32. Get Order

```bash
curl http://localhost:8080/api/v1/orders/1
```

---

# 33. Production Improvements Needed

Current service is NOT production-ready.

Missing:

```text
Validation
Unit Tests
Integration Tests
Docker
Security Headers
Swagger
Tracing
Caching
Messaging
Rate Limiting
Authentication
Authorization
Circuit Breaker
Retry Logic
Observability
```

We'll add all of these.

---

# 34. Final Architecture After This Phase

```text
Client
  |
  v
OrderController
  |
  v
OrderService
  |
  v
OrderRepository
  |
  v
PostgreSQL
```

---

# 35. Next Phase

Before AWS or Kubernetes, we must containerize the service.

Next:

```text
05-Dockerization.md
```

We will build:

```text
Order Service
      |
      v
Dockerfile
      |
      v
Docker Image
      |
      v
Docker Compose
      |
      v
Local Environment
      |
      v
Amazon ECR
```

Including:

```text
Multi-stage Dockerfile
Non-root user
Image optimization
Health checks
Security best practices
Docker Compose with PostgreSQL
Local development workflow
Image scanning
Production-ready image design
```

This is exactly how a real microservice starts before it ever reaches EKS.
