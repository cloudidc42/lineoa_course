# Part 73: Multi-Bot Architecture สำหรับจัดการหลาย LINE Bot

## บทนำ

เมื่อธุรกิจเติบโต อาจต้องการมี LINE Bot หลาย account เช่น Bot สำหรับแต่ละสาขา, Bot สำหรับแต่ละ product line หรือให้บริการ Bot ในรูปแบบ SaaS ให้ลูกค้าหลายราย บทนี้จะครอบคลุมสถาปัตยกรรมสำหรับจัดการ Bot หลายตัว

---

## 1. Overview Architecture

### 1.1 Multi-Bot System Design

```
┌─────────────────────────────────────────────────────────────────┐
│                    MULTI-BOT PLATFORM                           │
│                                                                  │
│  Tenant A: Brand A Bot    Tenant B: Brand B Bot                 │
│  ┌─────────────────┐      ┌─────────────────┐                  │
│  │ Channel: xxxxxx │      │ Channel: yyyyyy │                  │
│  │ Token:   aaaaaa │      │ Token:   bbbbbb │                  │
│  └────────┬────────┘      └────────┬────────┘                  │
│           │                         │                           │
│           ▼                         ▼                           │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                  Single Webhook Endpoint                 │   │
│  │              https://api.example.com/webhook             │   │
│  └─────────────────────┬───────────────────────────────────┘   │
│                         │                                       │
│  ┌──────────────────────▼──────────────────────────────────┐   │
│  │                  Webhook Router                          │   │
│  │    destination → Bot Registry → Channel Token           │   │
│  └──────────────────────┬──────────────────────────────────┘   │
│                          │                                      │
│         ┌────────────────┼────────────────┐                    │
│         │                │                │                    │
│         ▼                ▼                ▼                    │
│   ┌───────────┐   ┌───────────┐   ┌───────────┐               │
│   │  Bot A    │   │  Bot B    │   │  Bot C    │               │
│   │  Handler  │   │  Handler  │   │  Handler  │               │
│   └───────────┘   └───────────┘   └───────────┘               │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Shared Infrastructure                       │   │
│  │  Database | Cache | Queue | Analytics | User Service    │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Bot Registry Pattern

### 2.1 Bot Registry Implementation

```javascript
// bot-registry/src/BotRegistry.js
const Redis = require('ioredis');
const { Sequelize, DataTypes } = require('sequelize');
const logger = require('./utils/logger');

class BotRegistry {
  constructor(options = {}) {
    this.redis = new Redis({
      host: options.redisHost || process.env.REDIS_HOST,
      port: options.redisPort || process.env.REDIS_PORT,
      password: options.redisPassword || process.env.REDIS_PASSWORD,
    });

    this.sequelize = options.sequelize || createSequelizeInstance();
    this.Bot = this.defineModel();
    this.cacheTTL = options.cacheTTL || 300; // 5 minutes
  }

  defineModel() {
    return this.sequelize.define('Bot', {
      id: {
        type: DataTypes.UUID,
        defaultValue: DataTypes.UUIDV4,
        primaryKey: true
      },
      tenantId: {
        type: DataTypes.STRING(100),
        allowNull: false,
        index: true
      },
      botName: {
        type: DataTypes.STRING(255),
        allowNull: false
      },
      lineChannelId: {
        type: DataTypes.STRING(100),
        unique: true,
        allowNull: false
      },
      lineChannelSecret: {
        type: DataTypes.TEXT,
        allowNull: false
      },
      lineChannelAccessToken: {
        type: DataTypes.TEXT,
        allowNull: false
      },
      destination: {
        type: DataTypes.STRING(100),
        unique: true,
        allowNull: true  // กำหนดตอน activate
      },
      isActive: {
        type: DataTypes.BOOLEAN,
        defaultValue: true
      },
      config: {
        type: DataTypes.JSONB,
        defaultValue: {}
      },
      plan: {
        type: DataTypes.STRING(50),
        defaultValue: 'basic'
      },
      stats: {
        type: DataTypes.JSONB,
        defaultValue: {
          totalMessages: 0,
          totalFollowers: 0,
          lastActivityAt: null
        }
      }
    }, {
      tableName: 'bots',
      indexes: [
        { fields: ['tenantId'] },
        { fields: ['destination'] },
        { fields: ['lineChannelId'] },
        { fields: ['isActive'] }
      ]
    });
  }

  // ดึง bot config ด้วย destination ID
  async getBotByDestination(destination) {
    const cacheKey = `bot:dest:${destination}`;
    
    // ลอง cache ก่อน
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    // ดึงจาก database
    const bot = await this.Bot.findOne({
      where: { destination, isActive: true }
    });

    if (!bot) {
      return null;
    }

    const botData = bot.toJSON();
    
    // Cache ไว้
    await this.redis.setex(cacheKey, this.cacheTTL, JSON.stringify(botData));
    
    return botData;
  }

  // ดึง bot ด้วย channel ID
  async getBotByChannelId(channelId) {
    const cacheKey = `bot:channel:${channelId}`;
    
    const cached = await this.redis.get(cacheKey);
    if (cached) {
      return JSON.parse(cached);
    }

    const bot = await this.Bot.findOne({
      where: { lineChannelId: channelId, isActive: true }
    });

    if (!bot) return null;

    const botData = bot.toJSON();
    await this.redis.setex(cacheKey, this.cacheTTL, JSON.stringify(botData));
    
    return botData;
  }

  // ลงทะเบียน bot ใหม่
  async registerBot(params) {
    const {
      tenantId,
      botName,
      lineChannelId,
      lineChannelSecret,
      lineChannelAccessToken,
      plan = 'basic'
    } = params;

    // ตรวจสอบว่า channel ID ซ้ำหรือไม่
    const existing = await this.Bot.findOne({ where: { lineChannelId } });
    if (existing) {
      throw new Error(`Bot with channel ID ${lineChannelId} already registered`);
    }

    const bot = await this.Bot.create({
      tenantId,
      botName,
      lineChannelId,
      lineChannelSecret,
      lineChannelAccessToken,
      plan,
      config: this.getDefaultConfig(plan)
    });

    logger.info(`Bot registered: ${botName} (${lineChannelId}) for tenant ${tenantId}`);
    
    return bot.toJSON();
  }

  // Update destination เมื่อ bot ส่ง webhook ครั้งแรก
  async setDestination(lineChannelId, destination) {
    const bot = await this.Bot.findOne({ where: { lineChannelId } });
    if (!bot) throw new Error('Bot not found');

    await bot.update({ destination });
    
    // Clear cache
    await this.clearBotCache(lineChannelId, destination);
    
    logger.info(`Set destination for bot ${lineChannelId}: ${destination}`);
  }

  // Deactivate bot
  async deactivateBot(botId) {
    const bot = await this.Bot.findByPk(botId);
    if (!bot) throw new Error('Bot not found');

    await bot.update({ isActive: false });
    await this.clearBotCache(bot.lineChannelId, bot.destination);
    
    logger.info(`Bot deactivated: ${bot.botName}`);
  }

  async clearBotCache(channelId, destination) {
    const keys = [
      `bot:channel:${channelId}`,
      destination && `bot:dest:${destination}`,
    ].filter(Boolean);
    
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }

  getDefaultConfig(plan) {
    const configs = {
      basic: {
        maxFollowers: 1000,
        maxBroadcastPerDay: 1,
        features: ['webhook', 'push', 'reply']
      },
      standard: {
        maxFollowers: 10000,
        maxBroadcastPerDay: 5,
        features: ['webhook', 'push', 'reply', 'broadcast', 'richMenu']
      },
      premium: {
        maxFollowers: -1, // unlimited
        maxBroadcastPerDay: -1, // unlimited
        features: ['webhook', 'push', 'reply', 'broadcast', 'richMenu', 'analytics', 'api']
      }
    };
    
    return configs[plan] || configs.basic;
  }

  // รายชื่อ bots ทั้งหมดของ tenant
  async getBotsByTenant(tenantId) {
    return this.Bot.findAll({
      where: { tenantId },
      attributes: { exclude: ['lineChannelSecret', 'lineChannelAccessToken'] }
    });
  }

  // Update stats
  async incrementStats(destination, field, value = 1) {
    const bot = await this.Bot.findOne({ where: { destination } });
    if (!bot) return;

    const stats = { ...bot.stats };
    stats[field] = (stats[field] || 0) + value;
    stats.lastActivityAt = new Date().toISOString();
    
    await bot.update({ stats });
  }
}

function createSequelizeInstance() {
  return new Sequelize(process.env.DATABASE_URL, {
    dialect: 'postgres',
    logging: false,
    pool: {
      max: 20,
      min: 2,
      acquire: 30000,
      idle: 10000
    }
  });
}

module.exports = BotRegistry;
```

---

## 3. Webhook Router

### 3.1 Single Endpoint, Multiple Bots

```javascript
// webhook-router/src/index.js
const express = require('express');
const crypto = require('crypto');
const BotRegistry = require('./BotRegistry');
const { publishBotEvent } = require('./publisher');
const logger = require('./utils/logger');

const app = express();
const registry = new BotRegistry();

// Raw body parser สำหรับ signature validation
app.use(express.raw({ type: 'application/json', limit: '10mb' }));

app.post('/webhook', async (req, res) => {
  // ตอบ 200 ทันที
  res.status(200).json({ status: 'ok' });

  const rawBody = req.body;
  let body;
  
  try {
    body = JSON.parse(rawBody.toString());
  } catch (e) {
    logger.warn('Invalid JSON body received');
    return;
  }

  const { destination, events } = body;

  if (!destination || !events || events.length === 0) {
    return;
  }

  // ดึง bot config จาก registry
  const bot = await registry.getBotByDestination(destination);
  
  if (!bot) {
    logger.warn(`Unknown destination: ${destination}`);
    return;
  }

  // Validate signature ด้วย bot's channel secret
  const signature = req.headers['x-line-signature'];
  if (!validateSignature(rawBody, bot.lineChannelSecret, signature)) {
    logger.warn(`Invalid signature for bot: ${bot.botName}`);
    return;
  }

  // ตรวจสอบ bot active status
  if (!bot.isActive) {
    logger.warn(`Received events for inactive bot: ${bot.botName}`);
    return;
  }

  // Publish events พร้อม bot context
  for (const event of events) {
    await publishBotEvent({
      botId: bot.id,
      tenantId: bot.tenantId,
      botName: bot.botName,
      destination,
      channelAccessToken: bot.lineChannelAccessToken,
      event,
      receivedAt: Date.now()
    });
  }

  // Update stats
  await registry.incrementStats(destination, 'totalMessages', events.length).catch(err => {
    logger.error('Failed to update stats:', err);
  });
});

// Webhook handler discovery endpoint (สำหรับ auto-discover destination)
app.get('/webhook/info/:channelId', async (req, res) => {
  const bot = await registry.getBotByChannelId(req.params.channelId);
  if (!bot) {
    return res.status(404).json({ error: 'Bot not found' });
  }
  
  res.json({
    registered: true,
    botName: bot.botName,
    destination: bot.destination,
    isActive: bot.isActive
  });
});

function validateSignature(body, channelSecret, signature) {
  if (!signature) return false;
  
  const hash = crypto
    .createHmac('SHA256', channelSecret)
    .update(body)
    .digest('base64');
  
  return hash === signature;
}

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
  logger.info(`Webhook Router listening on port ${PORT}`);
});
```

### 3.2 Bot Event Publisher

```javascript
// webhook-router/src/publisher.js
const { Queue } = require('bullmq');
const IORedis = require('ioredis');

const redis = new IORedis({
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT,
  maxRetriesPerRequest: null,
});

const eventQueue = new Queue('bot:events', { connection: redis });

async function publishBotEvent(payload) {
  const { event, botId, tenantId } = payload;
  
  // กำหนด priority
  const priority = getPriority(event.type);

  await eventQueue.add(
    `${tenantId}:${event.type}`,
    payload,
    {
      priority,
      jobId: `${botId}-${event.source?.userId}-${event.timestamp}`,
      attempts: 3,
      backoff: { type: 'exponential', delay: 1000 }
    }
  );
}

function getPriority(eventType) {
  const priorities = {
    message: 1,
    postback: 2,
    follow: 3,
    unfollow: 5,
  };
  return priorities[eventType] || 10;
}

module.exports = { publishBotEvent };
```

---

## 4. Bot Handler Framework

### 4.1 Abstract Bot Handler

```javascript
// bot-framework/src/BaseBotHandler.js
const { LineMessagingClient } = require('./LineMessagingClient');
const logger = require('./utils/logger');

class BaseBotHandler {
  constructor(botConfig) {
    this.botId = botConfig.botId;
    this.botName = botConfig.botName;
    this.tenantId = botConfig.tenantId;
    this.config = botConfig.config || {};
    
    this.lineClient = new LineMessagingClient(
      botConfig.channelAccessToken
    );

    // Register default handlers
    this.messageHandlers = new Map();
    this.postbackHandlers = new Map();
    this.eventHandlers = new Map();
    
    this.setupDefaultHandlers();
  }

  // Override ใน subclass
  setupDefaultHandlers() {
    // Default welcome message for new followers
    this.onFollow(async (event) => {
      await this.reply(event.replyToken, [{
        type: 'text',
        text: `ยินดีต้อนรับสู่ ${this.botName}! 🎉\nพิมพ์ "เมนู" เพื่อดูคำสั่งทั้งหมด`
      }]);
    });
  }

  // Message handlers
  onText(keyword, handler) {
    if (keyword instanceof RegExp) {
      this.messageHandlers.set(keyword, { type: 'regex', handler });
    } else {
      this.messageHandlers.set(keyword.toLowerCase(), { type: 'exact', handler });
    }
    return this;
  }

  onPostback(data, handler) {
    this.postbackHandlers.set(data, handler);
    return this;
  }

  onFollow(handler) {
    this.eventHandlers.set('follow', handler);
    return this;
  }

  onUnfollow(handler) {
    this.eventHandlers.set('unfollow', handler);
    return this;
  }

  // Process incoming event
  async processEvent(event) {
    const context = this.createContext(event);

    try {
      switch (event.type) {
        case 'message':
          await this.handleMessage(event, context);
          break;
        case 'postback':
          await this.handlePostback(event, context);
          break;
        case 'follow':
          await this.handleFollow(event, context);
          break;
        case 'unfollow':
          await this.handleUnfollow(event, context);
          break;
        default:
          logger.debug(`Unhandled event type: ${event.type}`);
      }
    } catch (error) {
      logger.error(`Error processing event for bot ${this.botName}:`, error);
      await this.handleError(error, event, context);
    }
  }

  async handleMessage(event, context) {
    if (event.message.type !== 'text') {
      await this.handleNonTextMessage(event, context);
      return;
    }

    const text = event.message.text.toLowerCase().trim();

    // ลอง exact match ก่อน
    if (this.messageHandlers.has(text)) {
      const { handler } = this.messageHandlers.get(text);
      await handler(event, context);
      return;
    }

    // ลอง regex match
    for (const [key, value] of this.messageHandlers) {
      if (key instanceof RegExp && key.test(text)) {
        await value.handler(event, context);
        return;
      }
    }

    // Default handler
    await this.handleUnknownMessage(event, context);
  }

  async handlePostback(event, context) {
    const { data } = event.postback;
    const handler = this.postbackHandlers.get(data);
    
    if (handler) {
      await handler(event, context);
    } else {
      logger.debug(`No postback handler for: ${data}`);
    }
  }

  async handleFollow(event, context) {
    const handler = this.eventHandlers.get('follow');
    if (handler) await handler(event, context);
  }

  async handleUnfollow(event, context) {
    const handler = this.eventHandlers.get('unfollow');
    if (handler) await handler(event, context);
  }

  // Override in subclass for custom behavior
  async handleNonTextMessage(event, context) {
    await this.reply(event.replyToken, [{
      type: 'text',
      text: 'ขออภัย ไม่รองรับข้อความประเภทนี้ครับ'
    }]);
  }

  async handleUnknownMessage(event, context) {
    // Default: ไม่ตอบ หรือ ตอบ default message
  }

  async handleError(error, event, context) {
    logger.error(`Bot ${this.botName} error:`, error);
  }

  // Helper methods
  async reply(replyToken, messages) {
    return this.lineClient.replyMessage(replyToken, 
      Array.isArray(messages) ? messages : [messages]
    );
  }

  async push(userId, messages) {
    return this.lineClient.pushMessage(userId,
      Array.isArray(messages) ? messages : [messages]
    );
  }

  createContext(event) {
    return {
      botId: this.botId,
      botName: this.botName,
      tenantId: this.tenantId,
      userId: event.source?.userId,
      groupId: event.source?.groupId,
      roomId: event.source?.roomId,
      config: this.config,
      lineClient: this.lineClient,
    };
  }
}

module.exports = BaseBotHandler;
```

### 4.2 Bot Factory Pattern

```javascript
// bot-framework/src/BotFactory.js
const BaseBotHandler = require('./BaseBotHandler');
const logger = require('./utils/logger');

class BotFactory {
  constructor() {
    this.botTypes = new Map();
    this.botInstances = new Map();
  }

  // ลงทะเบียน bot type
  registerBotType(typeName, BotClass) {
    if (!(BotClass.prototype instanceof BaseBotHandler) && BotClass !== BaseBotHandler) {
      throw new Error(`${typeName} must extend BaseBotHandler`);
    }
    this.botTypes.set(typeName, BotClass);
    logger.info(`Registered bot type: ${typeName}`);
  }

  // สร้าง bot instance
  createBot(botConfig) {
    const { botId, botType = 'default' } = botConfig;

    if (this.botInstances.has(botId)) {
      return this.botInstances.get(botId);
    }

    const BotClass = this.botTypes.get(botType) || BaseBotHandler;
    const bot = new BotClass(botConfig);
    
    this.botInstances.set(botId, bot);
    logger.info(`Created bot instance: ${botConfig.botName} (type: ${botType})`);
    
    return bot;
  }

  // ดึง bot instance
  getBot(botId) {
    return this.botInstances.get(botId);
  }

  // ลบ bot instance (เมื่อ deactivate)
  removeBot(botId) {
    this.botInstances.delete(botId);
  }

  // List all bots
  listBots() {
    return Array.from(this.botInstances.values()).map(bot => ({
      botId: bot.botId,
      botName: bot.botName,
      tenantId: bot.tenantId
    }));
  }
}

// Singleton factory
const factory = new BotFactory();

module.exports = factory;
```

### 4.3 Example Custom Bot

```javascript
// bots/EcommerceBotHandler.js
const BaseBotHandler = require('../bot-framework/src/BaseBotHandler');
const { ProductService } = require('../services/ProductService');
const { OrderService } = require('../services/OrderService');

class EcommerceBotHandler extends BaseBotHandler {
  constructor(botConfig) {
    super(botConfig);
    this.productService = new ProductService(botConfig.tenantId);
    this.orderService = new OrderService(botConfig.tenantId);
    
    this.setupHandlers();
  }

  setupHandlers() {
    // สินค้า
    this.onText('สินค้า', async (event, ctx) => {
      const products = await this.productService.getFeaturedProducts();
      await this.reply(event.replyToken, [this.buildProductCarousel(products)]);
    });

    this.onText('ตรวจสอบออเดอร์', async (event, ctx) => {
      const orders = await this.orderService.getRecentOrders(ctx.userId);
      await this.reply(event.replyToken, [this.buildOrderList(orders)]);
    });

    // Postback handlers
    this.onPostback('action=view_product', async (event, ctx) => {
      const params = new URLSearchParams(event.postback.data);
      const productId = params.get('productId');
      const product = await this.productService.getProduct(productId);
      await this.reply(event.replyToken, [this.buildProductDetail(product)]);
    });

    this.onPostback('action=add_to_cart', async (event, ctx) => {
      const params = new URLSearchParams(event.postback.data);
      const productId = params.get('productId');
      await this.orderService.addToCart(ctx.userId, productId);
      await this.reply(event.replyToken, [{
        type: 'text',
        text: '✅ เพิ่มลงตะกร้าแล้วครับ!\nพิมพ์ "ตะกร้า" เพื่อดูรายการ'
      }]);
    });

    // กฎทั่วไป
    this.onText(/^order[#-](\d+)$/i, async (event, ctx) => {
      const match = event.message.text.match(/^order[#-](\d+)$/i);
      const orderId = match[1];
      const order = await this.orderService.getOrder(orderId, ctx.userId);
      
      if (!order) {
        await this.reply(event.replyToken, [{ type: 'text', text: 'ไม่พบออเดอร์ครับ' }]);
        return;
      }

      await this.reply(event.replyToken, [this.buildOrderStatus(order)]);
    });
  }

  buildProductCarousel(products) {
    return {
      type: 'flex',
      altText: 'รายการสินค้า',
      contents: {
        type: 'carousel',
        contents: products.slice(0, 10).map(product => ({
          type: 'bubble',
          hero: {
            type: 'image',
            url: product.imageUrl,
            size: 'full',
            aspectRatio: '20:13',
            aspectMode: 'cover',
            action: {
              type: 'postback',
              data: `action=view_product&productId=${product.id}`
            }
          },
          body: {
            type: 'box',
            layout: 'vertical',
            contents: [
              {
                type: 'text',
                text: product.name,
                weight: 'bold',
                size: 'md'
              },
              {
                type: 'text',
                text: `฿${product.price.toLocaleString()}`,
                size: 'lg',
                color: '#00B900'
              }
            ]
          },
          footer: {
            type: 'box',
            layout: 'vertical',
            contents: [{
              type: 'button',
              action: {
                type: 'postback',
                label: '🛒 เพิ่มลงตะกร้า',
                data: `action=add_to_cart&productId=${product.id}`
              },
              style: 'primary',
              color: '#00B900'
            }]
          }
        }))
      }
    };
  }

  buildOrderStatus(order) {
    const statusMap = {
      pending: '⏳ รอยืนยัน',
      confirmed: '✅ ยืนยันแล้ว',
      shipping: '🚚 กำลังจัดส่ง',
      delivered: '📦 ส่งแล้ว',
      cancelled: '❌ ยกเลิกแล้ว'
    };

    return {
      type: 'flex',
      altText: `สถานะออเดอร์ #${order.id}`,
      contents: {
        type: 'bubble',
        header: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'text',
            text: `ออเดอร์ #${order.id}`,
            weight: 'bold',
            color: '#ffffff'
          }],
          backgroundColor: '#00B900'
        },
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: statusMap[order.status] || order.status,
              size: 'xl',
              weight: 'bold'
            },
            {
              type: 'separator',
              margin: 'md'
            },
            {
              type: 'box',
              layout: 'vertical',
              margin: 'md',
              contents: order.items.map(item => ({
                type: 'box',
                layout: 'horizontal',
                contents: [
                  { type: 'text', text: item.name, flex: 3, wrap: true },
                  { type: 'text', text: `x${item.quantity}`, flex: 1, align: 'center' },
                  { type: 'text', text: `฿${item.price}`, flex: 2, align: 'end' }
                ]
              }))
            },
            {
              type: 'separator',
              margin: 'md'
            },
            {
              type: 'box',
              layout: 'horizontal',
              margin: 'md',
              contents: [
                { type: 'text', text: 'รวม', weight: 'bold', flex: 4 },
                { type: 'text', text: `฿${order.total.toLocaleString()}`, weight: 'bold', flex: 2, align: 'end', color: '#00B900' }
              ]
            }
          ]
        }
      }
    };
  }
}

module.exports = EcommerceBotHandler;
```

---

## 5. Shared User Database

### 5.1 Cross-Bot User Management

```javascript
// shared/src/UserManager.js
const { Sequelize, DataTypes, Op } = require('sequelize');

class UserManager {
  constructor(sequelize) {
    this.sequelize = sequelize;
    
    // User model - เชื่อมต่อกับ LINE user
    this.User = sequelize.define('User', {
      id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
      lineUserId: { type: DataTypes.STRING(100), allowNull: false, unique: true },
      displayName: { type: DataTypes.STRING(255) },
      pictureUrl: { type: DataTypes.TEXT },
      language: { type: DataTypes.STRING(10), defaultValue: 'th' },
      globalTags: { type: DataTypes.JSONB, defaultValue: [] },
      globalProfile: { type: DataTypes.JSONB, defaultValue: {} }
    }, { tableName: 'global_users' });

    // BotUser - ความสัมพันธ์ระหว่าง User และ Bot
    this.BotUser = sequelize.define('BotUser', {
      id: { type: DataTypes.UUID, defaultValue: DataTypes.UUIDV4, primaryKey: true },
      userId: { type: DataTypes.UUID, allowNull: false },
      botId: { type: DataTypes.STRING(100), allowNull: false },
      tenantId: { type: DataTypes.STRING(100), allowNull: false },
      isFollowing: { type: DataTypes.BOOLEAN, defaultValue: true },
      followedAt: { type: DataTypes.DATE, defaultValue: DataTypes.NOW },
      lastSeenAt: { type: DataTypes.DATE, defaultValue: DataTypes.NOW },
      messageCount: { type: DataTypes.INTEGER, defaultValue: 0 },
      tags: { type: DataTypes.JSONB, defaultValue: [] },
      profile: { type: DataTypes.JSONB, defaultValue: {} }  // Bot-specific data
    }, {
      tableName: 'bot_users',
      indexes: [
        { fields: ['userId', 'botId'], unique: true },
        { fields: ['botId', 'isFollowing'] },
        { fields: ['tenantId'] }
      ]
    });

    // Associations
    this.User.hasMany(this.BotUser, { foreignKey: 'userId' });
    this.BotUser.belongsTo(this.User, { foreignKey: 'userId' });
  }

  // ดึงหรือสร้าง user (global)
  async getOrCreateGlobalUser(lineUserId, profileData = null) {
    let [user, created] = await this.User.findOrCreate({
      where: { lineUserId },
      defaults: {
        lineUserId,
        displayName: profileData?.displayName,
        pictureUrl: profileData?.pictureUrl,
        language: profileData?.language || 'th'
      }
    });

    if (!created && profileData) {
      await user.update({
        displayName: profileData.displayName || user.displayName,
        pictureUrl: profileData.pictureUrl || user.pictureUrl,
      });
    }

    return user;
  }

  // ดึงหรือสร้าง bot user
  async getOrCreateBotUser(lineUserId, botId, tenantId) {
    const user = await this.getOrCreateGlobalUser(lineUserId);

    let [botUser, created] = await this.BotUser.findOrCreate({
      where: { userId: user.id, botId },
      defaults: {
        userId: user.id,
        botId,
        tenantId,
        isFollowing: true,
        followedAt: new Date()
      }
    });

    if (!created) {
      await botUser.update({
        lastSeenAt: new Date(),
        messageCount: botUser.messageCount + 1
      });
    }

    return { user, botUser };
  }

  // Follow event
  async handleFollow(lineUserId, botId, tenantId) {
    const { user, botUser } = await this.getOrCreateBotUser(lineUserId, botId, tenantId);
    
    await botUser.update({
      isFollowing: true,
      followedAt: new Date()
    });

    return { user, botUser, isNewUser: false };
  }

  // Unfollow event
  async handleUnfollow(lineUserId, botId) {
    const user = await this.User.findOne({ where: { lineUserId } });
    if (!user) return;

    await this.BotUser.update(
      { isFollowing: false },
      { where: { userId: user.id, botId } }
    );
  }

  // ดึง users ที่ follow bot นี้
  async getBotFollowers(botId, options = {}) {
    const { page = 1, limit = 100, tags = [] } = options;
    
    const where = { botId, isFollowing: true };
    if (tags.length > 0) {
      where.tags = { [Op.overlap]: tags };
    }

    const { count, rows } = await this.BotUser.findAndCountAll({
      where,
      include: [{
        model: this.User,
        attributes: ['lineUserId', 'displayName', 'pictureUrl']
      }],
      limit,
      offset: (page - 1) * limit
    });

    return {
      followers: rows.map(bu => ({
        lineUserId: bu.User.lineUserId,
        displayName: bu.User.displayName,
        pictureUrl: bu.User.pictureUrl,
        tags: bu.tags,
        isFollowing: bu.isFollowing,
        messageCount: bu.messageCount,
        lastSeenAt: bu.lastSeenAt
      })),
      total: count,
      page,
      pages: Math.ceil(count / limit)
    };
  }

  // Cross-bot: users ที่ follow หลาย bot (ของ tenant เดียวกัน)
  async getCrossBotUsers(tenantId) {
    const result = await this.BotUser.findAll({
      where: { tenantId, isFollowing: true },
      include: [{ model: this.User }],
      attributes: ['userId', [this.sequelize.fn('COUNT', this.sequelize.col('botId')), 'botCount']],
      group: ['userId', 'User.id'],
      having: this.sequelize.literal('COUNT("BotUser"."botId") > 1')
    });

    return result;
  }
}

module.exports = UserManager;
```

---

## 6. Cross-Bot Messaging

### 6.1 Cross-Bot Message System

```javascript
// cross-bot/src/CrossBotMessenger.js
const logger = require('./utils/logger');

class CrossBotMessenger {
  constructor(options) {
    this.botRegistry = options.botRegistry;
    this.userManager = options.userManager;
    this.messageQueue = options.messageQueue;
  }

  // ส่ง message จาก bot A ไปยัง users ของ bot B
  async sendCrossBot(fromBotId, toBotId, lineUserId, messages) {
    // ตรวจสอบว่า user follow bot ปลายทางด้วยไหม
    const botUser = await this.userManager.BotUser.findOne({
      where: { 
        userId: lineUserId, 
        botId: toBotId,
        isFollowing: true 
      }
    });

    if (!botUser) {
      logger.warn(`User ${lineUserId} does not follow bot ${toBotId}`);
      return { success: false, reason: 'user_not_following' };
    }

    // ดึง channel access token ของ bot ปลายทาง
    const targetBot = await this.botRegistry.getBotById(toBotId);
    if (!targetBot || !targetBot.isActive) {
      return { success: false, reason: 'target_bot_inactive' };
    }

    // Queue message
    await this.messageQueue.addPush(
      lineUserId,
      messages,
      {
        channelAccessToken: targetBot.lineChannelAccessToken,
        fromBotId,
        toBotId
      }
    );

    logger.info(`Cross-bot message queued from ${fromBotId} to ${toBotId} for user ${lineUserId}`);
    return { success: true };
  }

  // Share user data ระหว่าง bots (ของ tenant เดียวกัน)
  async shareUserData(fromBotId, toBotId, lineUserId, data) {
    // ตรวจสอบว่า bots อยู่ใน tenant เดียวกัน
    const [fromBot, toBot] = await Promise.all([
      this.botRegistry.getBotById(fromBotId),
      this.botRegistry.getBotById(toBotId)
    ]);

    if (!fromBot || !toBot || fromBot.tenantId !== toBot.tenantId) {
      throw new Error('Cross-tenant data sharing not allowed');
    }

    // Update bot user profile ของ bot ปลายทาง
    const user = await this.userManager.User.findOne({ where: { lineUserId } });
    if (!user) throw new Error('User not found');

    await this.userManager.BotUser.update(
      { profile: { ...data, sharedFrom: fromBotId, sharedAt: new Date().toISOString() } },
      { where: { userId: user.id, botId: toBotId } }
    );

    return { success: true };
  }

  // Broadcast ไปยัง users ที่ follow ทั้ง bot A และ bot B
  async broadcastToCommonUsers(botIds, messages, channelAccessToken) {
    const commonUsers = await this.findCommonUsers(botIds);
    
    if (commonUsers.length === 0) {
      return { success: true, recipientCount: 0 };
    }

    await this.messageQueue.addMulticast(
      commonUsers.map(u => u.lineUserId),
      messages,
      { channelAccessToken }
    );

    return { 
      success: true, 
      recipientCount: commonUsers.length 
    };
  }

  async findCommonUsers(botIds) {
    const { sequelize } = this.userManager;
    const { BotUser, User } = this.userManager;
    
    // ดึง users ที่ follow ทุก bot ใน botIds
    const userIdsPerBot = await Promise.all(
      botIds.map(botId => 
        BotUser.findAll({
          where: { botId, isFollowing: true },
          attributes: ['userId'],
          raw: true
        }).then(rows => rows.map(r => r.userId))
      )
    );

    // หา intersection
    const commonUserIds = userIdsPerBot.reduce((acc, ids) => {
      return acc.filter(id => ids.includes(id));
    });

    // ดึง lineUserId
    const users = await User.findAll({
      where: { id: commonUserIds },
      attributes: ['lineUserId'],
      raw: true
    });

    return users;
  }
}

module.exports = CrossBotMessenger;
```

---

## 7. White-Label Bot System

### 7.1 White-Label Configuration

```javascript
// white-label/src/WhiteLabelConfig.js
const Redis = require('ioredis');

class WhiteLabelConfig {
  constructor(redis) {
    this.redis = redis;
    this.cacheTTL = 600; // 10 minutes
  }

  // สร้าง white-label config สำหรับ tenant ใหม่
  async createTenantConfig(tenantId, config) {
    const defaultConfig = {
      branding: {
        name: 'My Bot',
        logoUrl: null,
        primaryColor: '#00B900',
        secondaryColor: '#ffffff',
        welcomeMessage: 'ยินดีต้อนรับ!',
        defaultLanguage: 'th'
      },
      features: {
        richMenu: true,
        broadcast: true,
        analytics: true,
        customWebhook: false,
        aiResponse: false
      },
      limits: {
        maxFollowers: 1000,
        maxBroadcastPerDay: 1,
        maxMessagesPerDay: 10000,
        storageGB: 1
      },
      customDomain: null,
      webhookUrl: null,
      createdAt: new Date().toISOString()
    };

    const tenantConfig = this.deepMerge(defaultConfig, config);
    
    await this.redis.setex(
      `tenant:config:${tenantId}`,
      this.cacheTTL,
      JSON.stringify(tenantConfig)
    );

    // Persist ลง database ด้วย
    await this.saveToDB(tenantId, tenantConfig);
    
    return tenantConfig;
  }

  async getTenantConfig(tenantId) {
    const cached = await this.redis.get(`tenant:config:${tenantId}`);
    if (cached) return JSON.parse(cached);

    const config = await this.loadFromDB(tenantId);
    if (config) {
      await this.redis.setex(
        `tenant:config:${tenantId}`,
        this.cacheTTL,
        JSON.stringify(config)
      );
    }

    return config;
  }

  async updateTenantConfig(tenantId, updates) {
    const current = await this.getTenantConfig(tenantId);
    if (!current) throw new Error('Tenant config not found');

    const updated = this.deepMerge(current, updates);
    
    await Promise.all([
      this.redis.setex(`tenant:config:${tenantId}`, this.cacheTTL, JSON.stringify(updated)),
      this.saveToDB(tenantId, updated)
    ]);

    return updated;
  }

  deepMerge(target, source) {
    const result = { ...target };
    Object.keys(source).forEach(key => {
      if (source[key] && typeof source[key] === 'object' && !Array.isArray(source[key])) {
        result[key] = this.deepMerge(target[key] || {}, source[key]);
      } else {
        result[key] = source[key];
      }
    });
    return result;
  }
}

module.exports = WhiteLabelConfig;
```

---

## 8. SaaS Bot Platform Architecture

### 8.1 Multi-Tenant Admin API

```javascript
// admin-api/src/index.js
const express = require('express');
const jwt = require('jsonwebtoken');
const BotRegistry = require('./BotRegistry');
const WhiteLabelConfig = require('./WhiteLabelConfig');
const BillingService = require('./BillingService');
const logger = require('./utils/logger');

const app = express();
app.use(express.json());

const registry = new BotRegistry();
const whiteLabel = new WhiteLabelConfig();
const billing = new BillingService();

// JWT middleware สำหรับ tenant admins
const tenantAuth = async (req, res, next) => {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'Unauthorized' });

  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET);
    req.tenant = decoded;
    next();
  } catch (e) {
    return res.status(401).json({ error: 'Invalid token' });
  }
};

// Register new bot
app.post('/api/bots', tenantAuth, async (req, res) => {
  try {
    const {
      botName,
      lineChannelId,
      lineChannelSecret,
      lineChannelAccessToken,
      plan = 'basic'
    } = req.body;

    // ตรวจสอบ plan limit
    const canAddBot = await billing.checkBotLimit(req.tenant.tenantId);
    if (!canAddBot) {
      return res.status(403).json({ 
        error: 'Bot limit reached for your plan',
        upgradeUrl: process.env.BILLING_UPGRADE_URL
      });
    }

    const bot = await registry.registerBot({
      tenantId: req.tenant.tenantId,
      botName,
      lineChannelId,
      lineChannelSecret,
      lineChannelAccessToken,
      plan
    });

    // สร้าง default config
    await whiteLabel.createTenantConfig(bot.id, {
      branding: { name: botName }
    });

    res.json({
      botId: bot.id,
      botName: bot.botName,
      webhookUrl: `${process.env.WEBHOOK_BASE_URL}/webhook`,
      status: 'registered',
      instructions: [
        `1. ไปที่ LINE Developers Console`,
        `2. ตั้ง Webhook URL เป็น: ${process.env.WEBHOOK_BASE_URL}/webhook`,
        `3. เปิด Use webhook`,
        `4. ส่ง test event เพื่อยืนยัน`
      ]
    });

  } catch (error) {
    logger.error('Error registering bot:', error);
    if (error.message.includes('already registered')) {
      return res.status(409).json({ error: error.message });
    }
    res.status(500).json({ error: 'Internal server error' });
  }
});

// List bots ของ tenant
app.get('/api/bots', tenantAuth, async (req, res) => {
  const bots = await registry.getBotsByTenant(req.tenant.tenantId);
  res.json({ bots });
});

// Get bot stats
app.get('/api/bots/:botId/stats', tenantAuth, async (req, res) => {
  const bot = await registry.getBotById(req.params.botId);
  
  if (!bot || bot.tenantId !== req.tenant.tenantId) {
    return res.status(404).json({ error: 'Bot not found' });
  }

  // ดึง stats จาก analytics service
  const stats = await getAnalyticsStats(bot.id, req.query);
  
  res.json({
    botId: bot.id,
    botName: bot.botName,
    stats: {
      ...bot.stats,
      ...stats
    }
  });
});

// Update bot config
app.put('/api/bots/:botId/config', tenantAuth, async (req, res) => {
  try {
    const bot = await registry.getBotById(req.params.botId);
    
    if (!bot || bot.tenantId !== req.tenant.tenantId) {
      return res.status(404).json({ error: 'Bot not found' });
    }

    const updatedConfig = await whiteLabel.updateTenantConfig(bot.id, req.body);
    
    res.json({ success: true, config: updatedConfig });

  } catch (error) {
    logger.error('Error updating bot config:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Deactivate bot
app.delete('/api/bots/:botId', tenantAuth, async (req, res) => {
  try {
    const bot = await registry.getBotById(req.params.botId);
    
    if (!bot || bot.tenantId !== req.tenant.tenantId) {
      return res.status(404).json({ error: 'Bot not found' });
    }

    await registry.deactivateBot(bot.id);
    res.json({ success: true, message: 'Bot deactivated' });

  } catch (error) {
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Tenant Dashboard Stats
app.get('/api/dashboard', tenantAuth, async (req, res) => {
  const [bots, usage, billing] = await Promise.all([
    registry.getBotsByTenant(req.tenant.tenantId),
    getUsageStats(req.tenant.tenantId),
    billing.getTenantBilling(req.tenant.tenantId)
  ]);

  res.json({
    summary: {
      totalBots: bots.length,
      activeBots: bots.filter(b => b.isActive).length,
      totalFollowers: bots.reduce((sum, b) => sum + (b.stats?.totalFollowers || 0), 0),
      totalMessages: bots.reduce((sum, b) => sum + (b.stats?.totalMessages || 0), 0)
    },
    bots: bots.map(b => ({
      id: b.id,
      name: b.botName,
      isActive: b.isActive,
      followers: b.stats?.totalFollowers || 0,
      messages: b.stats?.totalMessages || 0
    })),
    usage,
    billing
  });
});

const PORT = process.env.PORT || 3003;
app.listen(PORT, () => {
  logger.info(`Admin API running on port ${PORT}`);
});
```

---

## 9. Tenant Isolation

### 9.1 Data Isolation Strategy

```javascript
// tenant-isolation/src/TenantIsolation.js

// Row-Level Security (RLS) สำหรับ PostgreSQL
const RLS_SETUP_SQL = `
-- Enable Row Level Security
ALTER TABLE bot_users ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE analytics_events ENABLE ROW LEVEL SECURITY;

-- Policy: tenant can only see their data
CREATE POLICY tenant_isolation_bot_users ON bot_users
  USING (tenant_id = current_setting('app.current_tenant_id')::text);

CREATE POLICY tenant_isolation_messages ON messages
  USING (tenant_id = current_setting('app.current_tenant_id')::text);

-- Set current tenant in connection
-- SET app.current_tenant_id = 'tenant123';
`;

class TenantAwareRepository {
  constructor(sequelize, tenantId) {
    this.sequelize = sequelize;
    this.tenantId = tenantId;
  }

  // Execute query ใน tenant context
  async query(sql, params) {
    return this.sequelize.transaction(async (t) => {
      // Set tenant context
      await this.sequelize.query(
        `SET LOCAL app.current_tenant_id = '${this.tenantId}'`,
        { transaction: t }
      );
      
      return this.sequelize.query(sql, { 
        replacements: params,
        transaction: t 
      });
    });
  }

  // Tenant-scoped model operations
  async findAll(Model, where = {}) {
    return Model.findAll({
      where: { ...where, tenantId: this.tenantId }
    });
  }

  async create(Model, data) {
    return Model.create({ ...data, tenantId: this.tenantId });
  }
}

// Rate limiting per tenant
class TenantRateLimiter {
  constructor(redis) {
    this.redis = redis;
  }

  async checkLimit(tenantId, action, limit, windowSeconds = 86400) {
    const key = `ratelimit:${tenantId}:${action}:${Math.floor(Date.now() / (windowSeconds * 1000))}`;
    
    const current = await this.redis.incr(key);
    
    if (current === 1) {
      await this.redis.expire(key, windowSeconds);
    }

    if (current > limit) {
      const ttl = await this.redis.ttl(key);
      throw new TenantLimitError(
        `Rate limit exceeded for ${action}`,
        { limit, current, resetIn: ttl }
      );
    }

    return { allowed: true, remaining: limit - current };
  }
}

class TenantLimitError extends Error {
  constructor(message, details) {
    super(message);
    this.name = 'TenantLimitError';
    this.details = details;
  }
}

module.exports = { TenantAwareRepository, TenantRateLimiter, TenantLimitError };
```

---

## 10. Complete Docker Compose Setup

```yaml
# docker-compose.multi-bot.yml
version: '3.9'

x-bot-env: &bot-env
  NODE_ENV: production
  REDIS_HOST: redis
  DATABASE_URL: postgresql://linebot:password@postgres:5432/linebot_multibot
  RABBITMQ_URL: amqp://admin:password@rabbitmq:5672
  INTERNAL_SERVICE_TOKEN: ${INTERNAL_SERVICE_TOKEN}
  JWT_SECRET: ${JWT_SECRET}
  LOG_LEVEL: info

services:
  # ===========================
  # Core Infrastructure
  # ===========================

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: linebot
      POSTGRES_PASSWORD: password
      POSTGRES_DB: linebot_multibot
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./sql/init-multibot.sql:/docker-entrypoint-initdb.d/01-init.sql
      - ./sql/rls-policies.sql:/docker-entrypoint-initdb.d/02-rls.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U linebot"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - bot-network

  redis:
    image: redis:7.2-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD} --maxmemory 1gb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
    networks:
      - bot-network

  rabbitmq:
    image: rabbitmq:3.12-management
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: password
    volumes:
      - rabbitmq_data:/var/lib/rabbitmq
    networks:
      - bot-network

  # ===========================
  # Platform Services
  # ===========================

  webhook-router:
    build:
      context: ./webhook-router
    environment:
      <<: *bot-env
      PORT: 3001
    ports:
      - "3001:3001"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_started
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
    networks:
      - bot-network

  bot-event-worker:
    build:
      context: ./bot-event-worker
    environment:
      <<: *bot-env
      WORKER_CONCURRENCY: 20
    depends_on:
      rabbitmq:
        condition: service_started
      postgres:
        condition: service_healthy
    deploy:
      replicas: 5
      resources:
        limits:
          cpus: '1'
          memory: 512M
    networks:
      - bot-network

  admin-api:
    build:
      context: ./admin-api
    environment:
      <<: *bot-env
      PORT: 3003
      WEBHOOK_BASE_URL: ${WEBHOOK_BASE_URL}
      BILLING_UPGRADE_URL: ${BILLING_UPGRADE_URL}
    ports:
      - "3003:3003"
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - bot-network

  user-service:
    build:
      context: ./user-service
    environment:
      <<: *bot-env
      PORT: 3005
    ports:
      - "3005:3005"
    networks:
      - bot-network

  notification-service:
    build:
      context: ./notification-service
    environment:
      <<: *bot-env
      PORT: 3006
    networks:
      - bot-network

  # ===========================
  # API Gateway (Nginx)
  # ===========================

  nginx:
    image: nginx:alpine
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - certbot_certs:/etc/letsencrypt:ro
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - webhook-router
      - admin-api
    networks:
      - bot-network

  # ===========================
  # Monitoring
  # ===========================

  grafana:
    image: grafana/grafana:10.1.0
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana:/etc/grafana/provisioning
    ports:
      - "3000:3000"
    networks:
      - bot-network

volumes:
  postgres_data:
  redis_data:
  rabbitmq_data:
  grafana_data:
  certbot_certs:

networks:
  bot-network:
    driver: bridge
```

---

## 11. Nginx Configuration

```nginx
# nginx/conf.d/multibot.conf
upstream webhook_backend {
    least_conn;
    server webhook-router:3001;
    server webhook-router:3001;  # docker replicas
    keepalive 32;
}

upstream admin_backend {
    server admin-api:3003;
    keepalive 16;
}

# Webhook endpoint - ต้องรับได้เร็ว
server {
    listen 443 ssl http2;
    server_name api.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;

    # Security headers
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";

    # Webhook endpoint
    location /webhook {
        proxy_pass http://webhook_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 5s;
        proxy_send_timeout 5s;
        proxy_read_timeout 5s;

        # Buffer settings
        proxy_buffering off;
        proxy_request_buffering off;

        # Rate limiting
        limit_req zone=webhook burst=1000 nodelay;
        limit_req_status 429;
    }

    # Admin API
    location /api/ {
        proxy_pass http://admin_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        
        # Cache for GET requests
        proxy_cache_methods GET HEAD;
        proxy_cache_valid 200 60s;
    }
}

# Rate limiting zones
limit_req_zone $binary_remote_addr zone=webhook:10m rate=1000r/s;
```

---

## 12. Database Schema

```sql
-- sql/init-multibot.sql

-- Tenants
CREATE TABLE tenants (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    plan VARCHAR(50) DEFAULT 'basic',
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Bots
CREATE TABLE bots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID REFERENCES tenants(id),
    bot_name VARCHAR(255) NOT NULL,
    line_channel_id VARCHAR(100) UNIQUE NOT NULL,
    line_channel_secret TEXT NOT NULL,
    line_channel_access_token TEXT NOT NULL,
    destination VARCHAR(100) UNIQUE,
    is_active BOOLEAN DEFAULT true,
    plan VARCHAR(50) DEFAULT 'basic',
    config JSONB DEFAULT '{}',
    stats JSONB DEFAULT '{"totalMessages": 0, "totalFollowers": 0}',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Global Users
CREATE TABLE global_users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    line_user_id VARCHAR(100) UNIQUE NOT NULL,
    display_name VARCHAR(255),
    picture_url TEXT,
    language VARCHAR(10) DEFAULT 'th',
    global_tags JSONB DEFAULT '[]',
    global_profile JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Bot Users (relationship between users and bots)
CREATE TABLE bot_users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES global_users(id),
    bot_id UUID REFERENCES bots(id),
    tenant_id VARCHAR(100) NOT NULL,
    is_following BOOLEAN DEFAULT true,
    followed_at TIMESTAMP DEFAULT NOW(),
    unfollowed_at TIMESTAMP,
    last_seen_at TIMESTAMP DEFAULT NOW(),
    message_count INTEGER DEFAULT 0,
    tags JSONB DEFAULT '[]',
    profile JSONB DEFAULT '{}',
    UNIQUE(user_id, bot_id)
);

-- Indexes
CREATE INDEX idx_bots_tenant ON bots(tenant_id);
CREATE INDEX idx_bots_destination ON bots(destination);
CREATE INDEX idx_bot_users_bot ON bot_users(bot_id, is_following);
CREATE INDEX idx_bot_users_tenant ON bot_users(tenant_id);
CREATE INDEX idx_global_users_line_id ON global_users(line_user_id);
```

---

## 13. สรุปและ Best Practices

### 13.1 Architecture Decision Guide

```
สิ่งที่ต้องตัดสินใจ:

1. Single Database vs Multi-Database per Tenant?
   - Single DB + RLS: ง่ายกว่า, ต้นทุนต่ำกว่า, เหมาะกับ < 1000 tenants
   - Multi-DB: Isolation สูงกว่า, แต่ complex, เหมาะกับ enterprise

2. Shared vs Dedicated Infrastructure?
   - Shared Redis: ใช้ namespace separation (tenant:{id}:*)
   - Shared Queue: ใช้ routing + tenant tagging
   - Dedicated: เหมาะกับ enterprise tier เท่านั้น

3. Bot Config Storage?
   - Database: persistent, queryable
   - Redis Cache: fast access
   - Both: ดีที่สุด (write to DB, read from cache)

4. Cross-Bot Communication?
   - ต้องอยู่ใน tenant เดียวกัน เท่านั้น
   - ผ่าน message queue (ไม่ใช่ direct call)
   - Audit log ทุก cross-bot operation
```

### 13.2 Scaling Guide

```
Traffic Thresholds:

< 10 bots, < 100k messages/day:
  - 1-2 webhook-router instances
  - 1-2 bot-event-workers
  - Single Redis (no cluster)
  - Single PostgreSQL

10-100 bots, < 1M messages/day:
  - 3-5 webhook-router instances
  - 5-10 bot-event-workers
  - Redis Cluster (3 shards)
  - PostgreSQL + Read Replicas

100+ bots, > 1M messages/day:
  - 10+ webhook-router instances
  - 20+ bot-event-workers (auto-scaling)
  - Redis Cluster (6+ shards)
  - PostgreSQL Cluster (Citus หรือ sharding)
  - Per-tier dedicated infrastructure
```

บทถัดไปจะพูดถึง A/B Testing Framework สำหรับ LINE Bot เพื่อทดสอบและปรับปรุง engagement
