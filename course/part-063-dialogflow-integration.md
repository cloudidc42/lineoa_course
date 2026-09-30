# Part 63: Dialogflow Integration กับ LINE Bot

## บทนำ

Dialogflow เป็นแพลตฟอร์ม Conversational AI ของ Google ที่มีความสามารถในการเข้าใจภาษาธรรมชาติ (NLU) ขั้นสูง และสามารถจัดการการสนทนาที่ซับซ้อนได้ บทนี้จะครอบคลุมการผสาน Dialogflow กับ LINE Bot ทั้งแบบ ES (Essentials) และ CX (Customer Experience) พร้อม Fulfillment Webhooks สมบูรณ์

## เนื้อหาที่จะเรียนรู้

1. Dialogflow ES vs CX
2. การสร้าง Dialogflow Agent
3. Intents และ Training Phrases
4. Entities
5. Contexts/Sessions
6. Fulfillment Webhooks
7. การเชื่อมต่อ Dialogflow กับ LINE
8. Multi-turn Conversations
9. Fallback Handling
10. Testing Conversation Flows

---

## 1. Dialogflow ES vs CX เปรียบเทียบ

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Dialogflow ES vs CX                                  │
├──────────────────┬────────────────────────┬────────────────────────────┤
│ คุณสมบัติ       │ Dialogflow ES          │ Dialogflow CX              │
├──────────────────┼────────────────────────┼────────────────────────────┤
│ ราคา            │ Free tier มี           │ Pay-as-you-go              │
│                  │ Paid plans             │                            │
├──────────────────┼────────────────────────┼────────────────────────────┤
│ Complexity       │ Simple-Medium          │ Complex Enterprise         │
├──────────────────┼────────────────────────┼────────────────────────────┤
│ Context          │ Contexts (string)      │ Pages & Flows              │
├──────────────────┼────────────────────────┼────────────────────────────┤
│ Conversation     │ Linear                 │ State Machine              │
│ Flow             │                        │                            │
├──────────────────┼────────────────────────┼────────────────────────────┤
│ Languages        │ 20+                    │ 50+                        │
├──────────────────┼────────────────────────┼────────────────────────────┤
│ Version Control  │ Limited                │ Full version control       │
├──────────────────┼────────────────────────┼────────────────────────────┤
│ Testing          │ Basic                  │ Advanced test cases        │
├──────────────────┼────────────────────────┼────────────────────────────┤
│ แนะนำสำหรับ    │ เริ่มต้น              │ Production Enterprise       │
└──────────────────┴────────────────────────┴────────────────────────────┘
```

### สถาปัตยกรรม LINE + Dialogflow

```
┌──────────────────────────────────────────────────────────────────────┐
│                         LINE Platform                                │
│  User ──► LINE App ──► LINE Server ──► Webhook                      │
└──────────────────────────────────────────────────────────────────────┘
                                  │
                                  ▼
┌──────────────────────────────────────────────────────────────────────┐
│                     Node.js Backend                                  │
│                                                                      │
│  ┌───────────────┐    ┌──────────────────┐    ┌──────────────────┐  │
│  │  LINE Webhook │───▶│ Message Handler  │───▶│ Dialogflow       │  │
│  │  Middleware   │    │                  │    │ Client           │  │
│  └───────────────┘    └──────────────────┘    └──────────────────┘  │
│                               │                        │             │
│                               ▼                        ▼             │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                  Session Manager                              │  │
│  │  Redis Cache: User Sessions & Contexts                        │  │
│  └───────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
                                  │
          ┌───────────────────────┤
          ▼                       ▼
┌─────────────────┐    ┌──────────────────────┐
│  Dialogflow     │    │  Fulfillment Server  │
│  ES/CX API      │    │  (Webhook)           │
└─────────────────┘    └──────────────────────┘
          │                       │
          ▼                       ▼
    Intent Match           Business Logic
    Entity Extract         Database
    Context Mgmt           External APIs
```

---

## 2. ติดตั้งและตั้งค่า

```bash
npm init -y

# Core dependencies
npm install @line/bot-sdk @google-cloud/dialogflow express dotenv

# Dialogflow CX (ถ้าใช้ CX)
npm install @google-cloud/dialogflow-cx

# Utilities
npm install redis uuid winston

# Development
npm install -D nodemon
```

```env
# .env
LINE_CHANNEL_ACCESS_TOKEN=your_token
LINE_CHANNEL_SECRET=your_secret

# Dialogflow ES
DIALOGFLOW_PROJECT_ID=your-project-id
DIALOGFLOW_LANGUAGE_CODE=th
GOOGLE_APPLICATION_CREDENTIALS=./credentials/google-credentials.json

# Dialogflow CX (optional)
DIALOGFLOW_CX_AGENT_ID=your-cx-agent-id
DIALOGFLOW_CX_LOCATION=asia-southeast1

# Redis
REDIS_URL=redis://localhost:6379

PORT=3000
```

---

## 3. Dialogflow ES Integration

### 3.1 Dialogflow ES Client

```javascript
// src/services/dialogflowES.js
const dialogflow = require('@google-cloud/dialogflow');
const { v4: uuidv4 } = require('uuid');
const config = require('../config');
const logger = require('../utils/logger');

class DialogflowESService {
  constructor() {
    this.projectId = process.env.DIALOGFLOW_PROJECT_ID;
    this.languageCode = process.env.DIALOGFLOW_LANGUAGE_CODE || 'th';

    this.sessionClient = new dialogflow.SessionsClient({
      keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
    });
  }

  // สร้าง Session Path
  getSessionPath(sessionId) {
    return this.sessionClient.projectAgentSessionPath(
      this.projectId,
      sessionId
    );
  }

  // ส่งข้อความไปยัง Dialogflow ES
  async detectIntent(sessionId, text, contexts = []) {
    const sessionPath = this.getSessionPath(sessionId);

    const request = {
      session: sessionPath,
      queryInput: {
        text: {
          text,
          languageCode: this.languageCode,
        },
      },
      queryParams: {
        contexts: contexts.map((ctx) => ({
          name: `${sessionPath}/contexts/${ctx.name}`,
          lifespanCount: ctx.lifespan || 5,
          parameters: ctx.parameters
            ? this.structToObject(ctx.parameters)
            : undefined,
        })),
      },
    };

    try {
      const [response] = await this.sessionClient.detectIntent(request);
      const queryResult = response.queryResult;

      return this.parseQueryResult(queryResult, sessionPath);
    } catch (error) {
      logger.error('Dialogflow ES error', {
        error: error.message,
        sessionId,
        text: text.substring(0, 100),
      });
      throw error;
    }
  }

  // Parse ผลลัพธ์
  parseQueryResult(queryResult, sessionPath) {
    const parameters = queryResult.parameters
      ? this.structToObject(queryResult.parameters)
      : {};

    return {
      intent: queryResult.intent?.displayName || 'Default Fallback Intent',
      confidence: queryResult.intentDetectionConfidence,
      fulfillmentText: queryResult.fulfillmentText,
      fulfillmentMessages: queryResult.fulfillmentMessages,
      parameters,
      outputContexts: queryResult.outputContexts?.map((ctx) => ({
        name: ctx.name.split('/').pop(),
        lifespan: ctx.lifespanCount,
        parameters: ctx.parameters ? this.structToObject(ctx.parameters) : {},
      })) || [],
      allRequiredParamsPresent: queryResult.allRequiredParamsPresent,
      queryText: queryResult.queryText,
      action: queryResult.action,
      webhookState: queryResult.webhookState,
    };
  }

  // แปลง Google Struct เป็น JS Object
  structToObject(struct) {
    if (!struct || !struct.fields) return {};

    const obj = {};
    for (const [key, value] of Object.entries(struct.fields)) {
      obj[key] = this.valueToJS(value);
    }
    return obj;
  }

  valueToJS(value) {
    if (value.stringValue !== undefined) return value.stringValue;
    if (value.numberValue !== undefined) return value.numberValue;
    if (value.boolValue !== undefined) return value.boolValue;
    if (value.nullValue !== undefined) return null;
    if (value.listValue) {
      return value.listValue.values.map((v) => this.valueToJS(v));
    }
    if (value.structValue) return this.structToObject(value.structValue);
    return null;
  }

  // ส่ง Event ไปยัง Dialogflow
  async triggerEvent(sessionId, eventName, parameters = {}) {
    const sessionPath = this.getSessionPath(sessionId);

    const request = {
      session: sessionPath,
      queryInput: {
        event: {
          name: eventName,
          parameters: this.objectToStruct(parameters),
          languageCode: this.languageCode,
        },
      },
    };

    const [response] = await this.sessionClient.detectIntent(request);
    return this.parseQueryResult(response.queryResult, sessionPath);
  }

  objectToStruct(obj) {
    if (!obj) return null;
    const fields = {};
    for (const [key, value] of Object.entries(obj)) {
      fields[key] = this.jsToValue(value);
    }
    return { fields };
  }

  jsToValue(value) {
    if (typeof value === 'string') return { stringValue: value };
    if (typeof value === 'number') return { numberValue: value };
    if (typeof value === 'boolean') return { boolValue: value };
    if (value === null) return { nullValue: 0 };
    if (Array.isArray(value)) {
      return { listValue: { values: value.map((v) => this.jsToValue(v)) } };
    }
    if (typeof value === 'object') {
      return { structValue: this.objectToStruct(value) };
    }
    return { nullValue: 0 };
  }
}

module.exports = new DialogflowESService();
```

### 3.2 Session Manager กับ Redis

```javascript
// src/services/sessionManager.js
const Redis = require('ioredis');
const { v4: uuidv4 } = require('uuid');
const logger = require('../utils/logger');

class SessionManager {
  constructor() {
    this.redis = new Redis(process.env.REDIS_URL);
    this.SESSION_TTL = 30 * 60; // 30 นาที
    this.prefix = 'dialogflow:session:';
  }

  // รับหรือสร้าง Dialogflow Session ID
  async getSessionId(lineUserId) {
    const key = `${this.prefix}${lineUserId}`;
    let sessionId = await this.redis.get(key);

    if (!sessionId) {
      // Dialogflow session ID ต้องเป็น unique per conversation
      sessionId = uuidv4();
      await this.redis.setex(key, this.SESSION_TTL, sessionId);
      logger.debug('New Dialogflow session', { lineUserId, sessionId });
    } else {
      await this.redis.expire(key, this.SESSION_TTL);
    }

    return sessionId;
  }

  // รีเซ็ต Session
  async resetSession(lineUserId) {
    const key = `${this.prefix}${lineUserId}`;
    await this.redis.del(key);
    const newSessionId = uuidv4();
    await this.redis.setex(key, this.SESSION_TTL, newSessionId);
    return newSessionId;
  }

  // บันทึก Context ใน Redis
  async saveContext(lineUserId, contexts) {
    const key = `${this.prefix}context:${lineUserId}`;
    await this.redis.setex(key, this.SESSION_TTL, JSON.stringify(contexts));
  }

  // โหลด Context
  async loadContext(lineUserId) {
    const key = `${this.prefix}context:${lineUserId}`;
    const data = await this.redis.get(key);
    return data ? JSON.parse(data) : [];
  }

  // บันทึก Conversation State
  async saveState(lineUserId, state) {
    const key = `${this.prefix}state:${lineUserId}`;
    await this.redis.setex(key, this.SESSION_TTL, JSON.stringify(state));
  }

  // โหลด State
  async loadState(lineUserId) {
    const key = `${this.prefix}state:${lineUserId}`;
    const data = await this.redis.get(key);
    return data ? JSON.parse(data) : {};
  }
}

module.exports = new SessionManager();
```

---

## 4. Fulfillment Webhook Server

```javascript
// src/fulfillment/webhookServer.js
const express = require('express');
const { WebhookClient, Card, Suggestion, Image } = require('dialogflow-fulfillment');
const orderHandler = require('./handlers/orderHandler');
const inquiryHandler = require('./handlers/inquiryHandler');
const appointmentHandler = require('./handlers/appointmentHandler');
const logger = require('../utils/logger');

const router = express.Router();

router.post('/webhook', express.json(), async (req, res) => {
  const agent = new WebhookClient({ request: req, response: res });

  logger.info('Fulfillment webhook called', {
    intent: req.body.queryResult?.intent?.displayName,
    session: req.body.session,
  });

  // Map intents to handlers
  const intentHandlerMap = {
    'order.start': (agent) => orderHandler.startOrder(agent),
    'order.confirm': (agent) => orderHandler.confirmOrder(agent),
    'order.cancel': (agent) => orderHandler.cancelOrder(agent),
    'order.status': (agent) => orderHandler.checkStatus(agent),
    'inquiry.menu': (agent) => inquiryHandler.showMenu(agent),
    'inquiry.price': (agent) => inquiryHandler.showPrice(agent),
    'inquiry.location': (agent) => inquiryHandler.showLocation(agent),
    'inquiry.hours': (agent) => inquiryHandler.showHours(agent),
    'appointment.book': (agent) => appointmentHandler.bookAppointment(agent),
    'appointment.cancel': (agent) => appointmentHandler.cancelAppointment(agent),
    'appointment.check': (agent) => appointmentHandler.checkAppointment(agent),
    'Default Fallback Intent': (agent) => handleFallback(agent),
  };

  agent.handleRequest(intentHandlerMap);
});

function handleFallback(agent) {
  const responses = [
    'ขออภัยครับ ไม่เข้าใจครับ กรุณาลองถามใหม่',
    'ขออภัยครับ ช่วยอธิบายเพิ่มเติมได้ไหมครับ?',
    'ไม่แน่ใจว่าต้องการอะไร กรุณาพิมพ์คำถามใหม่ครับ',
  ];

  const response = responses[Math.floor(Math.random() * responses.length)];
  agent.add(response);

  // เพิ่ม Quick Replies
  agent.add(new Suggestion('ดูเมนู'));
  agent.add(new Suggestion('ราคาสินค้า'));
  agent.add(new Suggestion('ที่ตั้งร้าน'));
  agent.add(new Suggestion('เวลาทำการ'));
}

module.exports = router;
```

### 4.1 Order Handler

```javascript
// src/fulfillment/handlers/orderHandler.js
const { Card, Suggestion, Image, Payload } = require('dialogflow-fulfillment');
const OrderService = require('../../services/orderService');
const logger = require('../../utils/logger');

class OrderHandler {
  async startOrder(agent) {
    const { product, quantity } = agent.parameters;

    if (!product) {
      agent.add('ต้องการสั่งอะไรครับ? กรุณาระบุชื่อสินค้า');
      agent.add(new Suggestion('กาแฟลาเต้'));
      agent.add(new Suggestion('ชาเย็น'));
      agent.add(new Suggestion('น้ำส้ม'));
      return;
    }

    // ค้นหาสินค้า
    const productInfo = await OrderService.findProduct(product);

    if (!productInfo) {
      agent.add(`ขออภัยครับ ไม่พบสินค้า "${product}" ในระบบ`);
      agent.add(new Suggestion('ดูเมนูทั้งหมด'));
      return;
    }

    // แสดงข้อมูลสินค้าและยืนยัน
    const qty = quantity || 1;

    agent.add(new Card({
      title: productInfo.name,
      imageUrl: productInfo.imageUrl,
      text: `ราคา: ${productInfo.price} บาท\nจำนวน: ${qty} ชิ้น\nรวม: ${productInfo.price * qty} บาท`,
      buttonText: 'ยืนยันการสั่ง',
      buttonUrl: 'https://example.com/confirm',
    }));

    agent.add(`ยืนยันการสั่ง ${productInfo.name} x${qty} ราคา ${productInfo.price * qty} บาท ไหมครับ?`);
    agent.add(new Suggestion('ยืนยัน'));
    agent.add(new Suggestion('ยกเลิก'));

    // Set context สำหรับ step ถัดไป
    agent.setContext({
      name: 'order-confirm',
      lifespan: 3,
      parameters: {
        product: productInfo.name,
        productId: productInfo.id,
        price: productInfo.price,
        quantity: qty,
        total: productInfo.price * qty,
      },
    });
  }

  async confirmOrder(agent) {
    const context = agent.getContext('order-confirm');

    if (!context) {
      agent.add('ขออภัยครับ ไม่พบข้อมูลการสั่ง กรุณาเริ่มใหม่');
      return;
    }

    const { product, productId, price, quantity, total } = context.parameters;

    try {
      // บันทึก Order
      const order = await OrderService.createOrder({
        productId,
        product,
        price,
        quantity,
        total,
        userId: agent.session.split('/').pop(),
      });

      agent.add(`✅ สั่ง ${product} x${quantity} สำเร็จแล้วครับ!\n\nหมายเลขออเดอร์: ${order.id}\nราคารวม: ${total} บาท\n\nจะมีการแจ้งเตือนเมื่อสินค้าพร้อมครับ`);

      // ล้าง Context
      agent.clearContext('order-confirm');
    } catch (error) {
      logger.error('Order creation error', { error: error.message });
      agent.add('ขออภัยครับ เกิดข้อผิดพลาดในการสั่ง กรุณาลองใหม่');
    }
  }

  async cancelOrder(agent) {
    const context = agent.getContext('order-confirm');

    if (context) {
      agent.clearContext('order-confirm');
    }

    agent.add('ยกเลิกการสั่งแล้วครับ มีอะไรให้ช่วยอีกไหมครับ?');
    agent.add(new Suggestion('ดูเมนู'));
    agent.add(new Suggestion('ติดต่อพนักงาน'));
  }

  async checkStatus(agent) {
    const { orderId } = agent.parameters;

    if (!orderId) {
      agent.add('กรุณาระบุหมายเลขออเดอร์ครับ');
      return;
    }

    try {
      const order = await OrderService.getOrder(orderId);

      if (!order) {
        agent.add(`ไม่พบออเดอร์ ${orderId} ครับ`);
        return;
      }

      const statusMap = {
        pending: 'รอดำเนินการ',
        preparing: 'กำลังเตรียม',
        ready: 'พร้อมรับ',
        delivered: 'ส่งแล้ว',
        cancelled: 'ยกเลิกแล้ว',
      };

      agent.add(`📦 ออเดอร์ ${orderId}\nสถานะ: ${statusMap[order.status] || order.status}`);
    } catch (error) {
      agent.add('ขออภัยครับ ไม่สามารถตรวจสอบสถานะได้');
    }
  }
}

module.exports = new OrderHandler();
```

### 4.2 Inquiry Handler

```javascript
// src/fulfillment/handlers/inquiryHandler.js
const { Card, Suggestion, List, Image } = require('dialogflow-fulfillment');

class InquiryHandler {
  async showMenu(agent) {
    // ส่งเมนูในรูปแบบ List
    agent.add('เมนูของเราครับ:');

    const menu = [
      { name: 'กาแฟลาเต้', price: 65, emoji: '☕' },
      { name: 'ชาเย็น', price: 45, emoji: '🧋' },
      { name: 'น้ำส้มคั้นสด', price: 55, emoji: '🍊' },
      { name: 'เค้กช็อกโกแลต', price: 85, emoji: '🎂' },
    ];

    const menuText = menu
      .map((item) => `${item.emoji} ${item.name} - ${item.price} บาท`)
      .join('\n');

    agent.add(menuText);
    agent.add(new Suggestion('สั่งกาแฟลาเต้'));
    agent.add(new Suggestion('สั่งชาเย็น'));
    agent.add(new Suggestion('ดูราคา'));
  }

  async showPrice(agent) {
    const { product } = agent.parameters;

    const prices = {
      'กาแฟ': 65,
      'กาแฟลาเต้': 65,
      'ชาเย็น': 45,
      'น้ำส้ม': 55,
      'เค้ก': 85,
    };

    if (product && prices[product]) {
      agent.add(`${product} ราคา ${prices[product]} บาทครับ`);
    } else {
      agent.add('ราคาสินค้าของเราครับ:\n');
      agent.add(Object.entries(prices)
        .map(([name, price]) => `• ${name}: ${price} บาท`)
        .join('\n'));
    }
  }

  async showLocation(agent) {
    agent.add(`📍 ที่ตั้งของเรา:
      
ร้าน Coffee House
123/45 ถนนสุขุมวิท ซอย 11
แขวงคลองเตยเหนือ เขตวัฒนา
กรุงเทพฯ 10110

📞 โทร: 02-xxx-xxxx
🚇 BTS อโศก ออก 3 เดินประมาณ 5 นาที

🗺️ ดูแผนที่: https://maps.google.com/?q=...`);

    agent.add(new Suggestion('เส้นทาง BTS'));
    agent.add(new Suggestion('เส้นทาง MRT'));
    agent.add(new Suggestion('โทรหาร้าน'));
  }

  async showHours(agent) {
    const now = new Date();
    const hour = now.getHours();
    const day = now.getDay(); // 0=Sunday, 6=Saturday

    const isOpen = (day >= 1 && day <= 5 && hour >= 9 && hour < 18) ||
      (day === 6 && hour >= 10 && hour < 17);

    const status = isOpen ? '🟢 เปิดอยู่ขณะนี้' : '🔴 ปิดอยู่ขณะนี้';

    agent.add(`⏰ เวลาทำการ:

${status}

จันทร์-ศุกร์: 09:00-18:00 น.
เสาร์: 10:00-17:00 น.
อาทิตย์: ปิดทำการ
วันหยุดนักขัตฤกษ์: ปิดทำการ`);
  }
}

module.exports = new InquiryHandler();
```

---

## 5. Dialogflow CX Integration

```javascript
// src/services/dialogflowCX.js
const { SessionsClient } = require('@google-cloud/dialogflow-cx');
const { v4: uuidv4 } = require('uuid');
const logger = require('../utils/logger');

class DialogflowCXService {
  constructor() {
    this.projectId = process.env.DIALOGFLOW_PROJECT_ID;
    this.location = process.env.DIALOGFLOW_CX_LOCATION || 'asia-southeast1';
    this.agentId = process.env.DIALOGFLOW_CX_AGENT_ID;
    this.languageCode = process.env.DIALOGFLOW_LANGUAGE_CODE || 'th';

    this.client = new SessionsClient({
      apiEndpoint: `${this.location}-dialogflow.googleapis.com`,
      keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
    });
  }

  // สร้าง Session Path
  getSessionPath(sessionId) {
    return this.client.projectLocationAgentSessionPath(
      this.projectId,
      this.location,
      this.agentId,
      sessionId
    );
  }

  // ส่งข้อความไปยัง Dialogflow CX
  async detectIntent(sessionId, text, additionalParams = {}) {
    const sessionPath = this.getSessionPath(sessionId);

    const request = {
      session: sessionPath,
      queryInput: {
        text: {
          text,
        },
        languageCode: this.languageCode,
      },
      queryParams: {
        ...additionalParams,
      },
    };

    try {
      const [response] = await this.client.detectIntent(request);
      return this.parseResponse(response);
    } catch (error) {
      logger.error('Dialogflow CX error', {
        error: error.message,
        sessionId,
      });
      throw error;
    }
  }

  // Parse Response
  parseResponse(response) {
    const queryResult = response.queryResult;

    return {
      responseMessages: queryResult.responseMessages?.map((msg) => {
        if (msg.text) return { type: 'text', texts: msg.text.text };
        if (msg.payload) return { type: 'payload', payload: msg.payload };
        return msg;
      }) || [],
      currentPage: {
        name: queryResult.currentPage?.name,
        displayName: queryResult.currentPage?.displayName,
      },
      intent: {
        name: queryResult.match?.intent?.name,
        displayName: queryResult.match?.intent?.displayName,
        confidence: queryResult.match?.confidence,
        matchType: queryResult.match?.matchType,
      },
      parameters: queryResult.parameters?.fields || {},
      sessionInfo: queryResult.sessionInfo,
      diagnosticInfo: response.queryResult.diagnosticInfo,
    };
  }

  // ส่ง Event
  async triggerEvent(sessionId, event) {
    const sessionPath = this.getSessionPath(sessionId);

    const request = {
      session: sessionPath,
      queryInput: {
        event: { event },
        languageCode: this.languageCode,
      },
    };

    const [response] = await this.client.detectIntent(request);
    return this.parseResponse(response);
  }

  // DTMF Input (สำหรับ Voice)
  async sendDTMF(sessionId, digits) {
    const sessionPath = this.getSessionPath(sessionId);

    const request = {
      session: sessionPath,
      queryInput: {
        dtmf: {
          digits,
          finishDigit: '#',
        },
        languageCode: this.languageCode,
      },
    };

    const [response] = await this.client.detectIntent(request);
    return this.parseResponse(response);
  }
}

module.exports = new DialogflowCXService();
```

---

## 6. LINE Bot Handler หลัก

```javascript
// src/handlers/lineBotHandler.js
const line = require('@line/bot-sdk');
const dialogflowES = require('../services/dialogflowES');
const sessionManager = require('../services/sessionManager');
const logger = require('../utils/logger');

class LineBotHandler {
  constructor() {
    this.lineClient = new line.messagingApi.MessagingApiClient({
      channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
    });
  }

  async handleEvent(event) {
    const userId = event.source.userId;

    try {
      if (event.type === 'message' && event.message.type === 'text') {
        await this.handleTextMessage(event, userId);
      } else if (event.type === 'follow') {
        await this.handleFollow(event, userId);
      } else if (event.type === 'postback') {
        await this.handlePostback(event, userId);
      }
    } catch (error) {
      logger.error('Event handling error', { error: error.message, userId });
    }
  }

  async handleTextMessage(event, userId) {
    const text = event.message.text.trim();
    const replyToken = event.replyToken;

    // ตรวจสอบคำสั่งพิเศษ
    if (text === '/reset') {
      await sessionManager.resetSession(userId);
      await this.replyText(replyToken, '✅ รีเซ็ตการสนทนาแล้วครับ เริ่มใหม่ได้เลย!');
      return;
    }

    // แสดง Loading
    await this.lineClient.showLoadingAnimation({ chatId: userId, loadingSeconds: 5 });

    // รับ Session ID
    const sessionId = await sessionManager.getSessionId(userId);

    try {
      // ส่งไปยัง Dialogflow
      const result = await dialogflowES.detectIntent(sessionId, text);

      logger.info('Dialogflow response', {
        userId,
        intent: result.intent,
        confidence: result.confidence,
        action: result.action,
      });

      // บันทึก Context
      await sessionManager.saveContext(userId, result.outputContexts);

      // สร้าง LINE Messages จาก Dialogflow response
      const messages = await this.buildLineMessages(result, userId);

      if (messages.length > 0) {
        await this.lineClient.replyMessage({
          replyToken,
          messages,
        });
      }
    } catch (error) {
      logger.error('Dialogflow error', { error: error.message, userId });
      await this.replyText(
        replyToken,
        'ขออภัยครับ เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง'
      );
    }
  }

  // สร้าง LINE Messages จาก Dialogflow Result
  async buildLineMessages(result, userId) {
    const messages = [];

    // Text Response
    if (result.fulfillmentText) {
      messages.push({ type: 'text', text: result.fulfillmentText });
    }

    // Fulfillment Messages (Rich responses)
    if (result.fulfillmentMessages) {
      for (const msg of result.fulfillmentMessages) {
        if (msg.platform === 'LINE' || !msg.platform) {
          const lineMsg = this.convertToLineMessage(msg);
          if (lineMsg) messages.push(lineMsg);
        }
      }
    }

    // Quick Replies
    const quickReplies = this.extractQuickReplies(result);
    if (quickReplies.length > 0 && messages.length > 0) {
      // เพิ่ม Quick Reply ไปที่ข้อความสุดท้าย
      const lastMsg = messages[messages.length - 1];
      if (lastMsg.type === 'text') {
        lastMsg.quickReply = {
          items: quickReplies.map((qr) => ({
            type: 'action',
            action: {
              type: 'message',
              label: qr,
              text: qr,
            },
          })),
        };
      }
    }

    // ถ้าไม่มีข้อความ ใส่ fallback
    if (messages.length === 0) {
      messages.push({
        type: 'text',
        text: 'ขออภัยครับ ไม่เข้าใจคำถาม กรุณาลองใหม่',
      });
    }

    return messages.slice(0, 5); // LINE รับสูงสุด 5 messages
  }

  // แปลง Dialogflow message เป็น LINE message
  convertToLineMessage(dfMessage) {
    if (dfMessage.text) {
      const text = dfMessage.text.text?.[0] || '';
      if (!text) return null;
      return { type: 'text', text };
    }

    if (dfMessage.card) {
      const card = dfMessage.card;
      return {
        type: 'template',
        altText: card.title || 'Card',
        template: {
          type: 'buttons',
          title: card.title,
          text: card.subtitle || 'กรุณาเลือก',
          thumbnailImageUrl: card.imageUri,
          actions: (card.buttons || []).map((btn) => ({
            type: 'uri',
            label: btn.text,
            uri: btn.postback || btn.text,
          })),
        },
      };
    }

    if (dfMessage.quickReplies) {
      return null; // จัดการแยกใน Quick Replies
    }

    return null;
  }

  // ดึง Quick Replies
  extractQuickReplies(result) {
    const quickReplies = [];

    if (result.fulfillmentMessages) {
      for (const msg of result.fulfillmentMessages) {
        if (msg.quickReplies?.quickReplies) {
          quickReplies.push(...msg.quickReplies.quickReplies);
        }
      }
    }

    return quickReplies.slice(0, 13); // LINE Quick Reply สูงสุด 13
  }

  async handleFollow(event, userId) {
    // Trigger welcome event ใน Dialogflow
    const sessionId = await sessionManager.getSessionId(userId);
    const result = await dialogflowES.triggerEvent(sessionId, 'WELCOME');

    const welcomeText = result.fulfillmentText ||
      'สวัสดีครับ! ยินดีต้อนรับ มีอะไรให้ช่วยไหมครับ? 😊';

    await this.lineClient.replyMessage({
      replyToken: event.replyToken,
      messages: [{ type: 'text', text: welcomeText }],
    });
  }

  async handlePostback(event, userId) {
    const data = new URLSearchParams(event.postback.data);
    const action = data.get('action');
    const payload = data.get('payload');

    const sessionId = await sessionManager.getSessionId(userId);
    const result = await dialogflowES.detectIntent(
      sessionId,
      payload || action || 'postback'
    );

    const messages = await this.buildLineMessages(result, userId);

    if (messages.length > 0) {
      await this.lineClient.replyMessage({
        replyToken: event.replyToken,
        messages,
      });
    }
  }

  async replyText(replyToken, text) {
    return this.lineClient.replyMessage({
      replyToken,
      messages: [{ type: 'text', text }],
    });
  }
}

module.exports = new LineBotHandler();
```

---

## 7. Multi-turn Conversation Flow

### 7.1 Sequence Diagram

```
ผู้ใช้           LINE         Node.js       Dialogflow     Database
  │               │              │               │              │
  │──"สั่งกาแฟ"──►│              │               │              │
  │               │──Webhook────►│               │              │
  │               │              │──detectIntent►│              │
  │               │              │◄─Intent:      │              │
  │               │              │  order.start  │              │
  │               │              │──"กาแฟอะไร?"►│              │
  │◄──"กาแฟอะไร?"─│              │               │              │
  │               │              │               │              │
  │──"ลาเต้"──────►│              │               │              │
  │               │──Webhook────►│               │              │
  │               │              │──detectIntent►│              │
  │               │              │  (ctx: order-start)         │
  │               │              │◄─Intent:      │              │
  │               │              │  order.coffee │              │
  │               │              │──Fulfillment──────────────►  │
  │               │              │◄─"ยืนยัน?"──────────────── │
  │◄──"ยืนยัน?"──│              │               │              │
  │               │              │               │              │
  │──"ยืนยัน"────►│              │               │              │
  │               │──Webhook────►│               │              │
  │               │              │──detectIntent►│              │
  │               │              │  (ctx: order-confirm)        │
  │               │              │◄─Intent:      │              │
  │               │              │  order.confirm│              │
  │               │              │──Create Order─────────────► │
  │               │              │◄─Order ID─────────────────  │
  │◄──"สั่งแล้ว"─│              │               │              │
```

### 7.2 Context-aware Handler

```javascript
// src/handlers/contextAwareHandler.js

class ContextAwareHandler {
  constructor(dialogflowService, sessionManager) {
    this.df = dialogflowService;
    this.sm = sessionManager;
  }

  async process(userId, text) {
    const sessionId = await this.sm.getSessionId(userId);
    const currentState = await this.sm.loadState(userId);

    // ส่งข้อความพร้อม Context
    const result = await this.df.detectIntent(sessionId, text);

    // อัพเดท State ตาม Intent
    const newState = await this.updateState(currentState, result, userId);
    await this.sm.saveState(userId, newState);

    return result;
  }

  async updateState(currentState, dfResult, userId) {
    const { intent, parameters, outputContexts } = dfResult;

    const newState = { ...currentState };

    switch (intent) {
      case 'order.start':
        newState.inOrderFlow = true;
        newState.orderStep = 'product_selection';
        break;

      case 'order.coffee':
        if (newState.inOrderFlow) {
          newState.orderStep = 'confirmation';
          newState.pendingOrder = {
            product: parameters.product || parameters['coffee-type'],
            quantity: parameters.quantity || 1,
          };
        }
        break;

      case 'order.confirm':
        if (newState.inOrderFlow) {
          newState.inOrderFlow = false;
          newState.orderStep = null;
          newState.pendingOrder = null;
        }
        break;

      case 'order.cancel':
        newState.inOrderFlow = false;
        newState.orderStep = null;
        newState.pendingOrder = null;
        break;
    }

    return newState;
  }
}

module.exports = ContextAwareHandler;
```

---

## 8. Fallback Handling

```javascript
// src/handlers/fallbackHandler.js

class FallbackHandler {
  constructor(lineClient, openaiService) {
    this.lineClient = lineClient;
    this.openai = openaiService;
  }

  async handle(userId, text, dfResult) {
    // ตรวจสอบว่าเป็น Fallback intent และ confidence ต่ำ
    const isFallback = dfResult.intent === 'Default Fallback Intent' ||
      dfResult.confidence < 0.5;

    if (!isFallback) {
      return null; // ให้ Dialogflow จัดการ
    }

    logger.info('Fallback triggered, using OpenAI', {
      userId,
      intent: dfResult.intent,
      confidence: dfResult.confidence,
    });

    // ใช้ OpenAI เป็น Fallback
    try {
      const systemPrompt = `คุณคือ AI Assistant สำหรับ Coffee House ร้านกาแฟ
ตอบคำถามเกี่ยวกับ:
- เมนูและราคา
- เวลาทำการ
- ที่ตั้งร้าน
- การสั่งอาหาร/เครื่องดื่ม
ถ้าไม่ทราบข้อมูล ให้แนะนำให้โทรหาร้าน 02-xxx-xxxx`;

      const response = await this.openai.chat([
        { role: 'system', content: systemPrompt },
        { role: 'user', content: text },
      ]);

      return {
        text: response.content,
        source: 'openai_fallback',
      };
    } catch (error) {
      logger.error('OpenAI fallback error', { error: error.message });
      return {
        text: 'ขออภัยครับ ไม่สามารถตอบได้ในขณะนี้ กรุณาติดต่อ 02-xxx-xxxx',
        source: 'error_fallback',
      };
    }
  }

  // สร้าง Suggestion สำหรับ Fallback
  getSuggestions() {
    return [
      'ดูเมนู',
      'ราคาสินค้า',
      'ที่ตั้งร้าน',
      'เวลาทำการ',
      'สั่งสินค้า',
    ];
  }
}

module.exports = FallbackHandler;
```

---

## 9. Dialogflow Agent Setup Script

```javascript
// scripts/setupDialogflowAgent.js
// สคริปต์สำหรับสร้าง Intents และ Entities ผ่าน API

const dialogflow = require('@google-cloud/dialogflow');

const projectId = process.env.DIALOGFLOW_PROJECT_ID;
const intentsClient = new dialogflow.IntentsClient({
  keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
});
const entitiesClient = new dialogflow.EntityTypesClient({
  keyFilename: process.env.GOOGLE_APPLICATION_CREDENTIALS,
});

// สร้าง Intent
async function createIntent(displayName, trainingPhrases, responses) {
  const agentPath = intentsClient.projectAgentPath(projectId);

  const intent = {
    displayName,
    trainingPhrases: trainingPhrases.map((phrase) => ({
      type: 'EXAMPLE',
      parts: [{ text: phrase }],
    })),
    messages: [{
      text: { text: responses },
    }],
  };

  try {
    const [createdIntent] = await intentsClient.createIntent({
      parent: agentPath,
      intent,
    });
    console.log(`Created intent: ${createdIntent.displayName}`);
    return createdIntent;
  } catch (error) {
    if (error.code === 6) {
      console.log(`Intent ${displayName} already exists`);
    } else {
      throw error;
    }
  }
}

// สร้าง Entity Type
async function createEntity(displayName, entities) {
  const agentPath = entitiesClient.projectAgentPath(projectId);

  const entityType = {
    displayName,
    kind: 'KIND_MAP',
    entities: entities.map(({ value, synonyms }) => ({
      value,
      synonyms,
    })),
  };

  try {
    const [createdType] = await entitiesClient.createEntityType({
      parent: agentPath,
      entityType,
    });
    console.log(`Created entity type: ${createdType.displayName}`);
    return createdType;
  } catch (error) {
    console.error('Error creating entity:', error.message);
    throw error;
  }
}

// Setup ทั้งหมด
async function setup() {
  console.log('Setting up Dialogflow agent...');

  // สร้าง Entities
  await createEntity('coffee-type', [
    { value: 'latte', synonyms: ['ลาเต้', 'ลาเต', 'latte'] },
    { value: 'americano', synonyms: ['อเมริกาโน่', 'americano'] },
    { value: 'cappuccino', synonyms: ['คาปูชิโน่', 'cappuccino'] },
    { value: 'green-tea', synonyms: ['ชาเขียว', 'matcha'] },
  ]);

  // สร้าง Intents
  await createIntent(
    'order.start',
    ['สั่งกาแฟ', 'อยากสั่งกาแฟ', 'ขอสั่ง', 'จะซื้อกาแฟ'],
    ['ต้องการกาแฟอะไรครับ?']
  );

  await createIntent(
    'inquiry.menu',
    ['มีอะไรบ้าง', 'เมนูอะไรบ้าง', 'ขายอะไร', 'รายการเครื่องดื่ม'],
    ['เดี๋ยวดูเมนูให้ครับ']
  );

  await createIntent(
    'inquiry.price',
    ['ราคาเท่าไหร่', 'กี่บาท', 'แพงไหม', 'ราคา'],
    ['สินค้าที่ต้องการทราบราคาคืออะไรครับ?']
  );

  console.log('Dialogflow agent setup complete!');
}

setup().catch(console.error);
```

---

## 10. Testing Conversation Flows

```javascript
// tests/integration/dialogflow.test.js
const dialogflowES = require('../../src/services/dialogflowES');
const { v4: uuidv4 } = require('uuid');

describe('Dialogflow Conversation Flows', () => {
  let sessionId;

  beforeEach(() => {
    sessionId = uuidv4();
  });

  describe('Greeting Flow', () => {
    it('should match greeting intent', async () => {
      const result = await dialogflowES.detectIntent(sessionId, 'สวัสดีครับ');
      expect(result.intent).toBe('greeting');
      expect(result.confidence).toBeGreaterThan(0.7);
    });
  });

  describe('Order Flow', () => {
    it('should complete order flow', async () => {
      // Step 1: เริ่มสั่ง
      let result = await dialogflowES.detectIntent(sessionId, 'อยากสั่งกาแฟ');
      expect(result.intent).toBe('order.start');

      // Step 2: ระบุสินค้า
      result = await dialogflowES.detectIntent(sessionId, 'กาแฟลาเต้');
      expect(result.intent).toMatch(/order/);

      // ตรวจสอบ Context ถูกส่งต่อ
      const orderContext = result.outputContexts.find(
        (ctx) => ctx.name.includes('order')
      );
      expect(orderContext).toBeTruthy();
    });

    it('should handle cancellation', async () => {
      await dialogflowES.detectIntent(sessionId, 'อยากสั่งกาแฟ');
      const result = await dialogflowES.detectIntent(sessionId, 'ยกเลิก');
      expect(result.intent).toBe('order.cancel');
    });
  });

  describe('Inquiry Flow', () => {
    it('should show menu', async () => {
      const result = await dialogflowES.detectIntent(sessionId, 'มีเมนูอะไรบ้าง');
      expect(result.intent).toBe('inquiry.menu');
    });

    it('should show location', async () => {
      const result = await dialogflowES.detectIntent(sessionId, 'อยู่ที่ไหน');
      expect(result.intent).toBe('inquiry.location');
    });
  });

  describe('Fallback Handling', () => {
    it('should trigger fallback for unknown input', async () => {
      const result = await dialogflowES.detectIntent(
        sessionId,
        'asdfghjklqwerty'
      );
      expect(result.intent).toBe('Default Fallback Intent');
    });
  });
});
```

---

## 11. Deploy และ Production

```yaml
# .github/workflows/deploy.yml
name: Deploy LINE Dialogflow Bot

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3

      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci --only=production

      - name: Run tests
        run: npm test

      - name: Deploy to Cloud Run
        uses: google-github-actions/deploy-cloudrun@v1
        with:
          service: line-dialogflow-bot
          region: asia-southeast1
          image: gcr.io/${{ secrets.GCP_PROJECT_ID }}/line-dialogflow-bot
          env_vars: |
            LINE_CHANNEL_ACCESS_TOKEN=${{ secrets.LINE_CHANNEL_ACCESS_TOKEN }}
            LINE_CHANNEL_SECRET=${{ secrets.LINE_CHANNEL_SECRET }}
            DIALOGFLOW_PROJECT_ID=${{ secrets.DIALOGFLOW_PROJECT_ID }}
```

---

## สรุปบทที่ 63

ในบทนี้เราได้เรียนรู้:

1. **Dialogflow ES vs CX** - ความแตกต่างและการเลือกใช้
2. **Intent & Entity Setup** - การสร้างและจัดการผ่าน API
3. **Session Management** - การจัดการ Session ด้วย Redis
4. **Fulfillment Webhook** - การสร้าง Webhook สำหรับ Business Logic
5. **Multi-turn Conversations** - การสนทนาหลายรอบพร้อม Context
6. **Fallback Handling** - การใช้ OpenAI เป็น Fallback
7. **LINE Integration** - การแปลง Dialogflow response เป็น LINE messages
8. **Testing** - การทดสอบ Conversation Flows

---

*ต่อไป: Part 64 - Image Recognition สำหรับ LINE Bot*
