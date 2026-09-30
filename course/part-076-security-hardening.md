# Part 76: Security Hardening สำหรับ LINE Bot

## บทนำ

การรักษาความปลอดภัยสำหรับ LINE Bot เป็นสิ่งสำคัญอย่างยิ่ง เนื่องจาก Bot ของคุณมักจะจัดการข้อมูลส่วนตัวของผู้ใช้ เชื่อมต่อกับฐานข้อมูล และเข้าถึง LINE Messaging API ซึ่งมีค่าใช้จ่าย หากถูกโจมตี อาจเกิดความเสียหายทั้งด้านข้อมูลและการเงิน

บทนี้จะครอบคลุมทุกด้านของการรักษาความปลอดภัย ตั้งแต่การตรวจสอบ Webhook Signature ไปจนถึงการป้องกัน DDoS และการจัดการ Incident Response

```
┌─────────────────────────────────────────────────────────────────┐
│                    Security Layers                               │
│                                                                  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐   │
│  │ Network  │  │ App Layer│  │   Data   │  │  Monitoring  │   │
│  │ Security │→ │ Security │→ │ Security │→ │  & Alerting  │   │
│  │ (DDoS,  │  │ (Input   │  │ (Encrypt,│  │  (Logs,      │   │
│  │  HTTPS) │  │  Valid.) │  │  Backup) │  │   Alerts)    │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. Security Checklist สำหรับ LINE Bot

### 1.1 Checklist ก่อน Deploy Production

```markdown
## LINE Bot Security Checklist

### Authentication & Authorization
- [ ] ตรวจสอบ Webhook Signature ทุก request
- [ ] ไม่ hardcode Channel Secret หรือ Access Token
- [ ] ใช้ environment variables สำหรับ secrets ทุกตัว
- [ ] ตั้งค่า IP whitelist สำหรับ management endpoints
- [ ] ใช้ short-lived access tokens เมื่อเป็นไปได้

### Input Validation
- [ ] Validate ข้อมูลทุกอย่างที่รับจาก user
- [ ] Sanitize ข้อมูลก่อนนำไปประมวลผล
- [ ] จำกัดขนาด request body
- [ ] ป้องกัน SQL Injection
- [ ] ป้องกัน Command Injection

### LIFF Security
- [ ] ใช้ HTTPS เท่านั้น
- [ ] ตั้งค่า Content Security Policy
- [ ] ป้องกัน XSS ในทุก output
- [ ] ใช้ CSRF tokens
- [ ] Validate LIFF ID Token

### Infrastructure
- [ ] ใช้ HTTPS/TLS 1.2+ เท่านั้น
- [ ] ตั้งค่า Security Headers
- [ ] Rate limiting บน webhook endpoint
- [ ] DDoS protection (Cloudflare/AWS Shield)
- [ ] ตั้งค่า WAF rules

### Data Security
- [ ] เข้ารหัสข้อมูลสำคัญใน database
- [ ] ไม่เก็บข้อมูลที่ไม่จำเป็น
- [ ] กำหนด data retention policy
- [ ] Backup data อย่างสม่ำเสมอ

### Monitoring
- [ ] Log ทุก webhook request
- [ ] Alert เมื่อมี error rate สูงผิดปกติ
- [ ] Monitor API quota usage
- [ ] ตั้งค่า intrusion detection
```

---

## 2. Webhook Signature Verification

### 2.1 ทำความเข้าใจ LINE Webhook Signature

LINE ส่ง signature มาใน header `X-Line-Signature` ซึ่งเป็น HMAC-SHA256 hash ของ request body โดยใช้ Channel Secret เป็น key

```
Request Flow:
LINE Server → [HMAC-SHA256 signature] → Your Bot

Verification:
1. รับ request body (raw bytes)
2. คำนวณ HMAC-SHA256 ด้วย Channel Secret
3. เปรียบเทียบกับ X-Line-Signature header
4. ถ้าไม่ตรงกัน → ปฏิเสธ request
```

### 2.2 การ Implement ใน Node.js (Express)

```javascript
// middleware/lineSignatureVerification.js
const crypto = require('crypto');

/**
 * Middleware สำหรับตรวจสอบ LINE Webhook Signature
 * ต้องใช้ raw body ไม่ใช่ parsed body
 */
function verifyLineSignature(req, res, next) {
  const channelSecret = process.env.LINE_CHANNEL_SECRET;
  
  if (!channelSecret) {
    console.error('LINE_CHANNEL_SECRET is not set');
    return res.status(500).json({ error: 'Server configuration error' });
  }
  
  // รับ signature จาก header
  const signature = req.headers['x-line-signature'];
  
  if (!signature) {
    console.warn('Missing X-Line-Signature header', {
      ip: req.ip,
      userAgent: req.headers['user-agent'],
      timestamp: new Date().toISOString()
    });
    return res.status(401).json({ error: 'Missing signature' });
  }
  
  // ต้องการ raw body สำหรับการคำนวณ signature
  const body = req.rawBody;
  
  if (!body) {
    console.error('Raw body not available. Make sure to use rawBodyParser middleware');
    return res.status(500).json({ error: 'Server configuration error' });
  }
  
  // คำนวณ expected signature
  const expectedSignature = crypto
    .createHmac('SHA256', channelSecret)
    .update(body)
    .digest('base64');
  
  // เปรียบเทียบแบบ constant-time เพื่อป้องกัน timing attacks
  const signatureBuffer = Buffer.from(signature, 'base64');
  const expectedBuffer = Buffer.from(expectedSignature, 'base64');
  
  if (signatureBuffer.length !== expectedBuffer.length) {
    logSecurityEvent('invalid_signature', {
      ip: req.ip,
      reason: 'length_mismatch'
    });
    return res.status(401).json({ error: 'Invalid signature' });
  }
  
  // crypto.timingSafeEqual ป้องกัน timing attack
  if (!crypto.timingSafeEqual(signatureBuffer, expectedBuffer)) {
    logSecurityEvent('invalid_signature', {
      ip: req.ip,
      receivedSignature: signature.substring(0, 10) + '...',
      reason: 'value_mismatch'
    });
    return res.status(401).json({ error: 'Invalid signature' });
  }
  
  // Signature ถูกต้อง
  next();
}

/**
 * Middleware สำหรับเก็บ raw body
 * ต้องใช้ก่อน express.json()
 */
function rawBodyParser(req, res, next) {
  let rawBody = '';
  
  req.on('data', (chunk) => {
    rawBody += chunk.toString();
  });
  
  req.on('end', () => {
    req.rawBody = rawBody;
    
    // Parse JSON manually
    if (req.headers['content-type'] === 'application/json') {
      try {
        req.body = JSON.parse(rawBody);
      } catch (err) {
        return res.status(400).json({ error: 'Invalid JSON' });
      }
    }
    
    next();
  });
  
  req.on('error', (err) => {
    console.error('Error reading request body:', err);
    res.status(500).json({ error: 'Error reading request' });
  });
}

function logSecurityEvent(event, data) {
  console.warn(JSON.stringify({
    type: 'security_event',
    event,
    timestamp: new Date().toISOString(),
    ...data
  }));
}

module.exports = { verifyLineSignature, rawBodyParser };
```

### 2.3 การตั้งค่า Express Application

```javascript
// app.js
const express = require('express');
const { verifyLineSignature, rawBodyParser } = require('./middleware/lineSignatureVerification');

const app = express();

// ต้องใช้ rawBodyParser ก่อน
// ไม่ใช้ express.json() เพราะต้องการ raw body
app.use('/webhook', rawBodyParser);
app.use('/webhook', verifyLineSignature);

app.post('/webhook', (req, res) => {
  const events = req.body.events;
  
  // ประมวลผล events
  events.forEach(event => handleEvent(event));
  
  res.status(200).json({ status: 'ok' });
});

// สำหรับ routes อื่น ๆ ใช้ express.json() ปกติ
app.use(express.json({ limit: '1mb' }));
```

### 2.4 การ Implement ใน Python (FastAPI)

```python
# middleware/line_signature.py
import hashlib
import hmac
import base64
from fastapi import Request, HTTPException
from fastapi.middleware.base import BaseHTTPMiddleware
import logging

logger = logging.getLogger(__name__)

class LineSignatureMiddleware(BaseHTTPMiddleware):
    """Middleware สำหรับตรวจสอบ LINE Webhook Signature"""
    
    def __init__(self, app, channel_secret: str, webhook_path: str = "/webhook"):
        super().__init__(app)
        self.channel_secret = channel_secret.encode('utf-8')
        self.webhook_path = webhook_path
    
    async def dispatch(self, request: Request, call_next):
        # ตรวจสอบเฉพาะ webhook endpoint
        if request.url.path != self.webhook_path:
            return await call_next(request)
        
        # อ่าน raw body
        body = await request.body()
        
        # รับ signature จาก header
        signature = request.headers.get('X-Line-Signature')
        
        if not signature:
            logger.warning(f"Missing signature from {request.client.host}")
            raise HTTPException(status_code=401, detail="Missing signature")
        
        # คำนวณ expected signature
        hash_obj = hmac.new(
            self.channel_secret,
            body,
            hashlib.sha256
        )
        expected_signature = base64.b64encode(hash_obj.digest()).decode('utf-8')
        
        # เปรียบเทียบ signature แบบ constant-time
        if not hmac.compare_digest(signature, expected_signature):
            logger.warning(
                f"Invalid signature from {request.client.host}",
                extra={
                    "event": "invalid_signature",
                    "ip": request.client.host
                }
            )
            raise HTTPException(status_code=401, detail="Invalid signature")
        
        return await call_next(request)


# main.py
from fastapi import FastAPI
from middleware.line_signature import LineSignatureMiddleware
import os

app = FastAPI()

# เพิ่ม middleware
app.add_middleware(
    LineSignatureMiddleware,
    channel_secret=os.environ['LINE_CHANNEL_SECRET'],
    webhook_path="/webhook"
)

@app.post("/webhook")
async def webhook(request: Request):
    body = await request.json()
    events = body.get('events', [])
    
    for event in events:
        await handle_event(event)
    
    return {"status": "ok"}
```

### 2.5 Unit Test สำหรับ Signature Verification

```javascript
// tests/signatureVerification.test.js
const crypto = require('crypto');
const { verifyLineSignature } = require('../middleware/lineSignatureVerification');

describe('LINE Signature Verification', () => {
  const channelSecret = 'test-channel-secret-12345';
  
  beforeEach(() => {
    process.env.LINE_CHANNEL_SECRET = channelSecret;
  });
  
  function createSignature(body, secret) {
    return crypto
      .createHmac('SHA256', secret)
      .update(body)
      .digest('base64');
  }
  
  function mockReqRes(body, signature) {
    const req = {
      rawBody: body,
      ip: '127.0.0.1',
      headers: {
        'x-line-signature': signature,
        'user-agent': 'LineBotWebhook/2.0'
      }
    };
    
    const res = {
      status: jest.fn().mockReturnThis(),
      json: jest.fn().mockReturnThis()
    };
    
    const next = jest.fn();
    
    return { req, res, next };
  }
  
  test('ควรผ่านเมื่อ signature ถูกต้อง', () => {
    const body = JSON.stringify({ events: [] });
    const signature = createSignature(body, channelSecret);
    const { req, res, next } = mockReqRes(body, signature);
    
    verifyLineSignature(req, res, next);
    
    expect(next).toHaveBeenCalled();
    expect(res.status).not.toHaveBeenCalled();
  });
  
  test('ควรปฏิเสธเมื่อ signature ไม่ถูกต้อง', () => {
    const body = JSON.stringify({ events: [] });
    const { req, res, next } = mockReqRes(body, 'invalid-signature');
    
    verifyLineSignature(req, res, next);
    
    expect(next).not.toHaveBeenCalled();
    expect(res.status).toHaveBeenCalledWith(401);
  });
  
  test('ควรปฏิเสธเมื่อไม่มี signature', () => {
    const body = JSON.stringify({ events: [] });
    const req = {
      rawBody: body,
      ip: '127.0.0.1',
      headers: {}
    };
    const res = {
      status: jest.fn().mockReturnThis(),
      json: jest.fn().mockReturnThis()
    };
    const next = jest.fn();
    
    verifyLineSignature(req, res, next);
    
    expect(next).not.toHaveBeenCalled();
    expect(res.status).toHaveBeenCalledWith(401);
  });
  
  test('ควรปฏิเสธเมื่อ body ถูกแก้ไข', () => {
    const originalBody = JSON.stringify({ events: [] });
    const signature = createSignature(originalBody, channelSecret);
    
    // แก้ไข body หลังจากสร้าง signature
    const tamperedBody = JSON.stringify({ events: [], malicious: true });
    const { req, res, next } = mockReqRes(tamperedBody, signature);
    
    verifyLineSignature(req, res, next);
    
    expect(next).not.toHaveBeenCalled();
    expect(res.status).toHaveBeenCalledWith(401);
  });
});
```

---

## 3. Input Validation และ Sanitization

### 3.1 ทำความเข้าใจภัยคุกคาม

```
ประเภทของ Input ที่เป็นอันตราย:
┌────────────────────────────────────────────────────────────┐
│ Input ที่รับจาก User                                        │
│                                                             │
│  Text Messages: "ส่ง SQL injection หรือ script tags"       │
│  Postback Data: "tamper ค่า action/data"                   │
│  LIFF Form Input: "XSS payloads"                           │
│  File Uploads: "malicious files"                           │
│  URL Parameters: "path traversal"                          │
└────────────────────────────────────────────────────────────┘
```

### 3.2 Input Validation Library

```javascript
// utils/inputValidator.js
const validator = require('validator');
const DOMPurify = require('isomorphic-dompurify');

class InputValidator {
  /**
   * Validate และ sanitize text message จาก user
   */
  static validateTextMessage(text) {
    if (typeof text !== 'string') {
      return { valid: false, error: 'ข้อความต้องเป็น string' };
    }
    
    // จำกัดความยาว
    if (text.length > 5000) {
      return { valid: false, error: 'ข้อความยาวเกินไป (max 5000 chars)' };
    }
    
    // ตรวจสอบ null bytes
    if (text.includes('\x00')) {
      return { valid: false, error: 'ข้อความมี null bytes ที่ไม่อนุญาต' };
    }
    
    return {
      valid: true,
      sanitized: text.trim()
    };
  }
  
  /**
   * Validate User ID จาก LINE
   */
  static validateUserId(userId) {
    if (typeof userId !== 'string') {
      return { valid: false, error: 'Invalid user ID format' };
    }
    
    // LINE User IDs มีรูปแบบเฉพาะ
    const lineUserIdPattern = /^U[0-9a-f]{32}$/;
    if (!lineUserIdPattern.test(userId)) {
      return { valid: false, error: 'Invalid LINE User ID format' };
    }
    
    return { valid: true, sanitized: userId };
  }
  
  /**
   * Validate Group/Room ID
   */
  static validateGroupId(groupId) {
    if (typeof groupId !== 'string') {
      return { valid: false, error: 'Invalid group ID format' };
    }
    
    // Group IDs: C + 32 hex chars
    // Room IDs: Ra + 32 hex chars  
    const patterns = [
      /^C[0-9a-f]{32}$/,  // Group
      /^Ra[0-9a-f]{32}$/, // Room
    ];
    
    const isValid = patterns.some(p => p.test(groupId));
    if (!isValid) {
      return { valid: false, error: 'Invalid LINE Group/Room ID format' };
    }
    
    return { valid: true, sanitized: groupId };
  }
  
  /**
   * Validate postback data
   */
  static validatePostbackData(data) {
    if (typeof data !== 'string') {
      return { valid: false, error: 'Postback data must be string' };
    }
    
    if (data.length > 300) {
      return { valid: false, error: 'Postback data too long (max 300 chars)' };
    }
    
    // ตรวจสอบ pattern ตาม schema ที่กำหนด
    // เช่น "action=buy&productId=123"
    try {
      const params = new URLSearchParams(data);
      const sanitized = {};
      
      for (const [key, value] of params) {
        // Sanitize key
        const safeKey = key.replace(/[^a-zA-Z0-9_]/g, '');
        // Sanitize value
        const safeValue = validator.escape(value);
        sanitized[safeKey] = safeValue;
      }
      
      return { valid: true, sanitized };
    } catch (err) {
      return { valid: false, error: 'Invalid postback data format' };
    }
  }
  
  /**
   * Validate และ sanitize HTML (สำหรับ LIFF)
   */
  static sanitizeHtml(html) {
    if (typeof html !== 'string') {
      return { valid: false, error: 'HTML must be string' };
    }
    
    const clean = DOMPurify.sanitize(html, {
      ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p', 'br', 'span'],
      ALLOWED_ATTR: ['class', 'style'],
      FORBID_SCRIPTS: true,
      FORBID_TAGS: ['script', 'iframe', 'object', 'embed'],
      FORBID_ATTR: ['onclick', 'onload', 'onerror', 'onmouseover']
    });
    
    return { valid: true, sanitized: clean };
  }
  
  /**
   * Validate URL
   */
  static validateUrl(url) {
    if (typeof url !== 'string') {
      return { valid: false, error: 'URL must be string' };
    }
    
    // อนุญาตเฉพาะ HTTPS
    if (!validator.isURL(url, {
      protocols: ['https'],
      require_protocol: true,
      require_valid_protocol: true
    })) {
      return { valid: false, error: 'URL ต้องเป็น HTTPS และมีรูปแบบที่ถูกต้อง' };
    }
    
    // ป้องกัน SSRF - block internal IPs
    const blockedPatterns = [
      /^https:\/\/localhost/i,
      /^https:\/\/127\./,
      /^https:\/\/10\./,
      /^https:\/\/172\.(1[6-9]|2\d|3[01])\./,
      /^https:\/\/192\.168\./,
      /^https:\/\/169\.254\./,
      /^https:\/\/0\./
    ];
    
    if (blockedPatterns.some(p => p.test(url))) {
      return { valid: false, error: 'URL ไม่อนุญาต (internal network)' };
    }
    
    return { valid: true, sanitized: url };
  }
  
  /**
   * Validate phone number (ไทย)
   */
  static validateThaiPhoneNumber(phone) {
    const cleaned = phone.replace(/[\s\-\(\)]/g, '');
    
    // รูปแบบโทรศัพท์ไทย: 0X-XXXX-XXXX หรือ +66X-XXXX-XXXX
    const patterns = [
      /^0[0-9]{9}$/,        // 0812345678
      /^\+66[0-9]{9}$/,     // +66812345678
      /^66[0-9]{9}$/        // 66812345678
    ];
    
    if (!patterns.some(p => p.test(cleaned))) {
      return { valid: false, error: 'รูปแบบเบอร์โทรศัพท์ไม่ถูกต้อง' };
    }
    
    return { valid: true, sanitized: cleaned };
  }
  
  /**
   * Validate และ sanitize product/item ID
   */
  static validateItemId(itemId) {
    if (typeof itemId === 'number') {
      itemId = String(itemId);
    }
    
    if (typeof itemId !== 'string') {
      return { valid: false, error: 'Item ID must be string or number' };
    }
    
    // อนุญาตเฉพาะ alphanumeric, dash, underscore
    if (!/^[a-zA-Z0-9\-_]{1,100}$/.test(itemId)) {
      return { valid: false, error: 'Item ID มีตัวอักษรที่ไม่อนุญาต' };
    }
    
    return { valid: true, sanitized: itemId };
  }
}

module.exports = InputValidator;
```

### 3.3 การใช้งาน Validator

```javascript
// handlers/messageHandler.js
const InputValidator = require('../utils/inputValidator');

async function handleTextMessage(event) {
  const { message, source, replyToken } = event;
  
  // Validate user ID
  const userValidation = InputValidator.validateUserId(source.userId);
  if (!userValidation.valid) {
    console.error('Invalid user ID in event:', source.userId);
    return; // ไม่ประมวลผล event นี้
  }
  
  // Validate ข้อความ
  const textValidation = InputValidator.validateTextMessage(message.text);
  if (!textValidation.valid) {
    console.warn('Invalid text message from user:', userValidation.sanitized);
    return replyMessage(replyToken, 'ข้อความไม่ถูกต้อง');
  }
  
  const text = textValidation.sanitized;
  
  // ดำเนินการตาม text
  if (text.startsWith('/')) {
    await handleCommand(replyToken, text, userValidation.sanitized);
  } else {
    await handleNormalMessage(replyToken, text, userValidation.sanitized);
  }
}

async function handlePostback(event) {
  const { postback, source, replyToken } = event;
  
  // Validate postback data
  const dataValidation = InputValidator.validatePostbackData(postback.data);
  if (!dataValidation.valid) {
    console.warn('Invalid postback data:', postback.data);
    return;
  }
  
  const { action, productId, quantity } = dataValidation.sanitized;
  
  // Validate productId ถ้ามี
  if (productId) {
    const productValidation = InputValidator.validateItemId(productId);
    if (!productValidation.valid) {
      console.warn('Invalid product ID in postback:', productId);
      return;
    }
    
    // ใช้ productValidation.sanitized แทน productId โดยตรง
    await handleProductAction(action, productValidation.sanitized, quantity);
  }
}
```

---

## 4. SQL Injection Prevention

### 4.1 ภัยคุกคาม SQL Injection ใน LINE Bot

```javascript
// ❌ อันตราย - SQL Injection vulnerability
async function getUserOrders_UNSAFE(userId) {
  const query = `SELECT * FROM orders WHERE user_id = '${userId}'`;
  // ถ้า userId = "' OR '1'='1" จะได้ข้อมูลทั้งหมด!
  return await db.query(query);
}

// ❌ อันตราย
async function searchProducts_UNSAFE(keyword) {
  const query = `SELECT * FROM products WHERE name LIKE '%${keyword}%'`;
  // ถ้า keyword = "'; DROP TABLE products; --" อันตราย!
  return await db.query(query);
}
```

### 4.2 การป้องกัน SQL Injection

```javascript
// ✅ ปลอดภัย - ใช้ Parameterized Queries
const mysql = require('mysql2/promise');
const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  ssl: {
    rejectUnauthorized: true // ต้องใช้ SSL
  }
});

// ✅ ใช้ ? สำหรับ parameters
async function getUserOrders_SAFE(userId) {
  const [rows] = await pool.execute(
    'SELECT * FROM orders WHERE user_id = ?',
    [userId]
  );
  return rows;
}

// ✅ หลาย parameters
async function searchProducts_SAFE(keyword, categoryId, minPrice, maxPrice) {
  const [rows] = await pool.execute(
    `SELECT * FROM products 
     WHERE name LIKE ? 
     AND category_id = ?
     AND price BETWEEN ? AND ?
     LIMIT 20`,
    [`%${keyword}%`, categoryId, minPrice, maxPrice]
  );
  return rows;
}

// ✅ ใช้ Knex.js (Query Builder)
const knex = require('knex')({
  client: 'mysql2',
  connection: {
    host: process.env.DB_HOST,
    user: process.env.DB_USER,
    password: process.env.DB_PASSWORD,
    database: process.env.DB_NAME
  }
});

async function getOrdersByUser_Knex(userId, status) {
  // Knex จัดการ parameterized query ให้อัตโนมัติ
  const orders = await knex('orders')
    .where({ user_id: userId, status: status })
    .orderBy('created_at', 'desc')
    .limit(10);
  
  return orders;
}

// ✅ ใช้ Sequelize ORM
const { Order, Product } = require('../models');

async function getUserOrdersWithProducts(userId) {
  const orders = await Order.findAll({
    where: { userId }, // Sequelize escape โดยอัตโนมัติ
    include: [{
      model: Product,
      attributes: ['id', 'name', 'price']
    }],
    order: [['createdAt', 'DESC']],
    limit: 10
  });
  
  return orders;
}
```

### 4.3 Database Security Configuration

```javascript
// config/database.js
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize(
  process.env.DB_NAME,
  process.env.DB_USER,
  process.env.DB_PASSWORD,
  {
    host: process.env.DB_HOST,
    dialect: 'mysql',
    
    // Connection Pool
    pool: {
      max: 10,
      min: 2,
      acquire: 30000,
      idle: 10000
    },
    
    // SSL Configuration
    dialectOptions: {
      ssl: {
        require: true,
        rejectUnauthorized: true,
        ca: process.env.DB_SSL_CA
      }
    },
    
    // Logging (ปิดใน production)
    logging: process.env.NODE_ENV === 'development' ? console.log : false,
    
    // Query timeout
    dialectOptions: {
      connectTimeout: 10000,
      requestTimeout: 30000
    }
  }
);

module.exports = sequelize;
```

---

## 5. XSS Prevention ใน LIFF

### 5.1 ภัยคุกคาม XSS ใน LIFF

```javascript
// ❌ อันตราย - XSS vulnerability
function displayUserProfile_UNSAFE(profile) {
  // ถ้า profile.displayName มี <script>alert('XSS')</script>
  document.getElementById('username').innerHTML = profile.displayName;
}

// ❌ อันตราย
function renderMessage_UNSAFE(message) {
  const div = document.createElement('div');
  div.innerHTML = message; // XSS!
  document.body.appendChild(div);
}
```

### 5.2 การป้องกัน XSS

```javascript
// ✅ utils/htmlHelper.js
class HTMLHelper {
  /**
   * Escape HTML characters
   */
  static escapeHtml(str) {
    if (typeof str !== 'string') return '';
    
    const escapeMap = {
      '&': '&amp;',
      '<': '&lt;',
      '>': '&gt;',
      '"': '&quot;',
      "'": '&#x27;',
      '/': '&#x2F;',
      '`': '&#x60;',
      '=': '&#x3D;'
    };
    
    return str.replace(/[&<>"'`=\/]/g, (char) => escapeMap[char]);
  }
  
  /**
   * Set text content อย่างปลอดภัย
   */
  static setTextContent(elementId, text) {
    const element = document.getElementById(elementId);
    if (element) {
      element.textContent = text; // ใช้ textContent ไม่ใช่ innerHTML
    }
  }
  
  /**
   * สร้าง element อย่างปลอดภัย
   */
  static createElement(tag, text, className) {
    const element = document.createElement(tag);
    element.textContent = text; // ปลอดภัย
    if (className) {
      element.className = className; // ปลอดภัย
    }
    return element;
  }
}

// ✅ การแสดงข้อมูล user อย่างปลอดภัย
async function displayUserProfile() {
  const profile = await liff.getProfile();
  
  // ใช้ textContent ไม่ใช่ innerHTML
  document.getElementById('username').textContent = profile.displayName;
  document.getElementById('userId').textContent = profile.userId;
  
  // สำหรับ img src ต้อง validate URL ก่อน
  const imgElement = document.getElementById('avatar');
  if (profile.pictureUrl && isValidImageUrl(profile.pictureUrl)) {
    imgElement.src = profile.pictureUrl;
  }
}

function isValidImageUrl(url) {
  // ต้องมาจาก LINE CDN เท่านั้น
  const allowedDomains = [
    'profile.line-scdn.net',
    'obs.line-scdn.net',
    'sprofile.line-scdn.net'
  ];
  
  try {
    const parsedUrl = new URL(url);
    return parsedUrl.protocol === 'https:' && 
           allowedDomains.some(d => parsedUrl.hostname.endsWith(d));
  } catch {
    return false;
  }
}
```

### 5.3 Content Security Policy (CSP) สำหรับ LIFF

```html
<!-- liff/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  
  <!-- Content Security Policy -->
  <meta http-equiv="Content-Security-Policy" content="
    default-src 'self';
    script-src 'self' https://static.line-scdn.net https://cdn.jsdelivr.net;
    style-src 'self' 'unsafe-inline' https://fonts.googleapis.com;
    img-src 'self' https://*.line-scdn.net data:;
    connect-src 'self' https://api.line.me https://your-api-domain.com;
    font-src 'self' https://fonts.gstatic.com;
    frame-src 'none';
    object-src 'none';
    base-uri 'self';
    form-action 'self';
  ">
  
  <!-- ป้องกัน Clickjacking -->
  <meta http-equiv="X-Frame-Options" content="DENY">
  
  <title>LINE OA App</title>
</head>
<body>
  <div id="app"></div>
  <script src="https://static.line-scdn.net/liff/edge/versions/2.22.3/sdk.js"></script>
  <script src="app.js"></script>
</body>
</html>
```

---

## 6. CSRF Protection

### 6.1 CSRF ใน LIFF Applications

```javascript
// utils/csrfProtection.js
const crypto = require('crypto');

class CSRFProtection {
  /**
   * สร้าง CSRF token
   */
  static generateToken(sessionId) {
    const token = crypto.randomBytes(32).toString('hex');
    
    // เก็บ token ใน session (server-side)
    // หรือใช้ signed token โดยไม่ต้องเก็บ server-side
    const data = {
      token,
      sessionId,
      expires: Date.now() + 3600000 // 1 hour
    };
    
    return token;
  }
  
  /**
   * สร้าง Signed CSRF Token (Stateless)
   */
  static generateSignedToken(userId) {
    const secret = process.env.CSRF_SECRET;
    const timestamp = Date.now();
    const random = crypto.randomBytes(16).toString('hex');
    
    const payload = `${userId}:${timestamp}:${random}`;
    const signature = crypto
      .createHmac('SHA256', secret)
      .update(payload)
      .digest('hex');
    
    const token = Buffer.from(`${payload}:${signature}`).toString('base64');
    return token;
  }
  
  /**
   * Verify Signed CSRF Token
   */
  static verifySignedToken(token, userId, maxAgeMs = 3600000) {
    const secret = process.env.CSRF_SECRET;
    
    try {
      const decoded = Buffer.from(token, 'base64').toString('utf-8');
      const parts = decoded.split(':');
      
      if (parts.length !== 4) return false;
      
      const [tokenUserId, timestamp, random, signature] = parts;
      
      // ตรวจสอบ userId
      if (tokenUserId !== userId) return false;
      
      // ตรวจสอบ expiry
      if (Date.now() - parseInt(timestamp) > maxAgeMs) return false;
      
      // ตรวจสอบ signature
      const payload = `${tokenUserId}:${timestamp}:${random}`;
      const expectedSig = crypto
        .createHmac('SHA256', secret)
        .update(payload)
        .digest('hex');
      
      return crypto.timingSafeEqual(
        Buffer.from(signature, 'hex'),
        Buffer.from(expectedSig, 'hex')
      );
    } catch {
      return false;
    }
  }
}

// Express Middleware
function csrfMiddleware(req, res, next) {
  // ข้าม GET requests
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) {
    return next();
  }
  
  const token = req.headers['x-csrf-token'] || req.body._csrf;
  const userId = req.user?.lineUserId;
  
  if (!token || !userId) {
    return res.status(403).json({ error: 'CSRF token required' });
  }
  
  if (!CSRFProtection.verifySignedToken(token, userId)) {
    return res.status(403).json({ error: 'Invalid CSRF token' });
  }
  
  next();
}

module.exports = { CSRFProtection, csrfMiddleware };
```

```javascript
// LIFF Client-side CSRF handling
// liff/csrfHelper.js

class LIFFCSRFHelper {
  constructor() {
    this.token = null;
  }
  
  /**
   * ดึง CSRF token จาก server
   */
  async fetchCSRFToken(lineUserId) {
    const response = await fetch('/api/csrf-token', {
      headers: {
        'X-Line-User-Id': lineUserId
      }
    });
    
    const data = await response.json();
    this.token = data.token;
    return this.token;
  }
  
  /**
   * ทำ API request พร้อม CSRF token
   */
  async secureRequest(url, options = {}) {
    if (!this.token) {
      throw new Error('CSRF token not initialized');
    }
    
    return fetch(url, {
      ...options,
      headers: {
        ...options.headers,
        'X-CSRF-Token': this.token,
        'Content-Type': 'application/json'
      }
    });
  }
}

// การใช้งาน
const csrfHelper = new LIFFCSRFHelper();

liff.init({ liffId: 'your-liff-id' }).then(async () => {
  const profile = await liff.getProfile();
  await csrfHelper.fetchCSRFToken(profile.userId);
  
  // ทำ request อย่างปลอดภัย
  const response = await csrfHelper.secureRequest('/api/order', {
    method: 'POST',
    body: JSON.stringify({ productId: '123', quantity: 1 })
  });
});
```

---

## 7. Rate Limiting Implementation

### 7.1 Rate Limiting ป้องกัน Abuse

```javascript
// middleware/rateLimiter.js
const redis = require('redis');
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');

const redisClient = redis.createClient({
  host: process.env.REDIS_HOST,
  port: process.env.REDIS_PORT,
  password: process.env.REDIS_PASSWORD,
  tls: process.env.NODE_ENV === 'production'
});

// Rate Limiter สำหรับ Webhook
const webhookLimiter = rateLimit({
  windowMs: 60 * 1000, // 1 minute
  max: 1000, // สูงสุด 1000 requests ต่อ minute (LINE ส่ง batch)
  message: 'Too many requests',
  standardHeaders: true,
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args) => redisClient.sendCommand(args)
  }),
  keyGenerator: (req) => {
    // จำกัดตาม IP
    return req.ip;
  }
});

// Rate Limiter สำหรับ API endpoints (เช่น LIFF API)
const apiLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 60, // 60 requests per minute per user
  store: new RedisStore({
    sendCommand: (...args) => redisClient.sendCommand(args)
  }),
  keyGenerator: (req) => {
    // จำกัดตาม LINE User ID
    return req.user?.lineUserId || req.ip;
  },
  handler: (req, res) => {
    res.status(429).json({
      error: 'Too many requests',
      retryAfter: Math.ceil(req.rateLimit.resetTime / 1000),
      message: 'คุณส่งคำขอมากเกินไป กรุณารอสักครู่แล้วลองใหม่'
    });
  }
});

// Custom Rate Limiter สำหรับ LINE User (ป้องกัน spam)
class UserRateLimiter {
  constructor(redisClient) {
    this.redis = redisClient;
  }
  
  /**
   * ตรวจสอบว่า user ส่งข้อความมากเกินไปหรือไม่
   */
  async checkUserMessageRate(userId) {
    const key = `rate:message:${userId}`;
    const limit = 30; // 30 messages per minute
    const window = 60; // seconds
    
    const current = await this.redis.incr(key);
    
    if (current === 1) {
      await this.redis.expire(key, window);
    }
    
    if (current > limit) {
      const ttl = await this.redis.ttl(key);
      return {
        allowed: false,
        remaining: 0,
        resetIn: ttl,
        message: `กรุณารอ ${ttl} วินาทีแล้วลองใหม่`
      };
    }
    
    return {
      allowed: true,
      remaining: limit - current,
      resetIn: await this.redis.ttl(key)
    };
  }
  
  /**
   * ตรวจสอบ API call rate (เช่น payment processing)
   */
  async checkActionRate(userId, action, limit = 10, windowSecs = 3600) {
    const key = `rate:action:${action}:${userId}`;
    
    const current = await this.redis.incr(key);
    
    if (current === 1) {
      await this.redis.expire(key, windowSecs);
    }
    
    return {
      allowed: current <= limit,
      current,
      limit,
      remaining: Math.max(0, limit - current)
    };
  }
}

module.exports = {
  webhookLimiter,
  apiLimiter,
  UserRateLimiter
};
```

### 7.2 การใช้ Rate Limiting ใน Bot

```javascript
// handlers/messageHandler.js
const { UserRateLimiter } = require('../middleware/rateLimiter');
const rateLimiter = new UserRateLimiter(redisClient);

async function handleEvent(event) {
  const userId = event.source.userId;
  
  // ตรวจสอบ message rate
  const rateCheck = await rateLimiter.checkUserMessageRate(userId);
  
  if (!rateCheck.allowed) {
    // ส่งแจ้งเตือนเฉพาะครั้งแรกที่ถูก block
    const notifyKey = `notified:rate:${userId}`;
    const wasNotified = await redisClient.get(notifyKey);
    
    if (!wasNotified) {
      await lineClient.replyMessage(event.replyToken, {
        type: 'text',
        text: `⚠️ ${rateCheck.message}`
      });
      await redisClient.setex(notifyKey, 60, '1');
    }
    
    return;
  }
  
  // ประมวลผลปกติ
  await processEvent(event);
}

// Middleware สำหรับ payment action
async function handlePayment(event) {
  const userId = event.source.userId;
  
  const paymentRateCheck = await rateLimiter.checkActionRate(
    userId,
    'payment',
    5,    // 5 transactions
    3600  // per hour
  );
  
  if (!paymentRateCheck.allowed) {
    return lineClient.replyMessage(event.replyToken, {
      type: 'text',
      text: '⚠️ คุณทำรายการชำระเงินมากเกินไป กรุณารอ 1 ชั่วโมงแล้วลองใหม่'
    });
  }
  
  await processPayment(event);
}
```

---

## 8. Token Management Security

### 8.1 LINE Access Token Management

```javascript
// services/tokenManager.js
const crypto = require('crypto');

class TokenManager {
  constructor(redisClient) {
    this.redis = redisClient;
    this.encryptionKey = Buffer.from(process.env.TOKEN_ENCRYPTION_KEY, 'hex');
  }
  
  /**
   * เข้ารหัส token ก่อนเก็บ
   */
  encryptToken(token) {
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv('aes-256-gcm', this.encryptionKey, iv);
    
    let encrypted = cipher.update(token, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const authTag = cipher.getAuthTag();
    
    return {
      iv: iv.toString('hex'),
      encrypted,
      authTag: authTag.toString('hex')
    };
  }
  
  /**
   * ถอดรหัส token
   */
  decryptToken(encryptedData) {
    const decipher = crypto.createDecipheriv(
      'aes-256-gcm',
      this.encryptionKey,
      Buffer.from(encryptedData.iv, 'hex')
    );
    
    decipher.setAuthTag(Buffer.from(encryptedData.authTag, 'hex'));
    
    let decrypted = decipher.update(encryptedData.encrypted, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }
  
  /**
   * เก็บ user token อย่างปลอดภัย
   */
  async storeUserToken(userId, accessToken, expiresIn) {
    const encrypted = this.encryptToken(accessToken);
    const key = `token:user:${userId}`;
    
    await this.redis.setex(
      key,
      expiresIn - 60, // หมดอายุก่อน 1 นาที
      JSON.stringify(encrypted)
    );
  }
  
  /**
   * ดึง user token
   */
  async getUserToken(userId) {
    const key = `token:user:${userId}`;
    const stored = await this.redis.get(key);
    
    if (!stored) return null;
    
    try {
      const encrypted = JSON.parse(stored);
      return this.decryptToken(encrypted);
    } catch (err) {
      console.error('Failed to decrypt token:', err);
      await this.redis.del(key); // ลบ token ที่ corrupt
      return null;
    }
  }
  
  /**
   * Revoke token
   */
  async revokeUserToken(userId) {
    const key = `token:user:${userId}`;
    await this.redis.del(key);
  }
  
  /**
   * Rotate Channel Access Token
   * เปลี่ยน token ทุก X วัน
   */
  async rotateChannelToken() {
    const { default: axios } = await import('axios');
    
    // Issue new token
    const response = await axios.post(
      'https://api.line.me/oauth/accessToken',
      new URLSearchParams({
        grant_type: 'client_credentials',
        client_id: process.env.LINE_CHANNEL_ID,
        client_secret: process.env.LINE_CHANNEL_SECRET
      }).toString(),
      {
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded'
        }
      }
    );
    
    const newToken = response.data.access_token;
    const expiresIn = response.data.expires_in;
    
    // เก็บ token ใหม่
    await this.storeUserToken('channel', newToken, expiresIn);
    
    console.log(`Channel token rotated, expires in ${expiresIn}s`);
    return newToken;
  }
}

module.exports = TokenManager;
```

---

## 9. Secrets Management

### 9.1 Environment Variables Best Practices

```bash
# .env.example (commit นี้ได้ - ไม่มีค่าจริง)
NODE_ENV=production
PORT=3000

# LINE API
LINE_CHANNEL_SECRET=your-channel-secret-here
LINE_CHANNEL_ACCESS_TOKEN=your-access-token-here

# Database
DB_HOST=localhost
DB_PORT=3306
DB_NAME=linebot_db
DB_USER=linebot_user
DB_PASSWORD=your-strong-password-here
DB_SSL_CA=/path/to/ca-certificate.pem

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your-redis-password-here

# Security
JWT_SECRET=your-jwt-secret-32-chars-min
CSRF_SECRET=your-csrf-secret-32-chars-min
TOKEN_ENCRYPTION_KEY=your-32-byte-hex-key-here

# Monitoring
SENTRY_DSN=your-sentry-dsn-here
```

```javascript
// config/secrets.js
const fs = require('fs');

class SecretsManager {
  /**
   * โหลด secrets จาก environment variables
   * พร้อม validation
   */
  static loadAndValidate() {
    const required = [
      'LINE_CHANNEL_SECRET',
      'LINE_CHANNEL_ACCESS_TOKEN',
      'DB_HOST',
      'DB_USER',
      'DB_PASSWORD',
      'JWT_SECRET',
      'CSRF_SECRET'
    ];
    
    const missing = required.filter(key => !process.env[key]);
    
    if (missing.length > 0) {
      throw new Error(
        `Missing required environment variables: ${missing.join(', ')}`
      );
    }
    
    // ตรวจสอบความปลอดภัยของ secrets
    this.validateSecretStrength();
    
    return {
      line: {
        channelSecret: process.env.LINE_CHANNEL_SECRET,
        channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
      },
      database: {
        host: process.env.DB_HOST,
        port: parseInt(process.env.DB_PORT) || 3306,
        name: process.env.DB_NAME,
        user: process.env.DB_USER,
        password: process.env.DB_PASSWORD
      },
      jwt: {
        secret: process.env.JWT_SECRET,
        expiresIn: process.env.JWT_EXPIRES_IN || '24h'
      },
      csrf: {
        secret: process.env.CSRF_SECRET
      }
    };
  }
  
  static validateSecretStrength() {
    const jwtSecret = process.env.JWT_SECRET;
    if (jwtSecret && jwtSecret.length < 32) {
      throw new Error('JWT_SECRET must be at least 32 characters');
    }
    
    const dbPassword = process.env.DB_PASSWORD;
    if (dbPassword && dbPassword.length < 12) {
      console.warn('WARNING: DB_PASSWORD is less than 12 characters');
    }
  }
  
  /**
   * โหลด secrets จาก HashiCorp Vault
   */
  static async loadFromVault() {
    const vault = require('node-vault')({
      apiVersion: 'v1',
      endpoint: process.env.VAULT_ADDR,
      token: process.env.VAULT_TOKEN
    });
    
    try {
      const secrets = await vault.read('secret/data/linebot/production');
      const data = secrets.data.data;
      
      // ตั้งค่า environment variables
      Object.assign(process.env, {
        LINE_CHANNEL_SECRET: data.line_channel_secret,
        LINE_CHANNEL_ACCESS_TOKEN: data.line_channel_access_token,
        DB_PASSWORD: data.db_password,
        JWT_SECRET: data.jwt_secret
      });
      
      console.log('Secrets loaded from Vault successfully');
    } catch (err) {
      console.error('Failed to load secrets from Vault:', err);
      throw err;
    }
  }
  
  /**
   * โหลด secrets จาก AWS Secrets Manager
   */
  static async loadFromAWSSecretsManager() {
    const {
      SecretsManagerClient,
      GetSecretValueCommand
    } = require('@aws-sdk/client-secrets-manager');
    
    const client = new SecretsManagerClient({
      region: process.env.AWS_REGION || 'ap-southeast-1'
    });
    
    const secretName = process.env.AWS_SECRET_NAME || 'linebot/production';
    
    const response = await client.send(
      new GetSecretValueCommand({ SecretId: secretName })
    );
    
    const secrets = JSON.parse(response.SecretString);
    Object.assign(process.env, secrets);
    
    console.log('Secrets loaded from AWS Secrets Manager');
  }
}

module.exports = SecretsManager;
```

### 9.2 .gitignore สำหรับ Secrets

```gitignore
# .gitignore
# Environment variables
.env
.env.local
.env.development
.env.production
.env.staging

# SSL/TLS certificates
*.pem
*.key
*.crt
*.cer

# Credentials
credentials.json
*-credentials.json
service-account*.json

# Vault tokens
.vault-token
vault-token

# AWS credentials
.aws/credentials

# Secrets directory
secrets/
private/
```

---

## 10. DDoS Protection

### 10.1 Application-level DDoS Protection

```javascript
// middleware/ddosProtection.js
const Redis = require('ioredis');

class DDoSProtection {
  constructor(redisClient) {
    this.redis = redisClient;
    this.config = {
      // Sliding window rate limits
      requestsPerSecond: 50,
      requestsPerMinute: 1000,
      requestsPerHour: 10000,
      
      // Burst protection
      burstLimit: 100,
      burstWindow: 5, // seconds
      
      // Auto-ban threshold
      banThreshold: 5000, // requests per 5 min
      banDuration: 3600   // ban for 1 hour
    };
  }
  
  /**
   * ตรวจสอบและป้องกัน DDoS
   */
  async checkRequest(ip) {
    // ตรวจสอบ blacklist ก่อน
    const isBanned = await this.redis.get(`ban:${ip}`);
    if (isBanned) {
      return {
        allowed: false,
        reason: 'banned',
        ttl: await this.redis.ttl(`ban:${ip}`)
      };
    }
    
    const now = Date.now();
    const pipe = this.redis.pipeline();
    
    // เพิ่ม request count ใน sliding windows
    const windows = [
      { key: `req:1s:${ip}:${Math.floor(now / 1000)}`, ttl: 2 },
      { key: `req:1m:${ip}:${Math.floor(now / 60000)}`, ttl: 120 },
      { key: `req:1h:${ip}:${Math.floor(now / 3600000)}`, ttl: 7200 }
    ];
    
    for (const w of windows) {
      pipe.incr(w.key);
      pipe.expire(w.key, w.ttl);
    }
    
    const results = await pipe.exec();
    const [perSecond, , perMinute, , perHour] = results.map(r => r[1]);
    
    // ตรวจสอบ limits
    if (perSecond > this.config.requestsPerSecond) {
      await this.recordViolation(ip, 'per_second');
      return { allowed: false, reason: 'rate_limit_second' };
    }
    
    if (perMinute > this.config.requestsPerMinute) {
      await this.recordViolation(ip, 'per_minute');
      return { allowed: false, reason: 'rate_limit_minute' };
    }
    
    if (perHour > this.config.requestsPerHour) {
      await this.recordViolation(ip, 'per_hour');
      return { allowed: false, reason: 'rate_limit_hour' };
    }
    
    return { allowed: true };
  }
  
  /**
   * บันทึก violation และ auto-ban ถ้าจำเป็น
   */
  async recordViolation(ip, type) {
    const key = `violations:${ip}`;
    const count = await this.redis.incr(key);
    await this.redis.expire(key, 300); // 5 minute window
    
    console.warn(`DDoS violation from ${ip}: ${type} (count: ${count})`);
    
    // Auto-ban ถ้าละเมิดมากเกินไป
    if (count >= 10) {
      await this.banIP(ip, this.config.banDuration);
    }
  }
  
  /**
   * Ban IP
   */
  async banIP(ip, duration) {
    await this.redis.setex(`ban:${ip}`, duration, '1');
    
    console.error(`IP ${ip} has been banned for ${duration} seconds`);
    
    // แจ้งเตือน (webhook, Slack, etc.)
    await this.sendAlert({
      type: 'ip_banned',
      ip,
      duration,
      timestamp: new Date().toISOString()
    });
  }
  
  /**
   * Unban IP (สำหรับ admin)
   */
  async unbanIP(ip) {
    await this.redis.del(`ban:${ip}`);
    await this.redis.del(`violations:${ip}`);
    console.log(`IP ${ip} has been unbanned`);
  }
  
  async sendAlert(data) {
    // ส่งแจ้งเตือนไป monitoring system
    // เช่น Slack, PagerDuty, etc.
    console.log('Security Alert:', JSON.stringify(data));
  }
}

// Express Middleware
function createDDoSMiddleware(redisClient) {
  const protection = new DDoSProtection(redisClient);
  
  return async (req, res, next) => {
    const ip = req.ip || req.connection.remoteAddress;
    
    const result = await protection.checkRequest(ip);
    
    if (!result.allowed) {
      return res.status(429).json({
        error: 'Too Many Requests',
        reason: result.reason,
        ...(result.ttl && { retryAfter: result.ttl })
      });
    }
    
    next();
  };
}

module.exports = { DDoSProtection, createDDoSMiddleware };
```

---

## 11. HTTPS Enforcement

### 11.1 HTTPS Configuration

```javascript
// middleware/httpsEnforcement.js

/**
 * บังคับ HTTPS
 */
function enforceHTTPS(req, res, next) {
  // ตรวจสอบจาก header (สำหรับ reverse proxy)
  const proto = req.headers['x-forwarded-proto'];
  const host = req.headers['x-forwarded-host'] || req.headers.host;
  
  if (proto === 'http' || (!proto && !req.secure)) {
    // Redirect to HTTPS
    return res.redirect(301, `https://${host}${req.url}`);
  }
  
  next();
}

/**
 * HSTS (HTTP Strict Transport Security)
 */
function setHSTS(req, res, next) {
  res.setHeader(
    'Strict-Transport-Security',
    'max-age=31536000; includeSubDomains; preload'
  );
  next();
}

module.exports = { enforceHTTPS, setHSTS };
```

```nginx
# nginx/nginx.conf
server {
    listen 80;
    server_name your-bot-domain.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name your-bot-domain.com;
    
    # SSL Certificate (Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/your-bot-domain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-bot-domain.com/privkey.pem;
    
    # SSL Configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
    
    # Security Headers
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=webhook:10m rate=100r/s;
    
    location /webhook {
        limit_req zone=webhook burst=200 nodelay;
        proxy_pass http://app:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Body size limit
        client_max_body_size 1m;
    }
    
    location / {
        proxy_pass http://app:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 12. Security Headers

### 12.1 Helmet.js Configuration

```javascript
// config/security.js
const helmet = require('helmet');

function configureSecurityHeaders(app) {
  app.use(helmet({
    // Content Security Policy
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "https://static.line-scdn.net"],
        styleSrc: ["'self'", "'unsafe-inline'", "https://fonts.googleapis.com"],
        imgSrc: ["'self'", "https://*.line-scdn.net", "data:"],
        connectSrc: ["'self'", "https://api.line.me"],
        fontSrc: ["'self'", "https://fonts.gstatic.com"],
        objectSrc: ["'none'"],
        mediaSrc: ["'none'"],
        frameSrc: ["'none'"],
        frameAncestors: ["'none'"],
        formAction: ["'self'"],
        upgradeInsecureRequests: []
      }
    },
    
    // HSTS
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true
    },
    
    // Prevent clickjacking
    frameguard: { action: 'deny' },
    
    // Prevent MIME sniffing
    noSniff: true,
    
    // XSS filter
    xssFilter: true,
    
    // Referrer Policy
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
    
    // Hide X-Powered-By
    hidePoweredBy: true
  }));
  
  // Additional custom headers
  app.use((req, res, next) => {
    // Cache control สำหรับ API
    res.setHeader('Cache-Control', 'no-store, no-cache, must-revalidate, private');
    
    // Permissions Policy
    res.setHeader(
      'Permissions-Policy',
      'geolocation=(), microphone=(), camera=(), payment=()'
    );
    
    next();
  });
}

module.exports = configureSecurityHeaders;
```

---

## 13. Dependency Vulnerability Scanning

### 13.1 การตั้งค่า Automated Scanning

```json
// package.json scripts
{
  "scripts": {
    "security:audit": "npm audit --production",
    "security:audit:fix": "npm audit fix",
    "security:check": "npm run security:audit && npx snyk test",
    "security:monitor": "npx snyk monitor"
  }
}
```

```yaml
# .github/workflows/security-scan.yml
name: Security Scan

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 9 * * 1'  # Every Monday 9 AM

jobs:
  security:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run npm audit
        run: npm audit --production --audit-level=high
      
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high
      
      - name: Run OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'LINE Bot'
          path: '.'
          format: 'HTML'
          args: '--failOnCVSS 7'
      
      - name: Upload scan results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: security-reports
          path: reports/
```

### 13.2 Dependabot Configuration

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
      day: "monday"
      time: "09:00"
      timezone: "Asia/Bangkok"
    
    # จำกัดจำนวน PR
    open-pull-requests-limit: 10
    
    # Auto-merge สำหรับ patch updates
    versioning-strategy: lockfile-only
    
    labels:
      - "dependencies"
      - "security"
    
    ignore:
      # ไม่อัปเดต major versions โดยอัตโนมัติ
      - dependency-name: "*"
        update-types: ["version-update:semver-major"]
    
    commit-message:
      prefix: "chore"
      include: "scope"
```

---

## 14. Penetration Testing Guide

### 14.1 Scope และ Methodology

```
Penetration Testing Scope สำหรับ LINE Bot:
┌───────────────────────────────────────────────────────────────┐
│                    Testing Areas                               │
│                                                                │
│  1. Webhook Endpoint                                           │
│     - Signature bypass attempts                               │
│     - Replay attacks                                          │
│     - Malformed payloads                                      │
│                                                                │
│  2. LIFF Application                                          │
│     - XSS vulnerabilities                                     │
│     - CSRF attacks                                            │
│     - Authentication bypass                                   │
│                                                                │
│  3. API Endpoints                                             │
│     - Authorization testing                                   │
│     - Rate limit bypass                                       │
│     - Business logic flaws                                    │
│                                                                │
│  4. Infrastructure                                            │
│     - Open ports                                              │
│     - SSL/TLS configuration                                   │
│     - Admin panel exposure                                    │
└───────────────────────────────────────────────────────────────┘
```

```bash
# pentest/webhook-tests.sh
#!/bin/bash

# ========================
# Webhook Security Testing
# ========================

WEBHOOK_URL="https://your-bot-domain.com/webhook"
CHANNEL_SECRET="your-channel-secret"

echo "=== Testing Webhook Security ==="

# Test 1: ทดสอบ request ที่ไม่มี signature
echo "[+] Test 1: Missing signature"
curl -s -o /dev/null -w "%{http_code}" \
  -X POST "$WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{"events":[]}' \
  | xargs echo "Status:"

# Test 2: ทดสอบ invalid signature
echo "[+] Test 2: Invalid signature"
curl -s -o /dev/null -w "%{http_code}" \
  -X POST "$WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -H "X-Line-Signature: aW52YWxpZA==" \
  -d '{"events":[]}' \
  | xargs echo "Status:"

# Test 3: ทดสอบ replay attack (ส่ง signature เก่า)
echo "[+] Test 3: Replay attack simulation"
BODY='{"events":[]}'
VALID_SIG=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$CHANNEL_SECRET" -binary | base64)
sleep 2  # รอแล้วส่งซ้ำ
curl -s -o /dev/null -w "%{http_code}" \
  -X POST "$WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -H "X-Line-Signature: $VALID_SIG" \
  -d "$BODY" \
  | xargs echo "Status:"

# Test 4: SQL Injection ใน postback data
echo "[+] Test 4: SQL Injection in postback"
MALICIOUS_DATA='action=search&q=1'"'"' OR '"'"'1'"'"'='"'"'1'
BODY="{\"events\":[{\"type\":\"postback\",\"postback\":{\"data\":\"$MALICIOUS_DATA\"}}]}"
SIG=$(echo -n "$BODY" | openssl dgst -sha256 -hmac "$CHANNEL_SECRET" -binary | base64)
curl -s -o /dev/null -w "%{http_code}" \
  -X POST "$WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -H "X-Line-Signature: $SIG" \
  -d "$BODY" \
  | xargs echo "Status:"

echo "=== Testing Complete ==="
```

---

## 15. Incident Response Plan

### 15.1 Security Incident Response Procedure

```markdown
# LINE Bot Security Incident Response Plan

## ระดับความรุนแรง (Severity Levels)

### Critical (P0) - ตอบสนองภายใน 15 นาที
- Data breach ที่มีข้อมูล PII รั่วไหล
- Bot ถูกเข้าถึงโดยไม่ได้รับอนุญาต
- Channel Access Token ถูก compromise
- ระบบ payment ถูกโจมตี

### High (P1) - ตอบสนองภายใน 1 ชั่วโมง
- API quota ถูกใช้งานผิดปกติ
- DDoS attack ที่ทำให้ service ล่ม
- Unauthorized access พยายามเข้าระบบ

### Medium (P2) - ตอบสนองภายใน 4 ชั่วโมง
- Rate limiting triggered บ่อยผิดปกติ
- Dependency vulnerability ค้นพบ
- Error rate สูงผิดปกติ

### Low (P3) - ตอบสนองภายใน 24 ชั่วโมง
- Security scan พบ warning
- Suspicious patterns ที่ไม่รุนแรง
```

```javascript
// services/incidentResponse.js
class IncidentResponse {
  constructor(lineClient, alertWebhookUrl) {
    this.lineClient = lineClient;
    this.alertWebhookUrl = alertWebhookUrl;
    this.adminGroupId = process.env.ADMIN_GROUP_ID;
  }
  
  /**
   * ตรวจจับและตอบสนองต่อ incidents
   */
  async handleSecurityEvent(event) {
    const { type, severity, details } = event;
    
    // Log event
    console.error(JSON.stringify({
      type: 'security_incident',
      severity,
      event: type,
      timestamp: new Date().toISOString(),
      ...details
    }));
    
    // ตอบสนองตาม severity
    switch (severity) {
      case 'critical':
        await this.handleCritical(event);
        break;
      case 'high':
        await this.handleHigh(event);
        break;
      case 'medium':
        await this.handleMedium(event);
        break;
      default:
        await this.logEvent(event);
    }
  }
  
  async handleCritical(event) {
    // 1. Immediate containment
    await this.containThreat(event);
    
    // 2. Alert on-call team
    await this.alertTeam(event, '🚨 CRITICAL SECURITY INCIDENT');
    
    // 3. Create incident ticket
    await this.createIncidentTicket(event);
    
    // 4. Begin evidence collection
    await this.collectEvidence(event);
  }
  
  async containThreat(event) {
    const { details } = event;
    
    if (details.sourceIP) {
      // Block IP ทันที
      await redisClient.setex(`ban:${details.sourceIP}`, 86400, 'security_incident');
    }
    
    if (details.compromisedToken) {
      // Revoke token ทันที
      await revokeToken(details.compromisedToken);
      await issueNewToken();
    }
    
    if (details.affectedUserId) {
      // ล็อค user account ชั่วคราว
      await lockUserAccount(details.affectedUserId);
    }
  }
  
  async alertTeam(event, urgency) {
    const message = `
${urgency}
━━━━━━━━━━━━━━━━━━
Type: ${event.type}
Severity: ${event.severity.toUpperCase()}
Time: ${new Date().toISOString()}
Details: ${JSON.stringify(event.details, null, 2)}
━━━━━━━━━━━━━━━━━━
Action Required: ${event.suggestedAction || 'Investigate immediately'}
    `.trim();
    
    // ส่งไปยัง Admin Group ใน LINE
    if (this.adminGroupId) {
      await this.lineClient.pushMessage(this.adminGroupId, {
        type: 'text',
        text: message
      });
    }
    
    // ส่งไปยัง webhook (Slack, PagerDuty, etc.)
    if (this.alertWebhookUrl) {
      await fetch(this.alertWebhookUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          text: message,
          urgency: event.severity,
          timestamp: Date.now()
        })
      });
    }
  }
  
  /**
   * Post-incident review checklist
   */
  generatePostIncidentReport(incident) {
    return {
      timeline: incident.timeline,
      rootCause: null, // กรอกหลังการสืบสวน
      impact: {
        usersAffected: incident.usersAffected || 0,
        dataCompromised: incident.dataCompromised || [],
        serviceDuration: incident.downtime || 0
      },
      containmentActions: incident.actions,
      lessonsLearned: [],
      preventionMeasures: [],
      followUpItems: []
    };
  }
}

module.exports = IncidentResponse;
```

---

## 16. Security Testing ครบวงจร

### 16.1 Automated Security Test Suite

```javascript
// tests/security/securitySuite.test.js
const request = require('supertest');
const crypto = require('crypto');
const app = require('../../app');

describe('Security Test Suite', () => {
  const channelSecret = process.env.LINE_CHANNEL_SECRET;
  
  function createValidSignature(body) {
    return crypto
      .createHmac('SHA256', channelSecret)
      .update(typeof body === 'string' ? body : JSON.stringify(body))
      .digest('base64');
  }
  
  describe('Webhook Security', () => {
    test('ปฏิเสธ request ที่ไม่มี signature', async () => {
      const res = await request(app)
        .post('/webhook')
        .send({ events: [] });
      
      expect(res.status).toBe(401);
    });
    
    test('ปฏิเสธ request ที่มี invalid signature', async () => {
      const res = await request(app)
        .post('/webhook')
        .set('X-Line-Signature', 'invalidsig')
        .send({ events: [] });
      
      expect(res.status).toBe(401);
    });
    
    test('ยอมรับ request ที่มี valid signature', async () => {
      const body = JSON.stringify({ events: [] });
      const sig = createValidSignature(body);
      
      const res = await request(app)
        .post('/webhook')
        .set('X-Line-Signature', sig)
        .set('Content-Type', 'application/json')
        .send(body);
      
      expect(res.status).toBe(200);
    });
    
    test('ปฏิเสธ request ที่ body ขนาดใหญ่เกินไป', async () => {
      const largeBody = 'x'.repeat(10 * 1024 * 1024); // 10MB
      const sig = createValidSignature(largeBody);
      
      const res = await request(app)
        .post('/webhook')
        .set('X-Line-Signature', sig)
        .send(largeBody);
      
      expect(res.status).toBe(413);
    });
  });
  
  describe('Security Headers', () => {
    test('มี X-Frame-Options: DENY', async () => {
      const res = await request(app).get('/');
      expect(res.headers['x-frame-options']).toBe('DENY');
    });
    
    test('มี X-Content-Type-Options: nosniff', async () => {
      const res = await request(app).get('/');
      expect(res.headers['x-content-type-options']).toBe('nosniff');
    });
    
    test('มี Content-Security-Policy', async () => {
      const res = await request(app).get('/');
      expect(res.headers['content-security-policy']).toBeDefined();
    });
    
    test('ไม่มี X-Powered-By header', async () => {
      const res = await request(app).get('/');
      expect(res.headers['x-powered-by']).toBeUndefined();
    });
  });
  
  describe('Rate Limiting', () => {
    test('จำกัด request ที่มากเกินไป', async () => {
      const requests = [];
      
      for (let i = 0; i < 110; i++) {
        const body = JSON.stringify({ events: [] });
        const sig = createValidSignature(body);
        
        requests.push(
          request(app)
            .post('/webhook')
            .set('X-Line-Signature', sig)
            .send(body)
        );
      }
      
      const responses = await Promise.all(requests);
      const rateLimited = responses.filter(r => r.status === 429);
      
      expect(rateLimited.length).toBeGreaterThan(0);
    });
  });
  
  describe('Input Validation', () => {
    test('sanitize SQL injection attempts', async () => {
      const maliciousPayload = {
        events: [{
          type: 'postback',
          source: { type: 'user', userId: 'U1234567890abcdef1234567890abcdef' },
          postback: {
            data: "action=search&q=1' OR '1'='1"
          }
        }]
      };
      
      const body = JSON.stringify(maliciousPayload);
      const sig = createValidSignature(body);
      
      // ไม่ควร throw error หรือ return 500
      const res = await request(app)
        .post('/webhook')
        .set('X-Line-Signature', sig)
        .send(body);
      
      expect(res.status).not.toBe(500);
    });
  });
});
```

---

## สรุปบทที่ 76

### Key Takeaways

```
Security Best Practices สำหรับ LINE Bot:
┌─────────────────────────────────────────────────────────┐
│                                                          │
│  ✅ ตรวจสอบ Webhook Signature ทุก request               │
│  ✅ ใช้ Parameterized Queries ป้องกัน SQL Injection      │
│  ✅ Sanitize HTML output ป้องกัน XSS                    │
│  ✅ ใช้ CSRF tokens สำหรับ state-changing actions       │
│  ✅ Rate limit ทุก endpoint                              │
│  ✅ เก็บ secrets ใน environment variables               │
│  ✅ ใช้ HTTPS เท่านั้น                                  │
│  ✅ ตั้งค่า Security Headers ด้วย Helmet.js             │
│  ✅ Scan dependencies สม่ำเสมอ                          │
│  ✅ มี Incident Response Plan พร้อมใช้งาน               │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Security Audit Commands

```bash
# ตรวจสอบ dependencies
npm audit --production
npx snyk test

# ตรวจสอบ SSL configuration
openssl s_client -connect your-bot-domain.com:443 -tls1_2

# ตรวจสอบ security headers
curl -I https://your-bot-domain.com/webhook

# ตรวจสอบ rate limiting
for i in $(seq 1 20); do
  curl -o /dev/null -s -w "%{http_code}\n" https://your-bot-domain.com/webhook
done
```

---

## แหล่งข้อมูลเพิ่มเติม

- [LINE Bot SDK Security](https://developers.line.biz/en/reference/messaging-api/)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [Node.js Security Best Practices](https://nodejs.org/en/docs/guides/security/)
- [Helmet.js Documentation](https://helmetjs.github.io/)
- [Express Rate Limit](https://express-rate-limit.mintlify.app/)

**ต่อไป**: [Part 77 - PDPA Compliance](./part-077-pdpa-compliance.md)
