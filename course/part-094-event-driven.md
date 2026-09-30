# Part 94: Event-Driven Architecture สำหรับ LINE Bot

## บทนำ

Event-driven architecture (EDA) คือรูปแบบการออกแบบที่ components ติดต่อกันผ่าน events แทนการเรียกกันโดยตรง บทนี้ครอบคลุมการนำ Apache Kafka มาใช้กับระบบ LINE Bot พร้อม Event Sourcing, CQRS, และ Saga Pattern

---

## 1. Event-Driven Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                 EVENT-DRIVEN LINE BOT ARCHITECTURE                    │
│                                                                        │
│  ┌──────────────────────────────────────────────────────────────┐    │
│  │                     Event Sources                             │    │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │    │
│  │  │  LINE    │  │  User    │  │  Timer   │  │  External  │  │    │
│  │  │  Webhook │  │  Actions │  │  Events  │  │  Systems   │  │    │
│  │  └────┬─────┘  └────┬─────┘  └────┬─────┘  └─────┬──────┘  │    │
│  └───────┼─────────────┼─────────────┼───────────────┼──────────┘    │
│          └─────────────┴─────────────┴───────────────┘               │
│                                  │                                     │
│  ┌───────────────────────────────▼──────────────────────────────────┐ │
│  │                    Apache Kafka                                    │ │
│  │  ┌─────────────────────────────────────────────────────────────┐ │ │
│  │  │  Topics                                                       │ │ │
│  │  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────────────┐│ │ │
│  │  │  │ line.events │  │user.actions │  │ notifications.send  ││ │ │
│  │  │  │ line.msgs   │  │user.profile │  │ analytics.raw       ││ │ │
│  │  │  └─────────────┘  └─────────────┘  └──────────────────────┘│ │ │
│  │  └─────────────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────┘ │
│                                  │                                     │
│  ┌───────────────────────────────▼──────────────────────────────────┐ │
│  │                    Event Consumers                                 │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────┐  │ │
│  │  │ Message  │  │Analytics │  │Notification│ │  Audit Logger  │  │ │
│  │  │ Handler  │  │Processor │  │  Service  │  │               │  │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └────────────────┘  │ │
│  └───────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 2. Event Types ใน LINE Ecosystem

### 2.1 LINE Event Type Definitions

```typescript
// events/LineEventTypes.ts

// Base event interface
interface BaseLineEvent {
  id: string;
  timestamp: Date;
  tenantId: string;
  source: {
    type: 'user' | 'group' | 'room';
    userId?: string;
    groupId?: string;
    roomId?: string;
  };
}

// Message Events
export interface TextMessageEvent extends BaseLineEvent {
  type: 'message';
  messageType: 'text';
  replyToken: string;
  message: {
    id: string;
    text: string;
    emojis?: Array<{
      index: number;
      productId: string;
      emojiId: string;
    }>;
    mention?: {
      mentionees: Array<{
        index: number;
        length: number;
        userId?: string;
        type: 'user' | 'all';
      }>;
    };
  };
}

export interface ImageMessageEvent extends BaseLineEvent {
  type: 'message';
  messageType: 'image';
  replyToken: string;
  message: {
    id: string;
    contentProvider: {
      type: 'line' | 'external';
      originalContentUrl?: string;
      previewImageUrl?: string;
    };
    imageSet?: {
      id: string;
      index: number;
      total: number;
    };
  };
}

export interface StickerMessageEvent extends BaseLineEvent {
  type: 'message';
  messageType: 'sticker';
  replyToken: string;
  message: {
    id: string;
    packageId: string;
    stickerId: string;
    stickerResourceType: 'STATIC' | 'ANIMATION' | 'SOUND' | 'ANIMATION_SOUND' | 'POPUP' | 'POPUP_SOUND' | 'CUSTOM' | 'MESSAGE';
    keywords?: string[];
  };
}

// Postback Event
export interface PostbackEvent extends BaseLineEvent {
  type: 'postback';
  replyToken: string;
  postback: {
    data: string;
    params?: {
      date?: string;
      time?: string;
      datetime?: string;
      newRichMenuAliasId?: string;
      status?: string;
    };
  };
}

// Follow/Unfollow Events
export interface FollowEvent extends BaseLineEvent {
  type: 'follow';
  replyToken: string;
}

export interface UnfollowEvent extends BaseLineEvent {
  type: 'unfollow';
}

// Join/Leave Events
export interface JoinEvent extends BaseLineEvent {
  type: 'join';
  replyToken: string;
}

export interface LeaveEvent extends BaseLineEvent {
  type: 'leave';
}

// Video/Audio Events
export interface VideoPlayCompleteEvent extends BaseLineEvent {
  type: 'videoPlayComplete';
  replyToken: string;
  videoPlayComplete: {
    trackingId: string;
  };
}

// Beacon Events
export interface BeaconEvent extends BaseLineEvent {
  type: 'beacon';
  replyToken: string;
  beacon: {
    hwid: string;
    type: 'enter' | 'leave' | 'banner';
    dm?: string;
  };
}

// Account Link Event
export interface AccountLinkEvent extends BaseLineEvent {
  type: 'accountLink';
  replyToken: string;
  link: {
    result: 'ok' | 'failed';
    nonce: string;
  };
}

// Member Join/Leave Events
export interface MemberJoinedEvent extends BaseLineEvent {
  type: 'memberJoined';
  replyToken: string;
  joined: {
    members: Array<{
      type: 'user' | 'bot';
      userId?: string;
    }>;
  };
}

// Union type of all events
export type LineEvent = 
  | TextMessageEvent
  | ImageMessageEvent
  | StickerMessageEvent
  | PostbackEvent
  | FollowEvent
  | UnfollowEvent
  | JoinEvent
  | LeaveEvent
  | VideoPlayCompleteEvent
  | BeaconEvent
  | AccountLinkEvent
  | MemberJoinedEvent;
```

---

## 3. Apache Kafka Setup

### 3.1 Kafka Docker Compose

```yaml
# docker-compose.kafka.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    container_name: zookeeper
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOOKEEPER_SYNC_LIMIT: 2
    volumes:
      - zookeeper_data:/var/lib/zookeeper/data
      - zookeeper_log:/var/lib/zookeeper/log
    networks:
      - kafka-net

  kafka-1:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka-1
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:9092,PLAINTEXT_HOST://localhost:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_NUM_PARTITIONS: 6
      KAFKA_LOG_RETENTION_HOURS: 168
      KAFKA_LOG_SEGMENT_BYTES: 1073741824
      KAFKA_LOG_RETENTION_CHECK_INTERVAL_MS: 300000
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: 'false'
      KAFKA_JMX_PORT: 9101
      KAFKA_JMX_HOSTNAME: kafka-1
    volumes:
      - kafka1_data:/var/lib/kafka/data
    ports:
      - "9093:9093"
      - "9101:9101"
    networks:
      - kafka-net

  kafka-2:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka-2
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-2:9092,PLAINTEXT_HOST://localhost:9094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
    volumes:
      - kafka2_data:/var/lib/kafka/data
    ports:
      - "9094:9094"
    networks:
      - kafka-net

  kafka-3:
    image: confluentinc/cp-kafka:7.5.0
    container_name: kafka-3
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-3:9092,PLAINTEXT_HOST://localhost:9095
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
    volumes:
      - kafka3_data:/var/lib/kafka/data
    ports:
      - "9095:9095"
    networks:
      - kafka-net

  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    container_name: schema-registry
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka-1:9092,kafka-2:9092,kafka-3:9092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    ports:
      - "8081:8081"
    networks:
      - kafka-net

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    depends_on:
      - kafka-1
      - schema-registry
    environment:
      KAFKA_CLUSTERS_0_NAME: linebot-cluster
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka-1:9092,kafka-2:9092,kafka-3:9092
      KAFKA_CLUSTERS_0_SCHEMAREGISTRY: http://schema-registry:8081
    ports:
      - "8080:8080"
    networks:
      - kafka-net

  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.5.0
    container_name: kafka-connect
    depends_on:
      - kafka-1
      - schema-registry
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka-1:9092,kafka-2:9092,kafka-3:9092
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: linebot-connect
      CONNECT_CONFIG_STORAGE_TOPIC: connect-configs
      CONNECT_OFFSET_STORAGE_TOPIC: connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: connect-status
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_KEY_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
    ports:
      - "8083:8083"
    networks:
      - kafka-net

volumes:
  zookeeper_data:
  zookeeper_log:
  kafka1_data:
  kafka2_data:
  kafka3_data:

networks:
  kafka-net:
    driver: bridge
```

### 3.2 Topic Setup Script

```bash
#!/bin/bash
# scripts/setup-kafka-topics.sh

KAFKA_BROKER="kafka-1:9092"

# สร้าง topics สำหรับ LINE events
echo "Creating LINE event topics..."

# Main LINE events topic
kafka-topics.sh --create --bootstrap-server $KAFKA_BROKER \
  --topic line.events \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=604800000 \
  --config cleanup.policy=delete \
  --config min.insync.replicas=2 \
  --config compression.type=snappy

# Message events
kafka-topics.sh --create --bootstrap-server $KAFKA_BROKER \
  --topic line.messages \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=2592000000 \
  --config min.insync.replicas=2

# User events
kafka-topics.sh --create --bootstrap-server $KAFKA_BROKER \
  --topic user.events \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=2592000000

# Notification events
kafka-topics.sh --create --bootstrap-server $KAFKA_BROKER \
  --topic notifications.outbound \
  --partitions 12 \
  --replication-factor 3 \
  --config retention.ms=86400000

# Analytics events
kafka-topics.sh --create --bootstrap-server $KAFKA_BROKER \
  --topic analytics.raw \
  --partitions 24 \
  --replication-factor 3 \
  --config retention.ms=7776000000 \
  --config compression.type=snappy

# Dead Letter Queue
kafka-topics.sh --create --bootstrap-server $KAFKA_BROKER \
  --topic dlq.line.events \
  --partitions 3 \
  --replication-factor 3 \
  --config retention.ms=2592000000

# Saga events
kafka-topics.sh --create --bootstrap-server $KAFKA_BROKER \
  --topic saga.orchestration \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=604800000

echo "All topics created successfully"
kafka-topics.sh --list --bootstrap-server $KAFKA_BROKER
```

---

## 4. Kafka Producer

### 4.1 LINE Event Producer

```typescript
// kafka/LineEventProducer.ts
import { Kafka, Producer, ProducerRecord, CompressionTypes } from 'kafkajs';
import { SchemaRegistry } from '@kafkajs/confluent-schema-registry';
import { LineEvent } from '../events/LineEventTypes';

export class LineEventProducer {
  private producer: Producer;
  private schemaRegistry: SchemaRegistry;
  private isConnected: boolean = false;

  constructor() {
    const kafka = new Kafka({
      clientId: 'line-webhook-service',
      brokers: process.env.KAFKA_BROKERS!.split(','),
      ssl: process.env.KAFKA_SSL === 'true',
      sasl: process.env.KAFKA_SASL === 'true' ? {
        mechanism: 'scram-sha-256',
        username: process.env.KAFKA_USERNAME!,
        password: process.env.KAFKA_PASSWORD!
      } : undefined,
      connectionTimeout: 3000,
      retry: {
        initialRetryTime: 100,
        retries: 8
      }
    });

    this.producer = kafka.producer({
      maxInFlightRequests: 5,
      idempotent: true,
      transactionalId: `webhook-producer-${process.env.POD_NAME || 'default'}`,
      transactionTimeout: 30000,
    });

    this.schemaRegistry = new SchemaRegistry({
      host: process.env.SCHEMA_REGISTRY_URL!
    });
  }

  async connect(): Promise<void> {
    if (!this.isConnected) {
      await this.producer.connect();
      this.isConnected = true;
    }
  }

  async publishLineEvents(events: LineEvent[], tenantId: string): Promise<void> {
    await this.connect();

    // Group events by type for better partition distribution
    const messages = events.map(event => ({
      key: Buffer.from(event.source.userId || event.source.groupId || tenantId),
      value: Buffer.from(JSON.stringify({
        ...event,
        tenantId,
        publishedAt: new Date().toISOString()
      })),
      headers: {
        'event-type': event.type,
        'tenant-id': tenantId,
        'source-id': event.source.userId || event.source.groupId || 'unknown',
        'content-type': 'application/json'
      },
      timestamp: String(Date.now())
    }));

    const record: ProducerRecord = {
      topic: 'line.events',
      messages,
      compression: CompressionTypes.SNAPPY
    };

    try {
      const result = await this.producer.send(record);
      
      // Log partition/offset for tracing
      result.forEach(({ partition, offset, baseOffset }) => {
        console.log(`Published ${events.length} events: partition=${partition}, offset=${offset || baseOffset}`);
      });
    } catch (error) {
      console.error('Failed to publish LINE events:', error);
      // Retry logic handled by kafkajs
      throw error;
    }
  }

  async publishWithTransaction(events: LineEvent[], tenantId: string): Promise<void> {
    await this.connect();

    const transaction = await this.producer.transaction();

    try {
      const messages = events.map(event => ({
        key: Buffer.from(event.source.userId || tenantId),
        value: Buffer.from(JSON.stringify(event)),
        headers: {
          'event-type': event.type,
          'tenant-id': tenantId
        }
      }));

      await transaction.send({
        topic: 'line.events',
        messages,
        compression: CompressionTypes.SNAPPY
      });

      // ส่ง user events แยก
      const followEvents = events.filter(e => e.type === 'follow' || e.type === 'unfollow');
      if (followEvents.length > 0) {
        await transaction.send({
          topic: 'user.events',
          messages: followEvents.map(event => ({
            key: Buffer.from(event.source.userId || 'unknown'),
            value: Buffer.from(JSON.stringify(event)),
            headers: { 'event-type': event.type, 'tenant-id': tenantId }
          }))
        });
      }

      await transaction.commit();
    } catch (error) {
      await transaction.abort();
      throw error;
    }
  }

  async publishBatch(events: Array<{ event: LineEvent; tenantId: string }>): Promise<void> {
    await this.connect();

    // Group by topic for efficient batching
    const messagesByTopic: Record<string, any[]> = {
      'line.events': [],
      'user.events': [],
      'analytics.raw': []
    };

    for (const { event, tenantId } of events) {
      const message = {
        key: Buffer.from(event.source.userId || tenantId),
        value: Buffer.from(JSON.stringify(event)),
        headers: {
          'event-type': event.type,
          'tenant-id': tenantId
        }
      };

      messagesByTopic['line.events'].push(message);

      if (['follow', 'unfollow', 'memberJoined', 'memberLeft'].includes(event.type)) {
        messagesByTopic['user.events'].push(message);
      }

      // ส่งทุก event ไป analytics
      messagesByTopic['analytics.raw'].push({
        key: Buffer.from(tenantId),
        value: Buffer.from(JSON.stringify({
          event_type: event.type,
          tenant_id: tenantId,
          user_id: event.source.userId,
          timestamp: event.timestamp
        }))
      });
    }

    const topicMessages = Object.entries(messagesByTopic)
      .filter(([, msgs]) => msgs.length > 0)
      .map(([topic, messages]) => ({ topic, messages }));

    await this.producer.sendBatch({ topicMessages });
  }

  async disconnect(): Promise<void> {
    if (this.isConnected) {
      await this.producer.disconnect();
      this.isConnected = false;
    }
  }
}
```

---

## 5. Event Sourcing Pattern

### 5.1 Event Store Implementation

```typescript
// eventsourcing/EventStore.ts
import { Pool } from 'pg';
import { Redis } from 'ioredis';
import { EventEmitter } from 'events';

interface DomainEvent {
  id: string;
  aggregateId: string;
  aggregateType: string;
  tenantId: string;
  type: string;
  payload: any;
  metadata: {
    userId?: string;
    correlationId?: string;
    causationId?: string;
    timestamp: Date;
    version: number;
  };
}

interface EventStoreQuery {
  aggregateId?: string;
  aggregateType?: string;
  fromVersion?: number;
  toVersion?: number;
  fromTimestamp?: Date;
  toTimestamp?: Date;
  types?: string[];
}

export class EventStore extends EventEmitter {
  private pool: Pool;
  private redis: Redis;

  constructor(pool: Pool) {
    super();
    this.pool = pool;
    this.redis = new Redis(process.env.REDIS_URL!);
  }

  async append(event: DomainEvent): Promise<void> {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');

      // ตรวจสอบ optimistic concurrency
      if (event.metadata.version > 0) {
        const result = await client.query(
          `SELECT MAX(version) as current_version 
           FROM event_store 
           WHERE aggregate_id = $1 AND tenant_id = $2`,
          [event.aggregateId, event.tenantId]
        );
        
        const currentVersion = result.rows[0]?.current_version || 0;
        if (currentVersion !== event.metadata.version - 1) {
          throw new Error(`Optimistic concurrency conflict. Expected version ${event.metadata.version - 1}, got ${currentVersion}`);
        }
      }

      // เก็บ event
      await client.query(`
        INSERT INTO event_store (
          id, aggregate_id, aggregate_type, tenant_id,
          event_type, payload, metadata, version, created_at
        ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, NOW())
      `, [
        event.id,
        event.aggregateId,
        event.aggregateType,
        event.tenantId,
        event.type,
        JSON.stringify(event.payload),
        JSON.stringify(event.metadata),
        event.metadata.version
      ]);

      await client.query('COMMIT');

      // Emit event สำหรับ projections
      this.emit('eventAppended', event);
      
      // Publish ไป Redis สำหรับ real-time subscriptions
      await this.redis.publish(
        `events:${event.tenantId}:${event.aggregateType}`,
        JSON.stringify(event)
      );

    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async appendMany(events: DomainEvent[]): Promise<void> {
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');

      for (const event of events) {
        await client.query(`
          INSERT INTO event_store (
            id, aggregate_id, aggregate_type, tenant_id,
            event_type, payload, metadata, version, created_at
          ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, NOW())
        `, [
          event.id,
          event.aggregateId,
          event.aggregateType,
          event.tenantId,
          event.type,
          JSON.stringify(event.payload),
          JSON.stringify(event.metadata),
          event.metadata.version
        ]);
      }

      await client.query('COMMIT');

      // Emit all events
      for (const event of events) {
        this.emit('eventAppended', event);
      }
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async getEvents(query: EventStoreQuery, tenantId: string): Promise<DomainEvent[]> {
    const conditions: string[] = ['tenant_id = $1'];
    const values: any[] = [tenantId];
    let paramIndex = 2;

    if (query.aggregateId) {
      conditions.push(`aggregate_id = $${paramIndex++}`);
      values.push(query.aggregateId);
    }

    if (query.aggregateType) {
      conditions.push(`aggregate_type = $${paramIndex++}`);
      values.push(query.aggregateType);
    }

    if (query.fromVersion !== undefined) {
      conditions.push(`version >= $${paramIndex++}`);
      values.push(query.fromVersion);
    }

    if (query.toVersion !== undefined) {
      conditions.push(`version <= $${paramIndex++}`);
      values.push(query.toVersion);
    }

    if (query.fromTimestamp) {
      conditions.push(`created_at >= $${paramIndex++}`);
      values.push(query.fromTimestamp);
    }

    if (query.types && query.types.length > 0) {
      conditions.push(`event_type = ANY($${paramIndex++})`);
      values.push(query.types);
    }

    const result = await this.pool.query(`
      SELECT * FROM event_store
      WHERE ${conditions.join(' AND ')}
      ORDER BY version ASC, created_at ASC
    `, values);

    return result.rows.map(row => ({
      id: row.id,
      aggregateId: row.aggregate_id,
      aggregateType: row.aggregate_type,
      tenantId: row.tenant_id,
      type: row.event_type,
      payload: row.payload,
      metadata: row.metadata
    }));
  }

  async replayEvents(
    aggregateId: string,
    tenantId: string,
    fromVersion: number = 0
  ): Promise<DomainEvent[]> {
    return this.getEvents({ aggregateId, fromVersion }, tenantId);
  }
}

// Aggregate Base Class
export abstract class Aggregate {
  protected id: string;
  protected tenantId: string;
  private uncommittedEvents: DomainEvent[] = [];
  protected version: number = 0;

  constructor(id: string, tenantId: string) {
    this.id = id;
    this.tenantId = tenantId;
  }

  protected apply(event: DomainEvent): void {
    this.when(event);
    this.version++;
    this.uncommittedEvents.push(event);
  }

  protected abstract when(event: DomainEvent): void;

  getUncommittedEvents(): DomainEvent[] {
    return [...this.uncommittedEvents];
  }

  clearUncommittedEvents(): void {
    this.uncommittedEvents = [];
  }

  static reconstitute<T extends Aggregate>(
    AggregateClass: new (id: string, tenantId: string) => T,
    events: DomainEvent[]
  ): T {
    if (events.length === 0) throw new Error('Cannot reconstitute from empty events');
    
    const instance = new AggregateClass(events[0].aggregateId, events[0].tenantId);
    for (const event of events) {
      instance.when(event);
      instance.version++;
    }
    
    return instance;
  }
}

// Conversation Aggregate Example
export class ConversationAggregate extends Aggregate {
  private messages: any[] = [];
  private state: string = 'active';
  private userId: string = '';

  static create(id: string, tenantId: string, userId: string): ConversationAggregate {
    const conversation = new ConversationAggregate(id, tenantId);
    conversation.apply({
      id: require('crypto').randomUUID(),
      aggregateId: id,
      aggregateType: 'Conversation',
      tenantId,
      type: 'ConversationStarted',
      payload: { userId },
      metadata: {
        userId,
        timestamp: new Date(),
        version: 1
      }
    });
    return conversation;
  }

  addMessage(messageId: string, content: any, userId: string): void {
    if (this.state !== 'active') {
      throw new Error('Cannot add message to inactive conversation');
    }

    this.apply({
      id: require('crypto').randomUUID(),
      aggregateId: this.id,
      aggregateType: 'Conversation',
      tenantId: this.tenantId,
      type: 'MessageAdded',
      payload: { messageId, content, userId },
      metadata: {
        userId,
        timestamp: new Date(),
        version: this.version + 1
      }
    });
  }

  end(reason: string, userId: string): void {
    if (this.state === 'ended') return;

    this.apply({
      id: require('crypto').randomUUID(),
      aggregateId: this.id,
      aggregateType: 'Conversation',
      tenantId: this.tenantId,
      type: 'ConversationEnded',
      payload: { reason },
      metadata: {
        userId,
        timestamp: new Date(),
        version: this.version + 1
      }
    });
  }

  protected when(event: DomainEvent): void {
    switch (event.type) {
      case 'ConversationStarted':
        this.userId = event.payload.userId;
        this.state = 'active';
        break;
      case 'MessageAdded':
        this.messages.push(event.payload);
        break;
      case 'ConversationEnded':
        this.state = 'ended';
        break;
    }
  }

  getState() {
    return {
      id: this.id,
      userId: this.userId,
      state: this.state,
      messageCount: this.messages.length,
      version: this.version
    };
  }
}
```

---

## 6. CQRS Implementation

### 6.1 Command and Query Separation

```typescript
// cqrs/CommandBus.ts
import { EventEmitter } from 'events';

interface Command {
  type: string;
  payload: any;
  metadata: {
    tenantId: string;
    userId?: string;
    correlationId: string;
    timestamp: Date;
  };
}

interface CommandHandler<T extends Command> {
  handle(command: T): Promise<void>;
}

export class CommandBus extends EventEmitter {
  private handlers: Map<string, CommandHandler<any>> = new Map();

  register<T extends Command>(commandType: string, handler: CommandHandler<T>): void {
    this.handlers.set(commandType, handler);
  }

  async dispatch(command: Command): Promise<void> {
    const handler = this.handlers.get(command.type);
    if (!handler) {
      throw new Error(`No handler registered for command: ${command.type}`);
    }

    try {
      await handler.handle(command);
      this.emit('commandHandled', { command, success: true });
    } catch (error) {
      this.emit('commandFailed', { command, error });
      throw error;
    }
  }
}

// cqrs/QueryBus.ts
interface Query {
  type: string;
  params: any;
  tenantId: string;
}

interface QueryHandler<T extends Query, R> {
  handle(query: T): Promise<R>;
}

export class QueryBus {
  private handlers: Map<string, QueryHandler<any, any>> = new Map();

  register<T extends Query, R>(queryType: string, handler: QueryHandler<T, R>): void {
    this.handlers.set(queryType, handler);
  }

  async dispatch<R>(query: Query): Promise<R> {
    const handler = this.handlers.get(query.type);
    if (!handler) {
      throw new Error(`No handler registered for query: ${query.type}`);
    }
    return handler.handle(query);
  }
}

// Commands
export interface SendMessageCommand extends Command {
  type: 'SendMessage';
  payload: {
    userId: string;
    messageType: string;
    content: any;
    replyToken?: string;
  };
}

export interface StartConversationCommand extends Command {
  type: 'StartConversation';
  payload: {
    userId: string;
    initialMessage?: string;
  };
}

// Command Handlers
export class SendMessageCommandHandler implements CommandHandler<SendMessageCommand> {
  constructor(
    private eventStore: any,
    private lineClient: any
  ) {}

  async handle(command: SendMessageCommand): Promise<void> {
    const { userId, messageType, content, replyToken } = command.payload;
    const { tenantId, correlationId } = command.metadata;

    // Send via LINE API
    if (replyToken) {
      await this.lineClient.replyMessage(replyToken, [content]);
    } else {
      await this.lineClient.pushMessage(userId, [content]);
    }

    // Store event
    await this.eventStore.append({
      id: require('crypto').randomUUID(),
      aggregateId: userId,
      aggregateType: 'User',
      tenantId,
      type: 'MessageSent',
      payload: { userId, messageType, content },
      metadata: {
        correlationId,
        timestamp: new Date(),
        version: 1
      }
    });
  }
}

// Queries
export interface GetUserMessagesQuery extends Query {
  type: 'GetUserMessages';
  params: {
    userId: string;
    limit: number;
    cursor?: string;
  };
}

export interface GetConversationQuery extends Query {
  type: 'GetConversation';
  params: {
    conversationId: string;
  };
}

// Query Handler - ดึงจาก Read Model
export class GetUserMessagesQueryHandler implements QueryHandler<GetUserMessagesQuery, any> {
  constructor(private readDb: any) {}

  async handle(query: GetUserMessagesQuery): Promise<any> {
    const { userId, limit, cursor } = query.params;
    const { tenantId } = query;

    // ดึงจาก optimized read model (denormalized)
    return this.readDb.getUserMessages(tenantId, userId, { limit, cursor });
  }
}
```

---

## 7. Saga Orchestration

### 7.1 Saga สำหรับ LINE Broadcast Flow

```typescript
// saga/BroadcastSaga.ts
import { EventEmitter } from 'events';
import { LINEAPIClient } from '../line/LINEAPIClient';
import { NotificationService } from '../notifications/NotificationService';
import { Redis } from 'ioredis';

type SagaState = 
  | 'PENDING'
  | 'FETCHING_AUDIENCE'
  | 'SENDING_MESSAGES'
  | 'COLLECTING_STATS'
  | 'COMPLETED'
  | 'FAILED'
  | 'COMPENSATING'
  | 'COMPENSATED';

interface BroadcastSagaData {
  sagaId: string;
  tenantId: string;
  broadcastId: string;
  messageContent: any;
  audienceFilter: any;
  state: SagaState;
  targetUsers: string[];
  sentTo: string[];
  failedUsers: string[];
  statistics: {
    total: number;
    sent: number;
    failed: number;
    deliveryRate: number;
  };
  startedAt: Date;
  completedAt?: Date;
  error?: string;
}

export class BroadcastSaga extends EventEmitter {
  private redis: Redis;
  private lineClient: LINEAPIClient;
  private notifications: NotificationService;
  
  private readonly BATCH_SIZE = 500;
  private readonly RATE_LIMIT_DELAY = 100; // ms between batches

  constructor(tenantId: string) {
    super();
    this.redis = new Redis(process.env.REDIS_URL!);
    const accessToken = this.getTenantToken(tenantId);
    this.lineClient = new LINEAPIClient(accessToken);
    this.notifications = new NotificationService();
  }

  async start(broadcastId: string, tenantId: string, messageContent: any, audienceFilter: any): Promise<string> {
    const sagaId = `saga_${broadcastId}_${Date.now()}`;
    
    const sagaData: BroadcastSagaData = {
      sagaId,
      tenantId,
      broadcastId,
      messageContent,
      audienceFilter,
      state: 'PENDING',
      targetUsers: [],
      sentTo: [],
      failedUsers: [],
      statistics: { total: 0, sent: 0, failed: 0, deliveryRate: 0 },
      startedAt: new Date()
    };

    await this.saveSagaState(sagaData);
    
    // เริ่ม async process
    this.execute(sagaData).catch(async error => {
      sagaData.state = 'FAILED';
      sagaData.error = error.message;
      await this.saveSagaState(sagaData);
      await this.compensate(sagaData);
    });

    return sagaId;
  }

  private async execute(saga: BroadcastSagaData): Promise<void> {
    // Step 1: Fetch target audience
    saga.state = 'FETCHING_AUDIENCE';
    await this.saveSagaState(saga);
    
    saga.targetUsers = await this.fetchTargetAudience(
      saga.tenantId,
      saga.audienceFilter
    );
    saga.statistics.total = saga.targetUsers.length;

    // Step 2: Send messages in batches
    saga.state = 'SENDING_MESSAGES';
    await this.saveSagaState(saga);

    for (let i = 0; i < saga.targetUsers.length; i += this.BATCH_SIZE) {
      const batch = saga.targetUsers.slice(i, i + this.BATCH_SIZE);
      
      const results = await this.sendBatch(saga.tenantId, batch, saga.messageContent);
      
      saga.sentTo.push(...results.sent);
      saga.failedUsers.push(...results.failed);
      saga.statistics.sent += results.sent.length;
      saga.statistics.failed += results.failed.length;
      
      await this.saveSagaState(saga);
      
      // Rate limiting
      if (i + this.BATCH_SIZE < saga.targetUsers.length) {
        await this.delay(this.RATE_LIMIT_DELAY);
      }
    }

    // Step 3: Collect statistics
    saga.state = 'COLLECTING_STATS';
    await this.saveSagaState(saga);

    saga.statistics.deliveryRate = saga.statistics.total > 0
      ? (saga.statistics.sent / saga.statistics.total) * 100
      : 0;

    // Step 4: Complete
    saga.state = 'COMPLETED';
    saga.completedAt = new Date();
    await this.saveSagaState(saga);

    // Notify
    await this.notifications.sendBroadcastComplete({
      tenantId: saga.tenantId,
      broadcastId: saga.broadcastId,
      statistics: saga.statistics
    });

    this.emit('sagaCompleted', saga);
  }

  private async sendBatch(
    tenantId: string, 
    userIds: string[], 
    messageContent: any
  ): Promise<{ sent: string[]; failed: string[] }> {
    const sent: string[] = [];
    const failed: string[] = [];

    try {
      // LINE Multicast API สามารถส่งได้สูงสุด 500 users ต่อครั้ง
      await this.lineClient.multicast(userIds, [messageContent]);
      sent.push(...userIds);
    } catch (error) {
      // ถ้า batch ล้มเหลว ลองส่งทีละคน
      for (const userId of userIds) {
        try {
          await this.lineClient.pushMessage(userId, [messageContent]);
          sent.push(userId);
        } catch {
          failed.push(userId);
        }
      }
    }

    return { sent, failed };
  }

  private async compensate(saga: BroadcastSagaData): Promise<void> {
    saga.state = 'COMPENSATING';
    await this.saveSagaState(saga);

    // Compensation actions:
    // 1. Mark broadcast as failed in database
    await this.markBroadcastFailed(saga.broadcastId, saga.error);

    // 2. Notify admin
    await this.notifications.sendBroadcastFailed({
      tenantId: saga.tenantId,
      broadcastId: saga.broadcastId,
      error: saga.error,
      sentCount: saga.statistics.sent
    });

    saga.state = 'COMPENSATED';
    await this.saveSagaState(saga);
  }

  async getSagaState(sagaId: string): Promise<BroadcastSagaData | null> {
    const data = await this.redis.get(`saga:${sagaId}`);
    return data ? JSON.parse(data) : null;
  }

  private async saveSagaState(saga: BroadcastSagaData): Promise<void> {
    await this.redis.setex(
      `saga:${saga.sagaId}`,
      86400 * 7, // เก็บ 7 วัน
      JSON.stringify(saga)
    );
  }

  private async fetchTargetAudience(tenantId: string, filter: any): Promise<string[]> {
    // ดึง users ตาม filter จาก database
    return [];
  }

  private async markBroadcastFailed(broadcastId: string, error?: string): Promise<void> {
    // Update database
  }

  private getTenantToken(tenantId: string): string {
    return process.env.LINE_CHANNEL_ACCESS_TOKEN || '';
  }

  private delay(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}
```

---

## 8. Event Replay

### 8.1 Event Replay Service

```typescript
// replay/EventReplayService.ts
import { EventStore } from '../eventsourcing/EventStore';
import { ProjectionManager } from '../projections/ProjectionManager';

interface ReplayConfig {
  tenantId: string;
  fromTimestamp?: Date;
  toTimestamp?: Date;
  aggregateTypes?: string[];
  eventTypes?: string[];
  batchSize?: number;
}

interface ReplayProgress {
  id: string;
  status: 'running' | 'completed' | 'failed';
  processedEvents: number;
  totalEvents: number;
  startedAt: Date;
  completedAt?: Date;
  error?: string;
}

export class EventReplayService {
  private eventStore: EventStore;
  private projectionManager: ProjectionManager;

  constructor(eventStore: EventStore, projectionManager: ProjectionManager) {
    this.eventStore = eventStore;
    this.projectionManager = projectionManager;
  }

  async replay(config: ReplayConfig): Promise<string> {
    const replayId = `replay_${Date.now()}`;
    
    const progress: ReplayProgress = {
      id: replayId,
      status: 'running',
      processedEvents: 0,
      totalEvents: 0,
      startedAt: new Date()
    };

    // รัน async
    this.executeReplay(replayId, config, progress).catch(error => {
      progress.status = 'failed';
      progress.error = error.message;
    });

    return replayId;
  }

  private async executeReplay(
    replayId: string,
    config: ReplayConfig,
    progress: ReplayProgress
  ): Promise<void> {
    const batchSize = config.batchSize || 1000;
    let offset = 0;
    let hasMore = true;

    // Reset projections ก่อน replay
    if (config.aggregateTypes) {
      await this.projectionManager.resetProjections(config.aggregateTypes, config.tenantId);
    }

    while (hasMore) {
      const events = await this.eventStore.getEvents({
        aggregateType: config.aggregateTypes?.[0],
        types: config.eventTypes,
        fromTimestamp: config.fromTimestamp,
        toTimestamp: config.toTimestamp
      }, config.tenantId);

      if (events.length === 0) {
        hasMore = false;
        break;
      }

      // Process events in batch
      for (const event of events) {
        await this.projectionManager.applyEvent(event);
        progress.processedEvents++;
      }

      offset += events.length;
      
      if (events.length < batchSize) {
        hasMore = false;
      }

      console.log(`Replay progress: ${progress.processedEvents} events processed`);
    }

    progress.status = 'completed';
    progress.completedAt = new Date();
    
    console.log(`Replay ${replayId} completed: ${progress.processedEvents} events`);
  }
}
```

---

## 9. Dead Letter Events

### 9.1 Dead Letter Queue Handler

```typescript
// dlq/DeadLetterQueueHandler.ts
import { Kafka, Consumer, EachMessagePayload } from 'kafkajs';
import { Redis } from 'ioredis';

interface DeadLetterMessage {
  originalTopic: string;
  originalPartition: number;
  originalOffset: string;
  errorMessage: string;
  errorStack?: string;
  retryCount: number;
  firstFailedAt: Date;
  lastFailedAt: Date;
  payload: any;
  headers: Record<string, string>;
}

export class DeadLetterQueueHandler {
  private consumer: Consumer;
  private redis: Redis;
  private maxRetries: number = 3;
  private retryDelays: number[] = [60000, 300000, 900000]; // 1min, 5min, 15min

  constructor() {
    const kafka = new Kafka({
      clientId: 'dlq-handler',
      brokers: process.env.KAFKA_BROKERS!.split(',')
    });

    this.consumer = kafka.consumer({ groupId: 'dlq-processor' });
    this.redis = new Redis(process.env.REDIS_URL!);
  }

  async start(): Promise<void> {
    await this.consumer.connect();
    await this.consumer.subscribe({ topics: ['dlq.line.events'], fromBeginning: false });

    await this.consumer.run({
      eachMessage: async (payload: EachMessagePayload) => {
        await this.handleDeadLetter(payload);
      }
    });
  }

  private async handleDeadLetter(payload: EachMessagePayload): Promise<void> {
    const { message, topic, partition, heartbeat } = payload;
    
    const dlqMessage: DeadLetterMessage = JSON.parse(message.value?.toString() || '{}');
    
    console.log(`Processing DLQ message: ${dlqMessage.originalTopic}`, {
      retryCount: dlqMessage.retryCount,
      error: dlqMessage.errorMessage
    });

    if (dlqMessage.retryCount < this.maxRetries) {
      // กำหนดเวลา retry
      const delay = this.retryDelays[dlqMessage.retryCount] || 900000;
      const retryAt = Date.now() + delay;
      
      await this.scheduleRetry(dlqMessage, retryAt);
      
      console.log(`Scheduled retry ${dlqMessage.retryCount + 1} for ${new Date(retryAt).toISOString()}`);
    } else {
      // เกิน retry limit - เก็บใน permanent storage สำหรับ manual review
      await this.saveToPermanentStorage(dlqMessage);
      await this.alertOpsTeam(dlqMessage);
      
      console.error(`Message exceeded max retries, saved to permanent storage`, dlqMessage);
    }
    
    await heartbeat();
  }

  private async scheduleRetry(message: DeadLetterMessage, retryAt: number): Promise<void> {
    const key = `retry:${message.originalTopic}:${retryAt}`;
    
    await this.redis.zadd(
      'scheduled_retries',
      retryAt,
      JSON.stringify({
        ...message,
        retryCount: message.retryCount + 1,
        scheduledRetryAt: new Date(retryAt).toISOString()
      })
    );
  }

  async processScheduledRetries(): Promise<void> {
    const now = Date.now();
    
    // ดึง messages ที่ถึงเวลา retry
    const readyMessages = await this.redis.zrangebyscore(
      'scheduled_retries',
      0,
      now
    );

    if (readyMessages.length === 0) return;

    for (const messageStr of readyMessages) {
      const message: DeadLetterMessage = JSON.parse(messageStr);
      await this.reprocessMessage(message);
    }

    // Remove processed messages
    await this.redis.zremrangebyscore('scheduled_retries', 0, now);
  }

  private async reprocessMessage(message: DeadLetterMessage): Promise<void> {
    // Republish ไปยัง original topic
    const kafka = new Kafka({
      clientId: 'dlq-reprocessor',
      brokers: process.env.KAFKA_BROKERS!.split(',')
    });

    const producer = kafka.producer();
    await producer.connect();

    try {
      await producer.send({
        topic: message.originalTopic,
        messages: [{
          value: JSON.stringify(message.payload),
          headers: {
            ...message.headers,
            'x-retry-count': String(message.retryCount),
            'x-original-error': message.errorMessage
          }
        }]
      });
    } finally {
      await producer.disconnect();
    }
  }

  private async saveToPermanentStorage(message: DeadLetterMessage): Promise<void> {
    const key = `dlq:permanent:${message.originalTopic}:${Date.now()}`;
    await this.redis.setex(key, 86400 * 90, JSON.stringify(message)); // เก็บ 90 วัน
  }

  private async alertOpsTeam(message: DeadLetterMessage): Promise<void> {
    // Send alert via PagerDuty/Slack
    console.error(`DLQ Alert: Message from ${message.originalTopic} exceeded max retries`);
  }
}
```

---

## 10. Event Schema Registry

### 10.1 Avro Schema สำหรับ LINE Events

```json
{
  "namespace": "com.linebot.events",
  "type": "record",
  "name": "LineEvent",
  "fields": [
    { "name": "id", "type": "string" },
    { "name": "tenantId", "type": "string" },
    { "name": "timestamp", "type": "long", "logicalType": "timestamp-millis" },
    {
      "name": "eventType",
      "type": {
        "type": "enum",
        "name": "EventType",
        "symbols": [
          "MESSAGE_TEXT", "MESSAGE_IMAGE", "MESSAGE_VIDEO",
          "POSTBACK", "FOLLOW", "UNFOLLOW", "JOIN", "LEAVE",
          "MEMBER_JOINED", "BEACON", "ACCOUNT_LINK"
        ]
      }
    },
    {
      "name": "source",
      "type": {
        "type": "record",
        "name": "EventSource",
        "fields": [
          {
            "name": "sourceType",
            "type": {
              "type": "enum",
              "name": "SourceType",
              "symbols": ["USER", "GROUP", "ROOM"]
            }
          },
          { "name": "userId", "type": ["null", "string"], "default": null },
          { "name": "groupId", "type": ["null", "string"], "default": null }
        ]
      }
    },
    { "name": "replyToken", "type": ["null", "string"], "default": null },
    { "name": "payload", "type": "string" },
    { "name": "processedAt", "type": ["null", "long"], "default": null }
  ]
}
```

```typescript
// schema/SchemaRegistryService.ts
import { SchemaRegistry, SchemaType } from '@kafkajs/confluent-schema-registry';
import { readFileSync } from 'fs';
import { join } from 'path';

export class SchemaRegistryService {
  private registry: SchemaRegistry;

  constructor() {
    this.registry = new SchemaRegistry({
      host: process.env.SCHEMA_REGISTRY_URL!,
      auth: {
        username: process.env.SCHEMA_REGISTRY_USERNAME,
        password: process.env.SCHEMA_REGISTRY_PASSWORD
      }
    });
  }

  async registerSchemas(): Promise<void> {
    const schemas = [
      { subject: 'line.events-value', file: 'line-event.avsc' },
      { subject: 'user.events-value', file: 'user-event.avsc' },
      { subject: 'analytics.raw-value', file: 'analytics-event.avsc' }
    ];

    for (const { subject, file } of schemas) {
      const schema = readFileSync(
        join(__dirname, '../schemas', file),
        'utf8'
      );

      try {
        const { id } = await this.registry.register({
          type: SchemaType.AVRO,
          schema
        }, { subject });
        
        console.log(`Registered schema for ${subject}: ID=${id}`);
      } catch (error) {
        console.error(`Failed to register schema for ${subject}:`, error);
        throw error;
      }
    }
  }

  async encode(subject: string, data: any): Promise<Buffer> {
    const { id } = await this.registry.getLatestSchemaId(subject);
    return this.registry.encode(id, data);
  }

  async decode(buffer: Buffer): Promise<any> {
    return this.registry.decode(buffer);
  }

  async getSchemaHistory(subject: string): Promise<any[]> {
    const versions = await this.registry.getVersions(subject);
    return Promise.all(
      versions.map(v => this.registry.getSchema(v))
    );
  }
}
```

---

## 11. Consumer Groups

### 11.1 Consumer Group Setup

```typescript
// consumers/LineEventConsumer.ts
import { Kafka, Consumer, EachMessagePayload, KafkaMessage } from 'kafkajs';
import { LineEventHandler } from '../handlers/LineEventHandler';

export class LineEventConsumer {
  private consumer: Consumer;
  private handler: LineEventHandler;

  constructor(groupId: string) {
    const kafka = new Kafka({
      clientId: `line-event-consumer-${groupId}`,
      brokers: process.env.KAFKA_BROKERS!.split(','),
      retry: {
        initialRetryTime: 300,
        retries: 10
      }
    });

    this.consumer = kafka.consumer({
      groupId,
      sessionTimeout: 30000,
      heartbeatInterval: 3000,
      maxBytesPerPartition: 1048576, // 1MB
      minBytes: 1,
      maxBytes: 10485760, // 10MB
      maxWaitTimeInMs: 5000
    });

    this.handler = new LineEventHandler();
  }

  async start(): Promise<void> {
    await this.consumer.connect();
    
    await this.consumer.subscribe({
      topics: ['line.events'],
      fromBeginning: false
    });

    await this.consumer.run({
      autoCommit: false,
      autoCommitInterval: 5000,
      autoCommitThreshold: 100,
      partitionsConsumedConcurrently: 4,
      
      eachBatchAutoResolve: true,
      eachBatch: async ({ batch, resolveOffset, heartbeat, commitOffsetsIfNecessary, isRunning, isStale }) => {
        for (const message of batch.messages) {
          if (!isRunning() || isStale()) break;
          
          try {
            await this.processMessage(message, batch.topic, batch.partition);
            resolveOffset(message.offset);
            await heartbeat();
            await commitOffsetsIfNecessary();
          } catch (error) {
            console.error('Message processing failed:', error);
            await this.sendToDeadLetterQueue(message, batch.topic, error as Error);
            resolveOffset(message.offset); // Skip ไป DLQ แล้ว
          }
        }
      }
    });

    // Handle graceful shutdown
    process.on('SIGTERM', async () => {
      await this.stop();
    });
  }

  private async processMessage(
    message: KafkaMessage,
    topic: string,
    partition: number
  ): Promise<void> {
    if (!message.value) return;

    const event = JSON.parse(message.value.toString());
    const tenantId = message.headers?.['tenant-id']?.toString();
    const eventType = message.headers?.['event-type']?.toString();

    if (!tenantId || !eventType) {
      throw new Error('Missing required headers');
    }

    await this.handler.handle(event, { tenantId, eventType, topic, partition });
  }

  private async sendToDeadLetterQueue(
    message: KafkaMessage,
    originalTopic: string,
    error: Error
  ): Promise<void> {
    const kafka = new Kafka({
      clientId: 'dlq-producer',
      brokers: process.env.KAFKA_BROKERS!.split(',')
    });

    const producer = kafka.producer();
    await producer.connect();

    try {
      await producer.send({
        topic: `dlq.${originalTopic}`,
        messages: [{
          value: JSON.stringify({
            originalTopic,
            errorMessage: error.message,
            errorStack: error.stack,
            retryCount: 0,
            firstFailedAt: new Date().toISOString(),
            lastFailedAt: new Date().toISOString(),
            payload: JSON.parse(message.value?.toString() || '{}'),
            headers: Object.fromEntries(
              Object.entries(message.headers || {}).map(([k, v]) => [k, v?.toString()])
            )
          })
        }]
      });
    } finally {
      await producer.disconnect();
    }
  }

  async stop(): Promise<void> {
    await this.consumer.stop();
    await this.consumer.disconnect();
  }
}
```

---

## สรุป

Event-driven architecture สำหรับ LINE Bot ช่วยให้ระบบมี:

1. **Loose Coupling** - services ไม่ต้องรู้จักกันโดยตรง
2. **Scalability** - scale แต่ละ consumer ตาม load
3. **Resilience** - ระบบยังทำงานได้แม้ service หนึ่งล้มเหลว
4. **Auditability** - Event log เป็น source of truth
5. **Replay** - สามารถ replay events เพื่อ rebuild state

Kafka เป็นตัวเลือกที่ดีสำหรับ LINE Bot เพราะรองรับ:
- High throughput (millions of events/sec)
- Durable message storage
- Consumer groups สำหรับ parallel processing
- Schema registry สำหรับ schema evolution
