# 173 — Shipping Service

**File:** `173-Shipping-Service.md`

```markdown
# 173 — Shipping Service

## 1. Overview

The `shipping-service` manages the shipment lifecycle after an order is ready
for fulfillment.

It is responsible for:

- Shipping address validation
- Shipping method selection
- Shipping-rate calculation
- Carrier selection
- Shipment creation
- Carrier tracking number management
- Shipment status
- Carrier API integration
- Carrier webhooks
- Shipping-label metadata
- Shipping events
- Retry and idempotency
- Carrier failure handling
- Shipment reconciliation

It does not own:

- Order lifecycle
- Inventory
- Payment
- Delivery execution
- Customer profile
- Product catalog

Technology:

- TypeScript
- Node.js
- NestJS
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

---

# 2. Why Shipping Is a Separate Service

The shipping domain interacts with external carrier APIs.

Examples:

```text
Order Service
      |
      ↓
Shipping Service
      |
      ├── Carrier A
      ├── Carrier B
      └── Carrier C
```

Different carriers have different:

- APIs
- Authentication
- Rate limits
- Tracking formats
- Error codes
- Service levels
- Webhook formats
- Availability

Therefore carrier-specific logic should not be embedded directly inside
`order-service`.

---

# 3. Shipping Responsibilities

Shipping Service owns:

```text
Shipment
Shipment Item
Shipping Method
Shipping Rate
Carrier
Tracking Number
Shipment Status
Carrier Reference
Shipping Label Metadata
Carrier Webhook
```

Example lifecycle:

```text
CREATED
   |
   ↓
CARRIER_REQUESTED
   |
   ↓
SHIPMENT_CREATED
   |
   ↓
LABEL_READY
   |
   ↓
IN_TRANSIT
   |
   ↓
OUT_FOR_DELIVERY
   |
   ↓
DELIVERED
```

Failure states:

```text
CARRIER_FAILED
CANCELLED
RETURNED
```

---

# 4. High-Level Architecture

```text
                         Order Service
                              |
                              ↓
                       Shipping Service
                    /        |        \
                   /         |         \
                  ↓          ↓          ↓
             PostgreSQL    Redis      Kafka
                  |
                  ↓
            Carrier Adapter
             /     |      \
            ↓      ↓       ↓
        Carrier A Carrier B Carrier C
            |
            ↓
         Webhooks
            |
            ↓
       Shipping Service
```

---

# 5. Shipping and Order Flow

Typical flow:

```text
Order
  |
  ↓
Payment Captured
  |
  ↓
Inventory Confirmed
  |
  ↓
Shipping Requested
  |
  ↓
Shipping Service
  |
  ↓
Carrier
  |
  ↓
Shipment Created
  |
  ↓
Tracking Number
  |
  ↓
Delivery Service
```

Shipping and Delivery are intentionally separate.

Shipping answers:

> Which carrier is transporting the package and what shipment/tracking
> information exists?

Delivery answers:

> What is happening during the physical delivery process?

---

# 6. Shipping vs Delivery

### Shipping Service

Owns:

```text
Carrier selection
Shipment creation
Shipping rates
Tracking number
Carrier communication
Shipping label
```

### Delivery Service

Owns:

```text
Out-for-delivery
Delivery attempts
Delivery confirmation
Proof of delivery
Failed delivery
Customer delivery status
```

Therefore:

```text
Shipping Service
      |
      ↓
Shipment Created
      |
      ↓
Delivery Service
      |
      ↓
Physical Delivery
```

---

# 7. Repository Structure

```text
shipping-service/
│
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   │
│   ├── config/
│   │   └── configuration.ts
│   │
│   ├── database/
│   │   └── database.module.ts
│   │
│   ├── redis/
│   │   ├── redis.module.ts
│   │   └── redis.service.ts
│   │
│   ├── shipping/
│   │   ├── shipping.controller.ts
│   │   ├── shipping.service.ts
│   │   ├── shipping.repository.ts
│   │   ├── shipment.entity.ts
│   │   └── dto/
│   │       ├── create-shipment.dto.ts
│   │       ├── shipping-rate.dto.ts
│   │       └── cancel-shipment.dto.ts
│   │
│   ├── carrier/
│   │   ├── carrier.interface.ts
│   │   ├── carrier.module.ts
│   │   ├── carrier.service.ts
│   │   ├── carrier-a.adapter.ts
│   │   └── carrier-b.adapter.ts
│   │
│   ├── webhook/
│   │   ├── webhook.controller.ts
│   │   └── webhook.service.ts
│   │
│   ├── messaging/
│   │   ├── kafka.service.ts
│   │   └── sqs.service.ts
│   │
│   ├── health/
│   │   └── health.controller.ts
│   │
│   └── observability/
│       └── metrics.service.ts
│
├── migrations/
├── test/
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── package.json
├── package-lock.json
├── tsconfig.json
├── tsconfig.build.json
├── nest-cli.json
├── .env.example
├── README.md
│
├── helm/
│   └── shipping-service/
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

# 8. Technology Stack

```text
TypeScript
    ↓
Node.js
    ↓
NestJS
    ↓
TypeORM
    ↓
PostgreSQL
```

Caching:

```text
NestJS
   ↓
ioredis
   ↓
Redis
```

Messaging:

```text
NestJS
   ↓
Kafka / SQS
```

Infrastructure:

```text
Docker
 ↓
ECR
 ↓
EKS
 ↓
Helm
 ↓
ArgoCD
```

---

# 9. Package Configuration

## `package.json`

```json
{
  "name": "shipping-service",
  "version": "1.0.0",
  "description": "Ecommerce shipping service",
  "scripts": {
    "build": "nest build",
    "start": "node dist/main.js",
    "start:dev": "nest start --watch",
    "test": "jest",
    "test:cov": "jest --coverage",
    "lint": "eslint \"src/**/*.ts\"",
    "format": "prettier --check \"src/**/*.ts\""
  },
  "dependencies": {
    "@nestjs/common": "^11.0.0",
    "@nestjs/config": "^4.0.0",
    "@nestjs/core": "^11.0.0",
    "@nestjs/platform-express": "^11.0.0",
    "@nestjs/typeorm": "^11.0.0",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.1",
    "ioredis": "^5.4.0",
    "pg": "^8.13.0",
    "reflect-metadata": "^0.2.2",
    "rxjs": "^7.8.2",
    "typeorm": "^0.3.20"
  },
  "devDependencies": {
    "@nestjs/cli": "^11.0.0",
    "@nestjs/testing": "^11.0.0",
    "@types/jest": "^29.5.14",
    "@types/node": "^22.0.0",
    "jest": "^29.7.0",
    "prettier": "^3.5.0",
    "ts-jest": "^29.2.5",
    "typescript": "^5.7.0"
  }
}
```

Actual production dependency versions should be locked and tested before
deployment.

---

# 10. Environment Configuration

## `.env.example`

```text
NODE_ENV=development

PORT=8080

DATABASE_URL=postgresql://postgres:postgres@localhost:5432/shipping

REDIS_HOST=localhost
REDIS_PORT=6379

KAFKA_BROKERS=localhost:9092

CARRIER_A_BASE_URL=http://localhost:9001
CARRIER_B_BASE_URL=http://localhost:9002

CARRIER_TIMEOUT_MS=5000

SHIPPING_RATE_CACHE_TTL_SECONDS=300
```

Never commit:

```text
.env
production credentials
carrier API keys
webhook secrets
AWS credentials
```

---

# 11. Configuration Module

## `configuration.ts`

```typescript
export default () => ({
  app: {
    name: 'shipping-service',
    port: parseInt(process.env.PORT ?? '8080', 10),
  },

  database: {
    url: process.env.DATABASE_URL,
  },

  redis: {
    host: process.env.REDIS_HOST ?? 'localhost',
    port: parseInt(
      process.env.REDIS_PORT ?? '6379',
      10,
    ),
  },

  kafka: {
    brokers: (
      process.env.KAFKA_BROKERS ??
      'localhost:9092'
    ).split(','),
  },

  carrier: {
    timeoutMs: parseInt(
      process.env.CARRIER_TIMEOUT_MS ?? '5000',
      10,
    ),
  },

  shipping: {
    rateCacheTtlSeconds: parseInt(
      process.env.SHIPPING_RATE_CACHE_TTL_SECONDS ??
        '300',
      10,
    ),
  },
});
```

---

# 12. Shipment Entity

## `shipment.entity.ts`

```typescript
import {
  Column,
  CreateDateColumn,
  Entity,
  Index,
  PrimaryGeneratedColumn,
  UpdateDateColumn,
} from 'typeorm';

@Entity('shipments')
export class Shipment {
  @PrimaryGeneratedColumn()
  id: number;

  @Index({ unique: true })
  @Column({
    name: 'shipment_reference',
    length: 100,
  })
  shipmentReference: string;

  @Index()
  @Column({
    name: 'order_id',
    length: 100,
  })
  orderId: string;

  @Index()
  @Column({
    name: 'customer_id',
    length: 100,
  })
  customerId: string;

  @Column({
    length: 100,
  })
  carrier: string;

  @Column({
    name: 'carrier_reference',
    length: 200,
    nullable: true,
  })
  carrierReference: string | null;

  @Column({
    name: 'tracking_number',
    length: 200,
    nullable: true,
  })
  trackingNumber: string | null;

  @Column({
    length: 50,
  })
  status: string;

  @Column({
    name: 'shipping_method',
    length: 100,
  })
  shippingMethod: string;

  @Column({
    name: 'shipping_amount',
    type: 'numeric',
    precision: 19,
    scale: 4,
  })
  shippingAmount: string;

  @Column({
    length: 3,
  })
  currency: string;

  @Column({
    name: 'shipping_address',
    type: 'jsonb',
  })
  shippingAddress: Record<string, unknown>;

  @CreateDateColumn({
    name: 'created_at',
  })
  createdAt: Date;

  @UpdateDateColumn({
    name: 'updated_at',
  })
  updatedAt: Date;
}
```

---

# 13. Shipment Status

Recommended statuses:

```text
CREATED
CARRIER_REQUESTED
SHIPMENT_CREATED
LABEL_READY
IN_TRANSIT
OUT_FOR_DELIVERY
DELIVERED
CANCELLED
RETURNED
CARRIER_FAILED
```

The service must validate state transitions.

For example:

```text
DELIVERED
   ↓
CARRIER_FAILED
```

should normally be rejected.

---

# 14. Shipment State Machine

```text
CREATED
   |
   ↓
CARRIER_REQUESTED
   |
   ├── CARRIER_FAILED
   |
   ↓
SHIPMENT_CREATED
   |
   ↓
LABEL_READY
   |
   ↓
IN_TRANSIT
   |
   ↓
OUT_FOR_DELIVERY
   |
   ↓
DELIVERED
```

Return path:

```text
IN_TRANSIT
   |
   ↓
RETURNED
```

Cancellation:

```text
CREATED
   |
   ↓
CANCELLED
```

Allowed transitions should be encoded in application logic.

---

# 15. Shipping Rate API

Endpoint:

```text
GET /api/v1/shipping/rates
```

Example request:

```text
destinationPostalCode=500001
weightKg=2.5
```

Response:

```json
{
  "rates": [
    {
      "carrier": "carrier-a",
      "service": "STANDARD",
      "amount": "120.00",
      "currency": "INR",
      "estimatedDays": 5
    },
    {
      "carrier": "carrier-b",
      "service": "EXPRESS",
      "amount": "240.00",
      "currency": "INR",
      "estimatedDays": 2
    }
  ]
}
```

---

# 16. Shipping Rate Calculation

Rate calculation may depend on:

```text
Origin
Destination
Postal code
Weight
Dimensions
Shipping method
Carrier
Service level
Fuel surcharge
Remote-area surcharge
Business rules
```

Example:

```text
Base rate
+
Fuel surcharge
+
Remote-area surcharge
=
Final shipping cost
```

---

# 17. Rate Caching

Shipping rates can often be cached for a short period.

Example Redis key:

```text
shipping:rate:
<origin>:
<destination>:
<weight>:
<service>
```

TTL:

```text
5 minutes
```

is only an example.

Rates should not be cached longer than the business/provider validity period.

---

# 18. Create Shipment API

```text
POST /api/v1/shipments
```

Request:

```json
{
  "orderId": "ORD-1001",
  "customerId": "CUS-1001",
  "carrier": "carrier-a",
  "shippingMethod": "EXPRESS",
  "shippingAddress": {
    "name": "Customer",
    "addressLine1": "Example Street",
    "city": "Hyderabad",
    "state": "Telangana",
    "postalCode": "500001",
    "country": "IN"
  },
  "items": [
    {
      "productId": "P1001",
      "quantity": 2
    }
  ]
}
```

The service must validate that the caller is authorized and that the order
state permits shipment creation.

---

# 19. Idempotency

Shipment creation must be idempotent.

Example:

```text
POST /api/v1/shipments

Idempotency-Key:
ORDER-1001-SHIPMENT-1
```

Suppose:

```text
Shipping Service
      |
      ↓
Carrier
      |
      ↓
Shipment created
      |
      X
Network timeout
```

Client retries.

Without idempotency:

```text
Shipment A
Shipment B
```

may be created for the same order.

Correct behavior:

```text
Retry
  ↓
same idempotency key
  ↓
existing shipment
```

---

# 20. Idempotency Table

```sql
CREATE TABLE idempotency_records (

    id BIGSERIAL PRIMARY KEY,

    idempotency_key VARCHAR(200)
        NOT NULL,

    operation VARCHAR(100)
        NOT NULL,

    request_hash VARCHAR(128)
        NOT NULL,

    resource_reference VARCHAR(100),

    status VARCHAR(50)
        NOT NULL,

    response_payload TEXT,

    created_at TIMESTAMP
        NOT NULL,

    expires_at TIMESTAMP
        NOT NULL,

    CONSTRAINT uq_shipping_idempotency
        UNIQUE (
            idempotency_key,
            operation
        )
);
```

A reused key with a different request payload must be rejected.

---

# 21. Carrier Abstraction

Do not write:

```text
ShippingService
    ↓
CarrierA SDK everywhere
```

Instead:

```text
ShippingService
       |
       ↓
CarrierAdapter
    /       \
   ↓         ↓
Carrier A  Carrier B
```

Interface:

```typescript
export interface CarrierAdapter {
  getRates(
    request: ShippingRateRequest,
  ): Promise<ShippingRate[]>;

  createShipment(
    request: CreateCarrierShipmentRequest,
  ): Promise<CarrierShipmentResponse>;

  cancelShipment(
    carrierReference: string,
  ): Promise<void>;

  getTracking(
    trackingNumber: string,
  ): Promise<TrackingResponse>;

  verifyWebhook(
    payload: string,
    signature: string,
  ): boolean;
}
```

---

# 22. Why Carrier Abstraction Matters

Without abstraction:

```text
ShippingService
   |
   ├── Carrier A logic
   ├── Carrier B logic
   ├── Carrier C logic
   └── carrier-specific conditions
```

This becomes difficult to maintain.

With adapters:

```text
ShippingService
      |
      ↓
CarrierAdapter
      |
 ┌────┼────┐
 ↓    ↓    ↓
 A    B    C
```

The business logic stays independent from provider-specific APIs.

---

# 23. Carrier Selection

Carrier selection may consider:

```text
Destination
Weight
Package dimensions
Shipping SLA
Carrier availability
Cost
Customer-selected service
Historical carrier performance
```

Example:

```text
Express + Hyderabad
      |
      ├── Carrier A: ₹220, 2 days
      ├── Carrier B: ₹180, 4 days
      └── Carrier C: unavailable
```

Business rule:

```text
customer requested EXPRESS
```

may select Carrier A despite higher cost.

---

# 24. Carrier Timeout

Every carrier API call needs:

```text
Connect timeout
Read timeout
Overall request timeout
```

Example:

```text
5 seconds
```

is illustrative.

Never allow:

```text
carrier request
    ↓
wait indefinitely
```

because this can exhaust Node.js worker resources and cause cascading
latency.

---

# 25. Retry Policy

Retry only transient errors.

Potentially retryable:

```text
connection timeout
temporary network error
HTTP 429
HTTP 502
HTTP 503
HTTP 504
```

Potentially non-retryable:

```text
invalid address
invalid shipment
unsupported postal code
authentication failure
invalid carrier request
```

Use:

```text
Exponential Backoff
+
Jitter
+
Maximum Attempts
```

---

# 26. Carrier Rate Limits

Suppose carrier allows:

```text
100 requests/sec
```

and HPA scales:

```text
3 Pods → 30 Pods
```

That does not mean the service can safely generate:

```text
30 × application throughput
```

against the carrier.

Use:

```text
Concurrency limit
+
Rate limiter
+
Retry budget
```

around carrier integrations.

---

# 27. Redis Usage

Redis can be used for:

```text
Shipping rate cache
Carrier availability cache
Short-lived tracking cache
Rate-limit counters
Idempotency acceleration
```

PostgreSQL remains the durable source of truth for shipments.

---

# 28. PostgreSQL Migration

## `V1__create_shipments.sql`

```sql
CREATE TABLE shipments (

    id BIGSERIAL PRIMARY KEY,

    shipment_reference VARCHAR(100)
        NOT NULL UNIQUE,

    order_id VARCHAR(100)
        NOT NULL,

    customer_id VARCHAR(100)
        NOT NULL,

    carrier VARCHAR(100)
        NOT NULL,

    carrier_reference VARCHAR(200),

    tracking_number VARCHAR(200),

    status VARCHAR(50)
        NOT NULL,

    shipping_method VARCHAR(100)
        NOT NULL,

    shipping_amount NUMERIC(19,4)
        NOT NULL,

    currency VARCHAR(3)
        NOT NULL,

    shipping_address JSONB
        NOT NULL,

    created_at TIMESTAMP
        NOT NULL,

    updated_at TIMESTAMP
        NOT NULL
);

CREATE INDEX idx_shipments_order_id
    ON shipments(order_id);

CREATE INDEX idx_shipments_customer_id
    ON shipments(customer_id);

CREATE INDEX idx_shipments_tracking_number
    ON shipments(tracking_number);

CREATE INDEX idx_shipments_status
    ON shipments(status);
```

---

# 29. Carrier Webhooks

Carrier:

```text
Carrier
   |
   ↓
POST /api/v1/shipping/webhooks/carrier-a
```

Example event:

```json
{
  "eventId": "carrier-event-1001",
  "trackingNumber": "TRK1001",
  "status": "IN_TRANSIT",
  "timestamp": "2026-10-05T13:00:00Z"
}
```

The service must:

```text
Verify signature
      ↓
Validate event
      ↓
Check duplicate event
      ↓
Map carrier status
      ↓
Validate shipment transition
      ↓
Persist status
      ↓
Publish ShippingStatusChanged
```

---

# 30. Webhook Idempotency

Carriers may send:

```text
event-1001
event-1001
event-1001
```

Store:

```text
carrier
provider_event_id
```

with a unique constraint.

This prevents duplicate event processing.

---

# 31. Webhook Security

Never trust:

```json
{
  "status": "DELIVERED"
}
```

without verifying that the event came from the carrier.

Use:

```text
TLS
+
Signature Verification
+
Timestamp/Replay Protection
+
Provider Event ID
```

The exact mechanism depends on the carrier.

---

# 32. Tracking Number

Example:

```text
TRK-ABCD-10001
```

Tracking number is carrier-owned.

Do not generate a fake tracking number merely because the internal shipment
record exists.

Correct:

```text
Shipment created
      |
      ↓
Carrier response
      |
      ↓
carrierReference
trackingNumber
```

---

# 33. Shipping Event

Example:

```json
{
  "eventId": "ship-event-1001",
  "eventType": "ShipmentCreated",
  "version": 1,
  "occurredAt": "2026-10-05T13:00:00Z",
  "source": "shipping-service",
  "correlationId": "corr-order-1001",
  "aggregateId": "SHIP-1001",
  "payload": {
    "orderId": "ORD-1001",
    "shipmentReference": "SHIP-1001",
    "carrier": "carrier-a",
    "trackingNumber": "TRK-10001"
  }
}
```

---

# 34. Kafka Topics

Example:

```text
ecommerce.shipping.events
```

Events:

```text
ShipmentCreated
ShipmentLabelReady
ShipmentDispatched
ShipmentInTransit
ShipmentOutForDelivery
ShipmentDelivered
ShipmentCancelled
ShipmentReturned
ShipmentFailed
```

---

# 35. SQS Usage

SQS is useful for asynchronous carrier operations.

Example:

```text
Shipping Request
      |
      ↓
SQS
      |
      ↓
Shipping Worker
      |
      ↓
Carrier API
```

This is useful when shipment creation does not need to block the user-facing
request.

---

# 36. Synchronous vs Asynchronous Shipping

### Synchronous

Use when:

```text
Customer needs immediate shipping rate
```

Example:

```text
Checkout
 ↓
Shipping rates
 ↓
Response
```

### Asynchronous

Use for:

```text
Shipment creation
Tracking synchronization
Carrier polling
Reconciliation
```

Example:

```text
Order Confirmed
      |
      ↓
Kafka/SQS
      |
      ↓
Shipping Worker
      |
      ↓
Carrier
```

---

# 37. Shipping Controller

```typescript
@Controller('/api/v1/shipments')
export class ShippingController {
  constructor(
    private readonly shippingService: ShippingService,
  ) {}

  @Post()
  async createShipment(
    @Headers('idempotency-key')
    idempotencyKey: string,

    @Body()
    request: CreateShipmentDto,
  ) {
    return this.shippingService.createShipment(
      request,
      idempotencyKey,
    );
  }

  @Get(':shipmentReference')
  async getShipment(
    @Param('shipmentReference')
    shipmentReference: string,
  ) {
    return this.shippingService.getShipment(
      shipmentReference,
    );
  }

  @Get(':shipmentReference/tracking')
  async getTracking(
    @Param('shipmentReference')
    shipmentReference: string,
  ) {
    return this.shippingService.getTracking(
      shipmentReference,
    );
  }
}
```

---

# 38. DTO

```typescript
import {
  IsArray,
  IsNotEmpty,
  IsString,
  ValidateNested,
} from 'class-validator';

import { Type } from 'class-transformer';

export class CreateShipmentDto {
  @IsString()
  @IsNotEmpty()
  orderId: string;

  @IsString()
  @IsNotEmpty()
  customerId: string;

  @IsString()
  @IsNotEmpty()
  carrier: string;

  @IsString()
  @IsNotEmpty()
  shippingMethod: string;

  @IsNotEmpty()
  shippingAddress: Record<string, unknown>;

  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => ShipmentItemDto)
  items: ShipmentItemDto[];
}

export class ShipmentItemDto {
  @IsString()
  productId: string;

  quantity: number;
}
```

---

# 39. Shipping Service

Conceptual implementation:

```typescript
@Injectable()
export class ShippingService {
  constructor(
    private readonly repository: ShippingRepository,
    private readonly carrierService: CarrierService,
  ) {}

  async createShipment(
    request: CreateShipmentDto,
    idempotencyKey: string,
  ) {
    const existing =
      await this.repository.findByIdempotencyKey(
        idempotencyKey,
      );

    if (existing) {
      return existing;
    }

    const carrier =
      this.carrierService.getCarrier(
        request.carrier,
      );

    const providerResponse =
      await carrier.createShipment({
        orderId: request.orderId,
        shippingMethod:
          request.shippingMethod,
        shippingAddress:
          request.shippingAddress,
        items: request.items,
      });

    return this.repository.createShipment({
      ...request,
      carrierReference:
        providerResponse.carrierReference,
      trackingNumber:
        providerResponse.trackingNumber,
      status: 'SHIPMENT_CREATED',
    });
  }
}
```

Production implementation must make the idempotency and persistence boundary
transaction-safe and must handle the case where the carrier succeeds but the
database transaction fails.

---

# 40. The Hardest Shipping Failure

Consider:

```text
Shipping Service
       |
       ↓
Carrier
       |
       ↓
Shipment successfully created
       |
       X
Database connection lost
```

The application sees:

```text
timeout/error
```

but the carrier already created the shipment.

Retrying blindly can create:

```text
Shipment A
Shipment B
```

Therefore use:

```text
Idempotency key
+
carrier request reference
+
reconciliation
```

---

# 41. Shipping Reconciliation

Periodic job:

```text
Unknown/Failed/Pending Shipments
             |
             ↓
        Carrier Query
             |
             ↓
       Compare States
             |
       ┌─────┴─────┐
       ↓           ↓
     Match      Mismatch
                   |
                   ↓
               Investigate
```

Example:

```text
Internal:
SHIP-1001 = CARRIER_REQUESTED

Carrier:
SHIP-1001 = SHIPMENT_CREATED
```

Reconciliation updates the internal state.

---

# 42. Shipping and Inventory

Shipping should not create a shipment before inventory is confirmed.

Recommended:

```text
Order
 ↓
Inventory Reservation
 ↓
Payment
 ↓
Shipping Request
 ↓
Shipment
```

The exact Saga sequence can differ.

Shipping failure should not automatically imply inventory failure.

The Order/Saga orchestration layer owns the business compensation logic.

---

# 43. Shipping and Payment

Shipping should not capture payment.

Instead:

```text
Payment Service
      |
      ↓
Payment Captured
      |
      ↓
Order confirmed
      |
      ↓
Shipping
```

If shipping fails:

```text
Shipping failed
```

the business workflow may:

```text
retry carrier
OR
switch carrier
OR
hold order
OR
cancel order
OR
refund payment
```

The decision belongs to the order/Saga business workflow.

---

# 44. Dockerfile

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY . .

RUN npm run build


FROM node:22-alpine

WORKDIR /app

COPY package*.json ./

RUN npm ci --omit=dev

COPY --from=builder \
    /app/dist \
    ./dist

RUN addgroup \
    -S appgroup \
    && adduser \
    -S appuser \
    -G appgroup

USER appuser

EXPOSE 8080

CMD ["node", "dist/main.js"]
```

Pin a tested Node.js and base-image version for production.

---

# 45. `.dockerignore`

```text
.git
.github

node_modules
dist

coverage

.env
.env.*

*.log

README.md

test
```

---

# 46. Local Docker Compose

```yaml
services:

  postgres:

    image: postgres:16

    container_name: shipping-postgres

    environment:

      POSTGRES_DB: shipping

      POSTGRES_USER: postgres

      POSTGRES_PASSWORD: postgres

    ports:

      - "5432:5432"

    volumes:

      - postgres-data:/var/lib/postgresql/data


  redis:

    image: redis:7

    container_name: shipping-redis

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: shipping-kafka

    ports:

      - "9092:9092"

    # Development-only configuration.
    # Pin the tested Kafka version in the real repository.


  shipping-service:

    build:

      context: .

      dockerfile: Dockerfile

    depends_on:

      - postgres
      - redis
      - kafka

    environment:

      DATABASE_URL:
        postgresql://postgres:postgres@postgres:5432/shipping

      REDIS_HOST: redis

      REDIS_PORT: 6379

      KAFKA_BROKERS: kafka:9092

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

For carrier integration tests, use mocked carrier endpoints.

Do not make real carrier bookings from developer laptops.

---

# 47. Kubernetes Deployment

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: shipping-service

  labels:

    app: shipping-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: shipping-service

  template:

    metadata:

      labels:

        app: shipping-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: shipping-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: IfNotPresent

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: shipping-service-config

            - secretRef:

                name: shipping-service-secrets

          resources:

            {{- toYaml .Values.resources | nindent 12 }}

          startupProbe:

            httpGet:

              path: /health

              port: http

            failureThreshold: 30

            periodSeconds: 10

          readinessProbe:

            httpGet:

              path: /health

              port: http

            initialDelaySeconds: 10

            periodSeconds: 10

          livenessProbe:

            httpGet:

              path: /health

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

# 48. Service

```yaml
apiVersion: v1

kind: Service

metadata:

  name: shipping-service

  labels:

    app: shipping-service

spec:

  type: ClusterIP

  selector:

    app: shipping-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

---

# 49. ConfigMap

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: shipping-service-config

data:

  REDIS_HOST: redis.ecommerce.svc.cluster.local

  REDIS_PORT: "6379"

  KAFKA_BROKERS: kafka.ecommerce.svc.cluster.local:9092

  CARRIER_TIMEOUT_MS: "5000"

  SHIPPING_RATE_CACHE_TTL_SECONDS: "300"
```

---

# 50. ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: shipping-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: shipping-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/shipping-service

        property: DATABASE_URL

    - secretKey: CARRIER_A_API_KEY

      remoteRef:

        key: ecommerce/prod/shipping-service

        property: CARRIER_A_API_KEY

    - secretKey: CARRIER_A_WEBHOOK_SECRET

      remoteRef:

        key: ecommerce/prod/shipping-service

        property: CARRIER_A_WEBHOOK_SECRET
```

---

# 51. ServiceAccount

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: shipping-service

  labels:

    app: shipping-service
```

If AWS APIs are required:

```text
Shipping Pod
    ↓
EKS Pod Identity
    ↓
shipping-service IAM role
    ↓
specific AWS resource
```

No static AWS access keys.

---

# 52. NetworkPolicy

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: shipping-service

spec:

  podSelector:

    matchLabels:

      app: shipping-service

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

The exact egress design depends on whether carrier traffic exits directly,
through a NAT gateway, proxy, firewall, or another controlled egress layer.

---

# 53. Resource Requests and Limits

```yaml
resources:

  requests:

    cpu: 250m

    memory: 256Mi

  limits:

    cpu: "1"

    memory: 512Mi
```

These are starting values.

Production values must come from:

```text
Load testing
+
Performance testing
+
Observed utilization
```

---

# 54. HPA

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: shipping-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: shipping-service

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

Because Shipping calls external carriers, CPU-based HPA alone is not enough.

The platform should eventually consider:

```text
Request rate
Latency
Carrier queue depth
SQS backlog
Provider concurrency
```

---

# 55. PDB

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: shipping-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: shipping-service
```

---

# 56. ServiceMonitor

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: shipping-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: shipping-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

---

# 57. Shipping Metrics

Technical:

```text
shipping_requests_total

shipping_request_duration_seconds

shipping_errors_total

carrier_requests_total

carrier_request_duration_seconds

carrier_errors_total

carrier_timeouts_total

carrier_rate_limit_total
```

Business:

```text
shipments_created_total

shipments_failed_total

shipments_cancelled_total

shipments_delivered_total

shipping_rate_requests_total

shipping_rate_cache_hits_total

shipping_rate_cache_misses_total
```

Reconciliation:

```text
shipping_reconciliation_mismatch_total

shipping_unknown_state_total
```

---

# 58. Grafana Dashboard

```text
Shipping Service
│
├── Request Rate
├── P95
├── P99
├── HTTP 5xx
│
├── Shipment Creation
│   ├── Success
│   ├── Failure
│   └── Pending
│
├── Carrier
│   ├── Requests
│   ├── Latency
│   ├── 4xx
│   ├── 5xx
│   ├── Timeouts
│   └── Rate Limits
│
├── Rates
│   ├── Cache Hit
│   └── Cache Miss
│
├── Webhooks
│   ├── Received
│   ├── Invalid
│   └── Duplicate
│
└── Infrastructure
    ├── CPU
    ├── Memory
    ├── Pods
    ├── PostgreSQL
    └── Redis
```

---

# 59. Structured Logging

Example:

```json
{
  "@timestamp": "2026-10-05T14:00:00Z",
  "level": "INFO",
  "service": "shipping-service",
  "environment": "prod",
  "correlationId": "corr-order-1001",
  "shipmentReference": "SHIP-1001",
  "orderId": "ORD-1001",
  "carrier": "carrier-a",
  "trackingNumber": "TRK-1001",
  "status": "SHIPMENT_CREATED",
  "message": "Shipment created"
}
```

Never log:

```text
carrier API keys
webhook secrets
authorization tokens
customer credentials
unnecessary PII
```

---

# 60. OpenTelemetry

Architecture:

```text
Shipping Service
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

Trace:

```text
Order Service
      |
      ↓
Shipping Service
      |
      ↓
Carrier API
```

Use:

```text
traceId
spanId
correlationId
```

for investigation.

Do not place sensitive address information or credentials into span attributes.

---

# 61. SLO

Example initial targets:

Shipping API availability:

```text
99.95%
```

Shipping rate P95:

```text
< 300ms
```

Shipment request API P95:

```text
< 500ms
```

Carrier webhook processing:

```text
99.9% within defined processing window
```

Shipment reconciliation:

```text
0 unresolved critical mismatches
```

Final targets must be based on business requirements and carrier SLAs.

---

# 62. Important Alerts

```text
High Shipping API 5xx

High Shipping API Latency

High Carrier Timeout Rate

High Carrier 5xx

High Carrier 429

Shipment Creation Failure Spike

Unknown Shipment State Increase

Webhook Verification Failure

Webhook Processing Failure

Reconciliation Mismatch

Redis Failure

PostgreSQL Connection Pressure

Kafka Lag

SQS Backlog
```

---

# 63. Shipping Rate Failure

If carrier rate API fails:

```text
Customer
   |
   ↓
Shipping Rates
   |
   X
Carrier
```

Do not automatically display:

```text
₹0
```

as the shipping rate.

Possible behavior:

```text
Primary carrier fails
       |
       ↓
Try approved secondary carrier
       |
       ↓
Return valid rate
```

If all carriers fail:

```text
return controlled error
```

or use an approved cached rate if its validity is still acceptable.

---

# 64. Carrier Outage

Suppose Carrier A is unavailable.

```text
Carrier A
    X
```

Carrier selection can use:

```text
Carrier A → unhealthy
Carrier B → healthy
Carrier C → healthy
```

Then:

```text
Shipping Service
      |
      ├── A → unavailable
      |
      └── B → selected
```

Carrier health must not be inferred solely from one failed request.

Use:

```text
timeouts
error rate
provider health
circuit breaker
```

---

# 65. Circuit Breaker

```text
Carrier A
    |
    ↓
Circuit Breaker
    |
    ├── CLOSED
    |
    ├── OPEN
    |
    └── HALF_OPEN
```

If Carrier A consistently fails:

```text
OPEN
```

Requests fail fast or route to an approved alternate carrier.

This prevents all application Pods from waiting on a broken carrier.

---

# 66. Shipping and SQS

Example asynchronous shipment creation:

```text
Order Service
     |
     ↓
ShippingRequested
     |
     ↓
SQS
     |
     ↓
Shipping Worker
     |
     ↓
Carrier
```

Consumer behavior:

```text
Receive
  ↓
Validate
  ↓
Check idempotency
  ↓
Call carrier
  ↓
Persist result
  ↓
Publish event
  ↓
Delete SQS message
```

Delete the message only after successful processing.

---

# 67. SQS Visibility Timeout

If carrier processing takes:

```text
20 seconds
```

the visibility timeout must provide enough time for the worker to finish.

But it should not be arbitrarily huge.

Otherwise failed processing may take too long to become visible again.

Monitor:

```text
ApproximateAgeOfOldestMessage
ApproximateNumberOfMessagesVisible
```

and configure DLQ handling.

---

# 68. DLQ

```text
SQS
 |
 ├── Success → delete
 |
 └── repeated failure
          |
          ↓
         DLQ
```

DLQ reasons may include:

```text
invalid shipment
persistent carrier failure
poison message
schema mismatch
unexpected application bug
```

Do not automatically replay all DLQ messages into production.

Investigate first.

---

# 69. Shipping Reconciliation Job

A scheduled worker can identify:

```text
CARRIER_REQUESTED
UNKNOWN
SHIPMENT_CREATED without tracking
```

older than the expected processing window.

Then:

```text
Shipping Service
      |
      ↓
Carrier status API
      |
      ↓
Compare
      |
      ↓
Update
```

This protects against:

```text
carrier success
+
application timeout
```

---

# 70. Shipping Incident Example

## Incident ID

```text
INC-2026-SHIPPING-001
```

## Severity

```text
SEV-2
```

## Title

```text
Carrier API Timeout Caused Shipment Creation Backlog
```

## Date

```text
2026-10-05
```

---

# 71. Customer Impact

Customers experienced:

- Delayed shipment confirmation
- Missing tracking numbers for some orders
- Increased checkout/order-support inquiries
- Delayed warehouse handoff

No confirmed duplicate shipment creation occurred.

---

# 72. Timeline

```text
15:00
Shipping service healthy.

15:04
Carrier A latency begins increasing.

15:06
Carrier timeout rate increases.

15:07
Shipment creation latency increases.

15:08
SQS backlog starts increasing.

15:10
SEV-2 declared.

15:12
Platform confirms EKS healthy.

15:14
Shipping team confirms Carrier A degradation.

15:16
Carrier A circuit breaker opens.

15:17
New eligible shipments route to Carrier B.

15:20
SQS backlog stops increasing.

15:25
Carrier A begins recovering.

15:30
Reconciliation checks unknown shipments.

15:35
All unknown shipments reconciled.

15:40
Backlog returns to normal.

15:45
Incident resolved.
```

---

# 73. Root Cause

Carrier A experienced elevated API latency and intermittent timeouts.

The shipping service initially continued waiting for Carrier A requests until the
configured timeout.

This increased worker utilization and caused the asynchronous shipment queue
to grow.

---

# 74. Amplification Mechanism

```text
Carrier latency
      ↓
Worker waits
      ↓
Fewer available workers
      ↓
SQS backlog
      ↓
Shipment confirmation delay
```

After circuit breaking and carrier failover:

```text
Carrier A
   X
   ↓
Carrier B
   ↓
Normal processing
```

---

# 75. What Went Well

- SQS protected the user-facing order path
- Carrier circuit breaker eventually activated
- No duplicate shipment creation
- Idempotency protected retries
- Reconciliation found unknown shipments
- Carrier B was available

---

# 76. What Went Wrong

- Carrier health detection was too slow
- Circuit-breaker thresholds were not tuned
- Carrier-specific dashboard needed improvement
- Worker concurrency was too tightly coupled to carrier latency

---

# 77. Five Whys

### Why were shipments delayed?

Carrier requests took too long.

### Why?

Carrier A experienced elevated latency.

### Why did this create a backlog?

Workers waited for carrier responses.

### Why didn't traffic immediately move to Carrier B?

Carrier health detection and circuit breaking were too slow.

### Why?

Carrier-specific dependency SLOs and failover automation were insufficient.

---

# 78. Corrective Actions

| Action | Owner |
|---|---|
| Improve carrier health detection | Platform |
| Tune circuit breaker | Backend |
| Add carrier-specific SLO | SRE |
| Add carrier failover policy | Shipping |
| Add carrier latency dashboard | SRE |
| Add load tests with carrier latency | Performance |
| Test carrier outage regularly | SRE |
| Improve reconciliation | Shipping |
| Add DLQ operational runbook | Platform |

---

# 79. Bad Deployment Scenario

Suppose version:

```text
shipping-service:1.8.0
```

contains a carrier mapping bug:

```text
EXPRESS → STANDARD
```

Kubernetes:

```text
Healthy
```

Prometheus:

```text
CPU normal
Memory normal
Pods normal
```

But business behavior is wrong.

Therefore deployment validation must include:

```text
Business smoke test
```

such as:

```text
Request EXPRESS
 ↓
Verify carrier service = EXPRESS
```

Infrastructure health alone is not sufficient.

---

# 80. Rollback

Production flow:

```text
GitOps commit
     ↓
ArgoCD
     ↓
Bad shipping version
```

Rollback:

```text
Revert GitOps commit
     ↓
ArgoCD
     ↓
Known-good image
```

Do not rely only on:

```text
kubectl rollout undo
```

because GitOps can reconcile the undesired state again.

---

# 81. Shipping Security

```text
[ ] Carrier credentials in Secrets Manager
[ ] External Secrets
[ ] EKS Pod Identity where AWS access is needed
[ ] No credentials in Git
[ ] No credentials in images
[ ] Webhook signature verification
[ ] TLS
[ ] NetworkPolicy
[ ] Least privilege IAM
[ ] Non-root container
[ ] Read-only filesystem
[ ] Trivy
[ ] SonarQube
[ ] Dependency scanning
[ ] Secret scanning
[ ] Audit logs
```

---

# 82. Shipping Data Protection

Shipping addresses contain personal information.

Therefore:

```text
Do not log full address unnecessarily.
Do not put address into traces.
Do not expose internal shipping data through public APIs.
Restrict database access.
Use encryption at rest.
Use TLS in transit.
Apply retention policies.
```

The exact retention period depends on legal/business requirements.

---

# 83. Capacity Planning

Suppose:

```text
Shipping Pods = 10
```

Each Node.js process maintains:

```text
PostgreSQL pool = 10
```

Potential connections:

```text
10 × 10
=
100
```

Now add:

```text
HPA
+
deployment surge
+
background workers
```

Database capacity can be exceeded.

Therefore:

```text
HPA max replicas
+
DB connection pool
+
RDS connection budget
```

must be planned together.

---

# 84. External Carrier Capacity

Application capacity:

```text
1000 RPS
```

does not mean:

```text
Carrier API = 1000 RPS
```

The carrier may allow:

```text
100 RPS
```

Therefore implement:

```text
Application
    |
    ↓
Rate Limiter
    |
    ↓
Carrier
```

and use asynchronous queues for workloads that don't need immediate results.

---

# 85. CI Pipeline

## `.github/workflows/ci.yaml`

```yaml
name: Shipping Service CI

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

  ECR_REPOSITORY: shipping-service


jobs:

  test:

    runs-on: ubuntu-latest

    steps:

      - name: Checkout

        uses: actions/checkout@v4


      - name: Setup Node.js

        uses: actions/setup-node@v4

        with:

          node-version: "22"

          cache: npm


      - name: Install dependencies

        run: npm ci


      - name: Lint

        run: npm run lint


      - name: Test

        run: npm run test -- --runInBand


      - name: Build

        run: npm run build


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

Production repositories should pin action and dependency versions according to
the organization's supply-chain policy.

---

# 86. SonarQube

SonarQube runs centrally.

Pipeline:

```text
Checkout
 ↓
npm ci
 ↓
Lint
 ↓
Unit tests
 ↓
Coverage
 ↓
SonarQube
 ↓
Trivy FS
 ↓
Docker build
 ↓
Trivy image
 ↓
ECR
```

SonarQube should not be installed inside every application Pod.

---

# 87. ECR

Repository:

```text
company-ecommerce/shipping-service
```

Example:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/shipping-service:<git-sha>
```

Use:

```text
immutable tags
+
image digest
```

where supported by the GitOps promotion model.

---

# 88. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: shipping-service

description: Ecommerce shipping service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 89. Helm Production Values

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/shipping-service

  tag: ""

  pullPolicy: IfNotPresent


resources:

  requests:

    cpu: 250m

    memory: 256Mi

  limits:

    cpu: "1"

    memory: 512Mi


serviceAccount:

  create: true

  name: shipping-service


autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 15

  targetCPUUtilizationPercentage: 60
```

---

# 90. GitOps

Repository:

```text
company-ecommerce/gitops
```

Structure:

```text
gitops/
└── apps/
    └── shipping-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Flow:

```text
Developer
   ↓
GitHub PR
   ↓
CI
   ↓
Trivy/SonarQube
   ↓
Docker
   ↓
ECR
   ↓
GitOps PR
   ↓
ArgoCD
   ↓
EKS
```

---

# 91. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: shipping-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/shipping-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

Production promotion should use the project's approval policy.

---

# 92. Production Troubleshooting

## Problem: Shipping API returns 500

Check:

```bash
kubectl get pods -n ecommerce -l app=shipping-service

kubectl logs -n ecommerce \
  deploy/shipping-service \
  --tail=200

kubectl describe pod \
  -n ecommerce \
  -l app=shipping-service
```

Then check:

```text
PostgreSQL
Redis
Kafka
Carrier API
Secrets
NetworkPolicy
Recent deployment
```

---

# 93. Problem: Carrier API Timeout

Check:

```text
Carrier latency
Carrier 5xx
Carrier 429
Circuit breaker
DNS
TLS
Network path
NAT/proxy
Provider status
```

Do not immediately increase:

```text
timeout
retries
HPA
```

because this may increase the incident.

---

# 94. Problem: SQS Backlog

Check:

```text
Message age
Visible messages
In-flight messages
Consumer replicas
Worker errors
Carrier latency
DLQ
```

Then determine:

```text
Is consumer slow?
Is carrier slow?
Is database slow?
Are messages failing?
```

Scaling consumers without considering carrier rate limits can make the
incident worse.

---

# 95. Problem: Duplicate Shipment

Check:

```text
Idempotency key
Carrier request ID
Carrier reference
Database shipment records
Application logs
Correlation ID
```

Never simply delete one shipment record.

Determine whether the carrier actually created both shipments.

---

# 96. Problem: Tracking Number Missing

Flow:

```text
Shipment
 ↓
carrierReference
 ↓
trackingNumber
```

Check:

```text
Carrier response
Database
Webhook
Reconciliation
Carrier status API
```

A shipment can exist even when tracking information has not yet propagated.

---

# 97. Production Deployment Checklist

```text
[ ] Unit tests passing
[ ] Integration tests passing
[ ] Carrier mocks tested
[ ] Idempotency tested
[ ] Duplicate webhook tested
[ ] Carrier timeout tested
[ ] Carrier 429 tested
[ ] Carrier 5xx tested
[ ] Reconciliation tested
[ ] DLQ tested
[ ] Trivy passed
[ ] SonarQube passed
[ ] Image pushed to ECR
[ ] Image digest recorded
[ ] GitOps PR approved
[ ] ArgoCD healthy
[ ] Pods ready
[ ] HPA configured
[ ] PDB configured
[ ] NetworkPolicy validated
[ ] Secrets synced
[ ] Prometheus metrics available
[ ] Grafana dashboard available
[ ] Logs visible in ELK
[ ] Traces visible in Jaeger
[ ] Alerts configured
[ ] Runbook available
```

---

# 98. Shipping Production Principles

1. Shipping is a separate domain from Order and Delivery.
2. Carrier integrations must use adapters.
3. Never trust a carrier webhook without verification.
4. Shipment creation must be idempotent.
5. Carrier timeouts must not automatically mean shipment failure.
6. Unknown shipment states require reconciliation.
7. PostgreSQL is the durable shipment source of truth.
8. Redis is an optimization layer.
9. Kafka distributes shipping events.
10. SQS is useful for asynchronous carrier work.
11. Carrier rate limits must be respected.
12. Use bounded retries with backoff and jitter.
13. Use circuit breakers for unhealthy carriers.
14. Do not blindly increase HPA during carrier incidents.
15. More Pods can increase carrier pressure.
16. More workers can increase carrier rate-limit violations.
17. Shipping addresses require data protection.
18. Never log carrier secrets.
19. Webhook events must be idempotent.
20. Reconciliation is required for distributed transactions.
21. Kubernetes health does not prove carrier health.
22. Business SLOs must include shipment success and latency.
23. Carrier outages must be tested.
24. GitOps remains the production deployment source of truth.
25. A successful Kubernetes deployment does not guarantee a successful
    shipment workflow.

# End of 173 — Shipping Service
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
172 Fraud Service           ✅
173 Shipping Service        ✅

174 Delivery Service        ⏳
175 Notification Service    ⏳
176 Search Service          ⏳
177 Recommendation Service  ⏳
178 Review Service          ⏳
179 Tax Service             ⏳
180 Invoice Service         ⏳
181 Reporting Service       ⏳
```

**Next: `174-Delivery-Service.md`** — delivery lifecycle, tracking updates, delivery attempts, proof of delivery, carrier events, failed delivery/return flows, Kafka/SQS, PostgreSQL/Redis, idempotency, external integrations, Kubernetes, CI/CD, GitOps, observability, production failure scenarios and RCA.