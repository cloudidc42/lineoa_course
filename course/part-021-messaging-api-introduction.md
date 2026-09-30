# Part 21: LINE Messaging API เบื้องต้น

## สารบัญ

1. [LINE Messaging API คืออะไร?](#line-messaging-api-คืออะไร)
2. [ความแตกต่างระหว่าง LINE OA Manager และ Messaging API](#ความแตกต่าง)
3. [Developer Console Overview](#developer-console-overview)
4. [การสร้าง Channel](#การสร้าง-channel)
5. [Channel Access Token vs Long-lived Token](#channel-access-token)
6. [Channel Secret](#channel-secret)
7. [Webhook URL Setup](#webhook-url-setup)
8. [การทดสอบด้วย cURL](#การทดสอบดวย-curl)
9. [Rate Limits](#rate-limits)
10. [API Versioning](#api-versioning)
11. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. LINE Messaging API คืออะไร?

LINE Messaging API เป็น API ที่ LINE Corporation จัดให้นักพัฒนาสามารถสร้างแชทบอทและบริการอัตโนมัติบน LINE Platform ได้ โดยสามารถ:

- **รับข้อความ** จากผู้ใช้ผ่าน Webhook
- **ส่งข้อความ** ตอบกลับหรือ Push Message
- **จัดการผู้ติดตาม** (Follower Management)
- **ส่ง Rich Content** เช่น Flex Message, Carousel, Quick Reply
- **เชื่อมต่อกับระบบภายนอก** เช่น CRM, Database, AI

### สถาปัตยกรรมโดยรวม

```
ผู้ใช้ LINE ───► LINE Platform ───► Webhook Server (เซิร์ฟเวอร์คุณ)
                      │                      │
                      │◄─────────────────────│
                      │    Reply API / Push API
                      │
                ผู้ใช้ LINE
```

### Use Cases ที่พบบ่อย

1. **Customer Service Bot** - ตอบคำถามอัตโนมัติ 24/7
2. **E-Commerce Bot** - แสดงสินค้า รับออเดอร์ ตรวจสอบสถานะ
3. **Notification Service** - แจ้งเตือนข่าวสาร โปรโมชัน
4. **Appointment Booking** - นัดหมาย จองบริการ
5. **Survey & Feedback** - สำรวจความคิดเห็น
6. **Internal Tools** - แจ้งเตือนภายในองค์กร

---

## 2. ความแตกต่างระหว่าง LINE OA Manager และ Messaging API

| Feature | LINE OA Manager | Messaging API |
|---------|----------------|---------------|
| การตั้งค่า | ผ่าน Web UI | ผ่าน Code/API |
| Auto Reply | มีใน UI | ต้องเขียนโค้ด |
| Webhook | เปิด/ปิดได้ | จำเป็นต้องมี |
| Rich Menu | ผ่าน UI | ผ่าน API |
| Broadcast | ผ่าน UI | ผ่าน API |
| Analytics | มีใน Dashboard | ต้องดึงข้อมูลเอง |
| ความยืดหยุ่น | จำกัด | สูงมาก |
| ต้องเขียนโค้ด | ไม่ต้อง | ต้องใช้ |

### เมื่อไหรควรใช้ Messaging API?

**ใช้ LINE OA Manager เมื่อ:**
- ต้องการตั้งค่าง่ายๆ โดยไม่เขียนโค้ด
- ต้องการ Broadcast ง่ายๆ
- ร้านค้าขนาดเล็กที่ไม่มี Developer

**ใช้ Messaging API เมื่อ:**
- ต้องการ Logic ซับซ้อน เช่น เชื่อมต่อ Database
- ต้องการ Dynamic Content ที่เปลี่ยนตามผู้ใช้
- ต้องการ Integration กับระบบอื่น
- ต้องการ Custom Business Logic

---

## 3. Developer Console Overview

### การเข้าถึง LINE Developers Console

1. ไปที่ **https://developers.line.biz/**
2. ล็อกอินด้วยบัญชี LINE
3. สร้าง **Provider** (ชื่อองค์กรหรือบริษัทของคุณ)

### โครงสร้างของ LINE Developers Console

```
LINE Developers Console
│
├── Providers (ผู้ให้บริการ)
│   ├── My Company
│   │   ├── Channels
│   │   │   ├── LINE Login Channel
│   │   │   ├── Messaging API Channel ← ที่เราสนใจ
│   │   │   └── CLOVA Skill
│   │   └── Settings
│   └── Another Provider
│
└── Account Settings
```

### ส่วนประกอบใน Channel Settings

```
Channel Settings
├── Basic Settings
│   ├── Channel ID
│   ├── Channel Name
│   ├── Channel Description
│   └── Channel Icon
│
├── Messaging API
│   ├── Bot basic ID
│   ├── QR Code
│   ├── Webhook URL ← ตั้งค่าที่นี่
│   ├── Use webhook (เปิด/ปิด)
│   └── Auto-reply messages
│
├── Security
│   ├── Channel Secret ← สำคัญมาก
│   └── IP Whitelist
│
└── Statistics
    ├── Message Delivery
    └── Chat
```

---

## 4. การสร้าง Channel

### ขั้นตอนการสร้าง Messaging API Channel

**ขั้นตอนที่ 1: สร้าง Provider**

1. เข้า https://developers.line.biz/console/
2. คลิก **Create a new provider**
3. ใส่ชื่อ Provider (เช่น "My Business")
4. คลิก **Create**

**ขั้นตอนที่ 2: สร้าง Channel**

1. คลิก **Create a new channel**
2. เลือก **Messaging API**
3. กรอกข้อมูล:
   - **Company or owner's country or region**: Thailand
   - **Channel icon**: อัพโหลดรูป (ไม่บังคับ)
   - **Channel name**: ชื่อ Bot ของคุณ
   - **Channel description**: คำอธิบาย
   - **Category**: เลือกประเภทธุรกิจ
   - **Subcategory**: เลือกหมวดย่อย
   - **Email address**: อีเมลติดต่อ
4. ยอมรับ Terms of Service
5. คลิก **Create**

### ข้อมูลสำคัญหลังสร้าง Channel

```json
{
  "channelId": "1234567890",
  "channelName": "My Bot",
  "channelDescription": "LINE Bot สำหรับธุรกิจ",
  "basicId": "@abc1234d",
  "premiumId": null,
  "pictureUrl": "https://profile.line-scdn.net/...",
  "channelType": "messaging-api"
}
```

---

## 5. Channel Access Token

### ประเภทของ Access Token

LINE Messaging API มี Access Token 2 ประเภทหลัก:

#### 5.1 Long-lived Channel Access Token (ไม่แนะนำสำหรับ Production)

- Token ที่ไม่มีวันหมดอายุ
- ใช้สำหรับ Development/Testing
- ออกได้สูงสุด 30 tokens ต่อ Channel
- ควรเพิกถอนเมื่อไม่ใช้แล้ว

**การออก Long-lived Token:**
1. ไปที่ Channel Settings → Messaging API
2. เลื่อนลงหา **Channel access token (long-lived)**
3. คลิก **Issue**
4. คัดลอก Token

```
eyJhbGciOiJIUzI1NiJ9.eyJ...  (ยาวมาก)
```

#### 5.2 Stateless Channel Access Token (แนะนำสำหรับ Production)

- Token ที่มีอายุ 30 วัน
- ออกโดยใช้ Channel Secret
- ปลอดภัยกว่า Long-lived Token
- ต้อง Refresh ก่อนหมดอายุ

**การออก Stateless Token ด้วย cURL:**

```bash
curl -v -X POST https://api.line.me/oauth2/v3/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=client_credentials" \
  --data-urlencode "client_id={Channel ID}" \
  --data-urlencode "client_secret={Channel Secret}"
```

**Response:**

```json
{
  "access_token": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6IjEifQ...",
  "token_type": "Bearer",
  "expires_in": 2592000,
  "key_id": "aBCd123"
}
```

#### 5.3 Short-lived Channel Access Token

- Token ที่มีอายุ 30 วัน
- ออกโดยใช้ JWT assertion
- เหมาะสำหรับระบบที่ต้องการ Rotate Token บ่อย

**การสร้าง JWT Assertion:**

```javascript
const jwt = require('jsonwebtoken');
const fs = require('fs');

// อ่าน Private Key ที่ดาวน์โหลดจาก LINE Console
const privateKey = fs.readFileSync('private_key.pem');

const header = {
  alg: 'RS256',
  typ: 'JWT',
  kid: 'YOUR_KEY_ID'  // จาก LINE Console
};

const payload = {
  iss: 'YOUR_CHANNEL_ID',
  sub: 'YOUR_CHANNEL_ID',
  aud: 'https://api.line.me/',
  exp: Math.floor(Date.now() / 1000) + 60 * 30, // 30 นาที
  token_exp: 60 * 60 * 24 * 30 // Access Token อายุ 30 วัน
};

const assertion = jwt.sign(payload, privateKey, {
  algorithm: 'RS256',
  header: header
});

console.log('JWT Assertion:', assertion);
```

**ขอ Token ด้วย JWT Assertion:**

```bash
curl -v -X POST https://api.line.me/oauth2/v2.1/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  --data-urlencode "grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer" \
  --data-urlencode "client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer" \
  --data-urlencode "client_assertion=YOUR_JWT_ASSERTION"
```

### การจัดการ Token อย่างปลอดภัย

```javascript
// config.js - อย่าใส่ Token ใน Code โดยตรง!

// ❌ ผิด - Hard code Token
const ACCESS_TOKEN = 'eyJhbGciOiJIUzI1NiJ9...';

// ✅ ถูก - ใช้ Environment Variable
const ACCESS_TOKEN = process.env.LINE_CHANNEL_ACCESS_TOKEN;

// ✅ ถูก - ใช้ Config File ที่ถูก Gitignore
const config = require('./config.json'); // ไม่ push ขึ้น Git
const ACCESS_TOKEN = config.lineAccessToken;
```

---

## 6. Channel Secret

Channel Secret เป็นรหัสลับที่ใช้สำหรับ:

1. **Signature Verification** - ตรวจสอบว่า Webhook มาจาก LINE จริง
2. **Token Generation** - ใช้ออก Stateless Access Token

### การหา Channel Secret

1. ไปที่ Channel Settings → Basic settings
2. ดูที่หัวข้อ **Channel secret**
3. คัดลอกค่า

```
Channel Secret: a1b2c3d4e5f6789012345678901234ab
```

### การใช้ Channel Secret สำหรับ Signature Verification

เมื่อ LINE ส่ง Webhook มายังเซิร์ฟเวอร์ จะมี HTTP Header:
- `X-Line-Signature`: Signature ที่ LINE สร้างจาก Channel Secret

**การตรวจสอบ Signature:**

```javascript
const crypto = require('crypto');

function verifySignature(channelSecret, body, signature) {
  const hmac = crypto.createHmac('SHA256', channelSecret);
  hmac.update(body);
  const digest = hmac.digest('base64');
  
  return digest === signature;
}

// ตัวอย่างการใช้ใน Express
app.post('/webhook', (req, res) => {
  const signature = req.headers['x-line-signature'];
  const body = JSON.stringify(req.body);
  
  if (!verifySignature(process.env.LINE_CHANNEL_SECRET, body, signature)) {
    return res.status(403).json({ error: 'Invalid signature' });
  }
  
  // Process webhook events
  res.status(200).json({ message: 'OK' });
});
```

---

## 7. Webhook URL Setup

### Webhook คืออะไร?

Webhook เป็น HTTP Endpoint บนเซิร์ฟเวอร์ของคุณที่ LINE จะส่ง POST Request มาเมื่อมี Event เกิดขึ้น

```
Event เกิดขึ้น (ผู้ใช้ส่งข้อความ)
         │
         ▼
    LINE Platform
         │
         │ POST https://your-server.com/webhook
         │ Body: { "events": [...] }
         ▼
    เซิร์ฟเวอร์คุณ
         │
         │ ประมวลผล
         ▼
    Reply API / Push API
         │
         ▼
    LINE Platform
         │
         ▼
    ผู้ใช้ได้รับข้อความ
```

### ข้อกำหนดของ Webhook URL

1. **HTTPS เท่านั้น** - HTTP ไม่ได้รับอนุญาต
2. **Port 443** - ต้องใช้ port มาตรฐาน
3. **Valid SSL Certificate** - ต้องมี SSL ที่ถูกต้อง (ไม่ใช่ Self-signed)
4. **Response 200 ภายใน 1 วินาที** - ต้องตอบกลับเร็ว

### การตั้งค่า Webhook URL

1. ไปที่ Channel Settings → Messaging API
2. คลิก **Edit** ที่ Webhook URL
3. ใส่ URL เซิร์ฟเวอร์: `https://your-domain.com/webhook`
4. คลิก **Update**
5. คลิก **Verify** เพื่อทดสอบ

### ตัวอย่าง Webhook Server พื้นฐาน

```javascript
const express = require('express');
const app = express();

app.use(express.json());

app.post('/webhook', (req, res) => {
  // LINE จะส่ง verify request มาก่อน
  const events = req.body.events;
  
  console.log('Received events:', JSON.stringify(events, null, 2));
  
  // ต้องตอบกลับ 200 เสมอ
  res.status(200).json({ message: 'OK' });
});

app.listen(3000, () => {
  console.log('Webhook server running on port 3000');
});
```

### Webhook Event Structure

```json
{
  "destination": "xxxxxxxxxx",
  "events": [
    {
      "type": "message",
      "message": {
        "type": "text",
        "id": "14353798921116",
        "text": "Hello, World!"
      },
      "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
      "deliveryContext": {
        "isRedelivery": false
      },
      "timestamp": 1625665242211,
      "source": {
        "type": "user",
        "userId": "U4af4980629..."
      },
      "replyToken": "0f3779fba3b349968c5d07db31eab56f",
      "mode": "active"
    }
  ]
}
```

---

## 8. การทดสอบด้วย cURL

### 8.1 ทดสอบ Reply API

```bash
# Reply ข้อความหาผู้ใช้
curl -v -X POST https://api.line.me/v2/bot/message/reply \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer {YOUR_ACCESS_TOKEN}' \
  -d '{
    "replyToken": "{REPLY_TOKEN}",
    "messages": [
      {
        "type": "text",
        "text": "สวัสดีครับ! ยินดีต้อนรับ"
      }
    ]
  }'
```

**Response สำเร็จ:**
```json
{
  "sentMessages": [
    {
      "id": "461230966842"
    }
  ]
}
```

### 8.2 ทดสอบ Push API

```bash
# Push ข้อความหาผู้ใช้
curl -v -X POST https://api.line.me/v2/bot/message/push \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer {YOUR_ACCESS_TOKEN}' \
  -d '{
    "to": "{USER_ID}",
    "messages": [
      {
        "type": "text",
        "text": "นี่คือ Push Message!"
      }
    ]
  }'
```

### 8.3 ทดสอบ Broadcast API

```bash
# Broadcast ข้อความหาทุกคนที่ Follow
curl -v -X POST https://api.line.me/v2/bot/message/broadcast \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer {YOUR_ACCESS_TOKEN}' \
  -d '{
    "messages": [
      {
        "type": "text",
        "text": "ข่าวประกาศ: มีโปรโมชันใหม่!"
      }
    ]
  }'
```

### 8.4 ดึงข้อมูล Bot Profile

```bash
curl -v -X GET https://api.line.me/v2/bot/info \
  -H 'Authorization: Bearer {YOUR_ACCESS_TOKEN}'
```

**Response:**
```json
{
  "userId": "U4af4980629...",
  "basicId": "@abc.defg",
  "displayName": "My Bot",
  "pictureUrl": "https://profile.line-scdn.net/...",
  "chatMode": "bot",
  "markAsReadMode": "auto"
}
```

### 8.5 ดึงข้อมูล User Profile

```bash
curl -v -X GET "https://api.line.me/v2/bot/profile/{userId}" \
  -H 'Authorization: Bearer {YOUR_ACCESS_TOKEN}'
```

**Response:**
```json
{
  "userId": "U4af4980629...",
  "displayName": "John Doe",
  "pictureUrl": "https://profile.line-scdn.net/...",
  "statusMessage": "Hello!",
  "language": "th"
}
```

### 8.6 ตรวจสอบ Quota

```bash
# ดู Message Quota
curl -v -X GET https://api.line.me/v2/bot/message/quota \
  -H 'Authorization: Bearer {YOUR_ACCESS_TOKEN}'
```

**Response:**
```json
{
  "type": "limited",
  "value": 1000
}
```

```bash
# ดู Message Quota Consumption
curl -v -X GET https://api.line.me/v2/bot/message/quota/consumption \
  -H 'Authorization: Bearer {YOUR_ACCESS_TOKEN}'
```

**Response:**
```json
{
  "totalUsage": 539
}
```

### 8.7 ส่ง Multiple Messages

```bash
curl -v -X POST https://api.line.me/v2/bot/message/reply \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer {YOUR_ACCESS_TOKEN}' \
  -d '{
    "replyToken": "{REPLY_TOKEN}",
    "messages": [
      {
        "type": "text",
        "text": "ข้อความที่ 1"
      },
      {
        "type": "text",
        "text": "ข้อความที่ 2"
      },
      {
        "type": "sticker",
        "packageId": "446",
        "stickerId": "1988"
      }
    ]
  }'
```

### 8.8 ทดสอบ Image Message

```bash
curl -v -X POST https://api.line.me/v2/bot/message/push \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer {YOUR_ACCESS_TOKEN}' \
  -d '{
    "to": "{USER_ID}",
    "messages": [
      {
        "type": "image",
        "originalContentUrl": "https://example.com/image.jpg",
        "previewImageUrl": "https://example.com/preview.jpg"
      }
    ]
  }'
```

---

## 9. Rate Limits

### ประเภทของ Rate Limits

LINE Messaging API มี Rate Limits หลายระดับ:

#### 9.1 Message API Rate Limits

| API | Limit |
|-----|-------|
| Reply API | 1,000 request/min ต่อ Channel |
| Push API | 200 request/sec ต่อ Channel |
| Broadcast API | 1 request/sec |
| Multicast API | 200 request/sec |
| Narrowcast API | 1 request/sec |

#### 9.2 Monthly Message Limits (Free Tier)

| Plan | Messages/Month |
|------|---------------|
| Free | 500 |
| Light | 15,000 |
| Standard | 45,000 |
| ขึ้นอยู่กับ Plan | มากกว่า |

> **หมายเหตุ:** ข้อมูล Rate Limits อาจเปลี่ยนแปลง ตรวจสอบที่ https://developers.line.biz/en/docs/messaging-api/rate-limits/

#### 9.3 HTTP Rate Limit Headers

เมื่อถึง Rate Limit จะได้รับ Response:

```
HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1625665300
```

```json
{
  "message": "Too many requests"
}
```

#### 9.4 การจัดการ Rate Limit ใน Code

```javascript
const axios = require('axios');

class LineMessagingClient {
  constructor(accessToken) {
    this.accessToken = accessToken;
    this.requestQueue = [];
    this.processing = false;
    this.requestsPerSecond = 50; // Conservative limit
  }

  async sendMessage(to, messages) {
    return new Promise((resolve, reject) => {
      this.requestQueue.push({ to, messages, resolve, reject });
      this.processQueue();
    });
  }

  async processQueue() {
    if (this.processing || this.requestQueue.length === 0) return;
    
    this.processing = true;
    
    while (this.requestQueue.length > 0) {
      const batch = this.requestQueue.splice(0, 5); // Process 5 at a time
      
      await Promise.all(batch.map(async ({ to, messages, resolve, reject }) => {
        try {
          const response = await this._push(to, messages);
          resolve(response);
        } catch (error) {
          if (error.response && error.response.status === 429) {
            // Retry after delay
            const retryAfter = error.response.headers['retry-after'] || 1;
            await this.delay(retryAfter * 1000);
            this.requestQueue.unshift({ to, messages, resolve, reject });
          } else {
            reject(error);
          }
        }
      }));
      
      // Throttle requests
      await this.delay(1000 / this.requestsPerSecond * batch.length);
    }
    
    this.processing = false;
  }

  delay(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  async _push(to, messages) {
    return axios.post(
      'https://api.line.me/v2/bot/message/push',
      { to, messages },
      {
        headers: {
          'Authorization': `Bearer ${this.accessToken}`,
          'Content-Type': 'application/json'
        }
      }
    );
  }
}
```

#### 9.5 Retry Strategy

```javascript
async function sendWithRetry(fn, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (error.response) {
        const status = error.response.status;
        
        if (status === 429) {
          // Rate limited - รอแล้วลองใหม่
          const retryAfter = parseInt(error.response.headers['retry-after'] || '5');
          console.log(`Rate limited. Retrying after ${retryAfter} seconds...`);
          await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
          continue;
        }
        
        if (status >= 500) {
          // Server error - ลองใหม่ด้วย exponential backoff
          const delay = Math.pow(2, attempt) * 1000;
          console.log(`Server error. Retrying after ${delay}ms...`);
          await new Promise(resolve => setTimeout(resolve, delay));
          continue;
        }
        
        // Client error - ไม่ต้องลองใหม่
        throw error;
      }
      
      // Network error - ลองใหม่
      if (attempt < maxRetries - 1) {
        await new Promise(resolve => setTimeout(resolve, 1000));
        continue;
      }
      
      throw error;
    }
  }
  
  throw new Error(`Failed after ${maxRetries} retries`);
}
```

---

## 10. API Versioning

### เวอร์ชันปัจจุบันของ LINE APIs

| API | Version | Base URL |
|-----|---------|----------|
| Messaging API | v2 | `https://api.line.me/v2/bot/` |
| OAuth 2.0 | v3 | `https://api.line.me/oauth2/v3/` |
| LINE Login | v2.1 | `https://api.line.me/oauth2/v2.1/` |
| User Data | v2 | `https://api.line.me/v2/bot/` |

### Endpoint Reference

```
# Bot Management
GET  /v2/bot/info                    - ข้อมูล Bot
GET  /v2/bot/followers/ids           - รายชื่อ Followers

# Messages
POST /v2/bot/message/reply           - Reply Message
POST /v2/bot/message/push            - Push Message
POST /v2/bot/message/broadcast       - Broadcast Message
POST /v2/bot/message/multicast       - Multicast Message
POST /v2/bot/message/narrowcast      - Narrowcast Message

# Users
GET  /v2/bot/profile/{userId}        - User Profile
GET  /v2/bot/group/{groupId}/member/{userId}  - Group Member Profile

# Rich Menu
POST /v2/bot/richmenu                - Create Rich Menu
GET  /v2/bot/richmenu/{richMenuId}   - Get Rich Menu
DELETE /v2/bot/richmenu/{richMenuId} - Delete Rich Menu
POST /v2/bot/user/{userId}/richmenu/{richMenuId} - Link to User

# Content
GET  /v2/bot/message/{messageId}/content - Get Content (image/video/audio/file)
```

### การจัดการ API Version ใน Code

```javascript
// lib/lineApi.js
const BASE_URL = 'https://api.line.me';
const API_VERSION = 'v2';

const LINE_API = {
  // Messages
  reply: `${BASE_URL}/${API_VERSION}/bot/message/reply`,
  push: `${BASE_URL}/${API_VERSION}/bot/message/push`,
  broadcast: `${BASE_URL}/${API_VERSION}/bot/message/broadcast`,
  multicast: `${BASE_URL}/${API_VERSION}/bot/message/multicast`,
  
  // Users
  profile: (userId) => `${BASE_URL}/${API_VERSION}/bot/profile/${userId}`,
  
  // Rich Menu
  richMenu: `${BASE_URL}/${API_VERSION}/bot/richmenu`,
  richMenuById: (id) => `${BASE_URL}/${API_VERSION}/bot/richmenu/${id}`,
  
  // Bot Info
  botInfo: `${BASE_URL}/${API_VERSION}/bot/info`,
  
  // Quota
  quota: `${BASE_URL}/${API_VERSION}/bot/message/quota`,
  quotaConsumption: `${BASE_URL}/${API_VERSION}/bot/message/quota/consumption`,
};

module.exports = LINE_API;
```

### HTTP Status Codes

| Code | ความหมาย |
|------|----------|
| 200 | สำเร็จ |
| 400 | Bad Request - ข้อมูลไม่ถูกต้อง |
| 401 | Unauthorized - Token ไม่ถูกต้อง |
| 403 | Forbidden - ไม่มีสิทธิ์ |
| 404 | Not Found - ไม่พบข้อมูล |
| 429 | Too Many Requests - Rate Limited |
| 500 | Internal Server Error - LINE Server Error |

### Error Response Format

```json
{
  "message": "Invalid reply token"
}
```

```json
{
  "message": "The channel access token is invalid"
}
```

### การตรวจสอบ Error ใน Code

```javascript
async function handleLineApiError(error) {
  if (!error.response) {
    console.error('Network error:', error.message);
    return;
  }
  
  const { status, data } = error.response;
  
  switch (status) {
    case 400:
      console.error('Bad Request:', data.message);
      // ตรวจสอบ request body
      break;
      
    case 401:
      console.error('Unauthorized: Token ไม่ถูกต้องหรือหมดอายุ');
      // Refresh token
      await refreshAccessToken();
      break;
      
    case 403:
      console.error('Forbidden:', data.message);
      // ตรวจสอบสิทธิ์
      break;
      
    case 404:
      console.error('Not Found:', data.message);
      // ตรวจสอบ user ID หรือ message ID
      break;
      
    case 429:
      console.error('Rate Limited');
      // Wait and retry
      const retryAfter = error.response.headers['retry-after'];
      await new Promise(r => setTimeout(r, (retryAfter || 5) * 1000));
      break;
      
    case 500:
      console.error('LINE Server Error - ลองใหม่ภายหลัง');
      break;
      
    default:
      console.error(`Unexpected error ${status}:`, data);
  }
}
```

---

## 11. ตัวอย่างโค้ดสมบูรณ์: LINE API Client

```javascript
// lineClient.js
const axios = require('axios');
const crypto = require('crypto');

class LineAPIClient {
  constructor(options) {
    this.channelAccessToken = options.channelAccessToken;
    this.channelSecret = options.channelSecret;
    this.baseURL = 'https://api.line.me/v2/bot';
    
    this.httpClient = axios.create({
      baseURL: this.baseURL,
      headers: {
        'Authorization': `Bearer ${this.channelAccessToken}`,
        'Content-Type': 'application/json'
      },
      timeout: 10000
    });
    
    // Add request/response interceptors
    this.setupInterceptors();
  }
  
  setupInterceptors() {
    // Request interceptor - log requests
    this.httpClient.interceptors.request.use(
      (config) => {
        console.log(`LINE API Request: ${config.method?.toUpperCase()} ${config.url}`);
        return config;
      },
      (error) => Promise.reject(error)
    );
    
    // Response interceptor - handle errors
    this.httpClient.interceptors.response.use(
      (response) => {
        console.log(`LINE API Response: ${response.status}`);
        return response;
      },
      async (error) => {
        if (error.response?.status === 429) {
          const retryAfter = parseInt(error.response.headers['retry-after'] || '5');
          await new Promise(r => setTimeout(r, retryAfter * 1000));
          return this.httpClient.request(error.config);
        }
        return Promise.reject(error);
      }
    );
  }
  
  // Verify Webhook Signature
  verifySignature(body, signature) {
    const hmac = crypto.createHmac('SHA256', this.channelSecret);
    const bodyString = typeof body === 'string' ? body : JSON.stringify(body);
    hmac.update(bodyString);
    const digest = hmac.digest('base64');
    return digest === signature;
  }
  
  // Reply Message
  async replyMessage(replyToken, messages) {
    if (!Array.isArray(messages)) {
      messages = [messages];
    }
    
    const response = await this.httpClient.post('/message/reply', {
      replyToken,
      messages
    });
    
    return response.data;
  }
  
  // Push Message
  async pushMessage(to, messages) {
    if (!Array.isArray(messages)) {
      messages = [messages];
    }
    
    const response = await this.httpClient.post('/message/push', {
      to,
      messages
    });
    
    return response.data;
  }
  
  // Broadcast Message
  async broadcastMessage(messages) {
    if (!Array.isArray(messages)) {
      messages = [messages];
    }
    
    const response = await this.httpClient.post('/message/broadcast', {
      messages
    });
    
    return response.data;
  }
  
  // Multicast Message
  async multicastMessage(to, messages) {
    if (!Array.isArray(to)) {
      to = [to];
    }
    if (!Array.isArray(messages)) {
      messages = [messages];
    }
    
    // LINE multicast supports max 500 recipients
    const results = [];
    for (let i = 0; i < to.length; i += 500) {
      const batch = to.slice(i, i + 500);
      const response = await this.httpClient.post('/message/multicast', {
        to: batch,
        messages
      });
      results.push(response.data);
    }
    
    return results;
  }
  
  // Get User Profile
  async getUserProfile(userId) {
    const response = await this.httpClient.get(`/profile/${userId}`);
    return response.data;
  }
  
  // Get Bot Info
  async getBotInfo() {
    const response = await this.httpClient.get('/info');
    return response.data;
  }
  
  // Get Message Quota
  async getMessageQuota() {
    const response = await this.httpClient.get('/message/quota');
    return response.data;
  }
  
  // Get Message Quota Consumption
  async getMessageQuotaConsumption() {
    const response = await this.httpClient.get('/message/quota/consumption');
    return response.data;
  }
  
  // Get Content (for image, video, audio, file messages)
  async getMessageContent(messageId) {
    const response = await this.httpClient.get(
      `/message/${messageId}/content`,
      { responseType: 'stream' }
    );
    return response.data;
  }
}

module.exports = LineAPIClient;
```

### การใช้งาน Client

```javascript
// index.js
require('dotenv').config();
const express = require('express');
const LineAPIClient = require('./lineClient');

const app = express();
const client = new LineAPIClient({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
});

app.use(express.json({
  verify: (req, res, buf) => {
    req.rawBody = buf;
  }
}));

app.post('/webhook', async (req, res) => {
  // Verify signature
  const signature = req.headers['x-line-signature'];
  
  if (!client.verifySignature(req.rawBody.toString(), signature)) {
    return res.status(403).json({ error: 'Invalid signature' });
  }
  
  const events = req.body.events;
  
  // ตอบกลับ 200 ก่อนเพื่อไม่ให้ timeout
  res.status(200).json({ message: 'OK' });
  
  // Process events asynchronously
  for (const event of events) {
    try {
      await handleEvent(event);
    } catch (error) {
      console.error('Error handling event:', error);
    }
  }
});

async function handleEvent(event) {
  if (event.type === 'message' && event.message.type === 'text') {
    const userMessage = event.message.text;
    const replyToken = event.replyToken;
    
    await client.replyMessage(replyToken, {
      type: 'text',
      text: `คุณพิมพ์ว่า: ${userMessage}`
    });
  }
}

app.listen(process.env.PORT || 3000, () => {
  console.log('Server started');
});
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Channel และทดสอบ
1. สร้าง Provider และ Channel ใหม่บน LINE Developers Console
2. ออก Long-lived Access Token
3. ใช้ cURL ทดสอบเรียก `/v2/bot/info`
4. บันทึกผลลัพธ์ที่ได้

### แบบฝึกหัดที่ 2: Signature Verification
เขียนฟังก์ชัน `verifyWebhook` ใน Python ที่:
1. รับ body (bytes) และ signature (string)
2. ตรวจสอบความถูกต้องของ signature
3. Return True/False

```python
import hmac
import hashlib
import base64

def verify_webhook(channel_secret: str, body: bytes, signature: str) -> bool:
    # TODO: เขียนโค้ดที่นี่
    pass
```

### แบบฝึกหัดที่ 3: Rate Limit Simulation
เขียนโค้ดที่:
1. ส่ง Push Message 100 ครั้ง
2. จับ Error 429 และรอตามที่กำหนด
3. Log เวลาที่ใช้ทั้งหมด

### แบบฝึกหัดที่ 4: Token Management
สร้าง `TokenManager` class ที่:
1. ออก Stateless Token เมื่อต้องการ
2. Cache Token และตรวจสอบอายุ
3. Refresh อัตโนมัติเมื่อ Token จะหมดอายุ (เหลือน้อยกว่า 1 ชั่วโมง)

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **LINE Messaging API** - API สำหรับสร้าง Chatbot บน LINE
2. **ความแตกต่าง** - LINE OA Manager vs Messaging API
3. **Developer Console** - โครงสร้างและการใช้งาน
4. **Channel** - การสร้างและการตั้งค่า
5. **Access Token** - ประเภทต่างๆ และการจัดการ
6. **Channel Secret** - การใช้งานและความปลอดภัย
7. **Webhook** - การตั้งค่าและโครงสร้าง Event
8. **cURL Testing** - การทดสอบ API ด้วย Command Line
9. **Rate Limits** - ข้อจำกัดและวิธีจัดการ
10. **API Versioning** - Endpoints และ Error Handling

ในบทต่อไปเราจะเรียนรู้การสร้าง LINE Bot ด้วย Node.js อย่างละเอียด!

---

*ดูข้อมูลเพิ่มเติมที่ https://developers.line.biz/en/docs/messaging-api/*
