# Part 49: Redis Session Management สำหรับ LINE Bot

## บทนำ

Redis เป็น in-memory data store ที่เร็วมาก เหมาะอย่างยิ่งสำหรับการจัดการ session ของ LINE bot เพราะ:
- **เร็วมาก**: อ่าน/เขียนข้อมูลใน microseconds
- **รองรับ TTL**: กำหนดอายุข้อมูลได้อัตโนมัติ
- **Data Structures หลากหลาย**: String, Hash, List, Set, Sorted Set
- **Pub/Sub**: รองรับ real-time messaging
- **Persistence**: เก็บข้อมูลลง disk ได้

---

## 1. ติดตั้งและตั้งค่า Redis

### 1.1 ติดตั้ง Redis บน Linux/Mac

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install redis-server

# MacOS (Homebrew)
brew install redis

# เริ่ม service
sudo systemctl start redis-server
sudo systemctl enable redis-server

# ตรวจสอบ
redis-cli ping  # ควรตอบ PONG

# ตรวจสอบ version
redis-cli --version
```

### 1.2 Redis Config สำหรับ Production

```bash
# /etc/redis/redis.conf

# Network
bind 127.0.0.1 -::1          # เฉพาะ localhost (ปลอดภัยกว่า)
port 6379
timeout 300                    # disconnect client ที่ idle 5 นาที

# Security
requirepass your-strong-password-here

# Memory
maxmemory 256mb
maxmemory-policy allkeys-lru  # ลบ keys เก่าเมื่อ memory เต็ม

# Persistence
save 900 1                     # บันทึกถ้ามี 1 change ใน 900 วินาที
save 300 10                    # บันทึกถ้ามี 10 changes ใน 300 วินาที
save 60 10000                  # บันทึกถ้ามี 10000 changes ใน 60 วินาที

appendonly yes                 # AOF persistence (ปลอดภัยกว่า)
appendfsync everysec

# Logging
loglevel notice
logfile /var/log/redis/redis.log

# Keyspace Notifications (สำหรับ TTL events)
notify-keyspace-events Ex
```

### 1.3 Docker Redis

```yaml
# docker-compose.yml
version: '3.8'

services:
  redis:
    image: redis:7-alpine
    container_name: linebot_redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
      - ./redis.conf:/usr/local/etc/redis/redis.conf
    command: redis-server /usr/local/etc/redis/redis.conf
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    environment:
      - REDIS_PASSWORD=${REDIS_PASSWORD}

volumes:
  redis_data:
    driver: local
```

---

## 2. ติดตั้ง Redis Client

### 2.1 Node.js (ioredis)

```bash
npm install ioredis
npm install -D @types/ioredis
```

### 2.2 Python (redis-py)

```bash
pip install redis
pip install hiredis  # parser เร็วกว่า (optional)
```

---

## 3. Redis Data Structures สำหรับ LINE Bot

### 3.1 String - เก็บข้อมูลง่ายๆ

```javascript
// Node.js - String operations
const Redis = require('ioredis');

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: 6379,
  password: process.env.REDIS_PASSWORD,
  keyPrefix: 'linebot:',   // prefix ทุก key
  retryStrategy: (times) => Math.min(times * 100, 3000),
});

// =============================================
// STRING - ใช้สำหรับเก็บ state อย่างง่าย
// =============================================

// เก็บ conversation state
async function setUserState(userId, state, ttlSeconds = 1800) {
  const key = `state:${userId}`;
  await redis.setex(key, ttlSeconds, JSON.stringify(state));
}

async function getUserState(userId) {
  const key = `state:${userId}`;
  const data = await redis.get(key);
  return data ? JSON.parse(data) : null;
}

async function deleteUserState(userId) {
  await redis.del(`state:${userId}`);
}

// เก็บ rate limit counter
async function checkRateLimit(userId, maxRequests = 5, windowSeconds = 60) {
  const key = `ratelimit:${userId}`;
  
  const multi = redis.multi();
  multi.incr(key);
  multi.ttl(key);
  
  const results = await multi.exec();
  const count = results[0][1];
  const ttl = results[1][1];
  
  // ถ้า key ใหม่ (ttl = -1) ให้ set expire
  if (ttl === -1) {
    await redis.expire(key, windowSeconds);
  }
  
  return {
    allowed: count <= maxRequests,
    count,
    remaining: Math.max(0, maxRequests - count),
    resetIn: ttl > 0 ? ttl : windowSeconds,
  };
}

// ตัวอย่างใช้งาน
async function processMessage(userId, message) {
  // ตรวจสอบ rate limit
  const rateCheck = await checkRateLimit(userId);
  if (!rateCheck.allowed) {
    return `กรุณารอ ${rateCheck.resetIn} วินาที (ส่งข้อความเร็วเกินไป)`;
  }
  
  // ดึง state
  const state = await getUserState(userId);
  console.log(`User ${userId} state:`, state);
  
  // ประมวลผล...
}
```

### 3.2 Hash - เก็บ User Session

```javascript
// =============================================
// HASH - เก็บ session data หลายๆ field
// =============================================

// เก็บ user session ที่มีหลาย fields
async function setUserSession(userId, sessionData) {
  const key = `session:${userId}`;
  
  // แปลง nested objects เป็น string
  const flatData = {};
  for (const [k, v] of Object.entries(sessionData)) {
    flatData[k] = typeof v === 'object' ? JSON.stringify(v) : String(v);
  }
  
  await redis.hset(key, flatData);
  await redis.expire(key, 3600); // 1 ชั่วโมง
}

async function getUserSession(userId) {
  const key = `session:${userId}`;
  const data = await redis.hgetall(key);
  
  if (!data || Object.keys(data).length === 0) return null;
  
  // แปลง JSON strings กลับ
  const parsed = {};
  for (const [k, v] of Object.entries(data)) {
    try {
      parsed[k] = JSON.parse(v);
    } catch {
      parsed[k] = v;
    }
  }
  return parsed;
}

async function updateSessionField(userId, field, value) {
  const key = `session:${userId}`;
  const serialized = typeof value === 'object' ? JSON.stringify(value) : String(value);
  await redis.hset(key, field, serialized);
  await redis.expire(key, 3600); // รีเซ็ต TTL
}

async function getSessionField(userId, field) {
  const key = `session:${userId}`;
  const value = await redis.hget(key, field);
  if (!value) return null;
  try {
    return JSON.parse(value);
  } catch {
    return value;
  }
}

async function deleteSessionField(userId, ...fields) {
  const key = `session:${userId}`;
  await redis.hdel(key, ...fields);
}

// ตัวอย่างการใช้งาน
async function exampleHashUsage() {
  const userId = 'Uabc123';
  
  // เก็บ session
  await setUserSession(userId, {
    displayName: 'สมชาย',
    currentFlow: 'order_flow',
    currentStep: 'select_product',
    cartItems: [{ id: 'P001', qty: 2 }],
    loginAt: new Date().toISOString(),
  });
  
  // ดึงข้อมูลทั้งหมด
  const session = await getUserSession(userId);
  console.log(session);
  
  // อัปเดต field เดียว
  await updateSessionField(userId, 'currentStep', 'select_size');
  
  // ดึง field เดียว
  const step = await getSessionField(userId, 'currentStep');
  console.log('Current step:', step); // 'select_size'
  
  // ลบ field
  await deleteSessionField(userId, 'cartItems');
}
```

### 3.3 List - เก็บ Conversation History

```javascript
// =============================================
// LIST - เก็บ conversation history
// =============================================

const HISTORY_KEY = (userId) => `history:${userId}`;
const MAX_HISTORY = 20;

// เพิ่มข้อความลงใน history
async function addToHistory(userId, message) {
  const key = HISTORY_KEY(userId);
  const entry = JSON.stringify({
    ...message,
    timestamp: Date.now(),
  });
  
  // RPUSH เพิ่มที่ท้าย list
  await redis.rpush(key, entry);
  
  // ตัดให้เหลือแค่ MAX_HISTORY รายการ (เก็บล่าสุด)
  await redis.ltrim(key, -MAX_HISTORY, -1);
  
  // รีเซ็ต TTL ทุกครั้งที่มีการเพิ่ม
  await redis.expire(key, 24 * 3600); // 24 ชั่วโมง
}

// ดึงประวัติทั้งหมด
async function getHistory(userId, limit = MAX_HISTORY) {
  const key = HISTORY_KEY(userId);
  // LRANGE ดึงจากท้าย
  const items = await redis.lrange(key, -limit, -1);
  return items.map(item => JSON.parse(item));
}

// ดึงข้อความล่าสุด
async function getLastMessage(userId) {
  const key = HISTORY_KEY(userId);
  const item = await redis.lindex(key, -1);
  return item ? JSON.parse(item) : null;
}

// ล้าง history
async function clearHistory(userId) {
  await redis.del(HISTORY_KEY(userId));
}

// Queue สำหรับ outgoing messages
async function queueMessage(userId, message) {
  const key = `queue:${userId}`;
  await redis.rpush(key, JSON.stringify(message));
  await redis.expire(key, 300); // 5 นาที
}

async function dequeueMessage(userId) {
  const key = `queue:${userId}`;
  const item = await redis.lpop(key);
  return item ? JSON.parse(item) : null;
}

async function getAllQueuedMessages(userId) {
  const key = `queue:${userId}`;
  const items = await redis.lrange(key, 0, -1);
  await redis.del(key); // ล้าง queue หลังดึง
  return items.map(item => JSON.parse(item));
}

// ตัวอย่างการใช้งาน
async function exampleListUsage() {
  const userId = 'Uabc123';
  
  await addToHistory(userId, { role: 'user', text: 'สวัสดี' });
  await addToHistory(userId, { role: 'bot', text: 'สวัสดีครับ มีอะไรให้ช่วยไหม?' });
  await addToHistory(userId, { role: 'user', text: 'สั่งอาหาร' });
  
  const history = await getHistory(userId, 3);
  console.log('History:', history);
  
  const last = await getLastMessage(userId);
  console.log('Last message:', last);
}
```

### 3.4 Set - เก็บ User Groups

```javascript
// =============================================
// SET - เก็บ groups และ permissions
// =============================================

// จัดการ user groups
async function addUserToGroup(groupName, userId) {
  const key = `group:${groupName}`;
  await redis.sadd(key, userId);
}

async function removeUserFromGroup(groupName, userId) {
  const key = `group:${groupName}`;
  await redis.srem(key, userId);
}

async function isUserInGroup(groupName, userId) {
  const key = `group:${groupName}`;
  return await redis.sismember(key, userId) === 1;
}

async function getGroupMembers(groupName) {
  const key = `group:${groupName}`;
  return await redis.smembers(key);
}

async function getGroupCount(groupName) {
  const key = `group:${groupName}`;
  return await redis.scard(key);
}

// เช็ค permissions
async function getUserPermissions(userId) {
  const key = `permissions:${userId}`;
  return await redis.smembers(key);
}

async function hasPermission(userId, permission) {
  const key = `permissions:${userId}`;
  return await redis.sismember(key, permission) === 1;
}

async function grantPermission(userId, permission) {
  const key = `permissions:${userId}`;
  await redis.sadd(key, permission);
}

async function revokePermission(userId, permission) {
  const key = `permissions:${userId}`;
  await redis.srem(key, permission);
}

// เช็ค online users (Pub/Sub)
async function setUserOnline(userId) {
  await redis.sadd('online_users', userId);
  await redis.setex(`online:${userId}`, 300, '1'); // หมดอายุ 5 นาที
}

async function setUserOffline(userId) {
  await redis.srem('online_users', userId);
  await redis.del(`online:${userId}`);
}

async function isUserOnline(userId) {
  return await redis.exists(`online:${userId}`) === 1;
}

async function getOnlineCount() {
  return await redis.scard('online_users');
}

// ตัวอย่าง: เก็บ users ที่ subscribe newsletter
async function subscribeNewsletter(userId) {
  await redis.sadd('newsletter:subscribers', userId);
}

async function unsubscribeNewsletter(userId) {
  await redis.srem('newsletter:subscribers', userId);
}

async function getNewsletterSubscribers() {
  return await redis.smembers('newsletter:subscribers');
}
```

### 3.5 Sorted Set - Leaderboard

```javascript
// =============================================
// SORTED SET - Leaderboard, Rankings
// =============================================

// Points Leaderboard
async function addPoints(userId, points, displayName) {
  // เก็บคะแนนใน sorted set
  await redis.zincrby('leaderboard:points', points, userId);
  
  // เก็บ display name ไว้ใน hash
  await redis.hset('leaderboard:names', userId, displayName);
}

async function getUserRank(userId) {
  // rank จาก 0 (ZREVRANK = มากไปน้อย)
  const rank = await redis.zrevrank('leaderboard:points', userId);
  return rank !== null ? rank + 1 : null; // convert to 1-indexed
}

async function getUserPoints(userId) {
  const score = await redis.zscore('leaderboard:points', userId);
  return score ? parseFloat(score) : 0;
}

async function getTopUsers(limit = 10) {
  // ดึง top users พร้อมคะแนน
  const results = await redis.zrevrange('leaderboard:points', 0, limit - 1, 'WITHSCORES');
  
  const users = [];
  for (let i = 0; i < results.length; i += 2) {
    const userId = results[i];
    const score = parseFloat(results[i + 1]);
    const name = await redis.hget('leaderboard:names', userId);
    
    users.push({
      rank: users.length + 1,
      userId,
      displayName: name || 'ไม่ระบุ',
      points: score,
    });
  }
  
  return users;
}

async function getUserLeaderboardInfo(userId) {
  const [rank, points] = await Promise.all([
    getUserRank(userId),
    getUserPoints(userId),
  ]);
  
  return { rank, points };
}

// Time-based leaderboard (weekly)
function getWeeklyLeaderboardKey() {
  const now = new Date();
  const weekNumber = getWeekNumber(now);
  return `leaderboard:weekly:${now.getFullYear()}:${weekNumber}`;
}

async function addWeeklyPoints(userId, points) {
  const key = getWeeklyLeaderboardKey();
  await redis.zincrby(key, points, userId);
  
  // หมดอายุหลัง 2 สัปดาห์
  await redis.expire(key, 14 * 24 * 3600);
}

function getWeekNumber(date) {
  const startDate = new Date(date.getFullYear(), 0, 1);
  const days = Math.floor((date - startDate) / (24 * 60 * 60 * 1000));
  return Math.ceil((days + startDate.getDay() + 1) / 7);
}

// Activity Tracking ด้วย Sorted Set
async function trackActivity(userId) {
  const key = 'activity:daily';
  const score = Date.now();
  await redis.zadd(key, score, userId);
}

async function getActiveUsersLast24h() {
  const key = 'activity:daily';
  const since = Date.now() - 24 * 60 * 60 * 1000;
  return await redis.zcount(key, since, '+inf');
}
```

---

## 4. Pattern: User Waiting State

```javascript
// =============================================
// PATTERN: User Waiting State
// ใช้เมื่อ bot ต้องการ input เฉพาะจาก user
// =============================================

class WaitingStateManager {
  constructor(redis) {
    this.redis = redis;
    this.prefix = 'waiting:';
    this.defaultTTL = 300; // 5 นาที
  }

  /**
   * ตั้ง bot ให้รอ input จาก user
   * @param {string} userId
   * @param {string} waitingFor - อธิบายว่า bot รอ input อะไร
   * @param {object} context - ข้อมูลเพิ่มเติม
   * @param {number} ttl - timeout
   */
  async setWaiting(userId, waitingFor, context = {}, ttl = this.defaultTTL) {
    const key = `${this.prefix}${userId}`;
    const data = {
      waitingFor,
      context,
      startedAt: Date.now(),
    };
    await this.redis.setex(key, ttl, JSON.stringify(data));
  }

  /**
   * ตรวจสอบว่า bot กำลังรอ input อะไรจาก user
   */
  async getWaiting(userId) {
    const key = `${this.prefix}${userId}`;
    const data = await this.redis.get(key);
    return data ? JSON.parse(data) : null;
  }

  /**
   * ล้าง waiting state
   */
  async clearWaiting(userId) {
    const key = `${this.prefix}${userId}`;
    await this.redis.del(key);
  }

  /**
   * ประมวลผลข้อความเมื่อ bot กำลังรอ
   */
  async processWaitingInput(userId, input) {
    const waiting = await this.getWaiting(userId);
    if (!waiting) return null;

    // ล้าง waiting state
    await this.clearWaiting(userId);

    return {
      waitingFor: waiting.waitingFor,
      input,
      context: waiting.context,
    };
  }
}

// ตัวอย่าง: Bot รอ phone number
async function exampleWaitingPattern(userId, message, replyToken, lineClient) {
  const waitingManager = new WaitingStateManager(redis);
  
  // ตรวจสอบว่า bot กำลังรอ input อะไร
  const waiting = await waitingManager.getWaiting(userId);
  
  if (waiting) {
    switch (waiting.waitingFor) {
      case 'phone_number':
        // validate phone number
        const phoneRegex = /^0[0-9]{9}$/;
        if (!phoneRegex.test(message.replace(/[-\s]/g, ''))) {
          await lineClient.replyMessage(replyToken, [{
            type: 'text',
            text: 'หมายเลขโทรศัพท์ไม่ถูกต้อง กรุณากรอกอีกครั้ง (10 หลัก)',
          }]);
          return; // ยัง waiting อยู่
        }
        
        await waitingManager.clearWaiting(userId);
        // บันทึก phone number
        await updateSessionField(userId, 'phone', message);
        await lineClient.replyMessage(replyToken, [{
          type: 'text',
          text: `บันทึกเบอร์โทร ${message} แล้วครับ ✓`,
        }]);
        break;
        
      case 'confirmation_code':
        const savedCode = waiting.context.code;
        if (message !== savedCode) {
          await lineClient.replyMessage(replyToken, [{
            type: 'text',
            text: 'รหัสไม่ถูกต้อง กรุณาลองอีกครั้ง',
          }]);
          return;
        }
        
        await waitingManager.clearWaiting(userId);
        await lineClient.replyMessage(replyToken, [{
          type: 'text',
          text: 'ยืนยันสำเร็จ! ✓',
        }]);
        break;
    }
    return;
  }
  
  // Normal message processing
  if (message === 'ลงทะเบียน') {
    await waitingManager.setWaiting(userId, 'phone_number', {}, 300);
    await lineClient.replyMessage(replyToken, [{
      type: 'text',
      text: 'กรุณากรอกหมายเลขโทรศัพท์ของคุณ:',
    }]);
  }
}
```

---

## 5. Pattern: Multi-Step Order Flow

```javascript
// =============================================
// PATTERN: Multi-Step Order Flow with Redis
// =============================================

class OrderFlowManager {
  constructor(redis) {
    this.redis = redis;
    this.TTL = 30 * 60; // 30 นาที
  }

  getKey(userId) {
    return `order_flow:${userId}`;
  }

  async startFlow(userId) {
    const flowData = {
      step: 'select_product',
      selectedProduct: null,
      selectedSize: null,
      selectedAddons: [],
      quantity: 1,
      address: null,
      note: '',
      startedAt: Date.now(),
    };
    
    await this.redis.setex(
      this.getKey(userId),
      this.TTL,
      JSON.stringify(flowData)
    );
    
    return flowData;
  }

  async getFlow(userId) {
    const data = await this.redis.get(this.getKey(userId));
    return data ? JSON.parse(data) : null;
  }

  async updateFlow(userId, updates) {
    const current = await this.getFlow(userId);
    if (!current) throw new Error('No active flow');
    
    const updated = { ...current, ...updates };
    await this.redis.setex(
      this.getKey(userId),
      this.TTL,
      JSON.stringify(updated)
    );
    
    return updated;
  }

  async nextStep(userId, nextStep, data = {}) {
    return this.updateFlow(userId, { step: nextStep, ...data });
  }

  async endFlow(userId) {
    const flow = await this.getFlow(userId);
    await this.redis.del(this.getKey(userId));
    return flow;
  }

  async isActive(userId) {
    return await this.redis.exists(this.getKey(userId)) === 1;
  }

  // Handle user input ตาม step
  async handleInput(userId, input, replyToken, lineClient) {
    const flow = await this.getFlow(userId);
    
    if (!flow) {
      return false; // ไม่อยู่ใน flow
    }

    // Cancel keyword
    if (['ยกเลิก', 'cancel', 'หยุด'].includes(input.toLowerCase())) {
      await this.endFlow(userId);
      await lineClient.replyMessage(replyToken, [{
        type: 'text',
        text: '✓ ยกเลิกออเดอร์แล้วครับ',
      }]);
      return true;
    }

    switch (flow.step) {
      case 'select_product':
        await this._handleSelectProduct(userId, input, flow, replyToken, lineClient);
        break;
      case 'select_size':
        await this._handleSelectSize(userId, input, flow, replyToken, lineClient);
        break;
      case 'select_quantity':
        await this._handleSelectQuantity(userId, input, flow, replyToken, lineClient);
        break;
      case 'enter_address':
        await this._handleEnterAddress(userId, input, flow, replyToken, lineClient);
        break;
      case 'confirm':
        await this._handleConfirm(userId, input, flow, replyToken, lineClient);
        break;
    }
    
    return true;
  }

  async _handleSelectProduct(userId, input, flow, replyToken, lineClient) {
    const menu = {
      '1': { id: 'P001', name: 'พิซซ่า Margherita', price: 250 },
      '2': { id: 'P002', name: 'พิซซ่า Pepperoni', price: 280 },
      '3': { id: 'P003', name: 'พาสต้า Carbonara', price: 180 },
      'พิซซ่า margherita': { id: 'P001', name: 'พิซซ่า Margherita', price: 250 },
    };
    
    const product = menu[input] || menu[input.toLowerCase()];
    if (!product) {
      await lineClient.replyMessage(replyToken, [{
        type: 'text',
        text: 'ไม่พบสินค้า กรุณาพิมพ์หมายเลข 1, 2 หรือ 3',
      }]);
      return;
    }
    
    await this.nextStep(userId, 'select_size', { selectedProduct: product });
    await lineClient.replyMessage(replyToken, [{
      type: 'template',
      altText: 'เลือกขนาด',
      template: {
        type: 'buttons',
        text: `${product.name} - เลือกขนาด`,
        actions: [
          { type: 'message', label: 'S (฿' + (product.price - 50) + ')', text: 'S' },
          { type: 'message', label: 'M (฿' + product.price + ')', text: 'M' },
          { type: 'message', label: 'L (฿' + (product.price + 50) + ')', text: 'L' },
        ],
      },
    }]);
  }

  async _handleSelectSize(userId, input, flow, replyToken, lineClient) {
    const validSizes = ['S', 'M', 'L'];
    const size = input.toUpperCase();
    
    if (!validSizes.includes(size)) {
      await lineClient.replyMessage(replyToken, [{
        type: 'text',
        text: 'กรุณาเลือก S, M หรือ L',
      }]);
      return;
    }
    
    const adjustments = { S: -50, M: 0, L: 50 };
    const finalPrice = flow.selectedProduct.price + adjustments[size];
    
    await this.nextStep(userId, 'select_quantity', {
      selectedSize: size,
      unitPrice: finalPrice,
    });
    
    await lineClient.replyMessage(replyToken, [{
      type: 'text',
      text: `ขนาด ${size} ราคา ${finalPrice} บาท ✓\n\nกรุณากรอกจำนวน (1-10):`,
    }]);
  }

  async _handleSelectQuantity(userId, input, flow, replyToken, lineClient) {
    const qty = parseInt(input);
    if (isNaN(qty) || qty < 1 || qty > 10) {
      await lineClient.replyMessage(replyToken, [{
        type: 'text',
        text: 'กรุณากรอกจำนวน 1-10',
      }]);
      return;
    }
    
    const total = flow.unitPrice * qty;
    
    await this.nextStep(userId, 'enter_address', {
      quantity: qty,
      totalPrice: total,
    });
    
    await lineClient.replyMessage(replyToken, [{
      type: 'text',
      text: `จำนวน ${qty} ชิ้น ยอดรวม ${total} บาท ✓\n\nกรุณากรอกที่อยู่จัดส่ง:`,
    }]);
  }

  async _handleEnterAddress(userId, input, flow, replyToken, lineClient) {
    if (input.length < 10) {
      await lineClient.replyMessage(replyToken, [{
        type: 'text',
        text: 'กรุณากรอกที่อยู่ให้ครบถ้วน',
      }]);
      return;
    }
    
    await this.nextStep(userId, 'confirm', { address: input });
    
    const summary = [
      '📋 สรุปออเดอร์',
      '',
      `สินค้า: ${flow.selectedProduct.name}`,
      `ขนาด: ${flow.selectedSize}`,
      `จำนวน: ${flow.quantity} ชิ้น`,
      `ราคา: ฿${flow.unitPrice} × ${flow.quantity} = ฿${flow.totalPrice}`,
      `ที่อยู่: ${input}`,
      '',
      'พิมพ์ "ยืนยัน" เพื่อสั่ง หรือ "ยกเลิก" เพื่อยกเลิก',
    ].join('\n');
    
    await lineClient.replyMessage(replyToken, [{
      type: 'text',
      text: summary,
    }]);
  }

  async _handleConfirm(userId, input, flow, replyToken, lineClient) {
    if (input !== 'ยืนยัน') {
      await lineClient.replyMessage(replyToken, [{
        type: 'text',
        text: 'พิมพ์ "ยืนยัน" หรือ "ยกเลิก"',
      }]);
      return;
    }
    
    const orderNumber = `ORD${Date.now()}`;
    const finalData = await this.endFlow(userId);
    
    await lineClient.replyMessage(replyToken, [{
      type: 'flex',
      altText: 'ออเดอร์สำเร็จ',
      contents: {
        type: 'bubble',
        styles: { body: { backgroundColor: '#f0fff4' } },
        body: {
          type: 'box',
          layout: 'vertical',
          spacing: 'md',
          contents: [
            { type: 'text', text: '✅ รับออเดอร์แล้ว!', weight: 'bold', size: 'xl', color: '#00a844' },
            { type: 'text', text: `หมายเลข: ${orderNumber}`, weight: 'bold' },
            { type: 'separator', margin: 'md' },
            { type: 'text', text: `สินค้า: ${finalData.selectedProduct.name}` },
            { type: 'text', text: `ขนาด: ${finalData.selectedSize} × ${finalData.quantity}` },
            { type: 'text', text: `ยอดรวม: ฿${finalData.totalPrice}`, color: '#ff6b35', weight: 'bold' },
            { type: 'text', text: 'คาดว่าจะส่งถึงใน 30-45 นาที', color: '#888888', size: 'sm', margin: 'md' },
          ],
        },
      },
    }]);
  }
}
```

---

## 6. Pattern: Quiz Bot

```javascript
// =============================================
// PATTERN: Quiz Bot with Redis
// =============================================

class QuizManager {
  constructor(redis) {
    this.redis = redis;
    this.questions = [
      {
        id: 1,
        question: 'LINE เปิดตัวในปี?',
        options: ['2010', '2011', '2012', '2013'],
        correct: 1, // index 0-3
        points: 10,
      },
      {
        id: 2,
        question: 'LINE Messaging API ใช้ Protocol อะไร?',
        options: ['REST', 'GraphQL', 'WebSocket', 'gRPC'],
        correct: 0,
        points: 15,
      },
      {
        id: 3,
        question: 'LIFF ย่อมาจากอะไร?',
        options: [
          'LINE Interface Framework',
          'LINE Front-end Framework',
          'LINE Integration Framework',
          'LINE Interactive Framework',
        ],
        correct: 1,
        points: 20,
      },
    ];
  }

  getSessionKey(userId) {
    return `quiz:session:${userId}`;
  }

  async startQuiz(userId) {
    const session = {
      currentQuestion: 0,
      totalQuestions: this.questions.length,
      score: 0,
      answers: [],
      startedAt: Date.now(),
    };
    
    await this.redis.setex(
      this.getSessionKey(userId),
      600, // 10 นาที
      JSON.stringify(session)
    );
    
    // Update leaderboard
    await this.redis.zadd('quiz:leaderboard', 0, userId);
    
    return session;
  }

  async getSession(userId) {
    const data = await this.redis.get(this.getSessionKey(userId));
    return data ? JSON.parse(data) : null;
  }

  async getCurrentQuestion(userId) {
    const session = await this.getSession(userId);
    if (!session) return null;
    
    if (session.currentQuestion >= this.questions.length) {
      return { finished: true, score: session.score };
    }
    
    return {
      ...this.questions[session.currentQuestion],
      questionNumber: session.currentQuestion + 1,
      totalQuestions: session.totalQuestions,
      currentScore: session.score,
    };
  }

  async answerQuestion(userId, answerIndex) {
    const session = await this.getSession(userId);
    if (!session) return null;
    
    const question = this.questions[session.currentQuestion];
    if (!question) return null;
    
    const isCorrect = answerIndex === question.correct;
    const pointsEarned = isCorrect ? question.points : 0;
    
    session.answers.push({
      questionId: question.id,
      givenAnswer: answerIndex,
      correctAnswer: question.correct,
      isCorrect,
      pointsEarned,
    });
    
    session.score += pointsEarned;
    session.currentQuestion++;
    
    const isFinished = session.currentQuestion >= this.questions.length;
    
    if (isFinished) {
      // บันทึกคะแนนสุดท้าย
      await this.redis.zadd('quiz:leaderboard', session.score, userId);
      await this.redis.del(this.getSessionKey(userId));
    } else {
      await this.redis.setex(
        this.getSessionKey(userId),
        600,
        JSON.stringify(session)
      );
    }
    
    return {
      isCorrect,
      pointsEarned,
      totalScore: session.score,
      isFinished,
      correctAnswer: question.options[question.correct],
    };
  }

  async getLeaderboard(limit = 10) {
    const results = await this.redis.zrevrange(
      'quiz:leaderboard', 0, limit - 1, 'WITHSCORES'
    );
    
    const leaderboard = [];
    for (let i = 0; i < results.length; i += 2) {
      leaderboard.push({
        rank: leaderboard.length + 1,
        userId: results[i],
        score: parseFloat(results[i + 1]),
      });
    }
    
    return leaderboard;
  }
}

// =============================================
// Handler สำหรับ Quiz Bot
// =============================================
async function handleQuizBot(userId, message, replyToken, lineClient) {
  const quiz = new QuizManager(redis);
  
  // เริ่ม quiz
  if (message === 'เริ่มควิซ' || message === 'quiz') {
    await quiz.startQuiz(userId);
    const question = await quiz.getCurrentQuestion(userId);
    
    await lineClient.replyMessage(replyToken, [
      createQuizMessage(question)
    ]);
    return;
  }
  
  // ตอบคำถาม (A, B, C, D)
  const answerMap = { 'A': 0, 'B': 1, 'C': 2, 'D': 3 };
  const answerIndex = answerMap[message.toUpperCase()];
  
  if (answerIndex !== undefined) {
    const session = await quiz.getSession(userId);
    if (session) {
      const result = await quiz.answerQuestion(userId, answerIndex);
      
      if (result.isFinished) {
        // แสดงผลสรุป
        await lineClient.replyMessage(replyToken, [
          {
            type: 'text',
            text: result.isCorrect
              ? `✅ ถูก! +${result.pointsEarned} คะแนน`
              : `❌ ผิด! คำตอบที่ถูกคือ: ${result.correctAnswer}`,
          },
          createResultMessage(result.totalScore, 3),
        ]);
      } else {
        const nextQuestion = await quiz.getCurrentQuestion(userId);
        await lineClient.replyMessage(replyToken, [
          {
            type: 'text',
            text: result.isCorrect
              ? `✅ ถูก! +${result.pointsEarned} คะแนน (รวม: ${result.totalScore})`
              : `❌ ผิด! คำตอบที่ถูกคือ: ${result.correctAnswer}`,
          },
          createQuizMessage(nextQuestion),
        ]);
      }
      return;
    }
  }
  
  // แสดง leaderboard
  if (message === 'อันดับ' || message === 'leaderboard') {
    const leaderboard = await quiz.getLeaderboard(5);
    const text = leaderboard.length === 0
      ? 'ยังไม่มีผู้เล่น'
      : leaderboard.map(u => `${u.rank}. ${u.userId} - ${u.score} คะแนน`).join('\n');
    
    await lineClient.replyMessage(replyToken, [{
      type: 'text',
      text: `🏆 อันดับสูงสุด\n\n${text}`,
    }]);
  }
}

function createQuizMessage(question) {
  if (question.finished) {
    return {
      type: 'text',
      text: '🎉 จบแล้ว! พิมพ์ "อันดับ" เพื่อดูอันดับ',
    };
  }
  
  const optionLabels = ['A', 'B', 'C', 'D'];
  const optionsText = question.options
    .map((opt, i) => `${optionLabels[i]}. ${opt}`)
    .join('\n');
  
  return {
    type: 'text',
    text: `❓ ข้อ ${question.questionNumber}/${question.totalQuestions}\n\n${question.question}\n\n${optionsText}\n\nพิมพ์ A, B, C หรือ D`,
  };
}

function createResultMessage(score, totalQuestions) {
  const maxScore = 45; // 10+15+20
  const percentage = Math.round((score / maxScore) * 100);
  const grade = percentage >= 80 ? 'A' : percentage >= 60 ? 'B' : percentage >= 40 ? 'C' : 'F';
  
  return {
    type: 'flex',
    altText: 'ผลควิซ',
    contents: {
      type: 'bubble',
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          { type: 'text', text: '📊 ผลควิซ', weight: 'bold', size: 'xl' },
          { type: 'text', text: `คะแนน: ${score}/${maxScore}`, size: 'lg', margin: 'md' },
          { type: 'text', text: `เกรด: ${grade}`, size: 'xxl', weight: 'bold', color: grade === 'A' ? '#00a844' : '#ff6b35' },
          { type: 'text', text: `${percentage}%`, color: '#888888', margin: 'sm' },
        ],
      },
    },
  };
}
```

---

## 7. Redis + Python (redis-py)

```python
# redis_manager.py
import redis
import json
import time
from typing import Optional, Any, Dict, List

class RedisManager:
    def __init__(self):
        self.client = redis.Redis(
            host='localhost',
            port=6379,
            password=None,  # ใส่ password ถ้ามี
            decode_responses=True,  # return string แทน bytes
            socket_connect_timeout=5,
            socket_timeout=5,
            retry_on_timeout=True,
        )
        self.prefix = 'linebot:'
    
    def _key(self, key: str) -> str:
        return f"{self.prefix}{key}"
    
    # =============================================
    # String Operations
    # =============================================
    
    def set_state(self, user_id: str, state: dict, ttl: int = 1800) -> bool:
        """เก็บ state ของ user"""
        key = self._key(f"state:{user_id}")
        return self.client.setex(key, ttl, json.dumps(state))
    
    def get_state(self, user_id: str) -> Optional[dict]:
        """ดึง state ของ user"""
        key = self._key(f"state:{user_id}")
        data = self.client.get(key)
        return json.loads(data) if data else None
    
    def delete_state(self, user_id: str) -> bool:
        """ลบ state"""
        key = self._key(f"state:{user_id}")
        return self.client.delete(key) > 0
    
    # =============================================
    # Hash Operations
    # =============================================
    
    def set_session(self, user_id: str, session: dict, ttl: int = 3600):
        """เก็บ session ด้วย Hash"""
        key = self._key(f"session:{user_id}")
        flat = {k: json.dumps(v) if isinstance(v, (dict, list)) else str(v)
                for k, v in session.items()}
        pipe = self.client.pipeline()
        pipe.hset(key, mapping=flat)
        pipe.expire(key, ttl)
        pipe.execute()
    
    def get_session(self, user_id: str) -> Optional[dict]:
        """ดึง session"""
        key = self._key(f"session:{user_id}")
        data = self.client.hgetall(key)
        if not data:
            return None
        result = {}
        for k, v in data.items():
            try:
                result[k] = json.loads(v)
            except (json.JSONDecodeError, ValueError):
                result[k] = v
        return result
    
    def update_session_field(self, user_id: str, field: str, value: Any):
        """อัปเดต field เดียวใน session"""
        key = self._key(f"session:{user_id}")
        serialized = json.dumps(value) if isinstance(value, (dict, list)) else str(value)
        pipe = self.client.pipeline()
        pipe.hset(key, field, serialized)
        pipe.expire(key, 3600)
        pipe.execute()
    
    # =============================================
    # List Operations  
    # =============================================
    
    def add_to_history(self, user_id: str, message: dict, max_size: int = 20):
        """เพิ่มข้อความใน history"""
        key = self._key(f"history:{user_id}")
        entry = json.dumps({**message, 'timestamp': time.time()})
        
        pipe = self.client.pipeline()
        pipe.rpush(key, entry)
        pipe.ltrim(key, -max_size, -1)
        pipe.expire(key, 86400)  # 24 ชั่วโมง
        pipe.execute()
    
    def get_history(self, user_id: str, limit: int = 10) -> List[dict]:
        """ดึง history"""
        key = self._key(f"history:{user_id}")
        items = self.client.lrange(key, -limit, -1)
        return [json.loads(item) for item in items]
    
    # =============================================
    # Set Operations
    # =============================================
    
    def add_to_group(self, group: str, user_id: str):
        """เพิ่ม user เข้า group"""
        self.client.sadd(self._key(f"group:{group}"), user_id)
    
    def is_in_group(self, group: str, user_id: str) -> bool:
        """ตรวจสอบว่า user อยู่ใน group หรือไม่"""
        return self.client.sismember(self._key(f"group:{group}"), user_id)
    
    def get_group_members(self, group: str) -> List[str]:
        """ดึงสมาชิกของ group"""
        return list(self.client.smembers(self._key(f"group:{group}")))
    
    # =============================================
    # Sorted Set Operations (Leaderboard)
    # =============================================
    
    def add_points(self, user_id: str, points: float):
        """เพิ่มคะแนน"""
        self.client.zincrby(self._key('leaderboard'), points, user_id)
    
    def get_rank(self, user_id: str) -> Optional[int]:
        """ดึงอันดับ (1-indexed)"""
        rank = self.client.zrevrank(self._key('leaderboard'), user_id)
        return (rank + 1) if rank is not None else None
    
    def get_top_users(self, limit: int = 10) -> List[dict]:
        """ดึง top users"""
        results = self.client.zrevrange(
            self._key('leaderboard'), 0, limit - 1, withscores=True
        )
        return [
            {'rank': i + 1, 'userId': uid, 'points': int(score)}
            for i, (uid, score) in enumerate(results)
        ]
    
    # =============================================
    # Rate Limiting
    # =============================================
    
    def check_rate_limit(self, user_id: str, max_requests: int = 5, window: int = 60) -> dict:
        """ตรวจสอบ rate limit"""
        key = self._key(f"ratelimit:{user_id}")
        
        pipe = self.client.pipeline()
        pipe.incr(key)
        pipe.ttl(key)
        count, ttl = pipe.execute()
        
        if ttl == -1:
            self.client.expire(key, window)
            ttl = window
        
        return {
            'allowed': count <= max_requests,
            'count': count,
            'remaining': max(0, max_requests - count),
            'reset_in': ttl
        }
    
    # =============================================
    # Pub/Sub
    # =============================================
    
    def publish(self, channel: str, message: dict):
        """Publish ข้อความ"""
        self.client.publish(
            self._key(f"channel:{channel}"),
            json.dumps(message)
        )
    
    def subscribe(self, channel: str, callback):
        """Subscribe channel"""
        pubsub = self.client.pubsub()
        pubsub.subscribe(self._key(f"channel:{channel}"))
        
        for message in pubsub.listen():
            if message['type'] == 'message':
                data = json.loads(message['data'])
                callback(data)


# ตัวอย่างการใช้งาน
if __name__ == '__main__':
    rm = RedisManager()
    
    # Test state
    rm.set_state('U123', {'flow': 'order', 'step': 'select'})
    state = rm.get_state('U123')
    print(f"State: {state}")
    
    # Test leaderboard
    rm.add_points('U123', 100)
    rm.add_points('U456', 250)
    rm.add_points('U789', 175)
    
    top = rm.get_top_users(3)
    for user in top:
        print(f"#{user['rank']} {user['userId']}: {user['points']} pts")
    
    # Test rate limit
    for i in range(7):
        result = rm.check_rate_limit('U123', max_requests=5)
        print(f"Request {i+1}: allowed={result['allowed']}, remaining={result['remaining']}")
```

---

## 8. Redis Keyspace Notifications

```javascript
// =============================================
// REDIS KEYSPACE NOTIFICATIONS
// ตรวจสอบเมื่อ key หมดอายุ
// =============================================

const Redis = require('ioredis');

// Client สำหรับ subscribe
const subscriber = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: 6379,
});

// ตั้งค่า keyspace notifications ใน Redis
// notify-keyspace-events Ex (ต้องตั้งใน redis.conf หรือ runtime)
const publisher = new Redis({ host: process.env.REDIS_HOST || 'localhost' });
await publisher.config('SET', 'notify-keyspace-events', 'Ex');

// Subscribe to expired events
await subscriber.psubscribe('__keyevent@0__:expired');

subscriber.on('pmessage', async (pattern, channel, expiredKey) => {
  console.log(`Key expired: ${expiredKey}`);
  
  // ตรวจสอบว่าเป็น conversation state
  if (expiredKey.startsWith('linebot:state:')) {
    const userId = expiredKey.replace('linebot:state:', '');
    console.log(`Conversation state expired for user: ${userId}`);
    
    // ส่งแจ้งเตือน timeout ให้ user
    // (ต้องเก็บ pushToken ไว้ก่อน)
  }
  
  // ตรวจสอบว่าเป็น payment timeout
  if (expiredKey.startsWith('linebot:payment:')) {
    const orderId = expiredKey.replace('linebot:payment:', '');
    console.log(`Payment timeout for order: ${orderId}`);
    // ยกเลิก order
  }
});
```

---

## 9. Performance Tips

```javascript
// =============================================
// PERFORMANCE TIPS
// =============================================

// 1. Pipeline - รวม commands หลายอัน
async function batchOperations(userId, data) {
  const pipeline = redis.pipeline();
  
  pipeline.setex(`state:${userId}`, 1800, JSON.stringify(data.state));
  pipeline.hset(`session:${userId}`, data.session);
  pipeline.expire(`session:${userId}`, 3600);
  pipeline.zadd('active_users', Date.now(), userId);
  
  await pipeline.exec(); // ส่งทุก command พร้อมกัน
}

// 2. Lua Script - Atomic operations
const checkAndSetScript = `
  local current = redis.call('GET', KEYS[1])
  if current == false then
    redis.call('SETEX', KEYS[1], ARGV[1], ARGV[2])
    return 1
  end
  return 0
`;

async function atomicSetIfNotExists(key, ttl, value) {
  const result = await redis.eval(checkAndSetScript, 1, key, ttl, value);
  return result === 1;
}

// 3. Connection Pool
const redisPool = new Redis({
  host: process.env.REDIS_HOST,
  port: 6379,
  maxRetriesPerRequest: 3,
  enableAutoPipelining: true, // auto pipeline concurrent commands
  lazyConnect: true,
});

// 4. Compression สำหรับ large data
const zlib = require('zlib');
const { promisify } = require('util');
const compress = promisify(zlib.gzip);
const decompress = promisify(zlib.gunzip);

async function setCompressed(key, value, ttl) {
  const json = JSON.stringify(value);
  if (json.length > 1024) { // compress ถ้าใหญ่กว่า 1KB
    const compressed = await compress(json);
    await redis.setex(`${key}:gz`, ttl, compressed);
  } else {
    await redis.setex(key, ttl, json);
  }
}
```

---

## 10. Monitoring Redis

```javascript
// =============================================
// MONITORING
// =============================================

async function getRedisStats() {
  const info = await redis.info('all');
  const stats = {};
  
  info.split('\r\n').forEach(line => {
    const [key, value] = line.split(':');
    if (key && value) stats[key.trim()] = value.trim();
  });
  
  return {
    version: stats.redis_version,
    uptime: parseInt(stats.uptime_in_seconds),
    connectedClients: parseInt(stats.connected_clients),
    usedMemory: stats.used_memory_human,
    maxMemory: stats.maxmemory_human,
    hitRate: (
      parseInt(stats.keyspace_hits) /
      (parseInt(stats.keyspace_hits) + parseInt(stats.keyspace_misses)) * 100
    ).toFixed(2) + '%',
    totalCommands: parseInt(stats.total_commands_processed),
    keysTotal: parseInt(stats.db0?.split(',')[0]?.split('=')[1] || '0'),
  };
}

// Health check endpoint
async function redisHealthCheck() {
  try {
    const start = Date.now();
    await redis.ping();
    const latency = Date.now() - start;
    
    return {
      status: 'healthy',
      latency: `${latency}ms`,
    };
  } catch (error) {
    return {
      status: 'unhealthy',
      error: error.message,
    };
  }
}
```

---

## สรุป

Redis เป็นเครื่องมือสำคัญสำหรับ LINE bot ที่มีประสิทธิภาพสูง:

1. **String**: เก็บ state อย่างง่าย พร้อม TTL อัตโนมัติ
2. **Hash**: เก็บ session ที่มีหลาย fields ประหยัด memory
3. **List**: เก็บ conversation history และ message queues
4. **Set**: เก็บ user groups, permissions, online users
5. **Sorted Set**: leaderboard, rankings, activity tracking
6. **Pipeline**: รวม commands เพื่อลด round-trips
7. **Keyspace Notifications**: detect timeout events

การใช้ Redis ร่วมกับ LINE bot ทำให้:
- ตอบสนองเร็วขึ้น (microsecond latency)
- รองรับ concurrent users ได้มากขึ้น
- จัดการ session expiration ได้อัตโนมัติ
- Scale ได้ง่ายด้วย Redis Cluster
