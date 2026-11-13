# 171 — Payment Service

**File:** `171-Payment-Service.md`

```markdown
# 171 — Payment Service

## 1. Overview

The `payment-service` manages the payment lifecycle for the e-commerce
platform.

Payment is one of the most sensitive services in the architecture because it
handles financial transactions and communicates with external payment
providers.

The service must be designed around:

- Idempotency
- Payment state management
- Authorization
- Capture
- Refund
- Cancellation
- Webhook processing
- Duplicate protection
- Retry safety
- Reconciliation
- Auditability
- Fraud integration
- External provider failures
- Security
- Observability

Technology:

- Java 21
- Spring Boot
- Maven
- PostgreSQL
- Redis
- Kafka
- REST
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

# 2. Most Important Payment Principle

The payment service must never assume:

```text
HTTP 200 from payment provider
=
money successfully received
```

External payment providers can have:

- Timeouts
- Duplicate callbacks
- Delayed callbacks
- Network failures
- Partial failures
- Unknown transaction state
- Provider outages
- Retry behavior

Therefore payment processing must support:

```text
Idempotency
+
State machine
+
Webhook verification
+
Reconciliation
```

---

# 3. Payment Architecture

```text
                         Order Service
                              |
                              ↓
                       Payment Service
                      /       |       \
                     /        |        \
                    ↓         ↓         ↓
              PostgreSQL    Redis     Kafka
                    |
                    ↓
             Payment Provider
                    |
                    ↓
                 Webhook
                    |
                    ↓
             Payment Service
```

External payment provider:

```text
Payment Service
      |
      ↓
Payment Gateway
      |
      ↓
Bank / Card Network / UPI / Wallet
```

The actual gateway depends on the business and geography.

Examples can include:

- Stripe
- Adyen
- Razorpay
- PayPal
- Other approved payment providers

The service should hide provider-specific implementation behind a common
internal payment interface.

---

# 4. Why Payment Must Be Its Own Service

Payment has different requirements from normal CRUD services.

It requires:

- Strong auditability
- Idempotency
- External provider integration
- Secure credentials
- Webhook verification
- Reconciliation
- Financial reporting
- Refund workflows
- Strict state transitions

Therefore:

```text
Order Service
```

should not directly contain:

```text
Payment Gateway SDK
```

Instead:

```text
Order Service
      |
      ↓
Payment Service
      |
      ↓
Payment Provider
```

This provides a clear ownership boundary.

---

# 5. Payment Service Responsibilities

Payment Service owns:

- Payment
- Payment attempts
- Provider transaction ID
- Authorization state
- Capture state
- Refund state
- Provider response
- Webhook processing
- Payment events
- Idempotency
- Reconciliation metadata

It does not own:

- Order lifecycle
- Inventory
- Customer profile
- Product
- Cart
- Fraud rules

---

# 6. Payment State Machine

A payment should not simply have:

```text
SUCCESS
FAILED
```

A production state machine is more detailed.

Example:

```text
CREATED
   |
   ↓
PENDING
   |
   ↓
AUTHORIZED
   |
   ↓
CAPTURED
   |
   ↓
SETTLED
```

Failure states:

```text
FAILED
CANCELLED
EXPIRED
```

Refund:

```text
CAPTURED
   |
   ↓
REFUND_PENDING
   |
   ↓
REFUNDED
```

Partial refund:

```text
CAPTURED
   |
   ↓
PARTIALLY_REFUNDED
```

---

# 7. Payment State Transition Example

```text
Order
 |
 ↓
Payment Created
 |
 ↓
Provider Authorization
 |
 ├── SUCCESS
 │      |
 │      ↓
 │   AUTHORIZED
 │      |
 │      ↓
 │   CAPTURE
 │      |
 │      ↓
 │   CAPTURED
 │
 └── FAILURE
        |
        ↓
      FAILED
```

Not every payment method follows exactly the same authorization/capture
workflow.

The provider-specific adapter must map provider states to the internal state
machine.

---

# 8. Payment vs Authorization vs Capture

These are different concepts.

### Authorization

The provider confirms that the payment method can fund the transaction.

```text
AUTHORIZED
```

Money may not yet be captured.

### Capture

The payment is actually captured according to the provider's flow.

```text
CAPTURED
```

### Refund

Money is returned after capture.

```text
REFUNDED
```

For some payment methods, authorization and capture may happen as one
operation.

The internal service should still maintain explicit state semantics.

---

# 9. Repository Structure

```text
payment-service/
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/company/payment/
│   │   │
│   │   │       ├── PaymentServiceApplication.java
│   │   │
│   │   │       ├── controller/
│   │   │       │   ├── PaymentController.java
│   │   │       │   └── WebhookController.java
│   │   │
│   │   │       ├── service/
│   │   │       │   ├── PaymentService.java
│   │   │       │   ├── RefundService.java
│   │   │       │   └── ReconciliationService.java
│   │   │
│   │   │       ├── provider/
│   │   │       │   ├── PaymentProvider.java
│   │   │       │   ├── PaymentProviderClient.java
│   │   │       │   └── ProviderResponseMapper.java
│   │   │
│   │   │       ├── entity/
│   │   │       │   ├── Payment.java
│   │   │       │   ├── PaymentAttempt.java
│   │   │       │   ├── Refund.java
│   │   │       │   ├── WebhookEvent.java
│   │   │       │   └── IdempotencyRecord.java
│   │   │
│   │   │       ├── repository/
│   │   │       │   ├── PaymentRepository.java
│   │   │       │   ├── PaymentAttemptRepository.java
│   │   │       │   ├── RefundRepository.java
│   │   │       │   └── WebhookEventRepository.java
│   │   │
│   │   │       ├── dto/
│   │   │       │   ├── PaymentRequest.java
│   │   │       │   ├── PaymentResponse.java
│   │   │       │   └── RefundRequest.java
│   │   │
│   │   │       ├── messaging/
│   │   │       │   ├── PaymentEventPublisher.java
│   │   │       │   └── PaymentEventConsumer.java
│   │   │
│   │   │       ├── security/
│   │   │       │   └── WebhookSignatureVerifier.java
│   │   │       │
│   │   │       ├── exception/
│   │   │       │   ├── PaymentNotFoundException.java
│   │   │       │   ├── DuplicatePaymentException.java
│   │   │       │   └── GlobalExceptionHandler.java
│   │   │       │
│   │   │       └── config/
│   │   │           ├── RedisConfig.java
│   │   │           └── KafkaConfig.java
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
│   └── payment-service/
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
        payment-service
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

            <scope>runtime</scope>
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
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-test
            </artifactId>

            <scope>test</scope>
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

# 11. Application Configuration

## `application.yaml`

```yaml
server:

  port: 8080


spring:

  application:

    name: payment-service


  datasource:

    url: ${DB_URL:jdbc:postgresql://localhost:5432/payments}

    username: ${DB_USERNAME:postgres}

    password: ${DB_PASSWORD:postgres}

    hikari:

      maximum-pool-size: 10

      minimum-idle: 2

      connection-timeout: 3000

      validation-timeout: 1000

      idle-timeout: 600000

      max-lifetime: 1500000

      pool-name: payment-service-hikari


  jpa:

    hibernate:

      ddl-auto: update

    show-sql: false

    open-in-view: false


  data:

    redis:

      host: ${REDIS_HOST:localhost}

      port: ${REDIS_PORT:6379}

      timeout: 1000ms


  kafka:

    bootstrap-servers:
      ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}

    producer:

      acks: all

      enable-idempotence: true

      retries: 5


management:

  endpoints:

    web:

      exposure:

        include: health,info,prometheus

  endpoint:

    health:

      probes:

        enabled: true


payment:

  provider:

    base-url: ${PAYMENT_PROVIDER_BASE_URL:http://localhost:9000}

    connect-timeout-ms: 2000

    read-timeout-ms: 5000

  webhook:

    tolerance-seconds: 300

  idempotency:

    ttl-seconds: 86400


logging:

  level:

    root: INFO

    com.company.payment: INFO
```

---

# 12. Production Configuration

## `application-prod.yaml`

```yaml
server:

  port: 8080


spring:

  application:

    name: payment-service


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

      pool-name: payment-service-hikari


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

      enable-idempotence: true

      retries: 5


payment:

  provider:

    base-url: ${PAYMENT_PROVIDER_BASE_URL}

    connect-timeout-ms: 2000

    read-timeout-ms: 5000

  webhook:

    tolerance-seconds: 300

  idempotency:

    ttl-seconds: 86400
```

Provider API keys and webhook secrets must come from Secrets Manager through
External Secrets, not from this file.

---

# 13. Main Application

## `PaymentServiceApplication.java`

```java
package com.company.payment;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class PaymentServiceApplication {

    public static void main(String[] args) {

        SpringApplication.run(
            PaymentServiceApplication.class,
            args
        );
    }
}
```

---

# 14. Payment Entity

## `Payment.java`

```java
@Entity
@Table(
    name = "payments",
    indexes = {
        @Index(
            name = "idx_payments_order_id",
            columnList = "order_id"
        ),
        @Index(
            name = "idx_payments_customer_id",
            columnList = "customer_id"
        ),
        @Index(
            name = "idx_payments_status",
            columnList = "status"
        )
    }
)
public class Payment {

    @Id
    @GeneratedValue(
        strategy = GenerationType.IDENTITY
    )
    private Long id;

    @Column(
        name = "payment_reference",
        nullable = false,
        unique = true
    )
    private String paymentReference;

    @Column(
        name = "order_id",
        nullable = false
    )
    private Long orderId;

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
        nullable = false
    )
    private String provider;

    @Column(
        name = "provider_payment_id"
    )
    private String providerPaymentId;

    @Column(
        precision = 19,
        scale = 4,
        nullable = false
    )
    private BigDecimal amount;

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
}
```

---

# 15. Payment Attempt

A single payment can have multiple provider attempts.

Example:

```text
Payment
 |
 ├── Attempt 1 → timeout
 |
 ├── Attempt 2 → provider rejected
 |
 └── Attempt 3 → success
```

Therefore keep:

```text
payments
payment_attempts
```

separate.

Example fields:

```text
attempt_id
payment_id
provider
provider_request_id
provider_payment_id
status
request_time
response_time
error_code
```

---

# 16. Refund Entity

```java
@Entity
@Table(
    name = "refunds"
)
public class Refund {

    @Id
    @GeneratedValue(
        strategy = GenerationType.IDENTITY
    )
    private Long id;

    @Column(
        nullable = false,
        unique = true
    )
    private String refundReference;

    @Column(
        nullable = false
    )
    private Long paymentId;

    @Column(
        precision = 19,
        scale = 4,
        nullable = false
    )
    private BigDecimal amount;

    @Column(
        nullable = false
    )
    private String currency;

    @Column(
        nullable = false
    )
    private String status;

    private String providerRefundId;

    private Instant createdAt;

    private Instant updatedAt;
}
```

---

# 17. Webhook Event

External providers can send the same webhook multiple times.

Store the provider event ID.

```java
@Entity
@Table(
    name = "webhook_events",
    uniqueConstraints = {
        @UniqueConstraint(
            columnNames = {
                "provider",
                "provider_event_id"
            }
        )
    }
)
public class WebhookEvent {

    @Id
    @GeneratedValue(
        strategy = GenerationType.IDENTITY
    )
    private Long id;

    @Column(
        nullable = false
    )
    private String provider;

    @Column(
        name = "provider_event_id",
        nullable = false
    )
    private String providerEventId;

    @Column(
        nullable = false
    )
    private String eventType;

    @Column(
        nullable = false
    )
    private String status;

    @Column(
        nullable = false
    )
    private Instant receivedAt;

    private Instant processedAt;
}
```

---

# 18. Payment API

Create payment:

```text
POST /api/v1/payments
```

Get payment:

```text
GET /api/v1/payments/{paymentReference}
```

Authorize:

```text
POST /api/v1/payments/{paymentReference}/authorize
```

Capture:

```text
POST /api/v1/payments/{paymentReference}/capture
```

Refund:

```text
POST /api/v1/payments/{paymentReference}/refund
```

Webhook:

```text
POST /api/v1/payments/webhooks/{provider}
```

---

# 19. Create Payment Request

```json
{
  "orderId": 100001,
  "customerId": "C1001",
  "amount": "21237.64",
  "currency": "INR",
  "paymentMethod": "CARD"
}
```

The service must validate that:

```text
order amount
=
payment amount
```

and should not blindly trust an amount supplied by an untrusted client.

In a production architecture, the payment amount should normally originate
from the authoritative order state.

---

# 20. Payment Response

```json
{
  "paymentReference": "PAY-2026-000001",
  "orderId": 100001,
  "status": "PENDING",
  "amount": "21237.64",
  "currency": "INR",
  "provider": "example-provider"
}
```

Never return sensitive payment information.

---

# 21. Payment Controller

```java
@RestController
@RequestMapping("/api/v1/payments")
public class PaymentController {

    private final PaymentService paymentService;

    public PaymentController(
        PaymentService paymentService
    ) {
        this.paymentService = paymentService;
    }

    @PostMapping
    public ResponseEntity<PaymentResponse>
    createPayment(

        @RequestHeader("Idempotency-Key")
        String idempotencyKey,

        @Valid
        @RequestBody
        PaymentRequest request
    ) {

        return ResponseEntity
            .status(HttpStatus.CREATED)
            .body(
                paymentService.createPayment(
                    request,
                    idempotencyKey
                )
            );
    }

    @GetMapping("/{paymentReference}")
    public ResponseEntity<PaymentResponse>
    getPayment(
        @PathVariable
        String paymentReference
    ) {

        return ResponseEntity.ok(
            paymentService.getPayment(
                paymentReference
            )
        );
    }
}
```

---

# 22. Payment Provider Abstraction

Do not tightly couple the business service to one provider SDK.

Define:

```java
public interface PaymentProvider {

    AuthorizationResult authorize(
        PaymentRequest request,
        String idempotencyKey
    );

    CaptureResult capture(
        String providerPaymentId,
        String idempotencyKey
    );

    RefundResult refund(
        String providerPaymentId,
        BigDecimal amount,
        String idempotencyKey
    );

    boolean verifyWebhook(
        String payload,
        String signature
    );
}
```

Then implement:

```text
StripePaymentProvider
RazorpayPaymentProvider
AdyenPaymentProvider
...
```

The exact provider depends on the project/business.

---

# 23. Why Provider Abstraction Matters

Without abstraction:

```text
PaymentService
      |
      ↓
Razorpay SDK everywhere
```

Changing providers becomes expensive.

With abstraction:

```text
PaymentService
      |
      ↓
PaymentProvider
   /          \
  ↓            ↓
Provider A   Provider B
```

This is particularly useful for:

- Provider migration
- Multi-provider routing
- Disaster recovery
- Regional payment methods
- Cost optimization

---

# 24. Payment Idempotency

This is one of the most important parts of the service.

Suppose:

```text
POST /payments
Idempotency-Key: PAYREQ-123
```

Provider receives request.

Payment succeeds.

But the application never receives the response because of a network timeout.

The application retries.

If the retry creates a second provider transaction:

```text
₹20,000
+
₹20,000
=
₹40,000 charged
```

This is unacceptable.

Therefore use idempotency at two levels:

```text
Client
  ↓
Payment Service idempotency

Payment Service
  ↓
Payment Provider idempotency
```

---

# 25. Payment Idempotency Table

```sql
CREATE TABLE idempotency_records (

    id BIGSERIAL PRIMARY KEY,

    idempotency_key VARCHAR(200) NOT NULL,

    operation VARCHAR(100) NOT NULL,

    request_hash VARCHAR(128) NOT NULL,

    payment_reference VARCHAR(100),

    response_payload TEXT,

    status VARCHAR(30) NOT NULL,

    created_at TIMESTAMP NOT NULL,

    expires_at TIMESTAMP NOT NULL,

    CONSTRAINT uq_payment_idempotency
        UNIQUE (
            idempotency_key,
            operation
        )
);
```

A key must not silently be reused for a different request payload.

---

# 26. Payment Database Migration

## `V1__create_payments.sql`

```sql
CREATE TABLE payments (

    id BIGSERIAL PRIMARY KEY,

    payment_reference VARCHAR(100)
        NOT NULL UNIQUE,

    order_id BIGINT NOT NULL,

    customer_id VARCHAR(100)
        NOT NULL,

    status VARCHAR(50)
        NOT NULL,

    provider VARCHAR(100)
        NOT NULL,

    provider_payment_id VARCHAR(200),

    amount NUMERIC(19,4)
        NOT NULL,

    currency VARCHAR(3)
        NOT NULL,

    created_at TIMESTAMP
        NOT NULL,

    updated_at TIMESTAMP
        NOT NULL,

    version BIGINT
        NOT NULL DEFAULT 0
);

CREATE INDEX idx_payments_order_id
    ON payments(order_id);

CREATE INDEX idx_payments_customer_id
    ON payments(customer_id);

CREATE INDEX idx_payments_status
    ON payments(status);
```

---

# 27. Payment Attempts Migration

```sql
CREATE TABLE payment_attempts (

    id BIGSERIAL PRIMARY KEY,

    payment_id BIGINT NOT NULL,

    provider VARCHAR(100) NOT NULL,

    provider_request_id VARCHAR(200),

    provider_payment_id VARCHAR(200),

    status VARCHAR(50) NOT NULL,

    error_code VARCHAR(100),

    error_message VARCHAR(500),

    request_time TIMESTAMP NOT NULL,

    response_time TIMESTAMP,

    CONSTRAINT fk_payment_attempt
        FOREIGN KEY(payment_id)
        REFERENCES payments(id)
);
```

Do not store sensitive card data in this table.

---

# 28. Refund Migration

```sql
CREATE TABLE refunds (

    id BIGSERIAL PRIMARY KEY,

    refund_reference VARCHAR(100)
        NOT NULL UNIQUE,

    payment_id BIGINT NOT NULL,

    amount NUMERIC(19,4)
        NOT NULL,

    currency VARCHAR(3)
        NOT NULL,

    status VARCHAR(50)
        NOT NULL,

    provider_refund_id VARCHAR(200),

    created_at TIMESTAMP NOT NULL,

    updated_at TIMESTAMP NOT NULL,

    CONSTRAINT fk_refund_payment
        FOREIGN KEY(payment_id)
        REFERENCES payments(id)
);
```

---

# 29. Payment Webhook

External provider:

```text
Provider
   |
   ↓
POST /payments/webhooks/provider
```

The service must:

1. Receive raw payload
2. Verify signature
3. Validate timestamp/replay window
4. Identify provider event ID
5. Check duplicate event
6. Persist webhook event
7. Map provider status
8. Update payment state
9. Publish internal event
10. Return success

---

# 30. Webhook Security

Never trust:

```text
payment status = SUCCESS
```

just because the HTTP request reached your endpoint.

Verify:

```text
provider signature
+
timestamp
+
event ID
```

Example:

```text
Webhook
   |
   ↓
Signature Verification
   |
   ├── Invalid → reject
   |
   └── Valid
         |
         ↓
     Process event
```

The exact signature algorithm depends on the provider.

---

# 31. Webhook Controller

```java
@RestController
@RequestMapping("/api/v1/payments/webhooks")
public class WebhookController {

    private final PaymentWebhookService webhookService;

    public WebhookController(
        PaymentWebhookService webhookService
    ) {
        this.webhookService = webhookService;
    }

    @PostMapping("/{provider}")
    public ResponseEntity<Void> receiveWebhook(

        @PathVariable String provider,

        @RequestHeader(
            "X-Provider-Signature"
        )
        String signature,

        @RequestBody String payload
    ) {

        webhookService.process(
            provider,
            signature,
            payload
        );

        return ResponseEntity.ok().build();
    }
}
```

In a real provider integration, use the exact provider-required headers and
raw-body verification procedure.

---

# 32. Webhook Idempotency

Provider sends:

```text
event_123
```

First request:

```text
SUCCESS
```

Second request:

```text
event_123
```

Database constraint:

```text
(provider, provider_event_id)
```

prevents duplicate processing.

This is essential because webhook providers commonly retry delivery.

---

# 33. Unknown Payment State

Consider:

```text
Payment Service → Provider
```

Provider processes payment.

Network timeout occurs.

Payment Service receives:

```text
TIMEOUT
```

It does NOT know whether payment succeeded.

Wrong behavior:

```text
TIMEOUT
 ↓
immediately mark FAILED
```

Correct behavior:

```text
TIMEOUT
 ↓
UNKNOWN/PENDING
 ↓
provider status lookup
OR
wait for webhook
 ↓
determine authoritative state
```

This prevents accidental duplicate charges.

---

# 34. Payment State Machine

```text
             ┌───────────────┐
             │               ↓
CREATED → PENDING → AUTHORIZED → CAPTURED
             │
             ↓
          FAILED

CAPTURED
   |
   ↓
REFUND_PENDING
   |
   ↓
REFUNDED
```

Unknown provider responses should not be converted into `FAILED` without
checking whether the provider actually processed the transaction.

---

# 35. Payment Retry Rules

Not every error should be retried.

Retryable examples:

```text
Network timeout
Temporary provider unavailable
Connection reset
HTTP 429
Some HTTP 5xx
```

Potentially non-retryable:

```text
Invalid payment method
Insufficient funds
Invalid request
Expired payment method
Authentication failure
```

Provider-specific semantics must be followed.

---

# 36. Retry Strategy

Use:

```text
Exponential Backoff
+
Jitter
+
Maximum Attempts
```

Example:

```text
Attempt 1 → immediate
Attempt 2 → 1s
Attempt 3 → 2s
Attempt 4 → 4s
```

with jitter.

Never implement:

```text
while(true) retry
```

because payment retry storms can amplify provider outages.

---

# 37. Payment Circuit Breaker

If provider is failing:

```text
Payment Service
       |
       ↓
Circuit Breaker
       |
       ├── CLOSED → provider
       |
       ├── OPEN → fail fast
       |
       └── HALF OPEN → controlled test
```

Benefits:

- Protect application threads
- Prevent provider overload
- Prevent retry storm
- Improve failure behavior

---

# 38. Payment Provider Rate Limits

External provider may support only:

```text
X requests/sec
```

Scaling payment Pods from:

```text
3 → 30
```

does not mean:

```text
provider capacity
=
10×
```

The payment service must implement:

- Rate limiting
- Concurrency control
- Backpressure
- Retry budgets

---

# 39. Redis Usage

Redis can store:

```text
payment:idempotency:<key>
payment:rate-limit:<provider>
payment:temporary-state:<payment-reference>
```

Redis should not be the durable source of truth.

PostgreSQL remains authoritative.

---

# 40. Payment Event

Example:

```json
{
  "eventId": "evt-pay-1001",
  "eventType": "PaymentAuthorized",
  "version": 1,
  "occurredAt": "2026-10-05T12:00:00Z",
  "source": "payment-service",
  "correlationId": "corr-order-1001",
  "aggregateId": "PAY-1001",
  "payload": {
    "paymentReference": "PAY-1001",
    "orderId": 100001,
    "amount": "21237.64",
    "currency": "INR"
  }
}
```

---

# 41. Kafka Topics

Example:

```text
ecommerce.payment.events
```

Events:

```text
PaymentCreated
PaymentAuthorized
PaymentCaptureRequested
PaymentCaptured
PaymentFailed
PaymentRefundRequested
PaymentRefunded
```

The exact event taxonomy should be version-controlled.

---

# 42. Payment Consumer

Payment Service may consume:

```text
OrderCreated
PaymentCaptureRequested
RefundRequested
OrderCancelled
```

For example:

```text
OrderConfirmed
      |
      ↓
PaymentCaptureRequested
      |
      ↓
Payment Service
      |
      ↓
Provider Capture
      |
      ↓
PaymentCaptured
```

---

# 43. Payment Capture

Authorization:

```text
AUTHORIZED
```

Capture:

```text
POST /payments/{id}/capture
```

Before capture:

```text
validate payment state
validate amount
validate idempotency
validate provider transaction
```

Then:

```text
Provider
   ↓
Capture
   ↓
CAPTURED
```

---

# 44. Refund Flow

```text
Order
 |
 ↓
Refund Requested
 |
 ↓
Payment Service
 |
 ↓
Provider
 |
 ├── Success → REFUNDED
 |
 └── Failure → REFUND_PENDING
```

Never automatically mark:

```text
REFUNDED
```

until the provider confirms the refund.

---

# 45. Partial Refund

Example:

Original:

```text
₹10,000
```

Refund:

```text
₹2,000
```

Remaining refundable amount:

```text
₹8,000
```

The service must prevent:

```text
Refunded total > Captured amount
```

Database-level and application-level validation should both be considered.

---

# 46. Reconciliation

This is one of the most important production capabilities.

Sometimes:

```text
Our DB:
PAY-1001 = PENDING
```

Provider:

```text
PAY-1001 = CAPTURED
```

This discrepancy must be detected.

Reconciliation process:

```text
Provider transactions
        |
        ↓
Reconciliation job
        |
        ↓
Compare with internal DB
        |
        ├── Match
        |
        └── Mismatch
              |
              ↓
          Investigate
```

---

# 47. Reconciliation Job

Example:

```text
Every 15 minutes
```

but actual cadence depends on provider/business requirements.

Query:

```text
PENDING
UNKNOWN
RECENTLY_FAILED
```

payments.

Then ask provider for authoritative status.

Example:

```text
Internal:
PAY-1001 = PENDING

Provider:
PAY-1001 = CAPTURED

Reconciliation:
PAY-1001 → CAPTURED
```

Publish:

```text
PaymentCaptured
```

if appropriate and safe.

---

# 48. Reconciliation Safety

Never blindly overwrite states.

Example:

```text
REFUNDED
```

must not become:

```text
CAPTURED
```

without understanding the provider state and transaction history.

State transitions should be validated against the payment state machine.

---

# 49. Dockerfile

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

# 50. Local Docker Compose

```yaml
services:

  postgres:

    image: postgres:16

    container_name: payment-postgres

    environment:

      POSTGRES_DB: payments

      POSTGRES_USER: postgres

      POSTGRES_PASSWORD: postgres

    ports:

      - "5432:5432"

    volumes:

      - postgres-data:/var/lib/postgresql/data


  redis:

    image: redis:7

    container_name: payment-redis

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: payment-kafka

    ports:

      - "9092:9092"

    # Development-only configuration.
    # Pin a tested Kafka version for the actual project.


  payment-service:

    build:

      context: .

      dockerfile: Dockerfile

    depends_on:

      - postgres
      - redis
      - kafka

    environment:

      SPRING_PROFILES_ACTIVE: local

      DB_URL: jdbc:postgresql://postgres:5432/payments

      DB_USERNAME: postgres

      DB_PASSWORD: postgres

      REDIS_HOST: redis

      REDIS_PORT: 6379

      KAFKA_BOOTSTRAP_SERVERS: kafka:9092

      PAYMENT_PROVIDER_BASE_URL: http://mock-provider:9000

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

For local development, a mock provider should be used rather than sending
real transactions.

---

# 51. Kubernetes Deployment

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: payment-service

  labels:

    app: payment-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: payment-service

  template:

    metadata:

      labels:

        app: payment-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: payment-service

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

                name: payment-service-config

            - secretRef:

                name: payment-service-secrets

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

# 52. Service

```yaml
apiVersion: v1

kind: Service

metadata:

  name: payment-service

  labels:

    app: payment-service

spec:

  type: ClusterIP

  selector:

    app: payment-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 53. ConfigMap

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: payment-service-config

data:

  REDIS_HOST: redis.ecommerce.svc.cluster.local

  REDIS_PORT: "6379"

  KAFKA_BOOTSTRAP_SERVERS: kafka.ecommerce.svc.cluster.local:9092

  PAYMENT_PROVIDER_BASE_URL: https://provider.example.com
```

Do not put API keys or webhook secrets into this ConfigMap.

---

# 54. ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: payment-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: payment-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DB_URL

      remoteRef:

        key: ecommerce/prod/payment-service

        property: DB_URL

    - secretKey: DB_USERNAME

      remoteRef:

        key: ecommerce/prod/payment-service

        property: DB_USERNAME

    - secretKey: DB_PASSWORD

      remoteRef:

        key: ecommerce/prod/payment-service

        property: DB_PASSWORD

    - secretKey: PAYMENT_PROVIDER_API_KEY

      remoteRef:

        key: ecommerce/prod/payment-service

        property: PAYMENT_PROVIDER_API_KEY

    - secretKey: PAYMENT_WEBHOOK_SECRET

      remoteRef:

        key: ecommerce/prod/payment-service

        property: PAYMENT_WEBHOOK_SECRET
```

---

# 55. ServiceAccount and AWS Access

Payment service should not receive broad AWS permissions.

If it needs SQS:

```text
Payment Pod
    ↓
EKS Pod Identity
    ↓
payment-service IAM role
    ↓
specific SQS queue
```

Example permissions should be limited to:

```text
sqs:SendMessage
sqs:ReceiveMessage
sqs:DeleteMessage
```

on the specific required queue.

Do not grant:

```text
AdministratorAccess
```

to the payment workload.

---

# 56. NetworkPolicy

Payment has stricter network requirements.

Conceptually:

```text
Ingress:
  Order Service
  approved internal callers

Egress:
  PostgreSQL
  Redis
  Kafka
  Payment Provider
  DNS
```

The external payment provider connection should be explicitly allowed through
the approved egress architecture.

Do not simply allow:

```text
0.0.0.0/0
```

from the application Pod if the platform's network architecture can enforce
more restrictive egress.

---

# 57. HPA

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: payment-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: payment-service

  minReplicas: 3

  maxReplicas: 15

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

Payment scaling must consider provider rate limits.

More Pods can create more concurrent requests to the external provider.

Therefore:

```text
HPA
+
provider concurrency limit
+
rate limiter
```

must be designed together.

---

# 58. PDB

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: payment-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: payment-service
```

---

# 59. ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: payment-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: payment-service

  endpoints:

    - port: http

      path: /actuator/prometheus

      interval: 15s
```

---

# 60. Payment Metrics

Technical:

```text
payment_requests_total

payment_request_duration_seconds

payment_provider_latency_seconds

payment_provider_errors_total

payment_provider_timeouts_total

payment_webhook_total

payment_webhook_failures_total

payment_reconciliation_mismatch_total
```

Business:

```text
payments_created_total

payments_authorized_total

payments_captured_total

payments_failed_total

refunds_requested_total

refunds_completed_total

payment_success_rate
```

Security:

```text
invalid_webhook_signature_total

duplicate_webhook_total

invalid_payment_request_total
```

---

# 61. Grafana Dashboard

```text
Payment Service
│
├── Request Rate
├── P95 / P99
├── HTTP 5xx
│
├── Payment Success Rate
├── Payment Failure Rate
│
├── Provider
│   ├── Latency
│   ├── 4xx
│   ├── 5xx
│   ├── Timeouts
│   └── Rate Limits
│
├── Webhooks
│   ├── Received
│   ├── Invalid
│   ├── Duplicate
│   └── Processing Latency
│
├── Reconciliation
│   ├── Pending
│   └── Mismatches
│
├── PostgreSQL
│   ├── Connections
│   └── DB Load
│
└── Kubernetes
    ├── Pods
    ├── CPU
    └── Memory
```

---

# 62. Logging

Example:

```json
{
  "@timestamp": "2026-10-05T12:00:00Z",
  "level": "INFO",
  "service": "payment-service",
  "environment": "prod",
  "correlationId": "corr-order-1001",
  "paymentReference": "PAY-1001",
  "orderId": "100001",
  "provider": "example-provider",
  "operation": "AUTHORIZE",
  "status": "AUTHORIZED",
  "message": "Payment authorized"
}
```

Never log:

```text
card number
CVV
PIN
bank credentials
API keys
webhook secret
access tokens
full payment method details
```

Payment logging must be designed around data minimization and the applicable
compliance requirements.

---

# 63. Distributed Tracing

Example:

```text
Order Service
      |
      ↓
Payment Service
      |
      ↓
Provider API
```

Trace:

```text
traceId
 |
 ├── order-service
 |
 ├── payment-service
 |
 └── provider HTTP request
```

The provider request should be correlated using safe internal identifiers.

Do not put sensitive payment information into trace attributes.

---

# 64. Payment SLO

Initial targets:

Availability:

```text
99.95%
```

Payment API P95:

```text
< 500ms
```

Provider timeout rate:

```text
< defined provider-specific threshold
```

Webhook processing:

```text
99.9% within agreed processing window
```

Reconciliation mismatch:

```text
0 unresolved critical mismatches
```

The final SLOs must be based on payment-provider contracts and business
requirements.

---

# 65. Critical Alerts

Alert on:

```text
High Payment Failure Rate

High Provider Timeout Rate

High Provider 5xx

High Webhook Failure Rate

Invalid Webhook Signatures

Payment Reconciliation Mismatch

Large PENDING Payment Backlog

Payment API P95

Payment API 5xx

Hikari Connection Pressure

Kafka Consumer Lag

Redis Failure
```

A particularly important alert is:

```text
Payment succeeded at provider
but internal state remains pending
```

because it represents a financial reconciliation risk.

---

# 66. Payment Failure Scenario — Provider Timeout

```text
Payment Service
       |
       ↓
Provider
       |
       X
    Timeout
```

Do NOT immediately do:

```text
mark FAILED
retry blindly
```

Instead:

```text
TIMEOUT
   ↓
UNKNOWN/PENDING
   ↓
provider status query
or webhook
   ↓
authoritative state
```

This avoids duplicate charges.

---

# 67. Payment Failure Scenario — Duplicate Request

```text
Request 1
Idempotency-Key = ABC
       ↓
Payment succeeds
       ↓
Network timeout

Request 2
Idempotency-Key = ABC
       ↓
Existing payment returned
```

Result:

```text
ONE PAYMENT
```

---

# 68. Payment Failure Scenario — Duplicate Webhook

Provider:

```text
PaymentCaptured
event = EVT100
```

sends:

```text
EVT100
EVT100
EVT100
```

The database unique constraint:

```text
provider + provider_event_id
```

ensures one logical processing record.

---

# 69. Payment Failure Scenario — Provider Outage

```text
Provider
   ↓
5xx
   ↓
Payment Service
```

Circuit breaker:

```text
CLOSED
   ↓
errors increase
   ↓
OPEN
   ↓
fail fast
```

This prevents:

```text
Payment Pods
   ↓
retry storm
   ↓
Provider
   ↓
more failures
```

---

# 70. Payment Failure Scenario — PostgreSQL Outage

Payment state must remain durable.

If PostgreSQL is unavailable:

```text
Payment request
     ↓
Cannot safely persist state
     ↓
Do not claim success
```

The service should fail safely.

If an external provider was already contacted before the DB failure, the
payment may enter an uncertain state.

That transaction must later be reconciled.

---

# 71. Payment Failure Scenario — Webhook Endpoint Attack

Attacker sends:

```text
POST /webhooks/provider

status = CAPTURED
```

Without signature verification:

```text
payment becomes CAPTURED
```

This is catastrophic.

With verification:

```text
Invalid signature
      ↓
Reject
      ↓
Alert
```

Therefore webhook signature validation is a security boundary.

---

# 72. Payment Reconciliation Architecture

```text
                Payment Provider
                       |
                       ↓
              Provider Transactions
                       |
                       ↓
                Reconciliation
                       |
               ┌───────┴───────┐
               ↓               ↓
           Match            Mismatch
               |               |
             Done          Investigation
                               |
                               ↓
                         State Correction
                               |
                               ↓
                           Audit Event
```

---

# 73. Reconciliation Frequency

Example:

```text
Near-real-time:
Webhook processing

Periodic:
Pending/unknown payment reconciliation

Daily:
Financial reconciliation/reporting
```

Exact schedules depend on provider SLAs and finance requirements.

---

# 74. Financial Audit

Important fields:

```text
paymentReference
orderId
providerPaymentId
providerEventId
amount
currency
status
createdAt
authorizedAt
capturedAt
refundedAt
provider
```

Do not modify historical records destructively.

Use state history/audit records.

---

# 75. Payment Database Connection Planning

Suppose:

```text
10 Pods
×
10 Hikari connections
=
100 potential connections
```

During deployment:

```text
old Pods
+
new Pods
```

can temporarily increase this.

During HPA:

```text
3 → 15 Pods
```

potential connections become:

```text
15 × 10 = 150
```

Payment service must share the overall RDS connection budget with other
services.

---

# 76. CI Pipeline

```yaml
name: Payment Service CI

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

  ECR_REPOSITORY: payment-service


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


      - name: Test

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

# 77. SonarQube

Payment code requires strict quality controls.

Recommended pipeline:

```text
Checkout
 ↓
Maven compile
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
Secret scan
 ↓
Trivy filesystem
 ↓
Docker build
 ↓
Trivy image
 ↓
ECR
```

SonarQube should be centrally operated rather than installed inside payment
service Pods.

---

# 78. GitOps

Application repository:

```text
company-ecommerce/payment-service
```

GitOps:

```text
company-ecommerce/gitops
```

Example:

```text
gitops/
└── apps/
    └── payment-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Promotion:

```text
Build
 ↓
ECR
 ↓
GitOps PR
 ↓
Review
 ↓
Merge
 ↓
ArgoCD
 ↓
EKS
```

---

# 79. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: payment-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/payment-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

Production payment deployments should generally have stronger promotion
controls than ordinary internal services.

---

# 80. Payment Deployment Strategy

Recommended:

```text
DEV
 ↓
QA
 ↓
STAGING
 ↓
CANARY
 ↓
PRODUCTION
```

Canary metrics:

```text
Payment success rate
Payment failure rate
Provider timeout
Provider 5xx
Webhook processing
P95 latency
Duplicate payment attempts
```

---

# 81. Payment Rollback Warning

Application rollback:

```text
Version 1.2
 ↓
Version 1.1
```

is not the same as:

```text
Payment transaction rollback
```

A payment already captured at the provider cannot be undone by Kubernetes
rollback.

If a deployment causes financial problems:

```text
Stop deployment
 ↓
Contain
 ↓
Protect payment state
 ↓
Investigate provider transactions
 ↓
Reconcile
 ↓
Refund if required
```

Never treat:

```text
kubectl rollout undo
```

as a financial rollback mechanism.

---

# 82. Example Production Incident

## Incident ID

```text
INC-2026-PAYMENT-001
```

## Severity

```text
SEV-1
```

## Title

```text
Payment Provider Timeout Created Unknown Payment States
```

## Date

```text
2026-10-05
```

---

# 83. Customer Impact

Customers experienced:

- Checkout delays
- Some payment attempts showed pending
- Some customers retried payments
- Payment support tickets increased

No confirmed widespread duplicate charging occurred.

---

# 84. Timeline

```text
18:00
Payment service healthy.

18:05
Payment provider latency increases.

18:06
Provider timeout rate increases.

18:07
Payment API P95 increases.

18:08
Customers begin seeing pending payments.

18:09
Some clients retry payment requests.

18:10
Idempotency protection prevents duplicate internal payments.

18:11
SEV-1 declared.

18:13
Platform team confirms network path is healthy.

18:15
Payment team confirms provider-side degradation.

18:17
Circuit breaker opens for affected provider operations.

18:20
New payment attempts begin failing fast.

18:22
Reconciliation identifies unknown transactions.

18:25
Provider status API confirms several payments succeeded.

18:27
Internal payment states updated.

18:30
Provider latency begins recovering.

18:33
Circuit breaker moves toward normal operation.

18:36
Payment success rate returns to baseline.

18:40
Incident resolved.
```

---

# 85. Root Cause

The external payment provider experienced elevated latency and intermittent
timeouts.

The payment service could not determine the outcome of some provider
requests.

The service correctly retained those transactions as uncertain/pending rather
than immediately marking them failed.

The reconciliation workflow later determined the provider's authoritative
state.

---

# 86. Amplification Risk

The major amplification risk was:

```text
Provider timeout
    ↓
Customer retry
    ↓
More payment requests
    ↓
More provider load
    ↓
More timeouts
```

Idempotency and circuit breaking prevented this from becoming a much larger
incident.

---

# 87. What Went Well

- Payment idempotency worked
- Provider webhook verification worked
- Circuit breaker reduced retry pressure
- Reconciliation identified unknown payments
- No confirmed duplicate payment storm
- Observability showed provider latency
- Correlation IDs connected order and payment requests

---

# 88. What Went Wrong

- Provider degradation was detected after customer latency increased
- Pending-payment dashboard needed improvement
- Provider dependency alerting was not sufficiently early
- Customer-facing pending-payment messaging could be clearer

---

# 89. Corrective Actions

| Action | Owner |
|---|---|
| Improve provider latency alerts | SRE |
| Add pending payment dashboard | SRE |
| Improve circuit breaker tuning | Backend |
| Add provider dependency SLO | SRE |
| Increase reconciliation coverage | Payment |
| Add provider failure load tests | Performance |
| Improve customer pending-payment UX | Frontend |
| Test duplicate webhook handling | QA |
| Test unknown payment state | QA |
| Document provider incident runbook | SRE |

---

# 90. Five Whys

### Why did customers see pending payments?

The payment provider timed out.

### Why did the provider timeout?

The external provider experienced elevated latency.

### Why didn't the service immediately mark payments failed?

Because the provider might have processed the payment despite the timeout.

### Why is this important?

Marking a successful external payment as failed can cause duplicate customer
payment attempts.

### How was the problem safely resolved?

Idempotency, webhook processing, provider status lookup, and reconciliation
established the authoritative payment state.

---

# 91. Security Incident Scenario

Suppose:

```text
PAYMENT_PROVIDER_API_KEY
```

is leaked.

Immediate response:

```text
Contain
 ↓
Revoke old key
 ↓
Generate new provider credential
 ↓
Store in Secrets Manager
 ↓
External Secrets refresh
 ↓
Restart/reload payment service
 ↓
Verify provider connectivity
 ↓
Audit provider usage
```

Do not simply delete the Kubernetes Secret and assume the incident is
resolved.

The provider credential itself must be revoked/rotated.

---

# 92. Payment Security Checklist

```text
[ ] No card data stored
[ ] No CVV stored
[ ] No PIN stored
[ ] No secrets in Git
[ ] Secrets Manager
[ ] External Secrets
[ ] Least privilege IAM
[ ] EKS Pod Identity
[ ] TLS to provider
[ ] Webhook signature verification
[ ] Replay protection
[ ] Idempotency
[ ] Audit logging
[ ] CloudTrail
[ ] NetworkPolicy
[ ] Trivy
[ ] SonarQube
[ ] Dependency scanning
[ ] Secret scanning
[ ] Non-root container
[ ] Read-only filesystem
```

---

# 93. Important PCI Boundary

The architecture should minimize the payment service's exposure to raw card
data.

Preferred architecture:

```text
Customer
   |
   ↓
Provider-hosted payment UI/tokenization
   |
   ↓
Payment token
   |
   ↓
Payment Service
   |
   ↓
Provider
```

rather than:

```text
Customer
   |
   ↓
Our backend
   |
   ↓
Raw card number
CVV
PIN
```

The exact PCI scope depends on the chosen provider and integration model.

Never store raw card data unless there is a formally approved architecture,
legal basis, security controls, and compliance program for doing so.

---

# 94. Payment Capacity Planning

Suppose:

```text
Payment Pods = 10
Hikari = 10
```

Potential DB connections:

```text
10 × 10
=
100
```

External provider capacity might be:

```text
Provider = 500 requests/sec
```

Application capacity therefore cannot simply be:

```text
10 Pods × 100 RPS
=
1,000 RPS
```

because:

```text
RDS
+
Provider
+
Redis
+
Kafka
+
CPU
+
Network
```

all constrain the system.

---

# 95. Load Testing

Test:

```text
100 payment requests/sec
500 payment requests/sec
```

only as controlled staging/performance scenarios.

Measure:

```text
Payment success rate
Provider latency
P50
P95
P99
RDS connections
Redis latency
Kafka lag
CPU
Memory
Pod replicas
```

Also simulate:

```text
Provider timeout
Provider 429
Provider 500
Webhook duplicate
Webhook delay
Webhook invalid signature
Database failure
Redis failure
Kafka failure
```

---

# 96. Payment Failure Injection

Example:

```text
Provider latency
=
5 seconds
```

Expected:

```text
Payment timeout
 ↓
Pending/unknown state
 ↓
No duplicate charge
 ↓
Reconciliation
```

Another test:

```text
Provider returns 429
```

Expected:

```text
Rate limiting
+
backoff
+
circuit protection
```

---

# 97. Production Runbook

When payment failure alert fires:

```text
1. Confirm customer impact

2. Check payment success rate

3. Check payment P95/P99

4. Check provider latency

5. Check provider 4xx/5xx

6. Check provider rate limits

7. Check webhook failures

8. Check pending/unknown payments

9. Check reconciliation mismatches

10. Check recent deployment

11. Check Redis

12. Check PostgreSQL

13. Check Kafka

14. Check circuit breaker

15. Stop retry amplification

16. Protect customers from duplicate payment

17. Reconcile unknown payments

18. Verify payment state

19. Validate order state

20. Create RCA
```

---

# 98. Final Payment Architecture

```text
                              Customer
                                  |
                                  ↓
                              Checkout
                                  |
                                  ↓
                             Order Service
                                  |
                                  ↓
                            Payment Service
                         /        |        \
                        /         |         \
                       ↓          ↓          ↓
                 PostgreSQL     Redis       Kafka
                       |
                       ↓
                Payment State
                       |
                       ↓
              Payment Provider
                /           \
               ↓             ↓
          Provider API     Webhook
               |             |
               └──────┬──────┘
                      ↓
               Payment Service
                      |
                      ↓
                Reconciliation
                      |
                      ↓
                Payment Events
                      |
              ┌───────┼────────┐
              ↓       ↓        ↓
           Order   Reporting  Notification
```

---

# 99. Final Production Principles

The Payment Service follows these principles:

1. Never trust client payment status.
2. Never assume provider timeout means payment failure.
3. Use provider idempotency.
4. Use application idempotency.
5. Process webhooks idempotently.
6. Verify webhook signatures.
7. Protect against replay.
8. Keep payment state durable in PostgreSQL.
9. Use Redis only for optimization/temporary state.
10. Use Kafka for asynchronous payment events.
11. Use transactional Outbox where database state and events must remain
    consistent.
12. Use reconciliation to resolve unknown states.
13. Never store CVV/PIN.
14. Minimize PCI scope.
15. Never log payment secrets.
16. Use circuit breakers.
17. Use bounded retries with jitter.
18. Respect provider rate limits.
19. Scale Pods only within provider and database capacity.
20. Treat financial state separately from Kubernetes deployment state.
21. A Kubernetes rollback cannot undo a captured payment.
22. Every payment incident requires financial reconciliation.
23. Use canary deployment for high-risk payment changes.
24. Monitor business success rate, not only CPU and memory.
25. Test provider failures before production.

# End of 171 — Payment Service
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
171 Payment Service         ✅

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

**Next: `172-Fraud-Service.md` — fraud scoring, synchronous/asynchronous fraud checks, rules engine, risk signals, Redis, PostgreSQL, Kafka, external fraud provider/model integration, order/payment interaction, false positives, idempotency, security, Kubernetes, CI/CD, GitOps, observability, failure scenarios and production RCA.**