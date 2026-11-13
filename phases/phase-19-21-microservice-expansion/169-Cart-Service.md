# 169 — Cart Service

**File:** `169-Cart-Service.md`

```markdown
# 169 — Cart Service

## 1. Overview

The `cart-service` manages the customer's shopping cart.

It is one of the highest-read and frequently updated services in the
e-commerce platform.

The cart must provide:

- Add product to cart
- Update quantity
- Remove product
- View cart
- Clear cart
- Cart expiration
- Price snapshot/reference
- Product validation
- Inventory validation
- Promotion integration
- Cart totals
- Customer-specific cart
- Guest cart
- Cart merge after login
- Concurrency protection
- Idempotent cart operations

Technology:

- Java 21
- Spring Boot
- Maven
- Spring Data Redis
- PostgreSQL
- Docker
- Kubernetes
- Helm
- AWS EKS
- ECR
- GitHub Actions
- ArgoCD
- Prometheus
- Grafana
- ELK
- OpenTelemetry
- Jaeger

The cart architecture is intentionally different from the pricing service.

Pricing is:

```text
Redis → PostgreSQL
```

Cart is primarily:

```text
Redis → PostgreSQL persistence
```

because cart access is extremely frequent and latency-sensitive.

---

# 2. Business Responsibility

Cart Service owns:

- Customer cart
- Cart items
- Product quantities
- Cart expiration
- Cart totals
- Cart merge
- Cart state
- Cart item removal
- Cart clearing

It does NOT own:

- Product master data
- Inventory quantity
- Base product price
- Promotion rules
- Payment
- Order creation

Those belong to other services.

Architecture:

```text
cart-service
    |
    ├── Redis
    │
    ├── PostgreSQL
    │
    ├── product-service
    │
    ├── inventory-service
    │
    ├── pricing-service
    │
    └── promotion-service
```

---

# 3. Why Cart Uses Redis Heavily

Consider a customer browsing an e-commerce application.

A single user can perform:

```text
GET cart
ADD product
GET cart
UPDATE quantity
GET cart
REMOVE product
GET cart
```

A large number of requests can therefore hit the cart service.

If every request hits PostgreSQL:

```text
10,000 cart requests
        |
        ↓
10,000 PostgreSQL operations
```

Redis can provide:

```text
10,000 cart requests
        |
        ↓
Redis
        |
        ↓
Most requests served quickly
```

PostgreSQL remains the durable persistence layer.

---

# 4. High-Level Architecture

```text
                         Customer
                            |
                            ↓
                     CloudFront / WAF
                            |
                            ↓
                           ALB
                            |
                            ↓
                      cart-service
                       /        \
                      /          \
                     ↓            ↓
                  Redis       PostgreSQL
                     |
                     |
          ┌──────────┼───────────┐
          ↓          ↓           ↓
       Product    Pricing     Promotion
       Service    Service      Service
          |
          ↓
      Inventory
```

---

# 5. Cart Request Flow

## Read Cart

```text
Customer
   |
   ↓
cart-service
   |
   ↓
Redis
   |
   ├── HIT → return cart
   |
   └── MISS
        |
        ↓
    PostgreSQL
        |
        ↓
    Redis SET
        |
        ↓
    return cart
```

---

# 6. Add Item Flow

```text
Customer
   |
   ↓
cart-service
   |
   ├── Validate product
   |
   ├── Check inventory
   |
   ├── Get current price
   |
   ├── Update Redis cart
   |
   └── Persist cart state
```

Important:

> Adding an item to a cart does not reserve inventory.

Inventory reservation belongs to the order/checkout workflow.

Otherwise abandoned carts could lock inventory indefinitely.

---

# 7. Cart vs Order

Cart:

```text
temporary customer intent
```

Order:

```text
confirmed business transaction
```

Example:

```text
Cart
Product A × 2
₹1,000 each
```

does NOT mean:

```text
Inventory reserved = 2
```

At checkout:

```text
Cart
  ↓
Order
  ↓
Inventory reservation
  ↓
Payment
  ↓
Order confirmation
```

---

# 8. Cart State

Example:

```text
ACTIVE
```

Possible future states:

```text
ACTIVE
CHECKOUT_STARTED
CONVERTED
ABANDONED
EXPIRED
```

The cart service can maintain the cart lifecycle.

---

# 9. Repository Structure

```text
cart-service/
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/company/cart/
│   │   │       ├── CartServiceApplication.java
│   │   │       │
│   │   │       ├── controller/
│   │   │       │   └── CartController.java
│   │   │       │
│   │   │       ├── service/
│   │   │       │   ├── CartService.java
│   │   │       │   └── CartPricingService.java
│   │   │       │
│   │   │       ├── repository/
│   │   │       │   └── CartRepository.java
│   │   │       │
│   │   │       ├── entity/
│   │   │       │   ├── Cart.java
│   │   │       │   └── CartItem.java
│   │   │       │
│   │   │       ├── dto/
│   │   │       │   ├── AddCartItemRequest.java
│   │   │       │   ├── UpdateCartItemRequest.java
│   │   │       │   └── CartResponse.java
│   │   │       │
│   │   │       ├── client/
│   │   │       │   ├── ProductClient.java
│   │   │       │   ├── PricingClient.java
│   │   │       │   └── InventoryClient.java
│   │   │       │
│   │   │       ├── exception/
│   │   │       │   ├── CartNotFoundException.java
│   │   │       │   └── GlobalExceptionHandler.java
│   │   │       │
│   │   │       └── config/
│   │   │           ├── RedisConfig.java
│   │   │           └── WebClientConfig.java
│   │   │
│   │   └── resources/
│   │       ├── application.yaml
│   │       └── application-prod.yaml
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
│   └── cart-service/
│       ├── Chart.yaml
│       ├── values.yaml
│       ├── values-dev.yaml
│       ├── values-qa.yaml
│       ├── values-staging.yaml
│       ├── values-prod.yaml
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           ├── configmap.yaml
│           ├── externalsecret.yaml
│           ├── serviceaccount.yaml
│           ├── networkpolicy.yaml
│           ├── hpa.yaml
│           ├── pdb.yaml
│           └── servicemonitor.yaml
│
└── .github/
    ├── CODEOWNERS
    └── workflows/
        ├── ci.yaml
        └── security.yaml
```

---

# 10. Maven Configuration

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
        cart-service
    </artifactId>

    <version>
        1.0.0
    </version>

    <name>
        cart-service
    </name>

    <properties>

        <java.version>
            21
        </java.version>

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
                org.projectlombok
            </groupId>

            <artifactId>
                lombok
            </artifactId>

            <optional>
                true
            </optional>
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

The actual project should commit `mvnw`, `mvnw.cmd`, and `.mvn/wrapper` so the
CI pipeline uses the project's pinned Maven Wrapper version.

---

# 11. Application Configuration

## `application.yaml`

```yaml
server:

  port: 8080


spring:

  application:

    name: cart-service


  datasource:

    url: ${DB_URL:jdbc:postgresql://localhost:5432/cart}

    username: ${DB_USERNAME:postgres}

    password: ${DB_PASSWORD:postgres}

    hikari:

      maximum-pool-size: 10

      minimum-idle: 2

      connection-timeout: 3000

      validation-timeout: 1000

      idle-timeout: 600000

      max-lifetime: 1500000

      pool-name: cart-service-hikari


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

    com.company.cart: INFO


cart:

  ttl-seconds: 2592000

  max-items: 100
```

---

# 12. Production Configuration

## `application-prod.yaml`

```yaml
server:

  port: 8080


spring:

  application:

    name: cart-service


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

      pool-name: cart-service-hikari


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


management:

  endpoints:

    web:

      exposure:

        include: health,info,prometheus

  endpoint:

    health:

      probes:

        enabled: true


cart:

  ttl-seconds: 2592000

  max-items: 100
```

`maximum-pool-size: 10` is an illustrative starting point.

It must be tuned using:

```text
Pod count
×
Pool size
×
RDS capacity
```

and load-testing results.

---

# 13. Main Application

## `CartServiceApplication.java`

```java
package com.company.cart;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class CartServiceApplication {

    public static void main(String[] args) {

        SpringApplication.run(
            CartServiceApplication.class,
            args
        );
    }
}
```

---

# 14. Cart Entity

## `Cart.java`

```java
package com.company.cart.entity;

import jakarta.persistence.*;

import java.time.Instant;

@Entity
@Table(name = "carts")
public class Cart {

    @Id
    @GeneratedValue(
        strategy = GenerationType.IDENTITY
    )
    private Long id;

    @Column(
        nullable = false,
        unique = true
    )
    private String customerId;

    @Column(
        nullable = false
    )
    private String status;

    @Column(
        nullable = false
    )
    private Instant createdAt;

    @Column(
        nullable = false
    )
    private Instant updatedAt;

    public Long getId() {
        return id;
    }

    public String getCustomerId() {
        return customerId;
    }

    public void setCustomerId(
        String customerId
    ) {
        this.customerId = customerId;
    }

    public String getStatus() {
        return status;
    }

    public void setStatus(
        String status
    ) {
        this.status = status;
    }

    public Instant getCreatedAt() {
        return createdAt;
    }

    public void setCreatedAt(
        Instant createdAt
    ) {
        this.createdAt = createdAt;
    }

    public Instant getUpdatedAt() {
        return updatedAt;
    }

    public void setUpdatedAt(
        Instant updatedAt
    ) {
        this.updatedAt = updatedAt;
    }
}
```

---

# 15. Cart Item Entity

## `CartItem.java`

```java
package com.company.cart.entity;

import jakarta.persistence.*;

import java.math.BigDecimal;

@Entity
@Table(
    name = "cart_items",
    uniqueConstraints = {
        @UniqueConstraint(
            columnNames = {
                "cart_id",
                "product_id"
            }
        )
    }
)
public class CartItem {

    @Id
    @GeneratedValue(
        strategy = GenerationType.IDENTITY
    )
    private Long id;

    @ManyToOne(
        fetch = FetchType.LAZY
    )
    @JoinColumn(
        name = "cart_id",
        nullable = false
    )
    private Cart cart;

    @Column(
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
    private BigDecimal priceSnapshot;

    @Column(
        length = 3,
        nullable = false
    )
    private String currency;

    public Long getId() {
        return id;
    }

    public Cart getCart() {
        return cart;
    }

    public void setCart(
        Cart cart
    ) {
        this.cart = cart;
    }

    public String getProductId() {
        return productId;
    }

    public void setProductId(
        String productId
    ) {
        this.productId = productId;
    }

    public Integer getQuantity() {
        return quantity;
    }

    public void setQuantity(
        Integer quantity
    ) {
        this.quantity = quantity;
    }

    public BigDecimal getPriceSnapshot() {
        return priceSnapshot;
    }

    public void setPriceSnapshot(
        BigDecimal priceSnapshot
    ) {
        this.priceSnapshot = priceSnapshot;
    }

    public String getCurrency() {
        return currency;
    }

    public void setCurrency(
        String currency
    ) {
        this.currency = currency;
    }
}
```

---

# 16. Important Price Rule

The cart may store:

```text
priceSnapshot
```

but that does NOT mean the cart price is final.

Example:

```text
10:00
Cart price = ₹10,000

10:30
Pricing changes = ₹9,000

11:00
Customer checkout
```

The order/checkout process must revalidate the current authoritative price.

Therefore:

```text
Cart price
    =
display/reference price

Order price
    =
authoritative checkout price
```

---

# 17. Repository

## `CartRepository.java`

```java
package com.company.cart.repository;

import com.company.cart.entity.Cart;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface CartRepository
    extends JpaRepository<Cart, Long> {

    Optional<Cart>
    findByCustomerId(
        String customerId
    );
}
```

---

# 18. Cart Item Repository

## `CartItemRepository.java`

```java
package com.company.cart.repository;

import com.company.cart.entity.CartItem;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.List;
import java.util.Optional;

public interface CartItemRepository
    extends JpaRepository<CartItem, Long> {

    List<CartItem>
    findByCartId(Long cartId);

    Optional<CartItem>
    findByCartIdAndProductId(
        Long cartId,
        String productId
    );
}
```

---

# 19. Request DTO

## `AddCartItemRequest.java`

```java
package com.company.cart.dto;

import jakarta.validation.constraints.*;

public class AddCartItemRequest {

    @NotBlank
    private String productId;

    @Min(1)
    @Max(100)
    private Integer quantity;

    public String getProductId() {
        return productId;
    }

    public void setProductId(
        String productId
    ) {
        this.productId = productId;
    }

    public Integer getQuantity() {
        return quantity;
    }

    public void setQuantity(
        Integer quantity
    ) {
        this.quantity = quantity;
    }
}
```

---

# 20. Update Quantity DTO

## `UpdateCartItemRequest.java`

```java
package com.company.cart.dto;

import jakarta.validation.constraints.*;

public class UpdateCartItemRequest {

    @Min(1)
    @Max(100)
    private Integer quantity;

    public Integer getQuantity() {
        return quantity;
    }

    public void setQuantity(
        Integer quantity
    ) {
        this.quantity = quantity;
    }
}
```

---

# 21. Cart Response

## `CartResponse.java`

```java
package com.company.cart.dto;

import java.math.BigDecimal;
import java.util.List;

public record CartResponse(

    String customerId,

    List<CartItemResponse> items,

    BigDecimal subtotal,

    String currency

) {}
```

Item response:

```java
package com.company.cart.dto;

import java.math.BigDecimal;

public record CartItemResponse(

    String productId,

    Integer quantity,

    BigDecimal price,

    BigDecimal lineTotal,

    String currency

) {}
```

---

# 22. Redis Cart Representation

The Redis representation can be:

```text
cart:C1001
```

Example JSON:

```json
{
  "customerId": "C1001",
  "items": [
    {
      "productId": "P1001",
      "quantity": 2,
      "price": "9499.00",
      "currency": "INR"
    }
  ]
}
```

The Redis key should have a TTL.

Example:

```text
cart:C1001
TTL = 30 days
```

Every legitimate cart mutation can refresh the TTL.

---

# 23. Redis Configuration

## `RedisConfig.java`

```java
package com.company.cart.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;

import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, String>
    redisTemplate(
        RedisConnectionFactory factory
    ) {

        RedisTemplate<String, String>
            template = new RedisTemplate<>();

        template.setConnectionFactory(
            factory
        );

        template.setKeySerializer(
            new StringRedisSerializer()
        );

        template.setValueSerializer(
            new StringRedisSerializer()
        );

        return template;
    }
}
```

---

# 24. Cart Service

## `CartService.java`

```java
package com.company.cart.service;

import com.company.cart.dto.AddCartItemRequest;
import com.company.cart.entity.Cart;
import com.company.cart.entity.CartItem;
import com.company.cart.repository.CartItemRepository;
import com.company.cart.repository.CartRepository;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.Duration;

@Service
public class CartService {

    private final CartRepository cartRepository;

    private final CartItemRepository itemRepository;

    private final RedisTemplate<String, String>
        redisTemplate;

    public CartService(
        CartRepository cartRepository,
        CartItemRepository itemRepository,
        RedisTemplate<String, String>
            redisTemplate
    ) {

        this.cartRepository =
            cartRepository;

        this.itemRepository =
            itemRepository;

        this.redisTemplate =
            redisTemplate;
    }

    @Transactional
    public Cart addItem(
        String customerId,
        AddCartItemRequest request
    ) {

        Cart cart =
            cartRepository
                .findByCustomerId(
                    customerId
                )
                .orElseGet(() -> {

                    Cart newCart =
                        new Cart();

                    newCart.setCustomerId(
                        customerId
                    );

                    newCart.setStatus(
                        "ACTIVE"
                    );

                    newCart.setCreatedAt(
                        java.time.Instant.now()
                    );

                    newCart.setUpdatedAt(
                        java.time.Instant.now()
                    );

                    return cartRepository.save(
                        newCart
                    );
                });

        CartItem item =
            itemRepository
                .findByCartIdAndProductId(
                    cart.getId(),
                    request.getProductId()
                )
                .orElseGet(() -> {

                    CartItem newItem =
                        new CartItem();

                    newItem.setCart(cart);

                    newItem.setProductId(
                        request.getProductId()
                    );

                    newItem.setQuantity(0);

                    newItem.setPriceSnapshot(
                        BigDecimal.ZERO
                    );

                    newItem.setCurrency(
                        "INR"
                    );

                    return newItem;
                });

        item.setQuantity(
            item.getQuantity()
                + request.getQuantity()
        );

        itemRepository.save(item);

        cart.setUpdatedAt(
            java.time.Instant.now()
        );

        Cart savedCart =
            cartRepository.save(cart);

        invalidateCartCache(
            customerId
        );

        return savedCart;
    }

    private void invalidateCartCache(
        String customerId
    ) {

        redisTemplate.delete(
            "cart:" + customerId
        );
    }
}
```

This is the baseline implementation.

The final production version must add:

- Product validation
- Inventory validation
- Pricing integration
- Concurrency control
- Idempotency
- Redis write-through/read-through strategy
- Transaction boundaries
- Failure handling
- Event publication

---

# 25. Why the Basic Implementation Is Not Enough

A production cart cannot blindly do:

```text
read cart
+
modify
+
write cart
```

because two requests can arrive simultaneously.

Example:

```text
Quantity = 1
```

Request A:

```text
+1
```

Request B:

```text
+1
```

Both read:

```text
1
```

Both calculate:

```text
2
```

Final:

```text
2
```

Expected:

```text
3
```

This is a race condition.

---

# 26. Cart Concurrency

Possible approaches:

### Option 1

Database optimistic locking.

### Option 2

Redis atomic operations.

### Option 3

Distributed lock.

### Option 4

Single-writer/event-based cart updates.

For this project:

```text
Redis
+
optimistic versioning
+
idempotency
```

will be used where appropriate.

Distributed locks should not be introduced everywhere because locks can create
their own availability and operational problems.

---

# 27. Optimistic Locking

Add:

```java
@Version
private Long version;
```

to `Cart`.

Example:

```java
@Version
@Column(nullable = false)
private Long version;
```

Flow:

```text
Request A
version = 5

Request B
version = 5

A updates
version = 6

B tries version 5
      ↓
conflict
```

The application can retry or return a controlled conflict.

---

# 28. Cart Idempotency

Client sends:

```text
Idempotency-Key:
abc-123
```

If the same request is retried:

```text
abc-123
```

the service should not add the item twice.

This is important because clients, gateways, and retries can repeat requests.

Example:

```text
POST /cart/items

Idempotency-Key: abc-123
```

First request:

```text
SUCCESS
```

Retry:

```text
same result
```

not:

```text
item quantity doubled
```

---

# 29. Cart Expiration

Cart TTL:

```text
30 days
```

Example Redis:

```text
cart:C1001
TTL = 2592000
```

PostgreSQL can retain cart history according to the application's data
retention policy.

Redis expiration does not automatically mean the PostgreSQL record should be
deleted.

---

# 30. Guest Cart

Guest cart:

```text
guest:<session-id>
```

Example:

```text
guest:8a3d...
```

After login:

```text
Guest Cart
    +
Customer Cart
    ↓
Cart Merge
    ↓
Customer Cart
```

---

# 31. Cart Merge

Example:

Guest:

```text
P1001 × 2
P1002 × 1
```

Customer:

```text
P1001 × 1
P1003 × 2
```

After login:

```text
P1001 × 3
P1002 × 1
P1003 × 2
```

But merge must still validate:

- Product existence
- Maximum quantity
- Product availability
- Current price
- Promotion eligibility

---

# 32. Maximum Cart Quantity

Never allow:

```text
quantity = 999999999
```

Example policy:

```text
Maximum per item = 100
Maximum distinct products = 100
```

These are application/business limits.

They should be configurable.

---

# 33. Product Validation

Cart service should not duplicate the entire product database.

Instead:

```text
cart-service
     |
     ↓
product-service
```

Example:

```text
GET /api/v1/products/P1001
```

Validate:

```text
product exists
product active
product purchasable
```

---

# 34. Inventory Validation

When adding:

```text
P1001 × 5
```

cart service may check inventory availability.

But:

> This is an availability check, not a reservation.

The authoritative reservation happens during checkout/order processing.

---

# 35. Pricing Integration

Cart can request:

```text
pricing-service
```

for display pricing.

Example:

```text
P1001
quantity = 2
currency = INR
```

Response:

```text
unit price = ₹9,499
```

Cart calculates:

```text
2 × ₹9,499
=
₹18,998
```

But the final order must revalidate pricing.

---

# 36. Promotion Integration

Cart can ask:

```text
promotion-service
```

for a promotion preview.

Example:

```text
PROMO10
```

Response:

```text
discount = ₹1,899.80
```

But coupon redemption must not necessarily occur when the cart is merely
viewed.

Final redemption belongs to checkout/order flow.

---

# 37. Cart Calculation

Conceptually:

```text
subtotal
    =
Σ(unitPrice × quantity)

discount
    =
promotion result

subtotal after discount
    =
subtotal - discount

tax
    =
tax-service

final total
    =
subtotal
- discount
+ tax
```

The cart should avoid becoming the owner of tax and payment logic.

---

# 38. Dockerfile

## `Dockerfile`

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

# 39. `.dockerignore`

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

# 40. Local Docker Compose

## `docker-compose.yaml`

```yaml
services:

  postgres:

    image: postgres:16

    container_name: cart-postgres

    restart: unless-stopped

    environment:

      POSTGRES_DB: cart

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
          "pg_isready -U postgres -d cart"
        ]

      interval: 10s

      timeout: 5s

      retries: 5


  redis:

    image: redis:7

    container_name: cart-redis

    restart: unless-stopped

    ports:

      - "6379:6379"


  cart-service:

    build:

      context: .

      dockerfile: Dockerfile

    container_name: cart-service

    depends_on:

      postgres:
        condition: service_healthy

      redis:
        condition: service_started

    environment:

      SPRING_PROFILES_ACTIVE: local

      DB_URL: jdbc:postgresql://postgres:5432/cart

      DB_USERNAME: postgres

      DB_PASSWORD: postgres

      REDIS_HOST: redis

      REDIS_PORT: 6379

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

Inside Docker Compose:

```text
localhost
```

must not be used for PostgreSQL or Redis from the application container.

Use:

```text
postgres
redis
```

because Docker provides service-name DNS.

---

# 41. Kubernetes Deployment

## `templates/deployment.yaml`

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: cart-service

  labels:

    app: cart-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: cart-service

  template:

    metadata:

      labels:

        app: cart-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: cart-service

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

                name: cart-service-config

            - secretRef:

                name: cart-service-secrets

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

# 42. Kubernetes Service

## `service.yaml`

```yaml
apiVersion: v1

kind: Service

metadata:

  name: cart-service

  labels:

    app: cart-service

spec:

  type: ClusterIP

  selector:

    app: cart-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 43. ConfigMap

## `configmap.yaml`

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: cart-service-config

data:

  REDIS_HOST: redis.ecommerce.svc.cluster.local

  REDIS_PORT: "6379"

  CART_TTL_SECONDS: "2592000"

  CART_MAX_ITEMS: "100"
```

For production with Amazon ElastiCache, use the actual managed Redis/Valkey
endpoint.

---

# 44. ExternalSecret

## `externalsecret.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: cart-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: cart-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DB_URL

      remoteRef:

        key: ecommerce/prod/cart-service

        property: DB_URL

    - secretKey: DB_USERNAME

      remoteRef:

        key: ecommerce/prod/cart-service

        property: DB_USERNAME

    - secretKey: DB_PASSWORD

      remoteRef:

        key: ecommerce/prod/cart-service

        property: DB_PASSWORD
```

---

# 45. ServiceAccount

## `serviceaccount.yaml`

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: cart-service

  labels:

    app: cart-service
```

If cart needs AWS APIs:

```text
Cart Pod
   ↓
EKS Pod Identity
   ↓
IAM Role
   ↓
AWS API
```

Only required permissions should be granted.

---

# 46. NetworkPolicy

## `networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: cart-service

spec:

  podSelector:

    matchLabels:

      app: cart-service

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

The final policy should be tightened around the actual service-to-service
architecture.

---

# 47. HPA

## `hpa.yaml`

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: cart-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: cart-service

  minReplicas: 3

  maxReplicas: 30

  behavior:

    scaleUp:

      stabilizationWindowSeconds: 0

      policies:

        - type: Percent

          value: 100

          periodSeconds: 60

    scaleDown:

      stabilizationWindowSeconds: 300

      policies:

        - type: Percent

          value: 25

          periodSeconds: 60

  metrics:

    - type: Resource

      resource:

        name: cpu

        target:

          type: Utilization

          averageUtilization: 60
```

For a real production system, custom request-rate/latency metrics may be
evaluated in addition to CPU.

---

# 48. PDB

## `pdb.yaml`

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: cart-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: cart-service
```

---

# 49. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: cart-service

description: Ecommerce cart service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 50. Helm Values

## `values.yaml`

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/cart-service

  tag: ""

  digest: ""

  pullPolicy: IfNotPresent


service:

  type: ClusterIP

  port: 8080


resources:

  requests:

    cpu: 500m

    memory: 512Mi

  limits:

    cpu: "2"

    memory: 1Gi


autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 30

  targetCPUUtilizationPercentage: 60


serviceAccount:

  create: true

  name: cart-service


podDisruptionBudget:

  enabled: true

  minAvailable: 2
```

---

# 51. Production Values

## `values-prod.yaml`

```yaml
replicaCount: 3

resources:

  requests:

    cpu: 500m

    memory: 512Mi

  limits:

    cpu: "2"

    memory: 1Gi


autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 30

  targetCPUUtilizationPercentage: 60
```

The final replica and resource values will come from load testing.

---

# 52. Prometheus

## `servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: cart-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: cart-service

  endpoints:

    - port: http

      path: /actuator/prometheus

      interval: 15s
```

---

# 53. Important Application Metrics

```text
cart_requests_total

cart_request_duration_seconds

cart_add_item_total

cart_update_item_total

cart_remove_item_total

cart_read_total

cart_cache_hits_total

cart_cache_misses_total

cart_cache_errors_total

cart_conflict_total

cart_idempotency_duplicate_total

cart_checkout_total
```

Business metrics:

```text
active_carts

cart_items_added

cart_items_removed

cart_abandoned

cart_converted
```

Avoid putting:

```text
customerId
cartId
productId
```

as unrestricted Prometheus labels.

That can create high cardinality.

---

# 54. Grafana Dashboard

```text
Cart Service
│
├── Request Rate
├── Error Rate
├── P50
├── P95
└── P99
│
├── Redis
│   ├── Hit Ratio
│   ├── Miss Ratio
│   ├── Latency
│   └── Connections
│
├── PostgreSQL
│   ├── Connections
│   ├── Query Latency
│   └── DB Load
│
├── Cart Operations
│   ├── Adds
│   ├── Updates
│   ├── Removes
│   └── Reads
│
└── Kubernetes
    ├── Replicas
    ├── CPU
    ├── Memory
    └── Restarts
```

---

# 55. Structured Logging

Example:

```json
{
  "@timestamp": "2026-10-05T17:00:00Z",
  "level": "INFO",
  "service": "cart-service",
  "environment": "prod",
  "namespace": "ecommerce",
  "pod": "cart-service-7d9d8c",
  "version": "1.0.0",
  "correlationId": "corr-123",
  "operation": "ADD_ITEM",
  "productId": "P1001",
  "quantity": 2,
  "cacheHit": true,
  "message": "Cart item updated"
}
```

Never log:

```text
password
JWT
Authorization header
payment information
secret values
```

Customer identifiers should also be minimized or masked according to the
organization's logging policy.

---

# 56. Distributed Tracing

Cart request:

```text
Customer
   |
   ↓
cart-service
   |
   ├── product-service
   |
   ├── inventory-service
   |
   ├── pricing-service
   |
   └── promotion-service
```

Tracing:

```text
Trace ID
 |
 ├── GET /cart
 |
 ├── pricing request
 |
 └── Redis GET
```

For checkout:

```text
cart
 ↓
pricing
 ↓
promotion
 ↓
inventory
 ↓
order
```

Jaeger helps determine which dependency introduced latency.

---

# 57. SLO

Initial project SLO:

## Availability

```text
99.95%
```

## GET Cart P95

```text
< 100ms
```

## ADD Item P95

```text
< 200ms
```

## Error Rate

```text
< 0.5%
```

These are starting targets and must be validated through performance testing.

---

# 58. Cart Redis Failure

Normal:

```text
GET /cart
    ↓
Redis
    ↓
HIT
```

Redis failure:

```text
GET /cart
    ↓
Redis timeout
    ↓
PostgreSQL
    ↓
Return
```

But if:

```text
50,000 requests
```

all fall back to PostgreSQL:

```text
Redis failure
     ↓
PostgreSQL overload
     ↓
Hikari connection exhaustion
     ↓
Cart latency
     ↓
Order latency
```

Therefore cart needs:

- Short Redis timeout
- Controlled fallback
- Connection pool protection
- Circuit breaker
- Cache recovery
- Cache warming

---

# 59. Cart Cache Stampede

Suppose:

```text
100,000 customers
```

have active carts.

Redis experiences an outage and recovers.

If every application Pod immediately performs:

```text
DB → Redis
```

the recovery can create a PostgreSQL spike.

Therefore:

```text
Redis recovery
     ↓
Gradual cache warming
     ↓
Controlled DB reads
```

is safer than uncontrolled cache repopulation.

---

# 60. Cart Database Strategy

PostgreSQL stores:

```text
Cart
Cart Items
Cart state
Cart metadata
```

Redis stores:

```text
Hot cart representation
```

This gives:

```text
Redis
=
fast operational representation

PostgreSQL
=
durable source of truth
```

---

# 61. Should Cart Exist Only in Redis?

For this enterprise project:

```text
No.
```

A Redis-only cart creates recovery concerns.

Example:

```text
Redis data lost
```

Then:

```text
All active carts disappear
```

If the business accepts this risk, Redis-only carts can be designed.

But our enterprise architecture uses:

```text
Redis + PostgreSQL
```

to provide stronger recovery.

---

# 62. Database Connection Planning

Suppose:

```text
Cart Pods = 20

Hikari max pool = 10
```

Potential:

```text
20 × 10
=
200 connections
```

During deployment:

```text
maxSurge = 1
```

can temporarily create additional Pods.

During canary:

```text
old Pods + new Pods
```

also increases connections.

Therefore connection planning must include:

```text
HPA
+
RollingUpdate
+
Canary
+
Blue/Green
+
Background jobs
+
Other services
```

---

# 63. CI Pipeline

## `.github/workflows/ci.yaml`

```yaml
name: Cart Service CI

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

  ECR_REPOSITORY: cart-service


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


      - name: Make Maven Wrapper executable

        run: chmod +x mvnw


      - name: Test

        run: ./mvnw clean verify


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


      - name: Build Docker image

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |

          docker build \
            -t "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}" \
            .


      - name: Trivy image scan

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

---

# 64. SonarQube

Pipeline:

```text
Checkout
   ↓
Maven build
   ↓
Unit tests
   ↓
Integration tests
   ↓
Coverage
   ↓
SonarQube
   ↓
Dependency scanning
   ↓
Trivy
   ↓
Docker build
   ↓
Trivy image scan
   ↓
ECR
```

SonarQube is centralized.

The scanner runs in CI.

Do not install SonarQube inside the cart application Pods.

---

# 65. GitOps

Application repo:

```text
company-ecommerce/cart-service
```

GitOps repo:

```text
company-ecommerce/gitops
```

Example:

```text
gitops/
└── apps/
    └── cart-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Deployment:

```text
Git SHA
   ↓
Docker image
   ↓
ECR
   ↓
GitOps image update
   ↓
ArgoCD
   ↓
EKS
```

---

# 66. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: cart-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/cart-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

---

# 67. Failure Scenario — Lost Cart Update

Example:

```text
Initial quantity = 1
```

Two requests:

```text
A → +1
B → +1
```

Without concurrency protection:

```text
A reads 1
B reads 1

A writes 2
B writes 2

Final = 2
```

Expected:

```text
3
```

Mitigation:

```text
Optimistic locking
+
Idempotency
+
Atomic Redis/database operation
```

---

# 68. Failure Scenario — Redis Outage

Symptoms:

```text
Redis timeout
Cart GET latency increases
PostgreSQL connections increase
Hikari pending increases
```

Investigation:

```text
Grafana
  ↓
Redis metrics
  ↓
Application logs
  ↓
PostgreSQL connections
  ↓
Jaeger traces
```

Mitigation:

```text
Protect PostgreSQL
Reduce fallback concurrency
Restore Redis
Warm cache
Validate cart reads/writes
```

---

# 69. Failure Scenario — PostgreSQL Failure

If Redis contains the latest cart:

```text
GET cart
 ↓
Redis
 ↓
SUCCESS
```

But writes should not falsely report success if durable persistence is
unavailable.

For example:

```text
ADD item
 ↓
Redis updated
 ↓
PostgreSQL write fails
```

can create:

```text
Redis ≠ PostgreSQL
```

Therefore write ordering and consistency strategy must be explicitly designed.

One safer strategy is:

```text
Validate
  ↓
Persist durable state
  ↓
Update/invalidate cache
  ↓
Return success
```

Another high-scale design may use Redis as the operational write store with
durable asynchronous persistence.

The choice must be explicit rather than accidental.

---

# 70. Failure Scenario — Product Removed

Cart contains:

```text
P1001
```

Product is later discontinued.

Customer opens cart.

Cart service should identify:

```text
product no longer purchasable
```

and return a controlled state such as:

```text
ITEM_UNAVAILABLE
```

rather than blindly allowing checkout.

---

# 71. Failure Scenario — Price Changed

Cart:

```text
P1001
₹10,000
```

Current pricing:

```text
₹9,000
```

At checkout:

```text
Cart price
   ↓
Current Pricing Service
   ↓
₹9,000
```

The order should use the authoritative checkout price.

The customer UI can display:

```text
Price changed from ₹10,000 to ₹9,000.
```

depending on product requirements.

---

# 72. Production Incident

## Incident ID

```text
INC-2026-CART-001
```

## Severity

```text
SEV-2
```

## Title

```text
Redis Failure Caused Cart Latency and PostgreSQL Connection Pressure
```

---

# 73. Customer Impact

Customers experienced:

- Slow cart page
- Slow add-to-cart requests
- Some cart requests returned 5xx
- Checkout latency increased

No confirmed permanent cart data loss occurred.

---

# 74. Timeline

```text
15:00
Cart service healthy.

15:05
Redis latency begins increasing.

15:06
Redis timeout rate increases.

15:07
Cart cache hit ratio drops.

15:08
Cart service begins PostgreSQL fallback.

15:09
RDS connections increase rapidly.

15:10
Hikari pending connections increase.

15:11
GET /cart P95 increases from 80ms to 2.1s.

15:12
Add-to-cart errors begin.

15:13
SEV-2 declared.

15:15
Platform team confirms Redis connectivity issue.

15:17
Fallback concurrency reduced.

15:19
Noncritical cart refresh traffic reduced.

15:21
Redis connectivity restored.

15:23
Cache hit ratio starts recovering.

15:25
RDS connections return toward baseline.

15:27
Cart P95 returns to normal.

15:30
Incident resolved.
```

---

# 75. Root Cause

Redis experienced a temporary connectivity problem.

The cart service correctly attempted PostgreSQL fallback.

However, the fallback was not sufficiently rate-limited.

This caused:

```text
Redis failure
     ↓
Cache misses
     ↓
PostgreSQL reads
     ↓
Hikari pressure
     ↓
Cart latency
```

---

# 76. Amplification

The primary failure:

```text
Redis
```

Amplification:

```text
Uncontrolled database fallback
```

Contributing factors:

- High Redis dependency
- Large cart read traffic
- Insufficient fallback throttling
- Hikari connection pressure
- No cache stampede protection
- Incomplete Redis failure alerting

---

# 77. Five Whys

### Why did cart latency increase?

Cart requests became slow.

### Why?

PostgreSQL received excessive fallback traffic.

### Why?

Redis was unavailable.

### Why did Redis failure create database pressure?

The application fell back to PostgreSQL without sufficient concurrency limits.

### Why was this not prevented?

Redis failure-mode load testing had not been completed.

---

# 78. Corrective Actions

| Action | Owner |
|---|---|
| Add Redis circuit breaker | Backend |
| Add fallback concurrency limits | Backend |
| Add cache stampede protection | Backend |
| Add Redis failure load test | Performance |
| Add Redis-specific alerts | SRE |
| Review Hikari pool size | Platform/DB |
| Review HPA max replicas | Platform |
| Add cart dependency dashboard | SRE |
| Add Redis failure runbook | SRE |
| Add resilience testing to CI/CD | Platform |

---

# 79. Security Checklist

- [ ] No secrets in Git
- [ ] Secrets Manager
- [ ] External Secrets
- [ ] EKS Pod Identity
- [ ] Least privilege
- [ ] NetworkPolicy
- [ ] Private RDS
- [ ] Private Redis
- [ ] Non-root container
- [ ] Read-only filesystem
- [ ] Trivy
- [ ] SonarQube
- [ ] Dependency scanning
- [ ] Secret scanning
- [ ] No credentials in logs

---

# 80. Performance Checklist

- [ ] Redis caching
- [ ] PostgreSQL persistence
- [ ] Cache hit ratio
- [ ] Cache miss rate
- [ ] Redis latency
- [ ] Hikari pool sizing
- [ ] HPA testing
- [ ] Concurrency testing
- [ ] Cart merge testing
- [ ] Redis failure testing
- [ ] PostgreSQL failure testing
- [ ] Load testing
- [ ] Spike testing
- [ ] Soak testing

---

# 81. Production Checklist

## Application

- [ ] Java 21
- [ ] Spring Boot
- [ ] Maven
- [ ] DTO validation
- [ ] Exception handling
- [ ] Unit tests
- [ ] Integration tests
- [ ] Decimal money handling

## Cart

- [ ] Add item
- [ ] Update item
- [ ] Remove item
- [ ] Clear cart
- [ ] Guest cart
- [ ] Cart merge
- [ ] TTL
- [ ] Maximum quantities
- [ ] Idempotency
- [ ] Concurrency protection
- [ ] Price refresh
- [ ] Product validation
- [ ] Inventory availability check

## Database

- [ ] PostgreSQL
- [ ] RDS
- [ ] Hikari
- [ ] Connection budget
- [ ] Indexes
- [ ] Backups
- [ ] PITR

## Redis

- [ ] Cache
- [ ] TTL
- [ ] Invalidation
- [ ] Failure fallback
- [ ] Stampede protection
- [ ] Hit ratio
- [ ] Connection monitoring

## Kubernetes

- [ ] Deployment
- [ ] Service
- [ ] ConfigMap
- [ ] ExternalSecret
- [ ] ServiceAccount
- [ ] NetworkPolicy
- [ ] HPA
- [ ] PDB
- [ ] Resource requests
- [ ] Resource limits
- [ ] Startup probe
- [ ] Readiness probe
- [ ] Liveness probe

## CI/CD

- [ ] Maven
- [ ] Unit tests
- [ ] Integration tests
- [ ] SonarQube
- [ ] Dependency scan
- [ ] Secret scan
- [ ] Trivy
- [ ] Docker build
- [ ] ECR
- [ ] GitHub OIDC

## GitOps

- [ ] Helm
- [ ] GitOps repository
- [ ] ArgoCD
- [ ] Immutable image
- [ ] Environment promotion
- [ ] Rollback

## Observability

- [ ] Prometheus
- [ ] Grafana
- [ ] ELK
- [ ] OpenTelemetry
- [ ] Jaeger
- [ ] Correlation ID
- [ ] Alerts
- [ ] SLO
- [ ] Runbook
- [ ] RCA

---

# 82. Final Architecture

```text
                              Customer
                                 |
                                 ↓
                         CloudFront / WAF
                                 |
                                ALB
                                 |
                                 ↓
                           cart-service
                          /      |       \
                         /       |        \
                        ↓        ↓         ↓
                     Redis    Pricing    Product
                       |       Service    Service
                       |
                       ↓
                  PostgreSQL
                       |
                       ↓
                Durable Cart State


Checkout:

Cart
  |
  ↓
Pricing
  |
  ↓
Promotion
  |
  ↓
Inventory
  |
  ↓
Order
  |
  ↓
Payment


Observability:

Cart Service
     |
     ├── Prometheus → Grafana
     |
     ├── Fluent Bit → Logstash → Elasticsearch → Kibana
     |
     └── OpenTelemetry → Collector → Jaeger
```

---

# 83. Final DevOps Perspective

Cart Service demonstrates an important production architecture:

```text
                    HPA
                     |
             ┌───────┼───────┐
             ↓       ↓       ↓
           Pod 1   Pod 2   Pod 3
             |       |       |
             └───────┼───────┘
                     |
                   Redis
                     |
                 PostgreSQL
```

The application is horizontally scalable.

But the real scaling boundary is:

```text
Cart Pods
    ↓
Redis capacity
    ↓
PostgreSQL capacity
    ↓
Pricing/Product dependencies
```

Therefore:

> More Pods do not automatically mean more capacity.

For cart workloads, the most important production concerns are:

1. Redis latency
2. PostgreSQL fallback protection
3. Cart concurrency
4. Idempotency
5. Cart expiration
6. Price revalidation
7. Inventory availability
8. Hikari connection capacity
9. Cache consistency
10. Checkout correctness

# End of 169 — Cart Service
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

170 Order Service           ⏳
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

**Next: `170-Order-Service.md` — Java + Spring Boot + Maven, the central transaction/orchestration service connecting cart, pricing, promotion, inventory, payment, tax, shipping, events, PostgreSQL, Redis, Kafka/SQS, idempotency, Saga/outbox, CI/CD, Kubernetes, Helm, GitOps, observability, production deployment and RCA.**