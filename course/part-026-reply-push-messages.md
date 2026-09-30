# ส่วนที่ 26: Reply, Push, Multicast และ Broadcast Message API

---

## สารบัญ

1. [ภาพรวมประเภทการส่งข้อความ](#1-ภาพรวมประเภทการส่งข้อความ)
2. [Reply Message API](#2-reply-message-api)
3. [Push Message API](#3-push-message-api)
4. [Multicast Message API](#4-multicast-message-api)
5. [Broadcast Message API](#5-broadcast-message-api)
6. [Token-based vs Channel-based Messages](#6-token-based-vs-channel-based-messages)
7. [Message Quotas และ Billing](#7-message-quotas-และ-billing)
8. [Batch Sending](#8-batch-sending)
9. [Code Examples ใน Node.js](#9-code-examples-ใน-nodejs)
10. [Code Examples ใน Python](#10-code-examples-ใน-python)
11. [Best Practices สำหรับการเลือกประเภทข้อความ](#11-best-practices-สำหรับการเลือกประเภทข้อความ)
12. [Error Handling และ Retry Logic](#12-error-handling-และ-retry-logic)
13. [แบบฝึกหัดท้ายบท](#13-แบบฝึกหัดท้ายบท)

---

## 1. ภาพรวมประเภทการส่งข้อความ

LINE Messaging API มีวิธีการส่งข้อความให้ผู้ใช้หลายรูปแบบ แต่ละประเภทมีจุดประสงค์ เงื่อนไข และค่าใช้จ่ายที่แตกต่างกัน การเลือกใช้ให้ถูกต้องจะช่วยประหยัดต้นทุนและปรับปรุงประสบการณ์ผู้ใช้ได้อย่างมาก

### ตารางเปรียบเทียบประเภทการส่งข้อความ

| ประเภท | ทริกเกอร์ | ผู้รับ | นับโควต้า | ใช้กรณีใด |
|--------|----------|--------|----------|-----------|
| Reply | ต้องมี replyToken | 1 คน | ไม่นับ | ตอบกลับข้อความผู้ใช้ |
| Push | ไม่ต้องมี trigger | 1 คน | นับ | ส่งข้อความเชิงรุก |
| Multicast | ไม่ต้องมี trigger | หลายคน (สูงสุด 500) | นับ | ส่งข้อความกลุ่มเป้าหมาย |
| Broadcast | ไม่ต้องมี trigger | ทุกคนที่ Follow | นับ | ส่งข้อความหาทุกคน |
| Narrowcast | ไม่ต้องมี trigger | กลุ่มเป้าหมายเฉพาะ | นับ | ส่งตาม demographic/segment |

### Flow การทำงานของแต่ละประเภท

```
ผู้ใช้ส่งข้อความ → Webhook → Bot ประมวลผล
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
              Reply Token      Push Message    Broadcast
              (ฟรี ไม่นับ)   (นับโควต้า)   (นับโควต้า)
                    │               │               │
              ตอบกลับ 1:1    ส่งหา userId    ส่งทุก follower
```

---

## 2. Reply Message API

Reply Message API ใช้สำหรับตอบกลับข้อความที่ผู้ใช้ส่งมา โดยอาศัย `replyToken` ที่ได้รับจาก Webhook event ข้อดีคือ **ไม่นับโควต้า** แต่มีข้อจำกัดคือ replyToken มีอายุเพียง **30 วินาที**

### คุณสมบัติของ Reply Message

- **ไม่มีค่าใช้จ่าย** - ไม่นับต่อโควต้า messaging
- **ต้องใช้ replyToken** - ได้จาก Webhook event เท่านั้น
- **อายุ 30 วินาที** - replyToken หมดอายุเร็ว ต้องตอบกลับทันที
- **ส่งได้สูงสุด 5 ข้อความ** ต่อ 1 reply
- **1 replyToken ใช้ได้ครั้งเดียว** - ใช้แล้วใช้อีกไม่ได้

### โครงสร้าง Webhook Event ที่มี replyToken

```json
{
  "destination": "xxxxxxxxxx",
  "events": [
    {
      "type": "message",
      "message": {
        "type": "text",
        "id": "14353798921116",
        "text": "Hello, world"
      },
      "timestamp": 1625665242211,
      "source": {
        "type": "user",
        "userId": "U4af4980629..."
      },
      "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
      "deliveryContext": {
        "isRedelivery": false
      },
      "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA",
      "mode": "active"
    }
  ]
}
```

### Reply Message API Endpoint

```
POST https://api.line.me/v2/bot/message/reply
```

### Request Body สำหรับ Reply

```json
{
  "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA",
  "messages": [
    {
      "type": "text",
      "text": "สวัสดีครับ! ยินดีให้บริการ"
    }
  ],
  "notificationDisabled": false
}
```

### Parameters ของ Reply Message

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| replyToken | String | Yes | Token สำหรับตอบกลับ |
| messages | Array | Yes | ข้อความที่จะส่ง (สูงสุด 5) |
| notificationDisabled | Boolean | No | ปิดการแจ้งเตือน (default: false) |

### ตัวอย่าง Reply พร้อม Multiple Messages

```json
{
  "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA",
  "messages": [
    {
      "type": "text",
      "text": "ขอบคุณที่ติดต่อมาครับ"
    },
    {
      "type": "text",
      "text": "เราจะดำเนินการให้โดยเร็วที่สุด"
    },
    {
      "type": "image",
      "originalContentUrl": "https://example.com/image.jpg",
      "previewImageUrl": "https://example.com/preview.jpg"
    }
  ]
}
```

### เมื่อไหร่ควรใช้ Reply Message?

✅ **ควรใช้เมื่อ:**
- ผู้ใช้ส่งข้อความมาและต้องการตอบกลับทันที
- Bot ประมวลผลคำถามและต้องให้คำตอบ
- Chatbot ตอบโต้แบบ real-time

❌ **ไม่ควรใช้เมื่อ:**
- ต้องการส่งข้อความหลังจาก 30 วินาทีผ่านไป
- ต้องการส่งข้อความแบบ scheduled
- replyToken ถูกใช้ไปแล้ว

---

## 3. Push Message API

Push Message API ช่วยให้ Bot สามารถส่งข้อความหาผู้ใช้ได้โดยไม่ต้องรอให้ผู้ใช้ส่งมาก่อน เหมาะสำหรับการแจ้งเตือน การส่งข้อมูลอัปเดต หรือระบบ CRM ที่ต้องการ proactive messaging

### คุณสมบัติของ Push Message

- **นับโควต้า** - ทุกข้อความที่ส่งจะนับต่อโควต้า
- **ต้องรู้ userId/groupId/roomId** - ต้องเก็บไว้ล่วงหน้า
- **ไม่มี trigger** - ส่งได้ทุกเวลา
- **ส่งได้สูงสุด 5 ข้อความ** ต่อ 1 API call
- **Rate limit**: 100,000 requests/minute

### Push Message API Endpoint

```
POST https://api.line.me/v2/bot/message/push
```

### Request Body สำหรับ Push

```json
{
  "to": "U4af4980629...",
  "messages": [
    {
      "type": "text",
      "text": "สวัสดีครับ นี่คือข้อความแจ้งเตือน"
    }
  ],
  "notificationDisabled": false,
  "customAggregationUnits": ["promotion_a"]
}
```

### Parameters ของ Push Message

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| to | String | Yes | userId, groupId, หรือ roomId |
| messages | Array | Yes | ข้อความที่จะส่ง (สูงสุด 5) |
| notificationDisabled | Boolean | No | ปิดการแจ้งเตือน |
| customAggregationUnits | Array | No | หน่วยการรวมข้อมูล (สูงสุด 1) |

### ตัวอย่าง Use Cases ของ Push Message

**1. Order Confirmation**
```json
{
  "to": "U4af4980629...",
  "messages": [
    {
      "type": "text",
      "text": "✅ ยืนยันคำสั่งซื้อ #12345\n\nสินค้า: เสื้อยืด XL\nราคา: 599 บาท\nสถานะ: กำลังเตรียมจัดส่ง"
    }
  ]
}
```

**2. Reminder**
```json
{
  "to": "U4af4980629...",
  "messages": [
    {
      "type": "text",
      "text": "⏰ แจ้งเตือน: คุณมีนัดพบแพทย์พรุ่งนี้ เวลา 10:00 น."
    }
  ]
}
```

**3. Delivery Update**
```json
{
  "to": "U4af4980629...",
  "messages": [
    {
      "type": "text",
      "text": "🚚 พัสดุของคุณถูกส่งออกแล้ว\n\nเลขพัสดุ: TH123456789\nบริษัทขนส่ง: Flash Express\nคาดว่าถึง: วันนี้ 15:00-18:00 น."
    }
  ]
}
```

---

## 4. Multicast Message API

Multicast Message API ใช้ส่งข้อความเดียวกันหาผู้ใช้หลายคนพร้อมกัน โดยสามารถระบุ userId ได้สูงสุด 500 คนต่อ 1 API call เหมาะสำหรับการส่งข้อความแบบกำหนดกลุ่มเป้าหมาย

### คุณสมบัติของ Multicast

- **นับโควต้า** - นับตาม userId ที่ส่ง
- **ส่งได้สูงสุด 500 userId** ต่อ 1 call
- **ส่ง userId เดียวกันหลายครั้งได้** แต่จะนับโควต้าทุกครั้ง
- **ไม่มี limit** สำหรับจำนวนครั้งที่เรียก API
- **Rate limit**: 100,000 requests/minute

### Multicast Message API Endpoint

```
POST https://api.line.me/v2/bot/message/multicast
```

### Request Body สำหรับ Multicast

```json
{
  "to": [
    "U4af4980629...",
    "U0d1c0d4e5...",
    "Uf8d5d3b2c..."
  ],
  "messages": [
    {
      "type": "text",
      "text": "สวัสดีลูกค้า VIP ทุกท่าน!\nเรามีโปรโมชั่นพิเศษสำหรับคุณ"
    }
  ],
  "notificationDisabled": false
}
```

### การแบ่ง Batch สำหรับ Multicast

เมื่อมีผู้ใช้มากกว่า 500 คน ต้องแบ่ง batch:

```javascript
// ตัวอย่างการแบ่ง batch
const userIds = ['U001', 'U002', /* ... 10,000 คน */];
const batchSize = 500;

for (let i = 0; i < userIds.length; i += batchSize) {
  const batch = userIds.slice(i, i + batchSize);
  await sendMulticast(batch, messages);
  // รอสักครู่เพื่อไม่ให้ rate limit
  await sleep(100);
}
```

---

## 5. Broadcast Message API

Broadcast Message API ส่งข้อความหาผู้ติดตาม (Followers) ทั้งหมดของ LINE OA โดยอัตโนมัติ ไม่ต้องระบุ userId เหมาะสำหรับการประกาศข่าวสาร โปรโมชั่นทั่วไป

### คุณสมบัติของ Broadcast

- **นับโควต้า** - นับตามจำนวน followers ที่รับข้อความ
- **ไม่ต้องระบุ userId**
- **ส่งหา followers ทั้งหมด** โดยอัตโนมัติ
- **Rate limit**: 60 requests/hour

### Broadcast Message API Endpoint

```
POST https://api.line.me/v2/bot/message/broadcast
```

### Request Body สำหรับ Broadcast

```json
{
  "messages": [
    {
      "type": "text",
      "text": "📣 ประกาศสำคัญ!\n\nเราจะปิดปรับปรุงระบบในวันที่ 1 ม.ค. 2025\nเวลา 02:00 - 04:00 น.\nขออภัยในความไม่สะดวก"
    }
  ],
  "notificationDisabled": false
}
```

---

## 6. Token-based vs Channel-based Messages

### Channel Access Token

LINE Messaging API ใช้ Channel Access Token ในการยืนยันตัวตน มี 2 ประเภทหลัก:

#### 1. Stateless Channel Access Token (แนะนำ)

```javascript
// สร้าง Stateless Token
const jwt = require('jsonwebtoken');

function generateStatelessToken(channelId, privateKey) {
  const header = {
    alg: 'RS256',
    typ: 'JWT',
    kid: 'YOUR_KEY_ID'
  };
  
  const payload = {
    iss: channelId,
    sub: channelId,
    aud: 'https://api.line.me/',
    exp: Math.floor(Date.now() / 1000) + 30, // 30 วินาที
    token_exp: 60 * 60 * 24 * 30 // 30 วัน
  };
  
  return jwt.sign(payload, privateKey, { header });
}
```

#### 2. Long-lived Channel Access Token

```javascript
// Long-lived token (อายุ 30 วัน)
const channelAccessToken = 'YOUR_LONG_LIVED_TOKEN';

// ใช้ใน header
const headers = {
  'Authorization': `Bearer ${channelAccessToken}`,
  'Content-Type': 'application/json'
};
```

### ความแตกต่างของ Token Types

| ประเภท Token | อายุ | ความปลอดภัย | ใช้งาน |
|-------------|------|-------------|--------|
| Stateless | สั้นมาก (30 วิ) | สูงสุด | Production |
| Short-lived | 30 วัน | สูง | Production |
| Long-lived | ไม่หมดอายุ (ต่ออายุเอง) | ปานกลาง | Development |

---

## 7. Message Quotas และ Billing

### แผนราคา LINE OA

| แผน | ราคา/เดือน | Free Messages | Additional Messages |
|-----|-----------|---------------|---------------------|
| Free | ฟรี | 500 | ไม่สามารถซื้อเพิ่มได้ |
| Light | 399 บาท | 15,000 | 0.78 บาท/ข้อความ |
| Standard | 1,200 บาท | 45,000 | 0.60 บาท/ข้อความ |

### ข้อความที่นับโควต้า vs ไม่นับ

**นับโควต้า:**
- Push Message
- Multicast Message
- Broadcast Message
- Narrowcast Message

**ไม่นับโควต้า:**
- Reply Message (ตอบกลับ Webhook)
- ข้อความที่ผู้ใช้ส่งมาหา Bot

### ตรวจสอบโควต้าที่เหลือ

```
GET https://api.line.me/v2/bot/message/quota
```

Response:
```json
{
  "type": "limited",
  "value": 500
}
```

```
GET https://api.line.me/v2/bot/message/quota/consumption
```

Response:
```json
{
  "totalUsage": 234
}
```

---

## 8. Batch Sending

### Batch Send API

สำหรับการส่งข้อความจำนวนมากอย่างมีประสิทธิภาพ LINE มี Batch Send API ที่ช่วยลดจำนวน API calls

### Endpoint

```
POST https://api.line.me/v2/bot/message/push
```

### ตัวอย่าง Batch Send

```javascript
// ส่งข้อความ batch พร้อมกัน
async function sendBatchMessages(userGroups, message) {
  const promises = userGroups.map(async (userIds) => {
    return await client.multicast(userIds, [message]);
  });
  
  const results = await Promise.allSettled(promises);
  
  results.forEach((result, index) => {
    if (result.status === 'rejected') {
      console.error(`Batch ${index} failed:`, result.reason);
    }
  });
}
```

---

## 9. Code Examples ใน Node.js

### การติดตั้ง Dependencies

```bash
npm init -y
npm install @line/bot-sdk express axios dotenv
```

### โครงสร้างโปรเจกต์

```
project/
├── .env
├── app.js
├── handlers/
│   ├── messageHandler.js
│   └── eventHandler.js
├── services/
│   ├── lineService.js
│   └── userService.js
└── utils/
    └── helpers.js
```

### ไฟล์ .env

```env
LINE_CHANNEL_ACCESS_TOKEN=your_channel_access_token
LINE_CHANNEL_SECRET=your_channel_secret
PORT=3000
```

### app.js - Express Server พื้นฐาน

```javascript
'use strict';

require('dotenv').config();
const express = require('express');
const line = require('@line/bot-sdk');

const app = express();

// LINE SDK Config
const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};

const client = new line.messagingApi.MessagingApiClient({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

// Middleware
app.use('/webhook', line.middleware(config));

// Webhook endpoint
app.post('/webhook', async (req, res) => {
  const events = req.body.events;
  
  try {
    await Promise.all(events.map(handleEvent));
    res.json({ status: 'ok' });
  } catch (error) {
    console.error('Error handling events:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Handle each event
async function handleEvent(event) {
  console.log('Event received:', JSON.stringify(event, null, 2));
  
  if (event.type !== 'message' || event.message.type !== 'text') {
    return null;
  }
  
  return handleTextMessage(event);
}

// Handle text message
async function handleTextMessage(event) {
  const userMessage = event.message.text;
  const replyToken = event.replyToken;
  
  // ตัวอย่าง: Echo bot
  return replyMessage(replyToken, userMessage);
}

app.listen(process.env.PORT, () => {
  console.log(`Server running on port ${process.env.PORT}`);
});
```

### lineService.js - Reply Message

```javascript
'use strict';

require('dotenv').config();
const line = require('@line/bot-sdk');

const client = new line.messagingApi.MessagingApiClient({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

/**
 * ส่ง Reply Message
 * @param {string} replyToken - Token สำหรับตอบกลับ
 * @param {Array|Object} messages - ข้อความที่จะส่ง
 * @returns {Promise}
 */
async function replyMessage(replyToken, messages) {
  // แปลง string เป็น message object ถ้าจำเป็น
  if (typeof messages === 'string') {
    messages = [{ type: 'text', text: messages }];
  }
  
  if (!Array.isArray(messages)) {
    messages = [messages];
  }
  
  // จำกัดไม่เกิน 5 ข้อความ
  if (messages.length > 5) {
    throw new Error('Cannot send more than 5 messages per reply');
  }
  
  try {
    const result = await client.replyMessage({
      replyToken,
      messages
    });
    
    console.log('Reply sent successfully:', result);
    return result;
  } catch (error) {
    console.error('Error sending reply:', error);
    throw error;
  }
}

/**
 * ส่ง Push Message
 * @param {string} to - userId, groupId, หรือ roomId
 * @param {Array|Object} messages - ข้อความที่จะส่ง
 * @returns {Promise}
 */
async function pushMessage(to, messages) {
  if (typeof messages === 'string') {
    messages = [{ type: 'text', text: messages }];
  }
  
  if (!Array.isArray(messages)) {
    messages = [messages];
  }
  
  if (messages.length > 5) {
    throw new Error('Cannot send more than 5 messages per push');
  }
  
  try {
    const result = await client.pushMessage({
      to,
      messages
    });
    
    console.log('Push message sent successfully');
    return result;
  } catch (error) {
    console.error('Error sending push message:', error);
    
    // Handle specific errors
    if (error.statusCode === 400) {
      throw new Error(`Bad Request: ${error.message}`);
    } else if (error.statusCode === 429) {
      throw new Error('Rate limit exceeded. Please retry later.');
    }
    
    throw error;
  }
}

/**
 * ส่ง Multicast Message
 * @param {Array} userIds - รายการ userId (สูงสุด 500)
 * @param {Array|Object} messages - ข้อความที่จะส่ง
 * @returns {Promise}
 */
async function multicastMessage(userIds, messages) {
  if (!Array.isArray(userIds) || userIds.length === 0) {
    throw new Error('userIds must be a non-empty array');
  }
  
  if (userIds.length > 500) {
    throw new Error('Cannot multicast to more than 500 users at once');
  }
  
  if (typeof messages === 'string') {
    messages = [{ type: 'text', text: messages }];
  }
  
  if (!Array.isArray(messages)) {
    messages = [messages];
  }
  
  try {
    const result = await client.multicast({
      to: userIds,
      messages
    });
    
    console.log(`Multicast sent to ${userIds.length} users`);
    return result;
  } catch (error) {
    console.error('Error sending multicast:', error);
    throw error;
  }
}

/**
 * ส่ง Multicast ขนาดใหญ่ (มากกว่า 500 คน)
 * @param {Array} userIds - รายการ userId ทั้งหมด
 * @param {Array} messages - ข้อความที่จะส่ง
 * @param {number} batchSize - ขนาด batch (default: 500)
 * @param {number} delayMs - หน่วงเวลาระหว่าง batch (default: 100ms)
 */
async function multicastLarge(userIds, messages, batchSize = 500, delayMs = 100) {
  const batches = [];
  
  for (let i = 0; i < userIds.length; i += batchSize) {
    batches.push(userIds.slice(i, i + batchSize));
  }
  
  console.log(`Sending to ${userIds.length} users in ${batches.length} batches`);
  
  const results = [];
  
  for (let i = 0; i < batches.length; i++) {
    const batch = batches[i];
    
    try {
      const result = await multicastMessage(batch, messages);
      results.push({ batchIndex: i, success: true, result });
      console.log(`Batch ${i + 1}/${batches.length} sent successfully`);
    } catch (error) {
      results.push({ batchIndex: i, success: false, error: error.message });
      console.error(`Batch ${i + 1}/${batches.length} failed:`, error.message);
    }
    
    // หน่วงเวลาระหว่าง batch
    if (i < batches.length - 1) {
      await new Promise(resolve => setTimeout(resolve, delayMs));
    }
  }
  
  return results;
}

/**
 * ส่ง Broadcast Message
 * @param {Array|Object} messages - ข้อความที่จะส่ง
 * @returns {Promise}
 */
async function broadcastMessage(messages) {
  if (typeof messages === 'string') {
    messages = [{ type: 'text', text: messages }];
  }
  
  if (!Array.isArray(messages)) {
    messages = [messages];
  }
  
  try {
    const result = await client.broadcast({
      messages
    });
    
    console.log('Broadcast message sent successfully');
    return result;
  } catch (error) {
    console.error('Error broadcasting message:', error);
    throw error;
  }
}

/**
 * ตรวจสอบโควต้าที่เหลือ
 */
async function checkMessageQuota() {
  try {
    const quota = await client.getMessageQuota();
    const consumption = await client.getMessageQuotaConsumption();
    
    return {
      type: quota.type,
      limit: quota.value,
      used: consumption.totalUsage,
      remaining: quota.type === 'limited' ? quota.value - consumption.totalUsage : 'unlimited'
    };
  } catch (error) {
    console.error('Error checking quota:', error);
    throw error;
  }
}

module.exports = {
  replyMessage,
  pushMessage,
  multicastMessage,
  multicastLarge,
  broadcastMessage,
  checkMessageQuota
};
```

### handlers/messageHandler.js - จัดการ Events

```javascript
'use strict';

const lineService = require('../services/lineService');
const userService = require('../services/userService');

/**
 * จัดการ Message Event
 */
async function handleMessageEvent(event) {
  const { replyToken, source, message } = event;
  const userId = source.userId;
  
  // บันทึก/อัปเดตข้อมูลผู้ใช้
  await userService.upsertUser(userId);
  
  switch (message.type) {
    case 'text':
      return handleTextMessage(event);
    case 'image':
      return handleImageMessage(event);
    case 'sticker':
      return handleStickerMessage(event);
    default:
      return lineService.replyMessage(replyToken, 'ขอโทษครับ ไม่รองรับประเภทข้อความนี้');
  }
}

/**
 * จัดการ Text Message
 */
async function handleTextMessage(event) {
  const { replyToken, message, source } = event;
  const text = message.text.toLowerCase();
  const userId = source.userId;
  
  // Command router
  if (text === 'สวัสดี' || text === 'hello' || text === 'hi') {
    return lineService.replyMessage(replyToken, [
      { type: 'text', text: '👋 สวัสดีครับ! ยินดีให้บริการ' },
      { type: 'text', text: 'พิมพ์ "ช่วยเหลือ" เพื่อดูคำสั่งที่รองรับ' }
    ]);
  }
  
  if (text === 'ช่วยเหลือ' || text === 'help') {
    return sendHelpMessage(replyToken);
  }
  
  if (text.startsWith('ส่งข้อความ ')) {
    const targetText = text.replace('ส่งข้อความ ', '');
    return schedulePushMessage(userId, targetText);
  }
  
  // Default: echo
  return lineService.replyMessage(replyToken, `คุณพิมพ์ว่า: "${message.text}"`);
}

/**
 * ส่ง Help Message
 */
async function sendHelpMessage(replyToken) {
  const helpText = `📖 คำสั่งที่รองรับ:

• สวัสดี - ทักทาย
• ช่วยเหลือ - แสดงคำสั่ง
• โควต้า - ตรวจสอบโควต้าที่เหลือ
• ส่งข้อความ [ข้อความ] - ทดสอบส่งข้อความ`;

  return lineService.replyMessage(replyToken, helpText);
}

/**
 * จัดการ Sticker Message
 */
async function handleStickerMessage(event) {
  const { replyToken } = event;
  
  return lineService.replyMessage(replyToken, [
    { type: 'text', text: '😊 ขอบคุณสำหรับ sticker น่ารัก!' }
  ]);
}

/**
 * จัดการ Image Message
 */
async function handleImageMessage(event) {
  const { replyToken } = event;
  
  return lineService.replyMessage(replyToken, 'ขอบคุณสำหรับรูปภาพ!');
}

/**
 * จัดการ Follow Event (มีคน Follow)
 */
async function handleFollowEvent(event) {
  const { replyToken, source } = event;
  const userId = source.userId;
  
  // บันทึก follower ใหม่
  await userService.addFollower(userId);
  
  // ส่งข้อความต้อนรับ
  return lineService.replyMessage(replyToken, [
    {
      type: 'text',
      text: '🎉 ยินดีต้อนรับสู่ LINE OA ของเรา!\n\nขอบคุณที่ติดตามนะครับ'
    },
    {
      type: 'text',
      text: 'พิมพ์ "ช่วยเหลือ" เพื่อดูคำสั่งที่รองรับ'
    }
  ]);
}

/**
 * จัดการ Unfollow Event (มีคน Unfollow)
 */
async function handleUnfollowEvent(event) {
  const { source } = event;
  const userId = source.userId;
  
  // อัปเดตสถานะ follower
  await userService.removeFollower(userId);
  
  console.log(`User ${userId} unfollowed`);
}

/**
 * ส่ง Push Message แบบ Schedule (ตัวอย่าง)
 */
async function schedulePushMessage(userId, text) {
  // หน่วงเวลา 2 วินาที แล้วส่ง push
  setTimeout(async () => {
    try {
      await lineService.pushMessage(userId, {
        type: 'text',
        text: `📬 ข้อความที่ถูกส่งมาหาคุณ: "${text}"`
      });
    } catch (error) {
      console.error('Failed to send scheduled push:', error);
    }
  }, 2000);
  
  return lineService.replyMessage(null, 'ข้อความถูกกำหนดเวลาส่งแล้ว!');
}

module.exports = {
  handleMessageEvent,
  handleFollowEvent,
  handleUnfollowEvent
};
```

### services/userService.js - จัดการข้อมูลผู้ใช้

```javascript
'use strict';

// ในตัวอย่างนี้ใช้ in-memory store
// ในโปรดักชั่นควรใช้ฐานข้อมูลจริง
const users = new Map();
const followers = new Set();

/**
 * เพิ่มหรืออัปเดตผู้ใช้
 */
async function upsertUser(userId) {
  if (!users.has(userId)) {
    users.set(userId, {
      userId,
      createdAt: new Date(),
      lastSeen: new Date()
    });
  } else {
    const user = users.get(userId);
    user.lastSeen = new Date();
  }
}

/**
 * บันทึก follower ใหม่
 */
async function addFollower(userId) {
  followers.add(userId);
  await upsertUser(userId);
  
  const user = users.get(userId);
  user.isFollower = true;
  user.followedAt = new Date();
}

/**
 * ลบ follower
 */
async function removeFollower(userId) {
  followers.delete(userId);
  
  if (users.has(userId)) {
    const user = users.get(userId);
    user.isFollower = false;
    user.unfollowedAt = new Date();
  }
}

/**
 * ดึงรายการ followers ทั้งหมด
 */
async function getAllFollowers() {
  return Array.from(followers);
}

/**
 * ดึง followers ตามกลุ่ม
 */
async function getFollowersBySegment(segment) {
  const allFollowers = Array.from(followers);
  
  // ตัวอย่าง: แบ่ง segment ตาม pattern
  switch (segment) {
    case 'active':
      return allFollowers.filter(userId => {
        const user = users.get(userId);
        const oneDayAgo = new Date(Date.now() - 24 * 60 * 60 * 1000);
        return user && user.lastSeen > oneDayAgo;
      });
    
    case 'inactive':
      return allFollowers.filter(userId => {
        const user = users.get(userId);
        const oneWeekAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
        return user && user.lastSeen < oneWeekAgo;
      });
    
    default:
      return allFollowers;
  }
}

module.exports = {
  upsertUser,
  addFollower,
  removeFollower,
  getAllFollowers,
  getFollowersBySegment
};
```

### ตัวอย่าง Broadcast สำหรับ Campaign

```javascript
'use strict';

const lineService = require('./services/lineService');
const userService = require('./services/userService');

/**
 * ส่งข้อความ Campaign ไปยัง followers ทั้งหมด
 */
async function sendCampaignBroadcast() {
  // ตรวจสอบโควต้าก่อนส่ง
  const quota = await lineService.checkMessageQuota();
  
  console.log('Current quota status:', quota);
  
  if (quota.type === 'limited' && quota.remaining < 1000) {
    console.warn('Low quota warning! Remaining:', quota.remaining);
    return;
  }
  
  // สร้างข้อความ campaign
  const campaignMessages = [
    {
      type: 'text',
      text: '🎉 โปรโมชั่นพิเศษ!\n\nลด 30% สำหรับทุกสินค้า\nวันนี้ถึง 31 ธ.ค. เท่านั้น!'
    },
    {
      type: 'image',
      originalContentUrl: 'https://example.com/sale-banner.jpg',
      previewImageUrl: 'https://example.com/sale-banner-preview.jpg'
    }
  ];
  
  // ส่ง Broadcast
  try {
    await lineService.broadcastMessage(campaignMessages);
    console.log('Campaign broadcast sent successfully!');
  } catch (error) {
    console.error('Campaign broadcast failed:', error);
    throw error;
  }
}

/**
 * ส่งข้อความ Targeted Campaign ไปยัง segment เฉพาะ
 */
async function sendTargetedCampaign(segment) {
  const targetUsers = await userService.getFollowersBySegment(segment);
  
  if (targetUsers.length === 0) {
    console.log('No users in segment:', segment);
    return;
  }
  
  console.log(`Sending to ${targetUsers.length} users in segment: ${segment}`);
  
  const messages = [
    {
      type: 'text',
      text: `สวัสดี! เรามีข้อเสนอพิเศษสำหรับกลุ่ม ${segment} ของเรา`
    }
  ];
  
  // ส่ง multicast แบบ batch
  const results = await lineService.multicastLarge(targetUsers, messages);
  
  const successful = results.filter(r => r.success).length;
  const failed = results.filter(r => !r.success).length;
  
  console.log(`Campaign complete: ${successful} batches succeeded, ${failed} failed`);
  
  return results;
}

// Export สำหรับใช้ใน scheduler หรือ API endpoint
module.exports = {
  sendCampaignBroadcast,
  sendTargetedCampaign
};
```

### ตัวอย่าง Scheduled Push Notifications

```javascript
'use strict';

const cron = require('node-cron');
const lineService = require('./services/lineService');
const userService = require('./services/userService');

/**
 * ส่ง Daily Reminder ทุกวันเวลา 9:00 น.
 */
cron.schedule('0 9 * * *', async () => {
  console.log('Running daily reminder push...');
  
  const activeUsers = await userService.getFollowersBySegment('active');
  
  if (activeUsers.length > 0) {
    const message = {
      type: 'text',
      text: `🌅 สวัสดีตอนเช้า!\n\nวันนี้มีอะไรใหม่ที่น่าสนใจบ้าง?\nลองเข้ามาดูกันนะครับ 😊`
    };
    
    await lineService.multicastLarge(activeUsers, [message]);
    console.log(`Daily reminder sent to ${activeUsers.length} users`);
  }
}, {
  timezone: 'Asia/Bangkok'
});

/**
 * ส่ง Weekly Summary ทุกวันศุกร์เวลา 17:00 น.
 */
cron.schedule('0 17 * * 5', async () => {
  console.log('Running weekly summary broadcast...');
  
  const summaryMessages = [
    {
      type: 'text',
      text: '📊 สรุปสัปดาห์นี้\n\n• สินค้าขายดี 5 อันดับ\n• โปรโมชั่นที่กำลังมา\n• ข่าวสารอัปเดต'
    }
  ];
  
  await lineService.broadcastMessage(summaryMessages);
  console.log('Weekly summary broadcast sent');
}, {
  timezone: 'Asia/Bangkok'
});
```

### ตัวอย่างเต็มสำหรับ Express App

```javascript
'use strict';

require('dotenv').config();
const express = require('express');
const line = require('@line/bot-sdk');
const { handleMessageEvent, handleFollowEvent, handleUnfollowEvent } = require('./handlers/messageHandler');
const lineService = require('./services/lineService');

const app = express();

const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};

const client = new line.messagingApi.MessagingApiClient({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

// Webhook endpoint
app.post('/webhook', line.middleware(config), async (req, res) => {
  try {
    await Promise.all(req.body.events.map(processEvent));
    res.json({ status: 'ok' });
  } catch (err) {
    console.error(err);
    res.status(500).end();
  }
});

// Process individual event
async function processEvent(event) {
  switch (event.type) {
    case 'message':
      return handleMessageEvent(event);
    case 'follow':
      return handleFollowEvent(event);
    case 'unfollow':
      return handleUnfollowEvent(event);
    case 'join':
      return handleJoinEvent(event);
    case 'leave':
      return handleLeaveEvent(event);
    case 'postback':
      return handlePostbackEvent(event);
    default:
      console.log('Unhandled event type:', event.type);
  }
}

// Handle join group/room
async function handleJoinEvent(event) {
  const { replyToken } = event;
  return lineService.replyMessage(replyToken, '👋 สวัสดีทุกคน! ขอบคุณที่เชิญเข้ากลุ่มนะครับ');
}

// Handle leave group/room  
function handleLeaveEvent(event) {
  console.log('Bot left:', event.source);
}

// Handle postback
async function handlePostbackEvent(event) {
  const { replyToken, postback } = event;
  const data = new URLSearchParams(postback.data);
  const action = data.get('action');
  
  switch (action) {
    case 'confirm_order':
      const orderId = data.get('orderId');
      return lineService.replyMessage(replyToken, `✅ ยืนยันคำสั่งซื้อ #${orderId} แล้ว`);
    
    default:
      return lineService.replyMessage(replyToken, 'รับทราบแล้วครับ');
  }
}

// Admin API endpoints
app.post('/admin/push', express.json(), async (req, res) => {
  const { userId, message } = req.body;
  
  try {
    await lineService.pushMessage(userId, message);
    res.json({ success: true });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.post('/admin/broadcast', express.json(), async (req, res) => {
  const { message } = req.body;
  
  try {
    await lineService.broadcastMessage(message);
    res.json({ success: true });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.get('/admin/quota', async (req, res) => {
  try {
    const quota = await lineService.checkMessageQuota();
    res.json(quota);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.listen(process.env.PORT || 3000, () => {
  console.log(`LINE Bot server running on port ${process.env.PORT || 3000}`);
});
```

---

## 10. Code Examples ใน Python

### การติดตั้ง Dependencies

```bash
pip install line-bot-sdk flask python-dotenv requests
```

### โครงสร้างโปรเจกต์ Python

```
linebot/
├── .env
├── app.py
├── handlers/
│   ├── __init__.py
│   ├── message_handler.py
│   └── event_handler.py
├── services/
│   ├── __init__.py
│   ├── line_service.py
│   └── user_service.py
└── utils/
    ├── __init__.py
    └── helpers.py
```

### .env

```env
LINE_CHANNEL_ACCESS_TOKEN=your_channel_access_token
LINE_CHANNEL_SECRET=your_channel_secret
PORT=3000
```

### services/line_service.py

```python
import os
import time
import logging
from typing import Union, List, Dict, Any, Optional
from linebot.v3 import WebhookHandler
from linebot.v3.messaging import (
    Configuration,
    ApiClient,
    MessagingApi,
    ReplyMessageRequest,
    PushMessageRequest,
    MulticastRequest,
    BroadcastRequest,
    TextMessage,
    Message
)
from linebot.v3.exceptions import InvalidSignatureError, ApiException
from dotenv import load_dotenv

load_dotenv()

logger = logging.getLogger(__name__)

# Configuration
configuration = Configuration(
    access_token=os.environ.get('LINE_CHANNEL_ACCESS_TOKEN')
)

def get_messaging_api() -> MessagingApi:
    """สร้าง MessagingApi instance"""
    api_client = ApiClient(configuration)
    return MessagingApi(api_client)


def reply_message(
    reply_token: str,
    messages: Union[str, Dict, List],
    notification_disabled: bool = False
) -> bool:
    """
    ส่ง Reply Message
    
    Args:
        reply_token: Token สำหรับตอบกลับ (จาก Webhook)
        messages: ข้อความที่จะส่ง (string, dict, หรือ list)
        notification_disabled: ปิดการแจ้งเตือน
    
    Returns:
        bool: True ถ้าสำเร็จ
    """
    # แปลง input เป็น list ของ Message objects
    message_objects = _normalize_messages(messages)
    
    if len(message_objects) > 5:
        raise ValueError("Cannot send more than 5 messages per reply")
    
    try:
        api = get_messaging_api()
        
        request = ReplyMessageRequest(
            reply_token=reply_token,
            messages=message_objects,
            notification_disabled=notification_disabled
        )
        
        api.reply_message(request)
        logger.info("Reply message sent successfully")
        return True
        
    except ApiException as e:
        logger.error(f"Error sending reply: {e.status} - {e.reason}")
        raise
    except Exception as e:
        logger.error(f"Unexpected error sending reply: {e}")
        raise


def push_message(
    to: str,
    messages: Union[str, Dict, List],
    notification_disabled: bool = False,
    custom_aggregation_units: Optional[List[str]] = None
) -> bool:
    """
    ส่ง Push Message
    
    Args:
        to: userId, groupId, หรือ roomId
        messages: ข้อความที่จะส่ง
        notification_disabled: ปิดการแจ้งเตือน
        custom_aggregation_units: หน่วยการรวมข้อมูล
    
    Returns:
        bool: True ถ้าสำเร็จ
    """
    message_objects = _normalize_messages(messages)
    
    if len(message_objects) > 5:
        raise ValueError("Cannot send more than 5 messages per push")
    
    try:
        api = get_messaging_api()
        
        request = PushMessageRequest(
            to=to,
            messages=message_objects,
            notification_disabled=notification_disabled
        )
        
        if custom_aggregation_units:
            request.custom_aggregation_units = custom_aggregation_units
        
        api.push_message(request)
        logger.info(f"Push message sent to {to}")
        return True
        
    except ApiException as e:
        if e.status == 429:
            logger.warning("Rate limit exceeded, retrying after delay...")
            time.sleep(1)
            return push_message(to, messages, notification_disabled)
        
        logger.error(f"Error sending push: {e.status} - {e.reason}")
        raise
    except Exception as e:
        logger.error(f"Unexpected error sending push: {e}")
        raise


def multicast_message(
    user_ids: List[str],
    messages: Union[str, Dict, List],
    notification_disabled: bool = False
) -> bool:
    """
    ส่ง Multicast Message
    
    Args:
        user_ids: รายการ userId (สูงสุด 500)
        messages: ข้อความที่จะส่ง
        notification_disabled: ปิดการแจ้งเตือน
    
    Returns:
        bool: True ถ้าสำเร็จ
    """
    if not user_ids:
        raise ValueError("user_ids must not be empty")
    
    if len(user_ids) > 500:
        raise ValueError("Cannot multicast to more than 500 users at once")
    
    message_objects = _normalize_messages(messages)
    
    try:
        api = get_messaging_api()
        
        request = MulticastRequest(
            to=user_ids,
            messages=message_objects,
            notification_disabled=notification_disabled
        )
        
        api.multicast(request)
        logger.info(f"Multicast sent to {len(user_ids)} users")
        return True
        
    except ApiException as e:
        logger.error(f"Error sending multicast: {e.status} - {e.reason}")
        raise


def multicast_large(
    user_ids: List[str],
    messages: Union[str, Dict, List],
    batch_size: int = 500,
    delay_seconds: float = 0.1
) -> List[Dict]:
    """
    ส่ง Multicast ขนาดใหญ่ (มากกว่า 500 คน)
    
    Args:
        user_ids: รายการ userId ทั้งหมด
        messages: ข้อความที่จะส่ง
        batch_size: ขนาด batch (default: 500)
        delay_seconds: หน่วงเวลาระหว่าง batch
    
    Returns:
        List[Dict]: ผลลัพธ์แต่ละ batch
    """
    # แบ่ง batches
    batches = [
        user_ids[i:i + batch_size]
        for i in range(0, len(user_ids), batch_size)
    ]
    
    logger.info(f"Sending to {len(user_ids)} users in {len(batches)} batches")
    
    results = []
    
    for i, batch in enumerate(batches):
        try:
            multicast_message(batch, messages)
            results.append({
                'batch_index': i,
                'success': True,
                'user_count': len(batch)
            })
            logger.info(f"Batch {i + 1}/{len(batches)} sent successfully")
        except Exception as e:
            results.append({
                'batch_index': i,
                'success': False,
                'error': str(e),
                'user_count': len(batch)
            })
            logger.error(f"Batch {i + 1}/{len(batches)} failed: {e}")
        
        # หน่วงเวลาระหว่าง batch
        if i < len(batches) - 1:
            time.sleep(delay_seconds)
    
    successful = sum(1 for r in results if r['success'])
    failed = len(results) - successful
    logger.info(f"Multicast complete: {successful} batches succeeded, {failed} failed")
    
    return results


def broadcast_message(
    messages: Union[str, Dict, List],
    notification_disabled: bool = False
) -> bool:
    """
    ส่ง Broadcast Message ไปยัง followers ทั้งหมด
    
    Args:
        messages: ข้อความที่จะส่ง
        notification_disabled: ปิดการแจ้งเตือน
    
    Returns:
        bool: True ถ้าสำเร็จ
    """
    message_objects = _normalize_messages(messages)
    
    try:
        api = get_messaging_api()
        
        request = BroadcastRequest(
            messages=message_objects,
            notification_disabled=notification_disabled
        )
        
        api.broadcast(request)
        logger.info("Broadcast message sent successfully")
        return True
        
    except ApiException as e:
        logger.error(f"Error broadcasting: {e.status} - {e.reason}")
        raise


def check_message_quota() -> Dict:
    """
    ตรวจสอบโควต้าที่เหลือ
    
    Returns:
        Dict: ข้อมูลโควต้า
    """
    try:
        api = get_messaging_api()
        
        quota = api.get_message_quota()
        consumption = api.get_message_quota_consumption()
        
        result = {
            'type': quota.type,
            'limit': quota.value if quota.type == 'limited' else None,
            'used': consumption.total_usage,
        }
        
        if quota.type == 'limited':
            result['remaining'] = quota.value - consumption.total_usage
        else:
            result['remaining'] = 'unlimited'
        
        return result
        
    except ApiException as e:
        logger.error(f"Error checking quota: {e.status} - {e.reason}")
        raise


def _normalize_messages(messages: Union[str, Dict, List]) -> List[Message]:
    """
    แปลง input หลายรูปแบบเป็น list ของ Message objects
    
    Args:
        messages: string, dict, Message object, หรือ list
    
    Returns:
        List[Message]: รายการ Message objects
    """
    if isinstance(messages, str):
        return [TextMessage(type='text', text=messages)]
    
    if isinstance(messages, dict):
        return [_dict_to_message(messages)]
    
    if isinstance(messages, Message):
        return [messages]
    
    if isinstance(messages, list):
        result = []
        for msg in messages:
            if isinstance(msg, str):
                result.append(TextMessage(type='text', text=msg))
            elif isinstance(msg, dict):
                result.append(_dict_to_message(msg))
            elif isinstance(msg, Message):
                result.append(msg)
            else:
                raise ValueError(f"Unknown message type: {type(msg)}")
        return result
    
    raise ValueError(f"Cannot convert {type(messages)} to messages")


def _dict_to_message(data: Dict) -> Message:
    """แปลง dict เป็น Message object"""
    from linebot.v3.messaging import (
        ImageMessage, VideoMessage, AudioMessage,
        LocationMessage, StickerMessage, FlexMessage
    )
    
    msg_type = data.get('type', 'text')
    
    if msg_type == 'text':
        return TextMessage(type='text', text=data['text'])
    elif msg_type == 'image':
        return ImageMessage(
            type='image',
            original_content_url=data['originalContentUrl'],
            preview_image_url=data['previewImageUrl']
        )
    elif msg_type == 'sticker':
        return StickerMessage(
            type='sticker',
            package_id=data['packageId'],
            sticker_id=data['stickerId']
        )
    else:
        raise ValueError(f"Unsupported message type: {msg_type}")
```

### app.py - Flask Application

```python
import os
import logging
from flask import Flask, request, abort, jsonify
from linebot.v3 import WebhookHandler
from linebot.v3.exceptions import InvalidSignatureError
from linebot.v3.webhooks import (
    MessageEvent, TextMessageContent,
    FollowEvent, UnfollowEvent,
    JoinEvent, LeaveEvent,
    PostbackEvent
)
from services.line_service import (
    reply_message, push_message, multicast_message,
    broadcast_message, multicast_large, check_message_quota
)
from dotenv import load_dotenv

load_dotenv()

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

app = Flask(__name__)

handler = WebhookHandler(os.environ.get('LINE_CHANNEL_SECRET'))


@app.route('/webhook', methods=['POST'])
def webhook():
    """LINE Webhook endpoint"""
    signature = request.headers.get('X-Line-Signature', '')
    body = request.get_data(as_text=True)
    
    logger.info(f"Webhook received: {body[:200]}...")
    
    try:
        handler.handle(body, signature)
    except InvalidSignatureError:
        logger.error("Invalid signature")
        abort(400)
    
    return 'OK'


@handler.add(MessageEvent, message=TextMessageContent)
def handle_text_message(event):
    """จัดการ Text Message Event"""
    text = event.message.text
    reply_token = event.reply_token
    
    logger.info(f"Text message from {event.source.user_id}: {text}")
    
    # Command routing
    if text in ['สวัสดี', 'hello', 'hi']:
        reply_message(reply_token, [
            '👋 สวัสดีครับ! ยินดีให้บริการ',
            'พิมพ์ "ช่วยเหลือ" เพื่อดูคำสั่ง'
        ])
    
    elif text in ['ช่วยเหลือ', 'help']:
        help_text = """📖 คำสั่งที่รองรับ:
        
• สวัสดี - ทักทาย
• ช่วยเหลือ - แสดงคำสั่ง
• โควต้า - ตรวจสอบโควต้า"""
        reply_message(reply_token, help_text)
    
    elif text == 'โควต้า':
        quota = check_message_quota()
        quota_text = f"""📊 สถานะโควต้า:
        
ประเภท: {quota['type']}
ใช้ไปแล้ว: {quota['used']:,}
คงเหลือ: {quota['remaining']:,}"""
        reply_message(reply_token, quota_text)
    
    else:
        # Echo
        reply_message(reply_token, f'คุณพิมพ์ว่า: "{text}"')


@handler.add(FollowEvent)
def handle_follow(event):
    """จัดการเมื่อมีคน Follow"""
    user_id = event.source.user_id
    logger.info(f"New follower: {user_id}")
    
    reply_message(event.reply_token, [
        '🎉 ยินดีต้อนรับ! ขอบคุณที่ติดตามเรานะครับ',
        'พิมพ์ "ช่วยเหลือ" เพื่อเริ่มต้นใช้งาน'
    ])


@handler.add(UnfollowEvent)
def handle_unfollow(event):
    """จัดการเมื่อมีคน Unfollow"""
    user_id = event.source.user_id
    logger.info(f"User unfollowed: {user_id}")


@handler.add(JoinEvent)
def handle_join(event):
    """จัดการเมื่อ Bot ถูกเพิ่มเข้ากลุ่ม"""
    reply_message(event.reply_token, '👋 สวัสดีทุกคน! ขอบคุณที่เชิญเข้ากลุ่มนะครับ')


# Admin API endpoints
@app.route('/admin/push', methods=['POST'])
def admin_push():
    """ส่ง Push Message ผ่าน Admin API"""
    data = request.get_json()
    user_id = data.get('userId')
    message = data.get('message')
    
    if not user_id or not message:
        return jsonify({'error': 'userId and message required'}), 400
    
    try:
        push_message(user_id, message)
        return jsonify({'success': True})
    except Exception as e:
        return jsonify({'error': str(e)}), 500


@app.route('/admin/broadcast', methods=['POST'])
def admin_broadcast():
    """ส่ง Broadcast Message ผ่าน Admin API"""
    data = request.get_json()
    message = data.get('message')
    
    if not message:
        return jsonify({'error': 'message required'}), 400
    
    try:
        broadcast_message(message)
        return jsonify({'success': True})
    except Exception as e:
        return jsonify({'error': str(e)}), 500


@app.route('/admin/quota')
def admin_quota():
    """ตรวจสอบโควต้า"""
    try:
        quota = check_message_quota()
        return jsonify(quota)
    except Exception as e:
        return jsonify({'error': str(e)}), 500


@app.route('/health')
def health():
    return jsonify({'status': 'healthy'})


if __name__ == '__main__':
    port = int(os.environ.get('PORT', 3000))
    app.run(host='0.0.0.0', port=port, debug=False)
```

---

## 11. Best Practices สำหรับการเลือกประเภทข้อความ

### Decision Tree สำหรับเลือก Message Type

```
ผู้ใช้ส่งข้อความมาก่อน?
├── ใช่ → ตอบได้ภายใน 30 วินาที?
│         ├── ใช่ → ใช้ Reply Message (ฟรี!)
│         └── ไม่ → ใช้ Push Message
└── ไม่ → ส่งหาหลายคน?
          ├── ใช่ → ส่งหาทุก follower?
          │         ├── ใช่ → ใช้ Broadcast Message
          │         └── ไม่ → จำนวนคน ≤ 500?
          │                   ├── ใช่ → ใช้ Multicast Message
          │                   └── ไม่ → ใช้ Multicast แบบ Batch
          └── ไม่ → ใช้ Push Message
```

### ตารางสรุป Best Practices

| สถานการณ์ | วิธีที่แนะนำ | เหตุผล |
|----------|------------|--------|
| Chatbot ตอบคำถาม | Reply | ฟรี, ทันที |
| ส่งใบแจ้งยอด | Push | ต้องการ userId เฉพาะ |
| โปรโมชั่นทั่วไป | Broadcast | ส่งทุกคน, ง่าย |
| แจ้งเตือน VIP members | Multicast | เลือกกลุ่มเป้าหมาย |
| Flash sale เฉพาะกลุ่ม | Multicast | Targeted marketing |

### Tips ประหยัดโควต้า

1. **ใช้ Reply แทน Push เสมอเมื่อเป็นไปได้**
2. **Batch Multicast อย่างชาญฉลาด** - ส่งเฉพาะ active users
3. **ตั้ง notificationDisabled=true** สำหรับข้อความที่ไม่สำคัญ
4. **ตรวจสอบโควต้าก่อนส่ง** เพื่อวางแผนการส่ง
5. **ใช้ Narrowcast** สำหรับ demographic targeting

---

## 12. Error Handling และ Retry Logic

### Error Codes ที่สำคัญ

| HTTP Status | Error Code | ความหมาย | วิธีแก้ |
|------------|-----------|---------|--------|
| 400 | 1101 | Invalid reply token | replyToken หมดอายุ |
| 400 | 1300 | Message count limit | ส่งเกิน 5 ข้อความ |
| 401 | - | Unauthorized | ตรวจสอบ Access Token |
| 429 | - | Too Many Requests | Retry after delay |
| 500 | - | Internal Server Error | Retry |

### Retry Logic ใน Node.js

```javascript
/**
 * ส่ง Push Message พร้อม Retry Logic
 */
async function pushMessageWithRetry(to, messages, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await pushMessage(to, messages);
    } catch (error) {
      const isLastAttempt = attempt === maxRetries;
      
      if (error.statusCode === 429) {
        // Rate limit - รอแล้วลองใหม่
        const delay = Math.pow(2, attempt) * 1000; // Exponential backoff
        console.warn(`Rate limit hit, waiting ${delay}ms before retry ${attempt}/${maxRetries}`);
        
        if (!isLastAttempt) {
          await new Promise(resolve => setTimeout(resolve, delay));
          continue;
        }
      }
      
      if (error.statusCode >= 500 && !isLastAttempt) {
        // Server error - ลองใหม่
        const delay = 1000 * attempt;
        console.warn(`Server error, retrying in ${delay}ms (${attempt}/${maxRetries})`);
        await new Promise(resolve => setTimeout(resolve, delay));
        continue;
      }
      
      // Error อื่น หรือหมด retry แล้ว
      console.error(`Failed after ${attempt} attempts:`, error);
      throw error;
    }
  }
}
```

### Retry Logic ใน Python

```python
import time
import functools
from typing import Callable, Any

def with_retry(max_retries: int = 3, base_delay: float = 1.0):
    """
    Decorator สำหรับ retry logic
    
    Args:
        max_retries: จำนวนครั้งที่ลองซ้ำสูงสุด
        base_delay: เวลาหน่วงฐาน (วินาที)
    """
    def decorator(func: Callable) -> Callable:
        @functools.wraps(func)
        def wrapper(*args, **kwargs) -> Any:
            last_error = None
            
            for attempt in range(1, max_retries + 1):
                try:
                    return func(*args, **kwargs)
                except ApiException as e:
                    last_error = e
                    
                    if e.status == 429:
                        # Rate limit
                        delay = base_delay * (2 ** attempt)
                        logger.warning(f"Rate limit, waiting {delay:.1f}s (attempt {attempt}/{max_retries})")
                    elif e.status >= 500:
                        # Server error
                        delay = base_delay * attempt
                        logger.warning(f"Server error {e.status}, retrying in {delay:.1f}s")
                    else:
                        # Client error - ไม่ retry
                        raise
                    
                    if attempt < max_retries:
                        time.sleep(delay)
                    else:
                        raise
            
            raise last_error
        
        return wrapper
    return decorator


@with_retry(max_retries=3)
def push_with_retry(to: str, messages) -> bool:
    return push_message(to, messages)
```

---

## 13. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Echo Bot พื้นฐาน
สร้าง Bot ที่:
- รับข้อความจากผู้ใช้
- ตอบกลับด้วย Reply Message โดย echo ข้อความกลับ
- แสดง timestamp ของข้อความ

### แบบฝึกหัดที่ 2: Scheduled Broadcast
สร้างระบบที่:
- ส่ง broadcast ทุกวันเวลา 9:00 น.
- ตรวจสอบโควต้าก่อนส่ง
- บันทึก log การส่ง

### แบบฝึกหัดที่ 3: Targeted Multicast
สร้างระบบที่:
- เก็บข้อมูล userId ของผู้ติดตาม
- แบ่ง segment เป็น active/inactive
- ส่ง multicast ไปยัง segment ที่เลือก

### แบบฝึกหัดที่ 4: Quota Monitor
สร้าง API endpoint ที่:
- ตรวจสอบโควต้าที่เหลือ
- แจ้งเตือนเมื่อโควต้าต่ำกว่า threshold
- แสดงกราฟการใช้โควต้า

### เฉลยแบบฝึกหัดที่ 1 (Node.js)

```javascript
'use strict';

require('dotenv').config();
const express = require('express');
const line = require('@line/bot-sdk');

const app = express();

const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};

const client = new line.messagingApi.MessagingApiClient({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

app.post('/webhook', line.middleware(config), async (req, res) => {
  try {
    await Promise.all(req.body.events.map(handleEvent));
    res.json({ status: 'ok' });
  } catch (err) {
    console.error(err);
    res.status(500).end();
  }
});

async function handleEvent(event) {
  if (event.type !== 'message' || event.message.type !== 'text') {
    return null;
  }
  
  const userMessage = event.message.text;
  const timestamp = new Date(event.timestamp).toLocaleString('th-TH', {
    timeZone: 'Asia/Bangkok'
  });
  
  const replyText = `📨 Echo: "${userMessage}"\n⏰ เวลา: ${timestamp}`;
  
  return client.replyMessage({
    replyToken: event.replyToken,
    messages: [{ type: 'text', text: replyText }]
  });
}

app.listen(3000, () => console.log('Echo bot running on port 3000'));
```

---

**สรุปบทเรียน:**

ในบทนี้เราได้เรียนรู้เกี่ยวกับประเภทต่างๆ ของการส่งข้อความใน LINE Messaging API:

1. **Reply Message** - ตอบกลับโดยไม่นับโควต้า แต่ต้องมี replyToken
2. **Push Message** - ส่งหาผู้ใช้คนใดก็ได้ แต่นับโควต้า
3. **Multicast** - ส่งหาหลายคนพร้อมกัน (สูงสุด 500 คน/call)
4. **Broadcast** - ส่งหา followers ทั้งหมด

การเลือกใช้ให้ถูกต้องจะช่วยประหยัดโควต้าและต้นทุนได้อย่างมาก
