# 161 — Auth Service

**File:** `161-Auth-Service.md`

```markdown
# 161 — Auth Service

## 1. Overview

The `auth-service` is responsible for authentication and identity security for the Enterprise E-Commerce Platform.

It handles:

- User authentication
- Login
- Access-token generation
- Refresh-token management
- Logout
- Token validation
- Password management
- Authentication-related security events
- Login attempt protection
- Session/token state where required

The service is implemented using:

- Java 21
- Spring Boot
- Spring Security
- Maven
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

# 2. Why Auth Service Is Separate

Authentication should not be tightly coupled to the user profile service.

We separate:

```text
auth-service
    |
    +-- Credentials
    +-- Password hashing
    +-- Authentication
    +-- Access tokens
    +-- Refresh tokens
    +-- Login security
    +-- Authentication sessions


user-service
    |
    +-- User profile
    +-- Name
    +-- Email profile
    +-- Address
    +-- Preferences
```

The relationship is:

```text
                    ┌──────────────────┐
                    │   auth-service   │
                    │                  │
                    │ Authentication   │
                    │ Tokens           │
                    │ Credentials      │
                    └────────┬─────────┘
                             │
                             │ userId
                             ↓
                    ┌──────────────────┐
                    │   user-service   │
                    │                  │
                    │ User Profile     │
                    │ Customer Data    │
                    └──────────────────┘
```

The authentication database should not become the master database for all user information.

---

# 3. Business Responsibility

The auth-service owns:

| Responsibility | Owner |
|---|---|
| Password authentication | auth-service |
| Password hashing | auth-service |
| Access token issuance | auth-service |
| Refresh token rotation | auth-service |
| Logout | auth-service |
| Session management | auth-service |
| Login attempt protection | auth-service |
| Token validation | auth-service |
| Authentication events | auth-service |
| User profile | user-service |
| Customer profile | customer-service |
| Orders | order-service |
| Payments | payment-service |

---

# 4. Architecture

```text
                         Internet
                            |
                     CloudFront / WAF
                            |
                           ALB
                            |
                     auth-service
                            |
              ┌─────────────┼─────────────┐
              │             │             │
              ↓             ↓             ↓
         PostgreSQL       Redis         Kafka
              │             │             │
        Credentials    Rate limits    Auth events
        Auth data      Sessions       Security events
```

Production authentication flow:

```text
Client
  |
  | POST /api/v1/auth/login
  ↓
ALB
  |
  ↓
auth-service
  |
  ├── Validate request
  |
  ├── Check Redis rate limit
  |
  ├── Query PostgreSQL
  |
  ├── Verify password
  |
  ├── Generate access token
  |
  ├── Rotate refresh token
  |
  └── Publish authentication event
```

---

# 5. Technology Stack

| Layer | Technology |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot |
| Security | Spring Security |
| Build | Maven |
| Database | PostgreSQL |
| Cache/session state | Redis |
| Messaging | Kafka |
| Container | Docker |
| Orchestration | Kubernetes/EKS |
| Packaging | Helm |
| Container registry | Amazon ECR |
| CI | GitHub Actions |
| GitOps | ArgoCD |
| IaC | Terraform |
| Metrics | Prometheus |
| Dashboards | Grafana |
| Logs | ELK |
| Tracing | OpenTelemetry |
| Trace backend | Jaeger |

---

# 6. Repository

Repository:

```text
company-ecommerce/auth-service
```

Structure:

```text
auth-service/
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── company/
│   │   │           └── auth/
│   │   │               ├── AuthServiceApplication.java
│   │   │               │
│   │   │               ├── controller/
│   │   │               │   └── AuthController.java
│   │   │               │
│   │   │               ├── service/
│   │   │               │   └── AuthService.java
│   │   │               │
│   │   │               ├── repository/
│   │   │               │   ├── AuthUserRepository.java
│   │   │               │   └── RefreshTokenRepository.java
│   │   │               │
│   │   │               ├── entity/
│   │   │               │   ├── AuthUser.java
│   │   │               │   └── RefreshToken.java
│   │   │               │
│   │   │               ├── dto/
│   │   │               │   ├── LoginRequest.java
│   │   │               │   ├── LoginResponse.java
│   │   │               │   └── RefreshTokenRequest.java
│   │   │               │
│   │   │               ├── security/
│   │   │               │   ├── SecurityConfig.java
│   │   │               │   ├── JwtService.java
│   │   │               │   └── PasswordConfig.java
│   │   │               │
│   │   │               ├── config/
│   │   │               │   ├── RedisConfig.java
│   │   │               │   └── KafkaConfig.java
│   │   │               │
│   │   │               ├── exception/
│   │   │               │   ├── AuthException.java
│   │   │               │   └── GlobalExceptionHandler.java
│   │   │               │
│   │   │               └── filter/
│   │   │                   └── CorrelationIdFilter.java
│   │   │
│   │   └── resources/
│   │       ├── application.yaml
│   │       ├── application-prod.yaml
│   │       └── db/
│   │           └── migration/
│   │               ├── V1__create_auth_users.sql
│   │               └── V2__create_refresh_tokens.sql
│   │
│   └── test/
│       └── java/
│
├── pom.xml
├── mvnw
├── mvnw.cmd
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
│
├── helm/
│   └── auth-service/
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
│           ├── serviceaccount.yaml
│           ├── externalsecret.yaml
│           ├── networkpolicy.yaml
│           ├── hpa.yaml
│           ├── pdb.yaml
│           └── servicemonitor.yaml
│
├── .github/
│   ├── CODEOWNERS
│   └── workflows/
│       ├── ci.yaml
│       └── security.yaml
│
└── README.md
```

---

# 7. Maven Configuration

## `pom.xml`

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
        <version>3.5.6</version>
        <relativePath/>
    </parent>

    <groupId>com.company</groupId>
    <artifactId>auth-service</artifactId>

    <version>1.0.0</version>

    <name>auth-service</name>
    <description>Authentication Service</description>

    <properties>
        <java.version>21</java.version>
    </properties>

    <dependencies>

        <!-- REST API -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Security -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!-- Database -->
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

        <!-- Redis -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-redis</artifactId>
        </dependency>

        <!-- Kafka -->
        <dependency>
            <groupId>org.springframework.kafka</groupId>
            <artifactId>spring-kafka</artifactId>
        </dependency>

        <!-- Validation -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Actuator -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <!-- Prometheus -->
        <dependency>
            <groupId>io.micrometer</groupId>
            <artifactId>micrometer-registry-prometheus</artifactId>
        </dependency>

        <!-- Flyway -->
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>

        <!-- PostgreSQL Flyway database support -->
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-database-postgresql</artifactId>
        </dependency>

        <!-- JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>0.12.6</version>
        </dependency>

        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>0.12.6</version>
            <scope>runtime</scope>
        </dependency>

        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>0.12.6</version>
            <scope>runtime</scope>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
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

> Version numbers shown here are pinned examples for the project. Before production deployment, verify the selected versions against the organization's approved dependency baseline.

---

# 8. Main Application

## `AuthServiceApplication.java`

```java
package com.company.auth;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class AuthServiceApplication {

    public static void main(String[] args) {
        SpringApplication.run(AuthServiceApplication.class, args);
    }
}
```

---

# 9. Authentication Database

The auth service owns authentication data.

We do NOT store plaintext passwords.

```text
PostgreSQL
    |
    +-- auth_users
    |
    +-- refresh_tokens
    |
    +-- authentication/security metadata
```

---

# 10. Database Migration

## `V1__create_auth_users.sql`

```sql
CREATE TABLE auth_users (
    id BIGSERIAL PRIMARY KEY,

    user_id UUID NOT NULL UNIQUE,

    email VARCHAR(255) NOT NULL UNIQUE,

    password_hash VARCHAR(255) NOT NULL,

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',

    failed_login_attempts INTEGER NOT NULL DEFAULT 0,

    locked_until TIMESTAMP NULL,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_auth_users_email
ON auth_users(email);

CREATE INDEX idx_auth_users_user_id
ON auth_users(user_id);
```

---

# 11. Refresh Token Table

## `V2__create_refresh_tokens.sql`

```sql
CREATE TABLE refresh_tokens (
    id BIGSERIAL PRIMARY KEY,

    user_id UUID NOT NULL,

    token_hash VARCHAR(255) NOT NULL UNIQUE,

    expires_at TIMESTAMP NOT NULL,

    revoked BOOLEAN NOT NULL DEFAULT FALSE,

    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_refresh_tokens_user_id
ON refresh_tokens(user_id);

CREATE INDEX idx_refresh_tokens_expires_at
ON refresh_tokens(expires_at);
```

Refresh tokens should not be stored as plaintext.

The service stores a secure hash/reference and validates the supplied token against the stored representation.

---

# 12. Auth User Entity

## `AuthUser.java`

```java
package com.company.auth.entity;

import jakarta.persistence.*;

import java.time.LocalDateTime;
import java.util.UUID;

@Entity
@Table(name = "auth_users")
public class AuthUser {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "user_id", nullable = false, unique = true)
    private UUID userId;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(name = "password_hash", nullable = false)
    private String passwordHash;

    @Column(nullable = false)
    private String status;

    @Column(name = "failed_login_attempts", nullable = false)
    private Integer failedLoginAttempts = 0;

    @Column(name = "locked_until")
    private LocalDateTime lockedUntil;

    @Column(name = "created_at", nullable = false)
    private LocalDateTime createdAt;

    @Column(name = "updated_at", nullable = false)
    private LocalDateTime updatedAt;

    @PrePersist
    public void prePersist() {
        LocalDateTime now = LocalDateTime.now();

        createdAt = now;
        updatedAt = now;

        if (status == null) {
            status = "ACTIVE";
        }
    }

    @PreUpdate
    public void preUpdate() {
        updatedAt = LocalDateTime.now();
    }

    // Getters and setters
}
```

---

# 13. Repository

## `AuthUserRepository.java`

```java
package com.company.auth.repository;

import com.company.auth.entity.AuthUser;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;
import java.util.UUID;

public interface AuthUserRepository
        extends JpaRepository<AuthUser, Long> {

    Optional<AuthUser> findByEmail(String email);

    Optional<AuthUser> findByUserId(UUID userId);

    boolean existsByEmail(String email);
}
```

---

# 14. Login Request

## `LoginRequest.java`

```java
package com.company.auth.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;

public record LoginRequest(

        @NotBlank
        @Email
        String email,

        @NotBlank
        String password

) {
}
```

---

# 15. Login Response

## `LoginResponse.java`

```java
package com.company.auth.dto;

public record LoginResponse(

        String accessToken,

        String refreshToken,

        long expiresIn,

        String tokenType

) {
}
```

---

# 16. Refresh Token Request

## `RefreshTokenRequest.java`

```java
package com.company.auth.dto;

import jakarta.validation.constraints.NotBlank;

public record RefreshTokenRequest(

        @NotBlank
        String refreshToken

) {
}
```

---

# 17. Password Configuration

Passwords must never be stored directly.

Spring Security's password encoder is used.

## `PasswordConfig.java`

```java
package com.company.auth.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;

@Configuration
public class PasswordConfig {

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}
```

For a real organization, the password hashing algorithm and cost factor should follow the organization's security standard. Argon2id is also a strong option where supported by the security baseline.

---

# 18. JWT Service

## `JwtService.java`

```java
package com.company.auth.security;

import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.UUID;

@Service
public class JwtService {

    private final SecretKey secretKey;

    private final long expirationMillis;

    public JwtService(
            @Value("${security.jwt.secret}") String secret,
            @Value("${security.jwt.expiration-ms}") long expirationMillis) {

        this.secretKey = Keys.hmacShaKeyFor(
                secret.getBytes(StandardCharsets.UTF_8)
        );

        this.expirationMillis = expirationMillis;
    }

    public String generateAccessToken(UUID userId, String email) {

        Date now = new Date();

        Date expiration = new Date(
                now.getTime() + expirationMillis
        );

        return Jwts.builder()
                .subject(userId.toString())
                .claim("email", email)
                .issuedAt(now)
                .expiration(expiration)
                .signWith(secretKey)
                .compact();
    }
}
```

### Production security note

For production, the signing key must NOT be placed in Git or a Helm values file.

The key should come from:

```text
AWS Secrets Manager
        ↓
External Secrets Operator
        ↓
Kubernetes Secret
        ↓
auth-service
```

For larger deployments, asymmetric signing such as RSA/EC keys and a proper key-management/rotation strategy may be preferable.

---

# 19. Authentication Service

## `AuthService.java`

```java
package com.company.auth.service;

import com.company.auth.dto.LoginRequest;
import com.company.auth.dto.LoginResponse;
import com.company.auth.entity.AuthUser;
import com.company.auth.repository.AuthUserRepository;
import com.company.auth.security.JwtService;

import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class AuthService {

    private final AuthUserRepository repository;

    private final PasswordEncoder passwordEncoder;

    private final JwtService jwtService;

    public AuthService(
            AuthUserRepository repository,
            PasswordEncoder passwordEncoder,
            JwtService jwtService) {

        this.repository = repository;
        this.passwordEncoder = passwordEncoder;
        this.jwtService = jwtService;
    }

    @Transactional
    public LoginResponse login(LoginRequest request) {

        AuthUser user = repository
                .findByEmail(request.email())
                .orElseThrow(() ->
                        new RuntimeException("Invalid credentials"));

        if (!"ACTIVE".equals(user.getStatus())) {
            throw new RuntimeException("Account is not active");
        }

        if (!passwordEncoder.matches(
                request.password(),
                user.getPasswordHash())) {

            throw new RuntimeException("Invalid credentials");
        }

        String accessToken =
                jwtService.generateAccessToken(
                        user.getUserId(),
                        user.getEmail()
                );

        return new LoginResponse(
                accessToken,
                null,
                900,
                "Bearer"
        );
    }
}
```

This is the starting implementation.

Later iterations will add:

- refresh token rotation
- Redis rate limiting
- account lockout
- Kafka security events
- logout/token revocation
- token rotation
- audit events
- security controls

---

# 20. Controller

## `AuthController.java`

```java
package com.company.auth.controller;

import com.company.auth.dto.LoginRequest;
import com.company.auth.dto.LoginResponse;
import com.company.auth.service.AuthService;

import jakarta.validation.Valid;

import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/auth")
public class AuthController {

    private final AuthService authService;

    public AuthController(AuthService authService) {
        this.authService = authService;
    }

    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(
            @Valid @RequestBody LoginRequest request) {

        return ResponseEntity.ok(
                authService.login(request)
        );
    }

    @GetMapping("/health")
    public ResponseEntity<String> health() {

        return ResponseEntity.ok("AUTH_SERVICE_OK");
    }
}
```

---

# 21. Application Configuration

## `application.yaml`

```yaml
server:
  port: 8080

spring:
  application:
    name: auth-service

  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:5432/auth}
    username: ${DB_USERNAME:postgres}
    password: ${DB_PASSWORD:postgres}

    hikari:
      maximum-pool-size: 10
      minimum-idle: 2
      connection-timeout: 3000
      validation-timeout: 1000
      idle-timeout: 600000
      max-lifetime: 1500000
      pool-name: auth-service-hikari

  jpa:
    hibernate:
      ddl-auto: validate

    show-sql: false

    open-in-view: false

    properties:
      hibernate:
        format_sql: false

  flyway:
    enabled: true

  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}

  kafka:
    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS:localhost:9092}

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

  health:
    livenessstate:
      enabled: true

    readinessstate:
      enabled: true

security:

  jwt:
    secret: ${JWT_SECRET:change-me-in-local-only}
    expiration-ms: 900000

logging:

  level:
    root: INFO

    com.company.auth: INFO
```

---

# 22. Production Configuration

## `application-prod.yaml`

```yaml
server:
  port: 8080

spring:

  application:
    name: auth-service

  datasource:

    url: ${DB_URL}

    username: ${DB_USERNAME}

    password: ${DB_PASSWORD}

    hikari:

      maximum-pool-size: ${DB_POOL_MAX_SIZE:10}

      minimum-idle: ${DB_POOL_MIN_IDLE:2}

      connection-timeout: ${DB_CONNECTION_TIMEOUT:3000}

      validation-timeout: ${DB_VALIDATION_TIMEOUT:1000}

      idle-timeout: ${DB_IDLE_TIMEOUT:600000}

      max-lifetime: ${DB_MAX_LIFETIME:1500000}

      pool-name: auth-service-hikari

  jpa:

    hibernate:
      ddl-auto: validate

    show-sql: false

    open-in-view: false

  flyway:
    enabled: true

  data:

    redis:
      host: ${REDIS_HOST}
      port: ${REDIS_PORT:6379}

  kafka:

    bootstrap-servers: ${KAFKA_BOOTSTRAP_SERVERS}

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

  health:

    livenessstate:
      enabled: true

    readinessstate:
      enabled: true

security:

  jwt:

    secret: ${JWT_SECRET}

    expiration-ms: ${JWT_EXPIRATION_MS:900000}

logging:

  level:

    root: INFO

    com.company.auth: INFO
```

The Hikari values are **illustrative starting values**. They must be capacity-tested against the actual PostgreSQL connection budget.

---

# 23. Dockerfile

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

COPY --from=builder /build/target/*.jar app.jar

RUN groupadd --system appgroup \
    && useradd --system --gid appgroup appuser

USER appuser

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

---

# 24. `.dockerignore`

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

docker-compose.yaml

node_modules
```

---

# 25. Local Docker Compose

## `docker-compose.yaml`

```yaml
services:

  postgres:

    image: postgres:16

    container_name: auth-postgres

    restart: unless-stopped

    environment:

      POSTGRES_DB: auth

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
          "pg_isready -U postgres -d auth"
        ]

      interval: 10s

      timeout: 5s

      retries: 5

    networks:
      - auth-network


  redis:

    image: redis:7

    container_name: auth-redis

    restart: unless-stopped

    ports:

      - "6379:6379"

    networks:
      - auth-network


  kafka:

    image: apache/kafka:latest

    container_name: auth-kafka

    restart: unless-stopped

    ports:

      - "9092:9092"

    networks:
      - auth-network


  auth-service:

    build:

      context: .

      dockerfile: Dockerfile

    container_name: auth-service

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

      DB_URL: jdbc:postgresql://postgres:5432/auth

      DB_USERNAME: postgres

      DB_PASSWORD: postgres

      REDIS_HOST: redis

      REDIS_PORT: 6379

      KAFKA_BOOTSTRAP_SERVERS: kafka:9092

      JWT_SECRET: local-only-secret-change-this-value-very-long

    ports:

      - "8080:8080"

    networks:
      - auth-network


volumes:

  postgres-data:


networks:

  auth-network:

    driver: bridge
```

`latest` is shown here only as a convenient local-development example for Kafka. Production images will be pinned to approved immutable versions.

---

# 26. Local Build

```bash
./mvnw clean test
```

Package:

```bash
./mvnw clean package
```

Run:

```bash
java -jar target/auth-service-1.0.0.jar
```

---

# 27. Start Local Environment

```bash
docker compose up -d
```

Check:

```bash
docker compose ps
```

Logs:

```bash
docker compose logs -f auth-service
```

---

# 28. Health Check

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

# 29. Prometheus Endpoint

```bash
curl http://localhost:8080/actuator/prometheus
```

This endpoint will later be scraped by Prometheus through a Kubernetes `ServiceMonitor`.

---

# 30. Kubernetes Namespace

The production platform uses a namespace such as:

```text
ecommerce
```

---

# 31. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: auth-service

description: Authentication service for the ecommerce platform

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 32. Helm Values

## `values.yaml`

```yaml
replicaCount: 2

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/auth-service

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

  maxReplicas: 10

  targetCPUUtilizationPercentage: 60


podDisruptionBudget:

  enabled: true

  minAvailable: 1


serviceAccount:

  create: true

  name: auth-service

  annotations: {}


config:

  SPRING_PROFILES_ACTIVE: prod


secretStore:

  name: ecommerce-secrets


probes:

  startup:

    path: /actuator/health/liveness

  readiness:

    path: /actuator/health/readiness

  liveness:

    path: /actuator/health/liveness
```

---

# 33. Production Values

## `values-prod.yaml`

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/auth-service

  tag: ""

  digest: ""


resources:

  requests:

    cpu: 500m

    memory: 768Mi

  limits:

    cpu: "1"

    memory: 1Gi


autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 15

  targetCPUUtilizationPercentage: 60


podDisruptionBudget:

  enabled: true

  minAvailable: 2
```

These values are starting points for the project simulation, not blindly accepted production capacity values.

---

# 34. Deployment

## `templates/deployment.yaml`

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: {{ include "auth-service.fullname" . }}

  labels:
    app: auth-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:
      app: auth-service

  template:

    metadata:

      labels:
        app: auth-service

    spec:

      serviceAccountName: {{ .Values.serviceAccount.name }}

      terminationGracePeriodSeconds: 30

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: auth-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: {{ .Values.image.pullPolicy }}

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: auth-service-config

            - secretRef:

                name: auth-service-secrets

          resources:

            {{- toYaml .Values.resources | nindent 12 }}

          startupProbe:

            httpGet:

              path: {{ .Values.probes.startup.path }}

              port: http

            failureThreshold: 30

            periodSeconds: 10

          readinessProbe:

            httpGet:

              path: {{ .Values.probes.readiness.path }}

              port: http

            initialDelaySeconds: 10

            periodSeconds: 10

          livenessProbe:

            httpGet:

              path: {{ .Values.probes.liveness.path }}

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

# 35. Kubernetes Service

## `templates/service.yaml`

```yaml
apiVersion: v1

kind: Service

metadata:

  name: auth-service

  labels:
    app: auth-service

spec:

  type: ClusterIP

  selector:

    app: auth-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

Other microservices can call:

```text
http://auth-service.ecommerce.svc.cluster.local:8080
```

---

# 36. ConfigMap

## `templates/configmap.yaml`

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: auth-service-config

data:

  SPRING_PROFILES_ACTIVE: prod

  DB_POOL_MAX_SIZE: "10"

  DB_POOL_MIN_IDLE: "2"

  REDIS_PORT: "6379"

  JWT_EXPIRATION_MS: "900000"
```

No passwords are stored here.

---

# 37. External Secrets

## `templates/externalsecret.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: auth-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: {{ .Values.secretStore.name }}

    kind: ClusterSecretStore

  target:

    name: auth-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DB_URL

      remoteRef:

        key: ecommerce/prod/auth-service

        property: DB_URL

    - secretKey: DB_USERNAME

      remoteRef:

        key: ecommerce/prod/auth-service

        property: DB_USERNAME

    - secretKey: DB_PASSWORD

      remoteRef:

        key: ecommerce/prod/auth-service

        property: DB_PASSWORD

    - secretKey: JWT_SECRET

      remoteRef:

        key: ecommerce/prod/auth-service

        property: JWT_SECRET
```

Architecture:

```text
AWS Secrets Manager
        ↓
External Secrets Operator
        ↓
Kubernetes Secret
        ↓
auth-service Pod
```

---

# 38. ServiceAccount

## `templates/serviceaccount.yaml`

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: auth-service

  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/ecommerce-auth-service-role
```

The exact Pod Identity mechanism should follow the EKS identity architecture selected for the platform. We should not rely on static AWS access keys.

---

# 39. IAM Principle

Auth service should receive only the AWS permissions it actually requires.

For example:

```text
auth-service Pod
      ↓
EKS Pod Identity
      ↓
auth-service IAM role
      ↓
Secrets Manager
```

Potential permissions:

```text
secretsmanager:GetSecretValue
```

restricted to the auth service secret.

Do NOT give:

```text
AdministratorAccess
```

to the application.

---

# 40. NetworkPolicy

## `templates/networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: auth-service

spec:

  podSelector:

    matchLabels:

      app: auth-service

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

In a mature production implementation, egress rules should be tightened further to specific destinations and ports.

---

# 41. HPA

## `templates/hpa.yaml`

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: auth-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: auth-service

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

          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
```

For authentication workloads, later we can consider request-rate or latency/custom metrics if CPU alone doesn't represent demand.

---

# 42. PodDisruptionBudget

## `templates/pdb.yaml`

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: auth-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: auth-service
```

---

# 43. Prometheus ServiceMonitor

## `templates/servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: auth-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: auth-service

  endpoints:

    - port: http

      path: /actuator/prometheus

      interval: 15s
```

---

# 44. GitHub CODEOWNERS

## `.github/CODEOWNERS`

```text
/src/       @company-ecommerce/backend-team
/helm/      @company-ecommerce/platform-team
/.github/   @company-ecommerce/platform-team
/security/  @company-ecommerce/security-team
```

---

# 45. CI Pipeline

## `.github/workflows/ci.yaml`

```yaml
name: Auth Service CI

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

  ECR_REPOSITORY: auth-service


jobs:

  build-test:

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


      - name: Build and Test
        run: |
          chmod +x mvnw
          ./mvnw clean verify


      - name: SonarQube
        env:

          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

          SONAR_HOST_URL: ${{ secrets.SONAR_HOST_URL }}

        run: |
          ./mvnw sonar:sonar \
            -Dsonar.host.url="$SONAR_HOST_URL" \
            -Dsonar.token="$SONAR_TOKEN"


  security:

    runs-on: ubuntu-latest

    steps:

      - name: Checkout
        uses: actions/checkout@v4


      - name: Trivy Filesystem Scan
        uses: aquasecurity/trivy-action@master

        with:

          scan-type: fs

          scan-ref: .

          severity: HIGH,CRITICAL

          ignore-unfixed: true


  docker:

    needs:
      - build-test
      - security

    if: github.event_name == 'push'

    runs-on: ubuntu-latest

    steps:

      - name: Checkout
        uses: actions/checkout@v4


      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4

        with:

          role-to-assume: ${{ secrets.AWS_CI_ROLE_ARN }}

          aws-region: ${{ env.AWS_REGION }}


      - name: Login to ECR
        id: ecr

        uses: aws-actions/amazon-ecr-login@v2


      - name: Build Docker Image
        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

          IMAGE_TAG: ${{ github.sha }}

        run: |
          docker build \
            -t "$REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" \
            .


      - name: Trivy Image Scan
        uses: aquasecurity/trivy-action@master

        with:

          image-ref: ${{ steps.ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

          severity: HIGH,CRITICAL

          exit-code: "1"

          ignore-unfixed: true


      - name: Push Image
        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |
          docker push \
            "$REGISTRY/$ECR_REPOSITORY:${{ github.sha }}"


      - name: Get Image Digest
        env:

          REGISTRY: ${{ steps.ecr.outputs.registry }}

        run: |
          aws ecr describe-images \
            --repository-name "$ECR_REPOSITORY" \
            --image-ids imageTag="${GITHUB_SHA}" \
            --query 'imageDetails[0].imageDigest' \
            --output text
```

This pipeline intentionally separates:

```text
PR
 ↓
Build/Test/Security
```

from:

```text
main
 ↓
Docker
 ↓
Trivy
 ↓
ECR
```

---

# 46. CI Security Model

GitHub does not receive permanent AWS credentials.

The flow is:

```text
GitHub Actions
      |
      | OIDC
      ↓
AWS STS
      |
      ↓
IAM CI Role
      |
      ↓
ECR
```

GitHub permissions:

```yaml
permissions:
  contents: read
  id-token: write
```

The IAM trust policy is restricted to the correct repository/branch.

---

# 47. Image Tagging

Do not use:

```text
auth-service:latest
```

Use:

```text
auth-service:<git-sha>
```

Example:

```text
auth-service:7d9a1c4
```

Even better, production GitOps can deploy by digest:

```text
auth-service@sha256:abc123...
```

This gives us:

```text
Git commit
    ↓
CI
    ↓
Image
    ↓
ECR digest
    ↓
GitOps
    ↓
ArgoCD
    ↓
EKS
```

---

# 48. GitOps

The application repository builds the image.

The GitOps repository controls deployment.

Example:

```text
gitops/
└── apps/
    └── auth-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Production deployment:

```text
ECR
 ↓
GitOps PR
 ↓
Approval
 ↓
Merge
 ↓
ArgoCD detects change
 ↓
Sync
 ↓
EKS
```

CI should not directly execute:

```bash
kubectl apply
```

against production.

---

# 49. ArgoCD Application

Example:

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: auth-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/auth-service/prod

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

For production, automatic sync policy should follow the organization's change-management policy. Some organizations require an explicit production approval before synchronization.

---

# 50. Redis Responsibilities

Redis is used for short-lived authentication-related state such as:

```text
Login attempt counters
Rate limiting
Session metadata
Token revocation state
Temporary password reset state
```

Example:

```text
auth:login:attempts:<user>
auth:login:lock:<user>
auth:refresh:<token-id>
auth:revoked:<token-id>
```

Never store:

```text
plaintext password
```

in Redis.

---

# 51. PostgreSQL Is the Source of Truth

Redis should not become the authoritative credential store.

```text
PostgreSQL
    |
    +-- Durable authentication data

Redis
    |
    +-- Temporary state/cache/rate limiting
```

If Redis fails:

```text
Redis failure
     ↓
Authentication should degrade gracefully
     ↓
Protect PostgreSQL
     ↓
Rate limiting may temporarily use another mechanism
```

We must not allow Redis failure to cause uncontrolled database traffic.

---

# 52. Authentication Events

Auth service can publish events such as:

```text
UserAuthenticated
LoginFailed
UserLocked
PasswordChanged
PasswordResetRequested
UserLoggedOut
RefreshTokenRotated
```

Example event:

```json
{
  "eventId": "01J...",
  "eventType": "UserAuthenticated",
  "version": 1,
  "occurredAt": "2026-10-05T10:30:00Z",
  "source": "auth-service",
  "correlationId": "abc-123",
  "payload": {
    "userId": "..."
  }
}
```

Do not put passwords, access tokens, refresh tokens, or sensitive authentication material into Kafka events.

---

# 53. Security Rules

The auth-service must never log:

```text
password
Authorization header
access token
refresh token
JWT secret
database password
```

Bad:

```java
log.info("Login request: {}", request);
```

because the request contains a password.

Good:

```java
log.info(
    "Authentication attempt for user={}",
    request.email()
);
```

Even email addresses may be considered personal data depending on the organization's policy, so production logging should follow the company's PII policy.

---

# 54. Prometheus Metrics

Important metrics:

```text
http_server_requests_seconds
jvm_memory_used_bytes
jvm_threads_live_threads
hikaricp_connections_active
hikaricp_connections_idle
hikaricp_connections_pending
hikaricp_connections_max
```

Application-specific metrics should include:

```text
auth_login_success_total
auth_login_failure_total
auth_token_issued_total
auth_refresh_total
auth_logout_total
auth_account_locked_total
```

Avoid high-cardinality labels such as:

```text
user_id
email
token
session_id
```

---

# 55. Grafana Dashboard

Auth dashboard:

```text
Authentication Requests
        |
        +-- RPS
        +-- P50
        +-- P95
        +-- P99
        +-- 4xx
        +-- 5xx

Authentication
        |
        +-- Successful logins
        +-- Failed logins
        +-- Locked accounts

Database
        |
        +-- RDS connections
        +-- Hikari active
        +-- Hikari pending
        +-- DB latency

Redis
        |
        +-- latency
        +-- connections
        +-- hit/miss
        +-- errors

Kubernetes
        |
        +-- replicas
        +-- CPU
        +-- memory
        +-- restarts
```

---

# 56. Logging

Example structured log:

```json
{
  "@timestamp": "2026-10-05T10:30:00Z",
  "level": "INFO",
  "service": "auth-service",
  "environment": "prod",
  "namespace": "ecommerce",
  "pod": "auth-service-7d9c8f7b6d-x7abc",
  "version": "1.0.0",
  "correlationId": "abc-123",
  "message": "Authentication completed successfully"
}
```

Never include:

```text
password
accessToken
refreshToken
Authorization
JWT secret
```

---

# 57. Distributed Tracing

Architecture:

```text
Client
  ↓
ALB
  ↓
auth-service
  |
  ├── PostgreSQL
  |
  ├── Redis
  |
  └── Kafka
```

OpenTelemetry creates:

```text
Trace
 |
 +-- HTTP request
 |
 +-- PostgreSQL query
 |
 +-- Redis operation
 |
 +-- Kafka publish
```

Flow:

```text
auth-service
      ↓
OpenTelemetry Java Agent
      ↓
OTel Collector
      ↓
Jaeger
```

---

# 58. Correlation ID

Every request should have:

```text
X-Correlation-ID
```

If the client provides a valid ID:

```text
use it
```

Otherwise:

```text
generate one
```

Then propagate it through:

```text
auth-service
      ↓
user-service
      ↓
Kafka
      ↓
notification-service
```

This allows a production incident to be followed across multiple services.

---

# 59. SLO

Example starting SLOs:

### Authentication Availability

```text
99.9%
```

### Login Latency

```text
P95 < 300ms
```

### Token Issuance

```text
99.95% successful eligible requests
```

The exact values must be based on measured service behavior and business requirements.

Important:

A failed login because of an incorrect password is not necessarily a service availability failure.

We need to distinguish:

```text
Expected business rejection
```

from:

```text
System failure
```

---

# 60. Failure Scenarios

## Scenario 1 — PostgreSQL unavailable

```text
Login request
    ↓
PostgreSQL connection failure
    ↓
Authentication fails
```

Actions:

```text
Check RDS
Check DNS
Check SG
Check network
Check Hikari
Check RDS connections
Check recent deployment
```

---

## Scenario 2 — Redis unavailable

Potential impact:

```text
Rate limiting
Session state
Revocation
```

The application must have controlled fallback behavior.

Do not blindly retry Redis hundreds of times.

Use:

```text
short timeout
+
limited retry
+
backoff
+
circuit breaker
```

---

## Scenario 3 — Kafka unavailable

Authentication itself should not necessarily fail just because a noncritical security event cannot be published.

Depending on business requirements:

```text
Authentication
      |
      +-- critical
      |
      +-- audit event
```

Use an appropriate durability pattern such as an outbox where event delivery is business-critical.

---

## Scenario 4 — JWT signing key unavailable

This is a critical failure.

Do not generate tokens using a fallback hardcoded secret.

Instead:

```text
Fail safely
Alert
Investigate Secrets Manager / ESO / IAM / KMS
Restore trusted signing configuration
```

---

## Scenario 5 — Brute-force attack

Possible symptoms:

```text
Login failures ↑
Request rate ↑
WAF alerts
Redis rate-limit counters ↑
CPU ↑
```

Response:

```text
WAF
 ↓
Rate limiting
 ↓
Application protection
 ↓
Account protection
 ↓
Security investigation
```

---

# 61. Production Troubleshooting

## Check Pods

```bash
kubectl get pods -n ecommerce -l app=auth-service
```

## Check deployment

```bash
kubectl get deployment auth-service -n ecommerce
```

## Describe Pod

```bash
kubectl describe pod <pod-name> -n ecommerce
```

## Logs

```bash
kubectl logs <pod-name> -n ecommerce
```

## Previous container

```bash
kubectl logs <pod-name> -n ecommerce --previous
```

## Check service

```bash
kubectl get svc auth-service -n ecommerce
```

## Check endpoints

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=auth-service
```

## Check HPA

```bash
kubectl get hpa auth-service -n ecommerce
```

## Check rollout

```bash
kubectl rollout status \
  deployment/auth-service \
  -n ecommerce
```

---

# 62. Authentication Debugging Flow

```text
Login failing
      |
      ↓
Is auth Pod healthy?
      |
      +-- NO → Kubernetes/application investigation
      |
      +-- YES
           |
           ↓
Can Pod reach PostgreSQL?
           |
           +-- NO → DNS/network/SG/RDS
           |
           +-- YES
                |
                ↓
Does user exist?
                |
                +-- NO → expected authentication failure
                |
                +-- YES
                     |
                     ↓
Password verification?
                     |
                     ↓
JWT signing?
                     |
                     ↓
Redis?
                     |
                     ↓
Kafka/audit?
```

---

# 63. Example Production Incident

## Incident

```text
INC-2026-AUTH-001
```

### Title

```text
Auth Login Latency Increased Due to Redis Connection Exhaustion
```

### Severity

```text
SEV-2
```

### Impact

Users experienced:

```text
Login latency
Authentication failures
HTTP 5xx
```

No permanent credential data was lost.

---

# 64. Incident Timeline

```text
10:00
Normal authentication traffic.

10:05
Redis connection count begins increasing.

10:07
Redis latency increases.

10:08
auth-service Redis operations start timing out.

10:09
Application retries Redis operations.

10:10
CPU increases.

10:11
HPA scales auth-service from 3 → 8 Pods.

10:12
More Pods create more Redis connections.

10:13
Redis connection exhaustion occurs.

10:14
Authentication P95 increases from 220ms → 3.8s.

10:15
HTTP 5xx reaches 5%.

10:16
SEV-2 declared.

10:18
Retry behavior reduced.

10:19
Auth replicas stabilized.

10:21
Redis connections recover.

10:23
Authentication latency returns to normal.

10:25
Incident resolved.
```

---

# 65. Root Cause

Primary root cause:

```text
Redis connection capacity was exceeded.
```

Amplification mechanism:

```text
Redis latency
    ↓
Application retries
    ↓
More Redis operations
    ↓
HPA scaling
    ↓
More Pods
    ↓
More Redis connections
    ↓
Redis connection exhaustion
    ↓
Authentication latency
```

The actual root cause was not simply:

```text
"Redis was slow."
```

The production RCA must identify the **amplification mechanism**.

---

# 66. Five Whys

### Why did authentication become slow?

Redis operations timed out.

### Why?

Redis connection capacity was exhausted.

### Why?

Too many authentication Pods and Redis retries created excessive connections.

### Why?

The application retry policy was too aggressive and HPA scaled based on CPU caused by retry activity.

### Why?

Redis dependency limits and retry/HPA interaction had not been included in capacity testing.

---

# 67. Corrective Actions

| Action | Owner |
|---|---|
| Reduce Redis retry count | Backend |
| Add exponential backoff | Backend |
| Add circuit breaker | Backend |
| Define Redis connection budget | Platform |
| Tune HPA | Platform |
| Add Redis connection alerts | SRE |
| Add Redis failure load test | Performance |
| Review Redis client configuration | Backend |
| Add dependency SLO | SRE |
| Add incident runbook | SRE |

---

# 68. CI/CD Production Flow

The complete auth-service lifecycle is:

```text
Developer
   |
   ↓
Feature Branch
   |
   ↓
Pull Request
   |
   ↓
GitHub Actions
   |
   ├── Maven
   ├── Unit Tests
   ├── SonarQube
   ├── Dependency Scan
   ├── Secret Scan
   ├── Docker Build
   └── Trivy
   |
   ↓
ECR
   |
   ↓
GitOps Repository
   |
   ↓
ArgoCD
   |
   ↓
EKS
   |
   ↓
Prometheus/Grafana
   |
   ↓
ELK
   |
   ↓
OpenTelemetry/Jaeger
```

---

# 69. Production Checklist

## Application

- [ ] Authentication implemented
- [ ] Passwords hashed
- [ ] No plaintext passwords
- [ ] Token expiration configured
- [ ] Refresh-token rotation implemented
- [ ] Logout/revocation implemented
- [ ] Account protection implemented
- [ ] Validation implemented
- [ ] Exception handling implemented

## Database

- [ ] PostgreSQL private
- [ ] Separate application user
- [ ] Flyway migrations
- [ ] `ddl-auto=validate`
- [ ] Hikari configured
- [ ] Connection budget calculated
- [ ] Backups enabled
- [ ] PITR tested

## Redis

- [ ] Private networking
- [ ] Encryption
- [ ] Authentication
- [ ] Timeout configured
- [ ] Retry bounded
- [ ] Failure fallback designed
- [ ] Connection limits understood

## Kubernetes

- [ ] Deployment
- [ ] Service
- [ ] ConfigMap
- [ ] ExternalSecret
- [ ] ServiceAccount
- [ ] IAM
- [ ] NetworkPolicy
- [ ] Resource requests
- [ ] Resource limits
- [ ] Startup probe
- [ ] Readiness probe
- [ ] Liveness probe
- [ ] HPA
- [ ] PDB

## CI/CD

- [ ] Maven build
- [ ] Unit tests
- [ ] SonarQube
- [ ] Dependency scan
- [ ] Secret scan
- [ ] Trivy
- [ ] Docker build
- [ ] ECR
- [ ] Immutable image
- [ ] GitHub OIDC

## GitOps

- [ ] Helm chart
- [ ] Environment values
- [ ] ArgoCD Application
- [ ] Production approval
- [ ] Rollback strategy
- [ ] Image digest traceability

## Observability

- [ ] Prometheus
- [ ] Grafana
- [ ] Structured logs
- [ ] ELK
- [ ] Correlation ID
- [ ] OpenTelemetry
- [ ] Jaeger
- [ ] Alerts
- [ ] SLO
- [ ] Runbook

## Security

- [ ] No secrets in Git
- [ ] Least-privilege IAM
- [ ] Pod Identity
- [ ] NetworkPolicy
- [ ] Non-root container
- [ ] Read-only filesystem where compatible
- [ ] Security scanning
- [ ] No sensitive authentication data in logs
- [ ] Signing-key rotation strategy
- [ ] Audit trail

---

# 70. Final Architecture

```text
                           Internet
                              |
                       CloudFront / WAF
                              |
                             ALB
                              |
                       auth-service
                              |
              ┌───────────────┼────────────────┐
              │               │                │
              ↓               ↓                ↓
          PostgreSQL        Redis             Kafka
              │               │                │
              │               │                │
              └───────────────┼────────────────┘
                              |
                       Authentication
                              |
                   ┌──────────┴──────────┐
                   │                     │
              Prometheus             OpenTelemetry
                   │                     │
                Grafana               Collector
                                         |
                                       Jaeger

Application logs
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
```

---

# 71. Key DevOps Learning From Auth Service

This service demonstrates that DevOps is not just:

```text
Docker + Kubernetes
```

You need to understand the complete chain:

```text
Source Code
    ↓
Maven
    ↓
Tests
    ↓
SonarQube
    ↓
Security Scan
    ↓
Docker
    ↓
Trivy
    ↓
ECR
    ↓
GitOps
    ↓
ArgoCD
    ↓
EKS
    ↓
IAM
    ↓
Secrets
    ↓
PostgreSQL
    ↓
Redis
    ↓
Kafka
    ↓
HPA
    ↓
Prometheus
    ↓
Grafana
    ↓
ELK
    ↓
OpenTelemetry
    ↓
Jaeger
    ↓
SLO
    ↓
Incident Response
    ↓
RCA
```

The same platform principles will be reused across the remaining services, but **the application language, framework, dependencies, database usage, scaling mechanism, messaging pattern, and failure modes will change according to the business responsibility of each service.**

# End of 161 — Auth Service
```

**Phase 19 progress:**

```text
161 Auth Service          ✅
162 User Service          ⏳
163 Customer Service      ⏳
164 Catalog Service       ⏳
165 Product Service       ⏳
166 Inventory Service     ⏳
167 Pricing Service       ⏳
168 Promotion Service     ⏳
169 Cart Service          ⏳
170 Order Service         ⏳
171 Payment Service       ⏳
172 Fraud Service         ⏳
173 Shipping Service      ⏳
174 Delivery Service      ⏳
175 Notification Service  ⏳
176 Search Service        ⏳
177 Recommendation        ⏳
178 Review Service        ⏳
179 Tax Service           ⏳
180 Invoice Service       ⏳
181 Reporting Service     ⏳
```

**Next: `162-User-Service.md` — Python + FastAPI**, so we start applying the polyglot approach rather than making every service Java/Spring Boot.