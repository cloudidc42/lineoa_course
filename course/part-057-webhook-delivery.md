# Part 57: Webhook Reliability - ความน่าเชื่อถือของ Webhook

## บทนำ

Webhook เป็นหัวใจสำคัญของ LINE Bot ทุกครั้งที่ผู้ใช้ส่งข้อความหรือกระทำการใดๆ LINE จะส่ง HTTP POST request ไปยัง Webhook URL ของเรา หากระบบไม่สามารถรับหรือประมวลผล Webhook ได้อย่างเชื่อถือได้ ข้อความของผู้ใช้อาจหายไปหรือประมวลผลซ้ำ

### ปัญหาที่พบบ่อย

- Server ล่มชั่วคราว ทำให้ LINE ส่ง Webhook ซ้ำ
- ประมวลผล Webhook ช้าเกิน timeout
- Duplicate events จาก LINE
- Network latency ทำให้เกิด timeout

---

## 1. Webhook Reliability Architecture

### 1.1 โครงสร้างระบบที่ดี

```
ผู้ใช้ LINE
    │
    ▼
LINE Platform
    │  HTTP POST (Webhook)
    ▼
┌─────────────────────────────────────────────────────┐
│              Webhook Receiver (Express)              │
│  - รับ request ทันที                                │
│  - ตรวจสอบ Signature                                │
│  - ส่งไปที่ Queue                                   │
│  - ตอบ 200 OK ทันที                                 │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│                  Redis Queue (Bull)                   │
│  - เก็บ Job ที่รอประมวลผล                           │
│  - จัดการ Retry อัตโนมัติ                           │
│  - Dead Letter Queue                                │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│               Worker Processes                       │
│  - ประมวลผล Webhook                                 │
│  - เชื่อมต่อ Database                               │
│  - ส่ง Reply Message                                │
└─────────────────────────────────────────────────────┘
```

### 1.2 ติดตั้ง Dependencies

```bash
npm install bull redis ioredis @line/bot-sdk express dotenv
npm install --save-dev jest supertest
```

---

## 2. Webhook Signature Verification (การตรวจสอบ Signature)

### 2.1 ตรวจสอบ Signature อย่างถูกต้อง

```javascript
// middleware/webhook-validator.js
const crypto = require('crypto');

/**
 * ตรวจสอบความถูกต้องของ LINE Webhook Signature
 * LINE ใช้ HMAC-SHA256 กับ Channel Secret
 */
function validateWebhookSignature(req, res, next) {
  const channelSecret = process.env.LINE_CHANNEL_SECRET;
  const signature = req.headers['x-line-signature'];

  if (!signature) {
    console.error('ไม่พบ X-Line-Signature header');
    return res.status(401).json({ error: 'Missing signature' });
  }

  // คำนวณ HMAC-SHA256
  const body = req.rawBody || JSON.stringify(req.body);
  const expectedSignature = crypto
    .createHmac('sha256', channelSecret)
    .update(body)
    .digest('base64');

  // เปรียบเทียบแบบ constant-time เพื่อป้องกัน timing attack
  const signatureBuffer = Buffer.from(signature);
  const expectedBuffer = Buffer.from(expectedSignature);

  if (signatureBuffer.length !== expectedBuffer.length ||
      !crypto.timingSafeEqual(signatureBuffer, expectedBuffer)) {
    console.error('Signature ไม่ถูกต้อง');
    return res.status(401).json({ error: 'Invalid signature' });
  }

  next();
}

/**
 * Middleware สำหรับเก็บ raw body สำหรับการตรวจสอบ signature
 */
function captureRawBody(req, res, next) {
  let rawBody = '';
  req.on('data', chunk => {
    rawBody += chunk.toString();
  });
  req.on('end', () => {
    req.rawBody = rawBody;
    next();
  });
}

module.exports = { validateWebhookSignature, captureRawBody };
```

---

## 3. Idempotency Keys (กุญแจความไม่ซ้ำ)

### 3.1 การป้องกันการประมวลผลซ้ำ

```javascript
// utils/idempotency.js
const redis = require('../config/redis');

class IdempotencyManager {
  constructor() {
    this.keyPrefix = 'webhook:processed:';
    this.ttl = 86400; // 24 ชั่วโมง
  }

  /**
   * ตรวจสอบว่า Webhook นี้เคยประมวลผลแล้วหรือยัง
   * @param {string} eventId - ไอดีของ event (webhookEventId)
   * @returns {Promise<boolean>} true = เคยประมวลผลแล้ว
   */
  async isProcessed(eventId) {
    const key = `${this.keyPrefix}${eventId}`;
    const result = await redis.get(key);
    return result !== null;
  }

  /**
   * ทำเครื่องหมายว่าประมวลผลแล้ว
   * @param {string} eventId
   * @param {Object} result - ผลลัพธ์การประมวลผล
   */
  async markProcessed(eventId, result = {}) {
    const key = `${this.keyPrefix}${eventId}`;
    await redis.setex(
      key,
      this.ttl,
      JSON.stringify({ processedAt: new Date().toISOString(), ...result })
    );
  }

  /**
   * ดึงข้อมูลการประมวลผลก่อนหน้า
   * @param {string} eventId
   */
  async getProcessingResult(eventId) {
    const key = `${this.keyPrefix}${eventId}`;
    const result = await redis.get(key);
    return result ? JSON.parse(result) : null;
  }

  /**
   * ตรวจสอบหลาย Event พร้อมกัน
   * @param {string[]} eventIds
   * @returns {Promise<Set<string>>} ชุดของ eventId ที่เคยประมวลผลแล้ว
   */
  async getProcessedBatch(eventIds) {
    const keys = eventIds.map(id => `${this.keyPrefix}${id}`);
    const results = await redis.mget(...keys);
    const processed = new Set();

    eventIds.forEach((id, index) => {
      if (results[index] !== null) {
        processed.add(id);
      }
    });

    return processed;
  }
}

module.exports = new IdempotencyManager();
```

---

## 4. Queue System with Bull (ระบบ Queue)

### 4.1 ตั้งค่า Bull Queue

```javascript
// config/queue.js
const Bull = require('bull');

const REDIS_URL = process.env.REDIS_URL || 'redis://localhost:6379';

// สร้าง Queue หลัก
const webhookQueue = new Bull('webhook-processing', REDIS_URL, {
  defaultJobOptions: {
    attempts: 3,
    backoff: {
      type: 'exponential',
      delay: 1000 // เริ่มต้นที่ 1 วินาที
    },
    removeOnComplete: 100, // เก็บ completed jobs ล่าสุด 100 jobs
    removeOnFail: 50       // เก็บ failed jobs ล่าสุด 50 jobs
  }
});

// Dead Letter Queue
const deadLetterQueue = new Bull('webhook-dead-letter', REDIS_URL);

// Queue Events
webhookQueue.on('completed', (job, result) => {
  console.log(`Job ${job.id} เสร็จสมบูรณ์:`, result);
});

webhookQueue.on('failed', (job, err) => {
  console.error(`Job ${job.id} ล้มเหลว (attempt ${job.attemptsMade}):`, err.message);
});

webhookQueue.on('stalled', (job) => {
  console.warn(`Job ${job.id} หยุดค้างอยู่`);
});

module.exports = { webhookQueue, deadLetterQueue };
```

### 4.2 Webhook Receiver

```javascript
// webhook-receiver.js
require('dotenv').config();
const express = require('express');
const { validateWebhookSignature, captureRawBody } = require('./middleware/webhook-validator');
const { webhookQueue } = require('./config/queue');
const idempotency = require('./utils/idempotency');

const app = express();

// เก็บ raw body ก่อน parse JSON
app.use(captureRawBody);
app.use(express.json());

// Webhook endpoint
app.post('/webhook', validateWebhookSignature, async (req, res) => {
  // ตอบ 200 ทันทีก่อน
  res.sendStatus(200);

  const { events, destination } = req.body;

  if (!events || events.length === 0) return;

  // ส่ง events ไปที่ Queue
  for (const event of events) {
    const eventId = event.webhookEventId;

    try {
      // ตรวจสอบ idempotency
      if (await idempotency.isProcessed(eventId)) {
        console.log(`Event ${eventId} เคยประมวลผลแล้ว ข้าม`);
        continue;
      }

      // เพิ่ม job ไปที่ Queue
      await webhookQueue.add({
        event,
        destination,
        receivedAt: Date.now()
      }, {
        jobId: eventId, // ใช้ eventId เป็น jobId เพื่อป้องกัน duplicate
        priority: getEventPriority(event.type)
      });

      console.log(`เพิ่ม Event ${eventId} (${event.type}) ไปที่ Queue`);
    } catch (error) {
      if (error.message.includes('Job already exists')) {
        console.log(`Job ${eventId} มีอยู่แล้ว ข้าม`);
      } else {
        console.error(`ไม่สามารถเพิ่ม Event ไปที่ Queue: ${error.message}`);
      }
    }
  }
});

// กำหนด Priority ตามประเภท Event
function getEventPriority(eventType) {
  const priorities = {
    'message': 1,    // สำคัญที่สุด
    'follow': 2,
    'unfollow': 2,
    'join': 3,
    'leave': 3,
    'postback': 2,
    'memberJoined': 3,
    'memberLeft': 3
  };
  return priorities[eventType] || 5;
}

app.listen(3000, () => {
  console.log('Webhook Receiver พร้อมรับข้อมูลที่ port 3000');
});
```

### 4.3 Worker Process

```javascript
// worker.js
require('dotenv').config();
const { webhookQueue, deadLetterQueue } = require('./config/queue');
const idempotency = require('./utils/idempotency');
const EventProcessor = require('./services/event-processor');

const processor = new EventProcessor();

// ประมวลผล Webhook Jobs
webhookQueue.process('*', 5, async (job) => {
  const { event, destination } = job.data;
  const eventId = event.webhookEventId;

  console.log(`ประมวลผล Event ${eventId} (${event.type})`);

  // ตรวจสอบอีกครั้งเพื่อป้องกัน race condition
  if (await idempotency.isProcessed(eventId)) {
    console.log(`Event ${eventId} ประมวลผลแล้ว ข้าม`);
    return { skipped: true };
  }

  try {
    const result = await processor.processEvent(event, destination);

    // บันทึกว่าประมวลผลแล้ว
    await idempotency.markProcessed(eventId, { success: true });

    return result;
  } catch (error) {
    console.error(`เกิดข้อผิดพลาดในการประมวลผล Event ${eventId}:`, error);
    throw error; // Bull จะ retry อัตโนมัติ
  }
});

// จัดการ Failed Jobs -> Dead Letter Queue
webhookQueue.on('failed', async (job, err) => {
  if (job.attemptsMade >= job.opts.attempts) {
    console.error(`Event ${job.id} ล้มเหลวทุก Attempt ส่งไป Dead Letter Queue`);

    await deadLetterQueue.add({
      originalJob: job.data,
      error: err.message,
      failedAt: new Date().toISOString(),
      attempts: job.attemptsMade
    });
  }
});

console.log('Worker เริ่มทำงานแล้ว');
```

---

## 5. Event Processor (ตัวประมวลผล Event)

### 5.1 สร้าง Event Processor

```javascript
// services/event-processor.js
const line = require('@line/bot-sdk');

const lineClient = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

class EventProcessor {
  async processEvent(event, destination) {
    switch (event.type) {
      case 'message':
        return this.handleMessage(event);
      case 'follow':
        return this.handleFollow(event);
      case 'unfollow':
        return this.handleUnfollow(event);
      case 'postback':
        return this.handlePostback(event);
      case 'join':
        return this.handleJoin(event);
      case 'leave':
        return this.handleLeave(event);
      default:
        console.log(`Event type ${event.type} ไม่มีตัวจัดการ`);
        return { handled: false, eventType: event.type };
    }
  }

  async handleMessage(event) {
    const { replyToken, message, source } = event;

    switch (message.type) {
      case 'text':
        return this.handleTextMessage(replyToken, message.text, source);
      case 'image':
        return this.handleImageMessage(replyToken, message.id, source);
      case 'sticker':
        return this.handleStickerMessage(replyToken, message, source);
      default:
        await lineClient.replyMessage(replyToken, {
          type: 'text',
          text: `รับข้อความประเภท ${message.type} แล้ว`
        });
        return { handled: true, messageType: message.type };
    }
  }

  async handleTextMessage(replyToken, text, source) {
    let replyText = '';

    if (text.toLowerCase() === 'สวัสดี' || text.toLowerCase() === 'hello') {
      replyText = 'สวัสดีครับ! มีอะไรให้ช่วยไหมครับ?';
    } else if (text.includes('ราคา') || text.includes('price')) {
      replyText = 'กรุณาติดต่อทีมงานเพื่อสอบถามราคาครับ';
    } else {
      replyText = `ได้รับข้อความ: "${text}" ขอบคุณครับ`;
    }

    await lineClient.replyMessage(replyToken, {
      type: 'text',
      text: replyText
    });

    return { handled: true, reply: replyText };
  }

  async handleImageMessage(replyToken, messageId, source) {
    await lineClient.replyMessage(replyToken, {
      type: 'text',
      text: 'ได้รับรูปภาพแล้ว ขอบคุณครับ!'
    });
    return { handled: true, messageId };
  }

  async handleStickerMessage(replyToken, message, source) {
    // ตอบด้วย Sticker เดียวกัน
    await lineClient.replyMessage(replyToken, {
      type: 'sticker',
      packageId: message.packageId,
      stickerId: message.stickerId
    });
    return { handled: true, sticker: `${message.packageId}/${message.stickerId}` };
  }

  async handleFollow(event) {
    const { replyToken, source } = event;
    await lineClient.replyMessage(replyToken, {
      type: 'text',
      text: 'ยินดีต้อนรับนะครับ! ขอบคุณที่ติดตามเรา 🎉'
    });
    return { handled: true, userId: source.userId };
  }

  async handleUnfollow(event) {
    console.log(`ผู้ใช้ ${event.source.userId} ยกเลิกติดตาม`);
    return { handled: true, userId: event.source.userId };
  }

  async handlePostback(event) {
    const { replyToken, postback } = event;
    const data = new URLSearchParams(postback.data);

    await lineClient.replyMessage(replyToken, {
      type: 'text',
      text: `รับ Postback: ${postback.data}`
    });

    return { handled: true, action: data.get('action') };
  }

  async handleJoin(event) {
    const { replyToken } = event;
    await lineClient.replyMessage(replyToken, {
      type: 'text',
      text: 'สวัสดีทุกคน! ขอบคุณที่เชิญฉันเข้ากลุ่มนะครับ 😊'
    });
    return { handled: true };
  }

  async handleLeave(event) {
    console.log('Bot ถูกนำออกจากกลุ่ม');
    return { handled: true };
  }
}

module.exports = EventProcessor;
```

---

## 6. Dead Letter Queue Processing (คิวข้อความที่ล้มเหลว)

### 6.1 DLQ Processor

```javascript
// services/dlq-processor.js
const { deadLetterQueue } = require('../config/queue');
const AlertService = require('./alert-service');

const alertService = new AlertService();

// ประมวลผล Dead Letter Queue
deadLetterQueue.process(async (job) => {
  const { originalJob, error, failedAt, attempts } = job.data;
  const event = originalJob.event;

  console.error(`[DLQ] ประมวลผล Failed Job:`);
  console.error(`  Event ID: ${event.webhookEventId}`);
  console.error(`  Event Type: ${event.type}`);
  console.error(`  Error: ${error}`);
  console.error(`  Failed At: ${failedAt}`);
  console.error(`  Attempts: ${attempts}`);

  // บันทึกลงฐานข้อมูล
  await saveFailedEvent(job.data);

  // ส่งการแจ้งเตือน
  await alertService.sendAlert({
    level: 'error',
    title: 'Webhook Processing Failed',
    message: `Event ${event.webhookEventId} ล้มเหลวหลังจาก ${attempts} attempts`,
    details: { event: event.type, error, failedAt }
  });
});

async function saveFailedEvent(data) {
  // บันทึกลงฐานข้อมูลเพื่อ manual review
  const db = require('../database/db');
  db.prepare(`
    INSERT INTO failed_events (event_id, event_type, error, attempts, failed_at, raw_data)
    VALUES (?, ?, ?, ?, ?, ?)
  `).run(
    data.originalJob.event.webhookEventId,
    data.originalJob.event.type,
    data.error,
    data.attempts,
    data.failedAt,
    JSON.stringify(data.originalJob)
  );
}
```

---

## 7. Retry Strategies (กลยุทธ์การ Retry)

### 7.1 Exponential Backoff

```javascript
// utils/retry.js
class RetryStrategy {
  /**
   * Exponential Backoff
   * 1st retry: 1s, 2nd: 2s, 3rd: 4s, 4th: 8s ...
   */
  static exponentialBackoff(attempt, baseDelay = 1000, maxDelay = 30000) {
    const delay = Math.min(baseDelay * Math.pow(2, attempt - 1), maxDelay);
    // เพิ่ม Jitter เพื่อป้องกัน thundering herd
    const jitter = Math.random() * delay * 0.1;
    return delay + jitter;
  }

  /**
   * Linear Backoff
   */
  static linearBackoff(attempt, increment = 1000) {
    return attempt * increment;
  }

  /**
   * Fixed Delay
   */
  static fixedDelay(delay = 5000) {
    return delay;
  }
}

/**
 * Retry Function with configurable strategy
 */
async function withRetry(fn, options = {}) {
  const {
    maxAttempts = 3,
    strategy = 'exponential',
    baseDelay = 1000,
    onError = null,
    shouldRetry = () => true
  } = options;

  let lastError;

  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error;

      if (onError) {
        await onError(error, attempt);
      }

      if (attempt === maxAttempts || !shouldRetry(error)) {
        throw error;
      }

      let delay;
      switch (strategy) {
        case 'exponential':
          delay = RetryStrategy.exponentialBackoff(attempt, baseDelay);
          break;
        case 'linear':
          delay = RetryStrategy.linearBackoff(attempt, baseDelay);
          break;
        default:
          delay = RetryStrategy.fixedDelay(baseDelay);
      }

      console.log(`Retry attempt ${attempt}/${maxAttempts} ใน ${Math.round(delay)}ms`);
      await new Promise(resolve => setTimeout(resolve, delay));
    }
  }

  throw lastError;
}

module.exports = { RetryStrategy, withRetry };
```

---

## 8. Circuit Breaker Pattern

### 8.1 Circuit Breaker Implementation

```javascript
// utils/circuit-breaker.js

const STATES = {
  CLOSED: 'CLOSED',     // ปกติ
  OPEN: 'OPEN',         // เปิดอยู่ (ไม่ให้ผ่าน)
  HALF_OPEN: 'HALF_OPEN' // ทดสอบ
};

class CircuitBreaker {
  constructor(options = {}) {
    this.failureThreshold = options.failureThreshold || 5;
    this.successThreshold = options.successThreshold || 2;
    this.timeout = options.timeout || 60000; // 1 นาที

    this.state = STATES.CLOSED;
    this.failureCount = 0;
    this.successCount = 0;
    this.nextAttempt = Date.now();
  }

  async execute(fn) {
    if (this.state === STATES.OPEN) {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit Breaker เปิดอยู่ - กรุณาลองใหม่ภายหลัง');
      }
      // เปลี่ยนเป็น HALF_OPEN
      this.state = STATES.HALF_OPEN;
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  onSuccess() {
    this.failureCount = 0;

    if (this.state === STATES.HALF_OPEN) {
      this.successCount++;
      if (this.successCount >= this.successThreshold) {
        this.state = STATES.CLOSED;
        this.successCount = 0;
        console.log('Circuit Breaker ปิดแล้ว (CLOSED)');
      }
    }
  }

  onFailure() {
    this.failureCount++;

    if (this.state === STATES.HALF_OPEN) {
      this.state = STATES.OPEN;
      this.nextAttempt = Date.now() + this.timeout;
      console.log('Circuit Breaker เปิดอีกครั้ง (OPEN)');
      return;
    }

    if (this.failureCount >= this.failureThreshold) {
      this.state = STATES.OPEN;
      this.nextAttempt = Date.now() + this.timeout;
      console.log(`Circuit Breaker เปิดแล้ว หลังจากล้มเหลว ${this.failureCount} ครั้ง`);
    }
  }

  getState() {
    return {
      state: this.state,
      failureCount: this.failureCount,
      nextAttempt: this.state === STATES.OPEN ? new Date(this.nextAttempt) : null
    };
  }
}

module.exports = CircuitBreaker;
```

---

## 9. Webhook Timeout Best Practices

### 9.1 ตั้งค่า Timeout อย่างถูกต้อง

```javascript
// config/timeouts.js
module.exports = {
  // LINE ต้องการ response ภายใน 5 วินาที
  WEBHOOK_RESPONSE_TIMEOUT: 4500,

  // Timeout สำหรับ LINE API calls
  LINE_API_TIMEOUT: 10000,

  // Timeout สำหรับ Database queries
  DB_TIMEOUT: 5000,

  // Timeout สำหรับ External services
  EXTERNAL_SERVICE_TIMEOUT: 8000
};
```

### 9.2 Webhook Handler with Timeout Guard

```javascript
// middleware/timeout-guard.js
const timeouts = require('../config/timeouts');

function webhookTimeoutGuard(req, res, next) {
  const startTime = Date.now();

  // ตั้ง timeout ก่อนที่ LINE จะ timeout
  const timer = setTimeout(() => {
    if (!res.headersSent) {
      console.warn(`Webhook timeout หลังจาก ${Date.now() - startTime}ms`);
      res.sendStatus(200); // ตอบ 200 เพื่อไม่ให้ LINE retry
    }
  }, timeouts.WEBHOOK_RESPONSE_TIMEOUT);

  // ล้าง timer เมื่อ response ถูกส่ง
  res.on('finish', () => clearTimeout(timer));
  res.on('close', () => clearTimeout(timer));

  next();
}

module.exports = webhookTimeoutGuard;
```

---

## 10. Monitoring Webhook Processing

### 10.1 Metrics Collection

```javascript
// services/metrics-service.js
const redis = require('../config/redis');

class MetricsService {
  constructor() {
    this.keyPrefix = 'metrics:webhook:';
  }

  async recordReceived(eventType) {
    const key = `${this.keyPrefix}received:${eventType}`;
    await redis.incr(key);
    await redis.expire(key, 86400);
  }

  async recordProcessed(eventType, durationMs) {
    const pipe = redis.pipeline();

    // นับจำนวน
    pipe.incr(`${this.keyPrefix}processed:${eventType}`);

    // เก็บ duration
    pipe.lpush(`${this.keyPrefix}duration:${eventType}`, durationMs);
    pipe.ltrim(`${this.keyPrefix}duration:${eventType}`, 0, 999); // เก็บ 1000 ค่าล่าสุด

    await pipe.exec();
  }

  async recordFailed(eventType, error) {
    const pipe = redis.pipeline();
    pipe.incr(`${this.keyPrefix}failed:${eventType}`);
    pipe.incr(`${this.keyPrefix}errors:${error.name || 'Unknown'}`);
    await pipe.exec();
  }

  async getStats() {
    const eventTypes = ['message', 'follow', 'unfollow', 'postback', 'join'];
    const stats = {};

    for (const type of eventTypes) {
      const [received, processed, failed] = await Promise.all([
        redis.get(`${this.keyPrefix}received:${type}`),
        redis.get(`${this.keyPrefix}processed:${type}`),
        redis.get(`${this.keyPrefix}failed:${type}`)
      ]);

      const durations = await redis.lrange(`${this.keyPrefix}duration:${type}`, 0, -1);
      const avgDuration = durations.length > 0
        ? durations.reduce((a, b) => a + parseInt(b), 0) / durations.length
        : 0;

      stats[type] = {
        received: parseInt(received) || 0,
        processed: parseInt(processed) || 0,
        failed: parseInt(failed) || 0,
        avgDurationMs: Math.round(avgDuration)
      };
    }

    return stats;
  }
}

module.exports = new MetricsService();
```

### 10.2 Health Check Endpoint

```javascript
// routes/health.js
const express = require('express');
const router = express.Router();
const { webhookQueue, deadLetterQueue } = require('../config/queue');
const metrics = require('../services/metrics-service');

router.get('/health', async (req, res) => {
  try {
    const [queueCounts, dlqCounts, stats] = await Promise.all([
      webhookQueue.getJobCounts(),
      deadLetterQueue.getJobCounts(),
      metrics.getStats()
    ]);

    const health = {
      status: 'healthy',
      timestamp: new Date().toISOString(),
      queue: {
        waiting: queueCounts.waiting,
        active: queueCounts.active,
        completed: queueCounts.completed,
        failed: queueCounts.failed,
        delayed: queueCounts.delayed
      },
      deadLetterQueue: {
        waiting: dlqCounts.waiting,
        failed: dlqCounts.failed
      },
      stats
    };

    // ตรวจสอบสถานะ
    if (queueCounts.failed > 100 || dlqCounts.waiting > 50) {
      health.status = 'degraded';
    }

    const statusCode = health.status === 'healthy' ? 200 : 503;
    res.status(statusCode).json(health);
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message,
      timestamp: new Date().toISOString()
    });
  }
});

module.exports = router;
```

---

## 11. Complete Webhook Handler

```javascript
// Complete webhook-handler.js
require('dotenv').config();
const express = require('express');
const { validateWebhookSignature, captureRawBody } = require('./middleware/webhook-validator');
const webhookTimeoutGuard = require('./middleware/timeout-guard');
const { webhookQueue } = require('./config/queue');
const idempotency = require('./utils/idempotency');
const metrics = require('./services/metrics-service');
const healthRoutes = require('./routes/health');

const app = express();

// Middleware
app.use(captureRawBody);
app.use(express.json());

// Health check
app.use('/', healthRoutes);

// Webhook endpoint
app.post('/webhook',
  webhookTimeoutGuard,
  validateWebhookSignature,
  async (req, res) => {
    const receiveTime = Date.now();

    // ตอบ LINE ทันที
    res.sendStatus(200);

    const { events, destination } = req.body;
    if (!events || events.length === 0) return;

    for (const event of events) {
      const eventId = event.webhookEventId;

      try {
        await metrics.recordReceived(event.type);

        // ตรวจสอบ idempotency
        if (await idempotency.isProcessed(eventId)) {
          console.log(`[SKIP] Event ${eventId} ประมวลผลแล้ว`);
          continue;
        }

        // เพิ่มไปที่ Queue
        await webhookQueue.add({
          event,
          destination,
          receivedAt: receiveTime
        }, {
          jobId: eventId,
          priority: getEventPriority(event.type)
        });
      } catch (error) {
        if (!error.message.includes('already exists')) {
          console.error(`Queue error: ${error.message}`);
          await metrics.recordFailed(event.type, error);
        }
      }
    }
  }
);

function getEventPriority(eventType) {
  const priorities = {
    'message': 1, 'follow': 2, 'unfollow': 2,
    'postback': 2, 'join': 3, 'leave': 3
  };
  return priorities[eventType] || 5;
}

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Webhook Handler พร้อมที่ port ${PORT}`);
});

module.exports = app;
```

---

## 12. Testing

### 12.1 Unit Tests

```javascript
// tests/webhook-validator.test.js
const crypto = require('crypto');
const request = require('supertest');
const express = require('express');
const { validateWebhookSignature, captureRawBody } = require('../middleware/webhook-validator');

describe('Webhook Validator', () => {
  let app;
  const channelSecret = 'test-channel-secret';

  beforeEach(() => {
    process.env.LINE_CHANNEL_SECRET = channelSecret;
    app = express();
    app.use(captureRawBody);
    app.use(express.json());
    app.post('/webhook', validateWebhookSignature, (req, res) => {
      res.json({ success: true });
    });
  });

  test('ควรยอมรับ Webhook ที่มี Signature ถูกต้อง', async () => {
    const body = JSON.stringify({ events: [] });
    const signature = crypto
      .createHmac('sha256', channelSecret)
      .update(body)
      .digest('base64');

    const res = await request(app)
      .post('/webhook')
      .set('x-line-signature', signature)
      .set('content-type', 'application/json')
      .send(body);

    expect(res.status).toBe(200);
    expect(res.body.success).toBe(true);
  });

  test('ควรปฏิเสธ Webhook ที่ไม่มี Signature', async () => {
    const res = await request(app)
      .post('/webhook')
      .send({ events: [] });

    expect(res.status).toBe(401);
  });

  test('ควรปฏิเสธ Webhook ที่มี Signature ผิด', async () => {
    const res = await request(app)
      .post('/webhook')
      .set('x-line-signature', 'invalid-signature')
      .send({ events: [] });

    expect(res.status).toBe(401);
  });
});
```

### 12.2 Idempotency Tests

```javascript
// tests/idempotency.test.js
const idempotency = require('../utils/idempotency');

// Mock Redis
jest.mock('../config/redis', () => {
  const store = new Map();
  return {
    get: jest.fn(async (key) => store.get(key) || null),
    setex: jest.fn(async (key, ttl, value) => store.set(key, value)),
    mget: jest.fn(async (...keys) => keys.map(k => store.get(k) || null))
  };
});

describe('IdempotencyManager', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('isProcessed ควรคืนค่า false สำหรับ Event ใหม่', async () => {
    const result = await idempotency.isProcessed('new-event-id');
    expect(result).toBe(false);
  });

  test('isProcessed ควรคืนค่า true หลังจาก markProcessed', async () => {
    await idempotency.markProcessed('test-event-id', { success: true });
    const result = await idempotency.isProcessed('test-event-id');
    expect(result).toBe(true);
  });

  test('getProcessingResult ควรคืนค่าข้อมูลที่บันทึก', async () => {
    await idempotency.markProcessed('result-event-id', { handled: true });
    const result = await idempotency.getProcessingResult('result-event-id');
    expect(result).toMatchObject({ handled: true });
  });
});
```

### 12.3 Circuit Breaker Tests

```javascript
// tests/circuit-breaker.test.js
const CircuitBreaker = require('../utils/circuit-breaker');

describe('CircuitBreaker', () => {
  test('ควรอยู่ใน CLOSED state เริ่มต้น', () => {
    const cb = new CircuitBreaker({ failureThreshold: 3 });
    expect(cb.getState().state).toBe('CLOSED');
  });

  test('ควรเปลี่ยนเป็น OPEN หลังจากล้มเหลวเกิน threshold', async () => {
    const cb = new CircuitBreaker({ failureThreshold: 3, timeout: 1000 });
    const failFn = () => Promise.reject(new Error('ล้มเหลว'));

    for (let i = 0; i < 3; i++) {
      try { await cb.execute(failFn); } catch (e) {}
    }

    expect(cb.getState().state).toBe('OPEN');
  });

  test('ควรโยน Error เมื่ออยู่ใน OPEN state', async () => {
    const cb = new CircuitBreaker({ failureThreshold: 1 });
    const failFn = () => Promise.reject(new Error('ล้มเหลว'));

    try { await cb.execute(failFn); } catch (e) {}
    await expect(cb.execute(failFn)).rejects.toThrow('Circuit Breaker เปิดอยู่');
  });
});
```

---

## 13. Production Configuration

### 13.1 Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  webhook-receiver:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
      - LINE_CHANNEL_ACCESS_TOKEN=${LINE_CHANNEL_ACCESS_TOKEN}
      - LINE_CHANNEL_SECRET=${LINE_CHANNEL_SECRET}
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    restart: unless-stopped

  worker:
    build: .
    command: node worker.js
    environment:
      - NODE_ENV=production
      - LINE_CHANNEL_ACCESS_TOKEN=${LINE_CHANNEL_ACCESS_TOKEN}
      - LINE_CHANNEL_SECRET=${LINE_CHANNEL_SECRET}
      - REDIS_URL=redis://redis:6379
    depends_on:
      - redis
    restart: unless-stopped
    deploy:
      replicas: 3

  redis:
    image: redis:7-alpine
    volumes:
      - redis-data:/data
    restart: unless-stopped
    command: redis-server --appendonly yes

volumes:
  redis-data:
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Webhook Architecture** - โครงสร้างที่เชื่อถือได้ด้วย Queue
2. **Signature Verification** - ตรวจสอบความถูกต้องของ Webhook
3. **Idempotency** - ป้องกันการประมวลผลซ้ำ
4. **Bull Queue** - ระบบ Queue ด้วย Redis
5. **Retry Strategies** - Exponential Backoff
6. **Circuit Breaker** - ป้องกัน Cascade Failures
7. **Dead Letter Queue** - จัดการ Failed Events
8. **Monitoring** - ติดตามสถานะระบบ
9. **Testing** - Unit tests ครอบคลุม

ขั้นตอนต่อไป: ใน Part 58 เราจะเรียนรู้เรื่อง Error Handling แบบครบวงจร
