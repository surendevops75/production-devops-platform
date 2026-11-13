# 170 — Order Service

**File:** `170-Order-Service.md`

```markdown
# 170 — Order Service

## 1. Overview

The `order-service` is one of the most critical services in the e-commerce
platform.

It owns the order lifecycle after a customer decides to purchase the items in
the cart.

Unlike the Cart Service, which represents temporary customer intent, the Order
Service manages a durable business transaction.

Technology:

- Java 21
- Spring Boot
- Maven
- PostgreSQL
- Redis
- Kafka
- SQS
- Docker
- Kubernetes
- Helm
- AWS EKS
- Amazon ECR
- GitHub Actions
- ArgoCD
- Prometheus
- Grafana
- ELK
- OpenTelemetry
- Jaeger

Primary responsibilities:

- Create order
- Retrieve order
- Update order state
- Order lifecycle management
- Idempotency
- Price validation
- Inventory reservation coordination
- Payment coordination
- Tax coordination
- Shipping coordination
- Order events
- Transactional Outbox
- Saga orchestration
- Order status history
- Failure recovery
- Auditability

---

# 2. Why Order Service Is Different

The Order Service sits in the middle of the most important business workflow.

A simplified flow is:

Customer:

```text
Cart
 ↓
Checkout
 ↓
Order
 ↓
Inventory
 ↓
Payment
 ↓
Shipping
 ↓
Delivery
```

The order service therefore has to coordinate several distributed systems.

```text
                    ┌── Pricing
                    │
                    ├── Promotion
                    │
Cart ──→ Order ─────┼── Inventory
                    │
                    ├── Payment
                    │
                    ├── Tax
                    │
                    └── Shipping
```

This creates distributed transaction problems.

We cannot use one PostgreSQL transaction across all 21 microservices.

Therefore the project uses:

```text
Saga
+
Transactional Outbox
+
Idempotency
+
Event-driven communication
```

---

# 3. Order Service Ownership

Order Service owns:

- Order
- Order items
- Order status
- Order totals
- Order state transitions
- Order history
- Order idempotency
- Order events
- Saga state

It does not own:

- Product master
- Inventory quantity
- Payment account
- Shipping carrier
- Tax rules
- Customer profile

Those remain owned by their respective services.

---

# 4. Order Lifecycle

Example lifecycle:

```text
CREATED
   ↓
PENDING_VALIDATION
   ↓
INVENTORY_RESERVED
   ↓
PAYMENT_AUTHORIZED
   ↓
CONFIRMED
   ↓
PROCESSING
   ↓
SHIPPED
   ↓
DELIVERED
```

Failure paths:

```text
INVENTORY_FAILED
PAYMENT_FAILED
CANCELLED
REFUNDED
```

Example:

```text
CREATED
   ↓
INVENTORY_RESERVED
   ↓
PAYMENT_FAILED
   ↓
INVENTORY_RELEASED
   ↓
CANCELLED
```

---

# 5. High-Level Architecture

```text
                              Customer
                                  |
                                  ↓
                              ALB / API
                                  |
                                  ↓
                            Order Service
                           /      |       \
                          /       |        \
                         ↓        ↓         ↓
                    PostgreSQL  Redis      Kafka
                         |                  |
                         |                  ↓
                         |              Event Consumers
                         |
                         ↓
                 Transactional Outbox
```

External synchronous dependencies:

```text
Order
 |
 ├── Cart
 ├── Customer
 ├── Product
 ├── Pricing
 ├── Promotion
 └── Tax
```

Saga/event-driven dependencies:

```text
Order
 |
 ├── Inventory
 ├── Payment
 ├── Fraud
 ├── Shipping
 ├── Notification
 └── Reporting
```

---

# 6. Repository Structure

```text
order-service/
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/company/order/
│   │   │
│   │   │       ├── OrderServiceApplication.java
│   │   │
│   │   │       ├── controller/
│   │   │       │   └── OrderController.java
│   │   │
│   │   │       ├── service/
│   │   │       │   ├── OrderService.java
│   │   │       │   ├── OrderSagaService.java
│   │   │       │   └── OrderStateMachine.java
│   │   │
│   │   │       ├── repository/
│   │   │       │   ├── OrderRepository.java
│   │   │       │   ├── OrderItemRepository.java
│   │   │       │   ├── OutboxRepository.java
│   │   │       │   └── IdempotencyRepository.java
│   │   │
│   │   │       ├── entity/
│   │   │       │   ├── Order.java
│   │   │       │   ├── OrderItem.java
│   │   │       │   ├── OutboxEvent.java
│   │   │       │   └── IdempotencyRecord.java
│   │   │
│   │   │       ├── dto/
│   │   │       │   ├── CreateOrderRequest.java
│   │   │       │   ├── OrderResponse.java
│   │   │       │   └── OrderItemRequest.java
│   │   │
│   │   │       ├── client/
│   │   │       │   ├── CustomerClient.java
│   │   │       │   ├── PricingClient.java
│   │   │       │   ├── PromotionClient.java
│   │   │       │   └── TaxClient.java
│   │   │
│   │   │       ├── messaging/
│   │   │       │   ├── KafkaProducer.java
│   │   │       │   ├── KafkaConsumer.java
│   │   │       │   └── EventPublisher.java
│   │   │
│   │   │       ├── exception/
│   │   │       │   ├── OrderNotFoundException.java
│   │   │       │   ├── IdempotencyConflictException.java
│   │   │       │   └── GlobalExceptionHandler.java
│   │   │       │
│   │   │       └── config/
│   │   │           ├── RedisConfig.java
│   │   │           ├── KafkaConfig.java
│   │   │           └── WebClientConfig.java
│   │   │
│   │   └── resources/
│   │       ├── application.yaml
│   │       ├── application-prod.yaml
│   │       └── db/
│   │           └── migration/
│   │
│   └── test/
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── pom.xml
├── README.md
│
├── helm/
│   └── order-service/
│
└── .github/
    ├── CODEOWNERS
    └── workflows/
        ├── ci.yaml
        └── security.yaml
```

---

# 7. Maven Dependencies

## `pom.xml`

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="
         http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>
            org.springframework.boot
        </groupId>

        <artifactId>
            spring-boot-starter-parent
        </artifactId>

        <version>
            3.5.0
        </version>

        <relativePath/>
    </parent>

    <groupId>
        com.company
    </groupId>

    <artifactId>
        order-service
    </artifactId>

    <version>
        1.0.0
    </version>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>

        <dependency>
            <groupId>
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-web
            </artifactId>
        </dependency>

        <dependency>
            <groupId>
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-data-jpa
            </artifactId>
        </dependency>

        <dependency>
            <groupId>
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-data-redis
            </artifactId>
        </dependency>

        <dependency>
            <groupId>
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-kafka
            </artifactId>
        </dependency>

        <dependency>
            <groupId>
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-validation
            </artifactId>
        </dependency>

        <dependency>
            <groupId>
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-actuator
            </artifactId>
        </dependency>

        <dependency>
            <groupId>
                org.postgresql
            </groupId>

            <artifactId>
                postgresql
            </artifactId>

            <scope>
                runtime
            </scope>
        </dependency>

        <dependency>
            <groupId>
                io.micrometer
            </groupId>

            <artifactId>
                micrometer-registry-prometheus
            </artifactId>
        </dependency>

        <dependency>
            <groupId>
                org.flywaydb
            </groupId>

            <artifactId>
                flyway-core
            </artifactId>
        </dependency>

        <dependency>
            <groupId>
                org.flywaydb
            </groupId>

            <artifactId>
                flyway-database-postgresql
            </artifactId>
        </dependency>

        <dependency>
            <groupId>
                org.projectlombok
            </groupId>

            <artifactId>
                lombok
            </artifactId>

            <optional>true</optional>
        </dependency>

        <dependency>
            <groupId>
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-test
            </artifactId>

            <scope>
                test
            </scope>
        </dependency>

    </dependencies>

    <build>

        <plugins>

            <plugin>
                <groupId>
                    org.springframework.boot
                </groupId>

                <artifactId>
                    spring-boot-maven-plugin
                </artifactId>
            </plugin>

        </plugins>

    </build>

</project>
```

---

# 8. Application Configuration

## `application.yaml`

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

    hikari:

      maximum-pool-size: 10

      minimum-idle: 2

      connection-timeout: 3000

      validation-timeout: 1000

      idle-timeout: 600000

      max-lifetime: 1500000

      pool-name: order-service-hikari


  jpa:

    hibernate:

      ddl-auto: update

    show-sql: false

    properties:

      hibernate:

        format_sql: true


  data:

    redis:

      host: ${REDIS_HOST:localhost}

      port: ${REDIS_PORT:6379}

      timeout: 1000ms


  kafka:

    bootstrap-servers:
      ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}

    producer:

      key-serializer:
        org.apache.kafka.common.serialization.StringSerializer

      value-serializer:
        org.springframework.kafka.support.serializer.JsonSerializer

    consumer:

      group-id: order-service

      key-deserializer:
        org.apache.kafka.common.serialization.StringDeserializer

      value-deserializer:
        org.springframework.kafka.support.serializer.JsonDeserializer


management:

  endpoints:

    web:

      exposure:

        include: health,info,prometheus

  endpoint:

    health:

      probes:

        enabled: true


logging:

  level:

    root: INFO

    com.company.order: INFO


order:

  idempotency-ttl-seconds: 86400

  max-items: 100
```

---

# 9. Production Configuration

## `application-prod.yaml`

```yaml
server:

  port: 8080


spring:

  application:

    name: order-service


  datasource:

    url: ${DB_URL}

    username: ${DB_USERNAME}

    password: ${DB_PASSWORD}

    hikari:

      maximum-pool-size: 10

      minimum-idle: 2

      connection-timeout: 3000

      validation-timeout: 1000

      idle-timeout: 600000

      max-lifetime: 1500000

      pool-name: order-service-hikari


  jpa:

    hibernate:

      ddl-auto: validate

    show-sql: false

    open-in-view: false


  data:

    redis:

      host: ${REDIS_HOST}

      port: ${REDIS_PORT:6379}

      timeout: 1000ms


  kafka:

    bootstrap-servers:
      ${KAFKA_BOOTSTRAP_SERVERS}

    producer:

      acks: all

      retries: 5

      enable-idempotence: true

      compression-type: zstd


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

Pool sizes are illustrative.

The production value must be calculated using the database connection budget,
replica limits, deployment surge, and load testing.

---

# 10. Main Application

## `OrderServiceApplication.java`

```java
package com.company.order;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class OrderServiceApplication {

    public static void main(String[] args) {

        SpringApplication.run(
            OrderServiceApplication.class,
            args
        );
    }
}
```

---

# 11. Order Entity

## `Order.java`

```java
package com.company.order.entity;

import jakarta.persistence.*;

import java.math.BigDecimal;
import java.time.Instant;

@Entity
@Table(
    name = "orders",
    indexes = {
        @Index(
            name = "idx_orders_customer_id",
            columnList = "customer_id"
        ),
        @Index(
            name = "idx_orders_status",
            columnList = "status"
        )
    }
)
public class Order {

    @Id
    @GeneratedValue(
        strategy = GenerationType.IDENTITY
    )
    private Long id;

    @Column(
        name = "order_number",
        nullable = false,
        unique = true
    )
    private String orderNumber;

    @Column(
        name = "customer_id",
        nullable = false
    )
    private String customerId;

    @Column(
        nullable = false
    )
    private String status;

    @Column(
        precision = 19,
        scale = 4,
        nullable = false
    )
    private BigDecimal subtotal;

    @Column(
        precision = 19,
        scale = 4,
        nullable = false
    )
    private BigDecimal discount;

    @Column(
        precision = 19,
        scale = 4,
        nullable = false
    )
    private BigDecimal tax;

    @Column(
        precision = 19,
        scale = 4,
        nullable = false
    )
    private BigDecimal total;

    @Column(
        length = 3,
        nullable = false
    )
    private String currency;

    @Column(
        nullable = false
    )
    private Instant createdAt;

    @Column(
        nullable = false
    )
    private Instant updatedAt;

    @Version
    private Long version;

    // getters/setters omitted for brevity
}
```

In the actual repository, generate the complete getters/setters or use a
consistent project-wide Lombok policy.

---

# 12. Order Item

## `OrderItem.java`

```java
@Entity
@Table(
    name = "order_items",
    indexes = {
        @Index(
            name = "idx_order_items_order_id",
            columnList = "order_id"
        )
    }
)
public class OrderItem {

    @Id
    @GeneratedValue(
        strategy = GenerationType.IDENTITY
    )
    private Long id;

    @ManyToOne(
        fetch = FetchType.LAZY
    )
    @JoinColumn(
        name = "order_id",
        nullable = false
    )
    private Order order;

    @Column(
        name = "product_id",
        nullable = false
    )
    private String productId;

    @Column(
        nullable = false
    )
    private Integer quantity;

    @Column(
        precision = 19,
        scale = 4,
        nullable = false
    )
    private BigDecimal unitPrice;

    @Column(
        precision = 19,
        scale = 4,
        nullable = false
    )
    private BigDecimal lineTotal;

    @Column(
        length = 3,
        nullable = false
    )
    private String currency;
}
```

---

# 13. Order Status History

Production systems should preserve state transitions.

Example:

```text
order_status_history

ORDER_CREATED
INVENTORY_RESERVED
PAYMENT_AUTHORIZED
ORDER_CONFIRMED
ORDER_SHIPPED
ORDER_DELIVERED
```

This is valuable for:

- Audit
- Customer support
- RCA
- Business analytics
- Incident investigation
- Compliance

---

# 14. Outbox Entity

## `OutboxEvent.java`

```java
@Entity
@Table(
    name = "outbox_events",
    indexes = {
        @Index(
            name = "idx_outbox_status_created",
            columnList = "status,created_at"
        )
    }
)
public class OutboxEvent {

    @Id
    @GeneratedValue(
        strategy = GenerationType.IDENTITY
    )
    private Long id;

    @Column(
        nullable = false,
        unique = true
    )
    private String eventId;

    @Column(
        nullable = false
    )
    private String eventType;

    @Column(
        nullable = false
    )
    private String aggregateId;

    @Column(
        nullable = false,
        columnDefinition = "TEXT"
    )
    private String payload;

    @Column(
        nullable = false
    )
    private String status;

    @Column(
        nullable = false
    )
    private Instant createdAt;

    private Instant publishedAt;

    private Integer retryCount;
}
```

---

# 15. Why Transactional Outbox Is Required

Bad design:

```text
DB transaction
    ↓
commit

then

Kafka publish
```

What if:

```text
DB commit = SUCCESS
Kafka publish = FAILURE
```

The order exists but downstream services never receive the event.

The reverse is also dangerous:

```text
Kafka publish = SUCCESS
DB commit = FAILURE
```

Consumers may receive an order that does not exist.

Transactional Outbox solves this.

---

# 16. Transactional Outbox Pattern

```text
                 PostgreSQL Transaction
                         |
             ┌───────────┴───────────┐
             ↓                       ↓
         orders table          outbox_events
             |                       |
             └───────────┬───────────┘
                         |
                       COMMIT
                         |
                         ↓
                  Outbox Publisher
                         |
                         ↓
                       Kafka
                         |
             ┌───────────┼───────────┐
             ↓           ↓           ↓
         Inventory     Payment     Notification
```

Order and outbox event are committed together.

---

# 17. Create Order API

```text
POST /api/v1/orders
```

Headers:

```text
Authorization: Bearer <token>

Idempotency-Key: 8f7c...
X-Correlation-ID: corr-123
```

Request:

```json
{
  "customerId": "C1001",
  "items": [
    {
      "productId": "P1001",
      "quantity": 2
    }
  ],
  "shippingAddressId": "ADDR100"
}
```

Important:

The authenticated customer identity should be derived from the token rather
than blindly trusting a customer ID supplied by the caller.

---

# 18. Order Response

```json
{
  "orderId": "100001",
  "orderNumber": "ORD-2026-000001",
  "customerId": "C1001",
  "status": "PENDING_VALIDATION",
  "subtotal": "18998.00",
  "discount": "1000.00",
  "tax": "3239.64",
  "total": "21237.64",
  "currency": "INR"
}
```

---

# 19. Order Controller

## `OrderController.java`

```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {

    private final OrderService orderService;

    public OrderController(
        OrderService orderService
    ) {
        this.orderService = orderService;
    }

    @PostMapping
    public ResponseEntity<OrderResponse>
    createOrder(

        @RequestHeader("Idempotency-Key")
        String idempotencyKey,

        @RequestHeader(
            value = "X-Correlation-ID",
            required = false
        )
        String correlationId,

        @Valid
        @RequestBody
        CreateOrderRequest request
    ) {

        OrderResponse response =
            orderService.createOrder(
                request,
                idempotencyKey,
                correlationId
            );

        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(response);
    }

    @GetMapping("/{orderId}")
    public ResponseEntity<OrderResponse>
    getOrder(
        @PathVariable Long orderId
    ) {

        return ResponseEntity.ok(
            orderService.getOrder(orderId)
        );
    }
}
```

---

# 20. Order Creation Flow

```text
POST /orders
       |
       ↓
Authenticate customer
       |
       ↓
Validate request
       |
       ↓
Check Idempotency-Key
       |
       ↓
Validate cart
       |
       ↓
Validate product
       |
       ↓
Get current pricing
       |
       ↓
Validate promotion
       |
       ↓
Calculate tax
       |
       ↓
Create Order
       |
       ↓
Create Outbox Event
       |
       ↓
COMMIT
       |
       ↓
Kafka
       |
       ↓
Saga
```

---

# 21. Important Design Decision

Do not make one giant synchronous request:

```text
Order
 ↓
Customer
 ↓
Product
 ↓
Pricing
 ↓
Promotion
 ↓
Inventory
 ↓
Payment
 ↓
Tax
 ↓
Shipping
 ↓
Notification
```

This creates:

- Long latency
- Tight coupling
- Cascading failures
- Retry storms
- Difficult recovery

Instead use:

```text
Synchronous:
Validation / quote / critical request-response

Asynchronous:
Long-running business workflow
```

---

# 22. Saga Architecture

For the critical order workflow:

```text
Order
  |
  ↓
Inventory Reservation
  |
  ↓
Fraud Check
  |
  ↓
Payment Authorization
  |
  ↓
Order Confirmation
  |
  ↓
Shipping
```

If payment fails:

```text
Payment Failed
      |
      ↓
Release Inventory
      |
      ↓
Cancel Order
```

---

# 23. Saga State Machine

Example:

```text
CREATED
   |
   ↓
VALIDATED
   |
   ↓
INVENTORY_PENDING
   |
   ↓
INVENTORY_RESERVED
   |
   ↓
FRAUD_PENDING
   |
   ↓
FRAUD_APPROVED
   |
   ↓
PAYMENT_PENDING
   |
   ↓
PAYMENT_AUTHORIZED
   |
   ↓
CONFIRMED
```

Failure:

```text
PAYMENT_FAILED
      |
      ↓
RELEASE_INVENTORY
      |
      ↓
CANCEL_ORDER
```

---

# 24. Choreography vs Orchestration

## Choreography

Services react to events.

```text
OrderCreated
   ↓
Inventory
   ↓
InventoryReserved
   ↓
Payment
   ↓
PaymentAuthorized
```

Advantages:

- Loose coupling
- Simple individual services

Disadvantages:

- Workflow becomes difficult to understand
- Harder to manage complex compensation

## Orchestration

A central order workflow controls the process.

```text
Order Saga
   |
   ├── Reserve Inventory
   ├── Fraud Check
   ├── Authorize Payment
   └── Confirm Order
```

For this project:

```text
Critical order transaction
=
Orchestration

Independent side effects
=
Choreography
```

---

# 25. Idempotency

Order creation must be idempotent.

Example:

```text
Client
  |
  | POST /orders
  | Idempotency-Key: abc123
  |
  ↓
Order Service
```

Network timeout occurs.

Client retries:

```text
Idempotency-Key: abc123
```

Order service must return the existing result.

It must NOT create:

```text
ORD-1001
ORD-1002
```

for the same logical request.

---

# 26. Idempotency Table

```sql
CREATE TABLE idempotency_records (

    id BIGSERIAL PRIMARY KEY,

    idempotency_key VARCHAR(200) NOT NULL,

    customer_id VARCHAR(100) NOT NULL,

    request_hash VARCHAR(128) NOT NULL,

    order_id BIGINT,

    response_payload TEXT,

    status VARCHAR(30) NOT NULL,

    created_at TIMESTAMP NOT NULL,

    expires_at TIMESTAMP NOT NULL,

    CONSTRAINT uq_idempotency_key_customer
        UNIQUE (
            idempotency_key,
            customer_id
        )
);
```

The request hash prevents the same key from being reused with a materially
different request.

---

# 27. Database Migration

## `V1__create_orders.sql`

```sql
CREATE TABLE orders (

    id BIGSERIAL PRIMARY KEY,

    order_number VARCHAR(100) NOT NULL UNIQUE,

    customer_id VARCHAR(100) NOT NULL,

    status VARCHAR(50) NOT NULL,

    subtotal NUMERIC(19,4) NOT NULL,

    discount NUMERIC(19,4) NOT NULL,

    tax NUMERIC(19,4) NOT NULL,

    total NUMERIC(19,4) NOT NULL,

    currency VARCHAR(3) NOT NULL,

    created_at TIMESTAMP NOT NULL,

    updated_at TIMESTAMP NOT NULL,

    version BIGINT NOT NULL DEFAULT 0
);

CREATE INDEX idx_orders_customer_id
    ON orders(customer_id);

CREATE INDEX idx_orders_status
    ON orders(status);
```

---

# 28. Order Items Migration

## `V2__create_order_items.sql`

```sql
CREATE TABLE order_items (

    id BIGSERIAL PRIMARY KEY,

    order_id BIGINT NOT NULL,

    product_id VARCHAR(100) NOT NULL,

    quantity INTEGER NOT NULL,

    unit_price NUMERIC(19,4) NOT NULL,

    line_total NUMERIC(19,4) NOT NULL,

    currency VARCHAR(3) NOT NULL,

    CONSTRAINT fk_order_items_order
        FOREIGN KEY(order_id)
        REFERENCES orders(id)
);
```

---

# 29. Outbox Migration

## `V3__create_outbox.sql`

```sql
CREATE TABLE outbox_events (

    id BIGSERIAL PRIMARY KEY,

    event_id VARCHAR(100) NOT NULL UNIQUE,

    event_type VARCHAR(200) NOT NULL,

    aggregate_id VARCHAR(100) NOT NULL,

    payload TEXT NOT NULL,

    status VARCHAR(30) NOT NULL,

    retry_count INTEGER NOT NULL DEFAULT 0,

    created_at TIMESTAMP NOT NULL,

    published_at TIMESTAMP
);

CREATE INDEX idx_outbox_status_created
    ON outbox_events(status, created_at);
```

---

# 30. Order Event

Example:

```json
{
  "eventId": "evt-10001",
  "eventType": "OrderCreated",
  "version": 1,
  "occurredAt": "2026-10-05T12:00:00Z",
  "source": "order-service",
  "correlationId": "corr-1001",
  "aggregateId": "100001",
  "payload": {
    "orderId": "100001",
    "customerId": "C1001",
    "total": "21237.64",
    "currency": "INR"
  }
}
```

---

# 31. Event ID vs Correlation ID

`eventId`:

```text
Uniquely identifies an event.
```

`correlationId`:

```text
Connects all operations belonging to the same business request.
```

Example:

```text
Correlation ID = corr-1001

OrderCreated
InventoryReserved
PaymentAuthorized
OrderConfirmed
```

Each event has a different:

```text
eventId
```

but may share:

```text
correlationId
```

---

# 32. Kafka Topics

Example:

```text
ecommerce.order.events
ecommerce.inventory.events
ecommerce.payment.events
ecommerce.shipping.events
```

Events:

```text
OrderCreated
OrderCancelled
InventoryReserved
InventoryReleased
PaymentAuthorized
PaymentFailed
OrderConfirmed
```

Topic naming and ownership should be governed centrally.

---

# 33. Kafka Producer

```java
@Service
public class EventPublisher {

    private final KafkaTemplate<String, Object>
        kafkaTemplate;

    public EventPublisher(
        KafkaTemplate<String, Object>
            kafkaTemplate
    ) {
        this.kafkaTemplate = kafkaTemplate;
    }

    public void publish(
        String topic,
        String key,
        Object event
    ) {

        kafkaTemplate.send(
            topic,
            key,
            event
        );
    }
}
```

In production, publishing directly from the order transaction should be
replaced by the Outbox Publisher pattern.

---

# 34. Outbox Publisher

```text
PostgreSQL
    |
    ↓
outbox_events
    |
    ↓
Outbox Publisher
    |
    ↓
Kafka
```

The publisher:

1. Reads unpublished events
2. Publishes them
3. Waits for broker acknowledgement
4. Marks event published
5. Retries failures

Duplicate publishing can still happen around crashes.

Therefore consumers must be idempotent.

---

# 35. Exactly Once Reality

Do not assume:

```text
Kafka
=
business exactly once
```

Real production systems generally design for:

```text
At-least-once delivery
+
Idempotent consumers
```

Example:

```text
InventoryReserved
```

received twice.

Inventory service must not reserve the same inventory twice.

---

# 36. Order Service Redis Usage

Redis is useful for:

- Idempotency lookup
- Short-lived order workflow state
- Request deduplication
- Read caching
- Rate limiting
- Temporary Saga state

PostgreSQL remains authoritative for:

- Order
- Order items
- Status
- Financial values
- Durable audit

Redis is not the source of truth for the order.

---

# 37. Dockerfile

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

COPY --from=builder \
    /build/target/*.jar \
    app.jar

RUN useradd \
    --uid 10001 \
    --create-home \
    appuser

USER 10001

EXPOSE 8080

ENTRYPOINT [
    "java",
    "-jar",
    "app.jar"
]
```

---

# 38. `.dockerignore`

```text
.git
.github

.idea
.vscode

target

*.log

.env
.env.*

README.md

docker-compose.yaml
```

---

# 39. Local Docker Compose

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


  redis:

    image: redis:7

    container_name: order-redis

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: order-kafka

    ports:

      - "9092:9092"

    # Local-development configuration only.
    # Pin a tested Kafka image version in the actual repository.

  order-service:

    build:

      context: .

      dockerfile: Dockerfile

    depends_on:

      - postgres

      - redis

      - kafka

    environment:

      SPRING_PROFILES_ACTIVE: local

      DB_URL: jdbc:postgresql://postgres:5432/orders

      DB_USERNAME: postgres

      DB_PASSWORD: postgres

      REDIS_HOST: redis

      REDIS_PORT: 6379

      KAFKA_BOOTSTRAP_SERVERS: kafka:9092

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

For production, do not use the development Kafka container.

Use the project's production Kafka platform, such as Amazon MSK or the
approved Kafka platform.

---

# 40. Kubernetes Deployment

## `deployment.yaml`

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: order-service

  labels:

    app: order-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: order-service

  template:

    metadata:

      labels:

        app: order-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: order-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: IfNotPresent

          ports:

            - name: http

              containerPort: 8080

          env:

            - name: SPRING_PROFILES_ACTIVE

              value: prod

          envFrom:

            - configMapRef:

                name: order-service-config

            - secretRef:

                name: order-service-secrets

          resources:

            {{- toYaml .Values.resources | nindent 12 }}

          startupProbe:

            httpGet:

              path: /actuator/health

              port: http

            failureThreshold: 30

            periodSeconds: 10

          readinessProbe:

            httpGet:

              path: /actuator/health/readiness

              port: http

            initialDelaySeconds: 10

            periodSeconds: 10

          livenessProbe:

            httpGet:

              path: /actuator/health/liveness

              port: http

            initialDelaySeconds: 30

            periodSeconds: 20

          securityContext:

            allowPrivilegeEscalation: false

            readOnlyRootFilesystem: true

            capabilities:

              drop:

                - ALL
```

---

# 41. Service

```yaml
apiVersion: v1

kind: Service

metadata:

  name: order-service

  labels:

    app: order-service

spec:

  type: ClusterIP

  selector:

    app: order-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 42. ConfigMap

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: order-service-config

data:

  REDIS_HOST: redis.ecommerce.svc.cluster.local

  REDIS_PORT: "6379"

  KAFKA_BOOTSTRAP_SERVERS: kafka.ecommerce.svc.cluster.local:9092

  ORDER_IDEMPOTENCY_TTL_SECONDS: "86400"
```

For production, replace these with the actual managed/platform endpoints.

---

# 43. External Secrets

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: order-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: order-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DB_URL

      remoteRef:

        key: ecommerce/prod/order-service

        property: DB_URL

    - secretKey: DB_USERNAME

      remoteRef:

        key: ecommerce/prod/order-service

        property: DB_USERNAME

    - secretKey: DB_PASSWORD

      remoteRef:

        key: ecommerce/prod/order-service

        property: DB_PASSWORD
```

---

# 44. ServiceAccount

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: order-service

  labels:

    app: order-service
```

If Order Service publishes directly to AWS SQS:

```text
Order Pod
   ↓
EKS Pod Identity
   ↓
order-service IAM role
   ↓
SQS
```

Only the required queue ARN should be permitted.

---

# 45. NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: order-service

spec:

  podSelector:

    matchLabels:

      app: order-service

  policyTypes:

    - Ingress
    - Egress

  ingress:

    - from:

        - namespaceSelector:

            matchLabels:

              kubernetes.io/metadata.name: ecommerce

      ports:

        - protocol: TCP

          port: 8080

  egress:

    - ports:

        - protocol: TCP

          port: 5432

        - protocol: TCP

          port: 6379

        - protocol: TCP

          port: 9092

        - protocol: TCP

          port: 8080

    - to:

        - namespaceSelector:

            matchLabels:

              kubernetes.io/metadata.name: kube-system

      ports:

        - protocol: UDP

          port: 53

        - protocol: TCP

          port: 53
```

The final NetworkPolicy must match the real Kafka, Redis, database, and
service-to-service topology.

---

# 46. HPA

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: order-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: order-service

  minReplicas: 3

  maxReplicas: 20

  behavior:

    scaleUp:

      stabilizationWindowSeconds: 0

    scaleDown:

      stabilizationWindowSeconds: 300

  metrics:

    - type: Resource

      resource:

        name: cpu

        target:

          type: Utilization

          averageUtilization: 60
```

Order scaling must be constrained by:

```text
RDS capacity
+
Kafka partition capacity
+
Payment provider rate limits
+
Inventory capacity
```

Scaling from 3 to 100 Pods without dependency capacity can make an incident
worse.

---

# 47. PDB

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: order-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: order-service
```

---

# 48. ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: order-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: order-service

  endpoints:

    - port: http

      path: /actuator/prometheus

      interval: 15s
```

---

# 49. Order Metrics

Application metrics:

```text
orders_created_total

orders_confirmed_total

orders_failed_total

orders_cancelled_total

order_creation_duration_seconds

order_saga_duration_seconds

order_saga_failures_total

order_saga_compensation_total

order_idempotency_hits_total

order_outbox_pending_total

order_outbox_publish_failures_total
```

Business metrics:

```text
orders_per_minute

successful_orders

failed_orders

payment_failure_rate

inventory_failure_rate

order_conversion_rate
```

---

# 50. Grafana Dashboard

```text
Order Service Dashboard

├── Request Rate
├── P50
├── P95
├── P99
├── HTTP 5xx
│
├── Orders
│   ├── Created
│   ├── Confirmed
│   ├── Failed
│   └── Cancelled
│
├── Saga
│   ├── Active
│   ├── Duration
│   ├── Failures
│   └── Compensation
│
├── PostgreSQL
│   ├── Connections
│   ├── Query latency
│   └── DB load
│
├── Kafka
│   ├── Producer errors
│   ├── Consumer lag
│   └── Throughput
│
└── Kubernetes
    ├── Pods
    ├── CPU
    ├── Memory
    └── Restarts
```

---

# 51. Logging

Example:

```json
{
  "@timestamp": "2026-10-05T12:00:00Z",
  "level": "INFO",
  "service": "order-service",
  "environment": "prod",
  "namespace": "ecommerce",
  "pod": "order-service-7c8d9",
  "version": "1.0.0",
  "correlationId": "corr-1001",
  "eventId": "evt-10001",
  "orderId": "100001",
  "operation": "CREATE_ORDER",
  "status": "PENDING_VALIDATION",
  "message": "Order created"
}
```

Never log:

```text
Authorization headers
Payment credentials
Card data
Passwords
Secrets
AWS credentials
```

---

# 52. Distributed Tracing

Example trace:

```text
POST /orders
 |
 ├── customer-service
 |
 ├── product-service
 |
 ├── pricing-service
 |
 ├── promotion-service
 |
 └── tax-service
```

Then asynchronous processing:

```text
OrderCreated
      |
      ├── inventory-service
      |
      ├── fraud-service
      |
      └── payment-service
```

Trace context and correlation ID should be propagated through HTTP and
messaging.

---

# 53. Kafka Partition Key

For order events:

```text
key = orderId
```

Example:

```text
ORD-1001
```

This helps preserve ordering for events for the same order within a Kafka
partition.

Example:

```text
OrderCreated
InventoryReserved
PaymentAuthorized
OrderConfirmed
```

should not randomly reorder for the same order.

---

# 54. Order Event Ordering

Kafka ordering is partition-based, not global.

Therefore:

```text
Same orderId
→ same partition key
→ ordered within that partition
```

But:

```text
Order A
Order B
```

do not need global ordering.

This allows horizontal scaling.

---

# 55. Consumer Idempotency

Suppose:

```text
PaymentAuthorized
```

is delivered twice.

Payment/order processing must recognize:

```text
eventId
```

already processed.

Example table:

```sql
CREATE TABLE processed_events (

    event_id VARCHAR(200) PRIMARY KEY,

    consumer_name VARCHAR(200) NOT NULL,

    processed_at TIMESTAMP NOT NULL
);
```

Consumer transaction:

```text
BEGIN
  insert processed_event
  process business operation
  update order
COMMIT
```

If duplicate:

```text
unique constraint
```

prevents duplicate processing.

---

# 56. Payment Safety

Order Service must never assume:

```text
payment request
=
payment completed
```

Payment must support idempotency.

Example:

```text
orderId = ORD-1001
paymentAttemptId = PAY-1001
```

A retry should not charge the customer twice.

---

# 57. Inventory Safety

Similarly:

```text
Reserve inventory
```

must be idempotent.

Example:

```text
reservationId = RES-1001
orderId = ORD-1001
```

Repeated requests with the same reservation ID must not reserve stock twice.

---

# 58. Compensation

If:

```text
Inventory reserved
```

but:

```text
Payment failed
```

Order Saga sends:

```text
ReleaseInventory
```

Then:

```text
Inventory
 ↓
release reservation
 ↓
InventoryReleased
```

Order becomes:

```text
CANCELLED
```

This is a business compensation, not a database rollback.

---

# 59. Why Saga Is Necessary

A traditional database transaction:

```text
BEGIN

Order insert
Inventory update
Payment update
Shipping update

COMMIT
```

cannot safely span:

```text
PostgreSQL
+
Redis
+
Kafka
+
Inventory DB
+
Payment provider
+
Shipping provider
```

Therefore:

```text
Distributed transaction
       ↓
Saga
       ↓
Compensating actions
```

---

# 60. Transactional Boundaries

Order Service transaction:

```text
BEGIN
  create order
  create order items
  create status history
  create outbox event
COMMIT
```

Outside transaction:

```text
Kafka
Inventory
Payment
Shipping
Notification
```

This is an important microservice boundary.

---

# 61. Docker Image

Image:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/order-service:<git-sha>
```

Prefer immutable identifiers.

Better deployment reference:

```text
image digest
```

instead of:

```text
latest
```

---

# 62. CI Pipeline

```yaml
name: Order Service CI

on:

  pull_request:

    branches:
      - main

  push:

    branches:
      - main


permissions:

  contents: read

  id-token: write


env:

  AWS_REGION: ap-south-1

  ECR_REPOSITORY: order-service


jobs:

  test:

    runs-on: ubuntu-latest

    steps:

      - name: Checkout

        uses: actions/checkout@v4


      - name: Setup Java

        uses: actions/setup-java@v4

        with:

          distribution: temurin

          java-version: "21"

          cache: maven


      - name: Maven test

        run: |

          chmod +x mvnw

          ./mvnw clean verify


  security:

    runs-on: ubuntu-latest

    steps:

      - name: Checkout

        uses: actions/checkout@v4


      - name: Trivy filesystem scan

        uses: aquasecurity/trivy-action@master

        with:

          scan-type: fs

          scan-ref: .

          severity: HIGH,CRITICAL

          ignore-unfixed: true


  docker:

    needs:
      - test
      - security

    if: github.event_name == 'push'

    runs-on: ubuntu-latest

    steps:

      - name: Checkout

        uses: actions/checkout@v4


      - name: Configure AWS credentials

        uses: aws-actions/configure-aws-credentials@v4

        with:

          role-to-assume: ${{ secrets.AWS_CI_ROLE_ARN }}

          aws-region: ${{ env.AWS_REGION }}


      - name: Login to ECR

        id: ecr

        uses: aws-actions/amazon-ecr-login@v2


      - name: Build image

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |

          docker build \
            -t "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}" \
            .


      - name: Scan image

        uses: aquasecurity/trivy-action@master

        with:

          image-ref: ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

          severity: HIGH,CRITICAL

          exit-code: "1"

          ignore-unfixed: true


      - name: Push image

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |

          docker push \
            "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}"
```

Versions of GitHub Actions and external actions should be pinned and reviewed
according to the organization's supply-chain policy rather than blindly
following floating major tags.

---

# 63. SonarQube

Order Service requires stronger code-quality checks because it contains
critical business logic.

Pipeline:

```text
Maven
 ↓
Unit tests
 ↓
Integration tests
 ↓
Coverage
 ↓
SonarQube
 ↓
Dependency scan
 ↓
Trivy
 ↓
Docker
 ↓
Trivy image
 ↓
ECR
```

SonarQube checks:

- Bugs
- Vulnerabilities
- Code smells
- Duplications
- Coverage
- Maintainability

---

# 64. GitOps Flow

```text
Developer
   |
   ↓
GitHub PR
   |
   ↓
CI
   |
   ↓
ECR
   |
   ↓
GitOps PR
   |
   ↓
Review
   |
   ↓
Merge
   |
   ↓
ArgoCD
   |
   ↓
EKS
```

CI does not directly run:

```text
kubectl apply
```

against production.

---

# 65. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: order-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/order-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

For high-risk production changes, automatic sync may be replaced by an
approval-controlled promotion workflow.

---

# 66. Order SLO

Initial targets:

### Availability

```text
99.95%
```

### Create Order P95

```text
< 500ms
```

for the synchronous creation/validation portion.

The complete Saga may take longer because it includes asynchronous processing.

### Order creation error rate

```text
< 0.5%
```

### Event processing

```text
99.9% within defined business processing window
```

Actual values must be established from production requirements and load
testing.

---

# 67. Alert Rules

Important alerts:

```text
High Order 5xx

High Order Creation Latency

High Order Failure Rate

High Saga Failure Rate

High Saga Compensation Rate

Outbox Backlog

Outbox Publish Failure

Kafka Consumer Lag

RDS Connection Exhaustion

Hikari Pending Connections

Payment Failure Rate

Inventory Reservation Failure Rate
```

Business alerts can be more useful than infrastructure-only alerts.

Example:

```text
CPU = normal
Memory = normal
Pods = normal

BUT

Order success rate = 70%
```

This is still a production incident.

---

# 68. Failure Scenario — Payment Failure

```text
Order Created
      |
      ↓
Inventory Reserved
      |
      ↓
Payment
      |
      X
Payment Failed
      |
      ↓
Release Inventory
      |
      ↓
Cancel Order
```

No database rollback can undo a real-world inventory reservation or payment
attempt.

Compensation is required.

---

# 69. Failure Scenario — Kafka Failure

If Kafka is unavailable:

```text
Order DB transaction
       |
       ↓
Outbox event
       |
       ↓
COMMIT
```

The order can remain durable.

The outbox publisher retries later.

This is one of the major benefits of the Outbox pattern.

Without Outbox:

```text
DB SUCCESS
Kafka FAILURE
```

could permanently lose the event.

---

# 70. Failure Scenario — Outbox Backlog

Suppose:

```text
Outbox created = 10,000/min
Kafka publish = 5,000/min
```

Backlog grows:

```text
0
 ↓
5,000
 ↓
10,000
 ↓
50,000
```

Monitor:

```text
outbox_pending_total
outbox_oldest_age
publish_failure_rate
```

Potential mitigation:

- Increase publisher throughput
- Investigate Kafka
- Fix serialization errors
- Fix authentication
- Fix network
- Increase partitions if appropriate
- Prevent duplicate publishers
- Protect database

---

# 71. Failure Scenario — Database Failure

Order creation:

```text
Request
 ↓
PostgreSQL
 X
```

Response:

```text
503 Service Unavailable
```

Do not create an order only in Redis.

The order must not be reported as successfully created unless the durable
transaction has succeeded.

---

# 72. Failure Scenario — Duplicate Order Request

Client:

```text
POST /orders
Idempotency-Key: ABC123
```

Request times out.

Client retries.

Order Service:

```text
ABC123 already exists
```

returns the previous response.

Result:

```text
One order
```

instead of:

```text
Two orders
```

---

# 73. Failure Scenario — Duplicate Kafka Event

Kafka delivers:

```text
PaymentAuthorized
```

twice.

Consumer checks:

```text
processed_events
```

First:

```text
INSERT
SUCCESS
```

Second:

```text
UNIQUE CONSTRAINT
```

The business action is not repeated.

---

# 74. Failure Scenario — Bad Deployment

Suppose version:

```text
1.9.0
```

contains an order calculation bug.

Symptoms:

```text
HTTP 500
Order failures
```

Investigation:

```text
Git commit
 ↓
CI
 ↓
Image digest
 ↓
GitOps commit
 ↓
ArgoCD
 ↓
Pods
```

Rollback:

```text
GitOps
 ↓
known-good image
 ↓
ArgoCD
 ↓
EKS
```

But:

> Never automatically roll back the database schema together with the
> application.

Database compatibility must be checked first.

---

# 75. Production Incident RCA

## Incident

```text
INC-2026-ORDER-001
```

## Severity

```text
SEV-1
```

## Title

```text
Order Creation Failure Due to Database Connection Exhaustion
```

## Date

```text
2026-10-05
```

---

# 76. Customer Impact

Customers experienced:

- Checkout failures
- HTTP 500/503
- Increased order latency
- Some requests timed out

No duplicate successful orders were confirmed.

---

# 77. Timeline

```text
14:00
Order service healthy.

14:05
Traffic increases.

14:07
HPA increases replicas from 6 to 18.

14:08
RDS connections increase rapidly.

14:09
Hikari pending connections increase.

14:10
Order P95 increases.

14:11
HTTP 503 increases.

14:12
SEV-1 declared.

14:14
Platform team identifies connection amplification.

14:16
Order HPA maximum reduced.

14:18
Noncritical database workloads reduced.

14:20
Order replicas stabilize.

14:22
RDS connection count falls.

14:24
Order latency recovers.

14:27
5xx returns to baseline.

14:30
Incident resolved.
```

---

# 78. Root Cause

The order service had:

```text
18 Pods
×
20 DB connections
=
360 potential connections
```

Additional connections came from:

- Other microservices
- Admin connections
- Background jobs
- Deployment overlap

The RDS instance could not safely support the resulting connection load.

---

# 79. Amplification

Primary problem:

```text
Database connection capacity
```

Amplification:

```text
HPA scaling
+
large Hikari pool
+
retry traffic
```

The autoscaler responded to increasing latency, but additional Pods created
additional database connection capacity demand.

This made the original problem worse.

---

# 80. Five Whys

### Why did orders fail?

Database connections were unavailable.

### Why?

RDS reached its safe connection capacity.

### Why?

Order replicas increased and each Pod had a large Hikari pool.

### Why did replicas increase?

HPA reacted to increased latency.

### Why did this amplify the incident?

HPA was not constrained using the database connection budget.

---

# 81. Corrective Actions

| Action | Owner |
|---|---|
| Reduce Hikari pool size | Backend |
| Define DB connection budget | Platform + DBA |
| Cap HPA max replicas | Platform |
| Add Hikari metrics | SRE |
| Add RDS connection alerts | DBA/SRE |
| Test autoscaling against RDS | Performance |
| Review retry policy | Backend |
| Review deployment surge | Platform |
| Evaluate RDS Proxy | Platform + DBA |
| Add connection-capacity runbook | SRE |

---

# 82. DevOps Responsibility

DevOps/Platform:

- EKS
- HPA
- Deployment strategy
- Resource limits
- IAM
- Network
- Secrets integration
- ECR
- GitOps
- ArgoCD
- Observability
- CI/CD
- Incident coordination

Backend team:

- Order logic
- Idempotency
- Saga
- Outbox
- Kafka consumers
- Retry logic
- Database transactions
- Connection pool configuration

DBA/data team:

- PostgreSQL health
- `max_connections`
- Query performance
- Locks
- Long transactions
- Indexes
- Database parameters
- RDS failover
- Database capacity

SRE:

- SLOs
- Alerts
- Incident process
- Runbooks
- Capacity
- Reliability testing

---

# 83. Production Security

Order Service is highly sensitive.

Required controls:

```text
Least-privilege IAM
NetworkPolicy
Private RDS
Private Kafka
Private Redis
Secrets Manager
External Secrets
EKS Pod Identity
TLS
Encryption at rest
Audit logging
CloudTrail
ECR scanning
Trivy
SonarQube
Dependency scanning
No credentials in Git
```

Never put:

```text
AWS_ACCESS_KEY_ID
AWS_SECRET_ACCESS_KEY
DB_PASSWORD
payment credentials
```

inside GitHub repository files.

---

# 84. Production Deployment Strategy

For normal low-risk changes:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
PRODUCTION
```

For high-risk order changes:

```text
Production
   |
   ↓
Canary 5%
   |
   ↓
Observe
   |
   ↓
25%
   |
   ↓
50%
   |
   ↓
100%
```

Observe:

```text
Order success rate
HTTP 5xx
P95/P99
Saga failures
Payment failures
Inventory failures
DB connections
Kafka lag
```

---

# 85. Canary Rollback

Example:

```text
Version 1.8.0
        ↓
Version 1.9.0
        ↓
5% traffic
        ↓
Order failures increase
        ↓
STOP
        ↓
Rollback
```

The rollback decision should be automated or operator-approved according to
the production deployment policy.

---

# 86. Testing Strategy

## Unit Tests

Test:

- Order calculation
- State transitions
- Idempotency
- Validation
- Compensation logic

## Integration Tests

Test:

- PostgreSQL
- Redis
- Kafka
- Outbox

## Contract Tests

Test:

```text
Order ↔ Inventory
Order ↔ Payment
Order ↔ Pricing
Order ↔ Promotion
Order ↔ Tax
```

## Performance Tests

Test:

```text
100 RPS
500 RPS
1,000 RPS
2,000 RPS
```

These are simulation points, not guaranteed capacity.

## Failure Tests

Test:

- Redis failure
- PostgreSQL failure
- Kafka failure
- Payment timeout
- Inventory failure
- Duplicate event
- Duplicate request
- Pod restart
- Node failure
- AZ failure
- Bad deployment

---

# 87. Order Service Load Test

Example k6 scenario:

```javascript
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  vus: 100,
  duration: '5m'
};

export default function () {

  const payload = JSON.stringify({
    customerId: 'C1001',
    items: [
      {
        productId: 'P1001',
        quantity: 1
      }
    ]
  });

  const response = http.post(
    'https://staging.example.com/api/v1/orders',
    payload,
    {
      headers: {
        'Content-Type': 'application/json',
        'Idempotency-Key':
          `load-${__VU}-${__ITER}`
      }
    }
  );

  check(response, {
    'status is successful':
      (r) => r.status === 201
  });
}
```

Run performance tests against a dedicated performance/staging environment,
not uncontrolled production traffic.

---

# 88. Order Service Capacity Model

For synchronous API capacity:

```text
required pods
=
ceil(
    peak RPS
    /
    tested RPS per pod
)
```

But the final capacity is constrained by:

```text
Order Pods
   ↓
Hikari connections
   ↓
RDS
```

and:

```text
Order Pods
   ↓
Kafka
   ↓
Inventory/Payment consumers
```

Therefore order capacity is a system property, not just a Pod property.

---

# 89. Operational Runbook

When order failure alert fires:

```text
1. Confirm customer impact

2. Check order RPS

3. Check order 5xx

4. Check P95/P99

5. Check recent deployment

6. Check Pod health

7. Check HPA

8. Check Hikari connections

9. Check RDS

10. Check Kafka

11. Check Redis

12. Check payment failures

13. Check inventory failures

14. Inspect logs

15. Inspect traces

16. Determine primary failure

17. Stop amplification

18. Mitigate

19. Verify business transaction

20. Document RCA
```

---

# 90. Production Architecture Summary

```text
                           Internet
                              |
                              ↓
                        CloudFront/WAF
                              |
                              ↓
                             ALB
                              |
                              ↓
                        Order Service
                       /      |       \
                      /       |        \
                     ↓        ↓         ↓
                 PostgreSQL Redis      Kafka
                     |                  |
                     ↓                  ↓
                  Outbox          Event-driven Saga
                                        |
                 ┌──────────────────────┼───────────────────┐
                 ↓                      ↓                   ↓
             Inventory               Payment             Fraud
                 |                      |                   |
                 └──────────────────────┼───────────────────┘
                                        |
                                        ↓
                                   Order Confirmed
                                        |
                           ┌────────────┼────────────┐
                           ↓            ↓            ↓
                       Shipping    Notification   Reporting
```

---

# 91. Final Architecture Principles

The Order Service follows these principles:

```text
1. PostgreSQL is the durable order source of truth.

2. Redis is an optimization/temporary state layer.

3. Kafka handles durable asynchronous events.

4. Transactional Outbox prevents DB/event inconsistency.

5. Idempotency protects against duplicate requests.

6. Consumers are idempotent because delivery can repeat.

7. Saga manages distributed business transactions.

8. Compensation replaces distributed database rollback.

9. HPA must respect downstream capacity.

10. Database connection pools must be sized per Pod.

11. Payment operations require idempotency.

12. Inventory reservations require idempotency.

13. GitOps controls production deployment.

14. ArgoCD reconciles desired state.

15. Prometheus/Grafana monitor the service.

16. ELK provides centralized logs.

17. OpenTelemetry + Jaeger provide distributed tracing.

18. SLOs measure customer/business behavior.

19. Canary deployment reduces production risk.

20. Every serious incident produces an RCA and preventive actions.
```

# End of 170 — Order Service
```

### Phase 19 progress

```text
161 Auth Service            ✅
162 User Service            ✅
163 Customer Service        ✅
164 Catalog Service         ✅
165 Product Service         ✅
166 Inventory Service       ✅
167 Pricing Service         ✅
168 Promotion Service       ✅
169 Cart Service            ✅
170 Order Service           ✅

171 Payment Service         ⏳
172 Fraud Service           ⏳
173 Shipping Service        ⏳
174 Delivery Service        ⏳
175 Notification Service    ⏳
176 Search Service          ⏳
177 Recommendation Service  ⏳
178 Review Service          ⏳
179 Tax Service             ⏳
180 Invoice Service         ⏳
181 Reporting Service       ⏳
```

**Next: `171-Payment-Service.md` — payment authorization/capture, external payment gateway integration, idempotency, webhook handling, payment state machine, PCI/security boundaries, PostgreSQL, Redis, Kafka, retries, reconciliation, failure handling, Kubernetes, CI/CD, GitOps, observability and production RCA.**