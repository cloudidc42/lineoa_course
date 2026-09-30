# Part 71: Microservices Architecture สำหรับ LINE Bot Systems

## บทนำ

การออกแบบระบบ LINE Bot ในระดับ Production ที่รองรับผู้ใช้หลักล้านคน จำเป็นต้องใช้สถาปัตยกรรม Microservices เพื่อให้ระบบมีความยืดหยุ่น ปรับขนาดได้ และบำรุงรักษาง่าย

บทนี้จะครอบคลุม:
- เมื่อไหร่ควรแยกเป็น Microservices
- การออกแบบ Service สำหรับ LINE Bot
- API Gateway
- การสื่อสารระหว่าง Services
- Docker Compose Setup
- Distributed Tracing
- Inter-service Authentication

---

## 1. เมื่อไหร่ควรแยกเป็น Microservices

### 1.1 Monolith vs Microservices

```
┌─────────────────────────────────────────────────┐
│              MONOLITH ARCHITECTURE               │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │                                            │  │
│  │  Webhook + Handler + User + Notification  │  │
│  │  + Analytics + Scheduler + Payment        │  │
│  │                                            │  │
│  └────────────────────────────────────────────┘  │
│                        │                         │
│                   ┌────┴────┐                    │
│                   │Database │                    │
│                   └─────────┘                    │
└─────────────────────────────────────────────────┘

                          VS

┌─────────────────────────────────────────────────┐
│            MICROSERVICES ARCHITECTURE            │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │ Webhook  │  │ Message  │  │    User      │  │
│  │ Receiver │  │ Handler  │  │   Service    │  │
│  └────┬─────┘  └────┬─────┘  └──────┬───────┘  │
│       │              │               │           │
│  ┌────┴─────────────────────────────┴───────┐   │
│  │              Message Queue               │   │
│  └───────────────────────────────────────────┘  │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────────┐  │
│  │Notifica- │  │Analytics │  │  Scheduler   │  │
│  │  tion    │  │ Service  │  │   Service    │  │
│  └──────────┘  └──────────┘  └──────────────┘  │
└─────────────────────────────────────────────────┘
```

### 1.2 เมื่อไหร่ควรใช้ Microservices

**ใช้ Monolith เมื่อ:**
- ทีมขนาดเล็ก (1-5 คน)
- MVP หรือ Prototype
- Traffic ต่ำ (< 1,000 users)
- Business logic ยังไม่ชัดเจน

**เปลี่ยนเป็น Microservices เมื่อ:**
- ทีมใหญ่ขึ้น (5+ คน)
- มี Traffic สูง (100,000+ messages/day)
- ต้องการ Scale เฉพาะส่วน
- Deployment บ่อยครั้งและเป็นอิสระ
- มีหลาย Technology Stack

### 1.3 Pain Points ของ LINE Bot Monolith

```javascript
// ❌ ปัญหาของ Monolith
// 1. Webhook รับ message เยอะมาก → CPU พุ่ง → ส่ง reply ช้า
// 2. Report generation ใช้ CPU สูง → ทำให้ webhook response ช้า
// 3. Notification broadcast → ทำให้ระบบทั้งหมดช้า
// 4. Deploy ส่วนหนึ่ง → ต้อง down ทั้งระบบ
```

---

## 2. Service Decomposition สำหรับ LINE Bot

### 2.1 Overview Architecture

```
                         LINE Platform
                              │
                              ▼
                    ┌─────────────────┐
                    │   API Gateway   │
                    │  (Kong/Express) │
                    └────────┬────────┘
                             │
             ┌───────────────┼───────────────┐
             │               │               │
             ▼               ▼               ▼
    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
    │   Webhook    │ │    LIFF      │ │   Admin      │
    │   Receiver   │ │   Service    │ │   Service    │
    │  :3001       │ │  :3002       │ │  :3003       │
    └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
           │                │                │
           ▼                ▼                │
    ┌─────────────────────────────┐          │
    │      Message Queue          │          │
    │       (Redis/RabbitMQ)      │          │
    └──────────┬──────────────────┘          │
               │                             │
    ┌──────────┼──────────┐                  │
    │          │          │                  │
    ▼          ▼          ▼                  ▼
┌────────┐ ┌────────┐ ┌────────┐    ┌──────────────┐
│Message │ │  User  │ │Notifi- │    │  Analytics   │
│Handler │ │Service │ │cation  │    │   Service    │
│:3004   │ │:3005   │ │:3006   │    │  :3007       │
└────────┘ └────────┘ └────────┘    └──────────────┘
    │           │          │               │
    └───────────┴──────────┴───────────────┘
                           │
                    ┌──────┴──────┐
                    │  Database   │
                    │  Cluster    │
                    └─────────────┘
```

---

## 3. Webhook Receiver Service

### 3.1 โครงสร้าง Webhook Service

```
webhook-receiver/
├── src/
│   ├── index.js
│   ├── routes/
│   │   └── webhook.js
│   ├── middleware/
│   │   ├── validateSignature.js
│   │   └── rateLimiter.js
│   ├── publishers/
│   │   └── messagePublisher.js
│   └── health/
│       └── healthCheck.js
├── Dockerfile
├── package.json
└── .env
```

### 3.2 Webhook Receiver Implementation

```javascript
// webhook-receiver/src/index.js
const express = require('express');
const helmet = require('helmet');
const compression = require('compression');
const { validateSignature } = require('./middleware/validateSignature');
const { rateLimiter } = require('./middleware/rateLimiter');
const webhookRoutes = require('./routes/webhook');
const { healthCheck } = require('./health/healthCheck');
const logger = require('./utils/logger');

const app = express();
const PORT = process.env.PORT || 3001;

// Security middleware
app.use(helmet());
app.use(compression());

// Raw body parser สำหรับ LINE signature validation
app.use('/webhook', express.raw({ type: 'application/json' }));
app.use(express.json());

// Middleware
app.use(rateLimiter);

// Routes
app.get('/health', healthCheck);
app.post('/webhook', validateSignature, webhookRoutes);

// Error handler
app.use((err, req, res, next) => {
  logger.error('Unhandled error:', err);
  res.status(500).json({ error: 'Internal Server Error' });
});

app.listen(PORT, () => {
  logger.info(`Webhook Receiver running on port ${PORT}`);
});

module.exports = app;
```

```javascript
// webhook-receiver/src/middleware/validateSignature.js
const crypto = require('crypto');
const logger = require('../utils/logger');

const validateSignature = (req, res, next) => {
  const channelSecret = process.env.LINE_CHANNEL_SECRET;
  const signature = req.headers['x-line-signature'];

  if (!signature) {
    logger.warn('Missing LINE signature');
    return res.status(401).json({ error: 'Unauthorized' });
  }

  // คำนวณ HMAC-SHA256
  const body = req.body; // raw buffer
  const hash = crypto
    .createHmac('SHA256', channelSecret)
    .update(body)
    .digest('base64');

  if (hash !== signature) {
    logger.warn('Invalid LINE signature', {
      expected: hash,
      received: signature,
      ip: req.ip
    });
    return res.status(401).json({ error: 'Invalid signature' });
  }

  // Parse body หลังจาก validate
  req.body = JSON.parse(body.toString());
  next();
};

module.exports = { validateSignature };
```

```javascript
// webhook-receiver/src/middleware/rateLimiter.js
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const Redis = require('ioredis');

const redis = new Redis({
  host: process.env.REDIS_HOST || 'redis',
  port: process.env.REDIS_PORT || 6379,
});

const rateLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 10000, // สูงสุด 10,000 requests ต่อนาทีต่อ IP
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args) => redis.call(...args),
  }),
  message: {
    error: 'Too many requests',
    retryAfter: 60
  }
});

module.exports = { rateLimiter };
```

```javascript
// webhook-receiver/src/routes/webhook.js
const express = require('express');
const router = express.Router();
const { publishMessage } = require('../publishers/messagePublisher');
const logger = require('../utils/logger');

router.post('/', async (req, res) => {
  // ตอบ LINE Platform ก่อนเลย (ต้องตอบใน 1 วินาที)
  res.status(200).json({ status: 'ok' });

  const { events, destination } = req.body;

  if (!events || events.length === 0) {
    return;
  }

  logger.info(`Received ${events.length} events from LINE`);

  // Publish แต่ละ event ไปยัง queue
  const publishPromises = events.map(event => 
    publishMessage({
      event,
      destination,
      receivedAt: new Date().toISOString()
    })
  );

  try {
    await Promise.allSettled(publishPromises);
  } catch (error) {
    logger.error('Error publishing events:', error);
  }
});

module.exports = router;
```

```javascript
// webhook-receiver/src/publishers/messagePublisher.js
const amqp = require('amqplib');
const logger = require('../utils/logger');

let channel = null;
let connection = null;

const EXCHANGES = {
  WEBHOOK_EVENTS: 'webhook.events',
};

const ROUTING_KEYS = {
  MESSAGE: 'event.message',
  FOLLOW: 'event.follow',
  UNFOLLOW: 'event.unfollow',
  POSTBACK: 'event.postback',
  JOIN: 'event.join',
  LEAVE: 'event.leave',
};

async function getChannel() {
  if (channel) return channel;

  const url = process.env.RABBITMQ_URL || 'amqp://rabbitmq:5672';
  
  connection = await amqp.connect(url);
  connection.on('error', (err) => {
    logger.error('RabbitMQ connection error:', err);
    channel = null;
    connection = null;
  });

  channel = await connection.createChannel();
  
  // Declare exchange
  await channel.assertExchange(EXCHANGES.WEBHOOK_EVENTS, 'topic', {
    durable: true
  });

  logger.info('Connected to RabbitMQ');
  return channel;
}

async function publishMessage(payload) {
  try {
    const ch = await getChannel();
    const { event } = payload;

    // กำหนด routing key ตาม event type
    const routingKey = ROUTING_KEYS[event.type.toUpperCase()] || `event.${event.type}`;

    const message = Buffer.from(JSON.stringify(payload));
    
    ch.publish(
      EXCHANGES.WEBHOOK_EVENTS,
      routingKey,
      message,
      {
        persistent: true,
        contentType: 'application/json',
        timestamp: Date.now(),
        messageId: `${event.source?.userId}-${Date.now()}`,
      }
    );

    logger.debug(`Published event: ${routingKey}`, {
      userId: event.source?.userId,
      eventType: event.type
    });

  } catch (error) {
    logger.error('Failed to publish message:', error);
    throw error;
  }
}

module.exports = { publishMessage };
```

---

## 4. Message Handler Service

### 4.1 โครงสร้าง Message Handler

```
message-handler/
├── src/
│   ├── index.js
│   ├── consumers/
│   │   └── eventConsumer.js
│   ├── handlers/
│   │   ├── messageHandler.js
│   │   ├── postbackHandler.js
│   │   ├── followHandler.js
│   │   └── unfollowHandler.js
│   ├── services/
│   │   ├── lineApiService.js
│   │   ├── userServiceClient.js
│   │   └── analyticsClient.js
│   └── utils/
│       └── logger.js
├── Dockerfile
└── package.json
```

```javascript
// message-handler/src/index.js
const { startConsumer } = require('./consumers/eventConsumer');
const logger = require('./utils/logger');

async function main() {
  logger.info('Message Handler Service starting...');
  
  try {
    await startConsumer();
    logger.info('Message Handler Service started successfully');
  } catch (error) {
    logger.error('Failed to start Message Handler:', error);
    process.exit(1);
  }
}

// Graceful shutdown
process.on('SIGTERM', async () => {
  logger.info('Received SIGTERM, shutting down gracefully');
  process.exit(0);
});

process.on('SIGINT', async () => {
  logger.info('Received SIGINT, shutting down gracefully');
  process.exit(0);
});

main();
```

```javascript
// message-handler/src/consumers/eventConsumer.js
const amqp = require('amqplib');
const { handleMessage } = require('../handlers/messageHandler');
const { handlePostback } = require('../handlers/postbackHandler');
const { handleFollow } = require('../handlers/followHandler');
const { handleUnfollow } = require('../handlers/unfollowHandler');
const logger = require('../utils/logger');

const EXCHANGE = 'webhook.events';
const QUEUE = 'message.handler.queue';

const HANDLERS = {
  'event.message': handleMessage,
  'event.postback': handlePostback,
  'event.follow': handleFollow,
  'event.unfollow': handleUnfollow,
};

async function startConsumer() {
  const url = process.env.RABBITMQ_URL || 'amqp://rabbitmq:5672';
  
  const connection = await amqp.connect(url);
  const channel = await connection.createChannel();

  // Prefetch - รับ message ทีละ 10 ข้อความ
  channel.prefetch(10);

  // Declare exchange
  await channel.assertExchange(EXCHANGE, 'topic', { durable: true });

  // Declare queue
  await channel.assertQueue(QUEUE, {
    durable: true,
    arguments: {
      'x-dead-letter-exchange': 'webhook.events.dlx',
      'x-dead-letter-routing-key': 'dead.letter',
    }
  });

  // Bind queue กับ routing keys ที่ต้องการ
  const routingKeys = Object.keys(HANDLERS);
  for (const routingKey of routingKeys) {
    await channel.bindQueue(QUEUE, EXCHANGE, routingKey);
    logger.info(`Bound queue to routing key: ${routingKey}`);
  }

  // Start consuming
  channel.consume(QUEUE, async (msg) => {
    if (!msg) return;

    const routingKey = msg.fields.routingKey;
    const handler = HANDLERS[routingKey];

    try {
      const payload = JSON.parse(msg.content.toString());
      
      logger.info(`Processing event: ${routingKey}`, {
        userId: payload.event?.source?.userId,
        messageId: msg.properties.messageId
      });

      if (handler) {
        await handler(payload);
      } else {
        logger.warn(`No handler found for routing key: ${routingKey}`);
      }

      // Acknowledge message
      channel.ack(msg);

    } catch (error) {
      logger.error(`Error processing event ${routingKey}:`, error);
      
      // Retry หรือ Dead Letter Queue
      const retryCount = (msg.properties.headers?.['x-retry-count'] || 0) + 1;
      
      if (retryCount <= 3) {
        // Requeue with retry count
        channel.nack(msg, false, true);
      } else {
        // Send to DLQ
        channel.nack(msg, false, false);
      }
    }
  });

  logger.info(`Consumer started, listening to queue: ${QUEUE}`);
}

module.exports = { startConsumer };
```

```javascript
// message-handler/src/handlers/messageHandler.js
const { sendReplyMessage } = require('../services/lineApiService');
const { getOrCreateUser } = require('../services/userServiceClient');
const { trackEvent } = require('../services/analyticsClient');
const logger = require('../utils/logger');

async function handleMessage(payload) {
  const { event, destination } = payload;
  const { replyToken, message, source } = event;
  const userId = source.userId;

  logger.info(`Handling message from user: ${userId}`, {
    messageType: message.type,
    messageId: message.id
  });

  // Track analytics
  await trackEvent({
    userId,
    eventType: 'message',
    properties: {
      messageType: message.type,
      destination
    }
  });

  // ดึงข้อมูล user
  const user = await getOrCreateUser(userId);

  // Handle message ตาม type
  switch (message.type) {
    case 'text':
      await handleTextMessage(replyToken, message, user);
      break;
    case 'image':
      await handleImageMessage(replyToken, message, user);
      break;
    case 'sticker':
      await handleStickerMessage(replyToken, message, user);
      break;
    default:
      logger.info(`Unsupported message type: ${message.type}`);
  }
}

async function handleTextMessage(replyToken, message, user) {
  const text = message.text.toLowerCase();
  
  let replyMessages = [];

  if (text === 'สวัสดี' || text === 'hello' || text === 'hi') {
    replyMessages = [{
      type: 'text',
      text: `สวัสดีครับ คุณ${user.displayName || 'คุณ'}! 😊\nมีอะไรให้ช่วยไหมครับ?`
    }];
  } else if (text === 'เมนู' || text === 'menu') {
    replyMessages = [buildMenuMessage()];
  } else {
    // Default reply
    replyMessages = [{
      type: 'text',
      text: `ได้รับข้อความ: "${message.text}"\nพิมพ์ "เมนู" เพื่อดูคำสั่งทั้งหมดครับ`
    }];
  }

  await sendReplyMessage(replyToken, replyMessages);
}

function buildMenuMessage() {
  return {
    type: 'flex',
    altText: 'เมนูหลัก',
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [{
          type: 'text',
          text: 'เมนูหลัก',
          weight: 'bold',
          size: 'xl',
          color: '#ffffff'
        }],
        backgroundColor: '#00B900'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '📦 สินค้า',
              data: 'action=products'
            },
            style: 'secondary',
            margin: 'sm'
          },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '🛒 คำสั่งซื้อ',
              data: 'action=orders'
            },
            style: 'secondary',
            margin: 'sm'
          },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '📞 ติดต่อเรา',
              data: 'action=contact'
            },
            style: 'secondary',
            margin: 'sm'
          }
        ]
      }
    }
  };
}

module.exports = { handleMessage };
```

---

## 5. User Service

### 5.1 User Service Architecture

```
user-service/
├── src/
│   ├── index.js
│   ├── routes/
│   │   ├── users.js
│   │   └── segments.js
│   ├── models/
│   │   └── User.js
│   ├── services/
│   │   ├── lineProfileService.js
│   │   └── cacheService.js
│   └── middleware/
│       └── auth.js
├── migrations/
├── Dockerfile
└── package.json
```

```javascript
// user-service/src/index.js
const express = require('express');
const { Sequelize } = require('sequelize');
const userRoutes = require('./routes/users');
const segmentRoutes = require('./routes/segments');
const { authenticate } = require('./middleware/auth');
const logger = require('./utils/logger');

const app = express();
app.use(express.json());

// Internal service authentication
app.use(authenticate);

// Routes
app.use('/users', userRoutes);
app.use('/segments', segmentRoutes);

// Health check
app.get('/health', (req, res) => {
  res.json({ 
    status: 'ok',
    service: 'user-service',
    timestamp: new Date().toISOString()
  });
});

const PORT = process.env.PORT || 3005;
app.listen(PORT, () => {
  logger.info(`User Service running on port ${PORT}`);
});

module.exports = app;
```

```javascript
// user-service/src/models/User.js
const { DataTypes } = require('sequelize');
const { sequelize } = require('../database');

const User = sequelize.define('User', {
  id: {
    type: DataTypes.UUID,
    defaultValue: DataTypes.UUIDV4,
    primaryKey: true
  },
  lineUserId: {
    type: DataTypes.STRING(100),
    unique: true,
    allowNull: false
  },
  displayName: {
    type: DataTypes.STRING(255),
    allowNull: true
  },
  pictureUrl: {
    type: DataTypes.TEXT,
    allowNull: true
  },
  statusMessage: {
    type: DataTypes.TEXT,
    allowNull: true
  },
  language: {
    type: DataTypes.STRING(10),
    defaultValue: 'th'
  },
  isFollowing: {
    type: DataTypes.BOOLEAN,
    defaultValue: true
  },
  firstSeenAt: {
    type: DataTypes.DATE,
    defaultValue: DataTypes.NOW
  },
  lastSeenAt: {
    type: DataTypes.DATE,
    defaultValue: DataTypes.NOW
  },
  messageCount: {
    type: DataTypes.INTEGER,
    defaultValue: 0
  },
  tags: {
    type: DataTypes.JSONB,
    defaultValue: []
  },
  customFields: {
    type: DataTypes.JSONB,
    defaultValue: {}
  }
}, {
  tableName: 'users',
  indexes: [
    { fields: ['lineUserId'] },
    { fields: ['isFollowing'] },
    { fields: ['lastSeenAt'] }
  ]
});

module.exports = User;
```

```javascript
// user-service/src/routes/users.js
const express = require('express');
const router = express.Router();
const User = require('../models/User');
const { getLineProfile } = require('../services/lineProfileService');
const { getCached, setCache } = require('../services/cacheService');
const logger = require('../utils/logger');

// GET /users/:lineUserId
router.get('/:lineUserId', async (req, res) => {
  try {
    const { lineUserId } = req.params;
    
    // ลอง cache ก่อน
    const cached = await getCached(`user:${lineUserId}`);
    if (cached) {
      return res.json(cached);
    }

    const user = await User.findOne({ where: { lineUserId } });
    
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    await setCache(`user:${lineUserId}`, user, 300); // 5 minutes
    res.json(user);

  } catch (error) {
    logger.error('Error getting user:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// POST /users/getOrCreate
router.post('/getOrCreate', async (req, res) => {
  try {
    const { lineUserId } = req.body;

    if (!lineUserId) {
      return res.status(400).json({ error: 'lineUserId is required' });
    }

    let user = await User.findOne({ where: { lineUserId } });

    if (!user) {
      // ดึง profile จาก LINE API
      let profile = null;
      try {
        profile = await getLineProfile(lineUserId);
      } catch (err) {
        logger.warn(`Could not fetch LINE profile for ${lineUserId}:`, err.message);
      }

      user = await User.create({
        lineUserId,
        displayName: profile?.displayName,
        pictureUrl: profile?.pictureUrl,
        statusMessage: profile?.statusMessage,
        language: profile?.language || 'th',
        firstSeenAt: new Date(),
        lastSeenAt: new Date()
      });

      logger.info(`Created new user: ${lineUserId}`);
    } else {
      // Update lastSeenAt
      await user.update({
        lastSeenAt: new Date(),
        messageCount: user.messageCount + 1
      });
    }

    // Clear cache
    await setCache(`user:${lineUserId}`, null, 0);
    
    res.json(user);

  } catch (error) {
    logger.error('Error in getOrCreate:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// PUT /users/:lineUserId/tags
router.put('/:lineUserId/tags', async (req, res) => {
  try {
    const { lineUserId } = req.params;
    const { tags } = req.body;

    const user = await User.findOne({ where: { lineUserId } });
    if (!user) {
      return res.status(404).json({ error: 'User not found' });
    }

    await user.update({ tags });
    await setCache(`user:${lineUserId}`, null, 0);
    
    res.json({ success: true, tags: user.tags });

  } catch (error) {
    logger.error('Error updating tags:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// GET /users - with pagination and filtering
router.get('/', async (req, res) => {
  try {
    const {
      page = 1,
      limit = 50,
      isFollowing,
      tag,
      language
    } = req.query;

    const where = {};
    if (isFollowing !== undefined) {
      where.isFollowing = isFollowing === 'true';
    }
    if (language) {
      where.language = language;
    }

    const { count, rows } = await User.findAndCountAll({
      where,
      limit: parseInt(limit),
      offset: (parseInt(page) - 1) * parseInt(limit),
      order: [['lastSeenAt', 'DESC']]
    });

    res.json({
      users: rows,
      pagination: {
        total: count,
        page: parseInt(page),
        limit: parseInt(limit),
        pages: Math.ceil(count / parseInt(limit))
      }
    });

  } catch (error) {
    logger.error('Error listing users:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

module.exports = router;
```

---

## 6. Notification Service

### 6.1 Notification Service Design

```
notification-service/
├── src/
│   ├── index.js
│   ├── queues/
│   │   ├── broadcastQueue.js
│   │   └── pushQueue.js
│   ├── senders/
│   │   ├── lineSender.js
│   │   └── fcmSender.js
│   ├── templates/
│   │   └── templateEngine.js
│   └── consumers/
│       └── notificationConsumer.js
├── Dockerfile
└── package.json
```

```javascript
// notification-service/src/index.js
const express = require('express');
const { startNotificationConsumer } = require('./consumers/notificationConsumer');
const { broadcastQueue, pushQueue } = require('./queues');
const logger = require('./utils/logger');

const app = express();
app.use(express.json());

// Internal API endpoints
app.post('/broadcast', async (req, res) => {
  try {
    const { userIds, messages, scheduledAt } = req.body;

    if (!userIds || !messages) {
      return res.status(400).json({ error: 'userIds and messages are required' });
    }

    const jobId = await broadcastQueue.add('broadcast', {
      userIds,
      messages,
      scheduledAt
    }, {
      delay: scheduledAt ? new Date(scheduledAt) - Date.now() : 0,
      priority: scheduledAt ? 5 : 10,
    });

    logger.info(`Broadcast queued: job ${jobId} for ${userIds.length} users`);
    res.json({ jobId, status: 'queued', recipientCount: userIds.length });

  } catch (error) {
    logger.error('Error queuing broadcast:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.post('/push', async (req, res) => {
  try {
    const { userId, messages, priority = 'normal' } = req.body;

    const jobId = await pushQueue.add('push', {
      userId,
      messages
    }, {
      priority: priority === 'high' ? 20 : 10,
    });

    res.json({ jobId, status: 'queued' });

  } catch (error) {
    logger.error('Error queuing push:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.get('/health', (req, res) => {
  res.json({ 
    status: 'ok',
    service: 'notification-service',
    queues: {
      broadcast: 'active',
      push: 'active'
    }
  });
});

const PORT = process.env.PORT || 3006;

async function start() {
  await startNotificationConsumer();
  app.listen(PORT, () => {
    logger.info(`Notification Service running on port ${PORT}`);
  });
}

start();
```

```javascript
// notification-service/src/senders/lineSender.js
const axios = require('axios');
const logger = require('../utils/logger');

const LINE_API_URL = 'https://api.line.me/v2/bot';

class LineSender {
  constructor() {
    this.accessToken = process.env.LINE_CHANNEL_ACCESS_TOKEN;
    this.client = axios.create({
      baseURL: LINE_API_URL,
      headers: {
        'Authorization': `Bearer ${this.accessToken}`,
        'Content-Type': 'application/json'
      },
      timeout: 10000
    });
  }

  // Push message ไปยัง user เดียว
  async pushMessage(userId, messages) {
    try {
      const response = await this.client.post('/message/push', {
        to: userId,
        messages: Array.isArray(messages) ? messages : [messages]
      });

      logger.info(`Push message sent to ${userId}`, {
        messageCount: messages.length,
        requestId: response.headers['x-line-request-id']
      });

      return { success: true, requestId: response.headers['x-line-request-id'] };

    } catch (error) {
      const errorCode = error.response?.data?.message;
      logger.error(`Failed to push message to ${userId}:`, {
        status: error.response?.status,
        error: errorCode
      });

      // Handle specific errors
      if (error.response?.status === 400 && errorCode?.includes('Invalid reply token')) {
        throw new Error('INVALID_REPLY_TOKEN');
      }
      if (error.response?.status === 429) {
        throw new Error('RATE_LIMIT_EXCEEDED');
      }

      throw error;
    }
  }

  // Multicast - ส่งไปหลาย user (สูงสุด 500)
  async multicast(userIds, messages) {
    try {
      // แบ่ง userIds เป็น chunks ละ 500
      const chunks = this.chunkArray(userIds, 500);
      const results = [];

      for (const chunk of chunks) {
        const response = await this.client.post('/message/multicast', {
          to: chunk,
          messages: Array.isArray(messages) ? messages : [messages]
        });

        results.push({
          recipientCount: chunk.length,
          requestId: response.headers['x-line-request-id']
        });

        // Rate limiting: 20 requests/second
        await this.sleep(50);
      }

      logger.info(`Multicast sent to ${userIds.length} users`);
      return { success: true, chunks: results.length, totalRecipients: userIds.length };

    } catch (error) {
      logger.error('Multicast failed:', error);
      throw error;
    }
  }

  // Broadcast - ส่งทุก follower
  async broadcast(messages) {
    try {
      const response = await this.client.post('/message/broadcast', {
        messages: Array.isArray(messages) ? messages : [messages]
      });

      logger.info('Broadcast sent', {
        requestId: response.headers['x-line-request-id']
      });

      return { success: true, requestId: response.headers['x-line-request-id'] };

    } catch (error) {
      logger.error('Broadcast failed:', error);
      throw error;
    }
  }

  chunkArray(array, size) {
    const chunks = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

module.exports = new LineSender();
```

---

## 7. Analytics Service

### 7.1 Analytics Service Implementation

```javascript
// analytics-service/src/index.js
const express = require('express');
const { InfluxDB, Point } = require('@influxdata/influxdb-client');
const logger = require('./utils/logger');

const app = express();
app.use(express.json());

// InfluxDB client
const influxClient = new InfluxDB({
  url: process.env.INFLUXDB_URL || 'http://influxdb:8086',
  token: process.env.INFLUXDB_TOKEN
});

const writeApi = influxClient.getWriteApi(
  process.env.INFLUXDB_ORG,
  process.env.INFLUXDB_BUCKET
);

const queryApi = influxClient.getQueryApi(process.env.INFLUXDB_ORG);

// Track event endpoint
app.post('/track', async (req, res) => {
  try {
    const { userId, eventType, properties = {}, timestamp } = req.body;

    const point = new Point('line_events')
      .tag('eventType', eventType)
      .tag('userId', userId)
      .stringField('userId', userId)
      .stringField('eventType', eventType);

    // Add properties as fields
    Object.entries(properties).forEach(([key, value]) => {
      if (typeof value === 'number') {
        point.floatField(key, value);
      } else if (typeof value === 'boolean') {
        point.booleanField(key, value);
      } else {
        point.stringField(key, String(value));
      }
    });

    if (timestamp) {
      point.timestamp(new Date(timestamp));
    }

    writeApi.writePoint(point);
    await writeApi.flush();

    res.json({ success: true });

  } catch (error) {
    logger.error('Error tracking event:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Query analytics
app.get('/stats/messages', async (req, res) => {
  try {
    const { start = '-24h', end = 'now()' } = req.query;

    const query = `
      from(bucket: "${process.env.INFLUXDB_BUCKET}")
        |> range(start: ${start}, stop: ${end})
        |> filter(fn: (r) => r._measurement == "line_events")
        |> filter(fn: (r) => r.eventType == "message")
        |> aggregateWindow(every: 1h, fn: count)
        |> yield(name: "hourly_messages")
    `;

    const results = [];
    await queryApi.queryRows(query, {
      next: (row, tableMeta) => {
        results.push(tableMeta.toObject(row));
      },
      error: (error) => {
        logger.error('Query error:', error);
      },
      complete: () => {
        res.json({ data: results });
      }
    });

  } catch (error) {
    logger.error('Error querying analytics:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.get('/stats/active-users', async (req, res) => {
  try {
    const query = `
      from(bucket: "${process.env.INFLUXDB_BUCKET}")
        |> range(start: -7d)
        |> filter(fn: (r) => r._measurement == "line_events")
        |> distinct(column: "userId")
        |> count()
        |> yield(name: "active_users")
    `;

    let activeUsers = 0;
    await queryApi.queryRows(query, {
      next: (row, tableMeta) => {
        const obj = tableMeta.toObject(row);
        activeUsers = obj._value;
      },
      error: (error) => logger.error('Query error:', error),
      complete: () => {
        res.json({ activeUsers, period: '7d' });
      }
    });

  } catch (error) {
    logger.error('Error querying active users:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

app.get('/health', (req, res) => {
  res.json({ status: 'ok', service: 'analytics-service' });
});

const PORT = process.env.PORT || 3007;
app.listen(PORT, () => {
  logger.info(`Analytics Service running on port ${PORT}`);
});
```

---

## 8. API Gateway

### 8.1 Kong API Gateway Configuration

```yaml
# kong/kong.yml
_format_version: "3.0"

services:
  - name: webhook-service
    url: http://webhook-receiver:3001
    routes:
      - name: webhook-route
        paths:
          - /webhook
        methods:
          - POST
    plugins:
      - name: rate-limiting
        config:
          minute: 10000
          policy: redis
          redis_host: redis
      - name: request-size-limiting
        config:
          allowed_payload_size: 5

  - name: user-service
    url: http://user-service:3005
    routes:
      - name: users-route
        paths:
          - /api/users
        strip_path: true
    plugins:
      - name: jwt
        config:
          secret_is_base64: false
      - name: rate-limiting
        config:
          minute: 1000

  - name: analytics-service
    url: http://analytics-service:3007
    routes:
      - name: analytics-route
        paths:
          - /api/analytics
        strip_path: true
    plugins:
      - name: jwt
      - name: rate-limiting
        config:
          minute: 500

  - name: admin-service
    url: http://admin-service:3003
    routes:
      - name: admin-route
        paths:
          - /api/admin
        strip_path: true
    plugins:
      - name: jwt
      - name: ip-restriction
        config:
          allow:
            - 10.0.0.0/8
            - 172.16.0.0/12
            - 192.168.0.0/16

plugins:
  - name: cors
    config:
      origins:
        - "https://yourdomain.com"
      methods:
        - GET
        - POST
        - PUT
        - DELETE
        - OPTIONS
      headers:
        - Authorization
        - Content-Type
      exposed_headers:
        - X-Request-Id
      max_age: 3600

  - name: prometheus
    config:
      per_consumer: true

  - name: request-id
    config:
      header_name: X-Request-Id
      echo_downstream: true
```

### 8.2 Express Gateway (Alternative)

```javascript
// api-gateway/src/index.js
const express = require('express');
const { createProxyMiddleware } = require('http-proxy-middleware');
const rateLimit = require('express-rate-limit');
const jwt = require('jsonwebtoken');
const logger = require('./utils/logger');

const app = express();

// Service registry
const SERVICES = {
  webhook: {
    target: process.env.WEBHOOK_SERVICE_URL || 'http://webhook-receiver:3001',
    paths: ['/webhook'],
    auth: false,
    rateLimit: { windowMs: 60000, max: 10000 }
  },
  users: {
    target: process.env.USER_SERVICE_URL || 'http://user-service:3005',
    paths: ['/api/users'],
    auth: true,
    rateLimit: { windowMs: 60000, max: 1000 }
  },
  analytics: {
    target: process.env.ANALYTICS_SERVICE_URL || 'http://analytics-service:3007',
    paths: ['/api/analytics'],
    auth: true,
    rateLimit: { windowMs: 60000, max: 500 }
  },
  notifications: {
    target: process.env.NOTIFICATION_SERVICE_URL || 'http://notification-service:3006',
    paths: ['/api/notifications'],
    auth: true,
    rateLimit: { windowMs: 60000, max: 200 }
  }
};

// JWT Middleware
const authenticate = (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  
  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.user = decoded;
    next();
  } catch (error) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

// Request logging
app.use((req, res, next) => {
  const requestId = require('crypto').randomUUID();
  req.requestId = requestId;
  res.setHeader('X-Request-ID', requestId);
  
  const start = Date.now();
  res.on('finish', () => {
    logger.info('Request completed', {
      requestId,
      method: req.method,
      path: req.path,
      status: res.statusCode,
      duration: `${Date.now() - start}ms`
    });
  });
  
  next();
});

// Setup proxy for each service
Object.entries(SERVICES).forEach(([name, config]) => {
  const limiter = rateLimit(config.rateLimit);

  config.paths.forEach(path => {
    const middlewares = [limiter];
    
    if (config.auth) {
      middlewares.push(authenticate);
    }

    middlewares.push(createProxyMiddleware({
      target: config.target,
      changeOrigin: true,
      pathRewrite: (path) => path.replace(`/api/${name}`, ''),
      on: {
        error: (err, req, res) => {
          logger.error(`Proxy error for ${name}:`, err);
          res.status(502).json({ error: 'Bad Gateway', service: name });
        },
        proxyReq: (proxyReq, req) => {
          // Forward request ID
          proxyReq.setHeader('X-Request-ID', req.requestId);
          // Forward internal token
          proxyReq.setHeader('X-Internal-Token', process.env.INTERNAL_SERVICE_TOKEN);
        }
      }
    }));

    app.use(path, ...middlewares);
    logger.info(`Registered route: ${path} -> ${config.target}`);
  });
});

// Health check
app.get('/health', (req, res) => {
  res.json({ 
    status: 'ok',
    service: 'api-gateway',
    services: Object.keys(SERVICES)
  });
});

const PORT = process.env.PORT || 80;
app.listen(PORT, () => {
  logger.info(`API Gateway running on port ${PORT}`);
});
```

---

## 9. Inter-service Authentication

### 9.1 Service-to-Service JWT

```javascript
// shared/src/serviceAuth.js
const jwt = require('jsonwebtoken');

const INTERNAL_SECRET = process.env.INTERNAL_SERVICE_TOKEN || 'change-in-production';
const SERVICE_TOKENS_TTL = 60 * 60; // 1 hour

// สร้าง service token
function createServiceToken(serviceName) {
  return jwt.sign(
    {
      service: serviceName,
      type: 'service',
      iat: Math.floor(Date.now() / 1000)
    },
    INTERNAL_SECRET,
    { expiresIn: SERVICE_TOKENS_TTL }
  );
}

// Verify service token
function verifyServiceToken(token) {
  try {
    const decoded = jwt.verify(token, INTERNAL_SECRET);
    if (decoded.type !== 'service') {
      throw new Error('Invalid token type');
    }
    return decoded;
  } catch (error) {
    throw new Error(`Invalid service token: ${error.message}`);
  }
}

// Middleware สำหรับ internal services
function requireServiceAuth(req, res, next) {
  const token = req.headers['x-internal-token'];
  
  if (!token) {
    return res.status(401).json({ error: 'Missing service token' });
  }

  try {
    const decoded = verifyServiceToken(token);
    req.callerService = decoded.service;
    next();
  } catch (error) {
    return res.status(401).json({ error: error.message });
  }
}

module.exports = { createServiceToken, verifyServiceToken, requireServiceAuth };
```

```javascript
// shared/src/serviceClient.js
const axios = require('axios');
const { createServiceToken } = require('./serviceAuth');

class ServiceClient {
  constructor(serviceName, baseURL) {
    this.serviceName = serviceName;
    this.token = createServiceToken(serviceName);
    
    this.client = axios.create({
      baseURL,
      timeout: 5000,
      headers: {
        'Content-Type': 'application/json',
        'X-Internal-Token': this.token
      }
    });

    // Auto-refresh token
    setInterval(() => {
      this.token = createServiceToken(serviceName);
      this.client.defaults.headers['X-Internal-Token'] = this.token;
    }, 55 * 60 * 1000); // refresh ทุก 55 นาที
  }

  async get(path, params = {}) {
    const response = await this.client.get(path, { params });
    return response.data;
  }

  async post(path, data = {}) {
    const response = await this.client.post(path, data);
    return response.data;
  }

  async put(path, data = {}) {
    const response = await this.client.put(path, data);
    return response.data;
  }

  async delete(path) {
    const response = await this.client.delete(path);
    return response.data;
  }
}

// Pre-configured clients
const userServiceClient = new ServiceClient(
  'message-handler',
  process.env.USER_SERVICE_URL || 'http://user-service:3005'
);

const notificationServiceClient = new ServiceClient(
  'message-handler',
  process.env.NOTIFICATION_SERVICE_URL || 'http://notification-service:3006'
);

const analyticsServiceClient = new ServiceClient(
  'message-handler',
  process.env.ANALYTICS_SERVICE_URL || 'http://analytics-service:3007'
);

module.exports = {
  userServiceClient,
  notificationServiceClient,
  analyticsServiceClient
};
```

---

## 10. Docker Compose Setup

### 10.1 Complete Docker Compose Configuration

```yaml
# docker-compose.yml
version: '3.9'

x-common-variables: &common-variables
  RABBITMQ_URL: amqp://admin:password@rabbitmq:5672
  REDIS_HOST: redis
  REDIS_PORT: 6379
  LOG_LEVEL: info
  NODE_ENV: production

x-healthcheck-defaults: &healthcheck-defaults
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s

services:
  # ===========================
  # Infrastructure Services
  # ===========================
  
  rabbitmq:
    image: rabbitmq:3.12-management
    container_name: linebot-rabbitmq
    restart: unless-stopped
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: password
      RABBITMQ_DEFAULT_VHOST: linebot
    ports:
      - "5672:5672"
      - "15672:15672"  # Management UI
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
      - ./rabbitmq/rabbitmq.conf:/etc/rabbitmq/rabbitmq.conf
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      <<: *healthcheck-defaults
    networks:
      - linebot-network

  redis:
    image: redis:7.2-alpine
    container_name: linebot-redis
    restart: unless-stopped
    command: redis-server --requirepass redispassword --maxmemory 512mb --maxmemory-policy allkeys-lru
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "redispassword", "ping"]
      <<: *healthcheck-defaults
    networks:
      - linebot-network

  postgres:
    image: postgres:16-alpine
    container_name: linebot-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: linebot
      POSTGRES_PASSWORD: password
      POSTGRES_DB: linebot_main
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgres/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U linebot"]
      <<: *healthcheck-defaults
    networks:
      - linebot-network

  influxdb:
    image: influxdb:2.7
    container_name: linebot-influxdb
    restart: unless-stopped
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: password123
      DOCKER_INFLUXDB_INIT_ORG: linebot-org
      DOCKER_INFLUXDB_INIT_BUCKET: linebot-metrics
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: my-super-secret-token
    ports:
      - "8086:8086"
    volumes:
      - influxdb_data:/var/lib/influxdb2
    networks:
      - linebot-network

  # ===========================
  # Application Services
  # ===========================

  webhook-receiver:
    build:
      context: ./webhook-receiver
      dockerfile: Dockerfile
    container_name: linebot-webhook-receiver
    restart: unless-stopped
    environment:
      <<: *common-variables
      PORT: 3001
      LINE_CHANNEL_SECRET: ${LINE_CHANNEL_SECRET}
    ports:
      - "3001:3001"
    depends_on:
      rabbitmq:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3001/health"]
      <<: *healthcheck-defaults
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
    networks:
      - linebot-network

  message-handler:
    build:
      context: ./message-handler
      dockerfile: Dockerfile
    restart: unless-stopped
    environment:
      <<: *common-variables
      LINE_CHANNEL_ACCESS_TOKEN: ${LINE_CHANNEL_ACCESS_TOKEN}
      USER_SERVICE_URL: http://user-service:3005
      ANALYTICS_SERVICE_URL: http://analytics-service:3007
      INTERNAL_SERVICE_TOKEN: ${INTERNAL_SERVICE_TOKEN}
    depends_on:
      rabbitmq:
        condition: service_healthy
      postgres:
        condition: service_healthy
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 512M
    networks:
      - linebot-network

  user-service:
    build:
      context: ./user-service
      dockerfile: Dockerfile
    container_name: linebot-user-service
    restart: unless-stopped
    environment:
      <<: *common-variables
      PORT: 3005
      DATABASE_URL: postgresql://linebot:password@postgres:5432/linebot_main
      LINE_CHANNEL_ACCESS_TOKEN: ${LINE_CHANNEL_ACCESS_TOKEN}
      INTERNAL_SERVICE_TOKEN: ${INTERNAL_SERVICE_TOKEN}
    ports:
      - "3005:3005"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3005/health"]
      <<: *healthcheck-defaults
    networks:
      - linebot-network

  notification-service:
    build:
      context: ./notification-service
      dockerfile: Dockerfile
    container_name: linebot-notification-service
    restart: unless-stopped
    environment:
      <<: *common-variables
      PORT: 3006
      LINE_CHANNEL_ACCESS_TOKEN: ${LINE_CHANNEL_ACCESS_TOKEN}
      INTERNAL_SERVICE_TOKEN: ${INTERNAL_SERVICE_TOKEN}
    ports:
      - "3006:3006"
    depends_on:
      rabbitmq:
        condition: service_healthy
      redis:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3006/health"]
      <<: *healthcheck-defaults
    networks:
      - linebot-network

  analytics-service:
    build:
      context: ./analytics-service
      dockerfile: Dockerfile
    container_name: linebot-analytics-service
    restart: unless-stopped
    environment:
      <<: *common-variables
      PORT: 3007
      INFLUXDB_URL: http://influxdb:8086
      INFLUXDB_TOKEN: my-super-secret-token
      INFLUXDB_ORG: linebot-org
      INFLUXDB_BUCKET: linebot-metrics
      INTERNAL_SERVICE_TOKEN: ${INTERNAL_SERVICE_TOKEN}
    ports:
      - "3007:3007"
    depends_on:
      influxdb:
        condition: service_started
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3007/health"]
      <<: *healthcheck-defaults
    networks:
      - linebot-network

  # ===========================
  # API Gateway
  # ===========================

  api-gateway:
    build:
      context: ./api-gateway
      dockerfile: Dockerfile
    container_name: linebot-api-gateway
    restart: unless-stopped
    environment:
      PORT: 80
      WEBHOOK_SERVICE_URL: http://webhook-receiver:3001
      USER_SERVICE_URL: http://user-service:3005
      NOTIFICATION_SERVICE_URL: http://notification-service:3006
      ANALYTICS_SERVICE_URL: http://analytics-service:3007
      JWT_SECRET: ${JWT_SECRET}
      INTERNAL_SERVICE_TOKEN: ${INTERNAL_SERVICE_TOKEN}
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - webhook-receiver
      - user-service
      - notification-service
      - analytics-service
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      <<: *healthcheck-defaults
    networks:
      - linebot-network

  # ===========================
  # Monitoring
  # ===========================

  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: linebot-prometheus
    restart: unless-stopped
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
      - '--storage.tsdb.retention.time=15d'
    networks:
      - linebot-network

  grafana:
    image: grafana/grafana:10.1.0
    container_name: linebot-grafana
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_PASSWORD: grafana_password
      GF_USERS_ALLOW_SIGN_UP: false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
    networks:
      - linebot-network

  jaeger:
    image: jaegertracing/all-in-one:1.50
    container_name: linebot-jaeger
    restart: unless-stopped
    environment:
      COLLECTOR_ZIPKIN_HOST_PORT: ":9411"
    ports:
      - "5775:5775/udp"
      - "6831:6831/udp"
      - "6832:6832/udp"
      - "5778:5778"
      - "16686:16686"  # UI
      - "14268:14268"
      - "14250:14250"
      - "9411:9411"
    networks:
      - linebot-network

volumes:
  rabbitmq_data:
  redis_data:
  postgres_data:
  influxdb_data:
  prometheus_data:
  grafana_data:

networks:
  linebot-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

---

## 11. Distributed Tracing

### 11.1 OpenTelemetry Setup

```javascript
// shared/src/tracing.js
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');
const { Resource } = require('@opentelemetry/resources');
const { SemanticResourceAttributes } = require('@opentelemetry/semantic-conventions');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const opentelemetry = require('@opentelemetry/api');

function initTracing(serviceName) {
  const jaegerExporter = new JaegerExporter({
    endpoint: process.env.JAEGER_ENDPOINT || 'http://jaeger:14268/api/traces',
  });

  const sdk = new NodeSDK({
    resource: new Resource({
      [SemanticResourceAttributes.SERVICE_NAME]: serviceName,
      [SemanticResourceAttributes.SERVICE_VERSION]: process.env.APP_VERSION || '1.0.0',
      environment: process.env.NODE_ENV || 'production'
    }),
    traceExporter: jaegerExporter,
    instrumentations: [
      getNodeAutoInstrumentations({
        '@opentelemetry/instrumentation-fs': {
          enabled: false, // ปิด filesystem tracing เพื่อประสิทธิภาพ
        },
      })
    ],
  });

  sdk.start();
  
  process.on('SIGTERM', () => {
    sdk.shutdown()
      .then(() => console.log('Tracing terminated'))
      .catch((error) => console.log('Error terminating tracing', error))
      .finally(() => process.exit(0));
  });

  return sdk;
}

function createSpan(name, attributes = {}) {
  const tracer = opentelemetry.trace.getActiveSpan();
  if (!tracer) return null;
  
  return opentelemetry.trace.getTracer('linebot').startSpan(name, {
    attributes
  });
}

function getTraceId() {
  const span = opentelemetry.trace.getActiveSpan();
  if (!span) return null;
  return span.spanContext().traceId;
}

module.exports = { initTracing, createSpan, getTraceId };
```

```javascript
// webhook-receiver/src/index.js (with tracing)
const { initTracing } = require('./shared/tracing');

// ต้อง initialize tracing ก่อน require อื่นๆ
initTracing('webhook-receiver');

const express = require('express');
// ... rest of the code
```

---

## 12. Service Communication Patterns

### 12.1 REST vs gRPC vs Message Queue

```
┌─────────────────────────────────────────────────────────┐
│          Service Communication Decision Matrix           │
│                                                         │
│  Pattern    │ Use Case           │ Pros/Cons            │
│─────────────┼────────────────────┼──────────────────────│
│  REST       │ Request-Response   │ + Simple, widespread │
│             │ CRUD operations    │ - Synchronous        │
│             │                    │ - Tight coupling     │
│─────────────┼────────────────────┼──────────────────────│
│  gRPC       │ High-performance   │ + Fast (binary)      │
│             │ Internal calls     │ + Strong typing      │
│             │ Streaming          │ - Complex setup      │
│─────────────┼────────────────────┼──────────────────────│
│  Message    │ Async processing   │ + Decoupled          │
│  Queue      │ Event-driven       │ + Scalable           │
│             │ Long-running jobs  │ - Eventually consis. │
└─────────────────────────────────────────────────────────┘
```

### 12.2 gRPC User Service

```protobuf
// user-service/proto/user.proto
syntax = "proto3";

package user;

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
  rpc GetOrCreateUser (GetOrCreateUserRequest) returns (User);
  rpc UpdateUser (UpdateUserRequest) returns (User);
  rpc GetUsersByTag (GetUsersByTagRequest) returns (stream User);
  rpc BatchGetUsers (BatchGetUsersRequest) returns (BatchGetUsersResponse);
}

message User {
  string id = 1;
  string line_user_id = 2;
  string display_name = 3;
  string picture_url = 4;
  bool is_following = 5;
  int64 message_count = 6;
  string language = 7;
  int64 created_at = 8;
  int64 last_seen_at = 9;
  repeated string tags = 10;
}

message GetUserRequest {
  string line_user_id = 1;
}

message GetOrCreateUserRequest {
  string line_user_id = 1;
  bool force_refresh = 2;
}

message UpdateUserRequest {
  string line_user_id = 1;
  optional string display_name = 2;
  optional bool is_following = 3;
  repeated string add_tags = 4;
  repeated string remove_tags = 5;
}

message GetUsersByTagRequest {
  string tag = 1;
  int32 limit = 2;
}

message BatchGetUsersRequest {
  repeated string line_user_ids = 1;
}

message BatchGetUsersResponse {
  repeated User users = 1;
  repeated string not_found_ids = 2;
}
```

```javascript
// user-service/src/grpc-server.js
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');
const User = require('./models/User');

const PROTO_PATH = path.join(__dirname, '../proto/user.proto');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true
});

const userProto = grpc.loadPackageDefinition(packageDefinition).user;

// Implement service methods
const userService = {
  GetUser: async (call, callback) => {
    try {
      const { line_user_id } = call.request;
      const user = await User.findOne({ where: { lineUserId: line_user_id } });
      
      if (!user) {
        return callback({
          code: grpc.status.NOT_FOUND,
          message: `User ${line_user_id} not found`
        });
      }

      callback(null, mapUserToProto(user));
    } catch (error) {
      callback({ code: grpc.status.INTERNAL, message: error.message });
    }
  },

  GetOrCreateUser: async (call, callback) => {
    try {
      const { line_user_id } = call.request;
      
      let [user, created] = await User.findOrCreate({
        where: { lineUserId: line_user_id },
        defaults: {
          lineUserId: line_user_id,
          firstSeenAt: new Date()
        }
      });

      if (!created) {
        await user.update({ lastSeenAt: new Date() });
      }

      callback(null, mapUserToProto(user));
    } catch (error) {
      callback({ code: grpc.status.INTERNAL, message: error.message });
    }
  },

  GetUsersByTag: async (call) => {
    const { tag, limit } = call.request;
    
    const users = await User.findAll({
      where: {
        tags: { [Op.contains]: [tag] },
        isFollowing: true
      },
      limit: limit || 1000
    });

    users.forEach(user => {
      call.write(mapUserToProto(user));
    });
    
    call.end();
  }
};

function mapUserToProto(user) {
  return {
    id: user.id,
    line_user_id: user.lineUserId,
    display_name: user.displayName || '',
    picture_url: user.pictureUrl || '',
    is_following: user.isFollowing,
    message_count: user.messageCount,
    language: user.language,
    created_at: Math.floor(user.createdAt.getTime() / 1000),
    last_seen_at: Math.floor(user.lastSeenAt.getTime() / 1000),
    tags: user.tags || []
  };
}

function startGrpcServer() {
  const server = new grpc.Server();
  server.addService(userProto.UserService.service, userService);
  
  const port = process.env.GRPC_PORT || 50051;
  server.bindAsync(
    `0.0.0.0:${port}`,
    grpc.ServerCredentials.createInsecure(),
    (error) => {
      if (error) {
        console.error('gRPC server failed to start:', error);
        process.exit(1);
      }
      server.start();
      console.log(`gRPC server running on port ${port}`);
    }
  );
}

module.exports = { startGrpcServer };
```

---

## 13. Monitoring Setup

### 13.1 Prometheus Configuration

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'webhook-receiver'
    static_configs:
      - targets: ['webhook-receiver:3001']
    metrics_path: '/metrics'

  - job_name: 'message-handler'
    static_configs:
      - targets: ['message-handler:3004']
    metrics_path: '/metrics'

  - job_name: 'user-service'
    static_configs:
      - targets: ['user-service:3005']
    metrics_path: '/metrics'

  - job_name: 'notification-service'
    static_configs:
      - targets: ['notification-service:3006']
    metrics_path: '/metrics'

  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['rabbitmq:15692']
    metrics_path: '/metrics'

  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
```

```javascript
// shared/src/metrics.js
const client = require('prom-client');

// Enable default metrics
client.collectDefaultMetrics({ 
  prefix: 'linebot_',
  gcDurationBuckets: [0.001, 0.01, 0.1, 1, 2, 5]
});

// Custom metrics
const httpRequestDuration = new client.Histogram({
  name: 'linebot_http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 2, 5]
});

const webhookEventsTotal = new client.Counter({
  name: 'linebot_webhook_events_total',
  help: 'Total number of webhook events received',
  labelNames: ['eventType']
});

const messagesSentTotal = new client.Counter({
  name: 'linebot_messages_sent_total',
  help: 'Total number of messages sent via LINE API',
  labelNames: ['type', 'status']
});

const activeUsers = new client.Gauge({
  name: 'linebot_active_users',
  help: 'Number of active users in the last 24 hours'
});

const queueDepth = new client.Gauge({
  name: 'linebot_queue_depth',
  help: 'Current depth of message queues',
  labelNames: ['queue']
});

// HTTP metrics middleware
function metricsMiddleware(req, res, next) {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration
      .labels(req.method, req.route?.path || req.path, res.statusCode)
      .observe(duration);
  });
  
  next();
}

// Metrics endpoint
function metricsHandler(req, res) {
  res.set('Content-Type', client.register.contentType);
  client.register.metrics().then(metrics => {
    res.end(metrics);
  });
}

module.exports = {
  httpRequestDuration,
  webhookEventsTotal,
  messagesSentTotal,
  activeUsers,
  queueDepth,
  metricsMiddleware,
  metricsHandler
};
```

---

## 14. Dockerfile สำหรับแต่ละ Service

### 14.1 Standard Node.js Service Dockerfile

```dockerfile
# Base Dockerfile สำหรับ Node.js services
FROM node:20-alpine AS base
WORKDIR /app
RUN apk add --no-cache curl

# Dependencies stage
FROM base AS dependencies
COPY package*.json ./
RUN npm ci --only=production

# Development stage
FROM base AS development
COPY package*.json ./
RUN npm ci
COPY . .
CMD ["npm", "run", "dev"]

# Production stage
FROM base AS production
COPY --from=dependencies /app/node_modules ./node_modules
COPY . .

# Security: non-root user
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 && \
    chown -R nodejs:nodejs /app

USER nodejs

ENV NODE_ENV=production
EXPOSE 3001

HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
  CMD curl -f http://localhost:${PORT:-3001}/health || exit 1

CMD ["node", "src/index.js"]
```

---

## 15. Scaling Guidelines

### 15.1 Horizontal Scaling Strategy

```
การ Scale แต่ละ Service:

webhook-receiver:   Scale ตาม incoming traffic
                    → เพิ่ม replicas เมื่อ CPU > 70%
                    → ใช้ Load Balancer ด้านหน้า

message-handler:    Scale ตาม queue depth
                    → เพิ่ม consumers เมื่อ queue > 10,000
                    → ใช้ Kubernetes HPA

user-service:       Scale ตาม database connections
                    → ใช้ Connection Pool (PgBouncer)
                    → Read replicas สำหรับ queries

notification:       Scale ตาม outgoing messages/second
                    → LINE API limit: 1,000 msg/s
                    → ใช้ multiple instances + rate limiting

analytics:          Scale ตาม write throughput
                    → InfluxDB cluster สำหรับ high volume
                    → Batch writes แทน individual writes
```

```yaml
# kubernetes/webhook-receiver-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webhook-receiver-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webhook-receiver
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: External
      external:
        metric:
          name: rabbitmq_queue_messages
          selector:
            matchLabels:
              queue: webhook.processing
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 120
```

---

## 16. Health Checks และ Circuit Breaker

### 16.1 Circuit Breaker Pattern

```javascript
// shared/src/circuitBreaker.js
const CircuitBreaker = require('opossum');

function createCircuitBreaker(fn, options = {}) {
  const defaultOptions = {
    timeout: 3000,            // 3 seconds
    errorThresholdPercentage: 50,  // Open circuit เมื่อ error > 50%
    resetTimeout: 30000,      // ลอง reset หลัง 30 วินาที
    volumeThreshold: 10,      // ต้องมี 10 requests ก่อน evaluate
    name: 'default-circuit'
  };

  const breaker = new CircuitBreaker(fn, { ...defaultOptions, ...options });

  breaker.fallback((args, err) => {
    console.error(`Circuit breaker fallback for ${breaker.name}:`, err?.message);
    return null;
  });

  breaker.on('open', () => {
    console.warn(`Circuit breaker OPEN: ${breaker.name}`);
  });

  breaker.on('halfOpen', () => {
    console.info(`Circuit breaker HALF-OPEN: ${breaker.name}`);
  });

  breaker.on('close', () => {
    console.info(`Circuit breaker CLOSED: ${breaker.name}`);
  });

  return breaker;
}

// ตัวอย่างการใช้งาน
const userServiceBreaker = createCircuitBreaker(
  async (lineUserId) => {
    const response = await userServiceClient.post('/getOrCreate', { lineUserId });
    return response;
  },
  {
    name: 'user-service',
    timeout: 5000
  }
);

module.exports = { createCircuitBreaker, userServiceBreaker };
```

---

## 17. สรุปและ Best Practices

### 17.1 Checklist สำหรับ Production

```
✅ Infrastructure
   □ Load Balancer สำหรับ webhook receiver
   □ Redis Cluster สำหรับ high availability
   □ PostgreSQL replication
   □ RabbitMQ clustering

✅ Security
   □ Inter-service authentication ด้วย JWT
   □ Network policies ระหว่าง services
   □ Secret management (Vault หรือ K8s secrets)
   □ TLS สำหรับ internal communication

✅ Observability
   □ Distributed tracing (Jaeger)
   □ Metrics (Prometheus + Grafana)
   □ Centralized logging (ELK Stack)
   □ Health checks ทุก service

✅ Resilience
   □ Circuit breakers
   □ Retry logic with exponential backoff
   □ Dead letter queues
   □ Graceful shutdown

✅ Performance
   □ Connection pooling
   □ Response caching
   □ Database query optimization
   □ Horizontal auto-scaling
```

### 17.2 Performance Benchmarks

```
Webhook Receiver:
  - Target: < 100ms response time
  - Throughput: 10,000 events/minute per instance
  - P99 latency: < 200ms

Message Handler:
  - Processing time: < 500ms per message
  - Queue depth: < 1,000 messages normal operation
  - Error rate: < 0.1%

User Service:
  - API response time: < 50ms (cached)
  - Database query: < 20ms
  - Cache hit rate: > 90%

Notification Service:
  - Push message: < 1s delivery
  - Multicast throughput: 500 messages/second
  - Broadcast: depends on follower count
```

---

## บทสรุป

Microservices Architecture สำหรับ LINE Bot ช่วยให้:

1. **Scale แยกส่วน** - webhook receiver scale แยกจาก analytics
2. **Deploy อิสระ** - update notification service โดยไม่ affect webhook
3. **Technology Flexibility** - ใช้ Node.js สำหรับ real-time, Python สำหรับ analytics
4. **Fault Isolation** - analytics ล่ม ไม่กระทบ webhook processing
5. **Team Independence** - แต่ละทีมดูแล service ของตัวเอง

ในบทต่อไปจะพูดถึง Queue Management แบบละเอียด เพื่อให้ระบบรองรับ high load ได้อย่างมีประสิทธิภาพ
