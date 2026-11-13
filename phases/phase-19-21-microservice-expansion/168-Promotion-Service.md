# 168 — Promotion Service

**File:** `168-Promotion-Service.md`

```markdown
# 168 — Promotion Service

## 1. Overview

The `promotion-service` manages promotional campaigns and discount rules for
the Enterprise E-Commerce Platform.

The service is responsible for determining whether a promotion is applicable
to a customer/cart/order and calculating the discount according to the
configured promotion rules.

Technology:

- TypeScript
- Node.js
- NestJS
- PostgreSQL
- Redis
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

The service is intentionally implemented differently from the previous
services:

```text
161 Auth       → Java / Spring Boot
162 User       → Python / FastAPI
163 Customer   → TypeScript / NestJS
164 Catalog    → Go
165 Product    → Java / Spring Boot
166 Inventory  → Go
167 Pricing    → Python / FastAPI
168 Promotion  → TypeScript / Node.js / NestJS
```

This demonstrates how the same production DevOps platform can operate
microservices written in different programming languages.

---

# 2. Business Responsibility

Promotion Service owns:

- Promotion definitions
- Coupon codes
- Promotion eligibility
- Promotion validity
- Promotion priority
- Discount calculation
- Usage limits
- Customer usage limits
- Product/category applicability
- Promotion activation/deactivation

It does NOT own:

- Product master data
- Inventory
- Base product price
- Customer profile
- Tax
- Payment
- Order persistence

Architecture:

```text
catalog-service
       |
       ↓
product-service
       |
       ↓
pricing-service
       |
       ↓
promotion-service
       |
       ├── promotion rules
       ├── coupons
       └── usage limits
```

---

# 3. Why Promotion Is Separate From Pricing

Pricing answers:

> What is the current price?

Promotion answers:

> Is this customer/order eligible for a discount?

Example:

```text
Product price = ₹10,000

Promotion:
GOLD10

Discount:
10%

Final:
₹9,000
```

Keeping these responsibilities separate prevents the pricing service from
becoming a giant business-rule engine.

---

# 4. Promotion Examples

## Percentage Discount

```text
PROMO10
10% off
```

## Fixed Discount

```text
FLAT500
₹500 off
```

## Minimum Cart Value

```text
CART1000

Minimum:
₹5,000

Discount:
₹1,000
```

## Customer Tier

```text
GOLD20

Applicable:
GOLD customers

Discount:
20%
```

## Product Specific

```text
PHONE10

Applicable:
Selected products only
```

---

# 5. High-Level Architecture

```text
                         Internet
                            |
                     CloudFront / WAF
                            |
                           ALB
                            |
                            ↓
                   promotion-service
                      /          \
                     /            \
                    ↓              ↓
                 Redis         PostgreSQL
                    |
                    ↓
              Cached Promotion
```

Promotion requests are usually read-heavy.

Redis is therefore used for:

- Promotion lookup
- Coupon lookup
- Frequently accessed promotion rules
- Short-lived eligibility data

PostgreSQL remains the source of truth.

---

# 6. Promotion Request Flow

```text
Order Service
     |
     ↓
promotion-service
     |
     ↓
Redis
     |
     ├── HIT
     |    ↓
     |  Calculate
     |
     └── MISS
          ↓
      PostgreSQL
          ↓
       Cache
          ↓
       Calculate
```

---

# 7. Important Business Constraint

A promotion must never be treated as:

```text
just a Redis value
```

For example:

```text
PROMO10
usage limit = 100
```

If 500 requests arrive simultaneously, Redis caching alone must not allow:

```text
500 successful redemptions
```

when only 100 are permitted.

Usage limits require authoritative concurrency control.

---

# 8. Repository Structure

```text
promotion-service/
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
│   ├── promotion/
│   │   ├── promotion.controller.ts
│   │   ├── promotion.service.ts
│   │   ├── promotion.repository.ts
│   │   ├── promotion.entity.ts
│   │   └── dto/
│   │       ├── create-promotion.dto.ts
│   │       └── apply-promotion.dto.ts
│   │
│   ├── coupon/
│   │   ├── coupon.controller.ts
│   │   ├── coupon.service.ts
│   │   ├── coupon.entity.ts
│   │   └── dto/
│   │
│   ├── health/
│   │   └── health.controller.ts
│   │
│   └── observability/
│       └── metrics.service.ts
│
├── migrations/
│   ├── 001-create-promotions.sql
│   ├── 002-create-coupons.sql
│   └── 003-create-promotion-usage.sql
│
├── test/
│   ├── promotion.service.spec.ts
│   └── promotion.controller.spec.ts
│
├── Dockerfile
├── .dockerignore
├── docker-compose.yaml
├── package.json
├── package-lock.json
├── tsconfig.json
├── tsconfig.build.json
├── .env.example
├── nest-cli.json
├── README.md
│
├── helm/
│   └── promotion-service/
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

# 9. Node.js + TypeScript + NestJS

The application stack is:

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

Dependency management:

```text
package.json
+
package-lock.json
```

---

# 10. `package.json`

```json
{
  "name": "promotion-service",
  "version": "1.0.0",
  "description": "Ecommerce promotion service",
  "private": true,
  "scripts": {
    "build": "nest build",
    "start": "node dist/main.js",
    "start:dev": "nest start --watch",
    "start:prod": "node dist/main.js",
    "lint": "eslint \"{src,test}/**/*.ts\"",
    "format": "prettier --write \"src/**/*.ts\"",
    "format:check": "prettier --check \"src/**/*.ts\"",
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage"
  },
  "dependencies": {
    "@nestjs/common": "^11.0.0",
    "@nestjs/config": "^4.0.0",
    "@nestjs/core": "^11.0.0",
    "@nestjs/platform-express": "^11.0.0",
    "@nestjs/typeorm": "^11.0.0",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.1",
    "ioredis": "^5.4.1",
    "pg": "^8.13.1",
    "reflect-metadata": "^0.2.2",
    "rxjs": "^7.8.1",
    "typeorm": "^0.3.25"
  },
  "devDependencies": {
    "@nestjs/cli": "^11.0.0",
    "@nestjs/schematics": "^11.0.0",
    "@nestjs/testing": "^11.0.0",
    "@types/jest": "^29.5.14",
    "@types/node": "^22.10.0",
    "jest": "^29.7.0",
    "prettier": "^3.5.0",
    "ts-jest": "^29.2.5",
    "typescript": "^5.7.0"
  }
}
```

Production projects should lock exact dependency versions through
`package-lock.json` and review dependency updates through PRs.

---

# 11. Environment Configuration

## `.env.example`

```text
APP_NAME=promotion-service
NODE_ENV=local
PORT=8080

DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=promotions
DATABASE_USER=postgres
DATABASE_PASSWORD=postgres

REDIS_HOST=localhost
REDIS_PORT=6379

PROMOTION_CACHE_TTL_SECONDS=60
```

Real credentials must never be committed.

---

# 12. Configuration

## `src/config/configuration.ts`

```typescript
export default () => ({
  app: {
    name: process.env.APP_NAME || 'promotion-service',
    environment: process.env.NODE_ENV || 'local',
    port: Number(process.env.PORT || 8080),
  },

  database: {
    host: process.env.DATABASE_HOST,
    port: Number(process.env.DATABASE_PORT || 5432),
    name: process.env.DATABASE_NAME,
    user: process.env.DATABASE_USER,
    password: process.env.DATABASE_PASSWORD,
  },

  redis: {
    host: process.env.REDIS_HOST,
    port: Number(process.env.REDIS_PORT || 6379),
  },

  promotion: {
    cacheTtl: Number(
      process.env.PROMOTION_CACHE_TTL_SECONDS || 60,
    ),
  },
});
```

---

# 13. Promotion Entity

## `src/promotion/promotion.entity.ts`

```typescript
import {
  Column,
  CreateDateColumn,
  Entity,
  PrimaryGeneratedColumn,
  UpdateDateColumn,
} from 'typeorm';

@Entity('promotions')
export class Promotion {

  @PrimaryGeneratedColumn()
  id: number;

  @Column({
    unique: true,
    length: 100,
  })
  code: string;

  @Column({
    length: 30,
  })
  type: string;

  @Column({
    type: 'numeric',
    precision: 19,
    scale: 4,
  })
  value: string;

  @Column({
    type: 'numeric',
    precision: 19,
    scale: 4,
    nullable: true,
  })
  minimumCartValue: string | null;

  @Column({
    type: 'integer',
    nullable: true,
  })
  maxUsage: number | null;

  @Column({
    type: 'integer',
    default: 0,
  })
  currentUsage: number;

  @Column({
    type: 'integer',
    nullable: true,
  })
  maxUsagePerCustomer: number | null;

  @Column({
    length: 30,
    default: 'ACTIVE',
  })
  status: string;

  @Column({
    type: 'timestamptz',
  })
  startsAt: Date;

  @Column({
    type: 'timestamptz',
  })
  endsAt: Date;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

---

# 14. Promotion Types

Supported examples:

```text
PERCENTAGE
FIXED_AMOUNT
FREE_SHIPPING
BUY_X_GET_Y
```

Example:

```text
PROMO10
type = PERCENTAGE
value = 10
```

means:

```text
10% discount
```

---

# 15. Database Migration

## `001-create-promotions.sql`

```sql
CREATE TABLE promotions (

    id BIGSERIAL PRIMARY KEY,

    code VARCHAR(100) NOT NULL UNIQUE,

    type VARCHAR(30) NOT NULL,

    value NUMERIC(19,4) NOT NULL,

    minimum_cart_value NUMERIC(19,4),

    max_usage INTEGER,

    current_usage INTEGER NOT NULL DEFAULT 0,

    max_usage_per_customer INTEGER,

    status VARCHAR(30) NOT NULL DEFAULT 'ACTIVE',

    starts_at TIMESTAMPTZ NOT NULL,

    ends_at TIMESTAMPTZ NOT NULL,

    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    updated_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_promotions_status
ON promotions(status);

CREATE INDEX idx_promotions_dates
ON promotions(starts_at, ends_at);
```

---

# 16. Coupon Usage

A promotion with a usage limit requires a usage table.

## `003-create-promotion-usage.sql`

```sql
CREATE TABLE promotion_usage (

    id BIGSERIAL PRIMARY KEY,

    promotion_id BIGINT NOT NULL,

    customer_id VARCHAR(100) NOT NULL,

    order_id VARCHAR(100) NOT NULL,

    usage_count INTEGER NOT NULL DEFAULT 1,

    created_at TIMESTAMPTZ NOT NULL
        DEFAULT CURRENT_TIMESTAMP,

    UNIQUE (
        promotion_id,
        customer_id,
        order_id
    )
);

CREATE INDEX idx_promotion_usage_customer
ON promotion_usage(customer_id);

CREATE INDEX idx_promotion_usage_promotion
ON promotion_usage(promotion_id);
```

The unique constraint protects against duplicate usage records.

---

# 17. DTO — Apply Promotion

## `src/promotion/dto/apply-promotion.dto.ts`

```typescript
import {
  IsNumber,
  IsString,
  Min,
} from 'class-validator';

export class ApplyPromotionDto {

  @IsString()
  code: string;

  @IsString()
  customerId: string;

  @IsNumber()
  @Min(0)
  cartValue: number;
}
```

For production monetary calculations, the service should use decimal-safe
representations rather than JavaScript floating-point arithmetic.

---

# 18. Promotion Repository

## `src/promotion/promotion.repository.ts`

```typescript
import {
  Injectable,
} from '@nestjs/common';

import {
  DataSource,
  Repository,
} from 'typeorm';

import { Promotion } from './promotion.entity';

@Injectable()
export class PromotionRepository
  extends Repository<Promotion> {

  constructor(
    dataSource: DataSource,
  ) {

    super(
      Promotion,
      dataSource.createEntityManager(),
    );
  }

  async findActivePromotion(
    code: string,
  ): Promise<Promotion | null> {

    const now = new Date();

    return this.createQueryBuilder('promotion')
      .where(
        'promotion.code = :code',
        { code },
      )
      .andWhere(
        'promotion.status = :status',
        { status: 'ACTIVE' },
      )
      .andWhere(
        'promotion.startsAt <= :now',
        { now },
      )
      .andWhere(
        'promotion.endsAt >= :now',
        { now },
      )
      .getOne();
  }
}
```

---

# 19. Redis Service

## `src/redis/redis.service.ts`

```typescript
import {
  Injectable,
  OnModuleDestroy,
} from '@nestjs/common';

import Redis from 'ioredis';

@Injectable()
export class RedisService
  implements OnModuleDestroy {

  private readonly client: Redis;

  constructor() {

    this.client = new Redis({
      host:
        process.env.REDIS_HOST || 'localhost',

      port:
        Number(
          process.env.REDIS_PORT || 6379,
        ),

      maxRetriesPerRequest: 2,

      connectTimeout: 1000,

      commandTimeout: 1000,
    });
  }

  async get(
    key: string,
  ): Promise<string | null> {

    return this.client.get(key);
  }

  async set(
    key: string,
    value: string,
    ttl: number,
  ): Promise<void> {

    await this.client.set(
      key,
      value,
      'EX',
      ttl,
    );
  }

  async del(
    key: string,
  ): Promise<void> {

    await this.client.del(key);
  }

  async onModuleDestroy() {

    await this.client.quit();
  }
}
```

---

# 20. Cache Key

Example:

```text
promotion:PROMO10
```

For customer-specific eligibility:

```text
promotion:PROMO10:customer:C1001
```

Do not create unlimited high-cardinality keys without TTL.

---

# 21. Promotion Service

## `src/promotion/promotion.service.ts`

```typescript
import {
  ConflictException,
  Injectable,
  NotFoundException,
} from '@nestjs/common';

import { DataSource } from 'typeorm';

import { PromotionRepository } from './promotion.repository';

import { RedisService } from '../redis/redis.service';

import {
  ApplyPromotionDto,
} from './dto/apply-promotion.dto';

@Injectable()
export class PromotionService {

  constructor(
    private readonly repository:
      PromotionRepository,

    private readonly redis:
      RedisService,

    private readonly dataSource:
      DataSource,
  ) {}

  async getPromotion(
    code: string,
  ) {

    const cacheKey =
      `promotion:${code}`;

    const cached =
      await this.redis.get(
        cacheKey,
      );

    if (cached) {

      return JSON.parse(cached);
    }

    const promotion =
      await this.repository
        .findActivePromotion(code);

    if (!promotion) {

      throw new NotFoundException(
        'Promotion not found',
      );
    }

    await this.redis.set(
      cacheKey,
      JSON.stringify(promotion),
      60,
    );

    return promotion;
  }

  async applyPromotion(
    dto: ApplyPromotionDto,
  ) {

    const promotion =
      await this.getPromotion(
        dto.code,
      );

    if (
      promotion.minimumCartValue &&
      dto.cartValue <
        Number(
          promotion.minimumCartValue,
        )
    ) {

      throw new ConflictException(
        'Minimum cart value not met',
      );
    }

    const discount =
      this.calculateDiscount(
        promotion,
        dto.cartValue,
      );

    return {
      promotionCode:
        promotion.code,

      discount,

      finalValue:
        dto.cartValue - discount,
    };
  }

  private calculateDiscount(
    promotion: any,
    cartValue: number,
  ): number {

    if (
      promotion.type ===
      'PERCENTAGE'
    ) {

      return Math.min(
        cartValue,
        cartValue *
          Number(
            promotion.value,
          ) /
          100,
      );
    }

    if (
      promotion.type ===
      'FIXED_AMOUNT'
    ) {

      return Math.min(
        cartValue,
        Number(
          promotion.value,
        ),
      );
    }

    return 0;
  }
}
```

The production implementation should replace JavaScript `number` arithmetic for
money with a decimal-safe approach.

---

# 22. Controller

## `src/promotion/promotion.controller.ts`

```typescript
import {
  Body,
  Controller,
  Get,
  Param,
  Post,
} from '@nestjs/common';

import {
  ApplyPromotionDto,
} from './dto/apply-promotion.dto';

import {
  PromotionService,
} from './promotion.service';

@Controller(
  'api/v1/promotions',
)
export class PromotionController {

  constructor(
    private readonly service:
      PromotionService,
  ) {}

  @Get(':code')
  async getPromotion(
    @Param('code')
    code: string,
  ) {

    return this.service
      .getPromotion(code);
  }

  @Post('apply')
  async applyPromotion(
    @Body()
    dto: ApplyPromotionDto,
  ) {

    return this.service
      .applyPromotion(dto);
  }
}
```

---

# 23. NestJS Module

## `src/app.module.ts`

```typescript
import {
  Module,
} from '@nestjs/common';

import {
  ConfigModule,
} from '@nestjs/config';

import {
  TypeOrmModule,
} from '@nestjs/typeorm';

import configuration
  from './config/configuration';

import {
  Promotion,
} from './promotion/promotion.entity';

import {
  PromotionController,
} from './promotion/promotion.controller';

import {
  PromotionRepository,
} from './promotion/promotion.repository';

import {
  PromotionService,
} from './promotion/promotion.service';

import {
  RedisService,
} from './redis/redis.service';

@Module({

  imports: [

    ConfigModule.forRoot({
      isGlobal: true,
      load: [configuration],
    }),

    TypeOrmModule.forRoot({
      type: 'postgres',

      host:
        process.env.DATABASE_HOST,

      port:
        Number(
          process.env.DATABASE_PORT || 5432,
        ),

      username:
        process.env.DATABASE_USER,

      password:
        process.env.DATABASE_PASSWORD,

      database:
        process.env.DATABASE_NAME,

      entities: [
        Promotion,
      ],

      synchronize: false,

      autoLoadEntities: true,
    }),

    TypeOrmModule.forFeature([
      Promotion,
    ]),
  ],

  controllers: [
    PromotionController,
  ],

  providers: [
    PromotionRepository,
    PromotionService,
    RedisService,
  ],
})
export class AppModule {}
```

Production schema changes must be handled through migrations, not:

```typescript
synchronize: true
```

---

# 24. Application Entry Point

## `src/main.ts`

```typescript
import {
  ValidationPipe,
} from '@nestjs/common';

import {
  NestFactory,
} from '@nestjs/core';

import {
  AppModule,
} from './app.module';

async function bootstrap() {

  const app =
    await NestFactory.create(
      AppModule,
    );

  app.setGlobalPrefix('');

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      transform: true,
      forbidNonWhitelisted: true,
    }),
  );

  await app.listen(
    Number(
      process.env.PORT || 8080,
    ),
    '0.0.0.0',
  );
}

bootstrap();
```

---

# 25. Health Endpoint

Create:

```text
GET /health
GET /ready
```

Example controller:

## `src/health/health.controller.ts`

```typescript
import {
  Controller,
  Get,
} from '@nestjs/common';

@Controller()
export class HealthController {

  @Get('health')
  health() {

    return {
      status: 'UP',
    };
  }

  @Get('ready')
  ready() {

    return {
      status: 'READY',
    };
  }
}
```

Add this controller to `AppModule`.

---

# 26. Dockerfile

## `Dockerfile`

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json ./

RUN npm ci

COPY tsconfig*.json ./
COPY nest-cli.json ./

COPY src ./src

RUN npm run build


FROM node:22-alpine

WORKDIR /app

ENV NODE_ENV=production

COPY package*.json ./

RUN npm ci --omit=dev \
    && npm cache clean --force

COPY --from=builder \
    /app/dist \
    ./dist

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

CMD [
  "node",
  "dist/main.js"
]
```

For an even smaller hardened production image, the project can later use a
minimal runtime image after validating native dependency compatibility.

---

# 27. `.dockerignore`

```text
.git
.github

node_modules

dist

coverage

.env
.env.*

npm-debug.log

Dockerfile
docker-compose.yaml

README.md
```

---

# 28. Docker Compose

## `docker-compose.yaml`

```yaml
services:

  postgres:

    image: postgres:16

    container_name: promotion-postgres

    restart: unless-stopped

    environment:

      POSTGRES_DB: promotions

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
          "pg_isready -U postgres -d promotions"
        ]

      interval: 10s

      timeout: 5s

      retries: 5


  redis:

    image: redis:7

    container_name: promotion-redis

    restart: unless-stopped

    ports:

      - "6379:6379"


  promotion-service:

    build:

      context: .

      dockerfile: Dockerfile

    container_name: promotion-service

    depends_on:

      postgres:
        condition: service_healthy

      redis:
        condition: service_started

    environment:

      APP_NAME: promotion-service

      NODE_ENV: local

      PORT: 8080

      DATABASE_HOST: postgres

      DATABASE_PORT: 5432

      DATABASE_NAME: promotions

      DATABASE_USER: postgres

      DATABASE_PASSWORD: postgres

      REDIS_HOST: redis

      REDIS_PORT: 6379

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

---

# 29. Local Startup

```bash
npm install
```

Run infrastructure:

```bash
docker compose up -d postgres redis
```

Run application:

```bash
npm run start:dev
```

Build:

```bash
npm run build
```

Production mode:

```bash
npm run start:prod
```

---

# 30. API Testing

Get promotion:

```bash
curl \
  http://localhost:8080/api/v1/promotions/PROMO10
```

Apply promotion:

```bash
curl \
  -X POST \
  http://localhost:8080/api/v1/promotions/apply \
  -H "Content-Type: application/json" \
  -d '{
    "code": "PROMO10",
    "customerId": "C1001",
    "cartValue": 10000
  }'
```

Example:

```json
{
  "promotionCode": "PROMO10",
  "discount": 1000,
  "finalValue": 9000
}
```

---

# 31. Promotion Usage Concurrency

Suppose:

```text
PROMO10

Maximum usage:
100
```

At 10:00:

```text
currentUsage = 99
```

Two customers arrive simultaneously.

Bad implementation:

```text
SELECT currentUsage

if currentUsage < maxUsage:
    UPDATE currentUsage
```

Both requests can read:

```text
99
```

Both believe they can redeem.

Result:

```text
101 usages
```

This violates the promotion limit.

---

# 32. Atomic Usage Update

Use a conditional update:

```sql
UPDATE promotions
SET current_usage = current_usage + 1
WHERE id = $1
  AND current_usage < max_usage;
```

Then check:

```text
Rows affected = 1
```

Success.

```text
Rows affected = 0
```

Promotion exhausted.

This is the same concurrency principle used by the Inventory Service.

---

# 33. Customer-Specific Usage

Suppose:

```text
PROMO10

Maximum per customer:
1
```

Customer:

```text
C1001
```

must not be able to redeem:

```text
Order 1 → SUCCESS
Order 2 → SUCCESS
Order 3 → SUCCESS
```

The database must enforce:

```text
maximum usage per customer
```

rather than trusting application memory.

---

# 34. Promotion Usage Flow

```text
Order Service
      |
      ↓
promotion-service
      |
      ↓
Validate promotion
      |
      ↓
Validate customer
      |
      ↓
Validate cart value
      |
      ↓
Atomically reserve usage
      |
      ↓
Calculate discount
      |
      ↓
Return promotion result
```

Depending on business semantics, usage may be finalized only after order
confirmation rather than at the initial quote.

This distinction is important.

---

# 35. Quote vs Redemption

A pricing request can be:

```text
QUOTE
```

rather than:

```text
REDEEM
```

For example:

```text
Customer opens cart
```

should not necessarily consume a coupon.

Better lifecycle:

```text
Quote
  ↓
Promotion validation
  ↓
Checkout
  ↓
Order creation
  ↓
Promotion redemption
  ↓
Order confirmation
```

This prevents customers from consuming promotion quotas simply by refreshing
the cart.

---

# 36. Promotion State

```text
DRAFT
  |
  ↓
SCHEDULED
  |
  ↓
ACTIVE
  |
  ├── EXPIRED
  |
  └── DISABLED
```

Example:

```text
START:
2026-10-10 00:00

END:
2026-10-20 23:59
```

The application must evaluate effective timestamps consistently using UTC.

---

# 37. Redis Failure

Promotion service should degrade carefully.

Normal:

```text
Promotion
   ↓
Redis HIT
   ↓
Return
```

Redis failure:

```text
Promotion
   ↓
Redis failure
   ↓
PostgreSQL
```

But if traffic is high:

```text
Redis failure
   ↓
Millions of DB reads
   ↓
RDS pressure
   ↓
API latency
```

Therefore use:

- Short Redis timeouts
- Bounded retries
- Circuit breakers
- Request coalescing
- PostgreSQL protection
- Cache warming

Do not simply increase HPA indefinitely.

---

# 38. Cache Stampede

Example:

```text
10,000 requests
```

all arrive after:

```text
promotion:PROMO10
```

expires.

Without protection:

```text
10,000 Redis MISS
        ↓
10,000 PostgreSQL queries
```

Protection:

```text
First request
    ↓
DB
    ↓
Populate cache

Other requests
    ↓
Wait/use populated value
```

---

# 39. Cache Invalidation

When an administrator changes:

```text
PROMO10
10% → 20%
```

the service must invalidate:

```text
promotion:PROMO10
```

Flow:

```text
PostgreSQL update
      |
      ↓
Cache invalidation
      |
      ↓
PromotionChanged event
```

For distributed environments, event-driven invalidation provides better
consistency than relying only on local memory.

---

# 40. Kubernetes Deployment

## `templates/deployment.yaml`

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: promotion-service

  labels:

    app: promotion-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: promotion-service

  template:

    metadata:

      labels:

        app: promotion-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: promotion-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: IfNotPresent

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: promotion-service-config

            - secretRef:

                name: promotion-service-secrets

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

              path: /ready

              port: http

            initialDelaySeconds: 5

            periodSeconds: 10

          livenessProbe:

            httpGet:

              path: /health

              port: http

            initialDelaySeconds: 20

            periodSeconds: 20

          securityContext:

            allowPrivilegeEscalation: false

            readOnlyRootFilesystem: true

            capabilities:

              drop:

                - ALL
```

---

# 41. Kubernetes Service

## `templates/service.yaml`

```yaml
apiVersion: v1

kind: Service

metadata:

  name: promotion-service

  labels:

    app: promotion-service

spec:

  type: ClusterIP

  selector:

    app: promotion-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

Internal DNS:

```text
promotion-service.ecommerce.svc.cluster.local
```

---

# 42. ConfigMap

## `templates/configmap.yaml`

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: promotion-service-config

data:

  NODE_ENV: prod

  PORT: "8080"

  REDIS_HOST: redis.ecommerce.svc.cluster.local

  REDIS_PORT: "6379"

  PROMOTION_CACHE_TTL_SECONDS: "60"
```

For the production AWS architecture, replace the Redis service address with the
actual ElastiCache/Valkey endpoint.

---

# 43. ExternalSecret

## `templates/externalsecret.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: promotion-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: ecommerce-secrets

    kind: ClusterSecretStore

  target:

    name: promotion-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_PASSWORD

      remoteRef:

        key: ecommerce/prod/promotion-service

        property: DATABASE_PASSWORD
```

Database host, username and database name can remain configuration if they are
not sensitive; credentials should be handled as secrets.

---

# 44. ServiceAccount

## `templates/serviceaccount.yaml`

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: promotion-service

  labels:

    app: promotion-service
```

If AWS API access becomes necessary:

```text
Pod
 ↓
EKS Pod Identity
 ↓
IAM Role
 ↓
Specific AWS permission
```

---

# 45. NetworkPolicy

## `templates/networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: promotion-service

spec:

  podSelector:

    matchLabels:

      app: promotion-service

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

Production policy should be aligned with the actual RDS/ElastiCache networking
model.

---

# 46. HPA

## `templates/hpa.yaml`

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: promotion-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: promotion-service

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

Promotion is typically suitable for HPA because much of the workload is
stateless request processing.

However, Redis and PostgreSQL capacity still limit useful scaling.

---

# 47. PDB

## `templates/pdb.yaml`

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: promotion-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: promotion-service
```

---

# 48. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: promotion-service

description: Ecommerce promotion and coupon service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 49. Helm Values

## `values.yaml`

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/promotion-service

  tag: ""

  digest: ""

  pullPolicy: IfNotPresent


service:

  type: ClusterIP

  port: 8080


resources:

  requests:

    cpu: 250m

    memory: 256Mi

  limits:

    cpu: "1"

    memory: 512Mi


autoscaling:

  enabled: true

  minReplicas: 3

  maxReplicas: 30

  targetCPUUtilizationPercentage: 60


serviceAccount:

  create: true

  name: promotion-service


podDisruptionBudget:

  enabled: true

  minAvailable: 2
```

---

# 50. Production Values

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

These values are starting points for the project simulation.

Final values must come from load testing.

---

# 51. Prometheus ServiceMonitor

## `templates/servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: promotion-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: promotion-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

The NestJS service should expose Prometheus-compatible metrics through the
application metrics implementation used by the project.

---

# 52. Important Metrics

```text
promotion_requests_total

promotion_request_duration_seconds

promotion_cache_hits_total

promotion_cache_misses_total

promotion_cache_errors_total

promotion_application_success_total

promotion_application_failure_total

promotion_usage_exhausted_total

promotion_invalid_total

promotion_rule_evaluation_total
```

Business metrics:

```text
coupon_redemption_count

promotion_discount_value

promotion_expired_count

promotion_usage_limit_reached
```

Avoid uncontrolled labels such as:

```text
customerId
orderId
```

because they can create massive Prometheus cardinality.

---

# 53. Grafana Dashboard

```text
Promotion Service
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
│   └── Errors
│
├── PostgreSQL
│   ├── Connections
│   ├── Query Latency
│   └── DB Load
│
├── Promotion
│   ├── Applied
│   ├── Rejected
│   ├── Expired
│   └── Usage Exhausted
│
└── Kubernetes
    ├── Pods
    ├── CPU
    ├── Memory
    └── Restarts
```

---

# 54. Structured Logging

Example:

```json
{
  "@timestamp": "2026-10-05T16:00:00Z",
  "level": "INFO",
  "service": "promotion-service",
  "environment": "prod",
  "namespace": "ecommerce",
  "pod": "promotion-service-7d8c9f",
  "version": "1.0.0",
  "correlationId": "corr-123",
  "promotionCode": "PROMO10",
  "customerTier": "GOLD",
  "cacheHit": true,
  "message": "Promotion evaluated"
}
```

Never log:

```text
password
authorization token
payment credentials
secret values
unnecessary customer PII
```

---

# 55. OpenTelemetry

Request flow:

```text
order-service
      |
      ↓
promotion-service
      |
      ├── Redis
      |
      └── PostgreSQL
```

Tracing:

```text
Trace
 |
 ├── Order request
 |
 ├── Promotion request
 |
 ├── Redis GET
 |
 └── PostgreSQL query
```

Architecture:

```text
promotion-service
       ↓
OpenTelemetry
       ↓
OTel Collector
       ↓
Jaeger
```

This lets engineers identify whether a slow promotion calculation comes from:

- Application logic
- Redis
- PostgreSQL
- Network
- Rule evaluation

---

# 56. SLO

Example:

## Availability

```text
99.95%
```

## Promotion API P95

```text
< 150ms
```

## Promotion API P99

```text
< 300ms
```

## Correctness

Promotion usage limits:

```text
No unintended over-redemption
```

Business correctness is more important than simply achieving low CPU usage.

---

# 57. Failure Scenario — Redis Failure

Normal:

```text
Request
  ↓
Redis
  ↓
HIT
```

Failure:

```text
Request
  ↓
Redis timeout
  ↓
PostgreSQL
```

If traffic increases:

```text
Redis failure
    ↓
Cache miss
    ↓
PostgreSQL overload
    ↓
Connection pool pressure
    ↓
Promotion latency
    ↓
Order latency
```

Mitigation:

```text
Redis timeout reduction
+
Fallback protection
+
Connection pool protection
+
Circuit breaker
+
Redis recovery
```

---

# 58. Failure Scenario — Promotion Exhausted

Suppose:

```text
PROMO10
maxUsage = 100
```

Current:

```text
99
```

Two simultaneous redemption attempts arrive.

Atomic database update:

```sql
UPDATE promotions
SET current_usage = current_usage + 1
WHERE id = $1
  AND current_usage < max_usage;
```

Expected:

```text
Request A → SUCCESS
Request B → EXHAUSTED
```

This prevents over-redemption.

---

# 59. Failure Scenario — Cache Contains Expired Promotion

PostgreSQL:

```text
status = EXPIRED
```

Redis:

```text
status = ACTIVE
```

Possible causes:

- Cache invalidation failure
- Excessive TTL
- Promotion update failed to invalidate cache
- Incorrect cache key

Mitigation:

```text
Validate promotion effective dates
+
Short TTL
+
Event-based invalidation
+
Cache consistency monitoring
```

---

# 60. Failure Scenario — PostgreSQL Connection Exhaustion

Example:

```text
Promotion Pods = 30

Pool:
10 connections

Potential:
300 connections
```

If PostgreSQL safe capacity is lower:

```text
HPA
 ↓
More Pods
 ↓
More connections
 ↓
RDS exhaustion
```

Therefore:

```text
HPA maxReplicas
+
Node capacity
+
Redis capacity
+
RDS connection budget
```

must be considered together.

---

# 61. Failure Scenario — Incorrect Discount

Example:

Expected:

```text
Gold customer
₹10,000
10% discount
₹9,000
```

Actual:

```text
₹8,000
```

Possible causes:

- Multiple promotions incorrectly applied
- Wrong rule priority
- Incorrect customer tier
- Currency issue
- Duplicate promotion calculation
- Floating-point rounding
- Stale Redis rule

Investigation:

```text
Request
 ↓
Promotion rule
 ↓
Customer tier
 ↓
Cache
 ↓
Database
 ↓
Calculation
 ↓
Order snapshot
```

---

# 62. Promotion + Pricing Integration

Example:

```text
Product
₹10,000
     |
     ↓
Pricing Service
     |
     ↓
Base price = ₹10,000
     |
     ↓
Promotion Service
     |
     ↓
10% discount
     |
     ↓
₹9,000
```

Then:

```text
Tax Service
     |
     ↓
Final tax calculation
```

---

# 63. Do Not Double-Discount

A dangerous architecture:

```text
pricing-service
    ↓
already applies promotion

promotion-service
    ↓
applies promotion again
```

Result:

```text
₹10,000
 ↓
10%
₹9,000
 ↓
another 10%
₹8,100
```

Ownership must be clear.

Recommended:

```text
Pricing Service
    ↓
Base price

Promotion Service
    ↓
Promotion discount

Order Service
    ↓
Combines the results
```

---

# 64. CI Pipeline

## `.github/workflows/ci.yaml`

```yaml
name: Promotion Service CI

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

  ECR_REPOSITORY: promotion-service


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


      - name: Format check

        run: npm run format:check


      - name: Unit tests

        run: npm test -- --runInBand


      - name: Coverage

        run: npm run test:cov


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

# 65. SonarQube

Pipeline:

```text
Checkout
   ↓
npm ci
   ↓
Lint
   ↓
Tests
   ↓
Coverage
   ↓
SonarQube
   ↓
Dependency Scan
   ↓
Trivy
   ↓
Docker
   ↓
ECR
```

SonarQube remains a centralized service.

The scanner runs on the CI runner.

---

# 66. Trivy

Source scan:

```text
Repository
    ↓
Trivy filesystem scan
```

Container scan:

```text
Docker image
    ↓
Trivy image scan
```

ECR can additionally scan pushed images.

---

# 67. ECR

Repository:

```text
company-ecommerce/promotion-service
```

Image:

```text
123456789012.dkr.ecr.ap-south-1.amazonaws.com/promotion-service:<git-sha>
```

Prefer:

```text
Git SHA
```

or:

```text
Image digest
```

instead of:

```text
latest
```

---

# 68. GitOps

Application repository:

```text
company-ecommerce/promotion-service
```

GitOps repository:

```text
company-ecommerce/gitops
```

Structure:

```text
gitops/
└── apps/
    └── promotion-service/
        ├── dev/
        ├── qa/
        ├── staging/
        └── prod/
```

Promotion flow:

```text
Developer
   ↓
Pull Request
   ↓
CI
   ↓
Security
   ↓
Docker
   ↓
ECR
   ↓
GitOps
   ↓
ArgoCD
   ↓
EKS
```

---

# 69. ArgoCD

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: promotion-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/promotion-service/prod

  destination:

    server: https://kubernetes.default.svc

    namespace: ecommerce

  syncPolicy:

    automated:

      prune: false

      selfHeal: true
```

---

# 70. Production Incident

## Incident ID

```text
INC-2026-PROMOTION-001
```

## Severity

```text
SEV-2
```

## Title

```text
Promotion Redis Failure Caused PostgreSQL Load Spike
```

---

# 71. Customer Impact

Customers experienced:

- Increased checkout latency
- Promotion validation delays
- Some promotion requests returning 5xx

No incorrect promotion redemption occurred.

---

# 72. Timeline

```text
14:00
Promotion service healthy.

14:05
Redis latency begins increasing.

14:06
Redis errors increase.

14:07
Promotion cache hit ratio drops.

14:08
Promotion service falls back to PostgreSQL.

14:09
PostgreSQL connections increase rapidly.

14:10
Database query latency increases.

14:11
Checkout latency increases.

14:12
Promotion 5xx rate reaches 4%.

14:13
SEV-2 declared.

14:15
Platform team confirms Redis connectivity issue.

14:17
Application team reduces database fallback concurrency.

14:19
Redis recovery begins.

14:22
Redis connectivity restored.

14:24
Cache hit ratio begins recovering.

14:26
PostgreSQL connections decrease.

14:28
Checkout latency returns to normal.

14:30
Incident resolved.
```

---

# 73. Root Cause

Redis experienced a temporary availability/connectivity problem.

The promotion service correctly attempted PostgreSQL fallback.

However, fallback concurrency was not sufficiently limited.

Therefore:

```text
Redis failure
    ↓
Cache misses
    ↓
PostgreSQL reads
    ↓
RDS connection pressure
    ↓
Checkout latency
```

---

# 74. Amplification

The direct failure was:

```text
Redis
```

The incident became severe because of:

```text
unbounded fallback traffic
```

This is an important production lesson:

> A fallback mechanism can become the second failure.

---

# 75. Five Whys

### Why did checkout become slow?

Promotion requests became slow.

### Why?

PostgreSQL was handling significantly more promotion reads.

### Why?

Redis cache requests were failing.

### Why?

Redis experienced connectivity problems.

### Why did PostgreSQL become overloaded?

Fallback traffic did not have sufficient concurrency protection.

---

# 76. Corrective Actions

| Action | Owner |
|---|---|
| Add Redis failure circuit breaker | Backend |
| Add fallback concurrency limit | Backend |
| Add cache stampede protection | Backend |
| Add Redis alerts | SRE |
| Add PostgreSQL fallback dashboard | SRE |
| Add load test for Redis failure | Performance |
| Review HPA max replicas | Platform |
| Add dependency failure testing | Platform |
| Add pricing/promotion degradation runbook | SRE |

---

# 77. Security Checklist

- [ ] No secrets in Git
- [ ] Secrets Manager
- [ ] External Secrets
- [ ] EKS Pod Identity
- [ ] Least privilege
- [ ] NetworkPolicy
- [ ] Private RDS
- [ ] Private Redis
- [ ] TLS where appropriate
- [ ] Non-root container
- [ ] Read-only filesystem
- [ ] Trivy
- [ ] SonarQube
- [ ] Dependency scanning
- [ ] Secret scanning
- [ ] No credentials in logs

---

# 78. Production Checklist

## Application

- [ ] Node.js
- [ ] TypeScript
- [ ] NestJS
- [ ] DTO validation
- [ ] Error handling
- [ ] Unit tests
- [ ] Integration tests
- [ ] Decimal-safe money handling

## Promotion

- [ ] Promotion lifecycle
- [ ] Coupon validation
- [ ] Effective dates
- [ ] Usage limits
- [ ] Customer usage limits
- [ ] Atomic redemption
- [ ] Idempotency
- [ ] Promotion priority
- [ ] Cache invalidation

## Database

- [ ] PostgreSQL
- [ ] RDS
- [ ] Indexes
- [ ] Connection pool
- [ ] Backups
- [ ] PITR
- [ ] Private networking

## Redis

- [ ] Cache
- [ ] TTL
- [ ] Invalidation
- [ ] Stampede protection
- [ ] Failure fallback
- [ ] Hit ratio monitoring

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
- [ ] Probes

## CI/CD

- [ ] npm ci
- [ ] TypeScript build
- [ ] Unit tests
- [ ] Coverage
- [ ] Lint
- [ ] SonarQube
- [ ] Dependency scanning
- [ ] Secret scanning
- [ ] Trivy
- [ ] Docker
- [ ] ECR
- [ ] GitHub OIDC

## GitOps

- [ ] Helm
- [ ] GitOps
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

# 79. Final Architecture

```text
                         Internet
                            |
                     CloudFront / WAF
                            |
                           ALB
                            |
                            ↓
                   promotion-service
                      /          \
                     /            \
                    ↓              ↓
                 Redis         PostgreSQL
                    |               |
                    |               |
                    └──────┬────────┘
                           |
                     Promotion Rules
                           |
                           ↓
                     Order Service


Observability:

Promotion Service
      |
      ├── Prometheus → Grafana
      |
      ├── stdout → Fluent Bit → Logstash → Elasticsearch → Kibana
      |
      └── OpenTelemetry → Collector → Jaeger
```

---

# 80. Final DevOps Perspective

Promotion demonstrates another important production pattern.

The service is:

```text
stateless application
+
Redis cache
+
PostgreSQL source of truth
```

Therefore Kubernetes can scale the application horizontally:

```text
                 HPA
                  |
        ┌─────────┼─────────┐
        ↓         ↓         ↓
      Pod 1     Pod 2     Pod 3
        |         |         |
        └─────────┼─────────┘
                  |
          Redis / PostgreSQL
```

But scaling Pods does not automatically solve:

```text
Redis capacity
PostgreSQL capacity
promotion-rule contention
database connections
cache stampede
```

The production design must scale the complete dependency chain.

# End of 168 — Promotion Service
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

169 Cart Service            ⏳
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

**Next: `169-Cart-Service.md` — Java + Spring Boot + Maven, Redis-first cart architecture, PostgreSQL persistence, cart expiration, concurrency, CI/CD, Kubernetes, Helm, GitOps, observability, and production RCA.**