# 117-Event-Driven-Microservice-Architecture.md

````markdown
# 117-Event-Driven-Microservice-Architecture.md

# Event-Driven Microservice Architecture

**Project:** Enterprise E-Commerce Platform  
**Architecture:** Event-Driven Microservices  
**Platform:** AWS EKS  
**Messaging:** Kafka + SQS + SNS + RabbitMQ  
**Database:** Amazon RDS PostgreSQL  
**Cache:** Amazon ElastiCache Redis/Valkey  
**Deployment:** Helm + ArgoCD  
**Infrastructure:** Terraform  
**Observability:** Prometheus + Grafana + Alertmanager + ELK + Distributed Tracing

---

# 1. Objective

We have now implemented:

- RDS
- Redis
- SQS
- SNS
- RabbitMQ
- Kafka
- EKS
- Terraform
- Helm
- ArgoCD
- IAM / Pod Identity
- CI/CD
- GitOps
- Observability foundations

Now we connect them into one complete **production-style event-driven architecture**.

The objective is to understand what actually happens when a customer places an order.

We will follow the complete business workflow:

```text
Customer
   |
   v
Order
   |
   +--> Inventory
   |
   +--> Payment
   |
   +--> Fraud
   |
   +--> Shipping
   |
   +--> Delivery
   |
   +--> Notification
   |
   +--> Invoice
   |
   +--> Reporting
````

We will also answer:

* Which service calls which service?
* Which communication should be synchronous?
* Which communication should be asynchronous?
* Where should Kafka be used?
* Where should SQS be used?
* Where should SNS be used?
* Where does RabbitMQ fit?
* Where does the database transaction happen?
* How do we prevent lost events?
* How do we prevent duplicate processing?
* What happens when Payment fails?
* What happens when Inventory fails?
* What happens when Kafka is unavailable?
* What happens when a service crashes?
* How do we compensate a failed transaction?
* How do we trace one order across 20+ services?

---

# 2. The 21 Microservices

Our platform contains:

```text
01. auth-service
02. user-service
03. customer-service
04. catalog-service
05. product-service
06. inventory-service
07. pricing-service
08. promotion-service
09. cart-service
10. order-service
11. payment-service
12. fraud-service
13. shipping-service
14. delivery-service
15. notification-service
16. search-service
17. recommendation-service
18. review-service
19. tax-service
20. invoice-service
21. reporting-service
```

Each service owns its business responsibility.

The architecture should avoid creating a single giant shared application database.

---

# 3. Service Ownership

Example ownership:

| Service                | Responsibility                 |
| ---------------------- | ------------------------------ |
| auth-service           | Authentication / authorization |
| user-service           | User account data              |
| customer-service       | Customer profile               |
| catalog-service        | Product catalog                |
| product-service        | Product information            |
| inventory-service      | Stock availability             |
| pricing-service        | Product pricing                |
| promotion-service      | Discounts / promotions         |
| cart-service           | Shopping cart                  |
| order-service          | Order lifecycle                |
| payment-service        | Payment processing             |
| fraud-service          | Fraud evaluation               |
| shipping-service       | Shipment creation              |
| delivery-service       | Delivery tracking              |
| notification-service   | Email/SMS/push                 |
| search-service         | Search index                   |
| recommendation-service | Recommendations                |
| review-service         | Product reviews                |
| tax-service            | Tax calculation                |
| invoice-service        | Invoice generation             |
| reporting-service      | Analytics/reporting            |

---

# 4. The Most Important Architectural Rule

Each service owns its data.

Example:

```text
Order Service
    |
    +--> orders database

Payment Service
    |
    +--> payments database

Inventory Service
    |
    +--> inventory database
```

Avoid:

```text
All Services
     |
     v
One Shared Database
```

because that creates tight coupling.

---

# 5. Database-per-Service

Logical architecture:

```text
                         RDS PostgreSQL
                              |
        +---------------------+---------------------+
        |                     |                     |
        v                     v                     v
   Order Database       Payment Database      Inventory Database
        |                     |                     |
        v                     v                     v
   Order Service         Payment Service      Inventory Service
```

The physical implementation may use:

* Separate RDS instances
* Separate databases
* Separate schemas

depending on scale, security and operational requirements.

The logical ownership boundary remains important even when multiple databases share infrastructure.

---

# 6. Synchronous vs Asynchronous Communication

Not every interaction should become an event.

We use synchronous communication when the caller immediately needs a response.

Example:

```text
Order Service
     |
     | GET price
     v
Pricing Service
     |
     v
Price
```

We use asynchronous communication when the caller does not need to wait for downstream processing.

Example:

```text
OrderCreated
     |
     +--> Notification
     +--> Reporting
     +--> Search
```

---

# 7. Decision Framework

Use synchronous communication when:

```text
Caller needs immediate answer
AND
operation is request/response
AND
latency is acceptable
```

Use asynchronous communication when:

```text
Processing can happen later
OR
multiple consumers need the event
OR
consumer availability should not block producer
OR
work needs buffering
OR
event replay is valuable
```

---

# 8. Our Communication Strategy

Example:

| Communication           | Technology                     |
| ----------------------- | ------------------------------ |
| Browser -> Order API    | HTTPS                          |
| Order -> Pricing        | REST/gRPC                      |
| Order -> Promotion      | REST/gRPC                      |
| Order -> Tax            | REST/gRPC                      |
| Order -> Inventory      | Event-driven where appropriate |
| Order -> Payment        | Saga/event-driven              |
| Order -> Notification   | Kafka/SNS/SQS                  |
| Order -> Reporting      | Kafka                          |
| Order -> Search         | Kafka                          |
| Payment async work      | SQS where queue semantics fit  |
| Notification jobs       | SQS                            |
| Event fan-out           | SNS                            |
| Durable event stream    | Kafka                          |
| Advanced broker routing | RabbitMQ                       |

The exact choice is based on the business workflow rather than a blanket rule.

---

# 9. Complete Order Flow

The customer performs:

```text
POST /api/v1/orders
```

Request:

```json
{
  "customerId": "CUS-1001",
  "items": [
    {
      "productId": "PROD-1001",
      "quantity": 2
    }
  ],
  "shippingAddressId": "ADDR-1001"
}
```

The request reaches:

```text
Internet
   |
   v
CloudFront/WAF
   |
   v
ALB
   |
   v
EKS
   |
   v
order-service
```

---

# 10. Initial Order Processing

Order Service performs synchronous validation.

```text
Order Request
     |
     v
Validate customer
     |
     v
Validate products
     |
     v
Get pricing
     |
     v
Calculate promotions
     |
     v
Calculate tax
     |
     v
Create order
```

The exact calls depend on the business design.

---

# 11. Example Synchronous Calls

```text
order-service
     |
     +----> customer-service
     |
     +----> product-service
     |
     +----> pricing-service
     |
     +----> promotion-service
     |
     +----> tax-service
```

Example:

```http
GET /api/v1/products/PROD-1001
```

Then:

```http
POST /api/v1/pricing/calculate
```

Then:

```http
POST /api/v1/tax/calculate
```

---

# 12. Why Not Make Everything Synchronous?

Bad architecture:

```text
Order
 |
 +--> Inventory
 |
 +--> Payment
 |
 +--> Fraud
 |
 +--> Shipping
 |
 +--> Delivery
 |
 +--> Notification
 |
 +--> Invoice
 |
 +--> Reporting
 |
 +--> Search
```

All must respond before Order Service returns.

If Notification is slow:

```text
Notification slow
      |
      v
Order slow
      |
      v
Customer waits
```

If Reporting is down:

```text
Reporting DOWN
      |
      v
Order creation fails
```

This creates excessive coupling.

---

# 13. Event-Driven Order Processing

Instead:

```text
Order Service
      |
      v
Create Order
      |
      v
Commit DB + Outbox
      |
      v
Kafka
      |
      +------> Inventory
      |
      +------> Payment
      |
      +------> Fraud
      |
      +------> Reporting
      |
      +------> Notification
```

Now downstream services are decoupled.

---

# 14. Transactional Outbox

Order Service transaction:

```text
BEGIN TRANSACTION

INSERT INTO orders

INSERT INTO outbox_events

COMMIT
```

Both succeed or fail together.

Example:

```text
orders
--------------------------------
ORD-1001
CUSTOMER-1001
CREATED


outbox_events
--------------------------------
evt-001
OrderCreated
ORD-1001
PENDING
```

---

# 15. Outbox Publisher

The publisher reads:

```text
status = PENDING
```

and publishes:

```text
OrderCreated
```

to:

```text
ecommerce.orders.created.v1
```

Then:

```text
Kafka
 |
 +--> Inventory
 +--> Payment
 +--> Fraud
 +--> Notification
 +--> Reporting
```

---

# 16. OrderCreated Event

Example:

```json
{
  "eventId": "evt-001",
  "eventType": "OrderCreated",
  "eventVersion": 1,
  "occurredAt": "2026-09-28T09:30:00Z",
  "source": "order-service",
  "correlationId": "corr-12345",
  "payload": {
    "orderId": "ORD-1001",
    "customerId": "CUS-1001",
    "amount": 2499.00,
    "currency": "INR"
  }
}
```

---

# 17. Correlation ID

Every request gets:

```text
correlationId
```

Example:

```text
corr-12345
```

It travels through the system.

```text
Order
 |
 +--> Payment
 |
 +--> Inventory
 |
 +--> Fraud
 |
 +--> Shipping
 |
 +--> Notification
```

All logs contain:

```text
correlationId=corr-12345
```

This allows us to search the entire business transaction.

---

# 18. Event ID

Each event also has:

```text
eventId
```

Example:

```text
evt-001
```

Difference:

```text
correlationId
=
whole business workflow

eventId
=
specific event
```

Example:

```text
Correlation:
corr-12345

Events:

evt-001 OrderCreated
evt-002 InventoryReserved
evt-003 PaymentAuthorized
evt-004 FraudApproved
evt-005 ShipmentCreated
```

---

# 19. Saga Pattern

Our order workflow is a distributed transaction.

We cannot use one PostgreSQL transaction across:

```text
Order DB
Payment DB
Inventory DB
Fraud DB
Shipping DB
```

Instead we use a Saga.

A Saga consists of:

```text
Local Transaction
+
Event
+
Next Local Transaction
+
Compensation when necessary
```

---

# 20. Saga Example

```text
Create Order
     |
     v
Reserve Inventory
     |
     v
Authorize Payment
     |
     v
Fraud Check
     |
     v
Create Shipment
     |
     v
Complete Order
```

If something fails:

```text
Compensating actions
```

may be triggered.

---

# 21. Saga Choreography

In choreography there is no central coordinator.

Example:

```text
OrderCreated
     |
     v
Inventory Service
     |
     v
InventoryReserved
     |
     v
Payment Service
     |
     v
PaymentAuthorized
     |
     v
Fraud Service
     |
     v
FraudApproved
     |
     v
Shipping Service
```

Each service reacts to events.

---

# 22. Saga Orchestration

An alternative is a central orchestrator.

```text
             Order Saga
             Orchestrator
                  |
       +----------+----------+
       |          |          |
       v          v          v
   Inventory   Payment     Fraud
       |          |          |
       +----------+----------+
                  |
                  v
               Shipping
```

The orchestrator explicitly tells services what to do.

---

# 23. Choreography vs Orchestration

| Aspect                | Choreography                  | Orchestration                   |
| --------------------- | ----------------------------- | ------------------------------- |
| Central coordinator   | No                            | Yes                             |
| Coupling              | Event-based                   | Orchestrator-based              |
| Workflow visibility   | Distributed                   | Central                         |
| Simple flows          | Good fit                      | Can be excessive                |
| Complex workflows     | Can become difficult to trace | Often easier to manage          |
| Compensation          | Distributed                   | Central workflow can coordinate |
| Operational debugging | Potentially harder            | Central workflow visibility     |

Neither approach is universally correct.

For a complex e-commerce checkout workflow, orchestration can provide clearer business-process visibility, while choreography can remain useful for independent side effects such as reporting and notifications.

---

# 24. Our Hybrid Model

We will use a hybrid architecture.

### Core order workflow

Use orchestration:

```text
Order Saga Orchestrator
       |
       +--> Inventory
       +--> Payment
       +--> Fraud
       +--> Shipping
```

### Independent side effects

Use event-driven choreography:

```text
OrderCreated
    |
    +--> Reporting
    +--> Notification
    +--> Search
    +--> Recommendation
```

This gives us clearer control over the critical workflow while keeping non-critical consumers decoupled.

---

# 25. Order Saga State

Example:

```text
ORDER_CREATED
      |
      v
INVENTORY_PENDING
      |
      v
INVENTORY_RESERVED
      |
      v
PAYMENT_PENDING
      |
      v
PAYMENT_AUTHORIZED
      |
      v
FRAUD_PENDING
      |
      v
FRAUD_APPROVED
      |
      v
SHIPPING_PENDING
      |
      v
SHIPMENT_CREATED
      |
      v
ORDER_CONFIRMED
```

---

# 26. Failed Saga

Example:

```text
Order Created
      |
      v
Inventory Reserved
      |
      v
Payment Failed
```

We cannot simply leave inventory reserved.

Compensation:

```text
PaymentFailed
      |
      v
Release Inventory
      |
      v
InventoryReleased
      |
      v
Order Cancelled
```

---

# 27. Compensation Is Not Rollback

Important distinction:

Database rollback:

```text
ROLLBACK
```

can undo operations inside one transaction.

Saga compensation:

```text
CancelPayment
ReleaseInventory
CancelShipment
RefundCustomer
```

is a new business operation.

Example:

```text
Inventory Reserved
```

cannot be undone with a database rollback after the transaction has already committed.

Instead:

```text
ReleaseInventory
```

must be executed.

---

# 28. Payment Failure Example

```text
OrderCreated
     |
     v
InventoryReserved
     |
     v
PaymentRequested
     |
     v
PaymentFailed
     |
     +----------------+
     |                |
     v                v
ReleaseInventory   OrderCancelled
```

The customer-facing status can become:

```text
PAYMENT_FAILED
```

or:

```text
CANCELLED
```

depending on the business model.

---

# 29. Inventory Failure Example

```text
OrderCreated
     |
     v
InventoryReservation
     |
     v
InventoryReservationFailed
     |
     v
OrderCancelled
```

No payment should be captured if the business workflow requires inventory reservation before payment capture.

If payment was already authorized, compensation may be:

```text
PaymentAuthorized
       |
       v
InventoryFailure
       |
       v
Void Authorization
```

The exact action depends on the payment provider.

---

# 30. Fraud Failure

Suppose:

```text
Order
  |
  v
Inventory Reserved
  |
  v
Payment Authorized
  |
  v
Fraud Rejected
```

Compensation:

```text
FraudRejected
     |
     +--> Void/Reverse payment authorization
     |
     +--> Release inventory
     |
     +--> Cancel order
     |
     +--> Notification
```

Every compensation operation must itself be idempotent.

---

# 31. Shipping Failure

Suppose:

```text
Order
 |
 +--> Inventory Reserved
 |
 +--> Payment Authorized
 |
 +--> Fraud Approved
 |
 +--> Shipping Creation FAILED
```

Possible response:

```text
Retry shipping
```

before compensation.

If shipping remains unavailable:

```text
Cancel order
Release inventory
Refund/void payment
```

depending on the point reached in the payment lifecycle.

---

# 32. Payment Capture vs Authorization

A production payment workflow often distinguishes:

```text
Authorization
```

from:

```text
Capture
```

Example:

```text
Order
 |
 v
Payment Authorization
 |
 v
Inventory/Fraud
 |
 v
Payment Capture
```

This can reduce the need for a refund when later preconditions fail.

However, the exact payment flow depends on the payment provider and business requirements.

---

# 33. Kafka for Event Distribution

Kafka is ideal for:

```text
OrderCreated
PaymentAuthorized
PaymentFailed
InventoryReserved
InventoryReleased
FraudApproved
FraudRejected
ShipmentCreated
DeliveryUpdated
InvoiceCreated
```

These events can have multiple independent consumers.

---

# 34. SQS for Work Queues

Use SQS when the requirement is:

```text
Do this work asynchronously.
```

Example:

```text
Notification Service
       |
       v
SQS notification-jobs
       |
       +--> Worker 1
       +--> Worker 2
       +--> Worker 3
```

Example jobs:

```text
Send email
Send SMS
Generate notification
Call external notification provider
```

SQS provides queue semantics and managed DLQ/redrive functionality.

---

# 35. SNS for Fan-Out

SNS can be useful where a single notification needs to be distributed to multiple queues.

```text
Order Service
     |
     v
SNS
     |
     +--> Notification SQS
     +--> Analytics SQS
     +--> Audit SQS
```

This is different from Kafka's durable partitioned event-stream model.

---

# 36. RabbitMQ

RabbitMQ can be used for workflows requiring:

```text
Exchange-based routing
Routing keys
AMQP semantics
Explicit queue topology
```

Example:

```text
Order Service
     |
     v
RabbitMQ Exchange
     |
     +--> Queue A
     +--> Queue B
```

Do not introduce RabbitMQ and Kafka for the same event without a specific architectural reason.

---

# 37. Messaging Decision Table

| Requirement                            | Choice    |
| -------------------------------------- | --------- |
| Durable event stream                   | Kafka     |
| Replay historical events               | Kafka     |
| High partitioned throughput            | Kafka     |
| Simple asynchronous job                | SQS       |
| Managed queue + DLQ                    | SQS       |
| Pub/Sub fan-out                        | SNS       |
| SNS to independent queues              | SNS + SQS |
| Complex AMQP routing                   | RabbitMQ  |
| RabbitMQ-based application integration | RabbitMQ  |
| Long-lived event history               | Kafka     |

---

# 38. Order Workflow — Complete View

```text
Customer
   |
   v
API Gateway / ALB
   |
   v
Order Service
   |
   +--> Customer validation
   |
   +--> Product validation
   |
   +--> Pricing
   |
   +--> Promotion
   |
   +--> Tax
   |
   v
PostgreSQL
   |
   +--> orders
   |
   +--> outbox
            |
            v
           Kafka
            |
    +-------+--------+---------+---------+
    |       |        |         |         |
    v       v        v         v         v
Inventory Payment   Fraud  Reporting Notification
    |       |        |         |         |
    v       v        v         v         v
   DB      DB       DB       Data      SQS
                               |         |
                               v         v
                           Analytics   Workers
```

---

# 39. Inventory Workflow

Inventory consumes:

```text
OrderCreated
```

Then:

```text
Validate event
     |
     v
Check stock
     |
     +---- insufficient
     |          |
     |          v
     |    InventoryReservationFailed
     |
     +---- available
                |
                v
         Reserve inventory
                |
                v
        InventoryReserved
```

---

# 40. Inventory Database Transaction

```text
BEGIN

SELECT inventory
FOR UPDATE

Check available quantity

UPDATE inventory
SET reserved_quantity = reserved_quantity + requested

INSERT reservation

INSERT outbox event

COMMIT
```

The exact locking strategy depends on the database model and traffic.

---

# 41. Payment Workflow

Payment consumes:

```text
InventoryReserved
```

Then:

```text
Validate order
     |
     v
Create payment transaction
     |
     v
Call payment provider
     |
     +---- success
     |       |
     |       v
     | PaymentAuthorized
     |
     +---- failure
             |
             v
       PaymentFailed
```

---

# 42. Payment Idempotency

External payment APIs often support idempotency keys.

Use:

```text
paymentIdempotencyKey
```

Example:

```text
ORDER-1001-PAYMENT
```

If the payment request is retried:

```text
Attempt 1
   |
   v
Provider

Network timeout

Attempt 2
   |
   v
Provider
```

The provider should recognize the same idempotency key according to its API contract.

Otherwise we risk charging the customer twice.

---

# 43. Fraud Workflow

Fraud consumes:

```text
PaymentAuthorized
```

or another event appropriate to the business flow.

Example:

```text
PaymentAuthorized
       |
       v
Fraud Evaluation
       |
       +---- approved
       |       |
       |       v
       | FraudApproved
       |
       +---- rejected
               |
               v
          FraudRejected
```

---

# 44. Shipping Workflow

Shipping consumes:

```text
FraudApproved
```

Then:

```text
Validate address
     |
     v
Select carrier
     |
     v
Create shipment
     |
     v
ShipmentCreated
```

---

# 45. Delivery Workflow

Shipping produces:

```text
ShipmentCreated
```

Delivery Service tracks:

```text
PICKED_UP
IN_TRANSIT
OUT_FOR_DELIVERY
DELIVERED
```

Events:

```text
DeliveryStatusChanged
```

---

# 46. Notification Workflow

Notification Service should generally not block the order transaction.

Example:

```text
OrderCreated
      |
      v
Notification Service
      |
      v
SQS
      |
      +--> Email Worker
      +--> SMS Worker
      +--> Push Worker
```

If email provider is down:

```text
Order
=
Still operational
```

while notification backlog increases.

---

# 47. Notification Retry

```text
SQS
 |
 v
Worker
 |
 +--> success -> Delete message
 |
 +--> temporary failure -> Retry
 |
 +--> permanent failure -> DLQ
```

This is a good example of using SQS rather than forcing Kafka to handle every asynchronous work queue.

---

# 48. Reporting

Reporting Service consumes events:

```text
OrderCreated
PaymentAuthorized
InventoryReserved
ShipmentCreated
DeliveryStatusChanged
```

It can maintain analytical models.

Example:

```text
Kafka
 |
 v
Reporting Service
 |
 v
Reporting DB / Data Platform
```

Reporting should not directly query another service's private database to build its model.

---

# 49. Search

Product Service publishes:

```text
ProductCreated
ProductUpdated
ProductDeleted
```

Search Service consumes them.

```text
Product Service
      |
      v
Kafka
      |
      v
Search Service
      |
      v
Search Index
```

Now product writes don't need to synchronously update the search engine.

---

# 50. Recommendation Service

Recommendation Service can consume:

```text
ProductViewed
OrderCreated
ProductPurchased
ProductRated
```

and build recommendation models.

This is another strong Kafka use case because historical event streams can be valuable for processing.

---

# 51. Review Service

Review Service handles:

```text
POST /reviews
```

After a review is created:

```text
ReviewCreated
```

can be published.

Consumers:

```text
Recommendation
Search
Reporting
Notification
```

---

# 52. Invoice Service

Invoice Service consumes:

```text
PaymentCaptured
```

or the appropriate completed-payment event.

Then:

```text
Generate invoice
       |
       v
Store PDF in S3
       |
       v
InvoiceCreated
```

Notification Service can consume:

```text
InvoiceCreated
```

and send the invoice to the customer.

---

# 53. S3 + Invoice Architecture

```text
Invoice Service
      |
      v
Generate PDF
      |
      v
S3
 |
 +--> invoices/2026/09/INV-1001.pdf
      |
      v
Publish InvoiceCreated
      |
      v
Kafka
```

The event contains the S3 object reference rather than embedding the PDF.

---

# 54. Order State Machine

A useful order state model:

```text
CREATED
   |
   v
INVENTORY_PENDING
   |
   +---- failed ---> CANCELLED
   |
   v
INVENTORY_RESERVED
   |
   v
PAYMENT_PENDING
   |
   +---- failed ---> CANCELLED
   |
   v
PAYMENT_AUTHORIZED
   |
   v
FRAUD_PENDING
   |
   +---- rejected ---> CANCELLED
   |
   v
FRAUD_APPROVED
   |
   v
SHIPPING_PENDING
   |
   +---- retry
   |
   v
SHIPMENT_CREATED
   |
   v
DELIVERY_IN_PROGRESS
   |
   v
DELIVERED
   |
   v
COMPLETED
```

The actual state machine should be owned by Order Service or the Saga orchestrator according to the selected architecture.

---

# 55. Saga Orchestrator

Conceptual component:

```text
order-saga-orchestrator
```

It maintains workflow state.

Example:

```text
Saga ID:
SAGA-10001

Order:
ORD-1001

State:
PAYMENT_PENDING
```

It records:

```text
Current step
Completed steps
Failed step
Retry count
Compensation state
Correlation ID
```

---

# 56. Saga State Persistence

Possible table:

```sql
CREATE TABLE order_saga (
    saga_id UUID PRIMARY KEY,
    order_id VARCHAR(100) NOT NULL,
    current_state VARCHAR(100) NOT NULL,
    correlation_id VARCHAR(100) NOT NULL,
    retry_count INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL
);
```

Do not store workflow state only in application memory.

If the orchestrator pod crashes:

```text
Memory state = lost
```

Persistent state allows recovery.

---

# 57. Saga Retry

Suppose Payment Service is temporarily unavailable.

```text
PaymentRequested
       |
       v
Payment Service unavailable
       |
       v
Saga retry
       |
       v
PaymentRequested
       |
       v
Success
```

Retries should be:

```text
Bounded
Backoff-based
Idempotent
Observable
```

---

# 58. Saga Timeout

Suppose Inventory never responds.

We should not keep:

```text
INVENTORY_PENDING
```

forever.

Define a timeout.

Example:

```text
Inventory reservation timeout
        |
        v
Compensation
        |
        v
Order cancelled
```

Timeout values must reflect actual business SLAs.

---

# 59. Saga Failure Matrix

| Step                  | Failure           | Compensation                                 |
| --------------------- | ----------------- | -------------------------------------------- |
| Order creation        | DB failure        | Transaction rollback                         |
| Inventory reservation | Stock unavailable | Cancel order                                 |
| Payment authorization | Payment rejected  | Release inventory                            |
| Fraud                 | Rejected          | Release inventory + void payment if required |
| Shipping              | Temporary         | Retry                                        |
| Shipping              | Permanent         | Cancel/compensate according to payment state |
| Notification          | Failure           | SQS retry/DLQ                                |
| Reporting             | Failure           | Consumer retry; order continues              |

---

# 60. Eventual Consistency

Our architecture is eventually consistent.

Example:

```text
Order DB
   |
   v
Order = CREATED
```

A few seconds later:

```text
Inventory DB
   |
   v
Reservation = CREATED
```

A few seconds later:

```text
Payment DB
   |
   v
Payment = AUTHORIZED
```

This is normal.

We should not force synchronous consistency across every service just to avoid eventual consistency.

---

# 61. Customer-Facing Order Status

The customer may see:

```text
Order received
```

while the Saga is still processing.

Example API:

```http
GET /api/v1/orders/ORD-1001
```

Response:

```json
{
  "orderId": "ORD-1001",
  "status": "PAYMENT_PENDING"
}
```

The frontend can poll or use a suitable push mechanism.

---

# 62. API vs Event

An important rule:

Use API when asking:

```text
"What is the current state?"
```

Use event when saying:

```text
"Something happened."
```

Examples:

```text
GET /orders/1001
```

means:

```text
What is the order state?
```

while:

```text
OrderCreated
```

means:

```text
An order was created.
```

---

# 63. Event Naming

Prefer past-tense business events:

```text
OrderCreated
PaymentAuthorized
PaymentFailed
InventoryReserved
InventoryReleased
FraudApproved
ShipmentCreated
DeliveryDelivered
InvoiceCreated
```

Avoid command-like names for events:

```text
CreateOrder
ChargePayment
ReserveInventory
```

Those sound like commands rather than facts.

Commands and events are conceptually different.

---

# 64. Command vs Event

Command:

```text
ReserveInventory
```

means:

```text
Please perform this operation.
```

Event:

```text
InventoryReserved
```

means:

```text
This operation already happened.
```

This distinction is important in Saga design.

---

# 65. Kafka Command Topic vs Event Topic

Kafka can technically carry both commands and events.

However, we should define conventions clearly.

For this architecture:

```text
Commands
```

are generally sent directly to the responsible workflow/service or through a deliberate command channel.

```text
Events
```

are published as facts for independent consumers.

Avoid building an architecture where every service publishes arbitrary commands to every other service without ownership rules.

---

# 66. Event Ownership

Example:

```text
OrderCreated
```

is owned by:

```text
order-service
```

Inventory Service must not publish:

```text
OrderCreated
```

because it does not own the order lifecycle.

Inventory owns:

```text
InventoryReserved
InventoryReleased
InventoryReservationFailed
```

Payment owns:

```text
PaymentAuthorized
PaymentCaptured
PaymentFailed
PaymentRefunded
```

---

# 67. Event Contract Ownership Table

| Event                 | Owner     |
| --------------------- | --------- |
| OrderCreated          | Order     |
| OrderCancelled        | Order     |
| InventoryReserved     | Inventory |
| InventoryReleased     | Inventory |
| PaymentAuthorized     | Payment   |
| PaymentFailed         | Payment   |
| PaymentCaptured       | Payment   |
| FraudApproved         | Fraud     |
| FraudRejected         | Fraud     |
| ShipmentCreated       | Shipping  |
| DeliveryStatusChanged | Delivery  |
| InvoiceCreated        | Invoice   |
| ReviewCreated         | Review    |
| ProductUpdated        | Product   |

---

# 68. Versioning

Example:

```text
ecommerce.orders.created.v1
```

If a breaking event change is required:

```text
ecommerce.orders.created.v2
```

During migration:

```text
Producer
   |
   +--> v1
   |
   +--> v2
```

Consumers migrate gradually.

After all required consumers move:

```text
v1
```

can eventually be retired according to the organization's compatibility and retention policy.

---

# 69. Backward-Compatible Changes

Prefer:

```text
Add optional field
```

instead of:

```text
Rename existing field
Remove existing field
Change data type
```

Example safe-ish evolution:

```json
{
  "orderId": "ORD-1001",
  "amount": 1000,
  "currency": "INR"
}
```

provided the consumer contract allows the new field.

---

# 70. Correlation and Trace Propagation

Request:

```text
corr-12345
```

should appear in:

```text
HTTP headers
Kafka headers
Application logs
Database audit fields where appropriate
SQS message attributes where applicable
Tracing spans
```

Example:

```text
traceId=abc123
correlationId=corr-12345
orderId=ORD-1001
eventId=evt-001
```

This gives us multiple ways to investigate an incident.

---

# 71. End-to-End Trace

Example:

```text
POST /orders
      |
      v
order-service
      |
      v
PostgreSQL
      |
      v
outbox
      |
      v
Kafka
      |
      +----> inventory-service
      |
      +----> payment-service
      |
      +----> fraud-service
```

Tracing should allow us to see the relationship between:

```text
HTTP request
```

and:

```text
Kafka event
```

and:

```text
downstream processing
```

This will be implemented in detail in the observability chapters.

---

# 72. Failure: Kafka Unavailable

Suppose Kafka becomes unavailable.

Order API:

```text
Request
  |
  v
Order DB
  |
  +--> Order saved
  |
  +--> Outbox saved
```

Kafka:

```text
DOWN
```

Outbox:

```text
PENDING
```

Order creation can remain successful if the business design allows asynchronous event publication.

Later:

```text
Kafka RECOVERED
      |
      v
Outbox Publisher
      |
      v
Kafka
```

This avoids losing the event.

---

# 73. Failure: Order Service Crashes

During:

```text
POST /orders
```

Suppose the process crashes.

If PostgreSQL transaction never committed:

```text
Order does not exist
```

If transaction committed:

```text
Order exists
Outbox exists
```

Outbox publisher can later publish the event.

This is exactly why the outbox pattern matters.

---

# 74. Failure: Inventory Service Crashes

Kafka retains:

```text
OrderCreated
```

Inventory consumer crashes.

After restart/rebalance:

```text
Inventory consumer
       |
       v
Kafka
       |
       v
OrderCreated
```

If already processed:

```text
eventId exists
```

Skip duplicate business operation.

---

# 75. Failure: Payment Timeout

Payment provider:

```text
Request sent
      |
      v
Network timeout
```

We do not know whether the provider processed the payment.

Do not blindly send another payment request.

Use:

```text
Idempotency key
Payment transaction state
Provider status lookup
Controlled retry
```

This is a critical real-world payment scenario.

---

# 76. Failure: Notification Down

Notification provider is unavailable.

```text
Order
 |
 v
Kafka
 |
 v
Notification
 |
 v
SQS
 |
 v
Worker
 |
 X Provider unavailable
```

SQS retains the work according to its queue/retention configuration.

Order processing does not need to fail simply because email delivery is temporarily unavailable.

---

# 77. Failure: Reporting Down

Reporting consumer crashes.

```text
Kafka
 |
 v
Reporting consumer
 |
 X
```

Order processing continues.

Reporting consumer later resumes and catches up.

This is one of the major benefits of independent Kafka consumer groups.

---

# 78. Failure: Fraud Down

Fraud is different.

Fraud may be part of the critical order workflow.

Therefore:

```text
Fraud unavailable
```

may cause:

```text
Order remains
FRAUD_PENDING
```

until:

```text
Retry succeeds
```

or:

```text
Business timeout
```

causes compensation.

Not every service should be treated as a non-critical side effect.

---

# 79. Critical vs Non-Critical Services

### Critical path

```text
Order
Inventory
Payment
Fraud
Shipping
```

### Non-critical side effects

```text
Notification
Reporting
Recommendation
Search indexing
Analytics
```

This classification helps determine:

* Synchronous vs asynchronous
* Timeout
* Retry
* Compensation
* Alert severity
* Customer impact

---

# 80. Business Workflow Architecture

```text
                 CUSTOMER
                     |
                     v
                ORDER API
                     |
                     v
               ORDER SERVICE
                     |
             +-------+-------+
             |               |
             v               v
        PostgreSQL         Outbox
                             |
                             v
                           Kafka
                             |
       +----------+----------+----------+----------+
       |          |          |          |          |
       v          v          v          v          v
   Inventory   Payment     Fraud    Reporting Notification
       |          |          |          |          |
       +----------+----------+          |          |
                  |                     |          |
                  v                     v          v
             Saga State              Analytics   SQS
                                                  |
                                                  v
                                               Workers
```

---

# 81. Complete Critical Path

Our initial critical path:

```text
OrderCreated
     |
     v
Inventory
     |
     v
InventoryReserved
     |
     v
Payment
     |
     v
PaymentAuthorized
     |
     v
Fraud
     |
     v
FraudApproved
     |
     v
Shipping
     |
     v
ShipmentCreated
     |
     v
OrderConfirmed
```

The exact ordering can be changed depending on business and payment requirements.

For example, some systems perform fraud before payment authorization or use payment authorization before inventory reservation.

The important architectural point is that the Saga defines the business state transitions explicitly.

---

# 82. Complete Event Flow

```text
OrderCreated
     |
     +--> InventoryReserved
     |        |
     |        v
     |   PaymentAuthorized
     |        |
     |        v
     |    FraudApproved
     |        |
     |        v
     |   ShipmentCreated
     |        |
     |        v
     |   DeliveryStarted
     |        |
     |        v
     |   DeliveryCompleted
     |
     +--> Reporting
     |
     +--> Notification
     |
     +--> Recommendation
```

---

# 83. Compensation Flow

Example payment failure:

```text
OrderCreated
     |
     v
InventoryReserved
     |
     v
PaymentFailed
     |
     +-------------------+
     |                   |
     v                   v
ReleaseInventory     CancelOrder
     |
     v
InventoryReleased
```

Example fraud rejection:

```text
PaymentAuthorized
       |
       v
FraudRejected
       |
       +--> Void payment authorization
       |
       +--> Release inventory
       |
       +--> Cancel order
       |
       +--> Notify customer
```

---

# 84. Exactly-Once Discussion

Avoid saying:

```text
Kafka gives exactly-once business processing.
```

Instead distinguish:

```text
Kafka delivery semantics
```

from:

```text
Business transaction semantics
```

A production application should be designed around:

```text
At-least-once delivery
+
Idempotent processing
+
Transactional state changes
```

Kafka has transaction features for particular Kafka-to-Kafka processing patterns, but they do not automatically make external PostgreSQL/payment operations exactly once.

---

# 85. Idempotency Everywhere

We need idempotency at multiple boundaries.

```text
Order API
Payment API
Kafka consumer
SQS worker
Saga commands
Compensation operations
External provider calls
```

Example:

```text
POST /orders
Idempotency-Key: ORD-REQUEST-123
```

If the client retries because of a network timeout:

```text
Request 1
Request 2
```

the service should avoid creating two orders when the API contract requires idempotent creation.

---

# 86. Idempotency Storage

Possible table:

```sql
CREATE TABLE idempotency_keys (
    idempotency_key VARCHAR(200) PRIMARY KEY,
    request_hash VARCHAR(200) NOT NULL,
    response_status INTEGER,
    response_body JSONB,
    created_at TIMESTAMP NOT NULL,
    expires_at TIMESTAMP
);
```

The exact implementation depends on the API and retention requirements.

---

# 87. Event Processing Transaction

Consumer:

```text
BEGIN

Check processed_events

If already processed:
    COMMIT / return

Otherwise:
    Perform business update
    Insert processed_events

COMMIT
```

Only after the business transaction is safely committed should the consumer advance its Kafka progress according to the chosen processing model.

---

# 88. Backpressure

Suppose:

```text
Kafka
 |
 v
Consumer
 |
 v
PostgreSQL
```

PostgreSQL slows down.

We should not keep increasing consumer concurrency indefinitely.

Instead:

```text
Consumer concurrency
       |
       v
Controlled
       |
       v
Backpressure
       |
       v
Kafka lag temporarily increases
```

Temporary lag can be safer than overwhelming the downstream system.

---

# 89. Circuit Breaker

For synchronous dependencies:

```text
Order Service
     |
     v
Pricing
```

If Pricing becomes unhealthy:

```text
Request
  |
  v
Circuit Breaker
  |
  +--> OPEN
```

This prevents every order request from continuously hammering the failing service.

Circuit breakers are particularly useful for synchronous calls.

---

# 90. Timeout Budget

Every synchronous call should have an explicit timeout.

Example:

```text
Order request budget = 2 seconds
```

Within that:

```text
Customer = 200 ms
Pricing  = 300 ms
Promotion = 200 ms
Tax      = 300 ms
Database = remaining budget
```

These numbers are examples only.

Never allow:

```text
No timeout
```

on production service-to-service calls.

---

# 91. Retry Budget

Retries consume time.

If:

```text
timeout = 1 second
retries = 5
```

the operation can consume several seconds or more depending on backoff.

Therefore:

```text
Request timeout
+
Retry timeout
+
Downstream timeout
```

must be designed as one latency budget.

---

# 92. Avoid Retry Storms

Bad:

```text
Service A
  |
  +--> Service B
          |
          X failure
          |
       Retry
          |
       Retry
          |
       Retry
```

while thousands of requests do the same.

Better:

```text
Timeout
+
Exponential backoff
+
Jitter
+
Circuit breaker
+
Rate limiting
+
Bounded retries
```

---

# 93. Event Storm

A single event can generate many downstream events.

Example:

```text
OrderCreated
   |
   +--> InventoryReserved
   |       |
   |       +--> InventoryUpdated
   |
   +--> PaymentAuthorized
   |       |
   |       +--> PaymentCaptured
   |
   +--> FraudApproved
   |
   +--> NotificationRequested
```

This is an event chain.

We must avoid accidental infinite loops.

Bad:

```text
ProductUpdated
   |
   v
SearchUpdated
   |
   v
ProductUpdated
   |
   v
SearchUpdated
```

Event ownership rules prevent this.

---

# 94. Event Governance

Every event should have:

```text
Owner
Schema
Version
Producer
Consumers
Retention
PII classification
Security classification
Replay policy
Deprecation date
```

Example:

```yaml
event:
  name: OrderCreated
  version: 1
  owner: order-service
  topic: ecommerce.orders.created.v1
  retention: 7d
  pii: restricted
```

---

# 95. PII in Events

Do not put unnecessary personal information into Kafka.

Avoid:

```json
{
  "customerName": "...",
  "password": "...",
  "cardNumber": "...",
  "cvv": "..."
}
```

Instead:

```json
{
  "customerId": "CUS-1001",
  "orderId": "ORD-1001"
}
```

Consumers can retrieve required information through approved APIs or their own projections.

Never put secrets or sensitive payment credentials into events.

---

# 96. Event Security

Security controls:

```text
TLS
IAM
Least privilege
Topic authorization
Encryption at rest
Network isolation
Audit logging
Schema validation
PII minimization
```

Example:

```text
Payment Service
```

should not automatically be allowed to consume every Kafka topic.

---

# 97. Production Observability

For every event we want:

```text
eventId
eventType
correlationId
traceId
source
timestamp
orderId
```

Logs:

```text
INFO
event=OrderCreated
eventId=evt-001
orderId=ORD-1001
correlationId=corr-12345
```

Metrics:

```text
orders_created_total
payment_authorization_total
inventory_reservation_total
saga_failures_total
event_processing_duration
```

---

# 98. Incident Investigation

Customer says:

> My order is stuck.

Start with:

```text
orderId
```

Example:

```text
ORD-1001
```

Search:

```text
Logs
Metrics
Traces
Kafka
Saga DB
Order DB
Payment DB
Inventory DB
```

Expected investigation:

```text
ORD-1001
   |
   +--> Order = CREATED
   |
   +--> Inventory = RESERVED
   |
   +--> Payment = PENDING
   |
   +--> Kafka payment event
   |
   +--> Payment consumer lag
```

Now we know where to investigate.

---

# 99. Production Failure Matrix

| Failure                | Customer impact              | Recovery                    |
| ---------------------- | ---------------------------- | --------------------------- |
| Notification down      | Usually delayed notification | SQS retry                   |
| Reporting down         | Usually no checkout impact   | Kafka catch-up              |
| Search down            | Search degradation           | Replay Product events       |
| Inventory down         | Checkout may stop/pause      | Retry / Saga timeout        |
| Payment down           | Checkout may stop/pause      | Retry / provider recovery   |
| Fraud down             | Order pending                | Retry / timeout             |
| Kafka temporary outage | Events delayed               | Outbox recovery             |
| PostgreSQL outage      | Service state unavailable    | RDS recovery                |
| Redis outage           | Cache degradation            | RDS fallback where designed |
| EKS pod crash          | Usually automatic recovery   | Kubernetes restart          |
| Node failure           | Pod rescheduling             | EKS/Karpenter recovery      |

---

# 100. End-to-End Architecture

```text
                              CUSTOMER
                                  |
                                  v
                         CloudFront / WAF
                                  |
                                  v
                                 ALB
                                  |
                                  v
                                EKS
                                  |
             +--------------------+--------------------+
             |                    |                    |
             v                    v                    v
        Order Service       Customer Service      Product Service
             |
             +----> Pricing
             |
             +----> Promotion
             |
             +----> Tax
             |
             v
         PostgreSQL
             |
             +----> orders
             |
             +----> outbox
                       |
                       v
                  Event Publisher
                       |
                       v
                  AMAZON MSK
                       |
       +---------------+----------------+----------------+
       |               |                |                |
       v               v                v                v
   Inventory        Payment           Fraud          Reporting
       |               |                |                |
       v               v                v                v
   Inventory DB     Payment DB       Fraud DB       Analytics
       |
       v
InventoryReserved
       |
       v
Payment
       |
       v
PaymentAuthorized
       |
       v
Fraud
       |
       v
FraudApproved
       |
       v
Shipping
       |
       v
Delivery
       |
       v
Notification
       |
       v
      SQS
       |
       +----> Email Worker
       +----> SMS Worker
       +----> Push Worker

Product Events
       |
       v
Search Service
       |
       v
Search Index

Order / Product / Review Events
       |
       v
Recommendation Service

Payment / Order Events
       |
       v
Invoice Service
       |
       v
S3
```

---

# 101. Final Architecture Principles

Our platform follows these principles:

```text
1. Each service owns its business responsibility.

2. Each service owns its data.

3. Synchronous APIs are used when an immediate response is required.

4. Kafka is used for durable event streams and independent consumers.

5. SQS is used for asynchronous work queues.

6. SNS is used for managed pub/sub fan-out where appropriate.

7. RabbitMQ is used where AMQP routing semantics are required.

8. Database + event consistency uses the transactional outbox.

9. Consumers are idempotent.

10. Saga manages distributed business workflows.

11. Compensation is used instead of distributed database rollback.

12. Events have owners and versioned contracts.

13. Correlation IDs connect business workflows.

14. Retries are bounded and use backoff.

15. DLQs require operational ownership.

16. Kafka lag is monitored.

17. Scaling is limited by partitions and downstream capacity.

18. Payment operations use provider-supported idempotency.

19. Sensitive information is minimized in events.

20. Recovery is tested rather than assumed.
```

---

# 102. Complete Order Example

A realistic successful transaction:

```text
1. Customer submits order.

2. ALB routes request to order-service.

3. Order Service validates customer/product information.

4. Pricing Service calculates price.

5. Promotion Service calculates discount.

6. Tax Service calculates tax.

7. Order Service writes order + outbox in one transaction.

8. Outbox publisher publishes OrderCreated.

9. Kafka distributes OrderCreated.

10. Inventory Service reserves inventory.

11. Inventory publishes InventoryReserved.

12. Payment Service authorizes payment.

13. Payment publishes PaymentAuthorized.

14. Fraud Service evaluates transaction.

15. Fraud publishes FraudApproved.

16. Shipping Service creates shipment.

17. Shipping publishes ShipmentCreated.

18. Delivery Service tracks delivery.

19. Invoice Service generates invoice.

20. Invoice is stored in S3.

21. Notification Service sends customer notifications through SQS workers.

22. Reporting Service consumes events.

23. Search/Recommendation systems update their projections where applicable.

24. Order reaches COMPLETED after the business workflow finishes.
```

---

# 103. Complete Failure Example

Payment fails:

```text
Customer
   |
   v
OrderCreated
   |
   v
InventoryReserved
   |
   v
PaymentFailed
   |
   +----> ReleaseInventory
   |
   +----> CancelOrder
   |
   +----> Notification
```

No database rollback occurs across services.

Instead:

```text
Compensating transactions
```

restore the desired business state.

---

# 104. What We Have Built

At this point our architecture contains:

```text
AWS
 |
 +--> VPC
 +--> EKS
 +--> RDS
 +--> Redis
 +--> SQS
 +--> SNS
 +--> MSK
 +--> S3
 +--> IAM
 +--> KMS
 +--> ECR
 |
 +--> Terraform
 +--> Helm
 +--> ArgoCD
 |
 +--> Prometheus
 +--> Grafana
 +--> Alertmanager
 +--> ELK
 |
 +--> 21 Microservices
```

And the communication architecture is:

```text
REST/gRPC
   +
Kafka
   +
SQS
   +
SNS
   +
RabbitMQ
```

with:

```text
Transactional Outbox
+
Saga
+
Idempotency
+
Compensation
+
Event Versioning
+
Correlation IDs
+
Observability
```

---

# 105. Next Phase

The next chapters move into the **Observability Production Implementation** phase.

## 118 — Prometheus Production Implementation

We will implement:

```text
Prometheus
 |
 +--> Kubernetes metrics
 +--> Node metrics
 +--> Application metrics
 +--> Kafka metrics
 +--> RDS metrics
 +--> Redis metrics
 +--> EKS metrics
 |
 v
Prometheus
 |
 v
Grafana
```

We will cover:

* Prometheus installation on EKS
* Helm
* kube-state-metrics
* node-exporter
* ServiceMonitor
* PodMonitor
* application `/actuator/prometheus`
* scraping 21 microservices
* recording rules
* alert rules
* retention
* persistent storage
* high availability
* Prometheus resource sizing
* production troubleshooting
* Grafana integration
* Alertmanager integration

```
```
