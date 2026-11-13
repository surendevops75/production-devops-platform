# 163 — Customer Service

**File:** `163-Customer-Service.md`

# 163 — Customer Service

## 1. Overview

The `customer-service` manages customer-level business information for the
Enterprise E-Commerce Platform.

The service sits above the basic identity/profile layer.

The separation is:

    auth-service
        |
        +-- Authentication
        +-- Credentials
        +-- Tokens

    user-service
        |
        +-- Basic user profile
        +-- Name
        +-- Email
        +-- Phone

    customer-service
        |
        +-- Customer account
        +-- Customer status
        +-- Addresses
        +-- Preferences
        +-- Customer segmentation
        +-- Loyalty information
        +-- Customer business metadata

The service is implemented using:

- TypeScript
- Node.js
- NestJS
- Prisma
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

# 2. Why Customer Service Is Separate

We don't want one large user table to become responsible for every
customer-related business capability.

The ownership model is:

    auth-service
        |
        | userId
        ↓
    user-service
        |
        | userId
        ↓
    customer-service
        |
        +-- Customer account
        +-- Addresses
        +-- Preferences
        +-- Loyalty
        +-- Segmentation

For example:

Authentication:

    "Can this person log in?"

User profile:

    "What is the user's name and email?"

Customer service:

    "What customer account, addresses, preferences and business
     attributes belong to this user?"

---

# 3. Business Responsibilities

Customer service owns:

- Customer account
- Customer status
- Customer addresses
- Default address
- Customer preferences
- Loyalty information
- Customer segmentation
- Customer metadata
- Customer lifecycle events

It does not own:

- Passwords
- Authentication
- Access tokens
- Orders
- Payments
- Inventory
- Product catalog

---

# 4. Technology Stack

| Layer | Technology |
|---|---|
| Language | TypeScript |
| Runtime | Node.js |
| Framework | NestJS |
| ORM | Prisma |
| Database | PostgreSQL |
| Cache | Redis |
| Messaging | Kafka |
| Package manager | pnpm |
| Testing | Jest |
| Linting | ESLint |
| Formatting | Prettier |
| Container | Docker |
| Orchestration | Kubernetes/EKS |
| Packaging | Helm |
| Registry | Amazon ECR |
| CI | GitHub Actions |
| GitOps | ArgoCD |
| Metrics | Prometheus |
| Logs | ELK |
| Tracing | OpenTelemetry |
| Trace backend | Jaeger |

---

# 5. Architecture

    Client
       |
       ↓
    CloudFront / WAF
       |
       ↓
    ALB
       |
       ↓
    customer-service
       |
       ├──────────────┐
       ↓              ↓
    PostgreSQL      Redis
       |
       ↓
      Kafka

The service is stateless.

Customer data is stored in PostgreSQL.

Redis is used for read-heavy customer information and short-lived
business state.

Kafka is used for asynchronous customer events.

---

# 6. Example Customer Flow

A customer requests their profile:

    Client
      |
      ↓
    ALB
      |
      ↓
    customer-service
      |
      ↓
    Redis
      |
      ├── HIT → return customer
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
        response

For an update:

    Client
      |
      ↓
    customer-service
      |
      ↓
    PostgreSQL
      |
      ↓
    invalidate Redis
      |
      ↓
    publish CustomerUpdated event

---

# 7. APIs

## Get Customer

    GET /api/v1/customers/{customerId}

## Get Current Customer

    GET /api/v1/customers/me

## Update Customer

    PUT /api/v1/customers/{customerId}

## Get Addresses

    GET /api/v1/customers/{customerId}/addresses

## Add Address

    POST /api/v1/customers/{customerId}/addresses

## Update Address

    PUT /api/v1/customers/{customerId}/addresses/{addressId}

## Delete Address

    DELETE /api/v1/customers/{customerId}/addresses/{addressId}

## Health

    GET /health

## Readiness

    GET /ready

## Metrics

    GET /metrics

---

# 8. Repository Structure

    customer-service/
    │
    ├── src/
    │   ├── main.ts
    │   │
    │   ├── app.module.ts
    │   │
    │   ├── customer/
    │   │   ├── customer.controller.ts
    │   │   ├── customer.service.ts
    │   │   ├── customer.module.ts
    │   │   └── dto/
    │   │       ├── update-customer.dto.ts
    │   │       └── customer-response.dto.ts
    │   │
    │   ├── address/
    │   │   ├── address.controller.ts
    │   │   ├── address.service.ts
    │   │   ├── address.module.ts
    │   │   └── dto/
    │   │       ├── create-address.dto.ts
    │   │       └── update-address.dto.ts
    │   │
    │   ├── prisma/
    │   │   ├── prisma.service.ts
    │   │   └── prisma.module.ts
    │   │
    │   ├── redis/
    │   │   ├── redis.service.ts
    │   │   └── redis.module.ts
    │   │
    │   ├── kafka/
    │   │   ├── kafka.service.ts
    │   │   └── kafka.module.ts
    │   │
    │   ├── config/
    │   │   └── configuration.ts
    │   │
    │   └── observability/
    │       └── metrics.ts
    │
    ├── prisma/
    │   └── schema.prisma
    │
    ├── test/
    │   ├── customer.service.spec.ts
    │   └── customer.controller.spec.ts
    │
    ├── package.json
    ├── pnpm-lock.yaml
    ├── tsconfig.json
    ├── tsconfig.build.json
    ├── nest-cli.json
    │
    ├── Dockerfile
    ├── .dockerignore
    ├── docker-compose.yaml
    ├── .env.example
    ├── README.md
    │
    ├── helm/
    │   └── customer-service/
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
    └── .github/
        ├── CODEOWNERS
        └── workflows/
            ├── ci.yaml
            └── security.yaml

---

# 9. Node.js and TypeScript

The application stack is:

    TypeScript
        ↓
    Node.js runtime
        ↓
    NestJS
        ↓
    Prisma
        ↓
    PostgreSQL

This is different from our previous services.

Auth:

    Java → Spring Boot → Maven

User:

    Python → FastAPI → uv

Customer:

    TypeScript → Node.js → NestJS → pnpm

The Kubernetes platform remains standardized.

---

# 10. Package Configuration

## `package.json`

```json
{
  "name": "customer-service",
  "version": "1.0.0",
  "description": "Customer service for ecommerce platform",
  "private": true,
  "scripts": {
    "build": "nest build",
    "start": "node dist/main.js",
    "start:dev": "nest start --watch",
    "start:prod": "node dist/main.js",
    "lint": "eslint \"src/**/*.ts\" \"test/**/*.ts\"",
    "format": "prettier --write \"src/**/*.ts\" \"test/**/*.ts\"",
    "format:check": "prettier --check \"src/**/*.ts\" \"test/**/*.ts\"",
    "test": "jest",
    "test:coverage": "jest --coverage",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev",
    "prisma:deploy": "prisma migrate deploy"
  },
  "dependencies": {
    "@nestjs/common": "^11.0.0",
    "@nestjs/core": "^11.0.0",
    "@nestjs/config": "^4.0.0",
    "@nestjs/platform-express": "^11.0.0",
    "@nestjs/microservices": "^11.0.0",
    "@prisma/client": "^6.0.0",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.1",
    "ioredis": "^5.4.1",
    "kafkajs": "^2.2.4",
    "prom-client": "^15.1.0",
    "reflect-metadata": "^0.2.2",
    "rxjs": "^7.8.1"
  },
  "devDependencies": {
    "@nestjs/cli": "^11.0.0",
    "@nestjs/schematics": "^11.0.0",
    "@nestjs/testing": "^11.0.0",
    "@types/jest": "^29.5.14",
    "@types/node": "^22.0.0",
    "@typescript-eslint/eslint-plugin": "^8.0.0",
    "@typescript-eslint/parser": "^8.0.0",
    "eslint": "^9.0.0",
    "jest": "^29.7.0",
    "prettier": "^3.5.0",
    "prisma": "^6.0.0",
    "ts-jest": "^29.2.5",
    "ts-node": "^10.9.2",
    "typescript": "^5.7.0"
  }
}
```

The lockfile is committed:

    pnpm-lock.yaml

Production dependency updates should be performed deliberately and tested
through CI.

---

# 11. TypeScript Configuration

## `tsconfig.json`

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "declaration": true,
    "removeComments": true,
    "emitDecoratorMetadata": true,
    "experimentalDecorators": true,
    "allowSyntheticDefaultImports": true,
    "target": "ES2022",
    "sourceMap": true,
    "outDir": "./dist",
    "baseUrl": "./",
    "incremental": true,
    "strict": true,
    "skipLibCheck": true,
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "noImplicitAny": true
  }
}
```

---

# 12. Prisma Database Schema

## `prisma/schema.prisma`

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Customer {
  id              BigInt      @id @default(autoincrement())
  customerId      String      @unique @default(uuid()) @db.Uuid
  userId          String      @unique @db.Uuid
  status          CustomerStatus @default(ACTIVE)
  loyaltyTier     String      @default("STANDARD")
  marketingOptIn  Boolean     @default(false)
  createdAt       DateTime    @default(now())
  updatedAt       DateTime    @updatedAt

  addresses       Address[]

  @@index([userId])
  @@index([status])
}

model Address {
  id          BigInt    @id @default(autoincrement())
  addressId   String    @unique @default(uuid()) @db.Uuid
  customerId  BigInt
  type        AddressType
  line1       String
  line2       String?
  city        String
  state       String
  postalCode  String
  country     String
  isDefault   Boolean   @default(false)
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  customer    Customer  @relation(
    fields: [customerId],
    references: [id],
    onDelete: Cascade
  )

  @@index([customerId])
}

enum CustomerStatus {
  ACTIVE
  INACTIVE
  BLOCKED
}

enum AddressType {
  HOME
  WORK
  BILLING
  SHIPPING
  OTHER
}
```

The customer service owns these tables.

---

# 13. Prisma Service

## `src/prisma/prisma.service.ts`

```typescript
import {
  Injectable,
  OnModuleDestroy,
  OnModuleInit,
} from '@nestjs/common';

import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  async onModuleInit(): Promise<void> {
    await this.$connect();
  }

  async onModuleDestroy(): Promise<void> {
    await this.$disconnect();
  }
}
```

---

# 14. Prisma Module

## `src/prisma/prisma.module.ts`

```typescript
import { Global, Module } from '@nestjs/common';

import { PrismaService } from './prisma.service';

@Global()
@Module({
  providers: [PrismaService],
  exports: [PrismaService],
})
export class PrismaModule {}
```

---

# 15. Configuration

## `src/config/configuration.ts`

```typescript
export default () => ({
  app: {
    name: process.env.APP_NAME ?? 'customer-service',
    environment: process.env.NODE_ENV ?? 'local',
    port: Number(process.env.PORT ?? 8080),
  },

  database: {
    url: process.env.DATABASE_URL,
  },

  redis: {
    host: process.env.REDIS_HOST ?? 'localhost',
    port: Number(process.env.REDIS_PORT ?? 6379),
  },

  kafka: {
    brokers: (
      process.env.KAFKA_BOOTSTRAP_SERVERS ??
      'localhost:9092'
    ).split(','),
  },
});
```

---

# 16. Main Application

## `src/main.ts`

```typescript
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';

import { AppModule } from './app.module';

async function bootstrap(): Promise<void> {
  const app = await NestFactory.create(AppModule);

  app.setGlobalPrefix('api/v1');

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
    }),
  );

  app.enableShutdownHooks();

  await app.listen(
    Number(process.env.PORT ?? 8080),
    '0.0.0.0',
  );
}

bootstrap();
```

---

# 17. Application Module

## `src/app.module.ts`

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';

import configuration from './config/configuration';
import { PrismaModule } from './prisma/prisma.module';
import { CustomerModule } from './customer/customer.module';
import { AddressModule } from './address/address.module';
import { RedisModule } from './redis/redis.module';
import { KafkaModule } from './kafka/kafka.module';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      load: [configuration],
    }),

    PrismaModule,

    RedisModule,

    KafkaModule,

    CustomerModule,

    AddressModule,
  ],
})
export class AppModule {}
```

---

# 18. Customer DTO

## `src/customer/dto/update-customer.dto.ts`

```typescript
import {
  IsBoolean,
  IsOptional,
  IsString,
  MaxLength,
} from 'class-validator';

export class UpdateCustomerDto {
  @IsOptional()
  @IsString()
  @MaxLength(30)
  loyaltyTier?: string;

  @IsOptional()
  @IsBoolean()
  marketingOptIn?: boolean;
}
```

---

# 19. Customer Service

## `src/customer/customer.service.ts`

```typescript
import {
  Injectable,
  NotFoundException,
} from '@nestjs/common';

import { PrismaService } from '../prisma/prisma.service';
import { RedisService } from '../redis/redis.service';
import { KafkaService } from '../kafka/kafka.service';

import { UpdateCustomerDto } from './dto/update-customer.dto';

@Injectable()
export class CustomerService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly redis: RedisService,
    private readonly kafka: KafkaService,
  ) {}

  async findByCustomerId(customerId: string) {
    const cacheKey = `customer:${customerId}`;

    const cached = await this.redis.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    const customer = await this.prisma.customer.findUnique({
      where: {
        customerId,
      },
      include: {
        addresses: true,
      },
    });

    if (!customer) {
      throw new NotFoundException(
        'Customer not found',
      );
    }

    const response = {
      customerId: customer.customerId,
      userId: customer.userId,
      status: customer.status,
      loyaltyTier: customer.loyaltyTier,
      marketingOptIn: customer.marketingOptIn,
      addresses: customer.addresses,
    };

    await this.redis.set(
      cacheKey,
      JSON.stringify(response),
      300,
    );

    return response;
  }

  async update(
    customerId: string,
    dto: UpdateCustomerDto,
  ) {
    const customer =
      await this.prisma.customer.update({
        where: {
          customerId,
        },
        data: {
          loyaltyTier: dto.loyaltyTier,
          marketingOptIn: dto.marketingOptIn,
        },
      });

    await this.redis.delete(
      `customer:${customerId}`,
    );

    await this.kafka.publish(
      'customer-events',
      {
        eventType: 'CustomerUpdated',
        version: 1,
        payload: {
          customerId,
          userId: customer.userId,
        },
      },
    );

    return customer;
  }
}
```

The cache is invalidated after a successful database update.

---

# 20. Customer Controller

## `src/customer/customer.controller.ts`

```typescript
import {
  Body,
  Controller,
  Get,
  Param,
  Put,
} from '@nestjs/common';

import { CustomerService } from './customer.service';
import { UpdateCustomerDto } from './dto/update-customer.dto';

@Controller('customers')
export class CustomerController {
  constructor(
    private readonly customerService: CustomerService,
  ) {}

  @Get(':customerId')
  async getCustomer(
    @Param('customerId') customerId: string,
  ) {
    return this.customerService.findByCustomerId(
      customerId,
    );
  }

  @Put(':customerId')
  async updateCustomer(
    @Param('customerId') customerId: string,

    @Body() dto: UpdateCustomerDto,
  ) {
    return this.customerService.update(
      customerId,
      dto,
    );
  }
}
```

---

# 21. Customer Module

## `src/customer/customer.module.ts`

```typescript
import { Module } from '@nestjs/common';

import { CustomerController } from './customer.controller';
import { CustomerService } from './customer.service';

@Module({
  controllers: [CustomerController],

  providers: [CustomerService],
})
export class CustomerModule {}
```

---

# 22. Address DTO

## `src/address/dto/create-address.dto.ts`

```typescript
import {
  IsBoolean,
  IsEnum,
  IsOptional,
  IsString,
  MaxLength,
} from 'class-validator';

export enum AddressType {
  HOME = 'HOME',
  WORK = 'WORK',
  BILLING = 'BILLING',
  SHIPPING = 'SHIPPING',
  OTHER = 'OTHER',
}

export class CreateAddressDto {
  @IsEnum(AddressType)
  type: AddressType;

  @IsString()
  @MaxLength(255)
  line1: string;

  @IsOptional()
  @IsString()
  @MaxLength(255)
  line2?: string;

  @IsString()
  @MaxLength(100)
  city: string;

  @IsString()
  @MaxLength(100)
  state: string;

  @IsString()
  @MaxLength(30)
  postalCode: string;

  @IsString()
  @MaxLength(100)
  country: string;

  @IsOptional()
  @IsBoolean()
  isDefault?: boolean;
}
```

---

# 23. Address Service

## `src/address/address.service.ts`

```typescript
import {
  Injectable,
  NotFoundException,
} from '@nestjs/common';

import { PrismaService } from '../prisma/prisma.service';
import { RedisService } from '../redis/redis.service';

import { CreateAddressDto } from './dto/create-address.dto';

@Injectable()
export class AddressService {
  constructor(
    private readonly prisma: PrismaService,
    private readonly redis: RedisService,
  ) {}

  async create(
    customerId: string,
    dto: CreateAddressDto,
  ) {
    const customer =
      await this.prisma.customer.findUnique({
        where: {
          customerId,
        },
      });

    if (!customer) {
      throw new NotFoundException(
        'Customer not found',
      );
    }

    const address =
      await this.prisma.address.create({
        data: {
          customerId: customer.id,
          type: dto.type,
          line1: dto.line1,
          line2: dto.line2,
          city: dto.city,
          state: dto.state,
          postalCode: dto.postalCode,
          country: dto.country,
          isDefault: dto.isDefault ?? false,
        },
      });

    await this.redis.delete(
      `customer:${customerId}`,
    );

    return address;
  }

  async list(customerId: string) {
    const customer =
      await this.prisma.customer.findUnique({
        where: {
          customerId,
        },
      });

    if (!customer) {
      throw new NotFoundException(
        'Customer not found',
      );
    }

    return this.prisma.address.findMany({
      where: {
        customerId: customer.id,
      },
    });
  }
}
```

---

# 24. Address Controller

## `src/address/address.controller.ts`

```typescript
import {
  Body,
  Controller,
  Get,
  Param,
  Post,
} from '@nestjs/common';

import { AddressService } from './address.service';
import { CreateAddressDto } from './dto/create-address.dto';

@Controller('customers/:customerId/addresses')
export class AddressController {
  constructor(
    private readonly addressService: AddressService,
  ) {}

  @Post()
  async create(
    @Param('customerId') customerId: string,

    @Body() dto: CreateAddressDto,
  ) {
    return this.addressService.create(
      customerId,
      dto,
    );
  }

  @Get()
  async list(
    @Param('customerId') customerId: string,
  ) {
    return this.addressService.list(
      customerId,
    );
  }
}
```

---

# 25. Address Module

## `src/address/address.module.ts`

```typescript
import { Module } from '@nestjs/common';

import { AddressController } from './address.controller';
import { AddressService } from './address.service';

@Module({
  controllers: [AddressController],

  providers: [AddressService],
})
export class AddressModule {}
```

---

# 26. Redis Module

## `src/redis/redis.module.ts`

```typescript
import { Global, Module } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

import Redis from 'ioredis';

import { RedisService } from './redis.service';

@Global()
@Module({
  providers: [
    {
      provide: 'REDIS_CLIENT',

      inject: [ConfigService],

      useFactory: (
        config: ConfigService,
      ) => {
        return new Redis({
          host: config.get<string>(
            'redis.host',
          ),

          port: config.get<number>(
            'redis.port',
          ),

          connectTimeout: 1000,

          maxRetriesPerRequest: 1,
        });
      },
    },

    RedisService,
  ],

  exports: [
    RedisService,
  ],
})
export class RedisModule {}
```

---

# 27. Redis Service

## `src/redis/redis.service.ts`

```typescript
import {
  Inject,
  Injectable,
} from '@nestjs/common';

import Redis from 'ioredis';

@Injectable()
export class RedisService {
  constructor(
    @Inject('REDIS_CLIENT')
    private readonly client: Redis,
  ) {}

  async get(key: string): Promise<string | null> {
    try {
      return await this.client.get(key);
    } catch {
      return null;
    }
  }

  async set(
    key: string,
    value: string,
    ttlSeconds: number,
  ): Promise<void> {
    try {
      await this.client.set(
        key,
        value,
        'EX',
        ttlSeconds,
      );
    } catch {
      // Cache failure must not break the
      // primary database path.
    }
  }

  async delete(key: string): Promise<void> {
    try {
      await this.client.del(key);
    } catch {
      // Cache invalidation failure is monitored.
    }
  }
}
```

The production implementation should emit metrics when Redis errors are
suppressed. A silent `catch` must not hide a systemic dependency failure.

---

# 28. Kafka Module

## `src/kafka/kafka.module.ts`

```typescript
import {
  Global,
  Module,
} from '@nestjs/common';

import { KafkaService } from './kafka.service';

@Global()
@Module({
  providers: [KafkaService],
  exports: [KafkaService],
})
export class KafkaModule {}
```

---

# 29. Kafka Service

## `src/kafka/kafka.service.ts`

```typescript
import {
  Injectable,
  OnModuleDestroy,
  OnModuleInit,
} from '@nestjs/common';

import { Kafka, Producer } from 'kafkajs';

@Injectable()
export class KafkaService
  implements OnModuleInit, OnModuleDestroy
{
  private readonly producer: Producer;

  constructor() {
    const brokers = (
      process.env.KAFKA_BOOTSTRAP_SERVERS ??
      'localhost:9092'
    ).split(',');

    const kafka = new Kafka({
      clientId: 'customer-service',

      brokers,
    });

    this.producer = kafka.producer();
  }

  async onModuleInit(): Promise<void> {
    await this.producer.connect();
  }

  async onModuleDestroy(): Promise<void> {
    await this.producer.disconnect();
  }

  async publish(
    topic: string,
    event: Record<string, unknown>,
  ): Promise<void> {
    await this.producer.send({
      topic,

      messages: [
        {
          value: JSON.stringify({
            eventId: crypto.randomUUID(),

            eventType:
              event.eventType,

            version:
              event.version ?? 1,

            occurredAt:
              new Date().toISOString(),

            source: 'customer-service',

            payload:
              event.payload,
          }),
        },
      ],
    });
  }
}
```

For a production implementation, use the project's standard correlation ID
and transactional-outbox strategy where the business event must not be lost
between the database transaction and Kafka publication.

---

# 30. Health Endpoints

NestJS can expose dedicated health checks.

For the Kubernetes probes, we want:

```text
/health
/ready
```

The key principle is:

- Liveness should answer whether the process is alive.
- Readiness should answer whether the instance can safely receive traffic.

Do not make liveness depend on every downstream service.

---

# 31. Dockerfile

## `Dockerfile`

```dockerfile
FROM node:22-alpine AS builder

WORKDIR /build

RUN corepack enable

COPY package.json pnpm-lock.yaml ./

RUN pnpm install --frozen-lockfile

COPY prisma ./prisma

RUN pnpm prisma generate

COPY tsconfig.json tsconfig.build.json nest-cli.json ./

COPY src ./src

RUN pnpm build


FROM node:22-alpine AS production

WORKDIR /app

ENV NODE_ENV=production

RUN corepack enable

COPY package.json pnpm-lock.yaml ./

RUN pnpm install \
    --frozen-lockfile \
    --prod

COPY --from=builder /build/node_modules/.prisma \
    ./node_modules/.prisma

COPY --from=builder /build/dist \
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

CMD ["node", "dist/main.js"]
```

The production runtime image contains only production dependencies and the
compiled application.

---

# 32. `.dockerignore`

```text
.git
.github
.gitignore

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

# 33. Local Docker Compose

## `docker-compose.yaml`

```yaml
services:

  postgres:

    image: postgres:16

    container_name: customer-postgres

    restart: unless-stopped

    environment:

      POSTGRES_DB: customers

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
          "pg_isready -U postgres -d customers"
        ]

      interval: 10s

      timeout: 5s

      retries: 5


  redis:

    image: redis:7

    container_name: customer-redis

    restart: unless-stopped

    ports:

      - "6379:6379"


  kafka:

    image: apache/kafka:latest

    container_name: customer-kafka

    restart: unless-stopped

    ports:

      - "9092:9092"


  customer-service:

    build:

      context: .

      dockerfile: Dockerfile

    container_name: customer-service

    restart: unless-stopped

    depends_on:

      postgres:
        condition: service_healthy

      redis:
        condition: service_started

      kafka:
        condition: service_started

    environment:

      NODE_ENV: local

      PORT: 8080

      DATABASE_URL: >-
        postgresql://postgres:postgres@postgres:5432/customers

      REDIS_HOST: redis

      REDIS_PORT: 6379

      KAFKA_BOOTSTRAP_SERVERS: kafka:9092

    ports:

      - "8080:8080"


volumes:

  postgres-data:
```

As with earlier services, the floating Kafka tag is only for local
convenience. Production images must be pinned to approved versions.

---

# 34. Local Development

Install dependencies:

```bash
pnpm install
```

Generate Prisma client:

```bash
pnpm prisma generate
```

Run development server:

```bash
pnpm start:dev
```

Build:

```bash
pnpm build
```

Run production:

```bash
pnpm start:prod
```

Run tests:

```bash
pnpm test
```

Lint:

```bash
pnpm lint
```

Format check:

```bash
pnpm format:check
```

---

# 35. Prisma Migration

Development:

```bash
pnpm prisma migrate dev
```

Production:

```bash
pnpm prisma migrate deploy
```

Production schema changes should be controlled through the project's database
migration workflow.

Do not let every application Pod independently modify the production schema.

The preferred production architecture is:

```text
Git
 ↓
Migration
 ↓
CI validation
 ↓
Migration Job
 ↓
PostgreSQL
 ↓
Application deployment
```

---

# 36. Helm Chart

## `Chart.yaml`

```yaml
apiVersion: v2

name: customer-service

description: Customer business service

type: application

version: 0.1.0

appVersion: "1.0.0"
```

---

# 37. Helm Values

## `values.yaml`

```yaml
replicaCount: 2

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/customer-service

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

  minReplicas: 2

  maxReplicas: 10

  targetCPUUtilizationPercentage: 60


serviceAccount:

  create: true

  name: customer-service


podDisruptionBudget:

  enabled: true

  minAvailable: 1


secretStore:

  name: ecommerce-secrets
```

---

# 38. Production Values

## `values-prod.yaml`

```yaml
replicaCount: 3

image:

  repository: 123456789012.dkr.ecr.ap-south-1.amazonaws.com/customer-service

  tag: ""

  digest: ""


resources:

  requests:

    cpu: 500m

    memory: 512Mi

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

These are project starting values. Actual capacity must come from load testing.

---

# 39. Kubernetes Deployment

## `templates/deployment.yaml`

```yaml
apiVersion: apps/v1

kind: Deployment

metadata:

  name: customer-service

  labels:

    app: customer-service

spec:

  replicas: {{ .Values.replicaCount }}

  strategy:

    type: RollingUpdate

    rollingUpdate:

      maxUnavailable: 0

      maxSurge: 1

  selector:

    matchLabels:

      app: customer-service

  template:

    metadata:

      labels:

        app: customer-service

    spec:

      serviceAccountName:
        {{ .Values.serviceAccount.name }}

      terminationGracePeriodSeconds: 30

      securityContext:

        runAsNonRoot: true

        seccompProfile:

          type: RuntimeDefault

      containers:

        - name: customer-service

          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"

          imagePullPolicy: {{ .Values.image.pullPolicy }}

          ports:

            - name: http

              containerPort: 8080

          envFrom:

            - configMapRef:

                name: customer-service-config

            - secretRef:

                name: customer-service-secrets

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

# 40. Kubernetes Service

## `templates/service.yaml`

```yaml
apiVersion: v1

kind: Service

metadata:

  name: customer-service

  labels:

    app: customer-service

spec:

  type: ClusterIP

  selector:

    app: customer-service

  ports:

    - name: http

      port: 8080

      targetPort: http
```

Internal DNS:

```text
customer-service.ecommerce.svc.cluster.local
```

---

# 41. ConfigMap

## `templates/configmap.yaml`

```yaml
apiVersion: v1

kind: ConfigMap

metadata:

  name: customer-service-config

data:

  NODE_ENV: prod

  PORT: "8080"

  REDIS_PORT: "6379"
```

---

# 42. External Secrets

## `templates/externalsecret.yaml`

```yaml
apiVersion: external-secrets.io/v1beta1

kind: ExternalSecret

metadata:

  name: customer-service-secrets

spec:

  refreshInterval: 1h

  secretStoreRef:

    name: {{ .Values.secretStore.name }}

    kind: ClusterSecretStore

  target:

    name: customer-service-secrets

    creationPolicy: Owner

  data:

    - secretKey: DATABASE_URL

      remoteRef:

        key: ecommerce/prod/customer-service

        property: DATABASE_URL
```

Additional Kafka/Redis secrets should be added if the selected production
deployment requires authentication credentials.

---

# 43. ServiceAccount

## `templates/serviceaccount.yaml`

```yaml
apiVersion: v1

kind: ServiceAccount

metadata:

  name: customer-service
```

If AWS API access is required:

```text
customer-service Pod
        ↓
EKS Pod Identity
        ↓
customer-service IAM role
        ↓
specific AWS APIs
```

No static AWS access keys are placed inside the container.

---

# 44. NetworkPolicy

## `templates/networkpolicy.yaml`

```yaml
apiVersion: networking.k8s.io/v1

kind: NetworkPolicy

metadata:

  name: customer-service

spec:

  podSelector:

    matchLabels:

      app: customer-service

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

The production policy should be tightened to the actual database, Redis and
Kafka endpoints and namespace/network architecture.

---

# 45. HPA

## `templates/hpa.yaml`

```yaml
apiVersion: autoscaling/v2

kind: HorizontalPodAutoscaler

metadata:

  name: customer-service

spec:

  scaleTargetRef:

    apiVersion: apps/v1

    kind: Deployment

    name: customer-service

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

---

# 46. PodDisruptionBudget

## `templates/pdb.yaml`

```yaml
apiVersion: policy/v1

kind: PodDisruptionBudget

metadata:

  name: customer-service

spec:

  minAvailable: 2

  selector:

    matchLabels:

      app: customer-service
```

---

# 47. Prometheus ServiceMonitor

## `templates/servicemonitor.yaml`

```yaml
apiVersion: monitoring.coreos.com/v1

kind: ServiceMonitor

metadata:

  name: customer-service

  labels:

    release: kube-prometheus-stack

spec:

  selector:

    matchLabels:

      app: customer-service

  endpoints:

    - port: http

      path: /metrics

      interval: 15s
```

---

# 48. GitHub CODEOWNERS

## `.github/CODEOWNERS`

```text
/src/       @company-ecommerce/backend-team
/prisma/    @company-ecommerce/backend-team
/helm/      @company-ecommerce/platform-team
/.github/   @company-ecommerce/platform-team
/security/  @company-ecommerce/security-team
```

---

# 49. GitHub Actions CI

## `.github/workflows/ci.yaml`

```yaml
name: Customer Service CI

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

  ECR_REPOSITORY: customer-service


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

          cache: pnpm


      - name: Enable Corepack

        run: corepack enable


      - name: Install dependencies

        run: pnpm install --frozen-lockfile


      - name: Generate Prisma Client

        run: pnpm prisma generate


      - name: Lint

        run: pnpm lint


      - name: Format Check

        run: pnpm format:check


      - name: Unit Tests

        run: pnpm test -- --runInBand


      - name: Build

        run: pnpm build


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

      - test

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

        run: |
          docker build \
            -t "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}" \
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
            "$REGISTRY/$ECR_REPOSITORY:${GITHUB_SHA}"


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

---

# 50. CI Pipeline

The actual pipeline is:

    Pull Request
        |
        ↓
    Install dependencies
        |
        ↓
    Prisma Generate
        |
        ↓
    ESLint
        |
        ↓
    Prettier
        |
        ↓
    Jest
        |
        ↓
    Build
        |
        ↓
    SonarQube
        |
        ↓
    Dependency Scan
        |
        ↓
    Secret Scan
        |
        ↓
    Docker Build
        |
        ↓
    Trivy
        |
        ↓
    AWS OIDC
        |
        ↓
    ECR
        |
        ↓
    Image Digest
        |
        ↓
    GitOps

The exact SonarQube integration is kept consistent with the organization's
approved scanner/version.

---

# 51. ECR

Repository:

    company-ecommerce/customer-service

Example:

    123456789012.dkr.ecr.ap-south-1.amazonaws.com/customer-service:<git-sha>

Production should use an immutable image digest.

Example:

    customer-service@sha256:...

---

# 52. GitOps

Application repository:

    company-ecommerce/customer-service

GitOps repository:

    company-ecommerce/gitops

Example:

    gitops/
    └── apps/
        └── customer-service/
            ├── dev/
            ├── qa/
            ├── staging/
            └── prod/

Promotion:

    CI
     ↓
    ECR
     ↓
    GitOps PR
     ↓
    Approval
     ↓
    Merge
     ↓
    ArgoCD
     ↓
    EKS

CI does not directly deploy to production using:

    kubectl apply

---

# 53. ArgoCD Application

Example:

```yaml
apiVersion: argoproj.io/v1alpha1

kind: Application

metadata:

  name: customer-service-prod

  namespace: argocd

spec:

  project: ecommerce

  source:

    repoURL: https://github.com/company-ecommerce/gitops.git

    targetRevision: main

    path: apps/customer-service/prod

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

Production synchronization policy should follow the project's approval/change
management model.

---

# 54. Prometheus Metrics

Important platform metrics:

    HTTP request rate
    HTTP latency
    HTTP 4xx
    HTTP 5xx
    Node.js process CPU
    Node.js process memory
    Event loop behavior
    PostgreSQL connection usage
    Redis latency
    Kafka producer latency

Business metrics:

    customer_profile_get_total
    customer_profile_update_total
    address_created_total
    address_update_total
    customer_not_found_total

Do not use customer IDs, email addresses or phone numbers as metric labels.

---

# 55. Grafana Dashboard

Customer service dashboard:

    API
      ├── RPS
      ├── P50
      ├── P95
      ├── P99
      ├── 4xx
      └── 5xx

    Node.js
      ├── CPU
      ├── Memory
      ├── Event-loop behavior
      └── Restarts

    PostgreSQL
      ├── Connections
      ├── Query latency
      ├── DB load
      └── Errors

    Redis
      ├── Hit ratio
      ├── Latency
      ├── Errors
      └── Connections

    Kafka
      ├── Producer errors
      ├── Publish latency
      └── Event throughput

    Kubernetes
      ├── Replicas
      ├── HPA
      ├── CPU
      └── Memory

---

# 56. Logging

Application logs should be structured JSON.

Example:

```json
{
  "@timestamp": "2026-10-05T12:00:00Z",
  "level": "INFO",
  "service": "customer-service",
  "environment": "prod",
  "namespace": "ecommerce",
  "pod": "customer-service-7f4d9c-x7abc",
  "version": "1.0.0",
  "correlationId": "abc-123",
  "message": "Customer profile retrieved"
}
```

Do not log:

    Password
    Authorization header
    JWT
    Session token
    Database password
    Redis password
    Full sensitive customer information

---

# 57. OpenTelemetry

Architecture:

    Client
       |
       ↓
    ALB
       |
       ↓
    customer-service
       |
       ├── PostgreSQL
       |
       ├── Redis
       |
       └── Kafka

Telemetry:

    customer-service
          ↓
    OpenTelemetry
          ↓
    OTel Collector
          ↓
        Jaeger

The trace should allow engineers to understand:

    HTTP request
        ↓
    PostgreSQL query
        ↓
    Redis operation
        ↓
    Kafka publication

---

# 58. Correlation ID

Incoming request:

    X-Correlation-ID: abc-123

The service propagates it to downstream calls and asynchronous events.

Example event:

```json
{
  "eventId": "01J...",
  "eventType": "CustomerUpdated",
  "version": 1,
  "occurredAt": "2026-10-05T12:00:00Z",
  "source": "customer-service",
  "correlationId": "abc-123",
  "payload": {
    "customerId": "..."
  }
}
```

This allows:

    ALB
     ↓
    customer-service
     ↓
    PostgreSQL
     ↓
    Kafka
     ↓
    notification-service

to be investigated as one business transaction.

---

# 59. SLO

Initial project targets:

## Availability

    99.9%

## Customer GET P95

    < 200ms

## Customer Update P95

    < 300ms

These are simulation targets.

Actual production targets must be established from business requirements,
historical performance and load testing.

---

# 60. Failure Scenario — Redis

Redis failure:

    Redis unavailable
         ↓
    Cache miss
         ↓
    PostgreSQL traffic increases

Protection:

    Short Redis timeout
          +
    Bounded retry
          +
    Request coalescing
          +
    Database connection protection

We must avoid:

    Redis failure
         ↓
    thousands of simultaneous DB requests
         ↓
    RDS overload

---

# 61. Failure Scenario — PostgreSQL

If PostgreSQL becomes unavailable:

    customer-service
         ↓
    DB connection failure
         ↓
    API errors

Investigate:

    RDS status
    RDS events
    DNS
    Security groups
    NetworkPolicy
    credentials
    connection pool
    database connections
    query latency

Do not blindly increase connection pool size.

---

# 62. Failure Scenario — Kafka

If Kafka is temporarily unavailable:

    Customer update
         ↓
    PostgreSQL succeeds
         ↓
    Kafka publish fails

This creates a consistency problem.

For important customer events, the production design should use:

    Database transaction
          +
    Transactional Outbox
          ↓
    Kafka publisher
          ↓
    Kafka

This avoids the dangerous:

    DB SUCCESS
    Kafka FAILURE

without a durable event record.

---

# 63. Database Connection Capacity

Node.js/Prisma connection management must be included in platform capacity
planning.

Conceptually:

    Potential DB connections
        ≈
    Pods × connections per process

If Prisma is configured for a maximum of 10 connections:

    10 Pods
       ×
    10 connections
       =
    100 potential connections

But during deployments we can have:

    old Pods
       +
    new Pods
       +
    canary Pods

Therefore:

    Database capacity
        must include
    HPA
    rollout surge
    canary
    blue/green
    background jobs
    admin connections
    other services

---

# 64. Example Production Incident

## Incident ID

    INC-2026-CUSTOMER-001

## Severity

    SEV-2

## Title

    Customer API Latency Increased Due to Cache Stampede

---

# 65. Customer Impact

Customers experienced:

    Slow customer profile pages
    Increased API latency
    Some 5xx responses

Orders and payment processing remained operational.

---

# 66. Timeline

    15:00
    Normal customer API traffic.

    15:05
    Redis cluster experiences elevated latency.

    15:06
    Customer cache hit ratio drops.

    15:07
    Large number of requests fall back to PostgreSQL.

    15:08
    RDS read traffic increases sharply.

    15:09
    PostgreSQL query latency increases.

    15:10
    Customer API P95 increases from 180ms to 1.8s.

    15:11
    HPA increases customer-service Pods.

    15:12
    Additional Pods increase database connections.

    15:13
    5xx begins increasing.

    15:14
    SEV-2 declared.

    15:16
    Platform team confirms Redis degradation.

    15:18
    Application team enables stricter fallback concurrency.

    15:20
    Noncritical customer reads are rate limited.

    15:22
    Redis recovers.

    15:24
    Cache hit ratio begins recovering.

    15:26
    RDS load decreases.

    15:28
    Customer API latency returns to normal.

---

# 67. Root Cause

Primary failure:

    Redis became temporarily unavailable/degraded.

Amplification:

    Redis degradation
         ↓
    Cache misses
         ↓
    PostgreSQL traffic
         ↓
    RDS pressure
         ↓
    HPA scaling
         ↓
    More database connections
         ↓
    Customer API latency

The root cause was therefore not simply:

    "Redis failed."

The complete RCA identifies both:

    Primary failure
        +
    Amplification mechanism

---

# 68. Five Whys

### Why was customer API slow?

PostgreSQL queries became slow.

### Why?

Database traffic increased sharply.

### Why?

Redis cache misses increased.

### Why?

Redis became temporarily unavailable.

### Why did Redis failure cause such a large DB increase?

The cache fallback did not sufficiently limit concurrent PostgreSQL requests.

---

# 69. Corrective Actions

| Action | Owner |
|---|---|
| Add cache stampede protection | Backend |
| Add request coalescing | Backend |
| Add bounded DB fallback | Backend |
| Add Redis dependency alerts | SRE |
| Add RDS connection alerts | SRE |
| Review HPA max replicas | Platform |
| Add Redis failure load test | Performance |
| Add cache hit SLO | SRE |
| Add customer-service runbook | SRE |
| Add dependency failure tests | Platform |

---

# 70. Troubleshooting Commands

Check Pods:

```bash
kubectl get pods \
  -n ecommerce \
  -l app=customer-service
```

Check logs:

```bash
kubectl logs \
  -n ecommerce \
  <pod-name>
```

Check previous container:

```bash
kubectl logs \
  -n ecommerce \
  <pod-name> \
  --previous
```

Check HPA:

```bash
kubectl get hpa \
  -n ecommerce \
  customer-service
```

Check deployment:

```bash
kubectl get deployment \
  -n ecommerce \
  customer-service
```

Check Service:

```bash
kubectl get svc \
  -n ecommerce \
  customer-service
```

Check EndpointSlices:

```bash
kubectl get endpointslice \
  -n ecommerce \
  -l kubernetes.io/service-name=customer-service
```

Check node resource usage:

```bash
kubectl top pods \
  -n ecommerce
```

Check events:

```bash
kubectl get events \
  -n ecommerce \
  --sort-by=.lastTimestamp
```

---

# 71. Security Checklist

- [ ] No secrets in Git
- [ ] No production `.env`
- [ ] Least-privilege IAM
- [ ] Pod Identity where AWS APIs are required
- [ ] NetworkPolicy
- [ ] Non-root container
- [ ] Read-only root filesystem where compatible
- [ ] Dependency scanning
- [ ] Trivy
- [ ] SonarQube
- [ ] No sensitive customer data in logs
- [ ] PostgreSQL private
- [ ] Redis private
- [ ] TLS for production dependencies
- [ ] API input validation
- [ ] Authorization checks
- [ ] Audit events where required

---

# 72. Production Checklist

## Application

- [ ] TypeScript
- [ ] NestJS
- [ ] DTO validation
- [ ] Prisma
- [ ] Error handling
- [ ] Unit tests
- [ ] Integration tests
- [ ] Health endpoint
- [ ] Readiness endpoint
- [ ] Graceful shutdown

## Database

- [ ] PostgreSQL
- [ ] Prisma migrations
- [ ] Connection limits
- [ ] RDS backups
- [ ] PITR
- [ ] Private networking

## Redis

- [ ] Cache-aside
- [ ] TTL
- [ ] Cache invalidation
- [ ] Timeout
- [ ] Bounded retry
- [ ] Stampede protection
- [ ] Failure fallback

## Kafka

- [ ] Event contract
- [ ] Event ID
- [ ] Correlation ID
- [ ] Event version
- [ ] Idempotency
- [ ] Outbox where required

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

- [ ] pnpm
- [ ] ESLint
- [ ] Prettier
- [ ] Jest
- [ ] Prisma generation
- [ ] Build
- [ ] SonarQube
- [ ] Dependency scan
- [ ] Secret scan
- [ ] Docker
- [ ] Trivy
- [ ] ECR
- [ ] GitHub OIDC

## GitOps

- [ ] Helm
- [ ] Environment values
- [ ] ArgoCD
- [ ] Production approval
- [ ] Immutable image
- [ ] Image digest
- [ ] Rollback

## Observability

- [ ] Prometheus
- [ ] Grafana
- [ ] Structured logging
- [ ] ELK
- [ ] Correlation ID
- [ ] OpenTelemetry
- [ ] Jaeger
- [ ] Alerts
- [ ] SLO
- [ ] Runbook

---

# 73. Final Architecture

                         Internet
                            |
                     CloudFront / WAF
                            |
                           ALB
                            |
                    customer-service
                            |
              ┌─────────────┼──────────────┐
              │             │              │
              ↓             ↓              ↓
         PostgreSQL       Redis          Kafka
              │             │              │
              │             │              │
              └─────────────┼──────────────┘
                            |
                     Customer Domain
                            |
              ┌─────────────┴──────────────┐
              │                            │
          Prometheus                 OpenTelemetry
              │                            │
           Grafana                    Collector
                                           |
                                         Jaeger

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

---

# 74. DevOps Learning

This service introduces another application stack:

    TypeScript
        ↓
    Node.js
        ↓
    NestJS
        ↓
    Prisma
        ↓
    PostgreSQL

Compared with our previous services:

    Auth:
        Java
        Spring Boot
        Maven

    User:
        Python
        FastAPI
        uv

    Customer:
        TypeScript
        Node.js
        NestJS
        pnpm

But the platform remains:

    Docker
        ↓
    ECR
        ↓
    GitHub Actions
        ↓
    GitOps
        ↓
    ArgoCD
        ↓
    EKS
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

This is the core idea behind our polyglot platform.

# End of 163 — Customer Service
```

### Phase 19 progress

```text
161 Auth Service           ✅ Java / Spring Boot
162 User Service           ✅ Python / FastAPI
163 Customer Service       ✅ TypeScript / Node.js / NestJS

164 Catalog Service        ⏳ Go / Gin
165 Product Service        ⏳ Java / Spring Boot
166 Inventory Service      ⏳ Go / Gin
167 Pricing Service        ⏳ Python / FastAPI
168 Promotion Service      ⏳ Node.js / TypeScript / NestJS
169 Cart Service           ⏳ Java / Spring Boot
170 Order Service          ⏳ Java / Spring Boot
171 Payment Service        ⏳ C# / ASP.NET Core
172 Fraud Service          ⏳ Python / FastAPI
173 Shipping Service       ⏳ Node.js / TypeScript / NestJS
174 Delivery Service       ⏳ Go / Gin
175 Notification Service   ⏳ Python / FastAPI
176 Search Service         ⏳ Java / Spring Boot
177 Recommendation Service ⏳ Python / FastAPI
178 Review Service         ⏳ Ruby / Rails
179 Tax Service            ⏳ C# / ASP.NET Core
180 Invoice Service        ⏳ Node.js / TypeScript / NestJS
181 Reporting Service      ⏳ Python / FastAPI
```

**Next: `164-Catalog-Service.md` — Go + Gin.**