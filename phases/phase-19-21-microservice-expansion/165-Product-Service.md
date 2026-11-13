# 165 — Product Service

**File:** `165-Product-Service.md`

```markdown
# 165 — Product Service

## 1. Overview

The `product-service` manages the core product information of the
Enterprise E-Commerce Platform.

This service is responsible for product master data.

It owns:

- Product identity
- SKU
- Product name
- Product description
- Product attributes
- Product images metadata
- Brand
- Product category references
- Product status
- Product lifecycle
- Product metadata

It does NOT own:

- Product price
- Inventory quantity
- Orders
- Payments
- Customers
- Authentication
- Product search indexes as the authoritative source

The service uses:

- Java 21
- Spring Boot
- Maven
- Spring Data JPA
- PostgreSQL
- Redis
- Kafka
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

---

# 2. Product Service vs Catalog Service

This distinction is extremely important.

## Catalog Service

Catalog owns:

- Category hierarchy
- Visibility
- Catalog placement
- Display order
- Featured products

## Product Service

Product owns:

- Product master data
- SKU
- Product name
- Description
- Brand
- Product attributes
- Product lifecycle

Example:

    Catalog Service
          |
          | productId
          ↓
    Product Service
          |
          +── SKU
          +── Name
          +── Description
          +── Brand
          +── Attributes

Therefore catalog-service should NOT maintain a duplicate complete copy of
the product database.

---

# 3. Business Responsibility

The product service provides the authoritative product representation.

Example:

    Product
    ├── productId
    ├── SKU
    ├── name
    ├── description
    ├── brand
    ├── category
    ├── attributes
    ├── image metadata
    └── status

Example product:

    productId: 550e8400-e29b-41d4-a716-446655440000

    SKU:
    IPHONE-17-256-BLK

    Name:
    Example Smartphone

    Brand:
    Example Brand

    Status:
    ACTIVE

Pricing is retrieved from:

    pricing-service

Inventory from:

    inventory-service

Catalog placement from:

    catalog-service

---

# 4. Technology Stack

| Layer | Technology |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot |
| Build | Maven |
| ORM | Spring Data JPA / Hibernate |
| Database | PostgreSQL |
| Cache | Redis |
| Messaging | Kafka |
| Container | Docker |
| Orchestration | Kubernetes / EKS |
| Packaging | Helm |
| Registry | Amazon ECR |
| CI | GitHub Actions |
| GitOps | ArgoCD |
| Metrics | Prometheus |
| Dashboards | Grafana |
| Logging | ELK |
| Tracing | OpenTelemetry |
| Trace backend | Jaeger |

---

# 5. Architecture

```text
                    Internet
                       |
                CloudFront / WAF
                       |
                      ALB
                       |
                       ↓
                product-service
                  /          \
                 /            \
                ↓              ↓
          PostgreSQL          Redis
                |
                ↓
              Kafka
                |
                +----------------+
                |                |
                ↓                ↓
         search-service    catalog-service
```

The product service is the source of truth for product master data.

---

# 6. Request Flow

For a product read:

```text
Client
  |
  ↓
ALB
  |
  ↓
product-service
  |
  ↓
Redis
  |
  ├── HIT → Response
  |
  └── MISS
       |
       ↓
    PostgreSQL
       |
       ↓
      Redis
       |
       ↓
    Response
```

For a product update:

```text
Admin
  |
  ↓
product-service
  |
  ↓
PostgreSQL
  |
  ↓
Cache invalidation
  |
  ↓
Kafka
  |
  +── search-service
  +── catalog-service
  +── reporting-service
```

---

# 7. API Design

## Create Product

```text
POST /api/v1/products
```

## Get Product

```text
GET /api/v1/products/{productId}
```

## Get Product by SKU

```text
GET /api/v1/products/sku/{sku}
```

## List Products

```text
GET /api/v1/products
```

Example:

```text
GET /api/v1/products?page=0&size=20
```

## Update Product

```text
PUT /api/v1/products/{productId}
```

## Change Product Status

```text
PATCH /api/v1/products/{productId}/status
```

## Delete Product

Production systems should generally prefer soft deletion:

```text
DELETE /api/v1/products/{productId}
```

This changes product lifecycle state rather than physically deleting the
record.

## Health

```text
GET /actuator/health
```

## Readiness

```text
GET /actuator/health/readiness
```

## Metrics

```text
GET /actuator/prometheus
```

---

# 8. Repository Structure

```text
product-service/
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/company/product/
│   │   │       │
│   │   │       ├── ProductServiceApplication.java
│   │   │       │
│   │   │       ├── controller/
│   │   │       │   └── ProductController.java
│   │   │       │
│   │   │       ├── service/
│   │   │       │   └── ProductService.java
│   │   │       │
│   │   │       ├── repository/
│   │   │       │   └── ProductRepository.java
│   │   │       │
│   │   │       ├── entity/
│   │   │       │   └── Product.java
│   │   │       │
│   │   │       ├── dto/
│   │   │       │   ├── CreateProductRequest.java
│   │   │       │   ├── UpdateProductRequest.java
│   │   │       │   └── ProductResponse.java
│   │   │       │
│   │   │       ├── exception/
│   │   │       │   ├── ProductNotFoundException.java
│   │   │       │   └── GlobalExceptionHandler.java
│   │   │       │
│   │   │       ├── config/
│   │   │       │   ├── RedisConfig.java
│   │   │       │   └── KafkaConfig.java
│   │   │       │
│   │   │       └── messaging/
│   │   │           └── ProductEventPublisher.java
│   │   │
│   │   └── resources/
│   │       ├── application.yaml
│   │       ├── application-prod.yaml
│   │       └── db/
│   │           └── migration/
│   │
│   └── test/
│       └── java/
│
├── .mvn/
│
├── pom.xml
├── mvnw
├── mvnw.cmd
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── .env.example
├── README.md
│
├── helm/
│   └── product-service/
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

# 9. Maven Configuration

## `pom.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>

<project
    xmlns="http://maven.apache.org/POM/4.0.0"
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
            3.5.6
        </version>

        <relativePath/>
    </parent>

    <groupId>
        com.company.ecommerce
    </groupId>

    <artifactId>
        product-service
    </artifactId>

    <version>
        1.0.0
    </version>

    <name>
        product-service
    </name>

    <description>
        Product master data service
    </description>

    <properties>

        <java.version>
            21
        </java.version>

    </properties>

    <dependencies>

        <!-- REST -->

        <dependency>

            <groupId>
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-web
            </artifactId>

        </dependency>


        <!-- Validation -->

        <dependency>

            <groupId>
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-validation
            </artifactId>

        </dependency>


        <!-- JPA -->

        <dependency>

            <groupId>
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-data-jpa
            </artifactId>

        </dependency>


        <!-- PostgreSQL -->

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


        <!-- Redis -->

        <dependency>

            <groupId>
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-data-redis
            </artifactId>

        </dependency>


        <!-- Kafka -->

        <dependency>

            <groupId>
                org.springframework.kafka
            </groupId>

            <artifactId>
                spring-kafka
            </artifactId>

        </dependency>


        <!-- Actuator -->

        <dependency>

            <groupId>
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-actuator
            </artifactId>

        </dependency>


        <!-- Prometheus -->

        <dependency>

            <groupId>
                io.micrometer
            </groupId>

            <artifactId>
                micrometer-registry-prometheus
            </artifactId>

        </dependency>


        <!-- OpenTelemetry -->

        <dependency>

            <groupId>
                io.micrometer
            </groupId>

            <artifactId>
                micrometer-tracing-bridge-otel
            </artifactId>

        </dependency>


        <!-- Tests -->

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


        <dependency>

            <groupId>
                org.springframework.kafka
            </groupId>

            <artifactId>
                spring-kafka-test
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

The project uses Maven Wrapper so developers and CI use the repository's
controlled Maven version.

---

# 10. Main Application

## `ProductServiceApplication.java`

```java
package com.company.product;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class ProductServiceApplication {

    public static void main(String[] args) {

        SpringApplication.run(
            ProductServiceApplication.class,
            args
        );
    }
}
```

---

# 11. Product Entity

## `Product.java`

```java
package com.company.product.entity;

import jakarta.persistence.*;

import java.time.Instant;
import java.util.UUID;

@Entity
@Table(
    name = "products",
    indexes = {
        @Index(
            name = "idx_products_sku",
            columnList = "sku"
        ),
        @Index(
            name = "idx_products_status",
            columnList = "status"
        ),
        @Index(
            name = "idx_products_brand",
            columnList = "brand"
        )
    }
)
public class Product {

    @Id
    @GeneratedValue(
        strategy = GenerationType.IDENTITY
    )
    private Long id;

    @Column(
        name = "product_id",
        nullable = false,
        unique = true
    )
    private UUID productId;

    @Column(
        nullable = false,
        unique = true,
        length = 100
    )
    private String sku;

    @Column(
        nullable = false,
        length = 255
    )
    private String name;

    @Column(
        columnDefinition = "TEXT"
    )
    private String description;

    @Column(
        nullable = false,
        length = 150
    )
    private String brand;

    @Column(
        name = "category_id",
        nullable = false
    )
    private UUID categoryId;

    @Enumerated(EnumType.STRING)
    @Column(
        nullable = false,
        length = 30
    )
    private ProductStatus status;

    @Column(
        name = "created_at",
        nullable = false
    )
    private Instant createdAt;

    @Column(
        name = "updated_at",
        nullable = false
    )
    private Instant updatedAt;

    @PrePersist
    void prePersist() {

        if (productId == null) {
            productId = UUID.randomUUID();
        }

        Instant now = Instant.now();

        createdAt = now;
        updatedAt = now;

        if (status == null) {
            status = ProductStatus.ACTIVE;
        }
    }

    @PreUpdate
    void preUpdate() {

        updatedAt = Instant.now();
    }

    public Long getId() {
        return id;
    }

    public UUID getProductId() {
        return productId;
    }

    public String getSku() {
        return sku;
    }

    public void setSku(String sku) {
        this.sku = sku;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getDescription() {
        return description;
    }

    public void setDescription(
        String description
    ) {
        this.description = description;
    }

    public String getBrand() {
        return brand;
    }

    public void setBrand(String brand) {
        this.brand = brand;
    }

    public UUID getCategoryId() {
        return categoryId;
    }

    public void setCategoryId(
        UUID categoryId
    ) {
        this.categoryId = categoryId;
    }

    public ProductStatus getStatus() {
        return status;
    }

    public void setStatus(
        ProductStatus status
    ) {
        this.status = status;
    }

    public Instant getCreatedAt() {
        return createdAt;
    }

    public Instant getUpdatedAt() {
        return updatedAt;
    }
}
```

---

# 12. Product Status

## `ProductStatus.java`

```java
package com.company.product.entity;

public enum ProductStatus {

    DRAFT,

    ACTIVE,

    INACTIVE,

    DISCONTINUED
}
```

---

# 13. Create Product DTO

## `CreateProductRequest.java`

```java
package com.company.product.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

import java.util.UUID;

public record CreateProductRequest(

    @NotBlank
    @Size(max = 100)
    String sku,

    @NotBlank
    @Size(max = 255)
    String name,

    @Size(max = 5000)
    String description,

    @NotBlank
    @Size(max = 150)
    String brand,

    @NotNull
    UUID categoryId

) {}
```

---

# 14. Update Product DTO

## `UpdateProductRequest.java`

```java
package com.company.product.dto;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Size;

import java.util.UUID;

public record UpdateProductRequest(

    @NotBlank
    @Size(max = 255)
    String name,

    @Size(max = 5000)
    String description,

    @NotBlank
    @Size(max = 150)
    String brand,

    @NotNull
    UUID categoryId

) {}
```

---

# 15. Product Response

## `ProductResponse.java`

```java
package com.company.product.dto;

import com.company.product.entity.Product;

import java.time.Instant;
import java.util.UUID;

public record ProductResponse(

    UUID productId,

    String sku,

    String name,

    String description,

    String brand,

    UUID categoryId,

    String status,

    Instant createdAt,

    Instant updatedAt

) {

    public static ProductResponse from(
        Product product
    ) {

        return new ProductResponse(
            product.getProductId(),
            product.getSku(),
            product.getName(),
            product.getDescription(),
            product.getBrand(),
            product.getCategoryId(),
            product.getStatus().name(),
            product.getCreatedAt(),
            product.getUpdatedAt()
        );
    }
}
```

---

# 16. Repository

## `ProductRepository.java`

```java
package com.company.product.repository;

import com.company.product.entity.Product;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;
import java.util.UUID;

public interface ProductRepository
    extends JpaRepository<Product, Long> {

    Optional<Product> findByProductId(
        UUID productId
    );

    Optional<Product> findBySku(
        String sku
    );

    Page<Product> findByStatus(
        String status,
        Pageable pageable
    );

    boolean existsBySku(
        String sku
    );
}
```

---

# 17. Product Not Found Exception

## `ProductNotFoundException.java`

```java
package com.company.product.exception;

public class ProductNotFoundException
    extends RuntimeException {

    public ProductNotFoundException(
        String message
    ) {

        super(message);
    }
}
```

---

# 18. Product Service Layer

## `ProductService.java`

```java
package com.company.product.service;

import com.company.product.dto.CreateProductRequest;
import com.company.product.dto.ProductResponse;
import com.company.product.dto.UpdateProductRequest;

import com.company.product.entity.Product;
import com.company.product.entity.ProductStatus;

import com.company.product.exception.ProductNotFoundException;
import com.company.product.repository.ProductRepository;

import com.company.product.messaging.ProductEventPublisher;

import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

import org.springframework.stereotype.Service;

import org.springframework.transaction.annotation.Transactional;

import java.util.UUID;

@Service
public class ProductService {

    private final ProductRepository repository;

    private final ProductEventPublisher eventPublisher;

    public ProductService(
        ProductRepository repository,
        ProductEventPublisher eventPublisher
    ) {

        this.repository = repository;

        this.eventPublisher =
            eventPublisher;
    }

    @Transactional(readOnly = true)
    @Cacheable(
        value = "products",
        key = "#productId"
    )
    public ProductResponse getProduct(
        UUID productId
    ) {

        Product product =
            repository.findByProductId(
                productId
            ).orElseThrow(
                () -> new ProductNotFoundException(
                    "Product not found: "
                    + productId
                )
            );

        return ProductResponse.from(product);
    }

    @Transactional(readOnly = true)
    public ProductResponse getBySku(
        String sku
    ) {

        Product product =
            repository.findBySku(sku)
                .orElseThrow(
                    () ->
                        new ProductNotFoundException(
                            "Product not found for SKU: "
                            + sku
                        )
                );

        return ProductResponse.from(product);
    }

    @Transactional
    public ProductResponse createProduct(
        CreateProductRequest request
    ) {

        if (
            repository.existsBySku(
                request.sku()
            )
        ) {

            throw new IllegalArgumentException(
                "SKU already exists"
            );
        }

        Product product = new Product();

        product.setSku(request.sku());
        product.setName(request.name());
        product.setDescription(
            request.description()
        );
        product.setBrand(request.brand());
        product.setCategoryId(
            request.categoryId()
        );
        product.setStatus(
            ProductStatus.ACTIVE
        );

        Product saved =
            repository.save(product);

        eventPublisher.publishProductCreated(
            saved
        );

        return ProductResponse.from(saved);
    }

    @Transactional
    @CacheEvict(
        value = "products",
        key = "#productId"
    )
    public ProductResponse updateProduct(
        UUID productId,
        UpdateProductRequest request
    ) {

        Product product =
            repository.findByProductId(
                productId
            ).orElseThrow(
                () ->
                    new ProductNotFoundException(
                        "Product not found"
                    )
            );

        product.setName(
            request.name()
        );

        product.setDescription(
            request.description()
        );

        product.setBrand(
            request.brand()
        );

        product.setCategoryId(
            request.categoryId()
        );

        Product saved =
            repository.save(product);

        eventPublisher.publishProductUpdated(
            saved
        );

        return ProductResponse.from(saved);
    }

    @Transactional
    @CacheEvict(
        value = "products",
        key = "#productId"
    )
    public void discontinue(
        UUID productId
    ) {

        Product product =
            repository.findByProductId(
                productId
            ).orElseThrow(
                () ->
                    new ProductNotFoundException(
                        "Product not found"
                    )
            );

        product.setStatus(
            ProductStatus.DISCONTINUED
        );

        repository.save(product);

        eventPublisher.publishProductUpdated(
            product
        );
    }

    @Transactional(readOnly = true)
    public Page<ProductResponse> listProducts(
        Pageable pageable
    ) {

        return repository
            .findAll(pageable)
            .map(ProductResponse::from);
    }
}
```

---

# 19. Important Production Note

The simplified example above publishes Kafka after the database save.

For critical product events, the production implementation should eventually
use the transactional outbox pattern:

```text
Database Transaction
        |
        +── products
        |
        +── outbox_events
                 |
                 ↓
          Outbox Publisher
                 |
                 ↓
               Kafka
```

This avoids:

```text
Database SUCCESS
      |
      X
Kafka FAILURE
```

leaving downstream services unaware of the product change.

---

# 20. Redis Configuration

## `RedisConfig.java`

```java
package com.company.product.config;

import org.springframework.cache.annotation.EnableCaching;

import org.springframework.context.annotation.Configuration;

@Configuration
@EnableCaching
public class RedisConfig {
}
```

Spring Boot will configure Redis using the application properties.

---

# 21. Kafka Event Publisher

## `ProductEventPublisher.java`

```java
package com.company.product.messaging;

import com.company.product.entity.Product;

import org.springframework.kafka.core.KafkaTemplate;

import org.springframework.stereotype.Component;

import java.time.Instant;
import java.util.UUID;

@Component
public class ProductEventPublisher {

    private final KafkaTemplate<String, Object>
        kafkaTemplate;

    public ProductEventPublisher(
        KafkaTemplate<String, Object>
            kafkaTemplate
    ) {

        this.kafkaTemplate =
            kafkaTemplate;
    }

    public void publishProductCreated(
        Product product
    ) {

        publish(
            "ProductCreated",
            product
        );
    }

    public void publishProductUpdated(
        Product product
    ) {

        publish(
            "ProductUpdated",
            product
        );
    }

    private void publish(
        String eventType,
        Product product
    ) {

        ProductEvent event =
            new ProductEvent(
                UUID.randomUUID(),
                eventType,
                1,
                Instant.now(),
                "product-service",
                product.getProductId()
            );

        kafkaTemplate.send(
            "product-events",
            product.getProductId().toString(),
            event
        );
    }

    public record ProductEvent(
        UUID eventId,
        String eventType,
        int version,
        Instant occurredAt,
        String source,
        UUID productId
    ) {}
}
```

Again, the final production implementation should move important event
publication behind an outbox.

---

# 22. Product Controller

## `ProductController.java`

```java
package com.company.product.controller;

import com.company.product.dto.CreateProductRequest;
import com.company.product.dto.ProductResponse;
import com.company.product.dto.UpdateProductRequest;

import com.company.product.service.ProductService;

import jakarta.validation.Valid;

import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(
        ProductService productService
    ) {

        this.productService =
            productService;
    }

    @PostMapping
    public ResponseEntity<ProductResponse>
    createProduct(
        @Valid
        @RequestBody
        CreateProductRequest request
    ) {

        ProductResponse response =
            productService.createProduct(
                request
            );

        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(response);
    }

    @GetMapping("/{productId}")
    public ResponseEntity<ProductResponse>
    getProduct(
        @PathVariable
        UUID productId
    ) {

        return ResponseEntity.ok(
            productService.getProduct(
                productId
            )
        );
    }

    @GetMapping("/sku/{sku}")
    public ResponseEntity<ProductResponse>
    getBySku(
        @PathVariable
        String sku
    ) {

        return ResponseEntity.ok(
            productService.getBySku(
                sku
            )
        );
    }

    @GetMapping
    public ResponseEntity<
        Page<ProductResponse>
    > listProducts(
        Pageable pageable
    ) {

        return ResponseEntity.ok(
            productService.listProducts(
                pageable
            )
        );
    }

    @PutMapping("/{productId}")
    public ResponseEntity<ProductResponse>
    updateProduct(
        @PathVariable
        UUID productId,

        @Valid
        @RequestBody
        UpdateProductRequest request
    ) {

        return ResponseEntity.ok(
            productService.updateProduct(
                productId,
                request
            )
        );
    }

    @DeleteMapping("/{productId}")
    public ResponseEntity<Void>
    discontinueProduct(
        @PathVariable
        UUID productId
    ) {

        productService.discontinue(
            productId
        );

        return ResponseEntity.noContent()
            .build();
    }
}
```

---

# 23. Global Exception Handler

## `GlobalExceptionHandler.java`

```java
package com.company.product.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;

import org.springframework.web.bind.annotation.*;

import java.time.Instant;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(
        ProductNotFoundException.class
    )
    public ResponseEntity<?> handleNotFound(
        ProductNotFoundException exception
    ) {

        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(
                Map.of(
                    "timestamp",
                    Instant.now(),

                    "status",
                    404,

                    "error",
                    "PRODUCT_NOT_FOUND",

                    "message",
                    exception.getMessage()
                )
            );
    }

    @ExceptionHandler(
        IllegalArgumentException.class
    )
    public ResponseEntity<?> handleBadRequest(
        IllegalArgumentException exception
    ) {

        return ResponseEntity
            .badRequest()
            .body(
                Map.of(
                    "timestamp",
                    Instant.now(),

                    "status",
                    400,

                    "error",
                    "BAD_REQUEST",

                    "message",
                    exception.getMessage()
                )
            );
    }
}
```

---

# 24. Local Application Configuration

## `application.yaml`

```yaml
server:

  port: 8080


spring:

  application:

    name: product-service


  datasource:

    url: ${DB_URL:jdbc:postgresql://localhost:5432/products}

    username: ${DB_USERNAME:postgres}

    password: ${DB_PASSWORD:postgres}

    hikari:

      maximum-pool-size: 10

      minimum-idle: 2

      connection-timeout: 3000

      validation-timeout: 1000

      idle-timeout: 600000

      max-lifetime: 1500000

      pool-name: product-service-hikari


  jpa:

    hibernate:

      ddl-auto: update

    show-sql: false

    open-in-view: false

    properties:

      hibernate:

        format_sql: true


  data:

    redis:

      host: ${REDIS_HOST:localhost}

      port: ${REDIS_PORT:6379}

      timeout: 1s


  kafka:

    bootstrap-servers:
      ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}

    producer:

      key-serializer:
        org.apache.kafka.common.serialization.StringSerializer

      value-serializer:
        org.springframework.kafka.support.serializer.JsonSerializer


management:

  endpoints:

    web:

      exposure:

        include:
          health,info,prometheus


  endpoint:

    health:

      probes:

        enabled: true


logging:

  level:

    root: INFO

    com.company.product: INFO
```

---

# 25. Production Application Configuration

## `application-prod.yaml`

```yaml
server:

  port: 8080


spring:

  application:

    name: product-service


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

      pool-name: product-service-hikari


  jpa:

    hibernate:

      ddl-auto: validate

    show-sql: false

    open-in-view: false

    properties:

      hibernate:

        format_sql: false

        jdbc:

          batch_size: 50

        order_inserts: true

        order_updates: true


  data:

    redis:

      host: ${REDIS_HOST}

      port: ${REDIS_PORT:6379}

      timeout: 1s


  kafka:

    bootstrap-servers:
      ${KAFKA_BOOTSTRAP_SERVERS}

    producer:

      acks: all

      retries: 5

      properties:

        enable.idempotence: true


management:

  endpoints:

    web:

      exposure:

        include:
          health,info,prometheus


  endpoint:

    health:

      probes:

        enabled: true


logging:

  level:

    root: INFO

    com.company.product: INFO
```

The Hikari values above are starting values for this project, not universal
production settings. They must be validated through load testing and the
actual RDS connection budget.

---

# 26. Database Migration

Example:

## `V1__create_products.sql`

```sql
CREATE TABLE products (

    id BIGSERIAL PRIMARY KEY,

    product_id UUID NOT NULL UNIQUE,

    sku VARCHAR(100) NOT NULL UNIQUE,

    name VARCHAR(255) NOT NULL,

    description TEXT,

    brand VARCHAR(150) NOT NULL,

    category_id UUID NOT NULL,

    status VARCHAR(30) NOT NULL,

    created_at TIMESTAMPTZ NOT NULL,

    updated_at TIMESTAMPTZ NOT NULL
);


CREATE INDEX idx_products_status
ON products(status);


CREATE INDEX idx_products_brand
ON products(brand);


CREATE INDEX idx_products_category_id
ON products(category_id);
```

Future schema changes follow:

```text
V1__create_products.sql
V2__add_product_attributes.sql
V3__add_product_images.sql
V4__add_product_lifecycle.sql
```

Applied migrations are never edited.

---

# 27. Dockerfile

## `Dockerfile`

```dockerfile
FROM eclipse-temurin:21-jdk-alpine AS builder

WORKDIR /build

COPY .mvn/ .mvn/

COPY mvnw pom.xml ./

RUN chmod +x mvnw

RUN ./mvnw dependency:go-offline

COPY src ./src

RUN ./mvnw clean package \
    -DskipTests


FROM eclipse-temurin:21-jre-alpine

WORKDIR /app

COPY --from=builder \
    /build/target/*.jar \
    app.jar

RUN addgroup \
    --system \
    --gid 10001 \
    appgroup \
    && adduser \
    --system \
    --uid 10001 \
    --ingroup appgroup \
    appuser

USER 10001

EXPOSE 8080

ENTRYPOINT [
    "java",
    "-jar",
    "/app/app.jar"
]
```

The final runtime image contains:

- JRE
- Application JAR

It does not contain:

- Maven
- Source code
- Git metadata
- Build cache

---

# 28. `.dockerignore`

```text
.git
.github
.idea
.vscode

target

.mvn/wrapper/maven-wrapper.jar

*.log

.env
.env.*

README.md

docker-compose.yaml
```

---

# 29. Local Docker Compose

## `docker-compose.yaml`

```yaml
services:

  postgres:

    image: postgres:16

    container_name: product-postgres

    restart: unless-stopped

    environment:

      POSTGRES_DB: products

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
          "pg_isready -U postgres -d products"
        ]

      interval: 10s

      timeout: 5s

      retries: 5


  redis:

    image: redis:7

    container_name: product-redis

    restart: unless-stopped

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: product-kafka

    restart: unless-stopped

    ports:

      - "9092:9092"


  product-service:

    build:

      context: .

      dockerfile: Dockerfile

    container_name: product-service

    restart: unless-stopped

    depends_on:

      postgres:
        condition: service_healthy

      redis:
        condition: service_started

      kafka:
        condition: service_started

    environment:

      SPRING_PROFILES_ACTIVE: local

      DB_URL: jdbc:postgresql://postgres:5432/products

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

---

# 30. Local Testing

Start dependencies:

```bash
docker compose up -d postgres redis kafka
```

Build:

```bash
./mvnw clean package
```

Run application:

```bash
./mvnw spring-boot:run
```

Or:

```bash
java -jar target/product-service-1.0.0.jar
```

Health:

```bash
curl http://localhost:8080/actuator/health
```

---

# 31. Create Product

Example:

```bash
curl \
  -X POST \
  http://localhost:8080/api/v1/products \
  -H "Content-Type: application/json" \
  -d '{
    "sku": "PHONE-001-BLK",
    "name": "Example Smartphone",
    "description": "Example product",
    "brand": "Example Brand",
    "categoryId": "550e8400-e29b-41d4-a716-446655440000"
  }'
```

Expected:

```text
HTTP 201
```

---

# 32. Get Product

```bash
curl \
  http://localhost:8080/api/v1/products/<product-id>
```

First request:

```text
Application
    ↓
Redis MISS
    ↓
PostgreSQL
    ↓
Redis SET
    ↓
Response
```

Second request:

```text
Application
    ↓
Redis HIT
    ↓
Response
```

---

# 33. Update Product

```bash
curl \
  -X PUT \
  http://localhost:8080/api/v1/products/<product-id> \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Updated Smartphone",
    "description": "Updated description",
    "brand": "Example Brand",
    "categoryId": "550e8400-e29b-41d4-a716-446655440000"
  }'
```

The service should:

```text
Update PostgreSQL
       ↓
Invalidate Redis
       ↓
Publish ProductUpdated
```

---

# 34. Kubernetes ConfigMap

## `templates/configmap.yaml`

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: product-service-config

data:

  SPRING_PROFILES_ACTIVE: prod

  REDIS_HOST: redis.ecommerce.svc.cluster.local

  REDIS_PORT: "6379"

  KAFKA_BOOTSTRAP_SERVERS: kafka.ecommerce.svc.cluster.local:9092
```

The actual Redis and Kafka addresses depend on whether the project uses
ElastiCache/MSK or Kubernetes-hosted messaging.

---

# 35. External Secrets

## `templates/externalsecret.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: product-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: product-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DB_URL

      remoteRef:

        key: ecommerce/prod/product-service

        property: DB_URL

    - secretKey: DB_USERNAME

      remoteRef:

        key: ecommerce/prod/product-service

        property: DB_USERNAME

    - secretKey: DB_PASSWORD

      remoteRef:

        key: ecommerce/prod/product-service

        property: DB_PASSWORD
```

Architecture:

```text
AWS Secrets Manager
        ↓
External Secrets Operator
        ↓
Kubernetes Secret
        ↓
product-service Pod
```

---

# 36. ServiceAccount

## `templates/serviceaccount.yaml`

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: product-service
```

If the service later needs S3 access for product image metadata or another AWS
API:

```text
Product Pod
    ↓
EKS Pod Identity
    ↓
Product IAM Role
    ↓
S3
```

The IAM policy should only permit the required bucket/prefix.

---

# 37. Kubernetes Deployment

## `templates/deployment.yaml`

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: product-service

  labels:

    app: product-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: product-service

  template:

    metadata:

      labels:

        app: product-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      terminationGracePeriodSeconds: 30

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: product-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: IfNotPresent

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: product-service-config

            - secretRef:

                name: product-service-secrets

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

# 38. Kubernetes Service

## `templates/service.yaml`

```yaml
apiVersion: v1

kind: Service

metadata:

  name: product-service

  labels:

    app: product-service

spec:

  type: ClusterIP

  selector:

    app: product-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

Internal DNS:

```text
product-service.ecommerce.svc.cluster.local
```

---

# 39. NetworkPolicy

## `templates/networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: product-service

spec:

  podSelector:

    matchLabels:

      app: product-service

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

    - to:

        - namespaceSelector:

            matchLabels:

              kubernetes.io/metadata.name: ecommerce

      ports:

        - protocol: TCP

          port: 5432

        - protocol: TCP

          port: 6379

        - protocol: TCP

          port: 9092

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

This is a baseline example.

The final production NetworkPolicy should be based on the actual dependency
graph.

---

# 40. HPA

## `templates/hpa.yaml`

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: product-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: product-service

  minReplicas: {{ .Values.autoscaling.minReplicas }}

  maxReplicas: {{ .Values.autoscaling.maxReplicas }}

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

Product service is expected to receive substantial read traffic, so later we
can introduce a custom metric such as request rate if load testing proves that
CPU is not a sufficient scaling signal.

---

# 41. PodDisruptionBudget

## `templates/pdb.yaml`

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: product-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: product-service
```

---

# 42. Helm Values

## `values.yaml`

```yaml
replicaCount: 2

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/product-service

  tag: ""

  digest: ""

  pullPolicy: IfNotPresent


service:

  type: ClusterIP

  port: 8080


resources:

  requests:

    cpu: 250m

    memory: 512Mi

  limits:

    cpu: "1"

    memory: 1Gi


autoscaling:

  enabled: true

  minReplicas: 2

  maxReplicas: 20

  targetCPUUtilizationPercentage: 60


serviceAccount:

  create: true

  name: product-service


podDisruptionBudget:

  enabled: true

  minAvailable: 1
```

---

# 43. Production Values

## `values-prod.yaml`

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/product-service

  tag: ""

  digest: ""


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

  maxReplicas: 25

  targetCPUUtilizationPercentage: 60


podDisruptionBudget:

  enabled: true

  minAvailable: 2
```

These are simulation starting values.

Actual limits must be derived from:

```text
Load testing
+
RPS
+
P95/P99
+
JVM memory
+
CPU
+
RDS connections
+
Redis capacity
+
HPA behavior
```

---

# 44. Prometheus ServiceMonitor

## `templates/servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: product-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: product-service

  endpoints:

    - port: http

      path: /actuator/prometheus

      interval: 15s
```

---

# 45. CODEOWNERS

## `.github/CODEOWNERS`

```text
/src/main/java/     @company-ecommerce/backend-team
/src/main/resources/ @company-ecommerce/backend-team
/helm/              @company-ecommerce/platform-team
/.github/           @company-ecommerce/platform-team
/security/          @company-ecommerce/security-team
```

---

# 46. GitHub Actions CI

## `.github/workflows/ci.yaml`

```yaml
name: Product Service CI

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

  ECR_REPOSITORY: product-service


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


      - name: Compile and test

        run: ./mvnw clean verify


      - name: Package

        run: ./mvnw package -DskipTests


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


      - name: Resolve image digest

        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |

          aws ecr describe-images \
            --repository-name "$ECR_REPOSITORY" \
            --image-ids imageTag="${GITHUB_SHA}" \
            --query 'imageDetails[0].imageDigest' \
            --output text
```

---

# 47. CI Pipeline

The complete pipeline is:

```text
Developer
    |
    ↓
Pull Request
    |
    ↓
GitHub Actions
    |
    ├── Checkout
    |
    ├── Java 21
    |
    ├── Maven
    |
    ├── Unit Tests
    |
    ├── Integration Tests
    |
    ├── SonarQube
    |
    ├── Dependency Scan
    |
    ├── Secret Scan
    |
    ├── Docker Build
    |
    ├── Trivy
    |
    ├── AWS OIDC
    |
    ├── ECR Push
    |
    └── Image Digest
             |
             ↓
          GitOps
```

---

# 48. SonarQube

SonarQube analyzes:

- Java bugs
- Vulnerabilities
- Code smells
- Duplicated code
- Maintainability
- Test coverage

The project should fail CI when organization-defined quality gates are not
met.

SonarQube is a centralized platform service.

It is not installed inside every application Pod.

The scanner runs in the CI environment.

---

# 49. Trivy

Trivy is used at multiple points.

```text
Source Repository
       ↓
Filesystem Scan
       ↓
Docker Build
       ↓
Container Image
       ↓
Image Scan
       ↓
ECR
```

Production application Pods do not need Trivy installed inside them.

---

# 50. ECR

Repository:

```text
company-ecommerce/product-service
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/product-service:<git-sha>
```

Preferred deployment identity:

```text
product-service@sha256:<digest>
```

The repository should use immutable tags.

---

# 51. GitOps

Application repository:

```text
company-ecommerce/product-service
```

GitOps repository:

```text
company-ecommerce/gitops
```

Structure:

```text
gitops/
└── apps/
    └── product-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Promotion:

```text
Git commit
    ↓
CI
    ↓
Docker image
    ↓
ECR
    ↓
GitOps PR
    ↓
Approval
    ↓
ArgoCD
    ↓
EKS
```

CI does not directly run:

```bash
kubectl apply
```

against production.

---

# 52. ArgoCD Application

Example:

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: product-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/product-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true

    syncOptions:

      - CreateNamespace=true
```

---

# 53. Observability

Important application metrics:

```text
HTTP request rate
HTTP latency
HTTP 4xx
HTTP 5xx

JVM CPU
JVM memory
Heap usage
GC activity
Thread count

Hikari active connections
Hikari idle connections
Hikari pending requests

PostgreSQL query latency
PostgreSQL connections

Redis hit ratio
Redis latency
Redis errors

Kafka producer errors
Kafka publish latency

Pod restarts
HPA replicas
```

---

# 54. Business Metrics

Examples:

```text
product_created_total
product_updated_total
product_read_total
product_not_found_total
product_discontinued_total
```

Avoid high-cardinality labels.

Do NOT use:

```text
productId
SKU
customerId
email
phone
```

as uncontrolled Prometheus labels.

---

# 55. Grafana Dashboard

Recommended dashboard:

```text
Product Service
│
├── Traffic
│   ├── RPS
│   ├── 4xx
│   └── 5xx
│
├── Latency
│   ├── P50
│   ├── P95
│   └── P99
│
├── JVM
│   ├── Heap
│   ├── GC
│   └── Threads
│
├── Hikari
│   ├── Active
│   ├── Idle
│   ├── Pending
│   └── Acquisition latency
│
├── Redis
│   ├── Hit ratio
│   ├── Miss ratio
│   └── Errors
│
├── PostgreSQL
│   ├── Connections
│   ├── DB load
│   └── Query latency
│
├── Kafka
│   ├── Producer errors
│   └── Publish latency
│
└── Kubernetes
    ├── Pods
    ├── CPU
    ├── Memory
    └── HPA
```

---

# 56. Structured Logging

Example:

```json
{
  "@timestamp": "2026-10-05T13:00:00Z",
  "level": "INFO",
  "service": "product-service",
  "environment": "prod",
  "namespace": "ecommerce",
  "pod": "product-service-7c9d7f-x7abc",
  "version": "1.0.0",
  "correlationId": "abc-123",
  "route": "/api/v1/products/{productId}",
  "message": "Product retrieved"
}
```

Never log:

```text
Database passwords
Redis passwords
Authorization headers
JWTs
Secrets
Sensitive customer information
```

---

# 57. Distributed Tracing

Architecture:

```text
Client
  |
  ↓
ALB
  |
  ↓
product-service
  |
  ├── PostgreSQL
  |
  ├── Redis
  |
  └── Kafka
```

Tracing:

```text
product-service
       ↓
OpenTelemetry
       ↓
OTel Collector
       ↓
Jaeger
```

Example trace:

```text
GET /api/v1/products/{id}
        |
        +── Redis GET
        |
        +── PostgreSQL SELECT
        |
        +── Redis SET
```

This makes it possible to identify the actual latency source.

---

# 58. Correlation ID

Example:

```text
X-Correlation-ID: 8c7d...
```

The ID should propagate through:

```text
ALB
 ↓
product-service
 ↓
PostgreSQL/log context
 ↓
Kafka event
 ↓
search-service
 ↓
notification/reporting consumers
```

Event envelope:

```json
{
  "eventId": "01J...",
  "eventType": "ProductUpdated",
  "version": 1,
  "occurredAt": "2026-10-05T13:00:00Z",
  "source": "product-service",
  "correlationId": "abc-123",
  "payload": {
    "productId": "..."
  }
}
```

---

# 59. SLO

Initial project targets:

## Availability

```text
99.9%
```

## Product GET P95

```text
< 150ms
```

## Product GET P99

```text
< 300ms
```

## Product write P95

```text
< 500ms
```

These are simulation targets.

They must be validated through load and performance testing.

---

# 60. Failure Scenario — Redis Failure

Normal:

```text
Request
  ↓
Redis HIT
  ↓
Response
```

Redis failure:

```text
Request
  ↓
Redis failure
  ↓
PostgreSQL
```

If traffic is high:

```text
Redis failure
     ↓
Cache misses
     ↓
PostgreSQL traffic
     ↓
Hikari pressure
     ↓
RDS pressure
     ↓
API latency
```

Protection:

- Short Redis timeout
- Bounded retry
- Request coalescing
- Cache warming
- DB connection protection
- HPA guardrails

---

# 61. Failure Scenario — Database Failure

Symptoms:

```text
HTTP 5xx
Connection timeout
Hikari pending
High latency
```

Investigate:

```text
RDS status
RDS events
Database connections
Long-running queries
Locks
Security groups
DNS
NetworkPolicy
Credentials
Hikari pool
Recent deployments
```

Do not blindly increase:

```text
Hikari maximum-pool-size
```

because that can increase pressure on an already unhealthy database.

---

# 62. Failure Scenario — Kafka Failure

Product update:

```text
PostgreSQL
    ↓
SUCCESS

Kafka
    ↓
FAILURE
```

For critical product events, the solution is:

```text
PostgreSQL transaction
       |
       +── products
       |
       +── outbox_events
                 |
                 ↓
          Outbox publisher
                 |
                 ↓
               Kafka
```

This is the preferred production pattern.

---

# 63. Failure Scenario — Bad Product Deployment

Suppose version:

```text
product-service:1.5.0
```

has a serialization bug.

Deployment:

```text
GitOps
  ↓
ArgoCD
  ↓
EKS
```

Symptoms:

```text
HTTP 500
```

Investigation:

```bash
kubectl rollout status \
  deployment/product-service \
  -n ecommerce
```

Then:

```bash
kubectl logs \
  deployment/product-service \
  -n ecommerce
```

Check:

```text
Git commit
CI run
ECR digest
GitOps commit
ArgoCD sync
Pod version
```

Rollback should be performed through GitOps.

---

# 64. Example Production Incident

## Incident ID

```text
INC-2026-PRODUCT-001
```

## Severity

```text
SEV-2
```

## Title

```text
Product API Latency Increased Due to Redis Failure
```

---

# 65. Customer Impact

Customers experienced:

- Slow product detail pages
- Increased product API latency
- Some 5xx responses

Orders and payments were not directly impacted.

---

# 66. Timeline

```text
14:00
Normal operation.

14:05
Redis latency begins increasing.

14:06
Product cache hit ratio falls from 95% to 20%.

14:07
PostgreSQL read traffic increases.

14:08
Hikari active connections increase.

14:09
Product API P95 increases from 140ms to 1.2s.

14:10
HPA increases product-service replicas.

14:11
Additional Pods create additional DB connections.

14:12
RDS connection utilization becomes high.

14:13
HTTP 5xx increases.

14:14
SEV-2 declared.

14:16
Platform team confirms Redis degradation.

14:18
Application team limits database fallback concurrency.

14:20
Noncritical catalog requests are rate limited.

14:22
Redis recovers.

14:24
Cache hit ratio returns above 90%.

14:26
RDS load returns to normal.

14:28
Product API latency returns to normal.
```

---

# 67. Root Cause

Primary failure:

```text
Redis degradation
```

Amplification:

```text
Redis degradation
      ↓
Cache misses
      ↓
PostgreSQL load
      ↓
Hikari connection pressure
      ↓
HPA scaling
      ↓
More database connections
      ↓
RDS pressure
      ↓
Product API latency
```

---

# 68. Five Whys

### Why did product APIs become slow?

PostgreSQL response time increased.

### Why?

Product requests bypassed Redis and hit PostgreSQL.

### Why?

Redis cache hit ratio dropped sharply.

### Why?

Redis experienced elevated latency.

### Why did this become a customer-impacting incident?

The database fallback was not sufficiently bounded.

---

# 69. Corrective Actions

| Action | Owner |
|---|---|
| Add cache stampede protection | Backend |
| Add bounded DB fallback | Backend |
| Improve Redis alerts | SRE |
| Review Hikari pool | Platform + Backend |
| Review HPA max replicas | Platform |
| Add Redis failure load test | Performance |
| Add product cache SLO | SRE |
| Add RDS connection alert | SRE |
| Add product incident runbook | SRE |

---

# 70. Production Troubleshooting Commands

## Pods

```bash
kubectl get pods \
  -n ecommerce \
  -l app=product-service
```

## Deployment

```bash
kubectl get deployment \
  -n ecommerce \
  product-service
```

## Logs

```bash
kubectl logs \
  -n ecommerce \
  deployment/product-service
```

## Previous logs

```bash
kubectl logs \
  -n ecommerce \
  <pod-name> \
  --previous
```

## HPA

```bash
kubectl get hpa \
  -n ecommerce \
  product-service
```

## Service

```bash
kubectl get svc \
  -n ecommerce \
  product-service
```

## EndpointSlices

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=product-service
```

## Events

```bash
kubectl get events \
  -n ecommerce \
  --sort-by=.lastTimestamp
```

## Resource usage

```bash
kubectl top pods \
  -n ecommerce
```

---

# 71. Security Checklist

- [ ] No secrets in Git
- [ ] No production `.env`
- [ ] GitHub OIDC
- [ ] Least-privilege IAM
- [ ] EKS Pod Identity
- [ ] NetworkPolicy
- [ ] Non-root container
- [ ] Minimal runtime image
- [ ] Trivy
- [ ] SonarQube
- [ ] Dependency scanning
- [ ] Secret scanning
- [ ] PostgreSQL private
- [ ] Redis private
- [ ] Kafka secured
- [ ] TLS enabled where required
- [ ] No credentials in logs
- [ ] Immutable ECR image
- [ ] Image provenance

---

# 72. Production Checklist

## Application

- [ ] Java 21
- [ ] Spring Boot
- [ ] Maven
- [ ] Spring Data JPA
- [ ] Validation
- [ ] Exception handling
- [ ] Unit tests
- [ ] Integration tests
- [ ] Actuator
- [ ] Graceful shutdown

## Database

- [ ] PostgreSQL
- [ ] RDS
- [ ] Flyway migrations
- [ ] Hikari pool
- [ ] Connection budget
- [ ] Backup
- [ ] PITR
- [ ] Private subnet
- [ ] Security group

## Redis

- [ ] Cache-aside
- [ ] TTL
- [ ] Cache invalidation
- [ ] Timeout
- [ ] Bounded retry
- [ ] Stampede protection
- [ ] Cache warming

## Kafka

- [ ] Event schema
- [ ] Event ID
- [ ] Version
- [ ] Correlation ID
- [ ] Idempotency
- [ ] Transactional outbox

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
- [ ] Docker build
- [ ] Trivy
- [ ] ECR
- [ ] GitHub OIDC
- [ ] Image digest

## GitOps

- [ ] Helm
- [ ] Dev values
- [ ] QA values
- [ ] Staging values
- [ ] Production values
- [ ] ArgoCD
- [ ] Approval
- [ ] Rollback

## Observability

- [ ] Prometheus
- [ ] Grafana
- [ ] ELK
- [ ] Correlation ID
- [ ] OpenTelemetry
- [ ] Jaeger
- [ ] Alerts
- [ ] SLO
- [ ] Runbook
- [ ] RCA process

---

# 73. Final Architecture

```text
                         Internet
                            |
                     CloudFront / WAF
                            |
                           ALB
                            |
                            ↓
                    product-service
                            |
             ┌──────────────┼──────────────┐
             │              │              │
             ↓              ↓              ↓
        PostgreSQL        Redis          Kafka
             │              │              │
             │              │              ├── Search
             │              │              ├── Catalog
             │              │              └── Reporting
             │              │
             │              │
             └──────────────┼──────────────┘
                            |
                       Product Domain


Application Metrics
        |
        ↓
   Prometheus
        |
        ↓
     Grafana


Application Logs
        |
        ↓
   Fluent Bit
        |
        ↓
     Logstash
        |
        ↓
 Elasticsearch
        |
        ↓
     Kibana


Application Traces
        |
        ↓
 OpenTelemetry
        |
        ↓
 OTel Collector
        |
        ↓
      Jaeger
```

---

# 74. Polyglot Microservice Architecture

Our Phase 19 architecture is intentionally polyglot.

```text
161 Auth
    Java + Spring Boot + Maven

162 User
    Python + FastAPI

163 Customer
    TypeScript + Node.js + NestJS

164 Catalog
    Go + Gin

165 Product
    Java + Spring Boot + Maven
```

The remaining services will continue using different languages where they
make sense.

The important DevOps lesson is:

```text
Application language/framework
            ↓
        changes

CI/CD pattern
            ↓
        standardized

Docker pattern
            ↓
        standardized

ECR
            ↓
        standardized

GitOps
            ↓
        standardized

ArgoCD
            ↓
        standardized

EKS
            ↓
        standardized

Observability
            ↓
        standardized
```

This is exactly why the later **Phase 20 — Platform Engineering** exists.

In Phase 20 we will take these repeated patterns and turn them into:

- Reusable Helm templates
- Reusable GitHub Actions
- Terraform modules
- Reusable CI workflows
- Security policy templates
- Monitoring templates
- Alert templates
- NetworkPolicy templates
- ServiceAccount templates
- GitOps generators
- A complete microservice golden path

# End of 165 — Product Service
```

### Phase 19 progress

```text
161 Auth Service            ✅ Java / Spring Boot / Maven
162 User Service            ✅ Python / FastAPI
163 Customer Service        ✅ TypeScript / Node.js / NestJS
164 Catalog Service         ✅ Go / Gin / Go Modules
165 Product Service         ✅ Java / Spring Boot / Maven

166 Inventory Service       ⏳ Go / Gin
167 Pricing Service         ⏳ Python / FastAPI
168 Promotion Service       ⏳ Node.js / TypeScript / NestJS
169 Cart Service            ⏳ Java / Spring Boot
170 Order Service           ⏳ Java / Spring Boot
171 Payment Service         ⏳ C# / ASP.NET Core
172 Fraud Service           ⏳ Python / FastAPI
173 Shipping Service        ⏳ Node.js / TypeScript / NestJS
174 Delivery Service        ⏳ Go / Gin
175 Notification Service    ⏳ Python / FastAPI
176 Search Service          ⏳ Java / Spring Boot
177 Recommendation Service  ⏳ Python / FastAPI
178 Review Service          ⏳ Ruby / Rails
179 Tax Service             ⏳ C# / ASP.NET Core
180 Invoice Service         ⏳ Node.js / TypeScript / NestJS
181 Reporting Service       ⏳ Python / FastAPI
```

**Next: `166-Inventory-Service.md` — Go + Gin, with PostgreSQL + Kafka and the concurrency/inventory-reservation design.**