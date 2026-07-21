# Slice 1 — Dataset Upload & Schema Detection

## What Are We Building?

A user uploads a CSV file. The system parses it, auto-detects the schema, validates that required fields exist (`user_id`, `timestamp`, `event_name`), stores the raw file in MinIO (object storage), saves dataset metadata to PostgreSQL, and fires a `DatasetUploaded` domain event so downstream processes can react.

**Endpoint delivered:** `POST /datasets/upload`
**Architecture layers touched:** Presentation → Application → Domain → Infrastructure

---

## Vertical Slice Overview

```
HTTP Request (multipart CSV)
        ↓
  DatasetController             ← Presentation  (Driving Adapter)
        ↓ [IUploadDatasetUseCase]  ← Inbound Port
  UploadDatasetHandler          ← Application   (implements inbound port + ICommandHandler)
        ↓
  SchemaDetectorService         ← Domain Service
  DatasetValidatorService       ← Domain Service
  Dataset (Aggregate Root)      ← Domain
        ↓ [IDatasetRepository]    ← Outbound Port
        ↓ [IFileStoragePort]      ← Outbound Port
  DatasetRepository             ← Infrastructure (TypeORM adapter)
  FileStorageService            ← Infrastructure (MinIO adapter)
        ↓
  DatasetUploaded event         ← Domain Event
        ↓
  PostgreSQL + MinIO
```

---

## File Structure Created in This Slice

```
src/
├── modules/
│   └── dataset/
│       ├── domain/
│       │   ├── dataset.entity.ts
│       │   ├── dataset-schema.value-object.ts
│       │   ├── dataset-status.enum.ts
│       │   └── events/
│       │       └── dataset-uploaded.event.ts
│       ├── application/
│       │   ├── ports/
│       │   │   ├── inbound/
│       │   │   │   └── upload-dataset.use-case.ts
│       │   │   └── outbound/
│       │   │       ├── dataset-repository.port.ts
│       │   │       └── file-storage.port.ts
│       │   ├── commands/
│       │   │   ├── upload-dataset.command.ts
│       │   │   └── upload-dataset.handler.ts
│       │   └── services/
│       │       ├── schema-detector.service.ts
│       │       └── dataset-validator.service.ts
│       ├── infrastructure/
│       │   ├── persistence/
│       │   │   ├── dataset.orm-entity.ts
│       │   │   └── dataset.repository.ts
│       │   └── storage/
│       │       └── file-storage.service.ts
│       ├── presentation/
│       │   ├── dataset.controller.ts
│       │   └── dto/
│       │       └── upload-dataset-response.dto.ts
│       └── dataset.module.ts
├── shared/
│   └── events/
│       └── domain-event.interface.ts
docker-compose.yml
```

---

### Step 1 — Install Dependencies

## Step-by-Step Implementation Guide

---

### Progress Checklist

- [x] Step 1 — Install Dependencies (install packages)
- [ ] Step 2 — Docker Compose (start Postgres & MinIO)
- [ ] Step 3 — Environment Config (`.env`)
- [ ] Step 4 — Shared Domain Event Interface
- [ ] Step 5 — Domain Layer (create value object, status enum, aggregate, event)
  - [ ] 5a Dataset Status Enum: `src/modules/dataset/domain/dataset-status.enum.ts`
  - [ ] 5b Schema Value Object: `src/modules/dataset/domain/dataset-schema.value-object.ts`
  - [ ] 5c DatasetUploaded Event: `src/modules/dataset/domain/events/dataset-uploaded.event.ts`
  - [ ] 5d Dataset Aggregate Root: `src/modules/dataset/domain/dataset.entity.ts`
- [ ] Step 6 — Application Layer (CQRS & Ports)
  - [ ] 6a Ports (inbound/outbound): `src/modules/dataset/application/ports/**`
  - [ ] 6b Upload Command: `src/modules/dataset/application/commands/upload-dataset.command.ts`
  - [ ] 6c Schema Detector Service: `src/modules/dataset/application/services/schema-detector.service.ts`
  - [ ] 6d Dataset Validator Service: `src/modules/dataset/application/services/dataset-validator.service.ts`
  - [ ] 6e Row Counter Utility: add to schema-detector.service
  - [ ] 6f Upload Command Handler: `src/modules/dataset/application/commands/upload-dataset.handler.ts`
- [ ] Step 7 — Infrastructure Layer
  - [ ] 7a ORM Entity: `src/modules/dataset/infrastructure/persistence/dataset.orm-entity.ts`
  - [ ] 7b Repository: `src/modules/dataset/infrastructure/persistence/dataset.repository.ts`
  - [ ] 7c File Storage Service: `src/modules/dataset/infrastructure/storage/file-storage.service.ts`
- [ ] Step 8 — Presentation Layer
  - [ ] 8a Response DTO: `src/modules/dataset/presentation/dto/upload-dataset-response.dto.ts`
  - [ ] 8b Controller: `src/modules/dataset/presentation/dataset.controller.ts`
- [ ] Step 9 — Dataset Module: `src/modules/dataset/dataset.module.ts`
- [ ] Step 10 — Wire into AppModule: `src/app.module.ts`

### Acceptance Criteria Checklist

- [ ] AC1: `POST /datasets/upload` returns `201` and `datasetId`
- [ ] AC2: Missing required fields returns `422` with missing list
- [ ] AC3: Non-CSV upload returns `400 Bad Request`
- [ ] AC4: Oversize file returns `413 Payload Too Large`
- [ ] AC5: File stored in MinIO under `datasets/<uuid>/<filename>`
- [ ] AC6: `datasets` row created in PostgreSQL with metadata
- [ ] AC7: `DatasetUploadedEvent` emitted on NestJS event bus

---

### Step 1 — Install Dependencies

Run the following to add all packages needed for this slice:

```bash
pnpm add @nestjs/cqrs @nestjs/typeorm typeorm pg @nestjs/config \
         minio multer csv-parse uuid
pnpm add -D @types/multer @types/uuid
```

**Why each package:**

- `@nestjs/cqrs` — command bus for UploadDatasetHandler
- `typeorm` + `pg` — PostgreSQL ORM
- `@nestjs/config` — env variable loading
- `minio` — MinIO S3-compatible client
- `multer` — multipart file uploads in Express
- `csv-parse` — streaming CSV parser
- `uuid` — generate dataset IDs

---

### Step 2 — Docker Compose (Infrastructure)

**File: `docker-compose.yml`** (project root)

```yaml
version: '3.9'

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: pulseiq
      POSTGRES_PASSWORD: pulseiq
      POSTGRES_DB: pulseiq
    ports:
      - '5432:5432'
    volumes:
      - postgres_data:/var/lib/postgresql/data

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports:
      - '9000:9000'
      - '9001:9001'
    volumes:
      - minio_data:/data

volumes:
  postgres_data:
  minio_data:
```

Start infra:

```bash
docker compose up -d
```

---

### Step 3 — Environment Config

**File: `.env`** (project root)

```env
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USER=pulseiq
DATABASE_PASSWORD=pulseiq
DATABASE_NAME=pulseiq

MINIO_ENDPOINT=localhost
MINIO_PORT=9000
MINIO_ACCESS_KEY=minioadmin
MINIO_SECRET_KEY=minioadmin
MINIO_BUCKET=datasets
```

---

### Step 4 — Shared Domain Event Interface

This base interface is used by all domain events across every module.

**File: `src/shared/events/domain-event.interface.ts`**

```typescript
export interface DomainEvent {
  readonly occurredAt: Date;
}
```

---

### Step 5 — Domain Layer

#### 5a. Dataset Status Enum

**File: `src/modules/dataset/domain/dataset-status.enum.ts`**

```typescript
export enum DatasetStatus {
  PENDING = 'PENDING',
  VALIDATED = 'VALIDATED',
  PROCESSING = 'PROCESSING',
  READY = 'READY',
  FAILED = 'FAILED',
}
```

#### 5b. Schema Value Object

Holds the detected column names and which required fields were found. A value object has no identity — equality is by content.

**File: `src/modules/dataset/domain/dataset-schema.value-object.ts`**

```typescript
export interface DetectedField {
  name: string;
  sample: string | null;
}

export class DatasetSchema {
  constructor(
    public readonly fields: DetectedField[],
    public readonly hasUserId: boolean,
    public readonly hasTimestamp: boolean,
    public readonly hasEventName: boolean,
  ) {}

  get isValid(): boolean {
    return this.hasUserId && this.hasTimestamp && this.hasEventName;
  }

  get missingFields(): string[] {
    const missing: string[] = [];
    if (!this.hasUserId) missing.push('user_id');
    if (!this.hasTimestamp) missing.push('timestamp');
    if (!this.hasEventName) missing.push('event_name');
    return missing;
  }
}
```

#### 5c. DatasetUploaded Domain Event

**File: `src/modules/dataset/domain/events/dataset-uploaded.event.ts`**

```typescript
import { DomainEvent } from '../../../../shared/events/domain-event.interface';

export class DatasetUploadedEvent implements DomainEvent {
  readonly occurredAt = new Date();

  constructor(
    public readonly datasetId: string,
    public readonly originalFileName: string,
    public readonly storagePath: string,
    public readonly rowCount: number,
  ) {}
}
```

#### 5d. Dataset Aggregate Root (Entity)

The root of the Dataset aggregate. It owns the business rules for creation.

**File: `src/modules/dataset/domain/dataset.entity.ts`**

```typescript
import { DatasetSchema } from './dataset-schema.value-object';
import { DatasetStatus } from './dataset-status.enum';
import { DatasetUploadedEvent } from './events/dataset-uploaded.event';

export class Dataset {
  private _domainEvents: DatasetUploadedEvent[] = [];

  constructor(
    public readonly id: string,
    public readonly originalFileName: string,
    public readonly storagePath: string,
    public readonly rowCount: number,
    public readonly schema: DatasetSchema,
    public status: DatasetStatus,
    public readonly uploadedAt: Date,
  ) {}

  static create(props: {
    id: string;
    originalFileName: string;
    storagePath: string;
    rowCount: number;
    schema: DatasetSchema;
  }): Dataset {
    const dataset = new Dataset(
      props.id,
      props.originalFileName,
      props.storagePath,
      props.rowCount,
      props.schema,
      DatasetStatus.VALIDATED,
      new Date(),
    );

    dataset._domainEvents.push(
      new DatasetUploadedEvent(
        props.id,
        props.originalFileName,
        props.storagePath,
        props.rowCount,
      ),
    );

    return dataset;
  }

  pullDomainEvents(): DatasetUploadedEvent[] {
    const events = [...this._domainEvents];
    this._domainEvents = [];
    return events;
  }
}
```

---

### Step 6 — Application Layer (CQRS)

#### 6a. Ports — Inbound & Outbound

Ports are plain TypeScript interfaces that live in the **application layer**. They are the only coupling point between the application core and the outside world — nothing in `domain/` or `application/` ever imports from `infrastructure/` or `presentation/`.

| Direction    | Who defines it    | Who implements it      | Who calls it |
| ------------ | ----------------- | ---------------------- | ------------ |
| **Inbound**  | application layer | handler                | controller   |
| **Outbound** | application layer | infrastructure adapter | handler      |

NestJS does not support TypeScript interfaces as DI tokens directly, so each port exports a `Symbol` injection token alongside its interface.

---

**Inbound port — `src/modules/dataset/application/ports/inbound/upload-dataset.use-case.ts`**

```typescript
import { UploadDatasetCommand } from '../../commands/upload-dataset.command';

export interface UploadDatasetResult {
  datasetId: string;
}

export interface IUploadDatasetUseCase {
  execute(command: UploadDatasetCommand): Promise<UploadDatasetResult>;
}

export const UPLOAD_DATASET_USE_CASE = Symbol('IUploadDatasetUseCase');
```

> The controller depends only on `IUploadDatasetUseCase`. It never imports `UploadDatasetHandler` or `CommandBus` directly.

---

**Outbound port — `src/modules/dataset/application/ports/outbound/dataset-repository.port.ts`**

```typescript
import { Dataset } from '../../../domain/dataset.entity';

export interface IDatasetRepository {
  save(dataset: Dataset): Promise<void>;
}

export const DATASET_REPOSITORY = Symbol('IDatasetRepository');
```

**Outbound port — `src/modules/dataset/application/ports/outbound/file-storage.port.ts`**

```typescript
export interface IFileStoragePort {
  upload(
    objectPath: string,
    buffer: Buffer,
    contentType: string,
  ): Promise<void>;
}

export const FILE_STORAGE_PORT = Symbol('IFileStoragePort');
```

> The handler depends only on `IDatasetRepository` and `IFileStoragePort`. It never imports `DatasetRepository` or `FileStorageService`.

---

#### 6b. Upload Command

A plain data object — no logic, just carries intent.

**File: `src/modules/dataset/application/commands/upload-dataset.command.ts`**

```typescript
export class UploadDatasetCommand {
  constructor(public readonly file: Express.Multer.File) {}
}
```

#### 6c. Schema Detector Service

Reads the first two rows of the CSV to extract column names and sample values. Uses `csv-parse` in sync mode on the buffered file.

**File: `src/modules/dataset/application/services/schema-detector.service.ts`**

```typescript
import { Injectable } from '@nestjs/common';
import { parse } from 'csv-parse/sync';
import {
  DatasetSchema,
  DetectedField,
} from '../../domain/dataset-schema.value-object';

@Injectable()
export class SchemaDetectorService {
  detect(fileBuffer: Buffer): DatasetSchema {
    const rows = parse(fileBuffer, {
      columns: true,
      skip_empty_lines: true,
      to: 2, // only need header + 1 sample row
    }) as Record<string, string>[];

    if (rows.length === 0) {
      return new DatasetSchema([], false, false, false);
    }

    const columnNames = Object.keys(rows[0]);
    const sample = rows[0];

    const fields: DetectedField[] = columnNames.map((name) => ({
      name,
      sample: sample[name] ?? null,
    }));

    const normalizedNames = columnNames.map((c) => c.toLowerCase().trim());

    return new DatasetSchema(
      fields,
      normalizedNames.includes('user_id'),
      normalizedNames.includes('timestamp'),
      normalizedNames.includes('event_name'),
    );
  }
}
```

#### 6d. Dataset Validator Service

Throws a descriptive error if required fields are missing.

**File: `src/modules/dataset/application/services/dataset-validator.service.ts`**

```typescript
import { Injectable, UnprocessableEntityException } from '@nestjs/common';
import { DatasetSchema } from '../../domain/dataset-schema.value-object';

@Injectable()
export class DatasetValidatorService {
  validate(schema: DatasetSchema): void {
    if (!schema.isValid) {
      throw new UnprocessableEntityException(
        `CSV is missing required fields: ${schema.missingFields.join(', ')}`,
      );
    }
  }
}
```

#### 6e. Row Counter Utility

Count total rows (excluding header) without loading everything into memory.

**File: `src/modules/dataset/application/services/schema-detector.service.ts`** — add this method to `SchemaDetectorService`:

```typescript
countRows(fileBuffer: Buffer): number {
  const rows = parse(fileBuffer, {
    columns: true,
    skip_empty_lines: true,
  }) as Record<string, string>[];
  return rows.length;
}
```

#### 6f. Upload Command Handler

Orchestrates the full upload flow. It implements **both** `ICommandHandler` (so the CQRS command bus can dispatch to it) **and** `IUploadDatasetUseCase` (the inbound port the controller calls through). All infrastructure dependencies are injected via outbound port tokens — the handler never references concrete adapter classes.

**File: `src/modules/dataset/application/commands/upload-dataset.handler.ts`**

```typescript
import { CommandHandler, EventBus, ICommandHandler } from '@nestjs/cqrs';
import { Inject } from '@nestjs/common';
import { v4 as uuid } from 'uuid';
import { UploadDatasetCommand } from './upload-dataset.command';
import { SchemaDetectorService } from '../services/schema-detector.service';
import { DatasetValidatorService } from '../services/dataset-validator.service';
import { Dataset } from '../../domain/dataset.entity';
import {
  IUploadDatasetUseCase,
  UploadDatasetResult,
} from '../ports/inbound/upload-dataset.use-case';
import {
  IDatasetRepository,
  DATASET_REPOSITORY,
} from '../ports/outbound/dataset-repository.port';
import {
  IFileStoragePort,
  FILE_STORAGE_PORT,
} from '../ports/outbound/file-storage.port';

@CommandHandler(UploadDatasetCommand)
export class UploadDatasetHandler
  implements ICommandHandler<UploadDatasetCommand>, IUploadDatasetUseCase
{
  constructor(
    private readonly schemaDetector: SchemaDetectorService,
    private readonly validator: DatasetValidatorService,
    @Inject(DATASET_REPOSITORY)
    private readonly datasetRepo: IDatasetRepository,
    @Inject(FILE_STORAGE_PORT) private readonly fileStorage: IFileStoragePort,
    private readonly eventBus: EventBus,
  ) {}

  async execute(command: UploadDatasetCommand): Promise<UploadDatasetResult> {
    const { file } = command;

    // 1. Detect schema from CSV header
    const schema = this.schemaDetector.detect(file.buffer);

    // 2. Validate required fields
    this.validator.validate(schema);

    // 3. Count total rows
    const rowCount = this.schemaDetector.countRows(file.buffer);

    // 4. Upload raw file via outbound port (no MinIO import here)
    const datasetId = uuid();
    const storagePath = `datasets/${datasetId}/${file.originalname}`;
    await this.fileStorage.upload(storagePath, file.buffer, file.mimetype);

    // 5. Create aggregate (fires domain event internally)
    const dataset = Dataset.create({
      id: datasetId,
      originalFileName: file.originalname,
      storagePath,
      rowCount,
      schema,
    });

    // 6. Persist via outbound port (no TypeORM import here)
    await this.datasetRepo.save(dataset);

    // 7. Publish domain events
    const events = dataset.pullDomainEvents();
    this.eventBus.publishAll(events);

    return { datasetId: dataset.id };
  }
}
```

---

### Step 7 — Infrastructure Layer

#### 7a. TypeORM ORM Entity (database table mapping)

**File: `src/modules/dataset/infrastructure/persistence/dataset.orm-entity.ts`**

```typescript
import { Column, Entity, PrimaryColumn, CreateDateColumn } from 'typeorm';
import { DatasetStatus } from '../../domain/dataset-status.enum';

@Entity('datasets')
export class DatasetOrmEntity {
  @PrimaryColumn('uuid')
  id: string;

  @Column()
  originalFileName: string;

  @Column()
  storagePath: string;

  @Column('int')
  rowCount: number;

  @Column('jsonb')
  schema: object; // stores the DatasetSchema fields as JSON

  @Column({ type: 'enum', enum: DatasetStatus, default: DatasetStatus.PENDING })
  status: DatasetStatus;

  @CreateDateColumn()
  uploadedAt: Date;
}
```

#### 7b. Dataset Repository

Maps between the domain `Dataset` entity and the TypeORM `DatasetOrmEntity`.

**File: `src/modules/dataset/infrastructure/persistence/dataset.repository.ts`**

```typescript
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { DatasetOrmEntity } from './dataset.orm-entity';
import { Dataset } from '../../domain/dataset.entity';
import { IDatasetRepository } from '../../application/ports/outbound/dataset-repository.port';

@Injectable()
export class DatasetRepository implements IDatasetRepository {
  constructor(
    @InjectRepository(DatasetOrmEntity)
    private readonly ormRepo: Repository<DatasetOrmEntity>,
  ) {}

  async save(dataset: Dataset): Promise<void> {
    const ormEntity = this.ormRepo.create({
      id: dataset.id,
      originalFileName: dataset.originalFileName,
      storagePath: dataset.storagePath,
      rowCount: dataset.rowCount,
      schema: {
        fields: dataset.schema.fields,
        hasUserId: dataset.schema.hasUserId,
        hasTimestamp: dataset.schema.hasTimestamp,
        hasEventName: dataset.schema.hasEventName,
      },
      status: dataset.status,
      uploadedAt: dataset.uploadedAt,
    });

    await this.ormRepo.save(ormEntity);
  }
}
```

#### 7c. File Storage Service (MinIO)

**File: `src/modules/dataset/infrastructure/storage/file-storage.service.ts`**

```typescript
import { Injectable, OnModuleInit, Logger } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import * as Minio from 'minio';
import { IFileStoragePort } from '../../application/ports/outbound/file-storage.port';

@Injectable()
export class FileStorageService implements OnModuleInit, IFileStoragePort {
  private readonly logger = new Logger(FileStorageService.name);
  private readonly client: Minio.Client;
  private readonly bucket: string;

  constructor(private readonly config: ConfigService) {
    this.bucket = this.config.get<string>('MINIO_BUCKET', 'datasets');
    this.client = new Minio.Client({
      endPoint: this.config.get<string>('MINIO_ENDPOINT', 'localhost'),
      port: parseInt(this.config.get<string>('MINIO_PORT', '9000'), 10),
      useSSL: false,
      accessKey: this.config.get<string>('MINIO_ACCESS_KEY', 'minioadmin'),
      secretKey: this.config.get<string>('MINIO_SECRET_KEY', 'minioadmin'),
    });
  }

  async onModuleInit(): Promise<void> {
    const exists = await this.client.bucketExists(this.bucket);
    if (!exists) {
      await this.client.makeBucket(this.bucket);
      this.logger.log(`Created MinIO bucket: ${this.bucket}`);
    }
  }

  async upload(
    objectPath: string,
    buffer: Buffer,
    contentType: string,
  ): Promise<void> {
    await this.client.putObject(
      this.bucket,
      objectPath,
      buffer,
      buffer.length,
      {
        'Content-Type': contentType,
      },
    );
  }
}
```

---

### Step 8 — Presentation Layer

#### 8a. Response DTO

**File: `src/modules/dataset/presentation/dto/upload-dataset-response.dto.ts`**

```typescript
export class UploadDatasetResponseDto {
  datasetId: string;
  message: string;
}
```

#### 8b. Dataset Controller

Accepts multipart upload, enforces 10 MB limit, only allows `text/csv` and `.csv` files.

**File: `src/modules/dataset/presentation/dataset.controller.ts`**

The controller depends on the **inbound port interface** only — it has zero knowledge of the handler, command bus, or any infrastructure detail.

```typescript
import {
  Controller,
  Post,
  UploadedFile,
  UseInterceptors,
  BadRequestException,
  Inject,
} from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { memoryStorage } from 'multer';
import { UploadDatasetCommand } from '../application/commands/upload-dataset.command';
import { UploadDatasetResponseDto } from './dto/upload-dataset-response.dto';
import {
  IUploadDatasetUseCase,
  UPLOAD_DATASET_USE_CASE,
} from '../application/ports/inbound/upload-dataset.use-case';

@Controller('datasets')
export class DatasetController {
  constructor(
    @Inject(UPLOAD_DATASET_USE_CASE)
    private readonly uploadDatasetUseCase: IUploadDatasetUseCase,
  ) {}

  @Post('upload')
  @UseInterceptors(
    FileInterceptor('file', {
      storage: memoryStorage(),
      limits: { fileSize: 10 * 1024 * 1024 }, // 10 MB
      fileFilter: (_req, file, cb) => {
        if (
          file.mimetype === 'text/csv' ||
          file.originalname.toLowerCase().endsWith('.csv')
        ) {
          cb(null, true);
        } else {
          cb(new BadRequestException('Only CSV files are accepted'), false);
        }
      },
    }),
  )
  async upload(
    @UploadedFile() file: Express.Multer.File,
  ): Promise<UploadDatasetResponseDto> {
    if (!file) {
      throw new BadRequestException('No file provided');
    }

    const result = await this.uploadDatasetUseCase.execute(
      new UploadDatasetCommand(file),
    );

    return {
      datasetId: result.datasetId,
      message: 'Dataset uploaded and schema validated successfully',
    };
  }
}
```

---

### Step 9 — Dataset Module

Wires all providers, imports, and exports together.

**File: `src/modules/dataset/dataset.module.ts`**

The module is where port tokens get **bound to their concrete adapter implementations**. This is the only file that knows about both sides of each port.

```typescript
import { Module } from '@nestjs/common';
import { CqrsModule } from '@nestjs/cqrs';
import { TypeOrmModule } from '@nestjs/typeorm';

import { DatasetController } from './presentation/dataset.controller';
import { UploadDatasetHandler } from './application/commands/upload-dataset.handler';
import { SchemaDetectorService } from './application/services/schema-detector.service';
import { DatasetValidatorService } from './application/services/dataset-validator.service';
import { DatasetRepository } from './infrastructure/persistence/dataset.repository';
import { FileStorageService } from './infrastructure/storage/file-storage.service';
import { DatasetOrmEntity } from './infrastructure/persistence/dataset.orm-entity';
import { UPLOAD_DATASET_USE_CASE } from './application/ports/inbound/upload-dataset.use-case';
import { DATASET_REPOSITORY } from './application/ports/outbound/dataset-repository.port';
import { FILE_STORAGE_PORT } from './application/ports/outbound/file-storage.port';

@Module({
  imports: [CqrsModule, TypeOrmModule.forFeature([DatasetOrmEntity])],
  controllers: [DatasetController],
  providers: [
    // Inbound port → handler (controller calls through this symbol)
    { provide: UPLOAD_DATASET_USE_CASE, useClass: UploadDatasetHandler },
    // Also register for CQRS command bus dispatch
    UploadDatasetHandler,

    SchemaDetectorService,
    DatasetValidatorService,

    // Outbound port → infrastructure adapter bindings
    { provide: DATASET_REPOSITORY, useClass: DatasetRepository },
    { provide: FILE_STORAGE_PORT, useClass: FileStorageService },

    DatasetRepository,
    FileStorageService,
  ],
})
export class DatasetModule {}
```

---

### Step 10 — Wire into AppModule

**File: `src/app.module.ts`** — update to include TypeORM global config and DatasetModule:

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';
import { DatasetModule } from './modules/dataset/dataset.module';
import { DatasetOrmEntity } from './modules/dataset/infrastructure/persistence/dataset.orm-entity';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),

    TypeOrmModule.forRootAsync({
      inject: [ConfigService],
      useFactory: (config: ConfigService) => ({
        type: 'postgres',
        host: config.get('DATABASE_HOST'),
        port: config.get<number>('DATABASE_PORT'),
        username: config.get('DATABASE_USER'),
        password: config.get('DATABASE_PASSWORD'),
        database: config.get('DATABASE_NAME'),
        entities: [DatasetOrmEntity],
        synchronize: true, // dev only — use migrations in production
      }),
    }),

    DatasetModule,
  ],
})
export class AppModule {}
```

---

## Acceptance Criteria for Slice 1

| #   | Criteria                                                                                      |
| --- | --------------------------------------------------------------------------------------------- |
| 1   | `POST /datasets/upload` with a valid CSV returns `201` with a `datasetId`                     |
| 2   | CSV missing`user_id` / `timestamp` / `event_name` returns `422` with a list of missing fields |
| 3   | Uploading a non-CSV file returns`400 Bad Request`                                             |
| 4   | Uploading a file over 10 MB returns`413 Payload Too Large`                                    |
| 5   | File appears in MinIO under`datasets/<uuid>/<filename>`                                       |
| 6   | A row appears in the`datasets` PostgreSQL table with correct metadata                         |
| 7   | `DatasetUploadedEvent` is emitted on the NestJS event bus                                     |

---

## Manual Test — cURL

```bash
# Happy path
curl -X POST http://localhost:3000/datasets/upload \
  -F "file=@./sample_events.csv"

# Expected response:
# { "datasetId": "<uuid>", "message": "Dataset uploaded and schema validated successfully" }

# Missing required fields
curl -X POST http://localhost:3000/datasets/upload \
  -F "file=@./bad_events.csv"

# Expected response (422):
# { "statusCode": 422, "message": "CSV is missing required fields: user_id, timestamp" }
```

**Minimum valid `sample_events.csv` for testing:**

```csv
user_id,timestamp,event_name,channel
u001,2024-01-15T10:00:00Z,page_view,organic
u002,2024-01-15T10:05:00Z,sign_up,paid
```

---

## What This Slice Does NOT Cover

The following are intentionally left for future slices:

- **Slice 2** — Data normalization pipeline (timestamp parsing, null handling, type inference)
- **Slice 3** — Metrics calculation (DAU/WAU/MAU, new vs returning users)
- **Slice 4** — Retention analysis (D1/D7/D30)
- **Slice 5** — AI insight generation
- **Slice 6** — Dashboard API & frontend integration

---

## Dependency Summary

| Package                              | Purpose                |
| ------------------------------------ | ---------------------- |
| `@nestjs/cqrs`                       | Command/event bus      |
| `@nestjs/typeorm` + `typeorm` + `pg` | PostgreSQL ORM         |
| `@nestjs/config`                     | `.env` loading         |
| `minio`                              | Object storage client  |
| `multer`                             | File upload middleware |
| `csv-parse`                          | CSV parsing            |
| `uuid`                               | Generate dataset IDs   |
