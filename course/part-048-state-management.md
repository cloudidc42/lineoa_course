# Part 48: State Management สำหรับ Chatbot

## บทนำ

State Management เป็นหัวใจสำคัญของ chatbot ที่ซับซ้อน เมื่อ user คุยกับ bot หลายรอบ เราต้องจดจำว่าผู้ใช้ทำอะไรไปแล้ว อยู่ที่ขั้นตอนไหน และเก็บข้อมูลอะไรไว้

---

## 1. ทำความเข้าใจ Conversation State

### 1.1 State คืออะไร?

```
ตัวอย่าง: Bot สั่งพิซซ่า

User: "สั่งพิซซ่า"
Bot:  "เลือกขนาดได้เลยครับ: S, M, L"
                    ↑
              STATE: { flow: 'order', step: 'choose_size' }

User: "L"
Bot:  "เลือก topping ได้เลยครับ: Cheese, Pepperoni, Veggie"
                    ↑
              STATE: { flow: 'order', step: 'choose_topping', size: 'L' }

User: "Pepperoni"
Bot:  "สรุปออเดอร์: พิซซ่า L + Pepperoni ราคา 350 บาท ยืนยันไหม?"
                    ↑
              STATE: { flow: 'order', step: 'confirm', size: 'L', topping: 'Pepperoni', price: 350 }

User: "ยืนยัน"
Bot:  "รับออเดอร์แล้วครับ! หมายเลข: ORD001"
                    ↑
              STATE: { flow: null, step: null }  ← Clear state
```

### 1.2 ประเภทของ State

```
State ใน Chatbot มี 2 ระดับ:

1. USER STATE - ข้อมูลถาวรของ user
   ├── ชื่อ, email, phone
   ├── ประวัติการสั่งซื้อ
   ├── การตั้งค่า preferences
   └── Points/Rewards

2. CONVERSATION STATE - ข้อมูลชั่วคราว (session)
   ├── flow ที่กำลังทำอยู่
   ├── step ปัจจุบัน
   ├── ข้อมูลที่เก็บระหว่าง flow
   └── หมดอายุเมื่อ session หมด
```

---

## 2. State Machine Pattern

### 2.1 Finite State Machine (FSM)

```
State Machine สำหรับ Order Flow:

                    ┌─────────────┐
                    │    IDLE     │◄──────────────────────┐
                    └──────┬──────┘                       │
                           │ "สั่งสินค้า"                  │ cancel/complete
                           ▼                              │
                    ┌─────────────┐                       │
                    │  CHOOSING   │                       │
                    │  PRODUCT    │                       │
                    └──────┬──────┘                       │
                           │ เลือกสินค้า                   │
                           ▼                              │
                    ┌─────────────┐                       │
                    │  CHOOSING   │                       │
                    │  SIZE       │                       │
                    └──────┬──────┘                       │
                           │ เลือกขนาด                    │
                           ▼                              │
                    ┌─────────────┐                       │
                    │  ENTERING   │                       │
                    │  ADDRESS    │                       │
                    └──────┬──────┘                       │
                           │ กรอกที่อยู่                   │
                           ▼                              │
                    ┌─────────────┐                       │
                    │ CONFIRMING  │──── ยืนยัน ──────────►│
                    └─────────────┘                       │
                                                          │
```

### 2.2 State Machine Implementation (Node.js)

```javascript
// src/state/StateMachine.js

class State {
  constructor(name, handlers = {}) {
    this.name = name;
    this.handlers = handlers; // { 'event_name': nextStateName }
    this.onEnter = null;  // function ที่รันเมื่อเข้า state
    this.onExit = null;   // function ที่รันเมื่อออก state
  }
}

class StateMachine {
  constructor(states, initialState) {
    this.states = {};
    this.currentStateName = initialState;

    states.forEach(state => {
      this.states[state.name] = state;
    });
  }

  get currentState() {
    return this.states[this.currentStateName];
  }

  async transition(event, context = {}) {
    const current = this.currentState;

    if (!current) {
      throw new Error(`Invalid state: ${this.currentStateName}`);
    }

    const nextStateName = current.handlers[event];

    if (!nextStateName) {
      console.warn(`No transition for event '${event}' in state '${current.name}'`);
      return false;
    }

    const nextState = this.states[nextStateName];
    if (!nextState) {
      throw new Error(`Target state '${nextStateName}' not found`);
    }

    // Run exit handler
    if (current.onExit) {
      await current.onExit(context);
    }

    this.currentStateName = nextStateName;

    // Run enter handler
    if (nextState.onEnter) {
      await nextState.onEnter(context);
    }

    return true;
  }

  can(event) {
    return !!this.currentState?.handlers[event];
  }
}

// =============================================
// ORDER FLOW STATE MACHINE
// =============================================

function createOrderStateMachine() {
  const states = [
    new State('idle', {
      'start_order': 'choosing_product',
    }),
    new State('choosing_product', {
      'product_selected': 'choosing_variant',
      'cancel': 'idle',
    }),
    new State('choosing_variant', {
      'variant_selected': 'entering_address',
      'back': 'choosing_product',
      'cancel': 'idle',
    }),
    new State('entering_address', {
      'address_entered': 'confirming',
      'back': 'choosing_variant',
      'cancel': 'idle',
    }),
    new State('confirming', {
      'confirmed': 'payment',
      'back': 'entering_address',
      'cancel': 'idle',
    }),
    new State('payment', {
      'payment_success': 'completed',
      'payment_failed': 'confirming',
      'cancel': 'idle',
    }),
    new State('completed', {
      'restart': 'idle',
    }),
  ];

  return new StateMachine(states, 'idle');
}

module.exports = { StateMachine, createOrderStateMachine };
```

---

## 3. State Storage Options

### 3.1 In-Memory Storage (สำหรับ Development)

```javascript
// src/state/storage/MemoryStorage.js

class MemoryStorage {
  constructor() {
    this.store = new Map();
    this.ttls = new Map();
    
    // cleanup interval ทุก 5 นาที
    setInterval(() => this.cleanup(), 5 * 60 * 1000);
  }

  async get(key) {
    // ตรวจสอบ TTL
    const ttl = this.ttls.get(key);
    if (ttl && ttl < Date.now()) {
      this.store.delete(key);
      this.ttls.delete(key);
      return null;
    }
    return this.store.get(key) || null;
  }

  async set(key, value, ttlSeconds = 1800) { // default 30 นาที
    this.store.set(key, value);
    if (ttlSeconds > 0) {
      this.ttls.set(key, Date.now() + ttlSeconds * 1000);
    }
  }

  async delete(key) {
    this.store.delete(key);
    this.ttls.delete(key);
  }

  async exists(key) {
    const value = await this.get(key);
    return value !== null;
  }

  cleanup() {
    const now = Date.now();
    for (const [key, ttl] of this.ttls.entries()) {
      if (ttl < now) {
        this.store.delete(key);
        this.ttls.delete(key);
      }
    }
  }

  get size() {
    return this.store.size;
  }
}

module.exports = MemoryStorage;
```

### 3.2 Redis Storage

```javascript
// src/state/storage/RedisStorage.js
const Redis = require('ioredis');

class RedisStorage {
  constructor(options = {}) {
    this.client = new Redis({
      host: options.host || process.env.REDIS_HOST || 'localhost',
      port: options.port || parseInt(process.env.REDIS_PORT) || 6379,
      password: options.password || process.env.REDIS_PASSWORD,
      db: options.db || 0,
      keyPrefix: options.prefix || 'linebot:state:',
      retryStrategy: (times) => {
        const delay = Math.min(times * 50, 2000);
        return delay;
      },
    });

    this.defaultTTL = options.defaultTTL || 1800; // 30 นาที
  }

  async get(key) {
    const value = await this.client.get(key);
    if (!value) return null;
    try {
      return JSON.parse(value);
    } catch {
      return value;
    }
  }

  async set(key, value, ttlSeconds = this.defaultTTL) {
    const serialized = JSON.stringify(value);
    if (ttlSeconds > 0) {
      await this.client.setex(key, ttlSeconds, serialized);
    } else {
      await this.client.set(key, serialized);
    }
  }

  async delete(key) {
    await this.client.del(key);
  }

  async exists(key) {
    return (await this.client.exists(key)) === 1;
  }

  async expire(key, ttlSeconds) {
    await this.client.expire(key, ttlSeconds);
  }

  async ttl(key) {
    return await this.client.ttl(key);
  }

  async disconnect() {
    await this.client.quit();
  }
}

module.exports = RedisStorage;
```

### 3.3 Database Storage (MySQL)

```javascript
// src/state/storage/DatabaseStorage.js
const { pool } = require('../../database/mysql');

class DatabaseStorage {
  constructor(options = {}) {
    this.defaultTTL = options.defaultTTL || 1800; // 30 นาที
  }

  async get(key) {
    const [rows] = await pool.execute(
      `SELECT context_data FROM conversation_states 
       WHERE session_id = ? 
       AND is_active = TRUE 
       AND (expires_at IS NULL OR expires_at > NOW())
       LIMIT 1`,
      [key]
    );

    if (rows.length === 0) return null;
    return rows[0].context_data;
  }

  async set(key, value, ttlSeconds = this.defaultTTL) {
    const expiresAt = ttlSeconds > 0
      ? new Date(Date.now() + ttlSeconds * 1000)
      : null;

    // Extract user_id จาก key format "user:{userId}"
    const userId = this.extractUserId(key);

    await pool.execute(
      `INSERT INTO conversation_states 
       (user_id, session_id, current_flow, current_step, context_data, expires_at)
       VALUES (?, ?, ?, ?, ?, ?)
       ON DUPLICATE KEY UPDATE
         context_data = VALUES(context_data),
         current_flow = VALUES(current_flow),
         current_step = VALUES(current_step),
         expires_at = VALUES(expires_at),
         updated_at = NOW()`,
      [
        userId,
        key,
        value.flow || null,
        value.step || null,
        JSON.stringify(value),
        expiresAt
      ]
    );
  }

  async delete(key) {
    await pool.execute(
      'UPDATE conversation_states SET is_active = FALSE WHERE session_id = ?',
      [key]
    );
  }

  async exists(key) {
    const value = await this.get(key);
    return value !== null;
  }

  extractUserId(key) {
    const match = key.match(/user:(\d+)/);
    return match ? parseInt(match[1]) : null;
  }
}

module.exports = DatabaseStorage;
```

---

## 4. Complete State Manager

### 4.1 StateManager Class (Node.js)

```javascript
// src/state/StateManager.js
const RedisStorage = require('./storage/RedisStorage');

class StateManager {
  constructor(storage = null) {
    this.storage = storage || new RedisStorage();
    this.flowHandlers = new Map();
    this.defaultTTL = 30 * 60; // 30 นาที
  }

  /**
   * สร้าง key สำหรับเก็บ state
   * @param {string} userId - LINE User ID
   * @returns {string}
   */
  getStateKey(userId) {
    return `user:${userId}:state`;
  }

  /**
   * ดึง state ปัจจุบันของ user
   * @param {string} userId
   * @returns {object|null}
   */
  async getState(userId) {
    const key = this.getStateKey(userId);
    const state = await this.storage.get(key);
    return state;
  }

  /**
   * ตั้งค่า state
   * @param {string} userId
   * @param {string} flow - ชื่อ flow
   * @param {string} step - ชื่อ step
   * @param {object} data - ข้อมูลเพิ่มเติม
   * @param {number} ttl - เวลาหมดอายุ (วินาที)
   */
  async setState(userId, flow, step, data = {}, ttl = this.defaultTTL) {
    const key = this.getStateKey(userId);
    const state = {
      flow,
      step,
      data,
      createdAt: Date.now(),
      updatedAt: Date.now(),
    };
    await this.storage.set(key, state, ttl);
    return state;
  }

  /**
   * อัปเดต step ใน flow เดิม
   * @param {string} userId
   * @param {string} newStep
   * @param {object} additionalData - ข้อมูลเพิ่มเติมที่จะ merge
   */
  async updateStep(userId, newStep, additionalData = {}) {
    const current = await this.getState(userId);
    if (!current) {
      throw new Error(`No active state for user ${userId}`);
    }

    return this.setState(
      userId,
      current.flow,
      newStep,
      { ...current.data, ...additionalData }
    );
  }

  /**
   * อัปเดตข้อมูลใน state
   * @param {string} userId
   * @param {object} newData
   */
  async updateData(userId, newData) {
    const current = await this.getState(userId);
    if (!current) {
      throw new Error(`No active state for user ${userId}`);
    }

    return this.setState(
      userId,
      current.flow,
      current.step,
      { ...current.data, ...newData }
    );
  }

  /**
   * ล้าง state
   * @param {string} userId
   */
  async clearState(userId) {
    const key = this.getStateKey(userId);
    await this.storage.delete(key);
  }

  /**
   * ตรวจสอบว่า user มี active state หรือไม่
   * @param {string} userId
   * @returns {boolean}
   */
  async hasState(userId) {
    const state = await this.getState(userId);
    return state !== null;
  }

  /**
   * ดึงข้อมูลใน state
   * @param {string} userId
   * @param {string} key
   * @param {any} defaultValue
   */
  async getData(userId, key, defaultValue = null) {
    const state = await this.getState(userId);
    if (!state) return defaultValue;
    return state.data[key] !== undefined ? state.data[key] : defaultValue;
  }

  /**
   * ลงทะเบียน flow handler
   * @param {string} flowName
   * @param {Function} handler
   */
  registerFlow(flowName, handler) {
    this.flowHandlers.set(flowName, handler);
  }

  /**
   * ประมวลผลข้อความจาก user
   * @param {string} userId
   * @param {string} message
   * @param {object} context - LINE webhook context
   */
  async processMessage(userId, message, context) {
    const state = await this.getState(userId);

    if (!state) {
      // ไม่มี state → ส่งไปยัง default handler
      return this.handleDefaultMessage(userId, message, context);
    }

    const handler = this.flowHandlers.get(state.flow);
    if (!handler) {
      console.warn(`No handler for flow: ${state.flow}`);
      await this.clearState(userId);
      return this.handleDefaultMessage(userId, message, context);
    }

    return handler(userId, message, state, context);
  }

  async handleDefaultMessage(userId, message, context) {
    // Override in subclass หรือ pass default handler
    console.log(`Default message handler for user ${userId}: ${message}`);
  }
}

module.exports = StateManager;
```

### 4.2 Flow Handlers

```javascript
// src/flows/OrderFlow.js
class OrderFlow {
  constructor(stateManager, lineClient) {
    this.state = stateManager;
    this.line = lineClient;
    this.flowName = 'order_flow';
  }

  /**
   * เริ่ม order flow
   */
  async start(userId, replyToken) {
    await this.state.setState(userId, this.flowName, 'select_product', {});

    await this.line.replyMessage(replyToken, [
      {
        type: 'text',
        text: 'ยินดีต้อนรับสู่ร้านค้า! กรุณาเลือกสินค้า:',
      },
      {
        type: 'template',
        altText: 'เลือกสินค้า',
        template: {
          type: 'carousel',
          columns: [
            {
              thumbnailImageUrl: 'https://example.com/pizza.jpg',
              title: 'พิซซ่า',
              text: 'เลือกขนาดและ topping',
              actions: [
                {
                  type: 'postback',
                  label: 'เลือก',
                  data: 'action=select_product&product=pizza',
                },
              ],
            },
            {
              thumbnailImageUrl: 'https://example.com/pasta.jpg',
              title: 'พาสต้า',
              text: 'หลากหลายซอส',
              actions: [
                {
                  type: 'postback',
                  label: 'เลือก',
                  data: 'action=select_product&product=pasta',
                },
              ],
            },
          ],
        },
      },
    ]);
  }

  /**
   * Handle message ใน flow
   */
  async handle(userId, message, currentState, context) {
    const { step, data } = currentState;
    const { replyToken } = context;

    switch (step) {
      case 'select_product':
        return this.handleSelectProduct(userId, message, data, replyToken);

      case 'select_size':
        return this.handleSelectSize(userId, message, data, replyToken);

      case 'enter_address':
        return this.handleEnterAddress(userId, message, data, replyToken);

      case 'confirm_order':
        return this.handleConfirmOrder(userId, message, data, replyToken);

      default:
        await this.state.clearState(userId);
        return this.line.replyMessage(replyToken, [{
          type: 'text',
          text: 'เกิดข้อผิดพลาด กรุณาเริ่มใหม่',
        }]);
    }
  }

  async handleSelectProduct(userId, message, data, replyToken) {
    const products = {
      'pizza': { name: 'พิซซ่า', sizes: ['S (6 ชิ้น)', 'M (8 ชิ้น)', 'L (12 ชิ้น)'] },
      'pasta': { name: 'พาสต้า', sizes: ['S', 'M', 'L'] },
    };

    const product = products[message.toLowerCase()];
    if (!product) {
      return this.line.replyMessage(replyToken, [{
        type: 'text',
        text: 'ขอโทษครับ ไม่พบสินค้านี้ กรุณาเลือกใหม่',
      }]);
    }

    await this.state.updateStep(userId, 'select_size', {
      product: message.toLowerCase(),
      productName: product.name,
    });

    return this.line.replyMessage(replyToken, [
      {
        type: 'text',
        text: `เลือก ${product.name} ✓\n\nกรุณาเลือกขนาด:`,
      },
      {
        type: 'template',
        altText: 'เลือกขนาด',
        template: {
          type: 'buttons',
          text: 'เลือกขนาดที่ต้องการ',
          actions: product.sizes.map(size => ({
            type: 'message',
            label: size,
            text: size.split(' ')[0], // เอาแค่ S/M/L
          })),
        },
      },
    ]);
  }

  async handleSelectSize(userId, message, data, replyToken) {
    const validSizes = ['S', 'M', 'L'];
    const size = message.toUpperCase();

    if (!validSizes.includes(size)) {
      return this.line.replyMessage(replyToken, [{
        type: 'text',
        text: 'กรุณาเลือกขนาด S, M หรือ L',
      }]);
    }

    const prices = { S: 150, M: 250, L: 350 };
    const price = prices[size];

    await this.state.updateStep(userId, 'enter_address', {
      ...data,
      size,
      price,
    });

    return this.line.replyMessage(replyToken, [{
      type: 'text',
      text: `ขนาด ${size} ราคา ${price} บาท ✓\n\nกรุณากรอกที่อยู่จัดส่ง:\n(พิมพ์ที่อยู่หรือส่ง location)`,
    }]);
  }

  async handleEnterAddress(userId, message, data, replyToken) {
    if (message.length < 10) {
      return this.line.replyMessage(replyToken, [{
        type: 'text',
        text: 'กรุณากรอกที่อยู่ให้ครบถ้วน (อย่างน้อย 10 ตัวอักษร)',
      }]);
    }

    await this.state.updateStep(userId, 'confirm_order', {
      ...data,
      address: message,
    });

    const summary = `📋 สรุปออเดอร์\n\nสินค้า: ${data.productName}\nขนาด: ${data.size}\nราคา: ${data.price} บาท\nที่อยู่: ${message}\n\nยืนยันออเดอร์?`;

    return this.line.replyMessage(replyToken, [
      { type: 'text', text: summary },
      {
        type: 'template',
        altText: 'ยืนยันออเดอร์',
        template: {
          type: 'confirm',
          text: 'ยืนยันการสั่งซื้อ',
          actions: [
            { type: 'message', label: '✓ ยืนยัน', text: 'ยืนยัน' },
            { type: 'message', label: '✗ ยกเลิก', text: 'ยกเลิก' },
          ],
        },
      },
    ]);
  }

  async handleConfirmOrder(userId, message, data, replyToken) {
    if (message === 'ยกเลิก') {
      await this.state.clearState(userId);
      return this.line.replyMessage(replyToken, [{
        type: 'text',
        text: 'ยกเลิกออเดอร์แล้วครับ 😊',
      }]);
    }

    if (message !== 'ยืนยัน') {
      return this.line.replyMessage(replyToken, [{
        type: 'text',
        text: 'กรุณาตอบ "ยืนยัน" หรือ "ยกเลิก"',
      }]);
    }

    // สร้าง order
    const orderNumber = `ORD${Date.now()}`;
    
    // TODO: บันทึกลง database

    await this.state.clearState(userId);

    return this.line.replyMessage(replyToken, [{
      type: 'flex',
      altText: 'ออเดอร์สำเร็จ',
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            { type: 'text', text: '✅ รับออเดอร์แล้ว!', weight: 'bold', size: 'xl', color: '#06C755' },
            { type: 'text', text: `หมายเลข: ${orderNumber}`, margin: 'md' },
            { type: 'text', text: `สินค้า: ${data.productName} ${data.size}`, margin: 'sm' },
            { type: 'text', text: `ราคา: ฿${data.price}`, margin: 'sm', color: '#FF6B35' },
            { type: 'text', text: 'คาดว่าจะส่งถึง 30-45 นาที', margin: 'md', color: '#999999', size: 'sm' },
          ],
        },
      },
    }]);
  }
}

module.exports = OrderFlow;
```

---

## 5. Multi-Turn Conversation Design

### 5.1 Context Tracking

```javascript
// src/state/ConversationContext.js
class ConversationContext {
  constructor(stateManager) {
    this.state = stateManager;
    this.historyKey = (userId) => `user:${userId}:history`;
    this.maxHistory = 10;
  }

  /**
   * เพิ่มข้อความลงใน history
   */
  async addToHistory(userId, role, content) {
    const key = this.historyKey(userId);
    const current = await this.state.storage.get(key) || [];

    current.push({
      role, // 'user' หรือ 'bot'
      content,
      timestamp: Date.now(),
    });

    // เก็บแค่ maxHistory ข้อความล่าสุด
    const trimmed = current.slice(-this.maxHistory);

    await this.state.storage.set(key, trimmed, 3600); // 1 ชั่วโมง
    return trimmed;
  }

  /**
   * ดึงประวัติการสนทนา
   */
  async getHistory(userId, limit = null) {
    const key = this.historyKey(userId);
    const history = await this.state.storage.get(key) || [];
    return limit ? history.slice(-limit) : history;
  }

  /**
   * ล้างประวัติ
   */
  async clearHistory(userId) {
    const key = this.historyKey(userId);
    await this.state.storage.delete(key);
  }

  /**
   * สร้าง context summary สำหรับส่งให้ AI
   */
  async getContextSummary(userId) {
    const history = await this.getHistory(userId, 5);
    return history.map(h => `${h.role}: ${h.content}`).join('\n');
  }
}

module.exports = ConversationContext;
```

---

## 6. Timeout Handling

### 6.1 State Timeout

```javascript
// src/state/StateTimeout.js
class StateTimeout {
  constructor(stateManager, lineClient) {
    this.state = stateManager;
    this.line = lineClient;
    this.timeoutMessages = {
      order_flow: 'เซสชันการสั่งซื้อหมดเวลาแล้วครับ กรุณาเริ่มใหม่',
      booking_flow: 'เซสชันการจองหมดเวลาแล้วครับ กรุณาเริ่มใหม่',
      default: 'เซสชันหมดเวลาแล้วครับ',
    };
  }

  /**
   * ตรวจสอบ timeout เมื่อ user ส่งข้อความ
   * (ระบบจะตรวจสอบก่อน process message ทุกครั้ง)
   */
  async checkAndHandle(userId, replyToken) {
    const state = await this.state.getState(userId);
    if (!state) return false;

    const now = Date.now();
    const TIMEOUT_MS = 30 * 60 * 1000; // 30 นาที
    const stateAge = now - state.updatedAt;

    if (stateAge > TIMEOUT_MS) {
      const message = this.timeoutMessages[state.flow] || this.timeoutMessages.default;

      await this.state.clearState(userId);

      if (replyToken) {
        await this.line.replyMessage(replyToken, [{
          type: 'text',
          text: `⏰ ${message}\n\nพิมพ์ "เมนู" เพื่อกลับสู่เมนูหลักครับ`,
        }]);
      }

      return true; // timeout เกิดขึ้น
    }

    return false; // ไม่ timeout
  }

  /**
   * Extended timeout สำหรับ flows บางอัน
   */
  getFlowTimeout(flowName) {
    const timeouts = {
      payment_flow: 15 * 60, // 15 นาที
      order_flow: 30 * 60,   // 30 นาที
      booking_flow: 30 * 60,
      registration_flow: 60 * 60, // 1 ชั่วโมง
    };
    return timeouts[flowName] || 1800;
  }
}

module.exports = StateTimeout;
```

---

## 7. Complete Implementation

### 7.1 Main Bot Handler (Node.js)

```javascript
// src/bot.js
const line = require('@line/bot-sdk');
const StateManager = require('./state/StateManager');
const RedisStorage = require('./state/storage/RedisStorage');
const OrderFlow = require('./flows/OrderFlow');
const BookingFlow = require('./flows/BookingFlow');
const StateTimeout = require('./state/StateTimeout');
const ConversationContext = require('./state/ConversationContext');

const lineConfig = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET,
};

const client = new line.Client(lineConfig);
const storage = new RedisStorage();
const stateManager = new StateManager(storage);
const stateTimeout = new StateTimeout(stateManager, client);
const context = new ConversationContext(stateManager);

// ลงทะเบียน flows
const orderFlow = new OrderFlow(stateManager, client);
const bookingFlow = new BookingFlow(stateManager, client);

stateManager.registerFlow('order_flow', 
  (userId, msg, state, ctx) => orderFlow.handle(userId, msg, state, ctx)
);
stateManager.registerFlow('booking_flow',
  (userId, msg, state, ctx) => bookingFlow.handle(userId, msg, state, ctx)
);

// Default handler สำหรับเมื่อไม่มี state
stateManager.handleDefaultMessage = async (userId, message, ctx) => {
  const { replyToken } = ctx;

  switch (message.toLowerCase()) {
    case 'สั่งอาหาร':
    case 'สั่งสินค้า':
    case 'order':
      await orderFlow.start(userId, replyToken);
      break;

    case 'จอง':
    case 'นัดหมาย':
    case 'book':
      await bookingFlow.start(userId, replyToken);
      break;

    case 'เมนู':
    case 'menu':
      await showMainMenu(replyToken);
      break;

    default:
      await client.replyMessage(replyToken, [{
        type: 'text',
        text: '👋 สวัสดีครับ! พิมพ์ "เมนู" เพื่อดูตัวเลือก',
      }]);
  }
};

// Webhook handler
async function handleEvent(event) {
  if (event.type !== 'message') return;

  const userId = event.source.userId;
  const replyToken = event.replyToken;

  let message = '';
  if (event.message.type === 'text') {
    message = event.message.text.trim();
  } else if (event.message.type === 'postback') {
    message = event.postback.data;
  }

  // เพิ่มลงใน history
  await context.addToHistory(userId, 'user', message);

  // ตรวจสอบ timeout
  const timedOut = await stateTimeout.checkAndHandle(userId, replyToken);
  if (timedOut) return;

  // ประมวลผลข้อความ
  await stateManager.processMessage(userId, message, {
    userId,
    replyToken,
    event,
  });
}

async function showMainMenu(replyToken) {
  await client.replyMessage(replyToken, [{
    type: 'flex',
    altText: 'เมนูหลัก',
    contents: {
      type: 'bubble',
      body: {
        type: 'box',
        layout: 'vertical',
        spacing: 'md',
        contents: [
          { type: 'text', text: '🏠 เมนูหลัก', weight: 'bold', size: 'xl' },
          {
            type: 'box',
            layout: 'vertical',
            spacing: 'sm',
            margin: 'lg',
            contents: [
              { type: 'button', action: { type: 'message', label: '🍕 สั่งอาหาร', text: 'สั่งอาหาร' }, style: 'primary' },
              { type: 'button', action: { type: 'message', label: '📅 จองคิว', text: 'จอง' }, style: 'secondary' },
              { type: 'button', action: { type: 'uri', label: '🛍 ดูสินค้า', uri: 'https://example.com/shop' } },
            ],
          },
        ],
      },
    },
  }]);
}

module.exports = { handleEvent };
```

---

## 8. Python Implementation

```python
# state_manager.py
import json
import time
from typing import Optional, Dict, Any
from dataclasses import dataclass, asdict
import redis
import logging

logger = logging.getLogger(__name__)

@dataclass
class ConversationState:
    flow: Optional[str]
    step: Optional[str]
    data: Dict[str, Any]
    created_at: float
    updated_at: float
    
    def to_dict(self):
        return asdict(self)
    
    @classmethod
    def from_dict(cls, d: dict):
        return cls(
            flow=d.get('flow'),
            step=d.get('step'),
            data=d.get('data', {}),
            created_at=d.get('created_at', time.time()),
            updated_at=d.get('updated_at', time.time()),
        )


class StateManager:
    def __init__(self, redis_client: redis.Redis, default_ttl: int = 1800):
        self.redis = redis_client
        self.default_ttl = default_ttl
        self.flow_handlers = {}
        self.prefix = "linebot:state:"

    def _key(self, user_id: str) -> str:
        return f"{self.prefix}{user_id}"

    async def get_state(self, user_id: str) -> Optional[ConversationState]:
        """ดึง state ของ user"""
        key = self._key(user_id)
        data = self.redis.get(key)
        if not data:
            return None
        try:
            return ConversationState.from_dict(json.loads(data))
        except Exception as e:
            logger.error(f"Error deserializing state for {user_id}: {e}")
            return None

    async def set_state(
        self,
        user_id: str,
        flow: str,
        step: str,
        data: Dict[str, Any] = None,
        ttl: int = None
    ) -> ConversationState:
        """ตั้งค่า state"""
        key = self._key(user_id)
        now = time.time()
        
        state = ConversationState(
            flow=flow,
            step=step,
            data=data or {},
            created_at=now,
            updated_at=now,
        )
        
        self.redis.setex(
            key,
            ttl or self.default_ttl,
            json.dumps(state.to_dict())
        )
        return state

    async def update_step(
        self,
        user_id: str,
        new_step: str,
        additional_data: Dict[str, Any] = None
    ) -> Optional[ConversationState]:
        """อัปเดต step"""
        current = await self.get_state(user_id)
        if not current:
            raise ValueError(f"No active state for user {user_id}")
        
        merged_data = {**current.data, **(additional_data or {})}
        return await self.set_state(user_id, current.flow, new_step, merged_data)

    async def update_data(
        self,
        user_id: str,
        new_data: Dict[str, Any]
    ) -> Optional[ConversationState]:
        """อัปเดตข้อมูลใน state"""
        current = await self.get_state(user_id)
        if not current:
            raise ValueError(f"No active state for user {user_id}")
        
        return await self.set_state(
            user_id, current.flow, current.step,
            {**current.data, **new_data}
        )

    async def clear_state(self, user_id: str) -> bool:
        """ล้าง state"""
        key = self._key(user_id)
        result = self.redis.delete(key)
        return result > 0

    def register_flow(self, flow_name: str, handler):
        """ลงทะเบียน flow handler"""
        self.flow_handlers[flow_name] = handler

    async def process_message(
        self,
        user_id: str,
        message: str,
        context: dict
    ):
        """ประมวลผลข้อความ"""
        state = await self.get_state(user_id)
        
        if not state or not state.flow:
            return await self._handle_default(user_id, message, context)
        
        handler = self.flow_handlers.get(state.flow)
        if not handler:
            logger.warning(f"No handler for flow: {state.flow}")
            await self.clear_state(user_id)
            return await self._handle_default(user_id, message, context)
        
        return await handler(user_id, message, state, context)

    async def _handle_default(self, user_id: str, message: str, context: dict):
        """Default handler - override this"""
        pass


# =============================================
# ORDER FLOW (Python)
# =============================================
class OrderFlow:
    def __init__(self, state_manager: StateManager, line_client):
        self.state = state_manager
        self.line = line_client
        self.flow_name = 'order_flow'

    async def start(self, user_id: str, reply_token: str):
        """เริ่ม order flow"""
        await self.state.set_state(user_id, self.flow_name, 'select_product')
        
        await self.line.reply_message(reply_token, [
            {
                "type": "text",
                "text": "ยินดีต้อนรับ! กรุณาเลือกสินค้า:\n\n1. พิซซ่า\n2. พาสต้า\n\nพิมพ์ชื่อสินค้าที่ต้องการ"
            }
        ])

    async def handle(
        self,
        user_id: str,
        message: str,
        state: ConversationState,
        context: dict
    ):
        """Handle message ใน order flow"""
        reply_token = context.get('reply_token')
        
        handlers = {
            'select_product': self._handle_select_product,
            'select_size': self._handle_select_size,
            'enter_address': self._handle_enter_address,
            'confirm_order': self._handle_confirm_order,
        }
        
        handler = handlers.get(state.step)
        if not handler:
            await self.state.clear_state(user_id)
            return
        
        await handler(user_id, message, state.data, reply_token)

    async def _handle_select_product(self, user_id, message, data, reply_token):
        products = {
            'พิซซ่า': {'name': 'พิซซ่า', 'prices': {'S': 150, 'M': 250, 'L': 350}},
            'พาสต้า': {'name': 'พาสต้า', 'prices': {'S': 120, 'M': 180, 'L': 240}},
        }
        
        product = products.get(message)
        if not product:
            await self.line.reply_message(reply_token, [
                {"type": "text", "text": "ขอโทษครับ ไม่พบสินค้านี้"}
            ])
            return
        
        await self.state.update_step(user_id, 'select_size', {
            'product': message,
            'product_info': product,
        })
        
        await self.line.reply_message(reply_token, [{
            "type": "text",
            "text": f"เลือก {product['name']} ✓\n\nกรุณาเลือกขนาด:\nS, M, หรือ L"
        }])

    async def _handle_select_size(self, user_id, message, data, reply_token):
        size = message.upper()
        product_info = data.get('product_info', {})
        prices = product_info.get('prices', {})
        
        if size not in prices:
            await self.line.reply_message(reply_token, [{
                "type": "text",
                "text": "กรุณาเลือก S, M หรือ L"
            }])
            return
        
        price = prices[size]
        await self.state.update_step(user_id, 'enter_address', {
            **data,
            'size': size,
            'price': price,
        })
        
        await self.line.reply_message(reply_token, [{
            "type": "text",
            "text": f"ขนาด {size} ราคา {price} บาท ✓\n\nกรุณากรอกที่อยู่จัดส่ง:"
        }])

    async def _handle_enter_address(self, user_id, message, data, reply_token):
        if len(message) < 10:
            await self.line.reply_message(reply_token, [{
                "type": "text",
                "text": "กรุณากรอกที่อยู่ให้ครบถ้วนครับ"
            }])
            return
        
        await self.state.update_step(user_id, 'confirm_order', {
            **data,
            'address': message,
        })
        
        summary = (
            f"📋 สรุปออเดอร์\n\n"
            f"สินค้า: {data['product']} ขนาด {data.get('size', '')}\n"
            f"ราคา: {data.get('price', 0)} บาท\n"
            f"ที่อยู่: {message}\n\n"
            f"พิมพ์ 'ยืนยัน' หรือ 'ยกเลิก'"
        )
        
        await self.line.reply_message(reply_token, [{
            "type": "text",
            "text": summary
        }])

    async def _handle_confirm_order(self, user_id, message, data, reply_token):
        if message == 'ยกเลิก':
            await self.state.clear_state(user_id)
            await self.line.reply_message(reply_token, [{
                "type": "text",
                "text": "ยกเลิกออเดอร์แล้วครับ 😊"
            }])
            return
        
        if message != 'ยืนยัน':
            await self.line.reply_message(reply_token, [{
                "type": "text",
                "text": "กรุณาพิมพ์ 'ยืนยัน' หรือ 'ยกเลิก'"
            }])
            return
        
        # บันทึก order
        order_number = f"ORD{int(time.time())}"
        
        await self.state.clear_state(user_id)
        
        await self.line.reply_message(reply_token, [{
            "type": "text",
            "text": f"✅ รับออเดอร์แล้วครับ!\n\nหมายเลข: {order_number}\nประมาณ 30-45 นาที"
        }])
```

---

## 9. Testing State Management

```javascript
// tests/state.test.js
const { describe, it, expect, beforeEach } = require('@jest/globals');
const StateManager = require('../src/state/StateManager');
const MemoryStorage = require('../src/state/storage/MemoryStorage');

describe('StateManager', () => {
  let stateManager;

  beforeEach(() => {
    const storage = new MemoryStorage();
    stateManager = new StateManager(storage);
  });

  describe('setState', () => {
    it('should set and get state correctly', async () => {
      const userId = 'U123';
      await stateManager.setState(userId, 'order_flow', 'select_product', { items: [] });

      const state = await stateManager.getState(userId);

      expect(state).not.toBeNull();
      expect(state.flow).toBe('order_flow');
      expect(state.step).toBe('select_product');
      expect(state.data.items).toEqual([]);
    });
  });

  describe('updateStep', () => {
    it('should update step and merge data', async () => {
      const userId = 'U456';
      await stateManager.setState(userId, 'order_flow', 'step1', { a: 1 });
      await stateManager.updateStep(userId, 'step2', { b: 2 });

      const state = await stateManager.getState(userId);

      expect(state.step).toBe('step2');
      expect(state.data.a).toBe(1);
      expect(state.data.b).toBe(2);
    });
  });

  describe('clearState', () => {
    it('should clear state', async () => {
      const userId = 'U789';
      await stateManager.setState(userId, 'flow', 'step');
      await stateManager.clearState(userId);

      const state = await stateManager.getState(userId);
      expect(state).toBeNull();
    });
  });

  describe('hasState', () => {
    it('should return true when state exists', async () => {
      const userId = 'U999';
      await stateManager.setState(userId, 'flow', 'step');
      expect(await stateManager.hasState(userId)).toBe(true);
    });

    it('should return false when no state', async () => {
      expect(await stateManager.hasState('UNEW')).toBe(false);
    });
  });
});
```

---

## สรุป

State Management ที่ดีสำหรับ LINE bot ต้องมี:

1. **State Machine** สำหรับกำหนด flow และ transitions ที่ชัดเจน
2. **Storage Layer** ที่ abstract ให้เปลี่ยน backend ได้ง่าย (Memory, Redis, DB)
3. **TTL/Expiration** เพื่อ handle timeout โดยอัตโนมัติ
4. **Flow Registration** แยก business logic ออกจาก state management
5. **Context Tracking** สำหรับ multi-turn conversations
6. **Testing** ครอบคลุม state transitions ทั้งหมด

การออกแบบที่ดีทำให้ bot ฉลาดขึ้น จดจำ context ได้ และให้ประสบการณ์ที่ลื่นไหลแก่ผู้ใช้
