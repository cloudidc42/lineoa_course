# Part 94: Event-Driven Architecture สำหรับ LINE Bots

## บทนำ

Event-Driven Architecture (EDA) เป็นรูปแบบการออกแบบระบบที่เหมาะสมอย่างยิ่งสำหรับ LINE Bot ที่มีการใช้งานสูง เพราะช่วยให้ระบบสามารถรับมือกับ traffic จำนวนมากได้อย่างมีประสิทธิภาพ ในบทนี้เราจะเรียนรู้การออกแบบและ implement ระบบ Event-Driven สำหรับ LINE Bot อย่างครบถ้วน

---

## 1. ความเข้าใจ Event-Driven Architecture

### 1.1 อะไรคือ Event-Driven Architecture?

Event-Driven Architecture คือรูปแบบการออกแบบซอฟต์แวร์ที่การทำงานของระบบขับเคลื่อนด้วย Events หรือเหตุการณ์ต่างๆ แทนที่จะใช้การเรียกฟังก์ชันโดยตรง

**หลักการสำคัญ:**
- **Event Producers** - ส่วนที่สร้าง Events
- **Event Consumers** - ส่วนที่รับและประมวลผล Events
- **Event Bus/Broker** - ตัวกลางในการส่ง Events
- **Event Store** - ที่เก็บประวัติ Events

### 1.2 ทำไมต้องใช้ EDA กับ LINE Bot?

LINE Bot ในระบบ Production มักประสบปัญหาเหล่านี้:
- รับ Webhook events จำนวนมากในเวลาเดียวกัน
- ต้องประมวลผลหลาย services พร้อมกัน
- ต้องการ traceability และ audit log
- ต้องการ retry mechanism เมื่อเกิดข้อผิดพลาด

```
LINE Platform → Webhook → Event Broker → Multiple Consumers
                                    ↓
                              Message Service
                              Analytics Service  
                              CRM Service
                              Notification Service
```

### 1.3 เปรียบเทียบ Traditional vs Event-Driven

**Traditional (Synchronous):**
```
LINE → Webhook Handler → Database → LINE Reply → Done
                ↓
        (ถ้าช้า = Timeout)
```

**Event-Driven (Asynchronous):**
```
LINE → Webhook Handler → Kafka → [Consumer 1: Reply]
                              → [Consumer 2: Analytics]
                              → [Consumer 3: CRM Update]
                              → [Consumer 4: Notification]
```

---

## 2. Event Types ใน LINE Ecosystem

### 2.1 LINE Webhook Events ทั้งหมด

```typescript
// types/line-events.ts

export enum LineEventType {
  // Message Events
  TEXT_MESSAGE = 'message:text',
  IMAGE_MESSAGE = 'message:image',
  VIDEO_MESSAGE = 'message:video',
  AUDIO_MESSAGE = 'message:audio',
  FILE_MESSAGE = 'message:file',
  LOCATION_MESSAGE = 'message:location',
  STICKER_MESSAGE = 'message:sticker',
  
  // User Action Events
  FOLLOW = 'follow',
  UNFOLLOW = 'unfollow',
  JOIN = 'join',
  LEAVE = 'leave',
  
  // Interactive Events
  POSTBACK = 'postback',
  BEACON = 'beacon',
  
  // Account Link Events
  ACCOUNT_LINK = 'accountLink',
  
  // Things Events
  THINGS_LINK = 'things:link',
  THINGS_UNLINK = 'things:unlink',
  THINGS_SCENARIO = 'things:scenarioResult',
  
  // Internal Business Events
  ORDER_CREATED = 'business:order:created',
  ORDER_PAID = 'business:order:paid',
  ORDER_SHIPPED = 'business:order:shipped',
  ORDER_DELIVERED = 'business:order:delivered',
  
  // User Journey Events
  USER_REGISTERED = 'user:registered',
  USER_UPGRADED = 'user:upgraded',
  USER_CHURNED = 'user:churned',
  
  // Campaign Events
  CAMPAIGN_TRIGGERED = 'campaign:triggered',
  COUPON_REDEEMED = 'coupon:redeemed',
}

export interface BaseLineEvent {
  id: string;
  type: LineEventType;
  timestamp: number;
  source: LineEventSource;
  replyToken?: string;
  metadata: Record<string, any>;
}

export interface LineEventSource {
  type: 'user' | 'group' | 'room';
  userId?: string;
  groupId?: string;
  roomId?: string;
}

export interface LineMessageEvent extends BaseLineEvent {
  type: LineEventType.TEXT_MESSAGE | LineEventType.IMAGE_MESSAGE;
  message: LineMessage;
}

export interface LineMessage {
  id: string;
  type: string;
  text?: string;
  contentProvider?: ContentProvider;
}

export interface ContentProvider {
  type: 'line' | 'external';
  originalContentUrl?: string;
  previewImageUrl?: string;
}
```

### 2.2 Custom Business Events

```typescript
// events/business-events.ts

export interface OrderCreatedEvent {
  eventId: string;
  eventType: 'business:order:created';
  aggregateId: string; // orderId
  aggregateType: 'Order';
  occurredAt: Date;
  version: number;
  payload: {
    orderId: string;
    userId: string;
    lineUserId: string;
    items: OrderItem[];
    totalAmount: number;
    currency: string;
    status: 'pending';
  };
  metadata: {
    correlationId: string;
    causationId: string;
    userId: string;
    ipAddress: string;
  };
}

export interface OrderPaidEvent {
  eventId: string;
  eventType: 'business:order:paid';
  aggregateId: string;
  aggregateType: 'Order';
  occurredAt: Date;
  version: number;
  payload: {
    orderId: string;
    paymentId: string;
    amount: number;
    method: 'line_pay' | 'credit_card' | 'bank_transfer';
    paidAt: Date;
  };
  metadata: {
    correlationId: string;
    causationId: string;
  };
}

export interface UserFollowedEvent {
  eventId: string;
  eventType: 'line:user:followed';
  aggregateId: string; // userId
  aggregateType: 'LineUser';
  occurredAt: Date;
  version: number;
  payload: {
    lineUserId: string;
    followSource: string;
    isNewUser: boolean;
    referralCode?: string;
  };
}
```

### 2.3 Event Schema Design Principles

```typescript
// events/event-schema.ts

/**
 * หลักการออกแบบ Event Schema:
 * 
 * 1. Immutability - Events ไม่ควรถูกแก้ไขหลังจากสร้าง
 * 2. Self-describing - Event มีข้อมูลพอที่จะเข้าใจตัวเอง
 * 3. Versioning - รองรับการ evolve schema
 * 4. Causality - ติดตามว่า event นี้เกิดจากอะไร
 */

export interface DomainEvent<T = any> {
  // Identity
  id: string;
  type: string;
  version: string; // semantic versioning: "1.0.0"
  
  // Timing
  occurredAt: string; // ISO 8601
  recordedAt: string; // ISO 8601
  
  // Source
  source: string; // "line-bot-service"
  
  // Aggregate
  aggregateId: string;
  aggregateType: string;
  aggregateVersion: number; // monotonically increasing
  
  // Causality
  correlationId: string; // ติดตาม request เดิม
  causationId: string;   // event ที่ทำให้เกิด event นี้
  
  // Payload
  data: T;
  
  // Metadata
  metadata: {
    userId?: string;
    sessionId?: string;
    ipAddress?: string;
    userAgent?: string;
    environment: 'development' | 'staging' | 'production';
  };
}

// Factory function
export function createEvent<T>(
  type: string,
  aggregateId: string,
  aggregateType: string,
  data: T,
  options?: Partial<DomainEvent<T>>
): DomainEvent<T> {
  const now = new Date().toISOString();
  return {
    id: generateUUID(),
    type,
    version: '1.0.0',
    occurredAt: now,
    recordedAt: now,
    source: process.env.SERVICE_NAME || 'unknown',
    aggregateId,
    aggregateType,
    aggregateVersion: 1,
    correlationId: options?.correlationId || generateUUID(),
    causationId: options?.causationId || generateUUID(),
    data,
    metadata: {
      environment: (process.env.NODE_ENV as any) || 'development',
      ...options?.metadata,
    },
    ...options,
  };
}

function generateUUID(): string {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, (c) => {
    const r = (Math.random() * 16) | 0;
    const v = c === 'x' ? r : (r & 0x3) | 0x8;
    return v.toString(16);
  });
}
```

---

## 3. Apache Kafka Setup สำหรับ LINE Events

### 3.1 Docker Compose Setup

```yaml
# docker-compose.kafka.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper-data:/var/lib/zookeeper/data
      - zookeeper-logs:/var/lib/zookeeper/log
    networks:
      - kafka-network

  kafka-1:
    image: confluentinc/cp-kafka:7.4.0
    hostname: kafka-1
    container_name: kafka-1
    depends_on:
      - zookeeper
    ports:
      - "9091:9091"
      - "29091:29091"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:29091,PLAINTEXT_HOST://localhost:9091
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'false'
      KAFKA_LOG_RETENTION_HOURS: 168 # 7 days
      KAFKA_LOG_SEGMENT_BYTES: 1073741824 # 1 GB
    volumes:
      - kafka-1-data:/var/lib/kafka/data
    networks:
      - kafka-network

  kafka-2:
    image: confluentinc/cp-kafka:7.4.0
    hostname: kafka-2
    container_name: kafka-2
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-2:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'false'
      KAFKA_LOG_RETENTION_HOURS: 168
      KAFKA_LOG_SEGMENT_BYTES: 1073741824
    volumes:
      - kafka-2-data:/var/lib/kafka/data
    networks:
      - kafka-network

  kafka-3:
    image: confluentinc/cp-kafka:7.4.0
    hostname: kafka-3
    container_name: kafka-3
    depends_on:
      - zookeeper
    ports:
      - "9093:9093"
      - "29093:29093"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-3:29093,PLAINTEXT_HOST://localhost:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 2
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 3
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'false'
      KAFKA_LOG_RETENTION_HOURS: 168
      KAFKA_LOG_SEGMENT_BYTES: 1073741824
    volumes:
      - kafka-3-data:/var/lib/kafka/data
    networks:
      - kafka-network

  schema-registry:
    image: confluentinc/cp-schema-registry:7.4.0
    hostname: schema-registry
    container_name: schema-registry
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: 'kafka-1:29091,kafka-2:29092,kafka-3:29093'
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    networks:
      - kafka-network

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
      - schema-registry
    ports:
      - "8090:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka-1:29091,kafka-2:29092,kafka-3:29093
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
    networks:
      - kafka-network

  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.4.0
    container_name: kafka-connect
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
      - schema-registry
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: 'kafka-1:29091,kafka-2:29092,kafka-3:29093'
      CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect
      CONNECT_GROUP_ID: compose-connect-group
      CONNECT_CONFIG_STORAGE_TOPIC: docker-connect-configs
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_OFFSET_FLUSH_INTERVAL_MS: 10000
      CONNECT_OFFSET_STORAGE_TOPIC: docker-connect-offsets
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_STATUS_STORAGE_TOPIC: docker-connect-status
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 1
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
    networks:
      - kafka-network

volumes:
  zookeeper-data:
  zookeeper-logs:
  kafka-1-data:
  kafka-2-data:
  kafka-3-data:

networks:
  kafka-network:
    driver: bridge
```

### 3.2 Topic Configuration

```bash
#!/bin/bash
# scripts/create-topics.sh

KAFKA_BOOTSTRAP="localhost:9091"

# LINE Bot Events Topics
kafka-topics.sh --create \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --topic line.webhook.events \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config min.insync.replicas=2 \
  --config compression.type=lz4

kafka-topics.sh --create \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --topic line.messages.inbound \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config min.insync.replicas=2

kafka-topics.sh --create \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --topic line.messages.outbound \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000

# Business Events Topics
kafka-topics.sh --create \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --topic business.orders \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=2592000000 \
  --config min.insync.replicas=2

kafka-topics.sh --create \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --topic business.users \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=2592000000

kafka-topics.sh --create \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --topic business.payments \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=2592000000 \
  --config min.insync.replicas=2

# Analytics Topics
kafka-topics.sh --create \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --topic analytics.events \
  --partitions 24 \
  --replication-factor 3 \
  --config retention.ms=86400000

# Dead Letter Queue Topics
kafka-topics.sh --create \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --topic dlq.line.webhook.events \
  --partitions 3 \
  --replication-factor 3 \
  --config retention.ms=2592000000

kafka-topics.sh --create \
  --bootstrap-server $KAFKA_BOOTSTRAP \
  --topic dlq.business.orders \
  --partitions 3 \
  --replication-factor 3 \
  --config retention.ms=2592000000

echo "Topics created successfully!"

# List all topics
kafka-topics.sh --list --bootstrap-server $KAFKA_BOOTSTRAP
```

### 3.3 Kafka Producer Setup (Node.js)

```typescript
// kafka/producer.ts
import { Kafka, Producer, ProducerRecord, RecordMetadata } from 'kafkajs';
import { SchemaRegistry } from '@kafkajs/confluent-schema-registry';
import { logger } from '../utils/logger';
import { DomainEvent } from '../events/event-schema';

export class LineEventProducer {
  private kafka: Kafka;
  private producer: Producer;
  private registry: SchemaRegistry;
  private isConnected: boolean = false;

  constructor() {
    this.kafka = new Kafka({
      clientId: 'line-bot-producer',
      brokers: process.env.KAFKA_BROKERS?.split(',') || ['localhost:9091'],
      ssl: process.env.KAFKA_SSL === 'true',
      sasl: process.env.KAFKA_SASL === 'true' ? {
        mechanism: 'plain',
        username: process.env.KAFKA_USERNAME!,
        password: process.env.KAFKA_PASSWORD!,
      } : undefined,
      retry: {
        initialRetryTime: 100,
        retries: 8,
        multiplier: 2,
        maxRetryTime: 30000,
      },
    });

    this.producer = this.kafka.producer({
      allowAutoTopicCreation: false,
      transactionTimeout: 30000,
      idempotent: true, // Exactly-once semantics
      maxInFlightRequests: 5,
    });

    this.registry = new SchemaRegistry({
      host: process.env.SCHEMA_REGISTRY_URL || 'http://localhost:8081',
    });
  }

  async connect(): Promise<void> {
    if (this.isConnected) return;
    
    await this.producer.connect();
    this.isConnected = true;
    logger.info('Kafka producer connected');
  }

  async disconnect(): Promise<void> {
    if (!this.isConnected) return;
    
    await this.producer.disconnect();
    this.isConnected = false;
    logger.info('Kafka producer disconnected');
  }

  async publishEvent<T>(
    topic: string,
    event: DomainEvent<T>,
    options?: {
      partition?: number;
      key?: string;
      headers?: Record<string, string>;
    }
  ): Promise<RecordMetadata[]> {
    if (!this.isConnected) {
      await this.connect();
    }

    const record: ProducerRecord = {
      topic,
      messages: [
        {
          key: options?.key || event.aggregateId,
          value: JSON.stringify(event),
          headers: {
            'event-type': event.type,
            'event-version': event.version,
            'correlation-id': event.correlationId,
            'source': event.source,
            ...options?.headers,
          },
          timestamp: String(new Date(event.occurredAt).getTime()),
          partition: options?.partition,
        },
      ],
    };

    try {
      const result = await this.producer.send(record);
      
      logger.debug('Event published', {
        topic,
        eventType: event.type,
        eventId: event.id,
        partition: result[0].partition,
        offset: result[0].offset,
      });

      return result;
    } catch (error) {
      logger.error('Failed to publish event', {
        topic,
        eventType: event.type,
        eventId: event.id,
        error,
      });
      throw error;
    }
  }

  async publishBatch<T>(
    topic: string,
    events: DomainEvent<T>[]
  ): Promise<void> {
    if (!this.isConnected) {
      await this.connect();
    }

    const messages = events.map(event => ({
      key: event.aggregateId,
      value: JSON.stringify(event),
      headers: {
        'event-type': event.type,
        'event-version': event.version,
        'correlation-id': event.correlationId,
      },
      timestamp: String(new Date(event.occurredAt).getTime()),
    }));

    await this.producer.send({ topic, messages });
    logger.info(`Batch of ${events.length} events published to ${topic}`);
  }

  // Transactional publishing - Exactly-once semantics
  async publishTransactional<T>(
    events: Array<{ topic: string; event: DomainEvent<T> }>
  ): Promise<void> {
    const transaction = await this.producer.transaction();

    try {
      for (const { topic, event } of events) {
        await transaction.send({
          topic,
          messages: [
            {
              key: event.aggregateId,
              value: JSON.stringify(event),
              headers: {
                'event-type': event.type,
                'correlation-id': event.correlationId,
              },
            },
          ],
        });
      }

      await transaction.commit();
      logger.info(`Transaction committed with ${events.length} events`);
    } catch (error) {
      await transaction.abort();
      logger.error('Transaction aborted', { error });
      throw error;
    }
  }
}

// Singleton instance
export const eventProducer = new LineEventProducer();
```

### 3.4 Webhook Handler with Kafka

```typescript
// webhooks/line-webhook.handler.ts
import { Request, Response } from 'express';
import { WebhookEvent, validateSignature } from '@line/bot-sdk';
import { eventProducer } from '../kafka/producer';
import { createEvent } from '../events/event-schema';
import { logger } from '../utils/logger';

export class LineWebhookHandler {
  private channelSecret: string;

  constructor(channelSecret: string) {
    this.channelSecret = channelSecret;
  }

  async handle(req: Request, res: Response): Promise<void> {
    // Validate LINE signature
    const signature = req.headers['x-line-signature'] as string;
    const body = req.rawBody; // ต้องใช้ raw body

    if (!validateSignature(body, this.channelSecret, signature)) {
      res.status(401).json({ error: 'Invalid signature' });
      return;
    }

    // ตอบกลับ LINE ทันที (ต้องตอบภายใน 200ms)
    res.status(200).json({ status: 'ok' });

    // ประมวลผล events แบบ async
    const events: WebhookEvent[] = req.body.events;
    
    await Promise.allSettled(
      events.map(event => this.publishToKafka(event))
    );
  }

  private async publishToKafka(lineEvent: WebhookEvent): Promise<void> {
    try {
      const topic = this.getTopicForEvent(lineEvent);
      const domainEvent = createEvent(
        `line.${lineEvent.type}`,
        this.getAggregateId(lineEvent),
        'LineUser',
        lineEvent,
        {
          correlationId: lineEvent.webhookEventId || generateId(),
          metadata: {
            environment: process.env.NODE_ENV as any || 'development',
          },
        }
      );

      await eventProducer.publishEvent(topic, domainEvent);

      // Also publish to analytics topic
      await eventProducer.publishEvent('analytics.events', {
        ...domainEvent,
        type: `analytics.${lineEvent.type}`,
      });

    } catch (error) {
      logger.error('Failed to publish LINE event to Kafka', {
        eventType: lineEvent.type,
        error,
      });
    }
  }

  private getTopicForEvent(event: WebhookEvent): string {
    switch (event.type) {
      case 'message':
        return 'line.messages.inbound';
      case 'follow':
      case 'unfollow':
        return 'business.users';
      case 'postback':
        return 'line.webhook.events';
      default:
        return 'line.webhook.events';
    }
  }

  private getAggregateId(event: WebhookEvent): string {
    if (event.source.type === 'user') {
      return event.source.userId || 'unknown';
    }
    if (event.source.type === 'group') {
      return event.source.groupId || 'unknown';
    }
    return 'unknown';
  }
}

function generateId(): string {
  return Date.now().toString(36) + Math.random().toString(36).substr(2);
}
```

---

## 4. Event Sourcing Pattern

### 4.1 Event Store Implementation

```typescript
// event-store/event-store.ts
import { Pool } from 'pg';
import { DomainEvent } from '../events/event-schema';
import { logger } from '../utils/logger';

export interface StoredEvent extends DomainEvent {
  globalSequence: number;
  savedAt: Date;
}

export class PostgresEventStore {
  private pool: Pool;

  constructor(connectionString: string) {
    this.pool = new Pool({ connectionString });
    this.initializeSchema();
  }

  private async initializeSchema(): Promise<void> {
    await this.pool.query(`
      CREATE TABLE IF NOT EXISTS events (
        id UUID PRIMARY KEY,
        global_sequence BIGSERIAL,
        aggregate_id VARCHAR(255) NOT NULL,
        aggregate_type VARCHAR(100) NOT NULL,
        aggregate_version INT NOT NULL,
        event_type VARCHAR(255) NOT NULL,
        event_version VARCHAR(20) NOT NULL,
        occurred_at TIMESTAMPTZ NOT NULL,
        recorded_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        data JSONB NOT NULL,
        metadata JSONB DEFAULT '{}',
        correlation_id UUID,
        causation_id UUID,
        source VARCHAR(100),
        UNIQUE (aggregate_id, aggregate_version)
      );

      CREATE INDEX IF NOT EXISTS idx_events_aggregate 
        ON events (aggregate_id, aggregate_type, aggregate_version);
      
      CREATE INDEX IF NOT EXISTS idx_events_type 
        ON events (event_type);
      
      CREATE INDEX IF NOT EXISTS idx_events_occurred 
        ON events (occurred_at);
      
      CREATE INDEX IF NOT EXISTS idx_events_correlation 
        ON events (correlation_id);
      
      CREATE TABLE IF NOT EXISTS event_snapshots (
        aggregate_id VARCHAR(255) NOT NULL,
        aggregate_type VARCHAR(100) NOT NULL,
        aggregate_version INT NOT NULL,
        state JSONB NOT NULL,
        taken_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        PRIMARY KEY (aggregate_id, aggregate_type)
      );
    `);
  }

  async appendEvent<T>(event: DomainEvent<T>): Promise<StoredEvent> {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');

      // Optimistic concurrency check
      const existing = await client.query(
        `SELECT aggregate_version FROM events 
         WHERE aggregate_id = $1 AND aggregate_type = $2
         ORDER BY aggregate_version DESC LIMIT 1`,
        [event.aggregateId, event.aggregateType]
      );

      const expectedVersion = event.aggregateVersion - 1;
      const currentVersion = existing.rows[0]?.aggregate_version ?? -1;

      if (currentVersion !== expectedVersion) {
        throw new ConcurrencyError(
          `Concurrency conflict for ${event.aggregateType}#${event.aggregateId}. ` +
          `Expected version ${expectedVersion}, but got ${currentVersion}`
        );
      }

      const result = await client.query(
        `INSERT INTO events (
          id, aggregate_id, aggregate_type, aggregate_version,
          event_type, event_version, occurred_at, data, metadata,
          correlation_id, causation_id, source
        ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12)
        RETURNING *`,
        [
          event.id,
          event.aggregateId,
          event.aggregateType,
          event.aggregateVersion,
          event.type,
          event.version,
          event.occurredAt,
          JSON.stringify(event.data),
          JSON.stringify(event.metadata),
          event.correlationId,
          event.causationId,
          event.source,
        ]
      );

      await client.query('COMMIT');

      const row = result.rows[0];
      return {
        ...event,
        globalSequence: row.global_sequence,
        savedAt: row.recorded_at,
      };
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async getEvents(
    aggregateId: string,
    aggregateType: string,
    fromVersion?: number,
    toVersion?: number
  ): Promise<StoredEvent[]> {
    const conditions = [
      'aggregate_id = $1',
      'aggregate_type = $2',
    ];
    const params: any[] = [aggregateId, aggregateType];

    if (fromVersion !== undefined) {
      conditions.push(`aggregate_version >= $${params.length + 1}`);
      params.push(fromVersion);
    }

    if (toVersion !== undefined) {
      conditions.push(`aggregate_version <= $${params.length + 1}`);
      params.push(toVersion);
    }

    const result = await this.pool.query(
      `SELECT * FROM events WHERE ${conditions.join(' AND ')} ORDER BY aggregate_version`,
      params
    );

    return result.rows.map(this.mapRowToEvent);
  }

  async getEventsByType(
    eventType: string,
    limit: number = 100,
    fromGlobalSequence?: number
  ): Promise<StoredEvent[]> {
    const result = await this.pool.query(
      `SELECT * FROM events 
       WHERE event_type = $1 
       ${fromGlobalSequence ? 'AND global_sequence > $2' : ''}
       ORDER BY global_sequence 
       LIMIT ${limit}`,
      fromGlobalSequence ? [eventType, fromGlobalSequence] : [eventType]
    );

    return result.rows.map(this.mapRowToEvent);
  }

  async saveSnapshot(
    aggregateId: string,
    aggregateType: string,
    version: number,
    state: any
  ): Promise<void> {
    await this.pool.query(
      `INSERT INTO event_snapshots (aggregate_id, aggregate_type, aggregate_version, state)
       VALUES ($1, $2, $3, $4)
       ON CONFLICT (aggregate_id, aggregate_type) DO UPDATE
       SET aggregate_version = $3, state = $4, taken_at = NOW()`,
      [aggregateId, aggregateType, version, JSON.stringify(state)]
    );
  }

  async getSnapshot(
    aggregateId: string,
    aggregateType: string
  ): Promise<{ version: number; state: any } | null> {
    const result = await this.pool.query(
      `SELECT * FROM event_snapshots 
       WHERE aggregate_id = $1 AND aggregate_type = $2`,
      [aggregateId, aggregateType]
    );

    if (result.rows.length === 0) return null;

    return {
      version: result.rows[0].aggregate_version,
      state: result.rows[0].state,
    };
  }

  private mapRowToEvent(row: any): StoredEvent {
    return {
      id: row.id,
      type: row.event_type,
      version: row.event_version,
      occurredAt: row.occurred_at,
      recordedAt: row.recorded_at,
      source: row.source,
      aggregateId: row.aggregate_id,
      aggregateType: row.aggregate_type,
      aggregateVersion: row.aggregate_version,
      correlationId: row.correlation_id,
      causationId: row.causation_id,
      data: row.data,
      metadata: row.metadata,
      globalSequence: row.global_sequence,
      savedAt: row.recorded_at,
    };
  }
}

export class ConcurrencyError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'ConcurrencyError';
  }
}
```

### 4.2 Aggregate Root Pattern

```typescript
// aggregates/order.aggregate.ts
import { DomainEvent, createEvent } from '../events/event-schema';
import { ConcurrencyError } from '../event-store/event-store';

interface OrderState {
  id: string;
  userId: string;
  lineUserId: string;
  items: OrderItem[];
  status: OrderStatus;
  totalAmount: number;
  paymentId?: string;
  version: number;
}

interface OrderItem {
  productId: string;
  name: string;
  quantity: number;
  price: number;
}

type OrderStatus = 
  | 'pending'
  | 'confirmed'
  | 'paid'
  | 'preparing'
  | 'shipped'
  | 'delivered'
  | 'cancelled';

export class OrderAggregate {
  private state: OrderState;
  private pendingEvents: DomainEvent[] = [];

  private constructor(state: OrderState) {
    this.state = state;
  }

  // Reconstitute from events
  static fromEvents(events: DomainEvent[]): OrderAggregate {
    if (events.length === 0) {
      throw new Error('Cannot create aggregate from empty events');
    }

    const aggregate = new OrderAggregate({
      id: events[0].aggregateId,
      userId: '',
      lineUserId: '',
      items: [],
      status: 'pending',
      totalAmount: 0,
      version: 0,
    });

    events.forEach(event => aggregate.apply(event, false));
    return aggregate;
  }

  // Create new order
  static create(
    orderId: string,
    userId: string,
    lineUserId: string,
    items: OrderItem[],
    correlationId: string
  ): OrderAggregate {
    const aggregate = new OrderAggregate({
      id: orderId,
      userId,
      lineUserId,
      items: [],
      status: 'pending',
      totalAmount: 0,
      version: 0,
    });

    const totalAmount = items.reduce((sum, item) => sum + item.price * item.quantity, 0);

    const event = createEvent(
      'business:order:created',
      orderId,
      'Order',
      { orderId, userId, lineUserId, items, totalAmount, status: 'pending' },
      { correlationId, aggregateVersion: 1 }
    );

    aggregate.apply(event, true);
    return aggregate;
  }

  // Commands
  confirmOrder(): void {
    if (this.state.status !== 'pending') {
      throw new Error(`Cannot confirm order in ${this.state.status} status`);
    }

    const event = createEvent(
      'business:order:confirmed',
      this.state.id,
      'Order',
      { orderId: this.state.id },
      { aggregateVersion: this.state.version + 1 }
    );

    this.apply(event, true);
  }

  markAsPaid(paymentId: string, amount: number): void {
    if (this.state.status !== 'confirmed') {
      throw new Error(`Cannot mark as paid in ${this.state.status} status`);
    }

    const event = createEvent(
      'business:order:paid',
      this.state.id,
      'Order',
      { orderId: this.state.id, paymentId, amount, paidAt: new Date().toISOString() },
      { aggregateVersion: this.state.version + 1 }
    );

    this.apply(event, true);
  }

  cancel(reason: string): void {
    if (['delivered', 'cancelled'].includes(this.state.status)) {
      throw new Error(`Cannot cancel order in ${this.state.status} status`);
    }

    const event = createEvent(
      'business:order:cancelled',
      this.state.id,
      'Order',
      { orderId: this.state.id, reason, cancelledAt: new Date().toISOString() },
      { aggregateVersion: this.state.version + 1 }
    );

    this.apply(event, true);
  }

  // Apply event to state
  private apply(event: DomainEvent, isNew: boolean): void {
    this.handle(event);
    this.state.version = event.aggregateVersion;

    if (isNew) {
      this.pendingEvents.push(event);
    }
  }

  private handle(event: DomainEvent): void {
    switch (event.type) {
      case 'business:order:created':
        this.handleOrderCreated(event.data);
        break;
      case 'business:order:confirmed':
        this.state.status = 'confirmed';
        break;
      case 'business:order:paid':
        this.state.status = 'paid';
        this.state.paymentId = event.data.paymentId;
        break;
      case 'business:order:shipped':
        this.state.status = 'shipped';
        break;
      case 'business:order:delivered':
        this.state.status = 'delivered';
        break;
      case 'business:order:cancelled':
        this.state.status = 'cancelled';
        break;
    }
  }

  private handleOrderCreated(data: any): void {
    this.state.userId = data.userId;
    this.state.lineUserId = data.lineUserId;
    this.state.items = data.items;
    this.state.totalAmount = data.totalAmount;
    this.state.status = 'pending';
  }

  // Getters
  get id(): string { return this.state.id; }
  get status(): OrderStatus { return this.state.status; }
  get version(): number { return this.state.version; }
  get uncommittedEvents(): DomainEvent[] { return [...this.pendingEvents]; }

  markEventsAsCommitted(): void {
    this.pendingEvents = [];
  }
}
```

---

## 5. CQRS Implementation สำหรับ LINE Bots

### 5.1 Command Side

```typescript
// cqrs/commands/order.commands.ts

export interface CreateOrderCommand {
  commandType: 'CreateOrder';
  orderId: string;
  userId: string;
  lineUserId: string;
  items: Array<{
    productId: string;
    quantity: number;
  }>;
  correlationId: string;
}

export interface CancelOrderCommand {
  commandType: 'CancelOrder';
  orderId: string;
  reason: string;
  userId: string;
  correlationId: string;
}

export interface PayOrderCommand {
  commandType: 'PayOrder';
  orderId: string;
  paymentId: string;
  amount: number;
  correlationId: string;
}

// Command Handler
export class OrderCommandHandler {
  constructor(
    private eventStore: PostgresEventStore,
    private eventProducer: LineEventProducer,
    private productCatalog: ProductCatalogService
  ) {}

  async handle(command: CreateOrderCommand): Promise<void> {
    // Validate products
    const products = await this.productCatalog.getProducts(
      command.items.map(i => i.productId)
    );

    const orderItems = command.items.map(item => {
      const product = products.find(p => p.id === item.productId);
      if (!product) throw new Error(`Product ${item.productId} not found`);
      
      return {
        productId: item.productId,
        name: product.name,
        quantity: item.quantity,
        price: product.price,
      };
    });

    // Create aggregate
    const order = OrderAggregate.create(
      command.orderId,
      command.userId,
      command.lineUserId,
      orderItems,
      command.correlationId
    );

    // Save events
    for (const event of order.uncommittedEvents) {
      await this.eventStore.appendEvent(event);
    }

    // Publish to Kafka
    await this.eventProducer.publishBatch(
      'business.orders',
      order.uncommittedEvents
    );

    order.markEventsAsCommitted();
  }

  async handleCancel(command: CancelOrderCommand): Promise<void> {
    // Load aggregate from event store
    const events = await this.eventStore.getEvents(
      command.orderId,
      'Order'
    );

    const order = OrderAggregate.fromEvents(events);
    order.cancel(command.reason);

    // Save and publish
    for (const event of order.uncommittedEvents) {
      await this.eventStore.appendEvent(event);
      await this.eventProducer.publishEvent('business.orders', event);
    }

    order.markEventsAsCommitted();
  }
}
```

### 5.2 Query Side (Read Models)

```typescript
// cqrs/read-models/order.read-model.ts
import { Pool } from 'pg';
import { DomainEvent } from '../../events/event-schema';

export interface OrderReadModel {
  id: string;
  userId: string;
  lineUserId: string;
  status: string;
  items: any[];
  totalAmount: number;
  paymentId?: string;
  createdAt: Date;
  updatedAt: Date;
  version: number;
}

export class OrderProjector {
  private pool: Pool;

  constructor(connectionString: string) {
    this.pool = new Pool({ connectionString });
    this.initializeSchema();
  }

  private async initializeSchema(): Promise<void> {
    await this.pool.query(`
      CREATE TABLE IF NOT EXISTS orders_view (
        id VARCHAR(255) PRIMARY KEY,
        user_id VARCHAR(255) NOT NULL,
        line_user_id VARCHAR(255) NOT NULL,
        status VARCHAR(50) NOT NULL,
        items JSONB NOT NULL DEFAULT '[]',
        total_amount DECIMAL(10,2) NOT NULL DEFAULT 0,
        payment_id VARCHAR(255),
        created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        version INT NOT NULL DEFAULT 0
      );

      CREATE INDEX IF NOT EXISTS idx_orders_user 
        ON orders_view (line_user_id);
      
      CREATE INDEX IF NOT EXISTS idx_orders_status 
        ON orders_view (status);
    `);
  }

  async project(event: DomainEvent): Promise<void> {
    switch (event.type) {
      case 'business:order:created':
        await this.handleCreated(event);
        break;
      case 'business:order:confirmed':
        await this.handleStatusChange(event.aggregateId, 'confirmed', event.aggregateVersion);
        break;
      case 'business:order:paid':
        await this.handlePaid(event);
        break;
      case 'business:order:shipped':
        await this.handleStatusChange(event.aggregateId, 'shipped', event.aggregateVersion);
        break;
      case 'business:order:delivered':
        await this.handleStatusChange(event.aggregateId, 'delivered', event.aggregateVersion);
        break;
      case 'business:order:cancelled':
        await this.handleStatusChange(event.aggregateId, 'cancelled', event.aggregateVersion);
        break;
    }
  }

  private async handleCreated(event: DomainEvent): Promise<void> {
    const data = event.data;
    await this.pool.query(
      `INSERT INTO orders_view (id, user_id, line_user_id, status, items, total_amount, version)
       VALUES ($1, $2, $3, $4, $5, $6, $7)
       ON CONFLICT (id) DO NOTHING`,
      [
        data.orderId,
        data.userId,
        data.lineUserId,
        data.status,
        JSON.stringify(data.items),
        data.totalAmount,
        event.aggregateVersion,
      ]
    );
  }

  private async handleStatusChange(
    orderId: string,
    status: string,
    version: number
  ): Promise<void> {
    await this.pool.query(
      `UPDATE orders_view 
       SET status = $1, updated_at = NOW(), version = $2
       WHERE id = $3`,
      [status, version, orderId]
    );
  }

  private async handlePaid(event: DomainEvent): Promise<void> {
    await this.pool.query(
      `UPDATE orders_view 
       SET status = 'paid', payment_id = $1, updated_at = NOW(), version = $2
       WHERE id = $3`,
      [event.data.paymentId, event.aggregateVersion, event.aggregateId]
    );
  }
}

// Query Service
export class OrderQueryService {
  constructor(private pool: Pool) {}

  async getOrderById(orderId: string): Promise<OrderReadModel | null> {
    const result = await this.pool.query(
      'SELECT * FROM orders_view WHERE id = $1',
      [orderId]
    );

    return result.rows.length > 0 ? this.mapRow(result.rows[0]) : null;
  }

  async getUserOrders(
    lineUserId: string,
    options?: { status?: string; limit?: number; offset?: number }
  ): Promise<OrderReadModel[]> {
    let query = 'SELECT * FROM orders_view WHERE line_user_id = $1';
    const params: any[] = [lineUserId];

    if (options?.status) {
      params.push(options.status);
      query += ` AND status = $${params.length}`;
    }

    query += ` ORDER BY created_at DESC`;

    if (options?.limit) {
      params.push(options.limit);
      query += ` LIMIT $${params.length}`;
    }

    if (options?.offset) {
      params.push(options.offset);
      query += ` OFFSET $${params.length}`;
    }

    const result = await this.pool.query(query, params);
    return result.rows.map(this.mapRow);
  }

  private mapRow(row: any): OrderReadModel {
    return {
      id: row.id,
      userId: row.user_id,
      lineUserId: row.line_user_id,
      status: row.status,
      items: row.items,
      totalAmount: parseFloat(row.total_amount),
      paymentId: row.payment_id,
      createdAt: row.created_at,
      updatedAt: row.updated_at,
      version: row.version,
    };
  }
}
```

---

## 6. Saga Pattern สำหรับ Order Processing

### 6.1 Order Saga Orchestrator

```typescript
// sagas/order-processing.saga.ts
import { Kafka, Consumer, Producer } from 'kafkajs';
import { logger } from '../utils/logger';

export type SagaState = 
  | 'started'
  | 'inventory_checked'
  | 'payment_initiated'
  | 'payment_confirmed'
  | 'fulfillment_started'
  | 'notification_sent'
  | 'completed'
  | 'compensating'
  | 'failed';

interface SagaData {
  sagaId: string;
  orderId: string;
  userId: string;
  lineUserId: string;
  state: SagaState;
  steps: SagaStep[];
  createdAt: Date;
  updatedAt: Date;
}

interface SagaStep {
  name: string;
  status: 'pending' | 'completed' | 'failed' | 'compensated';
  startedAt?: Date;
  completedAt?: Date;
  error?: string;
}

export class OrderProcessingSaga {
  private kafka: Kafka;
  private consumer: Consumer;
  private producer: Producer;
  private sagaStore: Map<string, SagaData> = new Map(); // ควรใช้ Redis/DB

  constructor() {
    this.kafka = new Kafka({
      clientId: 'order-saga-orchestrator',
      brokers: process.env.KAFKA_BROKERS?.split(',') || ['localhost:9091'],
    });

    this.consumer = this.kafka.consumer({
      groupId: 'order-saga-orchestrator',
    });

    this.producer = this.kafka.producer();
  }

  async start(): Promise<void> {
    await this.consumer.connect();
    await this.producer.connect();

    await this.consumer.subscribe({
      topics: [
        'business.orders',
        'inventory.responses',
        'payment.responses',
        'fulfillment.responses',
        'notification.responses',
      ],
    });

    await this.consumer.run({
      eachMessage: async ({ topic, message }) => {
        const event = JSON.parse(message.value?.toString() || '{}');
        await this.handleEvent(topic, event);
      },
    });

    logger.info('Order Saga Orchestrator started');
  }

  private async handleEvent(topic: string, event: any): Promise<void> {
    switch (topic) {
      case 'business.orders':
        if (event.type === 'business:order:created') {
          await this.startSaga(event);
        }
        break;
      
      case 'inventory.responses':
        await this.handleInventoryResponse(event);
        break;
      
      case 'payment.responses':
        await this.handlePaymentResponse(event);
        break;
      
      case 'fulfillment.responses':
        await this.handleFulfillmentResponse(event);
        break;
      
      case 'notification.responses':
        await this.handleNotificationResponse(event);
        break;
    }
  }

  private async startSaga(orderEvent: any): Promise<void> {
    const sagaData: SagaData = {
      sagaId: generateId(),
      orderId: orderEvent.data.orderId,
      userId: orderEvent.data.userId,
      lineUserId: orderEvent.data.lineUserId,
      state: 'started',
      steps: [
        { name: 'check_inventory', status: 'pending' },
        { name: 'process_payment', status: 'pending' },
        { name: 'fulfill_order', status: 'pending' },
        { name: 'send_notification', status: 'pending' },
      ],
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    this.sagaStore.set(sagaData.sagaId, sagaData);

    // Step 1: Check Inventory
    await this.sendCommand('inventory.commands', {
      commandType: 'CheckInventory',
      sagaId: sagaData.sagaId,
      orderId: orderEvent.data.orderId,
      items: orderEvent.data.items,
    });

    sagaData.steps[0].status = 'pending';
    sagaData.steps[0].startedAt = new Date();
    
    logger.info(`Saga started for order ${orderEvent.data.orderId}`);
  }

  private async handleInventoryResponse(event: any): Promise<void> {
    const saga = this.findSagaByOrderId(event.orderId);
    if (!saga) return;

    if (event.success) {
      saga.steps[0].status = 'completed';
      saga.steps[0].completedAt = new Date();
      saga.state = 'inventory_checked';

      // Step 2: Process Payment
      await this.sendCommand('payment.commands', {
        commandType: 'InitiatePayment',
        sagaId: saga.sagaId,
        orderId: saga.orderId,
        amount: event.totalAmount,
        userId: saga.userId,
      });

      saga.steps[1].startedAt = new Date();
    } else {
      // Compensate
      await this.compensate(saga, 'Inventory check failed: ' + event.reason);
    }
  }

  private async handlePaymentResponse(event: any): Promise<void> {
    const saga = this.findSagaBySagaId(event.sagaId);
    if (!saga) return;

    if (event.success) {
      saga.steps[1].status = 'completed';
      saga.steps[1].completedAt = new Date();
      saga.state = 'payment_confirmed';

      // Step 3: Fulfill Order
      await this.sendCommand('fulfillment.commands', {
        commandType: 'FulfillOrder',
        sagaId: saga.sagaId,
        orderId: saga.orderId,
        paymentId: event.paymentId,
      });

      saga.steps[2].startedAt = new Date();
    } else {
      await this.compensate(saga, 'Payment failed: ' + event.reason);
    }
  }

  private async handleFulfillmentResponse(event: any): Promise<void> {
    const saga = this.findSagaBySagaId(event.sagaId);
    if (!saga) return;

    if (event.success) {
      saga.steps[2].status = 'completed';
      saga.steps[2].completedAt = new Date();
      saga.state = 'fulfillment_started';

      // Step 4: Send Notification
      await this.sendCommand('notification.commands', {
        commandType: 'SendOrderConfirmation',
        sagaId: saga.sagaId,
        orderId: saga.orderId,
        lineUserId: saga.lineUserId,
        trackingNumber: event.trackingNumber,
      });

      saga.steps[3].startedAt = new Date();
    } else {
      await this.compensate(saga, 'Fulfillment failed: ' + event.reason);
    }
  }

  private async handleNotificationResponse(event: any): Promise<void> {
    const saga = this.findSagaBySagaId(event.sagaId);
    if (!saga) return;

    if (event.success) {
      saga.steps[3].status = 'completed';
      saga.steps[3].completedAt = new Date();
      saga.state = 'completed';

      logger.info(`Saga completed for order ${saga.orderId}`);
    } else {
      // Notification failure - non-critical, saga still succeeds
      saga.steps[3].status = 'failed';
      saga.steps[3].error = event.reason;
      saga.state = 'completed'; // Order is still complete
      
      logger.warn(`Notification failed for order ${saga.orderId}, but saga completed`);
    }
  }

  private async compensate(saga: SagaData, reason: string): Promise<void> {
    saga.state = 'compensating';
    logger.warn(`Starting compensation for saga ${saga.sagaId}: ${reason}`);

    // Compensate completed steps in reverse order
    for (let i = saga.steps.length - 1; i >= 0; i--) {
      if (saga.steps[i].status === 'completed') {
        await this.compensateStep(saga, saga.steps[i].name);
        saga.steps[i].status = 'compensated';
      }
    }

    saga.state = 'failed';

    // Notify user about failure
    await this.sendCommand('notification.commands', {
      commandType: 'SendOrderFailed',
      orderId: saga.orderId,
      lineUserId: saga.lineUserId,
      reason,
    });

    logger.error(`Saga failed for order ${saga.orderId}: ${reason}`);
  }

  private async compensateStep(saga: SagaData, stepName: string): Promise<void> {
    switch (stepName) {
      case 'check_inventory':
        await this.sendCommand('inventory.commands', {
          commandType: 'ReleaseInventory',
          orderId: saga.orderId,
        });
        break;
      case 'process_payment':
        await this.sendCommand('payment.commands', {
          commandType: 'RefundPayment',
          orderId: saga.orderId,
        });
        break;
      case 'fulfill_order':
        await this.sendCommand('fulfillment.commands', {
          commandType: 'CancelFulfillment',
          orderId: saga.orderId,
        });
        break;
    }
  }

  private async sendCommand(topic: string, command: any): Promise<void> {
    await this.producer.send({
      topic,
      messages: [{
        key: command.orderId || command.sagaId,
        value: JSON.stringify(command),
        headers: {
          'command-type': command.commandType,
        },
      }],
    });
  }

  private findSagaByOrderId(orderId: string): SagaData | undefined {
    for (const saga of this.sagaStore.values()) {
      if (saga.orderId === orderId) return saga;
    }
    return undefined;
  }

  private findSagaBySagaId(sagaId: string): SagaData | undefined {
    return this.sagaStore.get(sagaId);
  }
}

function generateId(): string {
  return Date.now().toString(36) + Math.random().toString(36).substr(2);
}
```

---

## 7. Event Replay สำหรับ Debugging

### 7.1 Event Replay Service

```typescript
// replay/event-replay.service.ts
import { PostgresEventStore } from '../event-store/event-store';
import { OrderProjector } from '../cqrs/read-models/order.read-model';
import { logger } from '../utils/logger';

export interface ReplayOptions {
  fromGlobalSequence?: number;
  toGlobalSequence?: number;
  fromDate?: Date;
  toDate?: Date;
  eventTypes?: string[];
  aggregateType?: string;
  aggregateId?: string;
  batchSize?: number;
  dryRun?: boolean;
}

export class EventReplayService {
  constructor(
    private eventStore: PostgresEventStore,
    private projectors: Map<string, any>
  ) {}

  async replay(options: ReplayOptions = {}): Promise<ReplayStats> {
    const stats: ReplayStats = {
      totalEvents: 0,
      processedEvents: 0,
      skippedEvents: 0,
      failedEvents: 0,
      duration: 0,
      errors: [],
    };

    const startTime = Date.now();
    logger.info('Starting event replay', options);

    if (!options.dryRun) {
      await this.clearProjections(options);
    }

    let hasMore = true;
    let lastSequence = options.fromGlobalSequence || 0;
    const batchSize = options.batchSize || 100;

    while (hasMore) {
      const events = await this.fetchEventBatch(
        lastSequence,
        batchSize,
        options
      );

      if (events.length === 0) {
        hasMore = false;
        break;
      }

      for (const event of events) {
        stats.totalEvents++;
        
        try {
          if (!options.dryRun) {
            await this.replayEvent(event);
          }
          stats.processedEvents++;
        } catch (error: any) {
          stats.failedEvents++;
          stats.errors.push({
            eventId: event.id,
            eventType: event.type,
            error: error.message,
          });
          logger.error('Failed to replay event', { event, error });
        }

        lastSequence = event.globalSequence;
      }

      if (options.toGlobalSequence && lastSequence >= options.toGlobalSequence) {
        hasMore = false;
      }

      logger.info(`Replayed ${stats.processedEvents} events so far...`);
    }

    stats.duration = Date.now() - startTime;
    logger.info('Event replay completed', stats);
    return stats;
  }

  private async fetchEventBatch(
    fromSequence: number,
    batchSize: number,
    options: ReplayOptions
  ): Promise<any[]> {
    // Implementation depends on your event store
    return this.eventStore.getEventsBySequence(
      fromSequence,
      batchSize,
      options
    );
  }

  private async replayEvent(event: any): Promise<void> {
    const projector = this.projectors.get(event.aggregateType);
    if (projector) {
      await projector.project(event);
    }
  }

  private async clearProjections(options: ReplayOptions): Promise<void> {
    // Clear read model tables before replay
    logger.info('Clearing projections before replay');
    // Implementation specific to your read models
  }
}

interface ReplayStats {
  totalEvents: number;
  processedEvents: number;
  skippedEvents: number;
  failedEvents: number;
  duration: number;
  errors: Array<{
    eventId: string;
    eventType: string;
    error: string;
  }>;
}

// CLI Tool สำหรับ replay
// scripts/replay-events.ts
async function main() {
  const args = process.argv.slice(2);
  const fromDate = args[0] ? new Date(args[0]) : undefined;
  const toDate = args[1] ? new Date(args[1]) : undefined;
  const dryRun = args.includes('--dry-run');

  const eventStore = new PostgresEventStore(process.env.DATABASE_URL!);
  const projectors = new Map([
    ['Order', new OrderProjector(process.env.DATABASE_URL!)],
  ]);

  const replayService = new EventReplayService(eventStore, projectors);

  const stats = await replayService.replay({
    fromDate,
    toDate,
    dryRun,
    batchSize: 500,
  });

  console.log('Replay Stats:', JSON.stringify(stats, null, 2));
  process.exit(stats.failedEvents > 0 ? 1 : 0);
}

main().catch(console.error);
```

---

## 8. Dead Letter Events

### 8.1 Dead Letter Queue Handler

```typescript
// dlq/dead-letter-handler.ts
import { Kafka, Consumer, Producer } from 'kafkajs';
import { logger } from '../utils/logger';

interface DeadLetterRecord {
  originalTopic: string;
  originalPartition: number;
  originalOffset: string;
  originalTimestamp: string;
  originalKey?: string;
  originalValue: string;
  originalHeaders: Record<string, string>;
  errorMessage: string;
  errorStack?: string;
  retryCount: number;
  firstFailedAt: string;
  lastFailedAt: string;
}

export class DeadLetterHandler {
  private kafka: Kafka;
  private consumer: Consumer;
  private producer: Producer;
  private maxRetries: number;
  private retryDelays: number[]; // milliseconds

  constructor(
    maxRetries: number = 3,
    retryDelays: number[] = [1000, 5000, 30000]
  ) {
    this.kafka = new Kafka({
      clientId: 'dlq-handler',
      brokers: process.env.KAFKA_BROKERS?.split(',') || ['localhost:9091'],
    });

    this.consumer = this.kafka.consumer({
      groupId: 'dlq-handler',
    });

    this.producer = this.kafka.producer();
    this.maxRetries = maxRetries;
    this.retryDelays = retryDelays;
  }

  async start(): Promise<void> {
    await this.consumer.connect();
    await this.producer.connect();

    await this.consumer.subscribe({
      topics: [
        'dlq.line.webhook.events',
        'dlq.business.orders',
      ],
    });

    await this.consumer.run({
      eachMessage: async ({ topic, partition, message }) => {
        const record: DeadLetterRecord = JSON.parse(
          message.value?.toString() || '{}'
        );

        await this.handleDeadLetter(record);
      },
    });

    logger.info('Dead Letter Handler started');
  }

  private async handleDeadLetter(record: DeadLetterRecord): Promise<void> {
    logger.warn('Processing dead letter', {
      originalTopic: record.originalTopic,
      retryCount: record.retryCount,
      error: record.errorMessage,
    });

    if (record.retryCount < this.maxRetries) {
      await this.scheduleRetry(record);
    } else {
      await this.handleMaxRetriesExceeded(record);
    }
  }

  private async scheduleRetry(record: DeadLetterRecord): Promise<void> {
    const delay = this.retryDelays[record.retryCount] || 60000;
    
    // Schedule retry after delay
    setTimeout(async () => {
      try {
        await this.producer.send({
          topic: record.originalTopic,
          messages: [{
            key: record.originalKey,
            value: record.originalValue,
            headers: {
              ...record.originalHeaders,
              'retry-count': String(record.retryCount + 1),
              'original-timestamp': record.originalTimestamp,
            },
          }],
        });

        logger.info(`Retried message from DLQ`, {
          topic: record.originalTopic,
          retryCount: record.retryCount + 1,
        });
      } catch (error) {
        logger.error('Failed to retry message', { record, error });
      }
    }, delay);
  }

  private async handleMaxRetriesExceeded(record: DeadLetterRecord): Promise<void> {
    // Store in permanent error storage
    await this.storeInPermanentErrorLog(record);

    // Send alert
    await this.sendAlert(record);

    logger.error('Message exceeded max retries, stored in permanent error log', {
      originalTopic: record.originalTopic,
      errorMessage: record.errorMessage,
    });
  }

  private async storeInPermanentErrorLog(record: DeadLetterRecord): Promise<void> {
    // Store in database or S3 for manual review
    // Implementation depends on your storage preference
    logger.info('Storing in permanent error log', {
      topic: record.originalTopic,
      firstFailedAt: record.firstFailedAt,
    });
  }

  private async sendAlert(record: DeadLetterRecord): Promise<void> {
    // Send alert via LINE, Slack, PagerDuty, etc.
    logger.warn('ALERT: Dead letter exceeded max retries', {
      originalTopic: record.originalTopic,
      errorMessage: record.errorMessage,
      retryCount: record.retryCount,
    });
  }
}

// Middleware สำหรับส่ง events ที่ fail ไปยัง DLQ
export class DeadLetterMiddleware {
  constructor(
    private producer: Producer,
    private maxRetries: number = 3
  ) {}

  async withDeadLetter<T>(
    fn: () => Promise<T>,
    context: {
      topic: string;
      partition: number;
      offset: string;
      key?: string;
      value: string;
      headers: Record<string, string>;
      timestamp: string;
    }
  ): Promise<T | null> {
    try {
      return await fn();
    } catch (error: any) {
      const retryCount = parseInt(context.headers['retry-count'] || '0');
      
      if (retryCount >= this.maxRetries) {
        await this.sendToDeadLetter(context, error, retryCount);
        return null;
      }

      throw error; // Let Kafka retry
    }
  }

  private async sendToDeadLetter(
    context: any,
    error: Error,
    retryCount: number
  ): Promise<void> {
    const dlqTopic = `dlq.${context.topic}`;
    const record: DeadLetterRecord = {
      originalTopic: context.topic,
      originalPartition: context.partition,
      originalOffset: context.offset,
      originalTimestamp: context.timestamp,
      originalKey: context.key,
      originalValue: context.value,
      originalHeaders: context.headers,
      errorMessage: error.message,
      errorStack: error.stack,
      retryCount,
      firstFailedAt: context.headers['first-failed-at'] || new Date().toISOString(),
      lastFailedAt: new Date().toISOString(),
    };

    await this.producer.send({
      topic: dlqTopic,
      messages: [{
        key: context.key,
        value: JSON.stringify(record),
      }],
    });

    logger.warn(`Message sent to DLQ: ${dlqTopic}`, {
      retryCount,
      error: error.message,
    });
  }
}
```

---

## 9. Schema Registry กับ Avro

### 9.1 Avro Schema Definitions

```json
// schemas/line-webhook-event.avsc
{
  "type": "record",
  "name": "LineWebhookEvent",
  "namespace": "com.linebot.events",
  "doc": "LINE Webhook Event schema",
  "fields": [
    {
      "name": "id",
      "type": "string",
      "doc": "Unique event ID"
    },
    {
      "name": "type",
      "type": "string",
      "doc": "Event type"
    },
    {
      "name": "version",
      "type": "string",
      "default": "1.0.0"
    },
    {
      "name": "occurredAt",
      "type": "string",
      "doc": "ISO 8601 timestamp"
    },
    {
      "name": "source",
      "type": "string"
    },
    {
      "name": "aggregateId",
      "type": "string"
    },
    {
      "name": "aggregateType",
      "type": "string"
    },
    {
      "name": "aggregateVersion",
      "type": "int"
    },
    {
      "name": "correlationId",
      "type": "string"
    },
    {
      "name": "causationId",
      "type": "string"
    },
    {
      "name": "data",
      "type": {
        "type": "record",
        "name": "LineEventData",
        "fields": [
          { "name": "lineEventType", "type": "string" },
          { "name": "lineUserId", "type": ["null", "string"], "default": null },
          { "name": "groupId", "type": ["null", "string"], "default": null },
          { "name": "messageText", "type": ["null", "string"], "default": null },
          { "name": "replyToken", "type": ["null", "string"], "default": null },
          { "name": "rawEvent", "type": "string", "doc": "JSON string of original event" }
        ]
      }
    },
    {
      "name": "metadata",
      "type": {
        "type": "record",
        "name": "EventMetadata",
        "fields": [
          { "name": "environment", "type": "string" },
          { "name": "userId", "type": ["null", "string"], "default": null },
          { "name": "sessionId", "type": ["null", "string"], "default": null }
        ]
      }
    }
  ]
}
```

```typescript
// schemas/schema-registry.ts
import { SchemaRegistry, SchemaType } from '@kafkajs/confluent-schema-registry';
import * as fs from 'fs';
import * as path from 'path';

export class LineSchemaRegistry {
  private registry: SchemaRegistry;
  private schemaCache: Map<string, number> = new Map();

  constructor() {
    this.registry = new SchemaRegistry({
      host: process.env.SCHEMA_REGISTRY_URL || 'http://localhost:8081',
    });
  }

  async registerSchema(subject: string, schemaPath: string): Promise<number> {
    const schemaString = fs.readFileSync(
      path.join(__dirname, 'avro', schemaPath),
      'utf-8'
    );

    const { id } = await this.registry.register(
      {
        type: SchemaType.AVRO,
        schema: schemaString,
      },
      {
        subject,
        compatibility: 'BACKWARD',
      }
    );

    this.schemaCache.set(subject, id);
    return id;
  }

  async encode<T>(subject: string, data: T): Promise<Buffer> {
    let schemaId = this.schemaCache.get(subject);
    
    if (!schemaId) {
      schemaId = await this.registry.getLatestSchemaId(subject);
      this.schemaCache.set(subject, schemaId);
    }

    return this.registry.encode(schemaId, data);
  }

  async decode<T>(buffer: Buffer): Promise<T> {
    return this.registry.decode(buffer);
  }

  async setupSchemas(): Promise<void> {
    const schemas = [
      { subject: 'line.webhook.events-value', file: 'line-webhook-event.avsc' },
      { subject: 'business.orders-value', file: 'business-order-event.avsc' },
      { subject: 'business.users-value', file: 'business-user-event.avsc' },
    ];

    for (const schema of schemas) {
      const id = await this.registerSchema(schema.subject, schema.file);
      console.log(`Registered schema ${schema.subject} with ID ${id}`);
    }
  }
}

export const schemaRegistry = new LineSchemaRegistry();
```

---

## 10. Consumer Groups Strategy

### 10.1 Consumer Group Design

```
Consumer Groups สำหรับ LINE Bot System:

line.webhook.events topic (12 partitions):
├── cg-message-handler     (4 consumers)  → Handle reply messages
├── cg-analytics-collector (2 consumers)  → Collect analytics data
├── cg-crm-updater         (2 consumers)  → Update CRM
└── cg-ai-processor        (4 consumers)  → AI/NLP processing

business.orders topic (6 partitions):
├── cg-order-projector     (2 consumers)  → Update read models
├── cg-order-notifier      (2 consumers)  → Send LINE notifications
└── cg-order-analytics     (2 consumers)  → Analytics

analytics.events topic (24 partitions):
├── cg-realtime-dashboard  (6 consumers)  → Real-time dashboard
└── cg-data-warehouse      (6 consumers)  → Data warehouse sync
```

```typescript
// consumers/message-handler.consumer.ts
import { Kafka, Consumer, EachMessagePayload } from 'kafkajs';
import { LineClient } from '@line/bot-sdk';
import { logger } from '../utils/logger';
import { DeadLetterMiddleware } from '../dlq/dead-letter-handler';

export class MessageHandlerConsumer {
  private kafka: Kafka;
  private consumer: Consumer;
  private lineClient: LineClient;
  private dlqMiddleware: DeadLetterMiddleware;
  private isRunning: boolean = false;

  constructor(lineClient: LineClient) {
    this.kafka = new Kafka({
      clientId: `message-handler-${process.env.POD_NAME || 'local'}`,
      brokers: process.env.KAFKA_BROKERS?.split(',') || ['localhost:9091'],
    });

    this.consumer = this.kafka.consumer({
      groupId: 'cg-message-handler',
      sessionTimeout: 30000,
      heartbeatInterval: 3000,
      maxBytesPerPartition: 1048576, // 1 MB
      minBytes: 1,
      maxWaitTimeInMs: 5000,
    });

    this.lineClient = lineClient;
    this.dlqMiddleware = new DeadLetterMiddleware(
      this.kafka.producer()
    );
  }

  async start(): Promise<void> {
    await this.consumer.connect();
    
    await this.consumer.subscribe({
      topic: 'line.messages.inbound',
      fromBeginning: false,
    });

    // Handle graceful shutdown
    process.on('SIGTERM', () => this.stop());
    process.on('SIGINT', () => this.stop());

    await this.consumer.run({
      partitionsConsumedConcurrently: 3,
      eachMessage: async (payload) => {
        await this.processMessage(payload);
      },
    });

    this.isRunning = true;
    logger.info('Message Handler Consumer started');
  }

  private async processMessage(payload: EachMessagePayload): Promise<void> {
    const { topic, partition, message } = payload;
    
    const eventData = JSON.parse(message.value?.toString() || '{}');
    
    await this.dlqMiddleware.withDeadLetter(
      async () => {
        await this.handleLineEvent(eventData);
      },
      {
        topic,
        partition,
        offset: message.offset,
        key: message.key?.toString(),
        value: message.value?.toString() || '',
        headers: Object.fromEntries(
          Object.entries(message.headers || {}).map(([k, v]) => [k, v?.toString() || ''])
        ),
        timestamp: message.timestamp,
      }
    );
  }

  private async handleLineEvent(event: any): Promise<void> {
    const lineEvent = event.data;

    switch (lineEvent.type) {
      case 'message':
        await this.handleMessageEvent(lineEvent);
        break;
      case 'follow':
        await this.handleFollowEvent(lineEvent);
        break;
      case 'postback':
        await this.handlePostbackEvent(lineEvent);
        break;
      default:
        logger.debug(`Unhandled event type: ${lineEvent.type}`);
    }
  }

  private async handleMessageEvent(event: any): Promise<void> {
    if (event.message.type !== 'text') return;

    const userId = event.source.userId;
    const text = event.message.text;

    // ประมวลผล message
    const response = await this.processUserMessage(userId, text);

    // ส่ง reply
    await this.lineClient.replyMessage(event.replyToken, {
      type: 'text',
      text: response,
    });
  }

  private async processUserMessage(userId: string, text: string): Promise<string> {
    // ตัวอย่างการประมวลผล
    const lowerText = text.toLowerCase();
    
    if (lowerText.includes('สั่งซื้อ') || lowerText.includes('order')) {
      return 'กรุณากดที่เมนูด้านล่างเพื่อเริ่มสั่งซื้อสินค้า 🛍️';
    }
    
    if (lowerText.includes('สถานะ') || lowerText.includes('status')) {
      return 'กรุณาพิมพ์หมายเลขคำสั่งซื้อของคุณ';
    }
    
    return 'ขอบคุณสำหรับข้อความค่ะ ทีมงานจะติดต่อกลับโดยเร็ว 😊';
  }

  private async handleFollowEvent(event: any): Promise<void> {
    const userId = event.source.userId;
    
    await this.lineClient.pushMessage(userId, {
      type: 'text',
      text: 'ยินดีต้อนรับสู่ร้านของเรา! 🎉\n\nพิมพ์ "สั่งซื้อ" เพื่อเริ่มสั่งซื้อสินค้า',
    });
  }

  private async handlePostbackEvent(event: any): Promise<void> {
    const data = new URLSearchParams(event.postback.data);
    const action = data.get('action');

    switch (action) {
      case 'view_order':
        await this.showOrderDetails(event.replyToken, data.get('orderId') || '');
        break;
      case 'cancel_order':
        await this.confirmCancelOrder(event.replyToken, data.get('orderId') || '');
        break;
    }
  }

  private async showOrderDetails(replyToken: string, orderId: string): Promise<void> {
    await this.lineClient.replyMessage(replyToken, {
      type: 'text',
      text: `รายละเอียดคำสั่งซื้อ #${orderId}\nกำลังโหลดข้อมูล...`,
    });
  }

  private async confirmCancelOrder(replyToken: string, orderId: string): Promise<void> {
    await this.lineClient.replyMessage(replyToken, {
      type: 'template',
      altText: 'ยืนยันการยกเลิกคำสั่งซื้อ',
      template: {
        type: 'confirm',
        text: `ต้องการยกเลิกคำสั่งซื้อ #${orderId} ใช่หรือไม่?`,
        actions: [
          {
            type: 'postback',
            label: 'ยืนยัน',
            data: `action=confirm_cancel&orderId=${orderId}`,
          },
          {
            type: 'postback',
            label: 'ยกเลิก',
            data: `action=keep_order&orderId=${orderId}`,
          },
        ],
      },
    });
  }

  async stop(): Promise<void> {
    this.isRunning = false;
    await this.consumer.disconnect();
    logger.info('Message Handler Consumer stopped');
  }
}
```

---

## 11. Python Kafka Consumer Examples

### 11.1 Analytics Consumer ด้วย Python

```python
# consumers/analytics_consumer.py
from confluent_kafka import Consumer, KafkaError, KafkaException
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroDeserializer
import json
import logging
import signal
import sys
from datetime import datetime
from typing import Optional, Dict, Any
import psycopg2
from psycopg2.extras import execute_batch

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)


class AnalyticsConsumer:
    """
    Consumer สำหรับประมวลผล analytics events จาก LINE Bot
    """
    
    def __init__(
        self,
        bootstrap_servers: str,
        schema_registry_url: str,
        database_url: str,
        group_id: str = 'cg-analytics-collector'
    ):
        self.bootstrap_servers = bootstrap_servers
        self.group_id = group_id
        self.running = False
        
        # Kafka Consumer config
        self.consumer_config = {
            'bootstrap.servers': bootstrap_servers,
            'group.id': group_id,
            'auto.offset.reset': 'latest',
            'enable.auto.commit': False,  # Manual commit
            'max.poll.interval.ms': 300000,
            'session.timeout.ms': 45000,
            'heartbeat.interval.ms': 3000,
            'fetch.min.bytes': 1,
            'fetch.max.wait.ms': 500,
        }
        
        # Schema Registry
        schema_registry_conf = {'url': schema_registry_url}
        schema_registry_client = SchemaRegistryClient(schema_registry_conf)
        self.deserializer = AvroDeserializer(schema_registry_client)
        
        # Database
        self.db_conn = psycopg2.connect(database_url)
        self.db_cursor = self.db_conn.cursor()
        self._setup_db()
        
        # Consumer instance
        self.consumer = Consumer(self.consumer_config)
        
        # Batch processing
        self.event_batch: list = []
        self.batch_size = 100
        self.batch_timeout = 5.0  # seconds
        self.last_flush_time = datetime.now()
        
        # Signal handlers
        signal.signal(signal.SIGTERM, self._handle_signal)
        signal.signal(signal.SIGINT, self._handle_signal)
    
    def _setup_db(self):
        """สร้างตาราง analytics"""
        self.db_cursor.execute("""
            CREATE TABLE IF NOT EXISTS line_analytics (
                id SERIAL PRIMARY KEY,
                event_id VARCHAR(255) UNIQUE NOT NULL,
                event_type VARCHAR(255) NOT NULL,
                line_user_id VARCHAR(255),
                session_id VARCHAR(255),
                aggregate_id VARCHAR(255),
                occurred_at TIMESTAMPTZ NOT NULL,
                data JSONB,
                created_at TIMESTAMPTZ DEFAULT NOW()
            );
            
            CREATE INDEX IF NOT EXISTS idx_analytics_user 
                ON line_analytics (line_user_id);
            CREATE INDEX IF NOT EXISTS idx_analytics_type 
                ON line_analytics (event_type);
            CREATE INDEX IF NOT EXISTS idx_analytics_occurred 
                ON line_analytics (occurred_at);
            
            -- Daily summary table
            CREATE TABLE IF NOT EXISTS analytics_daily_summary (
                date DATE NOT NULL,
                metric_name VARCHAR(255) NOT NULL,
                metric_value BIGINT NOT NULL DEFAULT 0,
                created_at TIMESTAMPTZ DEFAULT NOW(),
                updated_at TIMESTAMPTZ DEFAULT NOW(),
                PRIMARY KEY (date, metric_name)
            );
        """)
        self.db_conn.commit()
        logger.info("Database schema initialized")
    
    def start(self, topics: list[str]):
        """เริ่ม consume messages"""
        self.consumer.subscribe(topics)
        self.running = True
        
        logger.info(f"Analytics consumer started, subscribing to: {topics}")
        
        try:
            while self.running:
                msg = self.consumer.poll(timeout=1.0)
                
                if msg is None:
                    self._flush_if_needed()
                    continue
                
                if msg.error():
                    if msg.error().code() == KafkaError._PARTITION_EOF:
                        logger.debug(f"End of partition: {msg.topic()} [{msg.partition()}]")
                    elif msg.error().code() == KafkaError.UNKNOWN_TOPIC_OR_PART:
                        logger.error(f"Topic not found: {msg.topic()}")
                    else:
                        raise KafkaException(msg.error())
                    continue
                
                self._process_message(msg)
                
        except Exception as e:
            logger.error(f"Consumer error: {e}", exc_info=True)
        finally:
            self._flush_batch()
            self.consumer.close()
            self.db_conn.close()
            logger.info("Analytics consumer stopped")
    
    def _process_message(self, msg):
        """ประมวลผลแต่ละ message"""
        try:
            # Deserialize
            event_data = json.loads(msg.value().decode('utf-8'))
            
            # เพิ่มเข้า batch
            self.event_batch.append({
                'event': event_data,
                'msg': msg,
            })
            
            # Flush ถ้า batch เต็ม
            if len(self.event_batch) >= self.batch_size:
                self._flush_batch()
            else:
                self._flush_if_needed()
                
        except json.JSONDecodeError as e:
            logger.error(f"Failed to decode message: {e}")
            self._send_to_dlq(msg, str(e))
    
    def _flush_if_needed(self):
        """Flush ถ้าเวลาผ่านไปนานพอ"""
        elapsed = (datetime.now() - self.last_flush_time).total_seconds()
        if elapsed >= self.batch_timeout and self.event_batch:
            self._flush_batch()
    
    def _flush_batch(self):
        """บันทึก batch ลง database"""
        if not self.event_batch:
            return
        
        try:
            records = []
            for item in self.event_batch:
                event = item['event']
                records.append((
                    event.get('id'),
                    event.get('type'),
                    event.get('metadata', {}).get('userId'),
                    event.get('metadata', {}).get('sessionId'),
                    event.get('aggregateId'),
                    event.get('occurredAt'),
                    json.dumps(event.get('data', {})),
                ))
            
            execute_batch(
                self.db_cursor,
                """
                INSERT INTO line_analytics 
                    (event_id, event_type, line_user_id, session_id, 
                     aggregate_id, occurred_at, data)
                VALUES (%s, %s, %s, %s, %s, %s, %s)
                ON CONFLICT (event_id) DO NOTHING
                """,
                records,
                page_size=100
            )
            
            self.db_conn.commit()
            
            # Commit Kafka offsets
            self.consumer.commit(asynchronous=False)
            
            # Update metrics
            self._update_daily_metrics()
            
            logger.info(f"Flushed batch of {len(self.event_batch)} events")
            
        except Exception as e:
            logger.error(f"Failed to flush batch: {e}", exc_info=True)
            self.db_conn.rollback()
        finally:
            self.event_batch.clear()
            self.last_flush_time = datetime.now()
    
    def _update_daily_metrics(self):
        """อัพเดท daily metrics"""
        try:
            today = datetime.now().date()
            
            # นับจำนวน messages ต่อวัน
            self.db_cursor.execute("""
                INSERT INTO analytics_daily_summary (date, metric_name, metric_value)
                SELECT 
                    %s::date,
                    event_type,
                    COUNT(*)
                FROM line_analytics
                WHERE occurred_at::date = %s::date
                GROUP BY event_type
                ON CONFLICT (date, metric_name) DO UPDATE
                SET metric_value = EXCLUDED.metric_value,
                    updated_at = NOW()
            """, (today, today))
            
            self.db_conn.commit()
        except Exception as e:
            logger.warning(f"Failed to update daily metrics: {e}")
            self.db_conn.rollback()
    
    def _send_to_dlq(self, msg, error: str):
        """ส่ง failed message ไปยัง DLQ"""
        logger.warning(f"Sending message to DLQ: {error}")
        # Implementation: produce to dlq.analytics.events topic
    
    def _handle_signal(self, signum, frame):
        """Handle shutdown signals"""
        logger.info(f"Received signal {signum}, stopping...")
        self.running = False


class UserBehaviorAnalyzer:
    """
    วิเคราะห์พฤติกรรมผู้ใช้จาก events
    """
    
    def __init__(self, db_conn):
        self.db_conn = db_conn
    
    def get_user_journey(self, line_user_id: str, days: int = 30) -> Dict[str, Any]:
        """ดึง user journey ย้อนหลัง N วัน"""
        cursor = self.db_conn.cursor()
        
        cursor.execute("""
            SELECT 
                event_type,
                occurred_at,
                data
            FROM line_analytics
            WHERE 
                line_user_id = %s
                AND occurred_at >= NOW() - INTERVAL '%s days'
            ORDER BY occurred_at ASC
        """, (line_user_id, days))
        
        events = cursor.fetchall()
        
        return {
            'userId': line_user_id,
            'period': f'last_{days}_days',
            'totalEvents': len(events),
            'events': [
                {
                    'type': e[0],
                    'occurredAt': e[1].isoformat(),
                    'data': e[2],
                }
                for e in events
            ],
        }
    
    def get_engagement_metrics(self, line_user_id: str) -> Dict[str, Any]:
        """คำนวณ engagement metrics สำหรับผู้ใช้"""
        cursor = self.db_conn.cursor()
        
        cursor.execute("""
            WITH user_events AS (
                SELECT 
                    DATE(occurred_at) as event_date,
                    event_type,
                    COUNT(*) as count
                FROM line_analytics
                WHERE 
                    line_user_id = %s
                    AND occurred_at >= NOW() - INTERVAL '30 days'
                GROUP BY DATE(occurred_at), event_type
            )
            SELECT 
                COUNT(DISTINCT event_date) as active_days,
                SUM(count) FILTER (WHERE event_type LIKE 'message:%%') as messages_sent,
                SUM(count) FILTER (WHERE event_type = 'postback') as postbacks,
                MIN(event_date) as first_event_date,
                MAX(event_date) as last_event_date
            FROM user_events
        """, (line_user_id,))
        
        row = cursor.fetchone()
        
        if not row:
            return {'userId': line_user_id, 'noData': True}
        
        return {
            'userId': line_user_id,
            'activeDays': row[0],
            'messagesSent': row[1] or 0,
            'postbacks': row[2] or 0,
            'firstEventDate': row[3].isoformat() if row[3] else None,
            'lastEventDate': row[4].isoformat() if row[4] else None,
            'engagementScore': self._calculate_engagement_score(row),
        }
    
    def _calculate_engagement_score(self, metrics) -> float:
        """คำนวณ engagement score (0-100)"""
        active_days = metrics[0] or 0
        messages = metrics[1] or 0
        postbacks = metrics[2] or 0
        
        # Simple scoring formula
        score = min(100, (
            (active_days / 30 * 40) +  # 40% from active days
            (min(messages, 50) / 50 * 40) +  # 40% from messages
            (min(postbacks, 20) / 20 * 20)  # 20% from postbacks
        ))
        
        return round(score, 2)


# Main execution
if __name__ == '__main__':
    import os
    
    consumer = AnalyticsConsumer(
        bootstrap_servers=os.getenv('KAFKA_BROKERS', 'localhost:9091'),
        schema_registry_url=os.getenv('SCHEMA_REGISTRY_URL', 'http://localhost:8081'),
        database_url=os.getenv('DATABASE_URL', 'postgresql://user:pass@localhost:5432/linebot'),
        group_id='cg-analytics-collector',
    )
    
    consumer.start([
        'analytics.events',
        'line.webhook.events',
    ])
```

### 11.2 Python CRM Consumer

```python
# consumers/crm_consumer.py
from confluent_kafka import Consumer, Producer
import json
import logging
import os
import requests
from datetime import datetime
from typing import Optional

logger = logging.getLogger(__name__)


class CRMUpdateConsumer:
    """
    Consumer สำหรับอัพเดท CRM เมื่อมี LINE events
    """
    
    def __init__(
        self,
        bootstrap_servers: str,
        crm_base_url: str,
        crm_api_key: str,
    ):
        self.consumer = Consumer({
            'bootstrap.servers': bootstrap_servers,
            'group.id': 'cg-crm-updater',
            'auto.offset.reset': 'latest',
            'enable.auto.commit': True,
            'auto.commit.interval.ms': 5000,
        })
        
        self.crm_base_url = crm_base_url
        self.crm_headers = {
            'Authorization': f'Bearer {crm_api_key}',
            'Content-Type': 'application/json',
        }
        self.session = requests.Session()
        self.session.headers.update(self.crm_headers)
    
    def start(self):
        """เริ่ม consume และอัพเดท CRM"""
        self.consumer.subscribe(['business.users', 'business.orders'])
        logger.info("CRM Consumer started")
        
        while True:
            msg = self.consumer.poll(1.0)
            
            if msg is None:
                continue
            
            if msg.error():
                logger.error(f"Kafka error: {msg.error()}")
                continue
            
            try:
                event = json.loads(msg.value().decode('utf-8'))
                self._handle_event(event)
            except Exception as e:
                logger.error(f"Error processing event: {e}", exc_info=True)
    
    def _handle_event(self, event: dict):
        """route events ไปยัง handler ที่เหมาะสม"""
        event_type = event.get('type')
        
        handlers = {
            'line:user:followed': self._handle_new_follower,
            'line:user:unfollowed': self._handle_unfollow,
            'business:order:created': self._handle_order_created,
            'business:order:paid': self._handle_order_paid,
            'business:order:delivered': self._handle_order_delivered,
        }
        
        handler = handlers.get(event_type)
        if handler:
            handler(event)
        else:
            logger.debug(f"No handler for event type: {event_type}")
    
    def _handle_new_follower(self, event: dict):
        """สร้างหรืออัพเดท contact ใน CRM"""
        line_user_id = event['data'].get('lineUserId')
        if not line_user_id:
            return
        
        # ดึง LINE profile
        profile = self._get_line_profile(line_user_id)
        
        # สร้าง/อัพเดท contact ใน CRM
        contact_data = {
            'lineUserId': line_user_id,
            'displayName': profile.get('displayName'),
            'pictureUrl': profile.get('pictureUrl'),
            'followedAt': event['occurredAt'],
            'status': 'active',
            'source': 'LINE',
        }
        
        try:
            response = self.session.post(
                f'{self.crm_base_url}/contacts',
                json=contact_data,
            )
            response.raise_for_status()
            logger.info(f"Created CRM contact for {line_user_id}")
        except requests.RequestException as e:
            logger.error(f"Failed to create CRM contact: {e}")
    
    def _handle_unfollow(self, event: dict):
        """อัพเดทสถานะ contact ใน CRM"""
        line_user_id = event['data'].get('lineUserId')
        if not line_user_id:
            return
        
        try:
            response = self.session.patch(
                f'{self.crm_base_url}/contacts/line/{line_user_id}',
                json={
                    'status': 'unfollowed',
                    'unfollowedAt': event['occurredAt'],
                }
            )
            response.raise_for_status()
        except requests.RequestException as e:
            logger.error(f"Failed to update CRM contact status: {e}")
    
    def _handle_order_created(self, event: dict):
        """สร้าง deal ใน CRM"""
        order_data = event['data']
        
        crm_deal = {
            'title': f"คำสั่งซื้อ #{order_data['orderId']}",
            'lineUserId': order_data['lineUserId'],
            'amount': order_data['totalAmount'],
            'status': 'new',
            'items': order_data['items'],
            'createdAt': event['occurredAt'],
            'source': 'LINE Bot',
        }
        
        try:
            response = self.session.post(
                f'{self.crm_base_url}/deals',
                json=crm_deal
            )
            response.raise_for_status()
            logger.info(f"Created CRM deal for order {order_data['orderId']}")
        except requests.RequestException as e:
            logger.error(f"Failed to create CRM deal: {e}")
    
    def _handle_order_paid(self, event: dict):
        """อัพเดทสถานะ deal ใน CRM"""
        order_id = event['aggregateId']
        
        try:
            response = self.session.patch(
                f'{self.crm_base_url}/deals/order/{order_id}',
                json={
                    'status': 'won',
                    'paidAt': event['occurredAt'],
                    'paymentId': event['data'].get('paymentId'),
                }
            )
            response.raise_for_status()
        except requests.RequestException as e:
            logger.error(f"Failed to update CRM deal status: {e}")
    
    def _handle_order_delivered(self, event: dict):
        """บันทึกการส่งสำเร็จใน CRM"""
        order_id = event['aggregateId']
        
        try:
            response = self.session.patch(
                f'{self.crm_base_url}/deals/order/{order_id}',
                json={
                    'status': 'delivered',
                    'deliveredAt': event['occurredAt'],
                }
            )
            response.raise_for_status()
        except requests.RequestException as e:
            logger.error(f"Failed to update CRM delivery status: {e}")
    
    def _get_line_profile(self, user_id: str) -> dict:
        """ดึง LINE user profile"""
        # ควร cache profile นี้
        try:
            response = requests.get(
                f'https://api.line.me/v2/bot/profile/{user_id}',
                headers={
                    'Authorization': f'Bearer {os.getenv("LINE_CHANNEL_ACCESS_TOKEN")}'
                }
            )
            return response.json()
        except Exception:
            return {}
```

---

## 12. Complete System Integration

### 12.1 Main Application Entry Point

```typescript
// src/main.ts
import express from 'express';
import { LineWebhookHandler } from './webhooks/line-webhook.handler';
import { MessageHandlerConsumer } from './consumers/message-handler.consumer';
import { OrderCommandHandler } from './cqrs/commands/order.commands';
import { OrderProjector, OrderQueryService } from './cqrs/read-models/order.read-model';
import { LineEventProducer } from './kafka/producer';
import { PostgresEventStore } from './event-store/event-store';
import { OrderProcessingSaga } from './sagas/order-processing.saga';
import { DeadLetterHandler } from './dlq/dead-letter-handler';
import { EventReplayService } from './replay/event-replay.service';
import { LineClient } from '@line/bot-sdk';
import { logger } from './utils/logger';

async function bootstrap() {
  const app = express();
  
  // Middleware
  app.use(express.json({
    verify: (req: any, res, buf) => {
      req.rawBody = buf;
    },
  }));

  // Initialize services
  const lineClient = new LineClient({
    channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN!,
    channelSecret: process.env.LINE_CHANNEL_SECRET!,
  });

  const eventStore = new PostgresEventStore(process.env.DATABASE_URL!);
  const eventProducer = new LineEventProducer();
  await eventProducer.connect();

  const orderProjector = new OrderProjector(process.env.DATABASE_URL!);
  const orderQueryService = new OrderQueryService(
    new (require('pg').Pool)({ connectionString: process.env.DATABASE_URL })
  );

  // Webhook Handler
  const webhookHandler = new LineWebhookHandler(
    process.env.LINE_CHANNEL_SECRET!
  );

  // Routes
  app.post('/webhook', (req, res) => webhookHandler.handle(req, res));

  app.get('/health', (req, res) => {
    res.json({ status: 'ok', timestamp: new Date().toISOString() });
  });

  app.get('/orders/:orderId', async (req, res) => {
    try {
      const order = await orderQueryService.getOrderById(req.params.orderId);
      if (!order) {
        return res.status(404).json({ error: 'Order not found' });
      }
      res.json(order);
    } catch (error) {
      res.status(500).json({ error: 'Internal server error' });
    }
  });

  // Start consumers
  const messageConsumer = new MessageHandlerConsumer(lineClient);
  messageConsumer.start().catch(err => {
    logger.error('Message consumer failed', err);
  });

  // Start saga orchestrator
  const orderSaga = new OrderProcessingSaga();
  orderSaga.start().catch(err => {
    logger.error('Order saga failed', err);
  });

  // Start DLQ handler
  const dlqHandler = new DeadLetterHandler();
  dlqHandler.start().catch(err => {
    logger.error('DLQ handler failed', err);
  });

  // Start HTTP server
  const PORT = process.env.PORT || 3000;
  app.listen(PORT, () => {
    logger.info(`LINE Bot Event-Driven service started on port ${PORT}`);
  });

  // Graceful shutdown
  process.on('SIGTERM', async () => {
    logger.info('Shutting down gracefully...');
    await eventProducer.disconnect();
    process.exit(0);
  });
}

bootstrap().catch(err => {
  logger.error('Failed to start application', err);
  process.exit(1);
});
```

### 12.2 Monitoring และ Metrics

```typescript
// monitoring/kafka-metrics.ts
import { Kafka } from 'kafkajs';
import { Registry, Gauge, Counter, Histogram } from 'prom-client';

export class KafkaMetricsCollector {
  private registry: Registry;
  private kafka: Kafka;
  private admin: any;

  // Metrics
  private consumerLag: Gauge;
  private messagesProduced: Counter;
  private messagesConsumed: Counter;
  private processingDuration: Histogram;
  private dlqMessages: Counter;

  constructor() {
    this.registry = new Registry();
    this.kafka = new Kafka({
      clientId: 'metrics-collector',
      brokers: process.env.KAFKA_BROKERS?.split(',') || ['localhost:9091'],
    });
    this.admin = this.kafka.admin();

    this.consumerLag = new Gauge({
      name: 'kafka_consumer_group_lag',
      help: 'Consumer group lag',
      labelNames: ['group', 'topic', 'partition'],
      registers: [this.registry],
    });

    this.messagesProduced = new Counter({
      name: 'kafka_messages_produced_total',
      help: 'Total messages produced',
      labelNames: ['topic', 'event_type'],
      registers: [this.registry],
    });

    this.messagesConsumed = new Counter({
      name: 'kafka_messages_consumed_total',
      help: 'Total messages consumed',
      labelNames: ['topic', 'group', 'event_type'],
      registers: [this.registry],
    });

    this.processingDuration = new Histogram({
      name: 'kafka_message_processing_duration_seconds',
      help: 'Message processing duration',
      labelNames: ['topic', 'event_type', 'status'],
      buckets: [0.1, 0.5, 1, 2, 5, 10],
      registers: [this.registry],
    });

    this.dlqMessages = new Counter({
      name: 'kafka_dlq_messages_total',
      help: 'Total messages sent to DLQ',
      labelNames: ['original_topic', 'reason'],
      registers: [this.registry],
    });
  }

  async collectLagMetrics(): Promise<void> {
    try {
      await this.admin.connect();
      
      const groups = await this.admin.listGroups();
      
      for (const group of groups.groups) {
        const offsets = await this.admin.fetchOffsets({
          groupId: group.groupId,
          topics: ['line.webhook.events', 'business.orders'],
        });

        for (const topicOffset of offsets) {
          for (const partition of topicOffset.partitions) {
            const topicInfo = await this.admin.fetchTopicOffsets(topicOffset.topic);
            const latest = topicInfo.find(
              (t: any) => t.partition === partition.partition
            );
            
            if (latest) {
              const lag = parseInt(latest.offset) - parseInt(partition.offset);
              
              this.consumerLag.set(
                {
                  group: group.groupId,
                  topic: topicOffset.topic,
                  partition: partition.partition.toString(),
                },
                lag
              );
            }
          }
        }
      }
    } finally {
      await this.admin.disconnect();
    }
  }

  getMetrics(): string {
    return this.registry.metrics() as unknown as string;
  }
}
```

---

## 13. Best Practices และ สรุป

### 13.1 Checklist สำหรับ Event-Driven LINE Bot

```
✅ Event Design
□ Events มี unique ID
□ Events เป็น immutable
□ Events มี version สำหรับ schema evolution
□ Events มี correlation ID และ causation ID
□ Events ใช้ past tense ในชื่อ (OrderCreated ไม่ใช่ CreateOrder)

✅ Kafka Configuration
□ Topics มี replication factor >= 3
□ Topics มี min.insync.replicas >= 2
□ Producers ใช้ idempotent mode
□ Consumers ใช้ manual commit
□ DLQ topics ถูกตั้งค่าสำหรับทุก topic หลัก
□ Retention policy เหมาะสมกับ use case

✅ Event Store
□ Optimistic concurrency control
□ Snapshot สำหรับ aggregates ที่มี events เยอะ
□ Event replay ใช้งานได้
□ Schema Registry ลงทะเบียน schemas แล้ว

✅ Consumer Groups
□ Consumer group IDs ชัดเจนและ descriptive
□ แต่ละ business concern มี consumer group ของตัวเอง
□ Consumer groups มีการ monitor lag
□ Backpressure handling ทำงานถูกต้อง

✅ Sagas
□ Saga state บันทึกใน durable storage
□ Compensation transactions ถูก implement
□ Saga timeout handling
□ Dead saga detection

✅ Monitoring
□ Consumer lag monitoring
□ Error rate alerts
□ DLQ monitoring
□ Processing latency tracking
```

### 13.2 Performance Tips

```
1. Partitioning Strategy:
   - ใช้ userId เป็น key สำหรับ message events (รับประกัน ordering ต่อ user)
   - ใช้ orderId เป็น key สำหรับ order events
   - Partition count ควรเป็น 2x จำนวน consumers

2. Batch Processing:
   - ใช้ batch size 100-500 records
   - Flush ทุก 1-5 วินาที
   - ใช้ execute_batch สำหรับ database writes

3. Compression:
   - ใช้ LZ4 สำหรับ real-time events (เร็วที่สุด)
   - ใช้ GZIP สำหรับ analytics events (compress ดีกว่า)

4. Consumer Configuration:
   - max.poll.records = 500
   - fetch.min.bytes = 1024
   - fetch.max.wait.ms = 500
```

### 13.3 การ Debug Event-Driven System

```bash
# ดู consumer lag
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9091 \
  --describe \
  --group cg-message-handler

# Replay events จาก topic
kafka-console-consumer.sh \
  --bootstrap-server localhost:9091 \
  --topic line.webhook.events \
  --from-beginning \
  --max-messages 100

# ดู events ใน DLQ
kafka-console-consumer.sh \
  --bootstrap-server localhost:9091 \
  --topic dlq.line.webhook.events \
  --from-beginning

# ค้นหา event specific ด้วย correlation ID
kafka-console-consumer.sh \
  --bootstrap-server localhost:9091 \
  --topic business.orders \
  --from-beginning \
  --property print.headers=true | grep "correlation-id: your-correlation-id"
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Event-Driven Architecture** สำหรับ LINE Bot ที่ช่วยให้รับมือกับ traffic สูงได้
2. **Apache Kafka** setup พร้อม replication และ Schema Registry
3. **Event Sourcing** ด้วย PostgreSQL Event Store
4. **CQRS Pattern** แยก Command และ Query side
5. **Saga Pattern** สำหรับ distributed transactions
6. **Event Replay** สำหรับ debugging และ rebuilding
7. **Dead Letter Queue** สำหรับจัดการ failed events
8. **Consumer Groups** strategy ที่เหมาะสม
9. **Python consumers** สำหรับ analytics และ CRM

ระบบ Event-Driven เหมาะที่สุดสำหรับ LINE Bot ที่มีผู้ใช้มากกว่า 100,000 คน หรือต้องการ integration กับหลาย services พร้อมกัน
