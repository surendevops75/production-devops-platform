# 179 — Tax Service

**File:** `179-Tax-Service.md`

```markdown
# 179 — Tax Service

## 1. Overview

The `tax-service` calculates taxes for the e-commerce platform.

Tax calculation can depend on:

- Customer location
- Shipping destination
- Billing address
- Product category
- Product tax classification
- Seller location
- Jurisdiction
- Tax rules
- Effective dates
- Exemptions
- Discounts
- Shipping charges
- Order value
- Tax provider response

The Tax Service is part of the order-pricing workflow, so it must be:

- Fast
- Deterministic
- Auditable
- Versioned
- Idempotent
- Highly available
- Failure-aware

Technology:

- Java 21
- Spring Boot
- Maven
- PostgreSQL
- Redis
- Kafka
- External tax provider
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
```

---

# 2. Why Tax Is a Separate Service

Tax calculation is business logic that can become complicated very quickly.

Without a dedicated service:

```text
Order Service
     |
     ├── Order logic
     ├── Payment logic
     ├── Inventory logic
     ├── Promotion logic
     └── Tax logic
```

This makes Order Service difficult to maintain.

Instead:

```text
Order Service
      |
      ↓
Tax Service
      |
      ├── Tax Rules
      ├── Jurisdiction
      ├── Tax Classification
      └── External Tax Provider
```

---

# 3. Important Principle

The Tax Service calculates tax.

It does **not** own:

```text
Product
Customer
Order
Payment
Inventory
```

Those remain owned by their respective services.

---

# 4. Tax Architecture

```text
                         Customer
                            |
                            ↓
                       Order Service
                            |
                            ↓
                        Tax Service
                     /       |       \
                    ↓        ↓        ↓
               PostgreSQL  Redis   Tax Provider
                    |
                    ↓
                 Kafka
                    |
          ┌─────────┼──────────┐
          ↓         ↓          ↓
      Reporting  Invoice   Audit/Data
```

---

# 5. Tax Calculation Flow

```text
Order
  |
  ↓
Collect taxable lines
  |
  ↓
Determine destination
  |
  ↓
Determine jurisdiction
  |
  ↓
Determine product tax category
  |
  ↓
Load applicable tax rules
  |
  ↓
Calculate tax
  |
  ↓
External provider if required
  |
  ↓
Return tax breakdown
  |
  ↓
Persist tax calculation
```

---

# 6. Why Tax Must Be Deterministic

Suppose the customer sees:

```text
Subtotal = ₹10,000
Tax = ₹1,800
Total = ₹11,800
```

The final order should not unexpectedly become:

```text
Tax = ₹2,050
```

simply because the same request was retried.

Therefore every calculation should have:

```text
calculationId
ruleVersion
tax jurisdiction
taxable amount
tax amount
calculation timestamp
```

---

# 7. Tax Calculation ID

Example:

```text
TAXCALC-2026-000001
```

This allows the organization to answer:

> Which tax rules were used to calculate this order?

---

# 8. Tax Rule Versioning

Never store only:

```text
taxRate = 18
```

Also store:

```text
taxRate = 18
ruleVersion = GST-2026-10-01
effectiveFrom = ...
effectiveTo = ...
```

This is critical for audits.

---

# 9. Effective-Dated Rules

Example:

```text
Rule A
18%
effective:
2026-01-01 → 2026-09-30

Rule B
20%
effective:
2026-10-01 → future
```

An order created on:

```text
2026-09-30
```

should not accidentally use the rule introduced on:

```text
2026-10-01
```

---

# 10. Tax Calculation Request

Example:

```json
{
  "orderId": "ORD-1001",
  "customerId": "CUS-1001",
  "currency": "INR",
  "shippingAddress": {
    "country": "IN",
    "state": "TS",
    "postalCode": "500001"
  },
  "lines": [
    {
      "productId": "PROD-1001",
      "quantity": 2,
      "unitPrice": 1000.00,
      "discount": 100.00,
      "taxCategory": "STANDARD"
    }
  ],
  "shippingAmount": 100.00
}
```

---

# 11. Tax Calculation Response

```json
{
  "calculationId": "TAXCALC-1001",
  "orderId": "ORD-1001",
  "currency": "INR",
  "subtotal": 1900.00,
  "taxableAmount": 1900.00,
  "taxAmount": 342.00,
  "total": 2242.00,
  "ruleVersion": "GST-2026-10",
  "jurisdiction": "IN-TS",
  "lines": [
    {
      "productId": "PROD-1001",
      "taxableAmount": 1900.00,
      "taxRate": 18.00,
      "taxAmount": 342.00
    }
  ]
}
```

The exact tax components depend on the applicable jurisdiction and business
rules.

---

# 12. API Design

```text
POST /api/v1/tax/calculate

GET  /api/v1/tax/calculations/{calculationId}

POST /api/v1/tax/validate

GET  /api/v1/tax/rules/{taxCategory}
```

Administrative rule-management APIs should be separately protected.

---

# 13. Repository Structure

```text
tax-service/
│
├── src/
│   ├── main/
│   │   ├── java/com/company/tax/
│   │   │   ├── TaxServiceApplication.java
│   │   │   │
│   │   │   ├── controller/
│   │   │   │   └── TaxController.java
│   │   │   │
│   │   │   ├── service/
│   │   │   │   ├── TaxCalculationService.java
│   │   │   │   ├── TaxRuleService.java
│   │   │   │   └── TaxProviderService.java
│   │   │   │
│   │   │   ├── repository/
│   │   │   │   ├── TaxCalculationRepository.java
│   │   │   │   └── TaxRuleRepository.java
│   │   │   │
│   │   │   ├── entity/
│   │   │   │   ├── TaxCalculation.java
│   │   │   │   ├── TaxCalculationLine.java
│   │   │   │   └── TaxRule.java
│   │   │   │
│   │   │   ├── dto/
│   │   │   │   ├── TaxCalculationRequest.java
│   │   │   │   └── TaxCalculationResponse.java
│   │   │   │
│   │   │   ├── provider/
│   │   │   │   └── ExternalTaxProviderClient.java
│   │   │   │
│   │   │   ├── cache/
│   │   │   │   └── TaxCacheService.java
│   │   │   │
│   │   │   ├── config/
│   │   │   │   ├── RedisConfig.java
│   │   │   │   └── WebClientConfig.java
│   │   │   │
│   │   │   └── exception/
│   │   │       └── TaxCalculationException.java
│   │   │
│   │   └── resources/
│   │       ├── application.yaml
│   │       └── application-prod.yaml
│   │
│   └── test/
│
├── src/main/resources/db/migration/
│   ├── V1__create_tax_rules.sql
│   └── V2__create_tax_calculations.sql
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── pom.xml
├── README.md
│
├── helm/
│   └── tax-service/
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

# 14. Maven

`pom.xml`:

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0">

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
    </parent>

    <groupId>
        com.company
    </groupId>

    <artifactId>
        tax-service
    </artifactId>

    <version>
        1.0.0
    </version>

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
                spring-boot-starter-validation
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
                org.springframework.boot
            </groupId>

            <artifactId>
                spring-boot-starter-actuator
            </artifactId>
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

Exact Spring Boot versions should be pinned to the version tested by the
project.

---

# 15. Main Application

```java
package com.company.tax;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class TaxServiceApplication {

    public static void main(String[] args) {

        SpringApplication.run(
            TaxServiceApplication.class,
            args
        );
    }
}
```

---

# 16. Money Handling

Do not use:

```java
double
```

for financial calculations.

Use:

```java
BigDecimal
```

Example:

```java
BigDecimal taxableAmount =
    unitPrice
        .multiply(quantity)
        .subtract(discount);
```

Tax calculations require deterministic decimal arithmetic.

---

# 17. Tax Request DTO

```java
public record TaxCalculationRequest(

    @NotBlank
    String orderId,

    @NotBlank
    String customerId,

    @NotBlank
    String currency,

    @NotNull
    Address shippingAddress,

    @NotEmpty
    List<TaxLineRequest> lines,

    @NotNull
    BigDecimal shippingAmount

) {}
```

---

# 18. Tax Line DTO

```java
public record TaxLineRequest(

    @NotBlank
    String productId,

    @NotNull
    @Positive
    Integer quantity,

    @NotNull
    @PositiveOrZero
    BigDecimal unitPrice,

    @NotNull
    @PositiveOrZero
    BigDecimal discount,

    @NotBlank
    String taxCategory

) {}
```

---

# 19. Tax Rule Entity

```java
@Entity
@Table(name = "tax_rules")
public class TaxRule {

    @Id
    @GeneratedValue
    private Long id;

    private String ruleCode;

    private String ruleVersion;

    private String jurisdiction;

    private String taxCategory;

    private BigDecimal rate;

    private LocalDate effectiveFrom;

    private LocalDate effectiveTo;

    private boolean active;
}
```

---

# 20. Tax Calculation Entity

```java
@Entity
@Table(name = "tax_calculations")
public class TaxCalculation {

    @Id
    @GeneratedValue
    private Long id;

    @Column(unique = true)
    private String calculationId;

    private String orderId;

    private String customerId;

    private String jurisdiction;

    private String ruleVersion;

    private BigDecimal taxableAmount;

    private BigDecimal taxAmount;

    private Instant calculatedAt;
}
```

---

# 21. Database Migration

## `V1__create_tax_rules.sql`

```sql
CREATE TABLE tax_rules (

    id BIGSERIAL PRIMARY KEY,

    rule_code VARCHAR(100) NOT NULL,

    rule_version VARCHAR(100) NOT NULL,

    jurisdiction VARCHAR(100) NOT NULL,

    tax_category VARCHAR(100) NOT NULL,

    rate NUMERIC(10,4) NOT NULL,

    effective_from DATE NOT NULL,

    effective_to DATE,

    active BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE INDEX idx_tax_rules_lookup
ON tax_rules(
    jurisdiction,
    tax_category,
    effective_from
);
```

---

# 22. Calculation Table

## `V2__create_tax_calculations.sql`

```sql
CREATE TABLE tax_calculations (

    id BIGSERIAL PRIMARY KEY,

    calculation_id VARCHAR(100)
        NOT NULL UNIQUE,

    order_id VARCHAR(100)
        NOT NULL,

    customer_id VARCHAR(100)
        NOT NULL,

    jurisdiction VARCHAR(100)
        NOT NULL,

    rule_version VARCHAR(100)
        NOT NULL,

    taxable_amount NUMERIC(19,4)
        NOT NULL,

    tax_amount NUMERIC(19,4)
        NOT NULL,

    calculated_at TIMESTAMP
        NOT NULL
);
```

---

# 23. Idempotency

Tax calculation requests can be retried.

Example:

```text
Order Service
     |
     ↓
Tax Service
     |
     ↓
timeout
     |
     ↓
retry
```

Without idempotency:

```text
calculation A
calculation B
```

could be created for the same logical operation.

Use:

```text
orderId
calculationId
requestId
```

according to the business workflow.

---

# 24. Idempotency Table

```sql
CREATE TABLE tax_idempotency (

    idempotency_key VARCHAR(200) PRIMARY KEY,

    order_id VARCHAR(100) NOT NULL,

    response JSONB NOT NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);
```

This allows repeated requests to return the same result where appropriate.

---

# 25. Tax Calculation Service

Conceptual implementation:

```java
@Service
public class TaxCalculationService {

    public TaxCalculationResponse calculate(
        TaxCalculationRequest request
    ) {

        TaxRule rule =
            taxRuleService.findApplicableRule(
                request.shippingAddress(),
                request.lines()
            );

        BigDecimal taxableAmount =
            calculateTaxableAmount(request);

        BigDecimal taxAmount =
            taxableAmount
                .multiply(rule.getRate())
                .divide(
                    BigDecimal.valueOf(100)
                );

        return buildResponse(
            request,
            taxableAmount,
            taxAmount,
            rule
        );
    }
}
```

The real implementation must handle jurisdiction-specific rules, exemptions,
rounding, tax-inclusive pricing and multiple tax components as applicable.

---

# 26. Tax Provider Integration

Some organizations use an external tax provider.

Architecture:

```text
Tax Service
     |
     ↓
Tax Provider Client
     |
     ↓
External Tax Provider
```

The external provider should not be called blindly for every request.

Use:

```text
timeouts
retry policy
circuit breaker
rate limits
idempotency
fallback
```

---

# 27. External Provider Timeout

Example:

```text
Tax Service
    |
    ↓
External Provider
    |
    X
timeout
```

The service should fail within a bounded time.

Do not allow:

```text
30-second provider timeout
```

to block every order request.

The exact timeout must be derived from the order-path latency budget.

---

# 28. Circuit Breaker

Conceptual:

```text
CLOSED
  |
  ↓
provider failures increase
  |
  ↓
OPEN
  |
  ↓
fallback / controlled failure
  |
  ↓
HALF-OPEN
  |
  ↓
provider recovered
  |
  ↓
CLOSED
```

For taxes, fallback must be legally and operationally valid.

Do not invent a tax rate simply to keep an order alive.

---

# 29. Tax Fallback Strategy

There are three possible strategies depending on the business/legal model:

```text
Option A
Use locally maintained authoritative tax rules.

Option B
Use last-known approved rules where legally acceptable.

Option C
Reject/defer checkout when authoritative tax calculation
cannot be safely completed.
```

The service must not silently return an approximate tax amount for a legally
binding transaction.

---

# 30. Redis

Redis can cache:

```text
tax rules
jurisdiction mappings
tax provider metadata
non-sensitive calculation results
```

Example:

```text
tax:rule:IN:TS:STANDARD
```

TTL should respect the rule's effective period.

Do not cache a tax rule beyond its validity window.

---

# 31. Tax Rule Cache Invalidation

When a new tax rule is activated:

```text
New Tax Rule
    |
    ↓
Kafka
    |
    ↓
Tax Service
    |
    ↓
Invalidate Redis
```

Example event:

```json
{
  "eventType": "TAX_RULE_UPDATED",
  "version": 1,
  "ruleVersion": "GST-2026-10"
}
```

---

# 32. Tax Rule Auditability

Every production tax calculation should be traceable to:

```text
calculationId
orderId
ruleVersion
jurisdiction
tax category
taxable amount
tax rate
tax amount
timestamp
```

This allows later investigation.

---

# 33. Order Integration

Recommended order flow:

```text
Customer
   |
   ↓
Order Service
   |
   ├── Product
   ├── Pricing
   ├── Promotion
   ├── Inventory
   |
   ↓
Tax Service
   |
   ↓
Tax Calculation
   |
   ↓
Order Total
   |
   ↓
Payment
```

Tax should be calculated before final payment authorization/capture according
to the business flow.

---

# 34. Important Order Principle

Never trust the client to calculate:

```text
tax
total
discount
```

The server must calculate authoritative values.

Example:

```text
Client:
subtotal = 1000
tax = 0
total = 1000
```

The server independently calculates:

```text
subtotal = 1000
tax = 180
total = 1180
```

---

# 35. Tax Calculation vs Payment

Payment Service should receive:

```text
final authoritative order amount
```

from the order workflow.

It should not independently invent the tax.

---

# 36. Tax Calculation vs Invoice

Invoice Service consumes the final tax information.

Example:

```text
Order
  |
  ↓
Tax
  |
  ↓
Final Order
  |
  ↓
Invoice
```

Invoice should preserve the tax calculation details required for financial
records.

---

# 37. Kafka Events

Tax Service may publish:

```text
TAX_CALCULATED
TAX_RECALCULATED
TAX_RULE_UPDATED
```

Example:

```json
{
  "eventId": "evt-tax-1001",
  "eventType": "TAX_CALCULATED",
  "version": 1,
  "occurredAt": "2026-10-05T12:00:00Z",
  "source": "tax-service",
  "correlationId": "corr-order-1001",
  "aggregateId": "ORD-1001",
  "payload": {
    "calculationId": "TAXCALC-1001",
    "taxAmount": "342.00",
    "ruleVersion": "GST-2026-10"
  }
}
```

---

# 38. Tax Precision

Avoid:

```text
float
double
```

for money.

Use:

```text
BigDecimal
NUMERIC(19,4)
```

and define explicit rounding rules.

Example:

```java
taxAmount.setScale(
    2,
    RoundingMode.HALF_UP
);
```

The exact rounding mode must match the applicable business/tax rules.

---

# 39. Tax-Inclusive vs Tax-Exclusive Pricing

Two models exist.

## Tax exclusive

```text
Product = 1000
Tax = 180
Total = 1180
```

## Tax inclusive

```text
Displayed price = 1180
Tax component is extracted from the total.
```

The service must know which pricing model applies.

Do not mix these models.

---

# 40. Tax Exemption

Some customers/products may have exemptions.

Example:

```text
Customer
    |
    ↓
Tax exemption
    |
    ↓
Tax rate = 0
```

But exemption information must come from an authoritative and auditable source.

Do not allow the client to simply send:

```json
{
  "taxExempt": true
}
```

and bypass tax.

---

# 41. Tax Jurisdiction

Jurisdiction may depend on:

```text
billing address
shipping address
seller location
product type
customer status
```

The actual legal determination must be implemented according to the applicable
tax regime.

The service should make the jurisdiction decision explicit and auditable.

---

# 42. Dockerfile

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

# 43. `.dockerignore`

```text
.git
.github
.idea
.vscode

target

.env
.env.*

*.log

README.md
```

---

# 44. Local Docker Compose

```yaml
services:

  postgres:

    image: postgres:16

    container_name: tax-postgres

    environment:

      POSTGRES_DB: tax

      POSTGRES_USER: postgres

      POSTGRES_PASSWORD: postgres

    ports:

      - "5432:5432"

    volumes:

      - postgres-data:/var/lib/postgresql/data


  redis:

    image: redis:7

    container_name: tax-redis

    ports:

      - "6379:6379"


  tax-service:

    build:

      context: .

      dockerfile: Dockerfile

    depends_on:

      - postgres
      - redis

    environment:

      SPRING_PROFILES_ACTIVE: local

      DB_URL:
        jdbc:postgresql://postgres:5432/tax

      DB_USERNAME:
        postgres

      DB_PASSWORD:
        postgres

      REDIS_URL:
        redis://redis:6379

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

---

# 45. `application.yaml`

```yaml
server:

  port: 8080


spring:

  application:

    name: tax-service

  datasource:

    url: ${DB_URL:jdbc:postgresql://localhost:5432/tax}

    username: ${DB_USERNAME:postgres}

    password: ${DB_PASSWORD:postgres}

    hikari:

      maximum-pool-size: 10

      minimum-idle: 2

      connection-timeout: 3000

      validation-timeout: 1000

      idle-timeout: 600000

      max-lifetime: 1500000

      pool-name: tax-service-hikari

  jpa:

    hibernate:

      ddl-auto: validate

    open-in-view: false

    show-sql: false

  flyway:

    enabled: true

  data:

    redis:

      url: ${REDIS_URL:redis://localhost:6379}


management:

  endpoints:

    web:

      exposure:

        include:
          - health
          - info
          - prometheus

  endpoint:

    health:

      probes:

        enabled: true


logging:

  level:

    root: INFO

    com.company.tax: INFO
```

---

# 46. Hikari Pool

The pool is **per application Pod**.

Example:

```text
10 Pods
×
10 max connections
=
100 potential connections
```

Therefore:

```text
Tax HPA
+
Hikari pool
+
RDS connection budget
```

must be designed together.

Do not blindly increase Hikari when tax requests become slow.

---

# 47. Production Configuration

```yaml
spring:

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

      pool-name: tax-service-hikari

  jpa:

    hibernate:

      ddl-auto: validate

    open-in-view: false
```

The pool values are illustrative.

They must be tuned from:

```text
RDS capacity
load testing
query latency
replica count
other services
deployment surge
```

---

# 48. Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:

  name: tax-service

  labels:

    app: tax-service

spec:

  replicas: 3

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: tax-service

  template:

    metadata:

      labels:

        app: tax-service

    spec:

      serviceAccountName: tax-service

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: tax-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: tax-service-config

            - secretRef:

                name: tax-service-secrets

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

# 49. Kubernetes Service

```yaml
apiVersion: v1
kind: Service

metadata:

  name: tax-service

  labels:

    app: tax-service

spec:

  type: ClusterIP

  selector:

    app: tax-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 50. ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap

metadata:

  name: tax-service-config

data:

  REDIS_URL:
    redis://redis.ecommerce.svc.cluster.local:6379

  TAX_PROVIDER_TIMEOUT_MS:
    "1500"

  TAX_RULE_CACHE_TTL:
    "300"
```

Timeout is an example only.

---

# 51. ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret

metadata:

  name: tax-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: tax-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/tax-service

        property: DATABASE_URL

    - secretKey: TAX_PROVIDER_API_KEY

      remoteRef:

        key: ecommerce/prod/tax-service

        property: TAX_PROVIDER_API_KEY
```

---

# 52. ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount

metadata:

  name: tax-service

  labels:

    app: tax-service
```

If the service needs AWS access:

```text
Tax Pod
   ↓
EKS Pod Identity
   ↓
IAM Role
   ↓
specific AWS resources
```

---

# 53. NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy

metadata:

  name: tax-service

spec:

  podSelector:

    matchLabels:

      app: tax-service

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

          port: 443

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

---

# 54. HPA

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler

metadata:

  name: tax-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: tax-service

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

Tax is on the order path, so aggressive scaling must not overwhelm:

```text
RDS
external tax provider
network
```

---

# 55. PDB

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget

metadata:

  name: tax-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: tax-service
```

---

# 56. ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor

metadata:

  name: tax-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: tax-service

  endpoints:

    - port: http

      path: /actuator/prometheus

      interval: 15s
```

---

# 57. Tax Metrics

Application:

```text
tax_calculation_requests_total

tax_calculation_duration_seconds

tax_calculation_errors_total

tax_calculation_success_total
```

Provider:

```text
tax_provider_requests_total

tax_provider_errors_total

tax_provider_timeout_total

tax_provider_duration_seconds

tax_provider_circuit_open_total
```

Database:

```text
hikaricp_connections_active

hikaricp_connections_idle

hikaricp_connections_pending

hikaricp_connections_max
```

Cache:

```text
tax_cache_hits_total

tax_cache_misses_total
```

Business:

```text
tax_calculation_fallback_total

tax_rule_version_usage_total
```

---

# 58. Grafana Dashboard

```text
Tax Service
│
├── API
│   ├── RPS
│   ├── P50
│   ├── P95
│   ├── P99
│   └── 5xx
│
├── Tax Provider
│   ├── Latency
│   ├── Errors
│   ├── Timeouts
│   └── Circuit Breaker
│
├── Database
│   ├── Connections
│   ├── Pending
│   └── Query Latency
│
├── Redis
│   └── Hit Ratio
│
└── Business
    ├── Calculations
    ├── Fallbacks
    └── Rule Versions
```

---

# 59. Logging

Example:

```json
{
  "@timestamp": "2026-10-05T13:00:00Z",
  "level": "INFO",
  "service": "tax-service",
  "correlationId": "corr-order-1001",
  "orderId": "ORD-1001",
  "calculationId": "TAXCALC-1001",
  "jurisdiction": "IN-TS",
  "ruleVersion": "GST-2026-10",
  "taxAmount": "342.00",
  "durationMs": 35
}
```

Do not log:

```text
tax provider API keys
customer secrets
full payment information
unnecessary personal information
```

---

# 60. OpenTelemetry

Example trace:

```text
Order Service
      |
      ↓
POST /tax/calculate
      |
      ├── Redis rule lookup
      |
      ├── PostgreSQL
      |
      └── External Tax Provider
```

Jaeger can identify:

```text
PostgreSQL = 20ms
Redis = 2ms
Provider = 500ms
```

This immediately shows the external provider as the latency contributor.

---

# 61. Tax SLO

Example:

```text
Availability:
99.95%

P95:
< 150ms for local-rule calculations

P95:
within order-path latency budget for provider-backed calculations

5xx:
< 0.1%

Provider timeout:
< defined threshold

Calculation correctness:
100% for validated rule scenarios
```

Correctness should be validated through automated tax-rule test suites, not
just infrastructure metrics.

---

# 62. Alerts

Critical:

```text
Tax Service unavailable
Tax calculation 5xx high
Tax calculation failure rate high
```

Warning:

```text
Tax provider latency high
Tax provider timeout increasing
Tax provider circuit open
RDS connections high
Hikari pending connections high
Redis hit ratio low
Tax fallback increasing
```

Business:

```text
Unexpected tax calculation deviation
Unknown tax rule version
Tax calculation mismatch
```

---

# 63. CI/CD

Pipeline:

```text
Developer
   |
   ↓
Pull Request
   |
   ↓
Build
   |
   ↓
Unit Tests
   |
   ↓
Integration Tests
   |
   ↓
SonarQube
   |
   ↓
Trivy
   |
   ↓
Docker Build
   |
   ↓
Image Scan
   |
   ↓
ECR
   |
   ↓
GitOps
   |
   ↓
ArgoCD
   |
   ↓
EKS
```

---

# 64. GitHub Actions

```yaml
name: Tax Service CI

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

  ECR_REPOSITORY: tax-service

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

      - name: Setup Java

        uses: actions/setup-java@v4

        with:

          distribution: temurin

          java-version: "21"

      - name: Configure AWS

        uses: aws-actions/configure-aws-credentials@v4

        with:

          role-to-assume: ${{ secrets.AWS_CI_ROLE_ARN }}

          aws-region: ${{ env.AWS_REGION }}

      - name: ECR Login

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

Actions and base images should be pinned according to the project's supply-chain
policy.

---

# 65. SonarQube

SonarQube checks:

```text
Java code
Spring Boot code
unit tests
coverage
security issues
code smells
duplication
complexity
```

Tax-specific tests should include:

```text
rounding
effective dates
jurisdiction
tax categories
exemptions
duplicate requests
provider failures
```

---

# 66. ECR

Repository:

```text
company-ecommerce/tax-service
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/tax-service:<git-sha>
```

Prefer immutable image tags.

---

# 67. Helm

## `Chart.yaml`

```yaml
apiVersion: v2

name: tax-service

description: Ecommerce tax calculation service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 68. Production Values

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/tax-service

  tag: ""

  pullPolicy: IfNotPresent

resources:

  requests:

    cpu: 250m

    memory: 512Mi

  limits:

    cpu: "1"

    memory: 1Gi

autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 15

serviceAccount:

  create: true

  name: tax-service
```

---

# 69. GitOps

```text
company-ecommerce/gitops
└── apps/
    └── tax-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Image promotion:

```text
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

# 70. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application

metadata:

  name: tax-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/tax-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

---

# 71. Failure Scenario — Tax Provider Down

```text
Order Service
     |
     ↓
Tax Service
     |
     ↓
Tax Provider
     X
```

The Tax Service should:

```text
detect
→ timeout
→ circuit breaker
→ controlled fallback
```

It must not:

```text
retry indefinitely
```

---

# 72. Failure Scenario — Provider Slow

Suppose:

```text
normal provider latency = 100ms

current latency = 3s
```

Without a timeout:

```text
Order request
    ↓
Tax
    ↓
Provider
    ↓
3 seconds
```

Many concurrent requests can consume all application threads/connections.

Therefore external dependency timeout is essential.

---

# 73. Failure Scenario — RDS Connection Exhaustion

Symptoms:

```text
Hikari pending ↑
tax P95 ↑
tax 5xx ↑
RDS connections ↑
```

Check:

```text
Pods
HPA
Hikari
RDS connections
slow queries
deployment surge
```

Remember:

```text
Pods × Hikari maximum pool
```

---

# 74. Failure Scenario — Redis Down

Tax rules may still be retrieved from PostgreSQL.

But:

```text
Redis failure
→ DB cache misses
→ PostgreSQL traffic
```

can overload RDS.

Use:

```text
short Redis timeout
bounded fallback
cache warming
request coalescing
```

---

# 75. Failure Scenario — Wrong Tax Rule

Suppose:

```text
Expected:
18%

Actual:
20%
```

Investigate:

```text
rule version
effective date
jurisdiction
cache
deployment
database migration
configuration
```

Never immediately assume the application calculation formula is wrong.

---

# 76. Production Incident

## Incident ID

```text
INC-2026-TAX-001
```

## Severity

```text
SEV-1
```

## Title

```text
Tax Provider Latency Caused Checkout Failures
```

## Date

```text
2026-10-05
```

---

# 77. Customer Impact

Customers experienced:

```text
checkout latency
HTTP 504
failed checkout attempts
```

Product browsing and cart operations remained healthy.

Payment capture was not incorrectly performed for failed tax calculations.

---

# 78. Timeline

```text
18:00
Tax provider healthy.

18:05
Provider latency starts increasing.

18:07
Tax P95 increases from 80ms to 700ms.

18:09
Tax provider timeout rate increases.

18:10
Order-service checkout P95 increases.

18:11
Order 5xx begins increasing.

18:12
SEV-1 declared.

18:14
Jaeger identifies external tax provider span.

18:16
Tax provider circuit breaker begins opening.

18:18
Order traffic is protected by bounded timeout.

18:20
Fallback to locally maintained rules enabled
where legally/business approved.

18:23
Checkout error rate decreases.

18:30
Tax provider latency begins recovering.

18:35
Circuit breaker closes after successful probes.

18:40
Checkout latency normal.

18:45
Incident resolved.
```

---

# 79. Root Cause

The external tax provider experienced severe latency.

The Tax Service initially allowed requests to wait too long for provider
responses.

This caused:

```text
provider latency
      ↓
Tax request latency
      ↓
Order request latency
      ↓
thread/concurrency pressure
      ↓
checkout failures
```

---

# 80. Amplification

The external provider was the primary failure.

The amplification mechanism was:

```text
long dependency timeout
+
insufficient early circuit breaking
+
synchronous checkout dependency
```

---

# 81. What Went Well

- Tax Service was isolated from Product and Inventory.
- Jaeger identified the external dependency.
- Prometheus detected latency.
- Checkout did not charge customers before successful tax completion.
- Controlled fallback was available.
- Circuit breaker eventually protected the service.

---

# 82. What Went Wrong

- Provider timeout was too high.
- Provider latency budget was not aligned with checkout SLO.
- Circuit breaker thresholds were too conservative.
- Provider dependency was not tested under degraded latency.
- Alerting detected the impact later than desired.

---

# 83. Five Whys

### Why did checkout fail?

Tax calculation requests timed out.

### Why?

The external provider became very slow.

### Why did slow provider responses affect checkout?

Requests waited too long.

### Why?

Timeout and circuit-breaker configuration was not aligned with the order-path
latency budget.

### Why?

External dependency failure testing was insufficient.

---

# 84. Corrective Actions

| Action | Owner |
|---|---|
| Reduce provider timeout | Backend |
| Tune circuit breaker | Backend/SRE |
| Add provider latency alert | SRE |
| Add dependency failure tests | QA |
| Add provider load/degradation tests | Performance |
| Validate fallback rules | Tax/Legal |
| Add dependency SLO | SRE |
| Add tax provider dashboard | Observability |
| Add controlled provider failover | Architecture |
| Document tax incident runbook | DevOps/SRE |

---

# 85. Tax Runbook

```text
1. Confirm customer impact.
2. Check Tax API RPS.
3. Check tax P95/P99.
4. Check tax 5xx.
5. Check provider latency.
6. Check provider timeout.
7. Check circuit breaker state.
8. Check RDS connections.
9. Check Redis.
10. Check recent deployment.
11. Check rule version.
12. Determine provider vs application failure.
13. Protect order path.
14. Enable approved fallback if available.
15. Verify tax correctness.
16. Restore provider dependency.
17. Verify checkout.
18. Record incident evidence.
19. Complete RCA.
```

---

# 86. Tax Security Checklist

```text
[ ] Authentication
[ ] Authorization
[ ] No client-controlled final tax
[ ] Secrets Manager
[ ] External Secrets
[ ] Pod Identity
[ ] Least-privilege IAM
[ ] NetworkPolicy
[ ] TLS to external provider
[ ] API key rotation
[ ] No secrets in Git
[ ] No secrets in Docker image
[ ] Trivy
[ ] SonarQube
[ ] Audit logs
[ ] PII minimization
```

---

# 87. Tax Reliability Checklist

```text
[ ] Idempotency
[ ] Database constraints
[ ] Rule versioning
[ ] Effective dates
[ ] BigDecimal
[ ] Explicit rounding
[ ] Provider timeout
[ ] Circuit breaker
[ ] Retry with limits
[ ] Retry backoff/jitter
[ ] Redis timeout
[ ] Cache stampede protection
[ ] RDS connection budget
[ ] HPA limits
[ ] PDB
[ ] Multi-AZ RDS
[ ] Provider failure test
[ ] Redis failure test
[ ] RDS failure test
[ ] Rule reconciliation
```

---

# 88. Final Architecture

```text
                         Customer
                            |
                            ↓
                         ALB
                            |
                            ↓
                      Order Service
                            |
                            ↓
                       Tax Service
                    /       |       \
                   ↓        ↓        ↓
              PostgreSQL  Redis   Tax Provider
                   |                 |
                   |                 ↓
                   |             External API
                   |
                   ↓
              Tax Calculation
                   |
                   ↓
               Order Total
                   |
                   ↓
                Payment


Tax Events
    |
    ↓
  Kafka
    |
    ├── Invoice Service
    ├── Reporting Service
    └── Audit/Data Platform
```

---

# 89. Final Production Principles

1. Tax Service owns tax calculation logic.
2. Order Service owns the order.
3. Payment Service owns payment.
4. Product Service owns product information.
5. Tax calculations must be deterministic.
6. Money must use decimal-safe arithmetic.
7. Java implementations should use `BigDecimal`.
8. PostgreSQL monetary fields should use `NUMERIC`.
9. Tax rules must be versioned.
10. Tax rules must have effective dates.
11. Jurisdiction selection must be explicit and auditable.
12. Clients must never control the authoritative tax amount.
13. Tax calculation requests must support idempotency.
14. Redis is a cache, not the authoritative tax-rule store.
15. Cache TTL must respect tax-rule validity.
16. PostgreSQL remains the authoritative store for tax records/rules in this
    implementation.
17. External tax providers must have bounded timeouts.
18. Retries must be limited and use backoff/jitter.
19. Circuit breakers protect the order path.
20. Fallback tax calculation must be legally and operationally approved.
21. Never invent an approximate tax amount during provider failure.
22. Tax calculations must remain auditable.
23. Hikari pool size must be planned with RDS capacity.
24. HPA replica limits must consider RDS connection budgets.
25. Tax provider latency must be included in the checkout latency budget.
26. Prometheus monitors infrastructure and application behavior.
27. Grafana provides operational and business dashboards.
28. ELK provides centralized logs.
29. OpenTelemetry and Jaeger provide distributed traces.
30. GitHub Actions performs CI.
31. Trivy performs security scanning.
32. SonarQube performs code-quality analysis.
33. ECR stores immutable application images.
34. GitOps controls deployment state.
35. ArgoCD reconciles Kubernetes state.
36. Tax rules and calculation correctness require dedicated automated tests.
37. Every major tax incident requires an RCA.
38. Tax failures must be prevented from silently corrupting financial data.
39. The final production flow remains:

    `Developer → PR → CI → ECR → GitOps → ArgoCD → EKS`
    
# End of 179 — Tax Service
```

### Phase 19 progress

```text
161 Auth Service             ✅
162 User Service             ✅
163 Customer Service         ✅
164 Catalog Service          ✅
165 Product Service          ✅
166 Inventory Service        ✅
167 Pricing Service          ✅
168 Promotion Service        ✅
169 Cart Service             ✅
170 Order Service            ✅
171 Payment Service          ✅
172 Fraud Service            ✅
173 Shipping Service         ✅
174 Delivery Service         ✅
175 Notification Service     ✅
176 Search Service           ✅
177 Recommendation Service   ✅
178 Review Service           ✅
179 Tax Service              ✅

180 Invoice Service          ⏳
181 Reporting Service        ⏳
```

**Next: `180-Invoice-Service.md`** — invoice generation, tax/price snapshotting, PDF generation, S3 storage, invoice numbering, Kafka events, idempotency, PostgreSQL, Kubernetes/EKS, CI/CD, GitOps, observability, failure scenarios and production RCA.