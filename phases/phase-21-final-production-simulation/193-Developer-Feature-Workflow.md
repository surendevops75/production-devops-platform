We will now upgrade Chapter 193 into the complete version. This replaces the previous Chapter 193; it is not an additional chapter.

The goal is to cover the developer's actual responsibilities from receiving a Jira requirement through implementation, testing, local validation, security checks, documentation and handing the feature over for Pull Request review.

I will distinguish production implementation patterns from illustrative examples wherever our existing application details have not been established.

# PHASE 21 — FINAL PRODUCTION SIMULATION

# CHAPTER 193 — DEVELOPER FEATURE WORKFLOW

Filename: `193-Developer-Feature-Workflow.md`

## 1. Objective

In this chapter, we simulate how a developer implements a new feature in an existing enterprise microservices application.

We will use the following scenario throughout the chapter:

Business requirement: Implement an Order Tracking API in the existing `order-service`.

The application is an enterprise e-commerce platform consisting of 21 microservices deployed on Amazon EKS.

Our technology stack:

| Category             | Technology                   |
| -------------------- | ---------------------------- |
| Application          | Java, Spring Boot            |
| Build                | Maven                        |
| Source control       | GitHub                       |
| Containerization     | Docker                       |
| Container registry   | Amazon ECR                   |
| Orchestration        | Amazon EKS                   |
| IaC                  | Terraform                    |
| CI/CD                | GitHub Actions, GitLab CI/CD |
| GitOps               | ArgoCD                       |
| Packaging            | Helm                         |
| Code quality         | SonarQube                    |
| Image security       | Trivy                        |
| Application security | Veracode                     |
| Monitoring           | Prometheus, Grafana          |
| Logging              | ELK Stack with Logstash      |
| Project management   | Jira                         |

By completing this chapter, you should understand how developers work inside a production DevOps environment.

### 1.1 Complete developer workflow

```
Business Requirement
        |
        v
Jira Story Creation
        |
        v
Requirement Analysis
        |
        v
Acceptance Criteria
        |
        v
Technical Design
        |
        v
Repository Analysis
        |
        v
Local Environment Setup
        |
        v
Feature Branch Creation
        |
        v
Application Development
        |
        v
Unit Testing
        |
        v
Integration Testing
        |
        v
Security Validation
        |
        v
Local Docker Validation
        |
        v
Code Quality Validation
        |
        v
Git Commit
        |
        v
Push Feature Branch
        |
        v
Developer Handoff
        |
        v
Chapter 194: Pull Request Workflow
```

# PART 1 — UNDERSTANDING THE BUSINESS REQUIREMENT

## 2. Production scenario

Imagine that our e-commerce company currently allows customers to place orders.

However, customers cannot retrieve detailed shipment tracking information through the Order Service API.

The business team wants to introduce a new endpoint.

```
GET /api/v1/orders/{orderId}/tracking
```

Example:

```
GET /api/v1/orders/ORD-10025/tracking
```

Expected response:

```
{
  "orderId": "ORD-10025",
  "status": "SHIPPED",
  "trackingNumber": "TRACK-987654",
  "carrier": "Example Logistics",
  "estimatedDelivery": "2026-11-05",
  "lastUpdated": "2026-11-01T09:30:00Z"
}
```

This response is illustrative. Before implementing it, the developer must establish which service owns the tracking information and which fields are part of the approved API contract.

## 3. Jira story creation

The business team creates a Jira issue.

ECOM-4821

To Do

## Implement Order Tracking API

Enterprise E-commerce Platform

Description

Customers should be able to retrieve the latest tracking information for their orders.

Acceptance criteria

- Customer can retrieve tracking information.
- API returns the correct order status.
- API returns tracking details when available.
- API handles orders without tracking information.
- Invalid order IDs are handled correctly.
- Customers cannot access other customers' orders.
- Existing order APIs remain compatible.

Priority: High

Component: order-service

Example assignee: Order Service Developer

### 3.1 What should the developer understand?

Before writing code, the developer should understand:

1. What business problem are we solving?
2. Which microservice owns the functionality?
3. What API should be exposed?
4. Where is the data stored?
5. Does another microservice provide the data?
6. What authentication is required?
7. What authorization rules apply?
8. Are database changes required?
9. Will existing APIs be affected?
10. What tests are required?

A developer should not start coding based on an unclear requirement.

## 4. Acceptance criteria and Definition of Done

Acceptance criteria describe expected application behavior.

| ID    | Acceptance criteria                       | Validation       |
| ----- | ----------------------------------------- | ---------------- |
| AC-01 | Retrieve tracking for an authorized order | API test         |
| AC-02 | Return correct order status               | Service test     |
| AC-03 | Return available tracking information     | Integration test |
| AC-04 | Handle unavailable tracking               | Negative test    |
| AC-05 | Handle nonexistent order                  | API test         |
| AC-06 | Prevent unauthorized order access         | Security test    |
| AC-07 | Maintain API response contract            | Contract test    |
| AC-08 | Preserve existing API behavior            | Regression test  |

### 4.1 Definition of Ready

A story is ready for development when:

- Requirement is understandable.
- Acceptance criteria are documented.
- API contract is agreed.
- Data ownership is identified.
- Dependencies are identified.
- Security requirements are understood.
- Test expectations are known.
- Required design decisions are available.

### 4.2 Definition of Done

A feature is not done merely because the developer has written code.

For our feature, the Definition of Done should include:

- Application code implemented.
- Unit tests completed.
- Integration tests completed where applicable.
- Security behavior validated.
- API compatibility verified.
- Code quality checks completed.
- Documentation updated.
- No known critical security issue left unresolved.
- Changes committed and pushed.
- Pull Request created and reviewed.
- Required CI checks passed.
- QA handoff completed according to team process.

The feature is not production-ready until the later release and approval stages are completed.

## 5. Non-functional requirements

The developer must consider more than whether the API returns the correct JSON.

| Area            | Consideration                                 |
| --------------- | --------------------------------------------- |
| Performance     | Avoid unnecessary database and service calls. |
| Security        | Validate authenticated customer ownership.    |
| Reliability     | Handle dependency failures safely.            |
| Availability    | Avoid disrupting existing APIs.               |
| Maintainability | Follow existing application architecture.     |
| Observability   | Produce useful logs and metrics.              |
| Compatibility   | Preserve existing API contracts.              |
| Scalability     | Avoid unnecessary synchronous work.           |

Actual latency, availability and throughput targets must be obtained from the existing service SLOs.

# PART 2 — UNDERSTANDING THE APPLICATION ARCHITECTURE

## 6. Understand the 21-microservice platform

Our application consists of 21 microservices.

The developer must identify the correct service before implementing a feature.

Example:

```
Enterprise E-commerce Platform
│
├── user-service
├── product-service
├── cart-service
├── order-service
├── payment-service
├── inventory-service
├── notification-service
├── shipping-service
├── review-service
├── pricing-service
├── promotion-service
├── search-service
├── recommendation-service
├── delivery-service
├── return-service
├── refund-service
├── invoice-service
├── warehouse-service
├── customer-service
├── authentication-service
└── reporting-service
```

This is an illustrative service inventory, not a claim that all these exact service names or integrations already exist.

### 6.1 Why is service ownership important?

Suppose the tracking number is managed by a shipping service.

The developer should not duplicate shipping functionality inside the Order Service.

Instead, the developer should determine whether the Order Service:

- Already stores a copy of tracking information.
- Retrieves tracking information through an internal API.
- Consumes tracking events.
- Uses another approved integration.

The correct architecture depends on the existing platform design.

### 6.2 Understand synchronous and asynchronous communication

Synchronous communication

Example:

```
Frontend
   |
   v
Order Service
   |
   v
Shipping Service
   |
   v
Tracking Response
```

The Order Service requests data and waits for a response.

Considerations:

- Connection timeout.
- Response timeout.
- Dependency availability.
- Retry policy.
- Circuit breaking, if the platform uses it.
- Fallback behavior.
- Authentication between services.

Asynchronous communication

Example:

```
Shipping Service
      |
      v
Tracking Event
      |
      v
Messaging Platform
      |
      v
Order Service Consumer
      |
      v
Order Database
```

The Order Service stores or updates tracking state from incoming events.

Considerations:

- Duplicate messages.
- Event ordering.
- Retry and dead-letter handling.
- Idempotency.
- Event schema compatibility.
- Consumer lag.

Do not introduce a new Kafka or RabbitMQ dependency simply because it is available in the platform. Use the approved existing integration.

## 7. Understand the existing Order Service

Before making changes, inspect the repository.

Example structure:

```
order-service/
│
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/company/order/
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       ├── repository/
│   │   │       ├── entity/
│   │   │       ├── dto/
│   │   │       ├── config/
│   │   │       ├── exception/
│   │   │       └── integration/
│   │   │
│   │   └── resources/
│   │       ├── application.yml
│   │       └── db/migration/
│   │
│   └── test/
│       └── java/com/company/order/
│
├── pom.xml
├── mvnw
├── mvnw.cmd
├── Dockerfile
├── helm/
├── .github/
│   └── workflows/
└── README.md
```

### 7.1 Clone the repository

```
git clone https://github.com/company-ecommerce/order-service.git
```

Move into the repository:

```
cd order-service
```

Check the current directory:

```
pwd
```

List the files:

```
ls -la
```

Check Git status:

```
git status
```

Expected clean working tree:

```
On branch main
Your branch is up to date with 'origin/main'.

nothing to commit, working tree clean
```

### 7.2 Inspect the Java source

Find controllers:

```
find src/main/java -type f | grep controller
```

Find services:

```
find src/main/java -type f | grep service
```

Find repositories:

```
find src/main/java -type f | grep repository
```

Find configuration:

```
find src/main/resources -type f
```

Search for existing order APIs:

```
grep -R "@GetMapping\|@PostMapping\|@PutMapping" src/main/java
```

Search for existing exception handlers:

```
grep -R "@ControllerAdvice\|@ExceptionHandler" src/main/java
```

### 7.3 Inspect Maven configuration

```
cat pom.xml
```

Identify:

- Java version.
- Spring Boot version.
- Maven dependencies.
- Testing libraries.
- Database drivers.
- Security dependencies.
- Code quality plugins.
- Build plugins.
- Dependency versions.

Do not upgrade Spring Boot or Java versions as part of an unrelated feature without reviewing compatibility and obtaining approval.

### 7.4 Understand application configuration

```
cat src/main/resources/application.yml
```

Look for:

- Server port.
- Database configuration.
- Authentication configuration.
- External service endpoints.
- Logging configuration.
- Health endpoint configuration.
- Application profiles.

Do not copy production secrets into local configuration.

# PART 3 — DEVELOPER ENVIRONMENT

## 8. Required development tools

| Tool                    | Why it is required               |
| ----------------------- | -------------------------------- |
| Git                     | Source-code version control      |
| Java 21                 | Run the application              |
| Maven Wrapper           | Reproducible project build       |
| IntelliJ IDEA / VS Code | Application development          |
| Docker                  | Local container testing          |
| curl                    | HTTP testing                     |
| Postman                 | API testing, if used by the team |

Additional tools may be needed for specific tasks:

- AWS CLI for approved AWS development activities.
- `kubectl` for approved Kubernetes diagnostics.
- Helm for chart validation.
- Terraform for infrastructure contributions.

Developers do not need unrestricted access to production resources.

## 9. Verify local environment

Check Git:

```
git --version
```

Check Java:

```
java -version
```

Check Docker:

```
docker --version
```

Check Maven Wrapper:

```
ls -l mvnw
```

Use the project's Maven Wrapper rather than assuming the system-installed Maven version is correct.

```
./mvnw --version
```

On Windows:

```
.\mvnw.cmd --version
```

### 9.1 Local environment consistency

A common problem is that a feature works on one developer's machine but fails in CI.

Reasons include:

- Different Java versions.
- Different dependency versions.
- Different environment variables.
- Different operating systems.
- Missing database configuration.
- External dependencies unavailable locally.

The project should define its required Java version, Maven Wrapper version, build commands and test prerequisites.

# PART 4 — GIT BRANCH MANAGEMENT

## 10. Understand the branching strategy

For this simulation, the developer works from the shared `main` branch.

Example:

```
main
 |
 +---- feature/ECOM-4821-order-status-tracking
 |
 +---- feature/ECOM-4822-payment-retry
 |
 +---- bugfix/ECOM-4823-cart-calculation
```

The precise branch model must match the organization's repository policy.

### 10.1 Update main

```
git checkout main
```

Fetch remote changes:

```
git fetch origin
```

Update local main:

```
git pull --ff-only origin main
```

### 10.2 Create feature branch

```
git checkout -b feature/ECOM-4821-order-status-tracking
```

Verify:

```
git branch --show-current
```

Expected:

```
feature/ECOM-4821-order-status-tracking
```

### 10.3 Why branch naming matters

Our convention:

```
feature/ECOM-4821-order-status-tracking
```

This gives us:

- Work type.
- Jira issue reference.
- Feature description.

The issue reference helps connect the requirement with the code change and eventual release.

## 11. Keep the feature branch updated

If other developers merge changes into `main` while we are working, our feature branch can become outdated.

Check remote updates:

```
git fetch origin
```

Review divergence:

```
git status
```

Inspect commit history:

```
git log --oneline --graph --decorate --all -15
```

Depending on the team's policy, update the feature branch using a merge or rebase.

Example rebase:

```
git rebase origin/main
```

If conflicts occur:

```
git status
```

Resolve conflicts, then:

```
git add <resolved-file>
git rebase --continue
```

To abandon an in-progress rebase safely:

```
git rebase --abort
```

Do not force-push shared branches without following team policy. If a rebase requires updating a branch already used by others, coordinate first and use an approved safe push method.

# PART 5 — API DESIGN AND FEATURE IMPLEMENTATION

## 12. Design the API contract

Our proposed endpoint:

```
GET /api/v1/orders/{orderId}/tracking
```

Request:

```
curl -X GET \
  "http://localhost:8080/api/v1/orders/ORD-10025/tracking" \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -H "Accept: application/json"
```

Example response:

```
{
  "orderId": "ORD-10025",
  "status": "SHIPPED",
  "trackingNumber": "TRACK-987654",
  "carrier": "Example Logistics",
  "estimatedDelivery": "2026-11-05",
  "lastUpdated": "2026-11-01T09:30:00Z"
}
```

### 12.1 Response scenarios

| Scenario                             | Expected response                      |
| ------------------------------------ | -------------------------------------- |
| Authorized customer and order exists | 200                                    |
| Invalid order ID format              | 400                                    |
| Order does not exist                 | 404                                    |
| Customer is not authorized           | 403 or policy-defined 404              |
| Tracking is pending                  | 200 with agreed pending representation |
| Internal dependency failure          | Appropriate 5xx response               |

### 12.2 API versioning

The endpoint uses:

```
/api/v1/
```

Versioning helps manage API compatibility.

Do not modify an existing API's response structure without considering consumers.

If a breaking change is necessary:

1. Identify all consumers.
2. Discuss the change with service owners.
3. Define a migration period.
4. Document the new contract.
5. Add compatibility tests.
6. Follow the platform's API deprecation policy.

### 12.3 API validation

The developer should validate:

- Order ID format.
- Missing path values.
- Authentication.
- Authorization.
- Response structure.
- Error response structure.
- Data serialization.
- Unexpected input.

Input validation should be implemented using the project's established validation framework.

## 13. Implement the response DTO

File:

```
src/main/java/com/company/order/dto/OrderTrackingResponse.java
```

Example:

```
package com.company.order.dto;

import java.time.Instant;
import java.time.LocalDate;

public record OrderTrackingResponse(
        String orderId,
        String status,
        String trackingNumber,
        String carrier,
        LocalDate estimatedDelivery,
        Instant lastUpdated
) {
}
```

Why use a DTO?

A database entity can contain:

- Internal database identifiers.
- Customer information.
- Internal processing fields.
- Audit information.
- Sensitive values.

The API should expose only the approved response fields.

## 14. Implement domain exceptions

Avoid using generic exceptions for every application failure.

Example:

```
package com.company.order.exception;

public class OrderNotFoundException extends RuntimeException {

    public OrderNotFoundException(String orderId) {
        super("Order not found");
    }
}
```

Authorization exception:

```
package com.company.order.exception;

public class OrderAccessDeniedException extends RuntimeException {

    public OrderAccessDeniedException() {
        super("Order access is not authorized");
    }
}
```

The external error message should not expose sensitive internal details.

## 15. Implement the repository layer

The repository should use the existing persistence model.

For illustration, suppose the project already has an `Order` entity and a repository.

A possible Spring Data JPA repository method:

```
Optional<Order> findByOrderIdAndCustomerId(
        String orderId,
        String customerId
);
```

This method illustrates the ownership check at the data-access boundary.

It assumes that the existing entity has corresponding fields. Do not create these fields or replace the real repository without inspecting the actual application.

The important principle is that the application should retrieve an order associated with the authenticated customer rather than retrieve any order and trust caller-supplied ownership information.

## 16. Implement the service layer

File:

```
src/main/java/com/company/order/service/OrderTrackingService.java
```

Illustrative implementation:

```
package com.company.order.service;

import com.company.order.dto.OrderTrackingResponse;
import com.company.order.entity.Order;
import com.company.order.exception.OrderNotFoundException;
import com.company.order.repository.OrderRepository;
import org.springframework.stereotype.Service;

@Service
public class OrderTrackingService {

    private final OrderRepository orderRepository;

    public OrderTrackingService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
    }

    public OrderTrackingResponse getTracking(
            String orderId,
            String customerId
    ) {

        Order order = orderRepository
                .findByOrderIdAndCustomerId(orderId, customerId)
                .orElseThrow(() ->
                        new OrderNotFoundException(orderId)
                );

        return new OrderTrackingResponse(
                order.getOrderId(),
                order.getStatus().name(),
                order.getTrackingNumber(),
                order.getCarrier(),
                order.getEstimatedDelivery(),
                order.getLastUpdated()
        );
    }
}
```

This example assumes the existing `Order` entity already contains those fields.

If tracking data belongs to another service, the service layer should call the approved integration adapter instead of pretending that the fields exist in the Order entity.

### 16.1 Why use a service layer?

It separates application business logic from HTTP handling.

Responsibilities:

- Business validation.
- Order ownership checks.
- Data retrieval.
- Dependency coordination.
- Mapping domain data into DTOs.
- Business exception handling.

The controller should not contain all the business logic.

## 17. Implement the controller

File:

```
src/main/java/com/company/order/controller/OrderTrackingController.java
```

Example using a Spring Security authenticated principal:

```
package com.company.order.controller;

import com.company.order.dto.OrderTrackingResponse;
import com.company.order.service.OrderTrackingService;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/orders")
public class OrderTrackingController {

    private final OrderTrackingService orderTrackingService;

    public OrderTrackingController(
            OrderTrackingService orderTrackingService
    ) {
        this.orderTrackingService = orderTrackingService;
    }

    @GetMapping("/{orderId}/tracking")
    public OrderTrackingResponse getTracking(
            @PathVariable String orderId,
            @AuthenticationPrincipal Jwt jwt
    ) {

        String customerId = jwt.getSubject();

        return orderTrackingService.getTracking(
                orderId,
                customerId
        );
    }
}
```

This example assumes the application authenticates users using Spring Security OAuth2 Resource Server and that the JWT subject is the stable customer identifier.

If the real platform uses a different claim or principal structure, map the verified identity according to the identity-provider contract.

Never assume that every JWT's `sub` claim is automatically the application's customer ID.

## 18. Exception handling

File:

```
src/main/java/com/company/order/exception/GlobalExceptionHandler.java
```

Example:

```
package com.company.order.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(OrderNotFoundException.class)
    public ResponseEntity<Map<String, String>> handleNotFound(
            OrderNotFoundException exception
    ) {

        return ResponseEntity
                .status(HttpStatus.NOT_FOUND)
                .body(Map.of(
                        "error", "ORDER_NOT_FOUND",
                        "message", "Order was not found"
                ));
    }

    @ExceptionHandler(OrderAccessDeniedException.class)
    public ResponseEntity<Map<String, String>> handleAccessDenied(
            OrderAccessDeniedException exception
    ) {

        return ResponseEntity
                .status(HttpStatus.FORBIDDEN)
                .body(Map.of(
                        "error", "ACCESS_DENIED",
                        "message", "Order access is not authorized"
                ));
    }
}
```

The example demonstrates the handling of domain exceptions.

The existing application should also have consistent handling for validation errors, authentication errors and unexpected exceptions. Do not add a second competing global exception handler if one already exists.

# PART 6 — EXTERNAL MICROSERVICE INTEGRATION

## 19. When tracking information belongs to another service

Suppose the shipping service owns tracking information.

We must not duplicate the shipping service's responsibility.

The Order Service may need to call an internal endpoint.

Illustrative architecture:

```
Customer
   |
   v
Order Service
   |
   v
OrderTrackingService
   |
   v
ShippingServiceClient
   |
   v
Shipping Service
```

### 19.1 Define an integration interface

Example:

```
public interface ShippingTrackingClient {

    ShippingTrackingData getTracking(String orderId);
}
```

The service layer can depend on this interface instead of directly depending on HTTP implementation details.

### 19.2 Integration requirements

Before implementing a real HTTP client, confirm:

- Internal endpoint.
- Authentication method.
- Request and response contract.
- Connection timeout.
- Response timeout.
- Retry policy.
- Rate limits.
- Failure behavior.
- Logging requirements.

### 19.3 Timeouts

An external call should not wait indefinitely.

Configure connection and response timeouts according to the platform's service SLOs.

A timeout is different from a successful response with no tracking data.

### 19.4 Retry handling

Retries should be used carefully.

For a read-only tracking request, a limited retry may be appropriate for selected transient failures.

Do not retry every failure.

Examples:

| Failure            | Typical handling                                                 |
| ------------------ | ---------------------------------------------------------------- |
| Connection timeout | Possibly retry within a strict limit                             |
| HTTP 500           | Follow dependency retry policy                                   |
| HTTP 401           | Fix authentication; retrying unchanged credentials will not help |
| HTTP 404           | Treat according to API contract                                  |
| HTTP 429           | Respect rate-limit policy                                        |
| Invalid response   | Fail safely and investigate                                      |

Retries can multiply load during an outage. They must have limits, backoff and an overall request deadline.

### 19.5 Circuit breaker

If the existing platform uses a circuit breaker library, follow its established configuration.

The intended behavior is:

```
Normal requests
      |
      v
Dependency healthy?
   /       \
 Yes       No
  |         |
  v         v
Call      Fail fast
service   or approved fallback
```

Do not add a circuit breaker implementation without checking existing platform standards.

### 19.6 Idempotency and event processing

If tracking data arrives asynchronously, consider:

- Duplicate events.
- Event ordering.
- Consumer restarts.
- Redelivery.
- Poison messages.
- Schema evolution.

For example, processing the same `ORDER_SHIPPED` event twice should not produce duplicate tracking records.

This may require an event ID, unique constraint or idempotent update operation depending on the existing architecture.

# PART 7 — DATABASE IMPLEMENTATION

## 20. Identify database ownership

Before changing a database, answer:

1. Which database does Order Service use?
2. Does the service own the table?
3. Does the service use JPA or another data-access method?
4. Does tracking information already exist?
5. Is a schema change actually necessary?
6. Are other applications dependent on the table?
7. Does the migration require a DBA review?

Never directly modify another microservice's database to retrieve its data.

Each microservice should follow the platform's database ownership rules.

## 21. Database migration strategy

If the feature requires a new column or table, use the application's approved migration mechanism.

For example, if the project uses Flyway:

```
src/main/resources/db/migration/
    V12__add_order_tracking_fields.sql
```

Illustrative migration:

```
ALTER TABLE orders
ADD COLUMN tracking_number VARCHAR(100);
```

This SQL is only an example. It is not a recommendation to add this field without examining the existing schema.

### 21.1 Why versioned migrations?

They provide:

- Ordered schema changes.
- Repeatability across environments.
- Change history.
- Reviewable SQL.
- Controlled deployment.

### 21.2 Backward-compatible database changes

Our application may run multiple versions during a rolling deployment.

For example:

```
Old Order Service Pods
        |
        v
Existing Database Schema
        |
        ^
        |
New Order Service Pods
```

Both versions may run simultaneously during a rollout.

A safe expand-and-contract strategy is:

1. Add a backward-compatible database change.
2. Deploy application code that can work with both old and new schema states.
3. Migrate or backfill data if required.
4. Verify all application versions no longer need the old schema.
5. Remove obsolete fields in a later controlled release.

Do not combine an incompatible schema change with an application deployment that assumes all pods update simultaneously.

### 21.3 Database testing

Validate:

- Migration succeeds on a test database.
- Existing data remains correct.
- Application starts against the migrated schema.
- Queries use appropriate indexes.
- Migration execution is safe for expected data volume.
- Rollback or forward-recovery strategy is documented.

Database rollback is not always equivalent to rolling back application code. A destructive schema change may require a separate recovery plan.

# PART 8 — TESTING STRATEGY

## 22. Understand the testing pyramid

```
             E2E Tests
           /-----------\
          / Integration \
         /---------------\
        /   Unit Tests    \
       /-------------------\
```

Unit tests should be numerous and fast.

Integration tests validate component interactions.

End-to-end tests validate important business flows across the platform.

The appropriate balance depends on application risk and architecture.

## 23. Unit testing

Example test path:

```
src/test/java/com/company/order/service/
    OrderTrackingServiceTest.java
```

Illustrative test with Mockito:

```
@ExtendWith(MockitoExtension.class)
class OrderTrackingServiceTest {

    @Mock
    private OrderRepository orderRepository;

    @InjectMocks
    private OrderTrackingService orderTrackingService;

    @Test
    void shouldReturnTrackingForOrderOwner() {

        Order order = TestOrderFactory.shippedOrder(
                "ORD-10025",
                "CUSTOMER-1001"
        );

        when(orderRepository.findByOrderIdAndCustomerId(
                "ORD-10025",
                "CUSTOMER-1001"
        )).thenReturn(Optional.of(order));

        OrderTrackingResponse response =
                orderTrackingService.getTracking(
                        "ORD-10025",
                        "CUSTOMER-1001"
                );

        assertEquals("ORD-10025", response.orderId());
        assertEquals("SHIPPED", response.status());
    }

    @Test
    void shouldRejectUnknownOrUnauthorizedOrder() {

        when(orderRepository.findByOrderIdAndCustomerId(
                "ORD-10025",
                "CUSTOMER-2000"
        )).thenReturn(Optional.empty());

        assertThrows(
                OrderNotFoundException.class,
                () -> orderTrackingService.getTracking(
                        "ORD-10025",
                        "CUSTOMER-2000"
                )
        );
    }
}
```

`TestOrderFactory` is an illustrative test helper that must be implemented using the real Order entity.

### 23.1 Important unit test cases

- Valid order.
- Missing order.
- Wrong customer.
- Missing tracking number.
- Pending tracking.
- Unexpected status.
- Invalid order ID.
- Null or malformed data from an integration adapter.

## 24. Controller tests

Controller tests validate HTTP behavior.

Possible test scenarios:

| Scenario                   | Expected result         |
| -------------------------- | ----------------------- |
| Authenticated request      | 200                     |
| Missing token              | 401                     |
| Insufficient authorization | 403                     |
| Unknown order              | 404                     |
| Invalid input              | 400                     |
| Internal error             | Consistent 5xx response |

Use the existing Spring Security test configuration.

For JWT-based authentication, tests should supply controlled test principals or mocked authentication, not use real production tokens.

## 25. Integration testing

Integration tests validate components together.

Example:

```
HTTP Request
    |
    v
Spring Security
    |
    v
Controller
    |
    v
Service
    |
    v
Repository
    |
    v
Test Database
```

Integration tests should check:

- Application context starts.
- Controller mapping is correct.
- Authentication is enforced.
- Authorization is enforced.
- Database query returns expected data.
- Exception mapping is correct.
- JSON response is correct.
- Database migrations work.

Use the project's existing test database strategy. If the team uses Testcontainers, follow its existing patterns.

Never run integration tests against a production database.

## 26. Contract testing

Contract testing is especially important in microservices.

Suppose Order Service depends on Shipping Service.

The consumer expects:

```
{
  "orderId": "ORD-10025",
  "trackingNumber": "TRACK-987654",
  "status": "SHIPPED"
}
```

If Shipping Service changes `trackingNumber` to a different field name without compatibility, the consumer may break.

Contract tests validate the agreed request and response structure.

They should cover:

- Field names.
- Field types.
- Required and optional fields.
- HTTP status codes.
- Error responses.
- Compatibility with consumers.

Use the organization's existing contract-testing tool and CI integration if one is established.

Do not invent a new contract-testing platform for this feature without agreement.

## 27. Regression testing

Regression testing ensures the new feature has not broken existing functionality.

For Order Service, test:

- Order creation.
- Order retrieval.
- Order update.
- Order cancellation.
- Existing order APIs.
- Existing database behavior.
- Existing integrations.

A new API that works while existing order creation breaks is not a successful change.

## 28. Performance testing considerations

The developer should consider:

- Database query efficiency.
- N+1 query risks.
- External service response time.
- Connection pool usage.
- Response payload size.
- Concurrent requests.

The existing platform uses Prometheus and Grafana for monitoring.

If the feature introduces a new performance-sensitive operation, coordinate with QA/SRE on suitable load-test coverage.

Do not claim a feature meets an SLO merely because one local request was fast.

# PART 9 — FEATURE FLAGS AND SAFE DEVELOPMENT

## 29. Why use feature flags?

A feature flag allows a team to control whether a feature is available without necessarily reverting the application deployment.

Example:

```
order.tracking.enabled=false
```

Illustrative configuration:

```
features:
  order-tracking:
    enabled: ${ORDER_TRACKING_ENABLED:false}
```

Example application condition:

```
@Value("${features.order-tracking.enabled:false}")
private boolean trackingEnabled;
```

This is a simple illustration. A production implementation should use the application's established feature-flag mechanism and consistent error handling.

### 29.1 Feature flag considerations

- Default state must be intentional.
- Only authorized teams should change the flag.
- The flag should be observable.
- Disabled behavior must be tested.
- Rollout behavior must be documented.
- Temporary flags should have an owner and removal date.

A feature flag does not replace authentication, authorization or production approval.

## 30. Backward compatibility

Suppose a new response field is introduced:

```
{
  "orderId": "ORD-10025",
  "status": "SHIPPED",
  "trackingNumber": "TRACK-987654",
  "estimatedDelivery": "2026-11-05"
}
```

Adding an optional field may be compatible with consumers that ignore unknown fields.

However, removing or changing an existing field can break consumers.

Before changing a contract:

1. Identify consumers.
2. Review the API schema.
3. Check compatibility requirements.
4. Communicate with dependent teams.
5. Add contract tests.
6. Plan deprecation where necessary.

# PART 10 — CODE QUALITY AND SECURITY

## 31. Code quality standards

Our platform uses SonarQube for code-quality analysis.

Developers should follow:

- Naming conventions.
- Formatting rules.
- Small, understandable methods.
- Appropriate exception handling.
- Avoidance of duplicated code.
- Test coverage requirements.
- Dependency version policies.
- Documentation standards.

Example:

```
./mvnw clean verify
```

The local build validates project-configured checks.

SonarQube analysis may run in CI using the approved scanner and credentials. Do not assume a developer's laptop has access to the production SonarQube server.

## 32. Static application security testing

Our security toolchain includes Veracode.

Application security checks may identify:

- Injection vulnerabilities.
- Unsafe deserialization.
- Broken access control.
- Sensitive information exposure.
- Insecure cryptographic practices.
- Other code-level security risks.

The developer should:

1. Understand the finding.
2. Identify the vulnerable code.
3. Reproduce or validate the issue.
4. Fix the root cause.
5. Add regression tests.
6. Submit the updated code for rescanning.
7. Follow the approved security exception process if a finding cannot be resolved immediately.

Do not suppress a finding merely to make the pipeline pass.

## 33. Dependency security

Java applications depend on third-party libraries.

The developer should review:

- Dependency versions.
- Known vulnerabilities.
- Transitive dependencies.
- Unsupported libraries.
- License requirements.
- Dependency source and integrity.

Inspect dependencies:

```
./mvnw dependency:tree
```

Review Maven dependency updates according to the team's policy.

Do not upgrade every dependency automatically as part of a feature branch.

An upgrade can introduce behavioral changes or compatibility problems.

## 34. Container security

Our platform uses Trivy for container image scanning.

At development time, the developer can inspect the Dockerfile and locally built image if Trivy is installed and configured.

Example:

```
trivy image order-service:local
```

The command's output should be reviewed according to the organization's severity thresholds.

Developers should avoid:

- Running containers as root unnecessarily.
- Including build tools in the runtime image without reason.
- Embedding secrets in image layers.
- Using unpinned or uncontrolled base images.
- Copying unnecessary files into the image.

The actual mandatory scan gates and exception policy are defined by the platform security configuration.

## 35. Secret management

Never hardcode credentials.

Bad example:

```
spring:
  datasource:
    username: admin
    password: SuperSecretPassword
```

Better example:

```
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

For local development, use approved local secret storage or environment variables.

Example:

```
export DB_URL="jdbc:postgresql://localhost:5432/orders"
export DB_USERNAME="local_user"
export DB_PASSWORD="local_password"
```

These are sample local credentials, not production credentials.

Add appropriate patterns to `.gitignore`:

```
.env
.env.*
*.pem
*.key
secrets/
```

Important: `.gitignore` does not remove secrets that have already been committed.

If a credential is exposed:

1. Revoke or rotate it immediately.
2. Notify the security team.
3. Investigate whether it was accessed.
4. Remove the credential from repository history as required.
5. Verify replacement credentials are stored securely.

## 36. Authentication and authorization security

These are different concepts.

Authentication: Who is making the request?

Authorization: What is that authenticated user allowed to access?

For our tracking endpoint:

```
Customer A
    |
    v
Authenticated identity: CUSTOMER-1001
    |
    v
Requests ORD-10025
    |
    v
Service checks order ownership
    |
    v
Access allowed or denied
```

Security tests should verify that Customer A cannot access Customer B's order by changing the order ID.

This is an important protection against insecure direct object reference vulnerabilities.

# PART 11 — LOCAL DOCKER VALIDATION

## 37. Inspect the Dockerfile

Example production-style multi-stage Dockerfile:

```
FROM maven:3.9-eclipse-temurin-21 AS build

WORKDIR /app

COPY pom.xml .
COPY mvnw .
COPY .mvn .mvn

RUN chmod +x mvnw

RUN ./mvnw dependency:go-offline

COPY src ./src

RUN ./mvnw clean package -DskipTests


FROM eclipse-temurin:21-jre

WORKDIR /app

RUN useradd --system --uid 10001 appuser

COPY --from=build /app/target/*.jar app.jar

USER 10001

EXPOSE 8080

ENTRYPOINT ["java", "-jar", "app.jar"]
```

This is an illustrative Dockerfile. In a real environment, use approved pinned base-image versions, verify the exact build output and follow the platform's image-hardening requirements.

### 37.1 Why multi-stage builds?

The build stage contains Maven and compilation dependencies.

The runtime stage contains the application runtime and packaged application.

Benefits:

- Smaller runtime image.
- Fewer unnecessary tools.
- Reduced attack surface.
- Separation of build and runtime dependencies.

### 37.2 Build local image

```
docker build -t order-service:local .
```

Check the image:

```
docker images | grep order-service
```

### 37.3 Inspect image configuration

```
docker inspect order-service:local
```

Check the runtime user and exposed ports.

### 37.4 Run local container

```
docker run --rm \
  -p 8080:8080 \
  --name order-service-local \
  order-service:local
```

Check logs:

```
docker logs order-service-local
```

Test health:

```
curl -i http://localhost:8080/actuator/health
```

This assumes Actuator and the health endpoint are enabled.

### 37.5 Stop the container

If running in the background:

```
docker stop order-service-local
```

### 37.6 Clean up local image

```
docker rmi order-service:local
```

Do not remove images used by other running containers.

# PART 12 — LOCAL OBSERVABILITY AND OPERATIONAL READINESS

## 38. Logging requirements

Our platform uses ELK Stack with Logstash.

Developers should produce logs that can be processed by the existing logging pipeline.

A useful structured log might contain:

```
{
  "level": "INFO",
  "service": "order-service",
  "event": "order_tracking_requested",
  "orderId": "ORD-10025",
  "requestId": "req-abc123"
}
```

Do not log:

- Access tokens.
- Passwords.
- Database credentials.
- Full sensitive customer records.
- Payment information.
- Secret configuration values.

### 38.1 Correlation IDs

A correlation ID helps operations teams follow a request across multiple services.

Example:

```
Frontend
   |
   v
API Gateway
   |
   v
Order Service
   |
   v
Shipping Service
```

If the platform already propagates a request ID, preserve it.

Do not generate a different unrelated identifier at every service boundary.

### 38.2 Metrics

The feature may benefit from existing application metrics.

Examples:

- Tracking requests.
- Successful responses.
- Failed responses.
- Dependency errors.
- Request duration.

Use existing metric conventions and avoid labels with unbounded cardinality, such as customer IDs or individual order IDs.

### 38.3 Health checks

Understand the difference:

- Liveness: Is the application process functioning?
- Readiness: Can this instance accept traffic?
- Startup: Has the application completed initialization?

Follow the application's existing Actuator and Kubernetes health-check conventions.

Do not make liveness depend on every external dependency. A temporary shipping-service outage should not necessarily cause all Order Service pods to restart continuously.

# PART 13 — GIT QUALITY AND PRE-COMMIT CHECKS

## 39. Review local changes

Check current branch:

```
git branch --show-current
```

Check status:

```
git status
```

List changed files:

```
git diff --name-only
```

Review code:

```
git diff
```

Check whitespace errors:

```
git diff --check
```

These checks help catch accidental files, whitespace problems and unrelated changes.

## 40. Git hooks

Git hooks can run local checks before commits.

For example, a pre-commit hook can reject a commit containing an obvious secret pattern or run formatting checks.

A simple hook:

```
#!/bin/sh

set -e

echo "Running local validation..."

./mvnw -q -DskipTests compile

echo "Validation successful"
```

Save it as an executable hook in the appropriate local Git hooks directory, according to the team's tooling.

Important:

- Local hooks can be bypassed.
- They are not a replacement for CI.
- Shared hooks should be managed through approved repository tooling.
- Mandatory security and quality checks must be enforced centrally.

## 41. Avoid unrelated changes

Before committing, ask:

- Did I modify files unrelated to ECOM-4821?
- Did I accidentally commit local configuration?
- Did I update generated files unnecessarily?
- Did I modify production GitOps values?
- Did I change dependencies without justification?
- Did I alter an existing API unintentionally?

A feature branch should contain only changes relevant to its requirement.

# PART 14 — DATABASE, API AND APPLICATION FAILURE HANDLING

## 42. Common development failures

### Failure 1: Database connection failure

Symptoms:

```
Connection refused
```

Investigate:

```
printenv | grep DB_
```

Check local database availability and connection configuration.

Possible causes:

- Database not running.
- Wrong port.
- Incorrect hostname.
- Wrong credentials.
- Database not initialized.
- Network access blocked.

Do not use production credentials to fix a local development issue.

### Failure 2: External service timeout

Symptoms:

```
Read timed out
```

Investigate:

- Service URL.
- Network configuration.
- Authentication.
- Dependency health.
- Timeout configuration.
- Request volume.

Do not immediately increase timeouts without checking the overall request deadline and service SLO.

### Failure 3: Unauthorized API response

Symptoms:

```
HTTP 401
```

Check authentication configuration, token validity, issuer, audience and expected claims.

For:

```
HTTP 403
```

Check the authorization policy and customer ownership.

Do not solve an authorization problem by disabling security.

### Failure 4: Application starts but endpoint returns 404

Check:

- Controller package scanning.
- Request mapping.
- HTTP method.
- Context path.
- Application port.
- Security routing configuration.

### Failure 5: Existing API breaks

Check:

- Shared DTO changes.
- Entity mapping.
- Database migration.
- Serialization changes.
- Common exception handlers.
- Dependency upgrades.

Run regression tests before committing.

### Failure 6: Build works locally but fails in CI

Compare:

- Java version.
- Maven Wrapper.
- Environment variables.
- Dependency availability.
- Test database.
- Operating system.
- Build profile.
- Quality-gate configuration.

The CI execution is covered in Chapter 195.

# PART 15 — DEVELOPER HANDOFF AND DOCUMENTATION

## 43. Update application documentation

The developer should update the Order Service README if the feature changes its API behavior.

Document:

- API endpoint.
- HTTP method.
- Authentication requirement.
- Request parameters.
- Successful response.
- Error responses.
- Configuration requirements.
- Local test instructions.

Example documentation:

```
## Order Tracking API

Endpoint:

GET /api/v1/orders/{orderId}/tracking

Authentication:

Bearer token required.

Purpose:

Returns tracking information for an order owned
by the authenticated customer.

Possible responses:

200 - Tracking information returned.
400 - Invalid request.
401 - Authentication required.
403 - Access denied, according to API policy.
404 - Order not found.
```

The final documentation must match the actual API contract.

## 44. Developer handoff evidence

Before creating the Pull Request, collect the evidence needed by reviewers.

Example:

| Evidence            | Example                                 |
| ------------------- | --------------------------------------- |
| Jira issue          | ECOM-4821                               |
| Repository          | order-service                           |
| Feature branch      | feature/ECOM-4821-order-status-tracking |
| Changed files       | Controller, service, DTO, tests         |
| Unit tests          | Passed                                  |
| Integration tests   | Passed where applicable                 |
| Maven build         | BUILD SUCCESS                           |
| Security validation | Findings reviewed                       |
| Local Docker test   | Completed where applicable              |
| API documentation   | Updated                                 |
| Commit              | Actual Git SHA                          |

Do not claim a test passed unless it was actually executed.

## 45. Handoff to QA

The developer should tell QA:

- Which feature was implemented.
- Which API endpoint changed.
- What authentication is required.
- What test identities or fixtures are available.
- Which scenarios must be validated.
- What known limitations exist.
- Whether database migration is involved.
- Whether a feature flag controls availability.

QA should be able to test the feature without reverse-engineering the developer's implementation.

## 46. Handoff to DevOps and Platform teams

Where required, provide:

- New configuration requirements.
- Environment variable names.
- Secret references, not secret values.
- New network connectivity requirements.
- New resource requirements.
- Health-check changes.
- New application dependencies.
- Dockerfile changes.
- Helm configuration changes, if needed.
- Monitoring and logging requirements.

Do not directly modify production deployment configuration without the appropriate review and ownership.

# PART 16 — COMMIT AND PUSH

## 47. Stage the changes

Check:

```
git status
```

Review:

```
git diff
```

Stage only the intended files:

```
git add src/
```

If the README or other approved files changed, stage those explicitly too.

Review staged changes:

```
git diff --cached
```

## 48. Commit

```
git commit -m "feat(ECOM-4821): add order tracking API"
```

Check the commit:

```
git log -1 --oneline
```

Example:

```
a1b2c3d feat(ECOM-4821): add order tracking API
```

The SHA is illustrative. The actual SHA is generated by Git.

## 49. Push the feature branch

Check:

```
git branch --show-current
```

Push:

```
git push -u origin feature/ECOM-4821-order-status-tracking
```

Verify:

```
git branch -r
```

The feature branch should now be available in GitHub.

# PART 17 — PRODUCTION DEVELOPMENT SCENARIOS

## 50. Scenario: Another developer modifies the same class

Suppose two developers work on `OrderController.java`.

Developer A adds order tracking.

Developer B modifies order retrieval.

Both changes may conflict.

Correct approach:

1. Communicate with the other developer.
2. Understand both changes.
3. Avoid blindly choosing one side of a conflict.
4. Resolve the combined implementation.
5. Run tests.
6. Review the resulting diff.
7. Record the resolution in the Pull Request if relevant.

## 51. Scenario: Business changes the requirement during development

Suppose the business now wants the tracking API to return delivery partner contact information.

Do not silently add new fields.

Instead:

1. Update the Jira requirement.
2. Confirm data ownership.
3. Review privacy implications.
4. Update the API contract.
5. Identify new dependencies.
6. Update test cases.
7. Reassess the implementation estimate.
8. Obtain the relevant stakeholder agreement.

## 52. Scenario: Feature is too large for one release

Break it into smaller deliverable stories.

For example:

```
ECOM-4821 — Basic tracking API
ECOM-4822 — Shipping provider integration
ECOM-4823 — Tracking history
ECOM-4824 — Delivery notifications
```

Only use separate stories when they represent independently manageable work.

## 53. Scenario: External service is unavailable

The developer must establish the expected behavior.

Possible approaches:

- Return a controlled service-unavailable response.
- Return cached tracking information if the architecture supports it.
- Use asynchronously updated tracking data.
- Apply an approved fallback.

Do not silently return stale data as current information.

The response must distinguish between unavailable data and successfully retrieved data.

## 54. Scenario: A database migration is not backward-compatible

Stop and redesign the rollout.

Consider:

- Expand-and-contract migration.
- Compatibility between old and new application versions.
- Data backfill.
- Rollback feasibility.
- Migration duration.
- Database locking.
- Recovery plan.

Do not assume Kubernetes rolling deployment makes a database change safe.

## 55. Scenario: Security scan identifies a critical issue

The developer should:

1. Stop the release progression.
2. Read the finding.
3. Identify the affected code or dependency.
4. Validate exploitability where appropriate.
5. Fix the root cause.
6. Add a regression test.
7. Run the relevant scan again.
8. Follow the approved exception process if necessary.

Do not bypass a mandatory security gate without authorization.

# PART 18 — DEVELOPER RESPONSIBILITIES AND BOUNDARIES

## 56. What developers can do

Depending on repository permissions:

- Clone the application repository.
- Create feature branches.
- Modify application source code.
- Write tests.
- Update application documentation.
- Build locally.
- Run approved security tools.
- Push feature branches.
- Create Pull Requests.

## 57. What developers should not do without authorization

- Deploy directly to production.
- Change production database records manually.
- Modify production secrets.
- Bypass mandatory security checks.
- Change production GitOps configuration outside the approved workflow.
- Publish arbitrary images to production repositories.
- Disable monitoring or security controls.
- Modify another microservice's database directly.
- Approve their own production change where separation of duties is required.

## 58. Developer workflow versus DevOps workflow

| Activity                      | Primary owner                                    |
| ----------------------------- | ------------------------------------------------ |
| Feature implementation        | Developer                                        |
| Unit tests                    | Developer                                        |
| Application integration tests | Developer / QA                                   |
| Feature branch                | Developer                                        |
| Pull Request review           | Reviewers                                        |
| CI templates                  | Platform / DevOps                                |
| CI pipeline execution         | Automated CI                                     |
| Security policies             | Security / Platform                              |
| ECR publishing                | Automated release pipeline                       |
| GitOps promotion              | Automated release workflow / authorized operator |
| Environment deployment        | ArgoCD                                           |
| Production approval           | Authorized approver                              |
| Production monitoring         | SRE / Operations                                 |

This separation is important in enterprise environments.

# PART 19 — FINAL VALIDATION

## 59. Complete developer validation commands

Run the appropriate checks for the project.

```
# Verify branch
git branch --show-current

# Check repository state
git status

# Review modified files
git diff --name-only

# Review code changes
git diff

# Check whitespace errors
git diff --check

# Inspect Maven configuration
cat pom.xml

# Inspect Java version
java -version

# Execute build and configured tests
./mvnw clean verify

# Inspect dependencies
./mvnw dependency:tree

# Build local Docker image
docker build -t order-service:local .

# Inspect local images
docker images

# Push feature branch
git push -u origin feature/ECOM-4821-order-status-tracking
```

Do not execute the push until you have reviewed and validated the changes.

## 60. Final completion checklist

Developer readiness

# 0%

0 / 38 checklist items

Jira requirement understood

Acceptance criteria reviewed

Definition of Ready satisfied

Order Service architecture inspected

Data ownership confirmed

Authentication and authorization requirements confirmed

API contract agreed

Local development environment verified

Feature branch created from the correct base

Application implementation completed

DTO and exception handling reviewed

Database changes assessed

External service dependencies assessed

Backward compatibility reviewed

Unit tests written and executed

Controller/security tests executed

Integration tests executed where applicable

Contract tests considered and executed where applicable

Regression tests completed

Performance implications reviewed

Feature flags considered where applicable

Dependency security reviewed

Secret handling reviewed

SonarQube requirements understood

Veracode requirements understood

Trivy requirements understood

Logging and metrics requirements reviewed

Health-check behavior reviewed

Docker image built locally where applicable

Local API behavior validated

Git diff reviewed

No unrelated changes staged

No secrets committed

Documentation updated

QA handoff information prepared

DevOps/Platform handoff information prepared

Commit created with Jira reference

Feature branch pushed to GitHub

# PART 20 — INTERVIEW PREPARATION

## 61. Scenario-based interview questions

Q1. A developer receives a new feature requirement. What do you do before writing code?

I first understand the business requirement, acceptance criteria, API contract, service ownership, dependencies, database impact and security requirements. Then I inspect the existing application and create a feature branch.

Q2. Why should a developer not directly modify the main branch?

Feature branches isolate changes and allow code review and automated checks before changes are merged.

Q3. How do you prevent one customer from accessing another customer's order?

I obtain the authenticated customer identity from the verified security context and enforce ownership in the service or data-access layer. I also write tests for unauthorized access.

Q4. What happens if the Order Service depends on Shipping Service and Shipping Service is unavailable?

I follow the defined dependency failure policy, including timeouts, limited retries where appropriate and an approved fallback or controlled error response. I avoid indefinite waits and uncontrolled retries.

Q5. How do you manage a database schema change during a rolling deployment?

I use a backward-compatible expand-and-contract strategy so old and new application versions can work during the rollout.

Q6. What is the difference between unit and integration testing?

Unit tests validate individual logic in isolation. Integration tests validate interactions between application components and dependencies.

Q7. Why is contract testing important in microservices?

It validates communication contracts between services and helps identify incompatible changes before deployment.

Q8. What do you do if a local Docker container works but the application fails in Kubernetes?

I compare environment configuration, secrets, network connectivity, service dependencies, resource settings, health probes, security context and container logs. Kubernetes deployment troubleshooting belongs to the later deployment chapters.

Q9. How do you manage secrets in a Spring Boot application?

I use externalized configuration and the approved secret-management mechanism. I do not hardcode secrets or commit `.env` files.

Q10. What happens after pushing a feature branch?

I create a Pull Request, link the Jira issue, request reviewers and wait for required code review and automated checks.

Q11. Can a feature be considered complete when the unit tests pass?

No. It must also satisfy the acceptance criteria, relevant integration and security requirements, compatibility checks, documentation and team Definition of Done.

Q12. Why should developers not directly deploy to production?

Production deployment must follow controlled release, authorization and traceability processes. This protects production from unreviewed and unauthorized changes.

# PART 21 — CHAPTER 193 FINAL RESULT

## 62. End-to-end developer workflow result

Our example Jira issue:

```
ECOM-4821
```

Our service:

```
order-service
```

Our feature:

```
Order Tracking API
```

Our branch:

```
feature/ECOM-4821-order-status-tracking
```

Our commit:

```
a1b2c3d
```

The commit SHA is illustrative. The actual SHA must be obtained from Git.

Our repository:

```
https://github.com/company-ecommerce/order-service
```

Our final state:

Developer implementation

Complete in simulation

Requirement analysis

Completed

Feature implementation

Completed

Local validation

Defined

Git commit and push

Completed in simulation

Pull Request

Not started

CI pipeline

Not started

Production deployment

Not started

Important: The code samples in this chapter are implementation patterns, not a tested, complete runnable repository. Actual application entities, authentication claims, database schema, service integrations and build configuration must be matched to the existing Order Service before use.

## 63. What comes next?

# CHAPTER 194 — PULL REQUEST WORKFLOW

Filename:

```
194-Pull-Request-Workflow.md
```

In Chapter 194, we continue with the exact same Jira issue and feature branch.

We will cover:

1. Creating a Pull Request in GitHub.
2. Selecting the correct base and feature branches.
3. Linking Jira issue `ECOM-4821`.
4. Writing a professional PR description.
5. Adding reviewers and code owners.
6. GitHub branch protection rules.
7. Required approvals and status checks.
8. Reviewing Java code and application architecture.
9. Reviewing Dockerfile and configuration changes.
10. Reviewing security and dependency findings.
11. Handling review comments.
12. Resolving merge conflicts.
13. Squash, merge and rebase strategies.
14. Preventing unauthorized merges.
15. Merging the approved PR.
16. Verifying the merged commit SHA.
17. Handing the merged change to the CI pipeline.

The next chapter will pick up exactly where this one ends. We will not restart the scenario or create a different feature.