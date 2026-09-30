# Part 81: Enterprise LINE Bot Architecture

## บทนำ

สถาปัตยกรรมระดับ Enterprise สำหรับ LINE Bot ต้องรองรับการใช้งานจำนวนมาก มีความน่าเชื่อถือสูง และบำรุงรักษาได้ง่ายในระยะยาว บทนี้จะครอบคลุมทุกด้านของการออกแบบระบบระดับ Enterprise ตั้งแต่ Architecture Patterns ไปจนถึงการ Deploy และ Monitor

---

## 1. ภาพรวม Enterprise LINE Bot Architecture

### 1.1 ความต้องการระดับ Enterprise

```
Enterprise Requirements:
┌─────────────────────────────────────────────────────────────┐
│  Functional Requirements                                     │
│  ├── รองรับผู้ใช้ 1-10 ล้านคน                              │
│  ├── Message throughput: 100,000+ msg/min                   │
│  ├── Response time < 200ms (p99)                            │
│  ├── Uptime 99.99% (52 นาที downtime/ปี)                   │
│  └── Multi-channel support (LINE, Web, Mobile)              │
│                                                             │
│  Non-Functional Requirements                                │
│  ├── Security: OWASP Top 10 compliant                      │
│  ├── Compliance: PDPA, ISO 27001                           │
│  ├── Audit trail: ทุก transaction                          │
│  ├── Data residency: ข้อมูลอยู่ในประเทศไทย               │
│  └── Disaster Recovery: RPO < 1hr, RTO < 4hr               │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 High-Level Architecture Diagram

```
                        ┌─────────────────┐
                        │   LINE Platform  │
                        │  (Webhook/API)   │
                        └────────┬────────┘
                                 │ HTTPS
                        ┌────────▼────────┐
                        │   API Gateway   │
                        │  (Kong/AWS GW)  │
                        │  Rate Limiting  │
                        │  Auth/JWT       │
                        └────────┬────────┘
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                   │
     ┌────────▼──────┐  ┌───────▼───────┐  ┌───────▼───────┐
     │  Message      │  │  User Profile  │  │  Campaign     │
     │  Service      │  │  Service       │  │  Service      │
     │  (Node.js)    │  │  (Python)      │  │  (Go)         │
     └────────┬──────┘  └───────┬───────┘  └───────┬───────┘
              │                  │                   │
              └──────────────────┼──────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │      Message Broker      │
                    │    (Apache Kafka)         │
                    │  Topics: events, cmds    │
                    └────────────┬────────────┘
                                 │
          ┌──────────────────────┼──────────────────────┐
          │                      │                       │
┌─────────▼────────┐  ┌─────────▼────────┐  ┌─────────▼────────┐
│  Event Processor  │  │  Analytics       │  │  Notification    │
│  (Flink/Spark)   │  │  Service         │  │  Service         │
└─────────┬────────┘  └─────────┬────────┘  └─────────┬────────┘
          │                      │                       │
          └──────────────────────┼──────────────────────┘
                                 │
                    ┌────────────▼────────────┐
                    │      Data Layer          │
                    │  ┌─────────────────┐    │
                    │  │  PostgreSQL      │    │
                    │  │  (Primary DB)    │    │
                    │  ├─────────────────┤    │
                    │  │  Redis Cluster   │    │
                    │  │  (Cache/Session) │    │
                    │  ├─────────────────┤    │
                    │  │  Elasticsearch   │    │
                    │  │  (Search/Logs)   │    │
                    │  ├─────────────────┤    │
                    │  │  S3/GCS          │    │
                    │  │  (File Storage)  │    │
                    │  └─────────────────┘    │
                    └─────────────────────────┘
```

---

## 2. Architecture Patterns

### 2.1 Monolith to Microservices Migration

#### Phase 1: Monolith (เริ่มต้น)

```javascript
// src/app.js - Monolith Architecture
const express = require('express');
const { Pool } = require('pg');
const redis = require('redis');
const line = require('@line/bot-sdk');

const app = express();
const db = new Pool({ connectionString: process.env.DATABASE_URL });
const cache = redis.createClient({ url: process.env.REDIS_URL });

const lineClient = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET,
});

// ทุก Logic อยู่ใน Monolith เดียว
app.post('/webhook', line.middleware({ channelSecret: process.env.LINE_CHANNEL_SECRET }), 
  async (req, res) => {
    const events = req.body.events;
    
    await Promise.all(events.map(async (event) => {
      if (event.type === 'message') {
        await handleMessage(event);
      } else if (event.type === 'follow') {
        await handleFollow(event);
      } else if (event.type === 'postback') {
        await handlePostback(event);
      }
    }));
    
    res.status(200).json({ status: 'ok' });
  }
);

async function handleMessage(event) {
  const userId = event.source.userId;
  const message = event.message.text;
  
  // Check cache
  const cachedUser = await cache.get(`user:${userId}`);
  let user = cachedUser ? JSON.parse(cachedUser) : null;
  
  if (!user) {
    // Get from DB
    const result = await db.query('SELECT * FROM users WHERE line_user_id = $1', [userId]);
    user = result.rows[0];
    
    if (!user) {
      // Create new user
      const profileResult = await lineClient.getProfile(userId);
      const profile = profileResult;
      
      const insertResult = await db.query(
        'INSERT INTO users (line_user_id, display_name, picture_url) VALUES ($1, $2, $3) RETURNING *',
        [userId, profile.displayName, profile.pictureUrl]
      );
      user = insertResult.rows[0];
    }
    
    await cache.setEx(`user:${userId}`, 3600, JSON.stringify(user));
  }
  
  // Process message
  const response = await processMessage(message, user);
  
  // Send reply
  await lineClient.replyMessage(event.replyToken, {
    type: 'text',
    text: response,
  });
  
  // Log to analytics
  await logMessageAnalytics(userId, message, response);
}

async function processMessage(message, user) {
  // NLP Processing
  const intent = await detectIntent(message);
  
  switch (intent) {
    case 'greeting':
      return `สวัสดีครับ คุณ${user.display_name}! มีอะไรให้ช่วยไหมครับ?`;
    case 'product_inquiry':
      return await getProductInfo(message);
    case 'order_status':
      return await getOrderStatus(user.id);
    default:
      return 'ขออภัยครับ ไม่เข้าใจคำถาม กรุณาถามใหม่อีกครั้ง';
  }
}

module.exports = app;
```

#### Phase 2: Modular Monolith (ขั้นกลาง)

```
src/
├── modules/
│   ├── messaging/
│   │   ├── MessageHandler.js
│   │   ├── MessageProcessor.js
│   │   └── MessageRepository.js
│   ├── users/
│   │   ├── UserService.js
│   │   ├── UserRepository.js
│   │   └── UserCache.js
│   ├── orders/
│   │   ├── OrderService.js
│   │   └── OrderRepository.js
│   └── analytics/
│       ├── AnalyticsService.js
│       └── EventTracker.js
├── shared/
│   ├── database/
│   ├── cache/
│   └── line-client/
└── app.js
```

```javascript
// src/modules/messaging/MessageHandler.js
class MessageHandler {
  constructor({ userService, orderService, analyticsService, lineClient }) {
    this.userService = userService;
    this.orderService = orderService;
    this.analyticsService = analyticsService;
    this.lineClient = lineClient;
  }
  
  async handle(event) {
    const startTime = Date.now();
    
    try {
      const user = await this.userService.getOrCreate(event.source.userId);
      const response = await this.processMessage(event.message.text, user);
      
      await this.lineClient.replyMessage(event.replyToken, {
        type: 'text',
        text: response,
      });
      
      await this.analyticsService.track({
        event: 'message_handled',
        userId: user.id,
        message: event.message.text,
        response,
        duration: Date.now() - startTime,
      });
    } catch (error) {
      console.error('Error handling message:', error);
      await this.lineClient.replyMessage(event.replyToken, {
        type: 'text',
        text: 'ขออภัยครับ เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง',
      });
    }
  }
  
  async processMessage(text, user) {
    // Delegation to specific processors
    const processor = this.getProcessor(text);
    return processor.process(text, user);
  }
  
  getProcessor(text) {
    if (text.includes('สั่งซื้อ') || text.includes('order')) {
      return new OrderProcessor(this.orderService);
    }
    return new DefaultProcessor();
  }
}

module.exports = MessageHandler;
```

#### Phase 3: Microservices (เต็มรูปแบบ)

```yaml
# docker-compose.microservices.yml
version: '3.8'

services:
  # API Gateway
  kong:
    image: kong:3.4
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-db
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
    ports:
      - "8000:8000"   # Proxy
      - "8001:8001"   # Admin API
    depends_on:
      - kong-db

  # Message Service
  message-service:
    build: ./services/message-service
    environment:
      DATABASE_URL: postgres://user:pass@message-db:5432/messages
      KAFKA_BROKERS: kafka:9092
      LINE_CHANNEL_ACCESS_TOKEN: ${LINE_CHANNEL_ACCESS_TOKEN}
      LINE_CHANNEL_SECRET: ${LINE_CHANNEL_SECRET}
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 512M
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3

  # User Profile Service
  user-service:
    build: ./services/user-service
    environment:
      DATABASE_URL: postgres://user:pass@user-db:5432/users
      REDIS_URL: redis://redis:6379
      KAFKA_BROKERS: kafka:9092
    deploy:
      replicas: 2

  # Campaign Service
  campaign-service:
    build: ./services/campaign-service
    environment:
      DATABASE_URL: postgres://user:pass@campaign-db:5432/campaigns
      KAFKA_BROKERS: kafka:9092
    deploy:
      replicas: 2

  # Notification Service
  notification-service:
    build: ./services/notification-service
    environment:
      KAFKA_BROKERS: kafka:9092
      LINE_CHANNEL_ACCESS_TOKEN: ${LINE_CHANNEL_ACCESS_TOKEN}
    deploy:
      replicas: 2

  # Apache Kafka
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  # Databases
  message-db:
    image: postgres:15
    environment:
      POSTGRES_DB: messages
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - message-db-data:/var/lib/postgresql/data

  user-db:
    image: postgres:15
    environment:
      POSTGRES_DB: users
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass

  # Redis Cluster
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes

  # Elasticsearch
  elasticsearch:
    image: elasticsearch:8.10.0
    environment:
      discovery.type: single-node
      xpack.security.enabled: "false"
    volumes:
      - es-data:/usr/share/elasticsearch/data

volumes:
  message-db-data:
  es-data:
```

---

## 3. Event Sourcing Pattern

### 3.1 แนวคิด Event Sourcing

```
Event Sourcing Concept:
┌─────────────────────────────────────────────────────────┐
│  Traditional (State-based)                              │
│  ┌──────────────┐    update    ┌──────────────┐        │
│  │  User State  │──────────────▶  User State  │        │
│  │  {status:A}  │              │  {status:B}  │        │
│  └──────────────┘              └──────────────┘        │
│  ← ข้อมูลเก่าหายไป →                                  │
│                                                         │
│  Event Sourcing                                         │
│  ┌──────────────────────────────────────────────┐      │
│  │ Event Store (Append-only)                    │      │
│  │ [UserCreated] → [UserFollowed] → [UserPurchased] │   │
│  │ → [UserUnfollowed] → [UserRefollowed]        │      │
│  └──────────────────────────────────────────────┘      │
│  ↑ ทุก event ถูกบันทึกไว้ → replay ได้ทุกเมื่อ       │
└─────────────────────────────────────────────────────────┘
```

### 3.2 Event Store Implementation

```javascript
// src/event-sourcing/EventStore.js
const { Pool } = require('pg');
const { EventEmitter } = require('events');

class EventStore extends EventEmitter {
  constructor(pool) {
    super();
    this.pool = pool;
  }

  async initialize() {
    await this.pool.query(`
      CREATE TABLE IF NOT EXISTS events (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        aggregate_id VARCHAR(255) NOT NULL,
        aggregate_type VARCHAR(100) NOT NULL,
        event_type VARCHAR(100) NOT NULL,
        event_data JSONB NOT NULL,
        metadata JSONB DEFAULT '{}',
        version INTEGER NOT NULL,
        created_at TIMESTAMPTZ DEFAULT NOW(),
        UNIQUE(aggregate_id, version)
      );
      
      CREATE INDEX IF NOT EXISTS idx_events_aggregate_id ON events(aggregate_id);
      CREATE INDEX IF NOT EXISTS idx_events_aggregate_type ON events(aggregate_type);
      CREATE INDEX IF NOT EXISTS idx_events_event_type ON events(event_type);
      CREATE INDEX IF NOT EXISTS idx_events_created_at ON events(created_at);
    `);
  }

  async append(aggregateId, aggregateType, events, expectedVersion) {
    const client = await this.pool.connect();
    
    try {
      await client.query('BEGIN');
      
      // Optimistic locking - ตรวจสอบ version
      const currentVersion = await this.getCurrentVersion(client, aggregateId);
      
      if (expectedVersion !== null && currentVersion !== expectedVersion) {
        throw new ConcurrencyError(
          `Expected version ${expectedVersion} but got ${currentVersion}`
        );
      }
      
      let version = currentVersion + 1;
      const savedEvents = [];
      
      for (const event of events) {
        const result = await client.query(
          `INSERT INTO events 
           (aggregate_id, aggregate_type, event_type, event_data, metadata, version)
           VALUES ($1, $2, $3, $4, $5, $6)
           RETURNING *`,
          [
            aggregateId,
            aggregateType,
            event.type,
            JSON.stringify(event.data),
            JSON.stringify(event.metadata || {}),
            version++,
          ]
        );
        savedEvents.push(result.rows[0]);
      }
      
      await client.query('COMMIT');
      
      // Emit events สำหรับ subscribers
      savedEvents.forEach(event => this.emit('event', event));
      
      return savedEvents;
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  async getEvents(aggregateId, fromVersion = 0) {
    const result = await this.pool.query(
      `SELECT * FROM events 
       WHERE aggregate_id = $1 AND version > $2
       ORDER BY version ASC`,
      [aggregateId, fromVersion]
    );
    return result.rows;
  }

  async getEventsByType(eventType, fromDate, toDate) {
    const result = await this.pool.query(
      `SELECT * FROM events 
       WHERE event_type = $1 
       AND created_at BETWEEN $2 AND $3
       ORDER BY created_at ASC`,
      [eventType, fromDate, toDate]
    );
    return result.rows;
  }

  async getCurrentVersion(client, aggregateId) {
    const result = await client.query(
      'SELECT MAX(version) as version FROM events WHERE aggregate_id = $1',
      [aggregateId]
    );
    return result.rows[0].version || 0;
  }
}

// User Aggregate
class UserAggregate {
  constructor(id) {
    this.id = id;
    this.version = 0;
    this.state = null;
    this.pendingEvents = [];
  }

  static create(lineUserId, displayName, pictureUrl) {
    const user = new UserAggregate(lineUserId);
    user.apply({
      type: 'UserCreated',
      data: { lineUserId, displayName, pictureUrl },
      metadata: { timestamp: new Date().toISOString() },
    });
    return user;
  }

  follow() {
    if (this.state?.isFollowing) {
      throw new Error('User is already following');
    }
    this.apply({
      type: 'UserFollowed',
      data: { followedAt: new Date().toISOString() },
    });
  }

  unfollow() {
    if (!this.state?.isFollowing) {
      throw new Error('User is not following');
    }
    this.apply({
      type: 'UserUnfollowed',
      data: { unfollowedAt: new Date().toISOString() },
    });
  }

  updateProfile(displayName, pictureUrl) {
    this.apply({
      type: 'UserProfileUpdated',
      data: { displayName, pictureUrl },
    });
  }

  apply(event) {
    this.pendingEvents.push(event);
    this.mutate(event);
  }

  mutate(event) {
    switch (event.type) {
      case 'UserCreated':
        this.state = {
          lineUserId: event.data.lineUserId,
          displayName: event.data.displayName,
          pictureUrl: event.data.pictureUrl,
          isFollowing: false,
          createdAt: event.data.timestamp || new Date().toISOString(),
        };
        break;
      case 'UserFollowed':
        this.state.isFollowing = true;
        this.state.followedAt = event.data.followedAt;
        break;
      case 'UserUnfollowed':
        this.state.isFollowing = false;
        this.state.unfollowedAt = event.data.unfollowedAt;
        break;
      case 'UserProfileUpdated':
        this.state.displayName = event.data.displayName;
        this.state.pictureUrl = event.data.pictureUrl;
        break;
    }
    this.version++;
  }

  static rehydrate(events) {
    const user = new UserAggregate(null);
    events.forEach(event => user.mutate(JSON.parse(event.event_data)));
    user.version = events[events.length - 1]?.version || 0;
    return user;
  }
}

// User Repository with Event Sourcing
class UserRepository {
  constructor(eventStore, snapshotStore) {
    this.eventStore = eventStore;
    this.snapshotStore = snapshotStore;
  }

  async save(user) {
    const events = user.pendingEvents.splice(0);
    if (events.length > 0) {
      await this.eventStore.append(
        user.id,
        'User',
        events,
        user.version - events.length
      );
    }
  }

  async findById(userId) {
    // ลอง load จาก snapshot ก่อน
    const snapshot = await this.snapshotStore.getLatest(userId);
    
    let fromVersion = 0;
    let user = new UserAggregate(userId);
    
    if (snapshot) {
      user.state = snapshot.state;
      user.version = snapshot.version;
      fromVersion = snapshot.version;
    }
    
    // Load events หลัง snapshot
    const events = await this.eventStore.getEvents(userId, fromVersion);
    
    if (events.length === 0 && !snapshot) {
      return null;
    }
    
    events.forEach(event => {
      user.mutate(JSON.parse(event.event_data));
    });
    
    // สร้าง snapshot ถ้า events มากเกินไป
    if (events.length > 50) {
      await this.snapshotStore.save(userId, user.state, user.version);
    }
    
    return user;
  }
}

module.exports = { EventStore, UserAggregate, UserRepository };
```

---

## 4. CQRS Pattern

### 4.1 CQRS Architecture

```
CQRS Pattern:
┌─────────────────────────────────────────────────────────┐
│                    LINE Bot Request                      │
│                          │                              │
│            ┌─────────────┴─────────────┐               │
│            ▼                           ▼               │
│      ┌──────────┐               ┌──────────┐           │
│      │ Command  │               │  Query   │           │
│      │ Handler  │               │ Handler  │           │
│      └────┬─────┘               └────┬─────┘           │
│           │                          │                  │
│           ▼                          ▼                  │
│    ┌─────────────┐           ┌─────────────┐           │
│    │  Write DB   │           │   Read DB   │           │
│    │ (PostgreSQL)│           │  (MongoDB/  │           │
│    │ Normalized  │           │  Elastic)   │           │
│    └──────┬──────┘           └─────────────┘           │
│           │                          ▲                  │
│           │    Event Sync            │                  │
│           └──────────────────────────┘                  │
└─────────────────────────────────────────────────────────┘
```

### 4.2 CQRS Implementation

```javascript
// src/cqrs/commands/CreateOrderCommand.js
class CreateOrderCommand {
  constructor({ userId, items, shippingAddress }) {
    this.userId = userId;
    this.items = items;
    this.shippingAddress = shippingAddress;
  }

  validate() {
    if (!this.userId) throw new Error('userId is required');
    if (!this.items || this.items.length === 0) throw new Error('items cannot be empty');
    if (!this.shippingAddress) throw new Error('shippingAddress is required');
  }
}

// src/cqrs/handlers/CreateOrderCommandHandler.js
class CreateOrderCommandHandler {
  constructor({ orderRepository, eventStore, inventoryService }) {
    this.orderRepository = orderRepository;
    this.eventStore = eventStore;
    this.inventoryService = inventoryService;
  }

  async handle(command) {
    command.validate();
    
    // Check inventory
    for (const item of command.items) {
      const available = await this.inventoryService.checkAvailability(
        item.productId,
        item.quantity
      );
      if (!available) {
        throw new Error(`Product ${item.productId} is out of stock`);
      }
    }
    
    // Create order aggregate
    const order = Order.create({
      userId: command.userId,
      items: command.items,
      shippingAddress: command.shippingAddress,
    });
    
    // Save to write DB
    await this.orderRepository.save(order);
    
    // Publish domain events
    await this.eventStore.append(
      order.id,
      'Order',
      order.pendingEvents,
      0
    );
    
    return { orderId: order.id, status: 'created' };
  }
}

// src/cqrs/queries/GetUserOrdersQuery.js
class GetUserOrdersQuery {
  constructor({ userId, page = 1, limit = 10 }) {
    this.userId = userId;
    this.page = page;
    this.limit = limit;
  }
}

// src/cqrs/handlers/GetUserOrdersQueryHandler.js
class GetUserOrdersQueryHandler {
  constructor({ readDatabase }) {
    this.readDatabase = readDatabase;
  }

  async handle(query) {
    // Read from denormalized read model (MongoDB)
    const { page, limit, userId } = query;
    const skip = (page - 1) * limit;
    
    const orders = await this.readDatabase.collection('order_views').find(
      { userId },
      {
        sort: { createdAt: -1 },
        skip,
        limit,
        projection: {
          orderId: 1,
          status: 1,
          items: 1,
          totalAmount: 1,
          createdAt: 1,
        },
      }
    ).toArray();
    
    const total = await this.readDatabase.collection('order_views')
      .countDocuments({ userId });
    
    return {
      orders,
      pagination: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
      },
    };
  }
}

// src/cqrs/projections/OrderProjection.js
// Projector รับ events จาก Event Store แล้ว update Read Model
class OrderProjection {
  constructor({ readDatabase }) {
    this.readDatabase = readDatabase;
  }

  async project(event) {
    switch (event.event_type) {
      case 'OrderCreated':
        await this.handleOrderCreated(event);
        break;
      case 'OrderStatusUpdated':
        await this.handleOrderStatusUpdated(event);
        break;
      case 'OrderCancelled':
        await this.handleOrderCancelled(event);
        break;
    }
  }

  async handleOrderCreated(event) {
    const data = JSON.parse(event.event_data);
    await this.readDatabase.collection('order_views').insertOne({
      orderId: event.aggregate_id,
      userId: data.userId,
      items: data.items,
      totalAmount: data.totalAmount,
      status: 'pending',
      shippingAddress: data.shippingAddress,
      createdAt: event.created_at,
      updatedAt: event.created_at,
    });
  }

  async handleOrderStatusUpdated(event) {
    const data = JSON.parse(event.event_data);
    await this.readDatabase.collection('order_views').updateOne(
      { orderId: event.aggregate_id },
      {
        $set: {
          status: data.newStatus,
          updatedAt: event.created_at,
        },
        $push: {
          statusHistory: {
            status: data.newStatus,
            updatedAt: event.created_at,
            reason: data.reason,
          },
        },
      }
    );
  }
}

// Command Bus และ Query Bus
class CommandBus {
  constructor() {
    this.handlers = new Map();
  }

  register(commandClass, handler) {
    this.handlers.set(commandClass.name, handler);
    return this;
  }

  async execute(command) {
    const handlerName = command.constructor.name;
    const handler = this.handlers.get(handlerName);
    
    if (!handler) {
      throw new Error(`No handler registered for ${handlerName}`);
    }
    
    return handler.handle(command);
  }
}

class QueryBus {
  constructor() {
    this.handlers = new Map();
  }

  register(queryClass, handler) {
    this.handlers.set(queryClass.name, handler);
    return this;
  }

  async execute(query) {
    const handlerName = query.constructor.name;
    const handler = this.handlers.get(handlerName);
    
    if (!handler) {
      throw new Error(`No handler registered for ${handlerName}`);
    }
    
    return handler.handle(query);
  }
}

module.exports = {
  CreateOrderCommand,
  CreateOrderCommandHandler,
  GetUserOrdersQuery,
  GetUserOrdersQueryHandler,
  OrderProjection,
  CommandBus,
  QueryBus,
};
```

---

## 5. Saga Pattern for Distributed Transactions

### 5.1 Saga Architecture

```
Saga Pattern - Order Processing:
┌─────────────────────────────────────────────────────────────────┐
│ Order Saga Orchestrator                                          │
│                                                                  │
│  Step 1: Reserve Inventory    ──success──▶  Step 2: Charge Card │
│       │                                          │               │
│       │ fail                                     │ fail          │
│       ▼                                          ▼               │
│  Compensate: Nothing         Compensate: Release Inventory       │
│                                                                  │
│  Step 3: Schedule Delivery   ──success──▶  Step 4: Notify User  │
│       │                                          │               │
│       │ fail                                     │ Complete!     │
│       ▼                                                          │
│  Compensate: Refund + Release Inventory                          │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Saga Implementation

```javascript
// src/sagas/OrderSaga.js
const { EventEmitter } = require('events');

class SagaStep {
  constructor({ action, compensation }) {
    this.action = action;
    this.compensation = compensation;
  }
}

class OrderSaga {
  constructor({ inventoryService, paymentService, deliveryService, notificationService }) {
    this.inventoryService = inventoryService;
    this.paymentService = paymentService;
    this.deliveryService = deliveryService;
    this.notificationService = notificationService;
    
    this.steps = [
      new SagaStep({
        action: (ctx) => this.reserveInventory(ctx),
        compensation: (ctx) => this.releaseInventory(ctx),
      }),
      new SagaStep({
        action: (ctx) => this.chargePayment(ctx),
        compensation: (ctx) => this.refundPayment(ctx),
      }),
      new SagaStep({
        action: (ctx) => this.scheduleDelivery(ctx),
        compensation: (ctx) => this.cancelDelivery(ctx),
      }),
      new SagaStep({
        action: (ctx) => this.notifyUser(ctx),
        compensation: null, // ไม่ต้อง compensate
      }),
    ];
  }

  async execute(orderData) {
    const context = {
      orderId: orderData.orderId,
      userId: orderData.userId,
      items: orderData.items,
      paymentMethod: orderData.paymentMethod,
      completedSteps: [],
    };

    for (let i = 0; i < this.steps.length; i++) {
      const step = this.steps[i];
      
      try {
        console.log(`Executing saga step ${i + 1}/${this.steps.length}`);
        const result = await step.action(context);
        context.completedSteps.push({ step: i, result });
        
      } catch (error) {
        console.error(`Saga step ${i + 1} failed:`, error);
        
        // Execute compensations in reverse order
        await this.compensate(context, i);
        
        throw new SagaFailedError(
          `Order saga failed at step ${i + 1}: ${error.message}`,
          context
        );
      }
    }
    
    return { success: true, context };
  }

  async compensate(context, failedStep) {
    console.log(`Starting compensation from step ${failedStep}`);
    
    for (let i = failedStep; i >= 0; i--) {
      const step = this.steps[i];
      
      if (step.compensation) {
        try {
          await step.compensation(context);
          console.log(`Compensation step ${i + 1} completed`);
        } catch (error) {
          console.error(`Compensation step ${i + 1} failed:`, error);
          // Log for manual intervention
          await this.logCompensationFailure(context, i, error);
        }
      }
    }
  }

  async reserveInventory(ctx) {
    const reservations = await Promise.all(
      ctx.items.map(item =>
        this.inventoryService.reserve(item.productId, item.quantity)
      )
    );
    ctx.reservationIds = reservations.map(r => r.reservationId);
    return { reservationIds: ctx.reservationIds };
  }

  async releaseInventory(ctx) {
    if (ctx.reservationIds) {
      await Promise.all(
        ctx.reservationIds.map(id => this.inventoryService.release(id))
      );
    }
  }

  async chargePayment(ctx) {
    const charge = await this.paymentService.charge({
      orderId: ctx.orderId,
      userId: ctx.userId,
      amount: ctx.totalAmount,
      method: ctx.paymentMethod,
    });
    ctx.chargeId = charge.chargeId;
    return { chargeId: ctx.chargeId };
  }

  async refundPayment(ctx) {
    if (ctx.chargeId) {
      await this.paymentService.refund(ctx.chargeId);
    }
  }

  async scheduleDelivery(ctx) {
    const delivery = await this.deliveryService.schedule({
      orderId: ctx.orderId,
      items: ctx.items,
      address: ctx.shippingAddress,
    });
    ctx.deliveryId = delivery.deliveryId;
    return { deliveryId: ctx.deliveryId };
  }

  async cancelDelivery(ctx) {
    if (ctx.deliveryId) {
      await this.deliveryService.cancel(ctx.deliveryId);
    }
  }

  async notifyUser(ctx) {
    await this.notificationService.sendOrderConfirmation({
      userId: ctx.userId,
      orderId: ctx.orderId,
      deliveryDate: ctx.deliveryDate,
    });
  }

  async logCompensationFailure(context, step, error) {
    // บันทึกลง Dead Letter Queue สำหรับ manual intervention
    await this.deadLetterQueue.push({
      type: 'COMPENSATION_FAILED',
      sagaContext: context,
      failedStep: step,
      error: error.message,
      timestamp: new Date().toISOString(),
    });
  }
}

// Saga State Machine with Persistence
class PersistentSaga {
  constructor({ sagaRepository, ...services }) {
    this.sagaRepository = sagaRepository;
    this.services = services;
  }

  async start(sagaId, orderData) {
    // บันทึก saga state เพื่อ resume ได้ถ้า crash
    await this.sagaRepository.create({
      sagaId,
      status: 'started',
      data: orderData,
      currentStep: 0,
      completedSteps: [],
    });
    
    return this.resume(sagaId);
  }

  async resume(sagaId) {
    const sagaState = await this.sagaRepository.findById(sagaId);
    
    if (!sagaState) {
      throw new Error(`Saga ${sagaId} not found`);
    }
    
    if (sagaState.status === 'completed') {
      return sagaState;
    }
    
    // Resume from where we left off
    const saga = new OrderSaga(this.services);
    
    try {
      const result = await saga.execute(sagaState.data);
      
      await this.sagaRepository.update(sagaId, {
        status: 'completed',
        completedAt: new Date().toISOString(),
        result,
      });
      
      return result;
    } catch (error) {
      await this.sagaRepository.update(sagaId, {
        status: 'failed',
        error: error.message,
        failedAt: new Date().toISOString(),
      });
      throw error;
    }
  }
}

module.exports = { OrderSaga, PersistentSaga };
```

---

## 6. Technology Stack Choices

### 6.1 Technology Comparison Matrix

```
┌──────────────────┬──────────────┬──────────────┬──────────────┐
│  Component       │  Option A    │  Option B    │  Recommended │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│  API Framework   │  Express.js  │  Fastify     │  Fastify     │
│  (Performance)   │  ~15k req/s  │  ~75k req/s  │  (5x faster) │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│  Message Queue   │  RabbitMQ    │  Apache Kafka│  Kafka       │
│                  │  ~50k msg/s  │  ~1M msg/s   │  (20x more)  │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│  Primary DB      │  MySQL       │  PostgreSQL  │  PostgreSQL  │
│                  │  Good        │  Better JSON │  (JSONB/GIS) │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│  Cache           │  Memcached   │  Redis       │  Redis       │
│                  │  Simple      │  Rich data   │  (Pub/Sub)   │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│  Search          │  Elasticsearch│ OpenSearch  │  ES/OS       │
│                  │  Complex     │  Open source │  (both fine) │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│  Container       │  Docker      │  Podman      │  Docker      │
│  Orchestration   │  Kubernetes  │  Nomad       │  Kubernetes  │
├──────────────────┼──────────────┼──────────────┼──────────────┤
│  Service Mesh    │  Istio       │  Linkerd     │  Istio       │
│                  │  Feature-rich│  Lightweight │  (Enterprise)│
├──────────────────┼──────────────┼──────────────┼──────────────┤
│  Observability   │  Prometheus  │  Datadog     │  Prometheus  │
│                  │  +Grafana    │  (Paid)      │  +Grafana    │
└──────────────────┴──────────────┴──────────────┴──────────────┘
```

### 6.2 Fastify Setup สำหรับ High Performance

```javascript
// src/server.js - Fastify Enterprise Setup
const fastify = require('fastify');
const { Pool } = require('pg');

const app = fastify({
  logger: {
    level: process.env.LOG_LEVEL || 'info',
    serializers: {
      req(request) {
        return {
          method: request.method,
          url: request.url,
          headers: {
            host: request.headers.host,
            'user-agent': request.headers['user-agent'],
          },
        };
      },
    },
  },
  trustProxy: true,
  connectionTimeout: 10000,
  keepAliveTimeout: 5000,
  maxParamLength: 200,
  bodyLimit: 1048576, // 1MB
});

// Register plugins
app.register(require('@fastify/helmet'), {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
    },
  },
});

app.register(require('@fastify/rate-limit'), {
  max: 1000,
  timeWindow: '1 minute',
  redis: require('./shared/redis').client,
  keyGenerator: (req) => req.headers['x-line-userid'] || req.ip,
});

app.register(require('@fastify/cors'), {
  origin: process.env.ALLOWED_ORIGINS?.split(',') || false,
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  credentials: true,
});

// PostgreSQL Connection Pool
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  min: 5,
  max: 50,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
  statement_timeout: 5000,
});

app.decorate('db', pool);

// Health Check
app.get('/health', {
  schema: {
    response: {
      200: {
        type: 'object',
        properties: {
          status: { type: 'string' },
          version: { type: 'string' },
          timestamp: { type: 'string' },
          uptime: { type: 'number' },
        },
      },
    },
  },
}, async (request, reply) => {
  // Check DB
  try {
    await pool.query('SELECT 1');
  } catch (error) {
    return reply.code(503).send({ status: 'unhealthy', error: 'Database unavailable' });
  }
  
  return {
    status: 'healthy',
    version: process.env.APP_VERSION || '1.0.0',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
  };
});

// Graceful Shutdown
const shutdown = async (signal) => {
  app.log.info(`Received ${signal}, shutting down gracefully`);
  
  await app.close();
  await pool.end();
  
  process.exit(0);
};

process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT', () => shutdown('SIGINT'));

module.exports = app;
```

---

## 7. Security Architecture

### 7.1 Security Layers

```
Security Architecture:
┌─────────────────────────────────────────────────────────────┐
│  Layer 1: Network Security                                   │
│  ├── WAF (Web Application Firewall)                         │
│  ├── DDoS Protection (Cloudflare/AWS Shield)                │
│  ├── TLS 1.3 Encryption                                     │
│  └── IP Allowlist for LINE Webhook IPs                      │
│                                                             │
│  Layer 2: Application Security                              │
│  ├── LINE Signature Verification                            │
│  ├── JWT Authentication                                     │
│  ├── RBAC Authorization                                     │
│  ├── Input Validation & Sanitization                        │
│  └── CSRF Protection                                        │
│                                                             │
│  Layer 3: Data Security                                     │
│  ├── Encryption at Rest (AES-256)                          │
│  ├── Encryption in Transit (TLS)                            │
│  ├── PII Masking                                            │
│  ├── Key Management (AWS KMS/HashiCorp Vault)               │
│  └── Database Encryption                                    │
│                                                             │
│  Layer 4: Infrastructure Security                           │
│  ├── Container Security (Trivy scanning)                    │
│  ├── Secrets Management (Vault)                             │
│  ├── Network Policies (Kubernetes)                          │
│  └── Pod Security Policies                                  │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Security Implementation

```javascript
// src/security/LineSignatureVerifier.js
const crypto = require('crypto');

class LineSignatureVerifier {
  constructor(channelSecret) {
    this.channelSecret = channelSecret;
  }

  verify(body, signature) {
    if (!signature) {
      throw new SecurityError('Missing X-Line-Signature header');
    }
    
    const hash = crypto
      .createHmac('sha256', this.channelSecret)
      .update(body)
      .digest('base64');
    
    // Timing-safe comparison เพื่อป้องกัน timing attack
    const expectedSignature = Buffer.from(hash);
    const receivedSignature = Buffer.from(signature.replace('sha256=', ''));
    
    if (expectedSignature.length !== receivedSignature.length) {
      throw new SecurityError('Invalid signature');
    }
    
    if (!crypto.timingSafeEqual(expectedSignature, receivedSignature)) {
      throw new SecurityError('Signature verification failed');
    }
    
    return true;
  }
}

// src/security/PIIMasker.js
class PIIMasker {
  maskPhoneNumber(phone) {
    if (!phone) return null;
    return phone.replace(/(\d{3})\d{4}(\d{3})/, '$1****$2');
  }
  
  maskEmail(email) {
    if (!email) return null;
    const [user, domain] = email.split('@');
    const maskedUser = user.substring(0, 2) + '*'.repeat(user.length - 2);
    return `${maskedUser}@${domain}`;
  }
  
  maskLineUserId(userId) {
    if (!userId) return null;
    return userId.substring(0, 5) + '...' + userId.substring(userId.length - 4);
  }
  
  maskObject(obj, fieldsToMask = ['phone', 'email', 'lineUserId']) {
    const masked = { ...obj };
    for (const field of fieldsToMask) {
      if (masked[field]) {
        if (field === 'phone') masked[field] = this.maskPhoneNumber(masked[field]);
        if (field === 'email') masked[field] = this.maskEmail(masked[field]);
        if (field === 'lineUserId') masked[field] = this.maskLineUserId(masked[field]);
      }
    }
    return masked;
  }
}

// src/security/EncryptionService.js
const { createCipheriv, createDecipheriv, randomBytes } = require('crypto');

class EncryptionService {
  constructor(kmsClient) {
    this.kmsClient = kmsClient;
    this.algorithm = 'aes-256-gcm';
  }

  async encrypt(plaintext) {
    // Get encryption key จาก KMS
    const key = await this.kmsClient.getDataKey();
    
    const iv = randomBytes(12);
    const cipher = createCipheriv(this.algorithm, key, iv);
    
    const encrypted = Buffer.concat([
      cipher.update(plaintext, 'utf8'),
      cipher.final(),
    ]);
    
    const authTag = cipher.getAuthTag();
    
    return {
      ciphertext: encrypted.toString('base64'),
      iv: iv.toString('base64'),
      authTag: authTag.toString('base64'),
      keyVersion: key.version,
    };
  }

  async decrypt(encryptedData) {
    const key = await this.kmsClient.getDataKey(encryptedData.keyVersion);
    
    const decipher = createDecipheriv(
      this.algorithm,
      key,
      Buffer.from(encryptedData.iv, 'base64')
    );
    
    decipher.setAuthTag(Buffer.from(encryptedData.authTag, 'base64'));
    
    const decrypted = Buffer.concat([
      decipher.update(Buffer.from(encryptedData.ciphertext, 'base64')),
      decipher.final(),
    ]);
    
    return decrypted.toString('utf8');
  }
}

// Audit Trail Middleware
class AuditLogger {
  constructor({ auditRepository }) {
    this.auditRepository = auditRepository;
  }

  middleware() {
    return async (req, reply) => {
      const startTime = Date.now();
      
      reply.addHook('onSend', async (request, reply, payload) => {
        await this.auditRepository.log({
          timestamp: new Date().toISOString(),
          userId: request.user?.id,
          lineUserId: request.lineEvent?.source?.userId,
          action: `${request.method} ${request.url}`,
          statusCode: reply.statusCode,
          duration: Date.now() - startTime,
          ip: request.ip,
          userAgent: request.headers['user-agent'],
          requestId: request.id,
        });
      });
    };
  }
}

module.exports = { LineSignatureVerifier, PIIMasker, EncryptionService, AuditLogger };
```

---

## 8. Disaster Recovery Architecture

### 8.1 DR Strategy

```
Disaster Recovery Tiers:
┌───────────────────────────────────────────────────────────────┐
│  Tier 1: Active-Active (Multi-AZ)                             │
│  RTO: ~0  RPO: ~0  Cost: $$$$                                 │
│  ├── Primary Region: ap-southeast-1 (Singapore)               │
│  ├── Secondary Region: ap-northeast-1 (Tokyo)                 │
│  └── Traffic: 70/30 split normally                            │
│                                                               │
│  Tier 2: Warm Standby                                         │
│  RTO: < 1hr  RPO: < 15min  Cost: $$$                         │
│  ├── Primary: Full traffic                                    │
│  ├── DR: Scaled-down (25% capacity)                           │
│  └── Failover: Route53 health check automatic                 │
│                                                               │
│  Tier 3: Pilot Light                                          │
│  RTO: 1-4hr  RPO: < 1hr  Cost: $$                            │
│  ├── Primary: Full traffic                                    │
│  ├── DR: Only core components running                         │
│  └── Failover: Scale up DR manually                           │
│                                                               │
│  Tier 4: Backup & Restore                                     │
│  RTO: 4-24hr  RPO: < 24hr  Cost: $                           │
│  └── Basic backup to S3/GCS                                   │
└───────────────────────────────────────────────────────────────┘
```

### 8.2 DR Implementation

```yaml
# terraform/dr/main.tf
# Multi-Region DR Setup

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Primary Region
provider "aws" {
  alias  = "primary"
  region = "ap-southeast-1"
}

# DR Region
provider "aws" {
  alias  = "dr"
  region = "ap-northeast-1"
}

# RDS Global Cluster (Primary → DR Replication)
resource "aws_rds_global_cluster" "main" {
  global_cluster_identifier = "linebot-global"
  engine                    = "aurora-postgresql"
  engine_version           = "15.3"
  database_name            = "linebot"
  deletion_protection      = true
}

resource "aws_rds_cluster" "primary" {
  provider = aws.primary
  
  cluster_identifier      = "linebot-primary"
  engine                  = "aurora-postgresql"
  engine_version         = "15.3"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  
  master_username = var.db_username
  master_password = var.db_password
  
  backup_retention_period = 7
  preferred_backup_window = "02:00-03:00"
  
  deletion_protection = true
  
  db_subnet_group_name   = aws_db_subnet_group.primary.name
  vpc_security_group_ids = [aws_security_group.rds_primary.id]
}

resource "aws_rds_cluster" "dr" {
  provider = aws.dr
  
  cluster_identifier        = "linebot-dr"
  engine                    = "aurora-postgresql"
  engine_version           = "15.3"
  global_cluster_identifier = aws_rds_global_cluster.main.id
  
  # DR cluster ไม่ต้องมี credentials (inherited from global)
  
  db_subnet_group_name   = aws_db_subnet_group.dr.name
  vpc_security_group_ids = [aws_security_group.rds_dr.id]
}

# Route 53 Health Check และ Failover
resource "aws_route53_health_check" "primary" {
  fqdn              = aws_lb.primary.dns_name
  port              = 443
  type              = "HTTPS"
  resource_path     = "/health"
  failure_threshold = 3
  request_interval  = 10

  tags = {
    Name = "linebot-primary-health"
  }
}

resource "aws_route53_record" "api_primary" {
  zone_id = var.hosted_zone_id
  name    = "api.linebot.example.com"
  type    = "A"

  failover_routing_policy {
    type = "PRIMARY"
  }

  set_identifier  = "primary"
  health_check_id = aws_route53_health_check.primary.id

  alias {
    name                   = aws_lb.primary.dns_name
    zone_id                = aws_lb.primary.zone_id
    evaluate_target_health = true
  }
}

resource "aws_route53_record" "api_dr" {
  zone_id = var.hosted_zone_id
  name    = "api.linebot.example.com"
  type    = "A"

  failover_routing_policy {
    type = "SECONDARY"
  }

  set_identifier = "dr"

  alias {
    name                   = aws_lb.dr.dns_name
    zone_id                = aws_lb.dr.zone_id
    evaluate_target_health = true
  }
}
```

---

## 9. Multi-Datacenter Setup

### 9.1 Multi-DC Architecture

```
Multi-Datacenter Architecture:
┌─────────────────────────────────────────────────────────────┐
│                     Global Traffic Manager                   │
│                  (AWS Route53 / Cloudflare)                  │
└──────────────────────────┬──────────────────────────────────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
    ┌──────▼──────┐ ┌──────▼──────┐ ┌──────▼──────┐
    │  DC1        │ │  DC2        │ │  DC3        │
    │  Singapore  │ │   Tokyo     │ │   Bangkok   │
    │  (Primary)  │ │  (DR)       │ │  (Edge)     │
    │             │ │             │ │             │
    │  ┌────────┐ │ │  ┌────────┐ │ │  ┌────────┐ │
    │  │ App    │ │ │  │ App    │ │ │  │ App    │ │
    │  │ Cluster│ │ │  │ Cluster│ │ │  │ Cluster│ │
    │  └────────┘ │ │  └────────┘ │ │  └────────┘ │
    │  ┌────────┐ │ │  ┌────────┐ │ │  ┌────────┐ │
    │  │  DB    │ │ │  │  DB    │ │ │  │ Cache  │ │
    │  │Primary │─┼─┼─▶│Replica │ │ │  │ Only   │ │
    │  └────────┘ │ │  └────────┘ │ │  └────────┘ │
    └─────────────┘ └─────────────┘ └─────────────┘
           │               │               │
           └───────────────┼───────────────┘
                           │
                    ┌──────▼──────┐
                    │   Global    │
                    │   Kafka     │
                    │  (Mirror)   │
                    └─────────────┘
```

---

## 10. Cost Estimation for Enterprise

### 10.1 Monthly Cost Breakdown

```
Enterprise LINE Bot - Monthly Cost Estimate (USD)

┌────────────────────────┬──────────┬──────────┬──────────┐
│  Component             │  Small   │  Medium  │  Large   │
│                        │ 100k usr │ 1M users │ 10M usr  │
├────────────────────────┼──────────┼──────────┼──────────┤
│  Compute (EKS)         │          │          │          │
│  ├── App Nodes (m5.xl) │   $720   │  $2,880  │ $11,520  │
│  ├── Worker Nodes      │   $360   │  $1,440  │  $5,760  │
│  └── Kafka Nodes       │   $480   │  $1,920  │  $7,680  │
├────────────────────────┼──────────┼──────────┼──────────┤
│  Database              │          │          │          │
│  ├── RDS Aurora        │   $500   │  $2,000  │  $8,000  │
│  ├── ElastiCache Redis │   $200   │    $800  │  $3,200  │
│  └── Elasticsearch     │   $300   │  $1,200  │  $4,800  │
├────────────────────────┼──────────┼──────────┼──────────┤
│  Storage               │          │          │          │
│  ├── S3 (100GB/1TB/10T)│    $23   │    $230  │  $2,300  │
│  └── EBS Volumes       │   $200   │    $800  │  $3,200  │
├────────────────────────┼──────────┼──────────┼──────────┤
│  Networking            │          │          │          │
│  ├── Data Transfer     │   $100   │    $500  │  $2,000  │
│  ├── Load Balancer     │   $100   │    $300  │    $600  │
│  └── CloudFront CDN    │    $50   │    $200  │    $800  │
├────────────────────────┼──────────┼──────────┼──────────┤
│  Security & Compliance │          │          │          │
│  ├── WAF               │   $100   │    $300  │  $1,000  │
│  ├── Shield Advanced   │  $3,000  │  $3,000  │  $3,000  │
│  └── KMS               │    $10   │     $50  │    $200  │
├────────────────────────┼──────────┼──────────┼──────────┤
│  Monitoring            │          │          │          │
│  ├── CloudWatch        │   $100   │    $400  │  $1,600  │
│  └── Datadog (optional)│   $500   │  $2,000  │  $8,000  │
├────────────────────────┼──────────┼──────────┼──────────┤
│  LINE API Costs        │          │          │          │
│  ├── Messaging API     │  $0-200  │ $200-500 │$500-2000 │
│  └── OA Premium        │   $500   │  $1,000  │  $2,000  │
├────────────────────────┼──────────┼──────────┼──────────┤
│  TOTAL (Approx)        │ ~$6,743  │ ~$19,020 │ ~$65,660 │
└────────────────────────┴──────────┴──────────┴──────────┘

Cost Optimization Tips:
- ใช้ Reserved Instances: ประหยัด 30-40%
- Spot Instances สำหรับ batch jobs: ประหยัด 60-70%
- Auto-scaling ตาม traffic: ประหยัด 20-30%
- Multi-tier storage: ย้ายข้อมูลเก่าไป S3 Glacier
```

---

## 11. Complete Enterprise Starter Kit

### 11.1 Project Structure

```
enterprise-line-bot/
├── apps/
│   ├── api-gateway/         # Kong configuration
│   ├── message-service/     # LINE message handling
│   ├── user-service/        # User management
│   ├── campaign-service/    # Marketing campaigns
│   ├── notification-service/# Push notifications
│   └── analytics-service/  # Analytics processing
├── packages/
│   ├── shared-types/        # TypeScript types
│   ├── shared-utils/        # Common utilities
│   ├── line-client/         # LINE API wrapper
│   └── event-bus/           # Kafka wrapper
├── infrastructure/
│   ├── terraform/           # IaC
│   ├── kubernetes/          # K8s manifests
│   ├── helm/                # Helm charts
│   └── scripts/             # Deployment scripts
├── monitoring/
│   ├── prometheus/          # Metrics
│   ├── grafana/             # Dashboards
│   └── alertmanager/        # Alerts
├── docs/
│   ├── architecture/        # Architecture docs
│   ├── api/                 # API documentation
│   └── runbooks/            # Operational runbooks
├── .github/
│   └── workflows/           # CI/CD pipelines
└── package.json
```

### 11.2 Kubernetes Deployment

```yaml
# kubernetes/message-service/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: message-service
  namespace: linebot
  labels:
    app: message-service
    version: "1.0.0"
    tier: backend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: message-service
  template:
    metadata:
      labels:
        app: message-service
        version: "1.0.0"
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9090"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: message-service
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - message-service
            topologyKey: kubernetes.io/hostname
      
      containers:
      - name: message-service
        image: gcr.io/myproject/message-service:1.0.0
        imagePullPolicy: Always
        
        ports:
        - containerPort: 3000
          name: http
        - containerPort: 9090
          name: metrics
        
        env:
        - name: NODE_ENV
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: url
        - name: LINE_CHANNEL_ACCESS_TOKEN
          valueFrom:
            secretKeyRef:
              name: line-credentials
              key: access-token
        - name: LINE_CHANNEL_SECRET
          valueFrom:
            secretKeyRef:
              name: line-credentials
              key: channel-secret
        - name: KAFKA_BROKERS
          valueFrom:
            configMapKeyRef:
              name: kafka-config
              key: brokers
        
        resources:
          requests:
            cpu: 250m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi
        
        livenessProbe:
          httpGet:
            path: /health/live
            port: 3000
          initialDelaySeconds: 15
          periodSeconds: 10
          failureThreshold: 3
        
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5
          failureThreshold: 3
        
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
      
      volumes:
      - name: tmp
        emptyDir: {}
      
      terminationGracePeriodSeconds: 30

---
apiVersion: v1
kind: Service
metadata:
  name: message-service
  namespace: linebot
spec:
  selector:
    app: message-service
  ports:
  - port: 80
    targetPort: 3000
    name: http
  - port: 9090
    targetPort: 9090
    name: metrics

---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: message-service-hpa
  namespace: linebot
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: message-service
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 3
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 120
```

### 11.3 CI/CD Pipeline

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    branches: [main]
    tags: ['v*']

env:
  REGISTRY: gcr.io
  PROJECT_ID: ${{ secrets.GCP_PROJECT_ID }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - run: npm run test:unit
      - run: npm run test:integration
      - run: npm run test:e2e
      
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          format: 'sarif'
          output: 'trivy-results.sarif'
      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
      
  build-and-push:
    needs: [test, security-scan]
    runs-on: ubuntu-latest
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
    steps:
      - uses: actions/checkout@v4
      
      - name: Authenticate to Google Cloud
        uses: google-github-actions/auth@v1
        with:
          credentials_json: ${{ secrets.GCP_SA_KEY }}
      
      - name: Set up Cloud SDK
        uses: google-github-actions/setup-gcloud@v1
      
      - name: Configure Docker for GCR
        run: gcloud auth configure-docker
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/message-service
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix={{branch}}-
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./apps/message-service
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=registry,ref=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/message-service:cache
          cache-to: type=registry,ref=${{ env.REGISTRY }}/${{ env.PROJECT_ID }}/message-service:cache,mode=max
      
  deploy-staging:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Staging
        uses: google-github-actions/get-gke-credentials@v1
        with:
          cluster_name: linebot-staging
          location: asia-southeast1
      
      - name: Update deployment
        run: |
          kubectl set image deployment/message-service \
            message-service=${{ needs.build-and-push.outputs.image-tag }} \
            -n linebot
          kubectl rollout status deployment/message-service -n linebot
      
      - name: Run smoke tests
        run: npm run test:smoke -- --env=staging
      
  deploy-production:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Production with Canary
        run: |
          # Deploy canary (10% traffic)
          kubectl apply -f kubernetes/message-service/canary.yaml
          kubectl set image deployment/message-service-canary \
            message-service=${{ needs.build-and-push.outputs.image-tag }} \
            -n linebot
          
          # Wait and check metrics
          sleep 300
          npm run check-canary-metrics
          
          # Promote to full production
          kubectl set image deployment/message-service \
            message-service=${{ needs.build-and-push.outputs.image-tag }} \
            -n linebot
          kubectl rollout status deployment/message-service -n linebot
          
          # Remove canary
          kubectl delete -f kubernetes/message-service/canary.yaml
```

---

## 12. Performance Benchmarks

```
Performance Benchmarks - Enterprise LINE Bot

Test Setup:
- 10,000 concurrent users
- Message rate: 50,000 msg/min
- Infrastructure: 10 x m5.xlarge nodes

Results:

┌──────────────────────┬──────────┬──────────┬──────────┐
│  Metric              │  p50     │  p95     │  p99     │
├──────────────────────┼──────────┼──────────┼──────────┤
│  Webhook Processing  │  45ms    │  120ms   │  185ms   │
│  Message Response    │  80ms    │  200ms   │  350ms   │
│  User Profile Fetch  │  12ms    │  35ms    │  75ms    │
│  DB Query (indexed)  │  5ms     │  15ms    │  30ms    │
│  Cache Hit           │  0.5ms   │  2ms     │  5ms     │
│  Kafka Produce       │  3ms     │  10ms    │  20ms    │
└──────────────────────┴──────────┴──────────┴──────────┘

Throughput:
├── Messages processed: 55,000/min
├── API requests: 120,000/min  
├── Kafka events: 200,000/min
└── Cache hit rate: 92%

Error Rates:
├── 5xx errors: < 0.01%
├── Timeout errors: < 0.05%
└── Retry success: 98%
```

---

## 13. Monitoring Strategy

### 13.1 Observability Stack

```yaml
# monitoring/prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'linebot-prod'
    env: 'production'

rule_files:
  - "alert_rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: 'message-service'
    kubernetes_sd_configs:
      - role: pod
        namespaces:
          names: ['linebot']
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        action: replace
        target_label: __metrics_path__
        regex: (.+)
      - source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        regex: ([^:]+)(?::\d+)?;(\d+)
        replacement: $1:$2
        target_label: __address__

  - job_name: 'kafka'
    static_configs:
      - targets: ['kafka-exporter:9308']

  - job_name: 'postgresql'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

### 13.2 Alert Rules

```yaml
# monitoring/prometheus/alert_rules/critical.yml
groups:
- name: linebot-critical
  interval: 30s
  rules:
  
  - alert: HighErrorRate
    expr: |
      rate(http_requests_total{status=~"5.."}[5m]) / 
      rate(http_requests_total[5m]) > 0.01
    for: 5m
    labels:
      severity: critical
      team: backend
    annotations:
      summary: "High error rate detected"
      description: "Error rate is {{ $value | humanizePercentage }} for {{ $labels.service }}"
      runbook: "https://runbooks.internal/high-error-rate"
  
  - alert: SlowResponseTime
    expr: |
      histogram_quantile(0.99, 
        rate(http_request_duration_seconds_bucket[5m])
      ) > 0.5
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "Slow response time (p99 > 500ms)"
      description: "p99 latency is {{ $value }}s for {{ $labels.service }}"
  
  - alert: KafkaConsumerLag
    expr: kafka_consumer_group_lag > 10000
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Kafka consumer lag is high"
      description: "Consumer group {{ $labels.consumergroup }} lag: {{ $value }}"
  
  - alert: DatabaseConnectionPoolExhausted
    expr: |
      pg_stat_activity_count{state="active"} / 
      pg_settings_max_connections > 0.8
    for: 2m
    labels:
      severity: critical
    annotations:
      summary: "Database connection pool nearly exhausted"
```

---

## สรุป

Enterprise LINE Bot Architecture ต้องการการวางแผนอย่างละเอียดในหลายด้าน:

1. **Architecture Patterns**: เลือก Pattern ที่เหมาะสมกับ scale และ complexity
2. **Event Sourcing + CQRS**: ช่วยให้ระบบ audit trail สมบูรณ์และ performance ดี
3. **Saga Pattern**: จัดการ distributed transactions อย่างปลอดภัย
4. **Security**: Multi-layer security ครอบคลุมทุกด้าน
5. **DR**: มี Disaster Recovery Plan ที่ชัดเจน
6. **Monitoring**: Observability stack ที่สมบูรณ์

การเริ่มต้นจาก Monolith แล้วค่อยๆ migrate ไป Microservices เป็นแนวทางที่แนะนำ เพราะลด risk และทำให้ team เรียนรู้ไปพร้อมกัน

---

*จบ Part 81: Enterprise LINE Bot Architecture*

---

## 14. API Gateway Configuration

### 14.1 Kong Gateway Setup

```yaml
# infrastructure/kong/kong.yaml
_format_version: "3.0"
_transform: true

services:
  - name: message-service
    url: http://message-service:3000
    plugins:
      - name: rate-limiting
        config:
          minute: 1000
          hour: 50000
          policy: redis
          redis_host: redis
      - name: jwt
        config:
          secret_is_base64: false
      - name: prometheus
    routes:
      - name: webhook
        paths:
          - /webhook
        methods:
          - POST
        strip_path: false

  - name: user-service
    url: http://user-service:3001
    plugins:
      - name: rate-limiting
        config:
          minute: 500
    routes:
      - name: users-api
        paths:
          - /api/users
        methods:
          - GET
          - POST
          - PUT
          - DELETE

  - name: campaign-service
    url: http://campaign-service:3002
    routes:
      - name: campaigns-api
        paths:
          - /api/campaigns

plugins:
  - name: cors
    config:
      origins:
        - "*"
      methods:
        - GET
        - POST
        - PUT
        - DELETE
        - OPTIONS
      headers:
        - Accept
        - Authorization
        - Content-Type
        - X-Line-Signature
      exposed_headers:
        - X-Auth-Token
      credentials: true
      max_age: 3600

  - name: request-size-limiting
    config:
      allowed_payload_size: 1
      size_unit: megabytes

  - name: response-transformer
    config:
      add:
        headers:
          - "X-Content-Type-Options: nosniff"
          - "X-Frame-Options: DENY"
          - "Strict-Transport-Security: max-age=31536000; includeSubDomains"
```

### 14.2 Service Mesh with Istio

```yaml
# infrastructure/istio/virtual-service.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: message-service
  namespace: linebot
spec:
  hosts:
    - message-service
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: message-service
            subset: canary
          weight: 100
    - route:
        - destination:
            host: message-service
            subset: stable
          weight: 90
        - destination:
            host: message-service
            subset: canary
          weight: 10
      retries:
        attempts: 3
        perTryTimeout: 2s
        retryOn: 5xx,gateway-error,connect-failure
      timeout: 10s

---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: message-service
  namespace: linebot
spec:
  host: message-service
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 100
      http:
        h2UpgradePolicy: UPGRADE
        http2MaxRequests: 1000
        maxRequestsPerConnection: 100
    loadBalancer:
      simple: LEAST_CONN
    outlierDetection:
      consecutiveGatewayErrors: 5
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
  subsets:
    - name: stable
      labels:
        version: stable
    - name: canary
      labels:
        version: canary

---
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: linebot
spec:
  mtls:
    mode: STRICT  # Require mTLS for all service-to-service communication
```

---

## 15. Database Schema Design

### 15.1 Core Tables

```sql
-- database/migrations/001_create_core_tables.sql

-- Users table
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    line_user_id VARCHAR(255) UNIQUE NOT NULL,
    display_name VARCHAR(500),
    picture_url TEXT,
    status VARCHAR(50) DEFAULT 'active',
    is_following BOOLEAN DEFAULT FALSE,
    followed_at TIMESTAMPTZ,
    unfollowed_at TIMESTAMPTZ,
    tags TEXT[] DEFAULT '{}',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_line_user_id ON users(line_user_id);
CREATE INDEX idx_users_status ON users(status);
CREATE INDEX idx_users_is_following ON users(is_following);
CREATE INDEX idx_users_created_at ON users(created_at DESC);
CREATE INDEX idx_users_tags ON users USING GIN(tags);

-- Messages table
CREATE TABLE messages (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    direction VARCHAR(10) NOT NULL, -- 'inbound' or 'outbound'
    message_type VARCHAR(50) NOT NULL,
    message_content JSONB NOT NULL,
    reply_token VARCHAR(500),
    line_message_id VARCHAR(255),
    intent VARCHAR(100),
    sentiment FLOAT,
    processed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_messages_user_id ON messages(user_id);
CREATE INDEX idx_messages_created_at ON messages(created_at DESC);
CREATE INDEX idx_messages_intent ON messages(intent);

-- Sessions table
CREATE TABLE user_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id) ON DELETE CASCADE,
    started_at TIMESTAMPTZ NOT NULL,
    ended_at TIMESTAMPTZ,
    message_count INT DEFAULT 0,
    context JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_sessions_user_id ON user_sessions(user_id);
CREATE INDEX idx_sessions_started_at ON user_sessions(started_at DESC);

-- Events table
CREATE TABLE events (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_id VARCHAR(255) NOT NULL,
    aggregate_type VARCHAR(100) NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    metadata JSONB DEFAULT '{}',
    version INTEGER NOT NULL,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(aggregate_id, version)
);

CREATE INDEX idx_events_aggregate_id ON events(aggregate_id);
CREATE INDEX idx_events_event_type ON events(event_type);
CREATE INDEX idx_events_created_at ON events(created_at DESC);

-- Audit Log
CREATE TABLE audit_logs (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID,
    line_user_id VARCHAR(255),
    action VARCHAR(255) NOT NULL,
    resource_type VARCHAR(100),
    resource_id VARCHAR(255),
    ip_address INET,
    user_agent TEXT,
    request_id VARCHAR(255),
    status_code INT,
    duration_ms INT,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_user_id ON audit_logs(user_id);
CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at DESC);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW
    EXECUTE FUNCTION update_updated_at_column();
```

---

## 16. Compliance Architecture (PDPA/ISO 27001)

```javascript
// src/compliance/ComplianceManager.js

class ComplianceManager {
  constructor({ db, auditLogger, encryptionService }) {
    this.db = db;
    this.auditLogger = auditLogger;
    this.encryption = encryptionService;
    
    // Personal data fields ที่ต้อง protect
    this.personalDataFields = [
      'displayName', 'pictureUrl', 'phoneNumber',
      'email', 'address', 'lineUserId',
    ];
  }
  
  // Data Subject Access Request (DSAR)
  async handleDataAccessRequest(lineUserId) {
    const userData = await this.collectAllUserData(lineUserId);
    
    // Log the access request
    await this.auditLogger.log({
      action: 'DSAR_ACCESS',
      lineUserId,
      timestamp: new Date().toISOString(),
    });
    
    return this.prepareDataExport(userData);
  }
  
  async collectAllUserData(lineUserId) {
    const [user, messages, purchases, sessions] = await Promise.all([
      this.db.query('SELECT * FROM users WHERE line_user_id = $1', [lineUserId]),
      this.db.query('SELECT * FROM messages WHERE user_id IN (SELECT id FROM users WHERE line_user_id = $1)', [lineUserId]),
      this.db.query('SELECT * FROM purchases WHERE user_id IN (SELECT id FROM users WHERE line_user_id = $1)', [lineUserId]),
      this.db.query('SELECT * FROM user_sessions WHERE user_id IN (SELECT id FROM users WHERE line_user_id = $1)', [lineUserId]),
    ]);
    
    return {
      profile: user.rows[0] || null,
      messages: messages.rows,
      purchases: purchases.rows,
      sessions: sessions.rows,
      exportedAt: new Date().toISOString(),
    };
  }
  
  prepareDataExport(data) {
    // Mask sensitive data before export
    if (data.profile) {
      data.profile = this.maskSensitiveFields(data.profile);
    }
    return data;
  }
  
  maskSensitiveFields(obj) {
    const masked = { ...obj };
    if (masked.picture_url) {
      masked.picture_url = '[REDACTED - Profile Image URL]';
    }
    return masked;
  }
  
  // Data Retention Policy
  async enforceRetentionPolicy() {
    const retentionPolicies = [
      { table: 'messages', field: 'created_at', retainDays: 365 },
      { table: 'audit_logs', field: 'created_at', retainDays: 730 },
      { table: 'user_sessions', field: 'created_at', retainDays: 90 },
    ];
    
    for (const policy of retentionPolicies) {
      const result = await this.db.query(
        `DELETE FROM ${policy.table} 
         WHERE ${policy.field} < NOW() - INTERVAL '${policy.retainDays} days'
         RETURNING id`
      );
      
      console.log(`Deleted ${result.rowCount} records from ${policy.table}`);
    }
  }
  
  // Breach Notification (72-hour requirement)
  async handleDataBreach(breachDetails) {
    const breach = {
      id: require('crypto').randomUUID(),
      detectedAt: new Date().toISOString(),
      affectedUsers: breachDetails.affectedUsers,
      dataTypes: breachDetails.dataTypes,
      severity: breachDetails.severity,
      description: breachDetails.description,
      status: 'detected',
    };
    
    // Record breach
    await this.db.query(
      'INSERT INTO data_breaches (id, details, detected_at) VALUES ($1, $2, NOW())',
      [breach.id, JSON.stringify(breach)]
    );
    
    // Notify PDPA authority within 72 hours
    await this.schedulePDPANotification(breach);
    
    // Notify affected users
    if (breach.severity === 'high') {
      await this.notifyAffectedUsers(breach);
    }
    
    return breach;
  }
  
  async schedulePDPANotification(breach) {
    const deadline = new Date(breach.detectedAt);
    deadline.setHours(deadline.getHours() + 72);
    
    await this.db.query(
      `INSERT INTO scheduled_tasks 
       (task_type, payload, scheduled_at, created_at)
       VALUES ('pdpa_notification', $1, $2, NOW())`,
      [JSON.stringify(breach), deadline.toISOString()]
    );
    
    console.log(`PDPA notification scheduled for: ${deadline.toISOString()}`);
  }
}

module.exports = ComplianceManager;
```

---

## 17. Load Testing

```javascript
// tests/load/k6-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate } from 'k6/metrics';

const errorRate = new Rate('errors');

export const options = {
  stages: [
    { duration: '2m', target: 100 },   // Ramp up
    { duration: '5m', target: 1000 },  // Stay at 1000 users
    { duration: '2m', target: 2000 },  // Peak load
    { duration: '5m', target: 2000 },  // Stay at peak
    { duration: '2m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(99)<500'],   // 99th percentile < 500ms
    http_req_failed: ['rate<0.01'],     // Error rate < 1%
    errors: ['rate<0.01'],
  },
};

const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';

// Simulate LINE Webhook payload
function createWebhookPayload(userId) {
  return JSON.stringify({
    destination: 'Utest123',
    events: [{
      type: 'message',
      mode: 'active',
      timestamp: Date.now(),
      source: {
        type: 'user',
        userId: userId,
      },
      replyToken: `reply-${Math.random().toString(36).substr(2, 9)}`,
      message: {
        type: 'text',
        id: `msg-${Date.now()}`,
        text: 'สวัสดีครับ',
      },
    }],
  });
}

function generateSignature(body, secret) {
  // In real test, compute actual HMAC-SHA256
  return 'test-signature';
}

export default function() {
  const userId = `U${Math.random().toString(36).substr(2, 32)}`;
  const payload = createWebhookPayload(userId);
  
  const params = {
    headers: {
      'Content-Type': 'application/json',
      'X-Line-Signature': generateSignature(payload, 'secret'),
    },
    timeout: '10s',
  };
  
  const res = http.post(`${BASE_URL}/webhook`, payload, params);
  
  const success = check(res, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'has ok response': (r) => {
      try {
        const body = JSON.parse(r.body);
        return body.status === 'ok';
      } catch {
        return false;
      }
    },
  });
  
  errorRate.add(!success);
  
  sleep(Math.random() * 2 + 0.5); // Random sleep 0.5-2.5s
}

export function handleSummary(data) {
  return {
    'reports/load-test-summary.json': JSON.stringify(data, null, 2),
    stdout: `
=== Load Test Results ===
Total Requests: ${data.metrics.http_reqs.values.count}
Request Rate: ${data.metrics.http_reqs.values.rate.toFixed(2)} req/s
P50 Duration: ${data.metrics.http_req_duration.values['p(50)'].toFixed(2)}ms
P95 Duration: ${data.metrics.http_req_duration.values['p(95)'].toFixed(2)}ms
P99 Duration: ${data.metrics.http_req_duration.values['p(99)'].toFixed(2)}ms
Error Rate: ${(data.metrics.http_req_failed.values.rate * 100).toFixed(2)}%
`,
  };
}
```

---

## 18. Operational Runbooks

### 18.1 Production Incident Response

```markdown
# Runbook: High Error Rate

## Symptoms
- Error rate > 1% for > 5 minutes
- PagerDuty alert triggered
- Grafana shows red status

## Immediate Actions (< 5 minutes)

1. Check service health:
   kubectl get pods -n linebot
   kubectl describe deployment message-service -n linebot

2. Check logs:
   kubectl logs -n linebot -l app=message-service --tail=100

3. Check database connectivity:
   kubectl exec -n linebot deploy/message-service -- node -e "
   const { Pool } = require('pg');
   const pool = new Pool({ connectionString: process.env.DATABASE_URL });
   pool.query('SELECT 1').then(() => console.log('DB OK')).catch(console.error);
   "

4. Check Kafka consumer lag:
   kubectl exec -n kafka kafka-0 -- kafka-consumer-groups.sh \
     --bootstrap-server localhost:9092 \
     --describe --group message-service-consumer

## Escalation
- 15 min: Notify Engineering Lead
- 30 min: Notify CTO
- 1 hour: Consider rollback

## Rollback Procedure
   kubectl rollout undo deployment/message-service -n linebot
   kubectl rollout status deployment/message-service -n linebot
```

---

## สรุปสมบูรณ์

Enterprise LINE Bot Architecture ที่ครอบคลุมทุกด้านสำหรับองค์กรขนาดใหญ่:

| Component | Technology | Purpose |
|-----------|-----------|---------|
| API Gateway | Kong | Rate limiting, Auth, Routing |
| Service Mesh | Istio | mTLS, Traffic control |
| App Layer | Fastify + Node.js | High-performance webhook |
| Message Queue | Apache Kafka | Async event processing |
| Primary DB | PostgreSQL + Aurora | ACID transactions |
| Cache | Redis Cluster | Session, rate limiting |
| Search | Elasticsearch | Full-text search |
| Observability | Prometheus + Grafana | Metrics, Alerting |
| Security | HashiCorp Vault | Secrets management |
| Container | Kubernetes (EKS/GKE) | Orchestration |
| Compliance | Custom + PDPA tools | Thai data protection |

---

*จบ Part 81: Enterprise LINE Bot Architecture (ฉบับสมบูรณ์)*
