# Part 75: Performance Optimization สำหรับ LINE Bot

## บทนำ

Performance เป็นสิ่งสำคัญมากสำหรับ LINE Bot เนื่องจาก LINE Platform กำหนดว่า webhook ต้องตอบกลับภายใน **1 วินาที** และ user จะรู้สึกได้ถ้า bot ตอบช้า บทนี้จะครอบคลุมทุกด้านของ performance optimization ตั้งแต่ database ไปจนถึง CDN

---

## 1. Performance Bottlenecks

### 1.1 Common Bottlenecks

```
┌─────────────────────────────────────────────────────────┐
│              Performance Bottleneck Analysis             │
│                                                         │
│  LINE Platform                                          │
│      │                                                  │
│      ▼                                                  │
│  API Gateway ──────────── Bottleneck 1: TLS/HTTP overhead│
│      │                                                  │
│      ▼                                                  │
│  Webhook Handler ─────── Bottleneck 2: Sync processing │
│      │                                                  │
│      ▼                                                  │
│  Message Handler ─────── Bottleneck 3: Serial processing│
│      │                                                  │
│      ├──→ Database ───── Bottleneck 4: N+1 queries      │
│      │                                                  │
│      ├──→ Redis ──────── Bottleneck 5: Cache misses     │
│      │                                                  │
│      └──→ LINE API ───── Bottleneck 6: Network latency  │
│                                                         │
│  Response Time Budget: 1000ms total                     │
│  ├── API Gateway: 5ms                                   │
│  ├── Webhook parsing: 5ms                               │
│  ├── Signature validation: 2ms                         │
│  ├── Queue publish: 10ms                                │
│  └── Response send: 5ms                                 │
│      TOTAL: ~30ms (target: < 100ms)                    │
└─────────────────────────────────────────────────────────┘
```

### 1.2 Profiling Setup

```javascript
// profiling/src/profiler.js
const { performance, PerformanceObserver } = require('perf_hooks');
const logger = require('../utils/logger');

// Performance timing middleware
function timingMiddleware(req, res, next) {
  const start = performance.now();
  const requestId = req.headers['x-request-id'] || generateId();
  
  req.timings = {
    requestId,
    start,
    checkpoints: []
  };

  req.checkpoint = (name) => {
    req.timings.checkpoints.push({
      name,
      elapsed: (performance.now() - start).toFixed(2)
    });
  };

  res.on('finish', () => {
    const total = (performance.now() - start).toFixed(2);
    
    logger.info('Request performance', {
      requestId,
      method: req.method,
      path: req.path,
      status: res.statusCode,
      totalMs: total,
      checkpoints: req.timings.checkpoints
    });

    // Alert ถ้าช้าเกิน 500ms
    if (parseFloat(total) > 500) {
      logger.warn('SLOW REQUEST detected', {
        requestId,
        totalMs: total,
        path: req.path,
        checkpoints: req.timings.checkpoints
      });
    }
  });

  next();
}

// Async function timer
function timeAsync(name, fn) {
  return async (...args) => {
    const start = performance.now();
    try {
      const result = await fn(...args);
      const elapsed = (performance.now() - start).toFixed(2);
      logger.debug(`${name}: ${elapsed}ms`);
      return result;
    } catch (error) {
      const elapsed = (performance.now() - start).toFixed(2);
      logger.error(`${name} failed after ${elapsed}ms`);
      throw error;
    }
  };
}

function generateId() {
  return Math.random().toString(36).substring(2, 11);
}

module.exports = { timingMiddleware, timeAsync };
```

---

## 2. Response Time Optimization

### 2.1 Async Response Pattern

```javascript
// optimization/src/asyncResponse.js
const express = require('express');
const router = express.Router();
const { publishToQueue } = require('../queue/producer');
const logger = require('../utils/logger');

// ❌ SLOW: Synchronous processing
router.post('/webhook-slow', async (req, res) => {
  const { events } = req.body;
  
  // Process events synchronously - ช้ามาก!
  for (const event of events) {
    await processEvent(event); // อาจใช้เวลา 200-2000ms
    await sendReply(event);    // + อีก 100-500ms
  }
  
  res.json({ status: 'ok' }); // ตอบช้าเกินไป!
});

// ✅ FAST: Async pattern
router.post('/webhook', async (req, res) => {
  // ตอบ LINE ทันที (< 50ms)
  res.status(200).json({ status: 'ok' });
  
  // Process async ใน background
  const { events, destination } = req.body;
  
  // Publish ทุก events พร้อมกัน
  const publishPromises = events.map(event => 
    publishToQueue(event, destination).catch(err => 
      logger.error('Failed to queue event:', err)
    )
  );
  
  // Fire and forget (ไม่ await)
  Promise.all(publishPromises);
});

// ✅ BETTER: Pre-validate + async
router.post('/webhook-optimal', (req, res) => {
  // Validate signature synchronously (already done in middleware)
  const { events, destination } = req.body;
  
  if (!events?.length) {
    return res.status(200).json({ status: 'ok' }); // Empty body
  }

  // ตอบทันทีโดยไม่ await
  res.status(200).json({ status: 'ok' });

  // Queue processing in background (non-blocking)
  setImmediate(async () => {
    await Promise.allSettled(
      events.map(event => publishToQueue(event, destination))
    );
  });
});
```

### 2.2 Connection Pooling

```javascript
// optimization/src/connectionPool.js
const { Pool } = require('pg');
const IORedis = require('ioredis');
const axios = require('axios');
const http = require('http');
const https = require('https');

// ===========================
// PostgreSQL Connection Pool
// ===========================
const pgPool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                    // Maximum connections
  min: 5,                     // Minimum idle connections
  idleTimeoutMillis: 30000,   // Close idle after 30s
  connectionTimeoutMillis: 2000,
  
  // SSL สำหรับ production
  ssl: process.env.NODE_ENV === 'production' 
    ? { rejectUnauthorized: false }
    : false
});

// Monitor pool
pgPool.on('connect', () => {
  logger.debug('New database connection established');
});

pgPool.on('error', (err) => {
  logger.error('Database pool error:', err);
});

// ===========================
// Redis Connection
// ===========================
const redisClient = new IORedis({
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT,
  password: process.env.REDIS_PASSWORD,
  
  // Connection pool
  maxRetriesPerRequest: 3,
  enableReadyCheck: true,
  
  // Retry strategy
  retryStrategy(times) {
    const delay = Math.min(times * 50, 2000);
    return delay;
  },
  
  // Lazy connect
  lazyConnect: false,
  
  // Keep alive
  keepAlive: 30000,
});

// ===========================
// HTTP/HTTPS Connection Pool
// ===========================
const httpAgent = new http.Agent({
  keepAlive: true,
  maxSockets: 50,
  maxFreeSockets: 10,
  timeout: 5000,
  freeSocketTimeout: 30000
});

const httpsAgent = new https.Agent({
  keepAlive: true,
  maxSockets: 50,
  maxFreeSockets: 10,
  timeout: 5000,
  freeSocketTimeout: 30000
});

// Line API client with connection pooling
const lineApiClient = axios.create({
  baseURL: 'https://api.line.me/v2/bot',
  httpAgent,
  httpsAgent,
  timeout: 10000,
  headers: {
    'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
    'Content-Type': 'application/json'
  }
});

module.exports = { pgPool, redisClient, lineApiClient };
```

---

## 3. Database Query Optimization

### 3.1 N+1 Query Problem

```javascript
// optimization/src/queryOptimization.js

// ❌ N+1 Problem
async function getUsersWithOrders_SLOW(userIds) {
  const users = await User.findAll({ where: { id: userIds } });
  
  // สร้าง N queries สำหรับ N users!
  for (const user of users) {
    user.orders = await Order.findAll({ where: { userId: user.id } });
  }
  
  return users;
}

// ✅ Eager Loading (JOIN)
async function getUsersWithOrders_FAST(userIds) {
  return User.findAll({
    where: { id: userIds },
    include: [{
      model: Order,
      where: { status: 'active' },
      required: false,
      // Select only needed fields
      attributes: ['id', 'total', 'status', 'createdAt']
    }],
    // Select only needed user fields
    attributes: ['id', 'lineUserId', 'displayName']
  });
}

// ✅ Bulk queries
async function getOrdersByUsers_FAST(userIds) {
  // ดึงทั้งหมดใน 1 query
  const orders = await Order.findAll({
    where: { 
      userId: userIds,
      status: 'active'
    },
    attributes: ['id', 'userId', 'total', 'status']
  });
  
  // Group by userId ใน application
  const ordersByUser = orders.reduce((acc, order) => {
    if (!acc[order.userId]) acc[order.userId] = [];
    acc[order.userId].push(order);
    return acc;
  }, {});
  
  return ordersByUser;
}

// ===========================
// Index Usage
// ===========================

// Bad query - ไม่ใช้ index
const badQuery = `
  SELECT * FROM bot_users 
  WHERE tags @> '["vip"]'::jsonb  -- JSONB containment can be slow without index
  AND is_following = true
`;

// Good - ใช้ proper indexes
const goodQuery = `
  SELECT line_user_id, display_name, last_seen_at 
  FROM bot_users bu
  JOIN global_users gu ON bu.user_id = gu.id
  WHERE bu.bot_id = $1 
    AND bu.is_following = true
    AND bu.tags @> $2::jsonb
  ORDER BY bu.last_seen_at DESC
  LIMIT 100
`;

// Database indexes ที่ควรมี
const INDEX_CREATION_SQL = `
-- Basic indexes
CREATE INDEX CONCURRENTLY idx_bot_users_bot_following 
  ON bot_users(bot_id, is_following) 
  WHERE is_following = true;

CREATE INDEX CONCURRENTLY idx_bot_users_last_seen 
  ON bot_users(last_seen_at DESC);

-- JSONB GIN index สำหรับ tags
CREATE INDEX CONCURRENTLY idx_bot_users_tags 
  ON bot_users USING GIN(tags);

-- Composite index สำหรับ common queries
CREATE INDEX CONCURRENTLY idx_bot_users_bot_following_seen
  ON bot_users(bot_id, is_following, last_seen_at DESC);

-- Partial index สำหรับ active users
CREATE INDEX CONCURRENTLY idx_active_bot_users
  ON bot_users(bot_id, last_seen_at DESC)
  WHERE is_following = true AND last_seen_at > NOW() - INTERVAL '30 days';
`;
```

### 3.2 Query Optimization Techniques

```javascript
// optimization/src/advancedQueries.js
const { Sequelize, Op } = require('sequelize');

class OptimizedUserRepository {
  constructor(sequelize) {
    this.sequelize = sequelize;
  }

  // Pagination with cursor (faster than OFFSET for large tables)
  async getUsersWithCursor(botId, { cursor, limit = 100 }) {
    const where = { botId, isFollowing: true };
    
    if (cursor) {
      // cursor = base64 encoded { lastSeenAt, id }
      const decoded = JSON.parse(Buffer.from(cursor, 'base64').toString());
      where[Op.or] = [
        { lastSeenAt: { [Op.lt]: decoded.lastSeenAt } },
        {
          lastSeenAt: decoded.lastSeenAt,
          id: { [Op.lt]: decoded.id }
        }
      ];
    }

    const users = await this.sequelize.models.BotUser.findAll({
      where,
      limit: limit + 1,
      order: [['lastSeenAt', 'DESC'], ['id', 'DESC']],
      include: [{
        model: this.sequelize.models.GlobalUser,
        attributes: ['lineUserId', 'displayName']
      }]
    });

    const hasMore = users.length > limit;
    const items = users.slice(0, limit);
    
    const nextCursor = hasMore 
      ? Buffer.from(JSON.stringify({
          lastSeenAt: items[items.length - 1].lastSeenAt,
          id: items[items.length - 1].id
        })).toString('base64')
      : null;

    return { items, nextCursor, hasMore };
  }

  // Batch upsert (เร็วกว่า individual inserts)
  async batchUpsertUsers(users) {
    const BATCH_SIZE = 1000;
    
    for (let i = 0; i < users.length; i += BATCH_SIZE) {
      const batch = users.slice(i, i + BATCH_SIZE);
      
      await this.sequelize.models.GlobalUser.bulkCreate(batch, {
        updateOnDuplicate: ['displayName', 'pictureUrl', 'updatedAt'],
        returning: false
      });
    }
  }

  // Aggregation query
  async getFollowerStats(botId) {
    const result = await this.sequelize.query(`
      SELECT
        COUNT(*) FILTER (WHERE is_following = true) as total_followers,
        COUNT(*) FILTER (WHERE is_following = false) as total_unfollowers,
        COUNT(*) FILTER (WHERE last_seen_at > NOW() - INTERVAL '7 days' AND is_following = true) as active_7d,
        COUNT(*) FILTER (WHERE last_seen_at > NOW() - INTERVAL '30 days' AND is_following = true) as active_30d,
        AVG(message_count) as avg_messages,
        MAX(last_seen_at) as last_activity
      FROM bot_users
      WHERE bot_id = :botId
    `, {
      replacements: { botId },
      type: Sequelize.QueryTypes.SELECT
    });

    return result[0];
  }

  // Full-text search (สำหรับ search ชื่อ user)
  async searchUsers(botId, query) {
    return this.sequelize.query(`
      SELECT 
        gu.line_user_id,
        gu.display_name,
        bu.is_following,
        bu.last_seen_at,
        ts_rank(to_tsvector('simple', gu.display_name), 
                plainto_tsquery('simple', :query)) as rank
      FROM bot_users bu
      JOIN global_users gu ON bu.user_id = gu.id
      WHERE bu.bot_id = :botId
        AND to_tsvector('simple', gu.display_name) @@ plainto_tsquery('simple', :query)
      ORDER BY rank DESC, bu.last_seen_at DESC
      LIMIT 20
    `, {
      replacements: { botId, query },
      type: Sequelize.QueryTypes.SELECT
    });
  }
}

module.exports = OptimizedUserRepository;
```

---

## 4. Caching Strategies

### 4.1 Multi-Level Cache

```javascript
// optimization/src/cacheManager.js
const Redis = require('ioredis');
const logger = require('../utils/logger');

class MultiLevelCache {
  constructor(redis) {
    this.redis = redis;
    
    // L1: In-memory cache (fastest, but process-local)
    this.l1Cache = new Map();
    this.l1MaxSize = 1000;
    this.l1TTL = 60 * 1000; // 60 seconds
    
    // Cleanup L1 cache ทุก 30 วินาที
    setInterval(() => this.cleanL1Cache(), 30000);
  }

  async get(key) {
    // L1 check
    const l1Result = this.getFromL1(key);
    if (l1Result !== undefined) {
      return l1Result;
    }

    // L2 (Redis) check
    const l2Result = await this.redis.get(key);
    if (l2Result !== null) {
      const parsed = JSON.parse(l2Result);
      // Populate L1
      this.setInL1(key, parsed);
      return parsed;
    }

    return null;
  }

  async set(key, value, ttlSeconds = 300) {
    // Set in both levels
    this.setInL1(key, value);
    await this.redis.setex(key, ttlSeconds, JSON.stringify(value));
  }

  async delete(key) {
    this.l1Cache.delete(key);
    await this.redis.del(key);
  }

  getFromL1(key) {
    const entry = this.l1Cache.get(key);
    if (!entry) return undefined;
    
    if (Date.now() > entry.expiresAt) {
      this.l1Cache.delete(key);
      return undefined;
    }
    
    return entry.value;
  }

  setInL1(key, value) {
    // LRU eviction ถ้า cache เต็ม
    if (this.l1Cache.size >= this.l1MaxSize) {
      const oldestKey = this.l1Cache.keys().next().value;
      this.l1Cache.delete(oldestKey);
    }
    
    this.l1Cache.set(key, {
      value,
      expiresAt: Date.now() + this.l1TTL
    });
  }

  cleanL1Cache() {
    const now = Date.now();
    for (const [key, entry] of this.l1Cache) {
      if (now > entry.expiresAt) {
        this.l1Cache.delete(key);
      }
    }
  }

  // Pattern: Cache-aside
  async getOrFetch(key, fetchFn, ttlSeconds = 300) {
    const cached = await this.get(key);
    if (cached !== null) return cached;

    const fresh = await fetchFn();
    if (fresh !== null && fresh !== undefined) {
      await this.set(key, fresh, ttlSeconds);
    }

    return fresh;
  }
}

module.exports = MultiLevelCache;
```

### 4.2 User Profile Cache

```javascript
// optimization/src/userProfileCache.js
const MultiLevelCache = require('./cacheManager');
const Redis = require('ioredis');
const axios = require('axios');

const redis = new Redis({ host: process.env.REDIS_HOST });
const cache = new MultiLevelCache(redis);

class UserProfileCache {
  constructor() {
    this.LINE_API_URL = 'https://api.line.me/v2/bot/profile';
    this.accessToken = process.env.LINE_CHANNEL_ACCESS_TOKEN;
    
    // Cache TTLs
    this.PROFILE_TTL = 3600;       // 1 hour
    this.FOLLOWING_STATUS_TTL = 300; // 5 minutes
  }

  // ดึง profile (with cache)
  async getProfile(lineUserId) {
    const key = `profile:${lineUserId}`;
    
    return cache.getOrFetch(key, async () => {
      const response = await axios.get(
        `${this.LINE_API_URL}/${lineUserId}`,
        {
          headers: { Authorization: `Bearer ${this.accessToken}` },
          timeout: 3000
        }
      );
      return response.data;
    }, this.PROFILE_TTL);
  }

  // Warm up cache สำหรับ users ที่กำลังจะใช้
  async warmUpCache(lineUserIds) {
    const uncachedIds = [];
    
    // ตรวจสอบว่า cache ไหนยังไม่มี
    for (const userId of lineUserIds) {
      const cached = await cache.get(`profile:${userId}`);
      if (!cached) uncachedIds.push(userId);
    }

    if (uncachedIds.length === 0) return;

    // Fetch profiles in parallel (max 10 at a time)
    const BATCH_SIZE = 10;
    for (let i = 0; i < uncachedIds.length; i += BATCH_SIZE) {
      const batch = uncachedIds.slice(i, i + BATCH_SIZE);
      await Promise.allSettled(
        batch.map(userId => this.getProfile(userId))
      );
      await new Promise(r => setTimeout(r, 100)); // Rate limit
    }
    
    logger.info(`Warmed up cache for ${uncachedIds.length} users`);
  }

  // Invalidate เมื่อ profile เปลี่ยน
  async invalidateProfile(lineUserId) {
    await cache.delete(`profile:${lineUserId}`);
  }
}

module.exports = new UserProfileCache();
```

### 4.3 Message Template Cache

```javascript
// optimization/src/templateCache.js
const MultiLevelCache = require('./cacheManager');
const Redis = require('ioredis');
const Handlebars = require('handlebars');

const redis = new Redis({ host: process.env.REDIS_HOST });
const cache = new MultiLevelCache(redis);

class TemplateCache {
  constructor() {
    this.compiledTemplates = new Map(); // In-memory compiled templates
  }

  // ดึงและ compile template
  async getTemplate(templateId) {
    // ลอง compiled cache ก่อน
    if (this.compiledTemplates.has(templateId)) {
      return this.compiledTemplates.get(templateId);
    }

    // ดึงจาก Redis/Database
    const template = await cache.getOrFetch(
      `template:${templateId}`,
      () => this.fetchFromDB(templateId),
      3600 // 1 hour
    );

    if (!template) return null;

    // Compile และ cache
    const compiled = Handlebars.compile(template.content);
    this.compiledTemplates.set(templateId, compiled);
    
    return compiled;
  }

  // Render template
  async render(templateId, data) {
    const compiledTemplate = await this.getTemplate(templateId);
    if (!compiledTemplate) {
      throw new Error(`Template not found: ${templateId}`);
    }

    return compiledTemplate(data);
  }

  // Invalidate เมื่อ template เปลี่ยน
  async invalidateTemplate(templateId) {
    this.compiledTemplates.delete(templateId);
    await cache.delete(`template:${templateId}`);
  }

  async fetchFromDB(templateId) {
    const { Template } = require('../models');
    return Template.findByPk(templateId, { raw: true });
  }
}

// Rich Menu Cache
class RichMenuCache {
  constructor() {
    this.cache = new MultiLevelCache(redis);
  }

  async getRichMenu(botId, userId) {
    // ลอง user-specific menu
    const userMenuKey = `richmenu:user:${botId}:${userId}`;
    const userMenu = await this.cache.get(userMenuKey);
    if (userMenu) return userMenu;

    // Default menu
    const defaultMenuKey = `richmenu:default:${botId}`;
    return this.cache.getOrFetch(
      defaultMenuKey,
      () => this.fetchDefaultMenu(botId),
      3600
    );
  }

  async fetchDefaultMenu(botId) {
    const axios = require('axios');
    try {
      const response = await axios.get(
        `https://api.line.me/v2/bot/richmenu/list`,
        { headers: { Authorization: `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}` } }
      );
      return response.data?.richmenus?.[0] || null;
    } catch (e) {
      return null;
    }
  }
}

module.exports = { TemplateCache: new TemplateCache(), RichMenuCache: new RichMenuCache() };
```

---

## 5. Rate Limit Management

### 5.1 LINE API Rate Limiting

```javascript
// optimization/src/rateLimitManager.js
const Bottleneck = require('bottleneck');
const Redis = require('ioredis');
const logger = require('../utils/logger');

class LineApiRateLimiter {
  constructor() {
    // LINE API limits:
    // - Push: 1,000 requests/second per channel
    // - Multicast: 500 requests/second
    // - Broadcast: 1 request/second
    
    this.pushLimiter = new Bottleneck({
      maxConcurrent: 50,
      minTime: 1,         // 1ms between requests (1000/s)
      highWater: 5000,    // Queue สูงสุด 5000
      strategy: Bottleneck.strategy.OVERFLOW_PRIORITY
    });

    this.multicastLimiter = new Bottleneck({
      maxConcurrent: 20,
      minTime: 2,         // 2ms between requests (500/s)
    });

    this.broadcastLimiter = new Bottleneck({
      maxConcurrent: 1,
      minTime: 1000,      // 1000ms between requests (1/s)
    });

    this.setupMonitoring();
  }

  async pushMessage(userId, messages, lineClient) {
    return this.pushLimiter.schedule(async () => {
      return lineClient.pushMessage(userId, messages);
    });
  }

  async multicastMessage(userIds, messages, lineClient) {
    return this.multicastLimiter.schedule(async () => {
      return lineClient.multicast(userIds, messages);
    });
  }

  async broadcastMessage(messages, lineClient) {
    return this.broadcastLimiter.schedule(async () => {
      return lineClient.broadcast(messages);
    });
  }

  setupMonitoring() {
    this.pushLimiter.on('queued', ({ options }) => {
      logger.debug('Push message queued in rate limiter');
    });

    this.pushLimiter.on('dropped', ({ options }) => {
      logger.warn('Push message DROPPED by rate limiter - queue full!');
    });

    // Monitor queue size
    setInterval(() => {
      const counts = this.pushLimiter.counts();
      if (counts.QUEUED > 1000) {
        logger.warn(`Rate limiter queue high: ${counts.QUEUED} messages queued`);
      }
    }, 5000);
  }
}

// Message batching ลด API calls
class MessageBatcher {
  constructor(lineApiRateLimiter) {
    this.limiter = lineApiRateLimiter;
    this.pendingMessages = new Map();
    this.batchInterval = 50; // 50ms batch window
    this.maxBatchSize = 5;   // LINE allows up to 5 messages per call
    
    this.startBatchProcessor();
  }

  addMessage(userId, message) {
    if (!this.pendingMessages.has(userId)) {
      this.pendingMessages.set(userId, []);
    }
    this.pendingMessages.get(userId).push(message);
  }

  startBatchProcessor() {
    setInterval(async () => {
      if (this.pendingMessages.size === 0) return;

      const batch = new Map(this.pendingMessages);
      this.pendingMessages.clear();

      const promises = [];
      for (const [userId, messages] of batch) {
        // Group messages into chunks of 5
        for (let i = 0; i < messages.length; i += this.maxBatchSize) {
          const chunk = messages.slice(i, i + this.maxBatchSize);
          promises.push(
            this.limiter.pushMessage(userId, chunk, lineClient)
              .catch(err => logger.error(`Batch send failed for ${userId}:`, err))
          );
        }
      }

      await Promise.allSettled(promises);
    }, this.batchInterval);
  }
}

module.exports = { LineApiRateLimiter, MessageBatcher };
```

---

## 6. Horizontal Scaling

### 6.1 Stateless Service Design

```javascript
// scaling/src/statelessHandler.js
// ✅ Stateless - ไม่เก็บ state ใน memory
class StatelessBotHandler {
  constructor() {
    this.redis = new Redis({ host: process.env.REDIS_HOST });
    // ไม่มี instance variables ที่ mutable
  }

  async handleMessage(event) {
    const userId = event.source.userId;
    
    // ดึง state จาก Redis แทน memory
    const userState = await this.getUserState(userId);
    
    // Process
    const reply = await this.processMessage(event.message, userState);
    
    // Save state กลับไปที่ Redis
    if (reply.newState) {
      await this.saveUserState(userId, reply.newState);
    }

    return reply.messages;
  }

  async getUserState(userId) {
    const key = `user:state:${userId}`;
    const state = await this.redis.get(key);
    return state ? JSON.parse(state) : { step: 'initial' };
  }

  async saveUserState(userId, state) {
    const key = `user:state:${userId}`;
    await this.redis.setex(key, 3600, JSON.stringify(state)); // 1 hour TTL
  }
}
```

### 6.2 Load Balancing Configuration

```nginx
# nginx/conf.d/load-balancer.conf

# Upstream servers
upstream webhook_receivers {
    least_conn;
    
    server webhook-1:3001 weight=1 max_fails=3 fail_timeout=30s;
    server webhook-2:3001 weight=1 max_fails=3 fail_timeout=30s;
    server webhook-3:3001 weight=1 max_fails=3 fail_timeout=30s;
    
    keepalive 100;
    keepalive_requests 1000;
    keepalive_timeout 60s;
}

upstream message_handlers {
    least_conn;
    
    server handler-1:3004;
    server handler-2:3004;
    server handler-3:3004;
    server handler-4:3004;
    server handler-5:3004;
}

# Main server block
server {
    listen 443 ssl http2;
    server_name api.example.com;

    # HTTP/2 for better multiplexing
    http2_push_preload on;

    # SSL optimization
    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;
    ssl_protocols TLSv1.2 TLSv1.3;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;

    location /webhook {
        proxy_pass http://webhook_receivers;
        
        # Connection settings
        proxy_http_version 1.1;
        proxy_set_header Connection "";  # Enable keepalive
        
        # Timeouts
        proxy_connect_timeout 2s;
        proxy_send_timeout 5s;
        proxy_read_timeout 5s;
        
        # Buffer optimization
        proxy_buffering off;
        proxy_buffer_size 4k;
        
        # Headers
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Rate limiting
        limit_req zone=webhook burst=500 nodelay;
    }
}
```

---

## 7. CDN สำหรับ Media

### 7.1 CloudFront / CDN Setup

```javascript
// optimization/src/cdnManager.js
const { S3Client, PutObjectCommand, GetObjectCommand } = require('@aws-sdk/client-s3');
const { CloudFrontClient, CreateInvalidationCommand } = require('@aws-sdk/client-cloudfront');
const { getSignedUrl } = require('@aws-sdk/s3-request-presigner');
const sharp = require('sharp');
const logger = require('../utils/logger');

class CDNManager {
  constructor() {
    this.s3 = new S3Client({ region: process.env.AWS_REGION });
    this.cloudfront = new CloudFrontClient({ region: process.env.AWS_REGION });
    
    this.bucket = process.env.S3_BUCKET;
    this.cdnDomain = process.env.CDN_DOMAIN;
    this.distributionId = process.env.CLOUDFRONT_DISTRIBUTION_ID;
  }

  // Upload image และ optimize
  async uploadImage(buffer, filename, options = {}) {
    const {
      quality = 80,
      format = 'webp',
      maxWidth = 1024,
      prefix = 'images'
    } = options;

    // Optimize image ด้วย Sharp
    const optimized = await sharp(buffer)
      .resize(maxWidth, null, { 
        withoutEnlargement: true,
        fit: 'inside'
      })
      .toFormat(format, { quality })
      .toBuffer();

    const key = `${prefix}/${Date.now()}-${filename}.${format}`;
    
    await this.s3.send(new PutObjectCommand({
      Bucket: this.bucket,
      Key: key,
      Body: optimized,
      ContentType: `image/${format}`,
      CacheControl: 'public, max-age=31536000, immutable', // 1 year
      Metadata: {
        'original-filename': filename,
        'optimized': 'true'
      }
    }));

    const cdnUrl = `https://${this.cdnDomain}/${key}`;
    
    logger.info(`Uploaded image: ${cdnUrl}`, {
      originalSize: buffer.length,
      optimizedSize: optimized.length,
      reduction: `${Math.round((1 - optimized.length / buffer.length) * 100)}%`
    });

    return cdnUrl;
  }

  // Generate CDN URL
  getCdnUrl(s3Key) {
    return `https://${this.cdnDomain}/${s3Key}`;
  }

  // Presigned URL สำหรับ private files
  async getPresignedUrl(s3Key, expiresIn = 3600) {
    const command = new GetObjectCommand({
      Bucket: this.bucket,
      Key: s3Key
    });

    return getSignedUrl(this.s3, command, { expiresIn });
  }

  // Invalidate CDN cache
  async invalidateCache(paths) {
    await this.cloudfront.send(new CreateInvalidationCommand({
      DistributionId: this.distributionId,
      InvalidationBatch: {
        CallerReference: Date.now().toString(),
        Paths: {
          Quantity: paths.length,
          Items: paths
        }
      }
    }));

    logger.info(`CDN cache invalidated for ${paths.length} paths`);
  }
}

module.exports = new CDNManager();
```

---

## 8. Performance Monitoring

### 8.1 APM Integration

```javascript
// monitoring/src/apm.js
const promClient = require('prom-client');
const os = require('os');

// Custom metrics
const responseTime = new promClient.Histogram({
  name: 'linebot_response_time_ms',
  help: 'Response time in milliseconds',
  labelNames: ['endpoint', 'method', 'status'],
  buckets: [10, 25, 50, 100, 250, 500, 1000, 2500, 5000]
});

const activeConnections = new promClient.Gauge({
  name: 'linebot_active_connections',
  help: 'Number of active connections'
});

const messageQueueDepth = new promClient.Gauge({
  name: 'linebot_queue_depth',
  help: 'Current message queue depth',
  labelNames: ['queue_name']
});

const lineApiCallDuration = new promClient.Histogram({
  name: 'linebot_line_api_duration_ms',
  help: 'LINE API call duration',
  labelNames: ['method', 'endpoint', 'status'],
  buckets: [50, 100, 200, 500, 1000, 2000, 5000]
});

const cacheHitRate = new promClient.Counter({
  name: 'linebot_cache_operations_total',
  help: 'Cache operations',
  labelNames: ['operation', 'result']  // operation: get/set, result: hit/miss
});

const errorCount = new promClient.Counter({
  name: 'linebot_errors_total',
  help: 'Total errors',
  labelNames: ['service', 'type']
});

// System metrics
const cpuUsage = new promClient.Gauge({
  name: 'linebot_cpu_usage_percent',
  help: 'CPU usage percentage'
});

const memoryUsage = new promClient.Gauge({
  name: 'linebot_memory_usage_bytes',
  help: 'Memory usage in bytes',
  labelNames: ['type']
});

// Collect system metrics
setInterval(() => {
  // CPU
  const cpus = os.cpus();
  const avgCpu = cpus.reduce((acc, cpu) => {
    const total = Object.values(cpu.times).reduce((a, b) => a + b, 0);
    const idle = cpu.times.idle;
    return acc + ((total - idle) / total * 100);
  }, 0) / cpus.length;
  cpuUsage.set(avgCpu);

  // Memory
  const used = process.memoryUsage();
  memoryUsage.labels('heapUsed').set(used.heapUsed);
  memoryUsage.labels('heapTotal').set(used.heapTotal);
  memoryUsage.labels('rss').set(used.rss);
  memoryUsage.labels('external').set(used.external);
}, 15000);

// Middleware
function metricsMiddleware(req, res, next) {
  const start = Date.now();
  activeConnections.inc();

  res.on('finish', () => {
    const duration = Date.now() - start;
    activeConnections.dec();
    
    responseTime
      .labels(req.route?.path || req.path, req.method, res.statusCode)
      .observe(duration);
  });

  next();
}

// Metrics endpoint
async function metricsHandler(req, res) {
  res.set('Content-Type', promClient.register.contentType);
  const metrics = await promClient.register.metrics();
  res.end(metrics);
}

module.exports = {
  responseTime,
  lineApiCallDuration,
  cacheHitRate,
  errorCount,
  messageQueueDepth,
  metricsMiddleware,
  metricsHandler
};
```

---

## 9. Load Testing with k6

### 9.1 k6 Test Scripts

```javascript
// load-testing/webhook-load-test.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend, Counter } from 'k6/metrics';
import crypto from 'k6/crypto';

// Custom metrics
const errorRate = new Rate('errors');
const webhookResponseTime = new Trend('webhook_response_time');
const successfulWebhooks = new Counter('successful_webhooks');

// Test configuration
export const options = {
  stages: [
    { duration: '30s', target: 10 },   // Ramp up
    { duration: '1m', target: 100 },   // Load
    { duration: '1m', target: 500 },   // Stress
    { duration: '1m', target: 1000 },  // Spike
    { duration: '30s', target: 0 },    // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'], // 95% < 500ms, 99% < 1s
    errors: ['rate<0.01'],                           // Error rate < 1%
    http_req_failed: ['rate<0.01'],
  },
};

const WEBHOOK_URL = __ENV.WEBHOOK_URL || 'http://localhost:3001/webhook';
const CHANNEL_SECRET = __ENV.CHANNEL_SECRET || 'test-secret';

function generateTextEvent() {
  const userId = `U${Math.random().toString(36).substring(2, 12)}`;
  const timestamp = Date.now();
  
  return JSON.stringify({
    destination: 'U1234567890',
    events: [{
      type: 'message',
      replyToken: `rt${Math.random().toString(36).substring(2)}`,
      source: {
        userId,
        type: 'user'
      },
      timestamp,
      message: {
        id: `m${timestamp}`,
        type: 'text',
        text: 'สวัสดี'
      }
    }]
  });
}

function generateSignature(body, secret) {
  const hash = crypto.hmac('sha256', secret, body, 'base64');
  return hash;
}

export default function () {
  const body = generateTextEvent();
  const signature = generateSignature(body, CHANNEL_SECRET);
  
  const start = Date.now();
  
  const response = http.post(WEBHOOK_URL, body, {
    headers: {
      'Content-Type': 'application/json',
      'X-Line-Signature': signature,
    },
    timeout: '2s',
  });

  const duration = Date.now() - start;
  webhookResponseTime.add(duration);

  const success = check(response, {
    'status is 200': (r) => r.status === 200,
    'response time < 500ms': (r) => r.timings.duration < 500,
    'response is ok': (r) => r.json('status') === 'ok',
  });

  if (!success) {
    errorRate.add(1);
  } else {
    successfulWebhooks.add(1);
  }

  sleep(0.1); // 10 requests/second per VU
}

export function handleSummary(data) {
  return {
    'load-test-results.json': JSON.stringify(data),
    stdout: generateTextSummary(data),
  };
}

function generateTextSummary(data) {
  const metrics = data.metrics;
  return `
=== LINE Bot Webhook Load Test Results ===
Duration: ${data.state.testRunDurationMs / 1000}s
VUs Max: ${data.state.maxVUs}

Response Times:
  P50: ${metrics.http_req_duration.values['p(50)'].toFixed(2)}ms
  P95: ${metrics.http_req_duration.values['p(95)'].toFixed(2)}ms  
  P99: ${metrics.http_req_duration.values['p(99)'].toFixed(2)}ms
  Max: ${metrics.http_req_duration.values.max.toFixed(2)}ms

Throughput:
  Requests/s: ${metrics.http_reqs.values.rate.toFixed(2)}
  Total: ${metrics.http_reqs.values.count}

Errors:
  Rate: ${(metrics.errors.values.rate * 100).toFixed(2)}%
  Total: ${metrics.errors.values.count}

Status: ${metrics.http_req_failed.values.rate < 0.01 ? '✅ PASS' : '❌ FAIL'}
`;
}
```

### 9.2 Run Load Tests

```bash
#!/bin/bash
# scripts/run-load-test.sh

WEBHOOK_URL=${WEBHOOK_URL:-"http://localhost:3001/webhook"}
CHANNEL_SECRET=${CHANNEL_SECRET:-"test-secret"}
OUTPUT_DIR="./load-test-results/$(date +%Y%m%d-%H%M%S)"

mkdir -p "$OUTPUT_DIR"

echo "Starting load test..."
echo "Target: $WEBHOOK_URL"

# Run k6 test
k6 run \
  --env WEBHOOK_URL="$WEBHOOK_URL" \
  --env CHANNEL_SECRET="$CHANNEL_SECRET" \
  --out json="$OUTPUT_DIR/results.json" \
  --out influxdb="http://localhost:8086/k6" \
  load-testing/webhook-load-test.js \
  2>&1 | tee "$OUTPUT_DIR/output.log"

echo "Load test completed. Results saved to $OUTPUT_DIR"

# Generate HTML report
k6 report "$OUTPUT_DIR/results.json" --output "$OUTPUT_DIR/report.html"
echo "HTML report: $OUTPUT_DIR/report.html"
```

---

## 10. Optimization Checklist

### 10.1 Performance Tuning Checklist

```
Webhook Receiver:
  ✅ ตอบ 200 OK ทันทีก่อน process
  ✅ Validate signature เร็ว (< 2ms)
  ✅ Async publish to queue
  ✅ Connection pooling สำหรับ Redis

Database:
  ✅ ทุก foreign key มี index
  ✅ ใช้ cursor pagination แทน OFFSET
  ✅ Eager loading ป้องกัน N+1
  ✅ Query profiling เปิดอยู่ใน development
  ✅ Connection pool max = 20

Caching:
  ✅ User profile: cache 1 hour
  ✅ Message templates: cache 1 hour
  ✅ Rich menu: cache 1 hour
  ✅ Feature flags: cache 1 minute
  ✅ Cache invalidation strategy ชัดเจน

LINE API:
  ✅ Rate limiter ป้องกัน 429
  ✅ Retry with exponential backoff
  ✅ HTTP/2 + keepalive
  ✅ Batch messages (max 5 per call)

Infrastructure:
  ✅ Load balancer + health checks
  ✅ Auto-scaling configured
  ✅ CDN สำหรับ static assets
  ✅ Gzip/Brotli compression

Monitoring:
  ✅ Response time alerts (> 500ms)
  ✅ Error rate alerts (> 1%)
  ✅ Queue depth alerts (> 10,000)
  ✅ Database slow query logging
```

### 10.2 Performance Targets

```
Target Response Times (p95):
  Webhook receiver: < 50ms
  Message handler: < 200ms
  Database queries: < 20ms
  Redis operations: < 5ms
  LINE API calls: < 500ms
  Total end-to-end: < 1,000ms

Throughput Targets:
  Small bot (< 1K users): 
    - 10 messages/second
    - 1 instance

  Medium bot (< 100K users):
    - 100 messages/second
    - 3-5 instances

  Large bot (> 100K users):
    - 1,000+ messages/second
    - 10+ instances + auto-scaling
    - Redis Cluster
    - PostgreSQL read replicas

Resource Usage:
  CPU: < 70% at peak
  Memory: < 80% of available
  Connections: < 80% of pool size
  Queue depth: < 10,000 normally
```

---

## 11. Production Setup

### 11.1 Complete docker-compose.prod.yml

```yaml
# docker-compose.prod.yml
version: '3.9'

x-common: &common
  restart: unless-stopped
  logging:
    driver: "json-file"
    options:
      max-size: "10m"
      max-file: "5"

services:
  # ===========================
  # Application (Optimized)
  # ===========================
  
  webhook-receiver:
    <<: *common
    build:
      context: ./webhook-receiver
      args:
        NODE_ENV: production
    environment:
      NODE_ENV: production
      PORT: 3001
      REDIS_HOST: redis
      REDIS_MAX_RETRIES: 3
      UV_THREADPOOL_SIZE: 64
      NODE_OPTIONS: "--max-old-space-size=512"
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    healthcheck:
      test: ["CMD", "node", "-e", "require('http').get('http://localhost:3001/health', r => process.exit(r.statusCode === 200 ? 0 : 1))"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 30s
    networks:
      - bot-network

  # PgBouncer สำหรับ connection pooling
  pgbouncer:
    <<: *common
    image: pgbouncer/pgbouncer:latest
    environment:
      DATABASES_HOST: postgres
      DATABASES_PORT: 5432
      DATABASES_DBNAME: linebot
      DATABASES_USER: linebot
      DATABASES_PASSWORD: ${DB_PASSWORD}
      POOL_MODE: transaction
      MAX_CLIENT_CONN: 500
      DEFAULT_POOL_SIZE: 20
      MIN_POOL_SIZE: 5
      SERVER_IDLE_TIMEOUT: 600
    ports:
      - "6432:5432"
    networks:
      - bot-network

  # Redis Sentinel สำหรับ HA
  redis-sentinel:
    <<: *common
    image: redis:7.2-alpine
    command: redis-sentinel /etc/redis/sentinel.conf
    volumes:
      - ./redis/sentinel.conf:/etc/redis/sentinel.conf
    networks:
      - bot-network

  # Nginx สำหรับ reverse proxy
  nginx:
    <<: *common
    image: nginx:1.25-alpine
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - nginx_cache:/var/cache/nginx
      - certbot_www:/var/www/certbot:ro
      - certbot_certs:/etc/letsencrypt:ro
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - webhook-receiver
    networks:
      - bot-network
    sysctls:
      net.core.somaxconn: 65535

volumes:
  nginx_cache:
  certbot_www:
  certbot_certs:

networks:
  bot-network:
    driver: overlay
    driver_opts:
      encrypted: "true"
```

---

## 12. สรุปและ Optimization Journey

### 12.1 Performance Evolution

```
Version 1.0 (Monolith, No optimization):
  - Response time: 2-5 seconds
  - Throughput: 10 req/s
  - Users: < 1,000

Version 2.0 (Async + Queue):
  - Response time: 50-100ms
  - Throughput: 100 req/s
  - Users: < 10,000

Version 3.0 (Caching + DB Optimization):
  - Response time: 20-50ms
  - Throughput: 500 req/s
  - Users: < 100,000

Version 4.0 (Microservices + Scaling):
  - Response time: 10-30ms
  - Throughput: 5,000+ req/s
  - Users: 1,000,000+
```

### 12.2 Key Takeaways

```
1. ตอบ LINE Platform ก่อนเสมอ (async pattern)
   → เป้าหมาย: < 100ms ตอบ webhook

2. Cache คือกุญแจหลัก
   → User profile, templates, rich menu: cache ทั้งหมด
   → Cache hit rate เป้าหมาย: > 90%

3. Database optimization มีผลมาก
   → Index ที่ถูกต้อง: ลด query time 10-100x
   → ป้องกัน N+1: ลด queries 90%

4. Rate limiting ป้องกัน LINE API errors
   → Bottleneck library จัดการ automatically

5. Monitor ทุกอย่าง
   → ถ้าไม่ measure ก็ไม่รู้ว่า optimize ได้แค่ไหน

6. Load test ก่อน production
   → k6 test ให้รู้ bottleneck จริงๆ
```

ยินดีด้วย! คุณจบหมวด Advanced Architecture แล้ว ในหมวดถัดไปจะพูดถึงการ deploy ในระดับ enterprise และการจัดการ LINE Bot ในองค์กรขนาดใหญ่
