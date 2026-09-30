# Part 38: Postback Events - การจัดการ Postback ใน LINE OA

## บทนำ

Postback Events เป็นกลไกสำคัญในการสร้าง Interactive Bot ที่ซับซ้อน โดย Postback ช่วยให้ส่งข้อมูลกลับมายัง Webhook โดยที่ผู้ใช้ไม่เห็น Data ที่ส่งมา ทำให้สามารถสร้าง Multi-step Forms, Booking Systems, และ State Machine ได้อย่างมีประสิทธิภาพ

---

## 1. Postback Events คืออะไร?

Postback Event เกิดขึ้นเมื่อผู้ใช้กดปุ่มที่มี Postback Action โดย:
- ไม่แสดงข้อมูลในห้องแชท (เว้นแต่กำหนด `displayText`)
- ส่ง Data string ไปยัง Webhook
- ใช้ใน Buttons Template, Quick Reply, Rich Menu, Flex Message

```
ผู้ใช้กด [ยืนยัน]
    │
    ├─── displayText: "ยืนยัน" ──→ แสดงในแชท
    │
    └─── data: "action=confirm&orderId=123" ──→ Webhook
```

### เปรียบเทียบ Message vs Postback

| ประเภท | Message Action | Postback Action |
|--------|---------------|-----------------|
| ข้อความที่ส่ง | แสดงในแชท | ไม่แสดง (เว้นแต่ displayText) |
| ข้อมูลที่ส่ง | Text เท่านั้น | Custom data string |
| Event ที่ได้รับ | MessageEvent | PostbackEvent |
| ความยืดหยุ่น | จำกัด | สูงมาก |
| ปกป้องข้อมูล | ไม่ได้ | ได้ (data ไม่แสดง) |

---

## 2. โครงสร้าง Postback Action

### Basic Structure

```json
{
  "type": "postback",
  "label": "ยืนยัน",
  "data": "action=confirm&orderId=123",
  "displayText": "ยืนยันการสั่งซื้อ"
}
```

### Fields

| Field | Type | Required | Max Length | Description |
|-------|------|----------|------------|-------------|
| `type` | String | Yes | - | ต้องเป็น `"postback"` |
| `label` | String | Yes | 20 chars | ข้อความบนปุ่ม |
| `data` | String | Yes | 300 chars | Data ที่ส่งไป Webhook |
| `displayText` | String | No | 300 chars | ข้อความแสดงในแชท |
| `inputOption` | String | No | - | สำหรับ Datetime Picker |
| `fillInText` | String | No | 300 chars | ข้อความ pre-fill |

---

## 3. โครงสร้าง Postback Event

เมื่อผู้ใช้กดปุ่ม Postback Webhook จะได้รับ Event ดังนี้:

```json
{
  "type": "postback",
  "mode": "active",
  "timestamp": 1712345678901,
  "source": {
    "type": "user",
    "userId": "U1234567890abcdef"
  },
  "webhookEventId": "abc123",
  "deliveryContext": {
    "isRedelivery": false
  },
  "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA",
  "postback": {
    "data": "action=confirm&orderId=123",
    "params": {
      "datetime": "2024-01-15T14:00"
    }
  }
}
```

### Postback Object Fields

| Field | Description |
|-------|-------------|
| `data` | String ที่กำหนดใน action.data |
| `params` | Object สำหรับ Datetime Picker (Optional) |
| `params.date` | วันที่ (รูปแบบ YYYY-MM-DD) |
| `params.time` | เวลา (รูปแบบ HH:mm) |
| `params.datetime` | วันที่และเวลา (รูปแบบ YYYY-MM-DDThh:mm) |

---

## 4. Datetime Picker ใน Postback

### สร้าง Datetime Picker

```json
{
  "type": "postback",
  "label": "เลือกวันที่",
  "data": "action=select_date",
  "displayText": "เลือกวันนัดหมาย",
  "inputOption": "openCalendar"
}
```

### Datetime Picker Action

```json
{
  "type": "datetimepicker",
  "label": "เลือกวันที่จอง",
  "data": "action=booking_datetime",
  "mode": "datetime",
  "initial": "2024-06-01T10:00",
  "min": "2024-01-01T00:00",
  "max": "2024-12-31T23:59"
}
```

### Mode Options

| Mode | Format | Description |
|------|--------|-------------|
| `date` | YYYY-MM-DD | เลือกแค่วัน |
| `time` | HH:mm | เลือกแค่เวลา |
| `datetime` | YYYY-MM-DDThh:mm | เลือกทั้งคู่ |

### ตัวอย่าง Webhook Event จาก Datetime Picker

```json
{
  "type": "postback",
  "postback": {
    "data": "action=booking_datetime",
    "params": {
      "datetime": "2024-06-15T14:30"
    }
  }
}
```

---

## 5. การจัดการ Postback ใน Webhook

### Node.js Basic Handler

```javascript
// postbackHandler.js
const line = require('@line/bot-sdk');
const express = require('express');

const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET,
};

const client = new line.Client(config);
const app = express();

app.post('/webhook', line.middleware(config), async (req, res) => {
  const events = req.body.events;

  await Promise.all(
    events.map(event => handleEvent(event))
  );

  res.json({ status: 'ok' });
});

async function handleEvent(event) {
  switch (event.type) {
    case 'message':
      return handleMessage(event);
    case 'postback':
      return handlePostback(event);
    case 'follow':
      return handleFollow(event);
    default:
      console.log('Unknown event type:', event.type);
  }
}

async function handlePostback(event) {
  const userId = event.source.userId;
  const replyToken = event.replyToken;

  // Parse data
  const data = new URLSearchParams(event.postback.data);
  const action = data.get('action');

  // Get datetime params if available
  const datetimeParams = event.postback.params || {};

  console.log(`Postback from ${userId}: action=${action}`, datetimeParams);

  // Route ตาม action
  const handlers = {
    'products': handleProductsAction,
    'select_category': handleSelectCategory,
    'view_product': handleViewProduct,
    'add_to_cart': handleAddToCart,
    'checkout': handleCheckout,
    'confirm_order': handleConfirmOrder,
    'cancel_order': handleCancelOrder,
    'booking_datetime': handleBookingDatetime,
    'main_menu': handleMainMenu,
  };

  const handler = handlers[action];
  if (handler) {
    await handler(event, data, datetimeParams);
  } else {
    await client.replyMessage(replyToken, {
      type: 'text',
      text: `ไม่รู้จัก action: ${action}`,
    });
  }
}

async function handleProductsAction(event, data) {
  await client.replyMessage(event.replyToken, {
    type: 'text',
    text: 'กำลังแสดงสินค้า...',
  });
}

async function handleBookingDatetime(event, data, params) {
  const datetime = params.datetime || params.date || params.time;

  await client.replyMessage(event.replyToken, {
    type: 'text',
    text: `✅ รับทราบ! วันที่จอง: ${datetime}`,
  });
}

async function handleMainMenu(event, data) {
  // ส่งกลับไปเมนูหลัก
  await client.replyMessage(event.replyToken, createMainMenuMessage());
}

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## 6. Stateful Conversations ด้วย Postback

### State Management Pattern

```javascript
// stateManager.js
const redis = require('redis');
const client = redis.createClient({
  url: process.env.REDIS_URL,
});

class ConversationStateManager {
  constructor() {
    this.redisClient = client;
    this.ttl = 3600; // 1 ชั่วโมง
  }

  async setState(userId, state) {
    const key = `state:${userId}`;
    await this.redisClient.setEx(
      key,
      this.ttl,
      JSON.stringify(state)
    );
    console.log(`State set for ${userId}:`, state);
  }

  async getState(userId) {
    const key = `state:${userId}`;
    const value = await this.redisClient.get(key);
    return value ? JSON.parse(value) : null;
  }

  async clearState(userId) {
    const key = `state:${userId}`;
    await this.redisClient.del(key);
    console.log(`State cleared for ${userId}`);
  }

  async updateState(userId, updates) {
    const current = await this.getState(userId) || {};
    const newState = { ...current, ...updates };
    await this.setState(userId, newState);
    return newState;
  }
}

// In-memory fallback (สำหรับ development)
class InMemoryStateManager {
  constructor() {
    this.states = new Map();
  }

  async setState(userId, state) {
    this.states.set(userId, { ...state, updatedAt: Date.now() });
  }

  async getState(userId) {
    return this.states.get(userId) || null;
  }

  async clearState(userId) {
    this.states.delete(userId);
  }

  async updateState(userId, updates) {
    const current = await this.getState(userId) || {};
    const newState = { ...current, ...updates };
    await this.setState(userId, newState);
    return newState;
  }
}

module.exports = process.env.REDIS_URL
  ? new ConversationStateManager()
  : new InMemoryStateManager();
```

---

## 7. Multi-step Form ด้วย Postback

### ตัวอย่าง: แบบฟอร์มสมัครสมาชิก

```javascript
// registrationFlow.js
const line = require('@line/bot-sdk');
const stateManager = require('./stateManager');

const client = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
});

// State machine สำหรับการสมัครสมาชิก
const REGISTRATION_STATES = {
  START: 'registration_start',
  NAME: 'registration_name',
  PHONE: 'registration_phone',
  EMAIL: 'registration_email',
  GENDER: 'registration_gender',
  BIRTHDATE: 'registration_birthdate',
  CONFIRM: 'registration_confirm',
  DONE: 'registration_done',
};

// เริ่มการสมัคร
async function startRegistration(event) {
  const userId = event.source.userId;

  await stateManager.setState(userId, {
    flow: 'registration',
    step: REGISTRATION_STATES.NAME,
    data: {},
  });

  await client.replyMessage(event.replyToken, {
    type: 'text',
    text: '📝 เริ่มกระบวนการสมัครสมาชิก\n\nขั้นตอนที่ 1/5: กรุณาพิมพ์ชื่อ-นามสกุลของคุณ',
  });
}

// ขั้นตอนถามเพศ (ใช้ Postback)
async function askGender(event, partialData) {
  const userId = event.source.userId;

  await stateManager.updateState(userId, { step: REGISTRATION_STATES.GENDER });

  await client.replyMessage(event.replyToken, {
    type: 'template',
    altText: 'กรุณาเลือกเพศ',
    template: {
      type: 'buttons',
      text: 'ขั้นตอนที่ 4/5: กรุณาเลือกเพศ',
      actions: [
        {
          type: 'postback',
          label: '👨 ชาย',
          data: 'action=set_gender&gender=male',
          displayText: 'เพศชาย',
        },
        {
          type: 'postback',
          label: '👩 หญิง',
          data: 'action=set_gender&gender=female',
          displayText: 'เพศหญิง',
        },
        {
          type: 'postback',
          label: '⚧ ไม่ระบุ',
          data: 'action=set_gender&gender=other',
          displayText: 'ไม่ระบุเพศ',
        },
      ],
    },
  });
}

// ขั้นตอนเลือกวันเกิด (ใช้ Datetime Picker)
async function askBirthdate(event) {
  const userId = event.source.userId;

  await stateManager.updateState(userId, { step: REGISTRATION_STATES.BIRTHDATE });

  await client.replyMessage(event.replyToken, {
    type: 'template',
    altText: 'กรุณาเลือกวันเกิด',
    template: {
      type: 'buttons',
      text: 'ขั้นตอนที่ 5/5: กรุณาเลือกวันเกิด',
      actions: [
        {
          type: 'datetimepicker',
          label: '📅 เลือกวันเกิด',
          data: 'action=set_birthdate',
          mode: 'date',
          max: new Date().toISOString().split('T')[0],
          min: '1900-01-01',
        },
      ],
    },
  });
}

// ยืนยันข้อมูล
async function showConfirmation(event, userData) {
  const userId = event.source.userId;

  await stateManager.updateState(userId, { step: REGISTRATION_STATES.CONFIRM });

  const summary = [
    `📋 สรุปข้อมูลการสมัคร`,
    ``,
    `👤 ชื่อ: ${userData.name}`,
    `📞 โทร: ${userData.phone}`,
    `📧 อีเมล: ${userData.email}`,
    `⚧ เพศ: ${userData.gender === 'male' ? 'ชาย' : userData.gender === 'female' ? 'หญิง' : 'ไม่ระบุ'}`,
    `🎂 วันเกิด: ${userData.birthdate}`,
  ].join('\n');

  await client.replyMessage(event.replyToken, [
    {
      type: 'text',
      text: summary,
    },
    {
      type: 'template',
      altText: 'ยืนยันการสมัคร',
      template: {
        type: 'confirm',
        text: 'ข้อมูลถูกต้องใช่ไหม?',
        actions: [
          {
            type: 'postback',
            label: '✅ ยืนยัน',
            data: 'action=confirm_registration',
            displayText: 'ยืนยันการสมัคร',
          },
          {
            type: 'postback',
            label: '✏️ แก้ไข',
            data: 'action=edit_registration',
            displayText: 'แก้ไขข้อมูล',
          },
        ],
      },
    },
  ]);
}

// Main Message Handler
async function handleRegistrationMessage(event) {
  const userId = event.source.userId;
  const state = await stateManager.getState(userId);

  if (!state || state.flow !== 'registration') return false;

  const text = event.message.text;

  switch (state.step) {
    case REGISTRATION_STATES.NAME:
      await stateManager.updateState(userId, {
        data: { ...state.data, name: text },
        step: REGISTRATION_STATES.PHONE,
      });
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: `ขั้นตอนที่ 2/5: กรุณาพิมพ์เบอร์โทรศัพท์`,
      });
      break;

    case REGISTRATION_STATES.PHONE:
      if (!/^\d{9,10}$/.test(text.replace(/-/g, ''))) {
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: '❌ เบอร์โทรไม่ถูกต้อง กรุณาพิมพ์ใหม่ (เช่น 0812345678)',
        });
        return true;
      }
      await stateManager.updateState(userId, {
        data: { ...state.data, phone: text },
        step: REGISTRATION_STATES.EMAIL,
      });
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: 'ขั้นตอนที่ 3/5: กรุณาพิมพ์อีเมล',
      });
      break;

    case REGISTRATION_STATES.EMAIL:
      if (!/^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(text)) {
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: '❌ อีเมลไม่ถูกต้อง กรุณาพิมพ์ใหม่',
        });
        return true;
      }
      const updatedData = { ...state.data, email: text };
      await stateManager.updateState(userId, { data: updatedData });
      await askGender(event, updatedData);
      break;

    default:
      return false;
  }

  return true;
}

// Main Postback Handler
async function handleRegistrationPostback(event) {
  const userId = event.source.userId;
  const data = new URLSearchParams(event.postback.data);
  const action = data.get('action');

  if (action === 'start_registration') {
    await startRegistration(event);
    return true;
  }

  const state = await stateManager.getState(userId);
  if (!state || state.flow !== 'registration') return false;

  switch (action) {
    case 'set_gender':
      const gender = data.get('gender');
      await stateManager.updateState(userId, {
        data: { ...state.data, gender },
      });
      await askBirthdate(event);
      break;

    case 'set_birthdate':
      const birthdate = event.postback.params.date;
      const updatedData = { ...state.data, birthdate };
      await stateManager.updateState(userId, { data: updatedData });
      await showConfirmation(event, updatedData);
      break;

    case 'confirm_registration':
      await saveRegistration(userId, state.data);
      await stateManager.clearState(userId);
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: `🎉 ยินดีด้วย! คุณสมัครสมาชิกสำเร็จแล้ว\n\nยินดีต้อนรับ ${state.data.name}!`,
      });
      break;

    case 'edit_registration':
      await stateManager.setState(userId, {
        flow: 'registration',
        step: REGISTRATION_STATES.NAME,
        data: {},
      });
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: '✏️ เริ่มใหม่ตั้งแต่ต้น\n\nขั้นตอนที่ 1/5: กรุณาพิมพ์ชื่อ-นามสกุล',
      });
      break;

    default:
      return false;
  }

  return true;
}

async function saveRegistration(userId, data) {
  console.log('Saving registration:', { userId, data });
  // บันทึกลง Database
}

module.exports = { handleRegistrationMessage, handleRegistrationPostback };
```

---

## 8. Booking Flow ด้วย Postback

### Restaurant Booking System

```javascript
// bookingFlow.js
const line = require('@line/bot-sdk');
const stateManager = require('./stateManager');

const client = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
});

const BOOKING_STEPS = {
  START: 'booking_start',
  DATE: 'booking_date',
  TIME: 'booking_time',
  GUESTS: 'booking_guests',
  TABLE_TYPE: 'booking_table_type',
  SPECIAL_REQUEST: 'booking_special_request',
  CONFIRM: 'booking_confirm',
};

// สร้างข้อความเลือกวันที่
function createDateSelectionMessage() {
  const today = new Date();
  const items = [];

  for (let i = 1; i <= 7; i++) {
    const date = new Date(today);
    date.setDate(today.getDate() + i);

    const dayNames = ['อา', 'จ', 'อ', 'พ', 'พฤ', 'ศ', 'ส'];
    const dayName = dayNames[date.getDay()];
    const dateStr = `${date.getDate()}/${date.getMonth() + 1}`;

    items.push({
      type: 'action',
      action: {
        type: 'postback',
        label: `${dayName} ${dateStr}`,
        data: `action=set_booking_date&date=${date.toISOString().split('T')[0]}`,
        displayText: `เลือกวัน ${dateStr}`,
      },
    });
  }

  return {
    type: 'text',
    text: '📅 เลือกวันที่ต้องการจอง:',
    quickReply: {
      items: items.slice(0, 13),
    },
  };
}

// เลือกจำนวนคน
function createGuestCountMessage(selectedDate) {
  const counts = [1, 2, 3, 4, 5, 6, 7, 8, 10];

  return {
    type: 'text',
    text: `📅 วันที่: ${selectedDate}\n\n👥 เลือกจำนวนผู้ร่วมรับประทาน:`,
    quickReply: {
      items: [
        ...counts.map(count => ({
          type: 'action',
          action: {
            type: 'postback',
            label: `${count} คน`,
            data: `action=set_guests&count=${count}`,
            displayText: `${count} คน`,
          },
        })),
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '← เลือกวันใหม่',
            data: 'action=back_to_date',
          },
        },
      ],
    },
  };
}

// เลือกประเภทโต๊ะ
function createTableTypeMessage() {
  const tableTypes = [
    { label: '🪑 โต๊ะทั่วไป', type: 'regular', price: 'ไม่มีค่าใช้จ่าย' },
    { label: '🎉 โต๊ะ VIP', type: 'vip', price: '฿500/ชม.' },
    { label: '🌿 โต๊ะกลางแจ้ง', type: 'outdoor', price: 'ไม่มีค่าใช้จ่าย' },
    { label: '🔒 ห้องส่วนตัว', type: 'private', price: '฿1000/ชม.' },
  ];

  return {
    type: 'flex',
    altText: 'เลือกประเภทโต๊ะ',
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: '🍽️ เลือกประเภทโต๊ะ',
            size: 'xl',
            weight: 'bold',
          },
        ],
        backgroundColor: '#FF6B6B',
        paddingAll: '15px',
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: tableTypes.map(table => ({
          type: 'box',
          layout: 'horizontal',
          contents: [
            {
              type: 'button',
              action: {
                type: 'postback',
                label: table.label,
                data: `action=set_table_type&type=${table.type}`,
              },
              style: 'secondary',
              flex: 2,
            },
            {
              type: 'text',
              text: table.price,
              size: 'sm',
              color: '#888888',
              align: 'end',
              flex: 1,
              offsetStart: '10px',
            },
          ],
          paddingBottom: '10px',
        })),
      },
    },
  };
}

// สรุปการจอง
function createBookingSummaryMessage(bookingData) {
  const {
    date,
    time,
    guests,
    tableType,
    specialRequest,
  } = bookingData;

  const tableNames = {
    regular: 'โต๊ะทั่วไป',
    vip: 'โต๊ะ VIP',
    outdoor: 'โต๊ะกลางแจ้ง',
    private: 'ห้องส่วนตัว',
  };

  return {
    type: 'flex',
    altText: 'ยืนยันการจอง',
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: '📋 สรุปการจองโต๊ะ',
            size: 'xl',
            weight: 'bold',
            color: '#FFFFFF',
          },
        ],
        backgroundColor: '#2C5282',
        paddingAll: '15px',
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          { type: 'text', text: `📅 วันที่: ${date}`, size: 'md', margin: 'sm' },
          { type: 'text', text: `⏰ เวลา: ${time}`, size: 'md', margin: 'sm' },
          { type: 'text', text: `👥 จำนวน: ${guests} คน`, size: 'md', margin: 'sm' },
          { type: 'text', text: `🪑 ประเภท: ${tableNames[tableType]}`, size: 'md', margin: 'sm' },
          specialRequest ? { type: 'text', text: `📝 หมายเหตุ: ${specialRequest}`, size: 'md', margin: 'sm', wrap: true } : null,
        ].filter(Boolean),
      },
      footer: {
        type: 'box',
        layout: 'horizontal',
        contents: [
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '✅ ยืนยัน',
              data: 'action=confirm_booking',
            },
            style: 'primary',
            color: '#27AE60',
          },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '✏️ แก้ไข',
              data: 'action=edit_booking',
            },
            style: 'secondary',
            margin: 'sm',
          },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '❌ ยกเลิก',
              data: 'action=cancel_booking',
            },
            style: 'secondary',
            color: '#E74C3C',
            margin: 'sm',
          },
        ],
      },
    },
  };
}

// Booking Flow Handler
async function handleBookingFlow(event) {
  const userId = event.source.userId;
  const data = new URLSearchParams(event.postback.data);
  const action = data.get('action');

  if (action === 'start_booking') {
    await stateManager.setState(userId, {
      flow: 'booking',
      step: BOOKING_STEPS.DATE,
      data: {},
    });
    await client.replyMessage(event.replyToken, createDateSelectionMessage());
    return true;
  }

  const state = await stateManager.getState(userId);
  if (!state || state.flow !== 'booking') return false;

  switch (action) {
    case 'set_booking_date':
      const date = data.get('date');
      await stateManager.updateState(userId, {
        data: { ...state.data, date },
        step: BOOKING_STEPS.TIME,
      });
      // แสดงตัวเลือกเวลา
      await client.replyMessage(event.replyToken, {
        type: 'template',
        altText: 'เลือกเวลา',
        template: {
          type: 'buttons',
          text: `📅 ${date}\n⏰ เลือกเวลา:`,
          actions: [
            { type: 'postback', label: '12:00', data: `action=set_time&time=12:00` },
            { type: 'postback', label: '13:00', data: `action=set_time&time=13:00` },
            { type: 'postback', label: '18:00', data: `action=set_time&time=18:00` },
            { type: 'postback', label: '19:00', data: `action=set_time&time=19:00` },
          ],
        },
      });
      break;

    case 'set_time':
      const time = data.get('time');
      await stateManager.updateState(userId, {
        data: { ...state.data, time },
        step: BOOKING_STEPS.GUESTS,
      });
      await client.replyMessage(event.replyToken, createGuestCountMessage(state.data.date));
      break;

    case 'set_guests':
      const guests = parseInt(data.get('count'));
      await stateManager.updateState(userId, {
        data: { ...state.data, guests },
        step: BOOKING_STEPS.TABLE_TYPE,
      });
      await client.replyMessage(event.replyToken, createTableTypeMessage());
      break;

    case 'set_table_type':
      const tableType = data.get('type');
      const bookingData = { ...state.data, tableType };
      await stateManager.updateState(userId, {
        data: bookingData,
        step: BOOKING_STEPS.CONFIRM,
      });
      await client.replyMessage(event.replyToken, createBookingSummaryMessage(bookingData));
      break;

    case 'confirm_booking':
      const bookingId = await saveBooking(userId, state.data);
      await stateManager.clearState(userId);
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: `🎉 จองโต๊ะสำเร็จ!\n\n🔖 หมายเลขการจอง: ${bookingId}\n\nเราจะส่งการยืนยันทาง SMS อีกครั้ง`,
      });
      break;

    case 'cancel_booking':
      await stateManager.clearState(userId);
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: '❌ ยกเลิกการจองแล้ว',
      });
      break;

    case 'back_to_date':
      await client.replyMessage(event.replyToken, createDateSelectionMessage());
      break;

    default:
      return false;
  }

  return true;
}

async function saveBooking(userId, data) {
  const bookingId = `BK${Date.now()}`;
  console.log('Saving booking:', { bookingId, userId, data });
  return bookingId;
}

module.exports = { handleBookingFlow };
```

---

## 9. Ordering Flow

```javascript
// orderingFlow.js

const CART_KEY = (userId) => `cart:${userId}`;

class ShoppingCart {
  constructor(stateManager) {
    this.sm = stateManager;
  }

  async addItem(userId, product) {
    const state = await this.sm.getState(userId) || { cart: [] };
    const cart = state.cart || [];
    
    const existing = cart.find(item => item.id === product.id);
    if (existing) {
      existing.qty += 1;
    } else {
      cart.push({ ...product, qty: 1 });
    }

    await this.sm.setState(userId, { ...state, cart });
    return cart;
  }

  async removeItem(userId, productId) {
    const state = await this.sm.getState(userId) || { cart: [] };
    const cart = (state.cart || []).filter(item => item.id !== productId);
    await this.sm.setState(userId, { ...state, cart });
    return cart;
  }

  async getCart(userId) {
    const state = await this.sm.getState(userId);
    return state?.cart || [];
  }

  async clearCart(userId) {
    const state = await this.sm.getState(userId) || {};
    await this.sm.setState(userId, { ...state, cart: [] });
  }

  async getTotal(userId) {
    const cart = await this.getCart(userId);
    return cart.reduce((sum, item) => sum + item.price * item.qty, 0);
  }
}

// สร้าง Flex Message สำหรับตะกร้าสินค้า
function createCartMessage(cart, total) {
  if (cart.length === 0) {
    return {
      type: 'text',
      text: '🛒 ตะกร้าของคุณว่างเปล่า\nกดปุ่มด้านล่างเพื่อดูสินค้า',
      quickReply: {
        items: [
          {
            type: 'action',
            action: {
              type: 'postback',
              label: '🛍️ ดูสินค้า',
              data: 'action=products',
            },
          },
        ],
      },
    };
  }

  const cartItems = cart.map(item => ({
    type: 'box',
    layout: 'horizontal',
    contents: [
      {
        type: 'text',
        text: item.name,
        flex: 3,
        size: 'sm',
      },
      {
        type: 'text',
        text: `x${item.qty}`,
        flex: 1,
        align: 'center',
        size: 'sm',
      },
      {
        type: 'text',
        text: `฿${item.price * item.qty}`,
        flex: 2,
        align: 'end',
        size: 'sm',
      },
      {
        type: 'button',
        action: {
          type: 'postback',
          label: '✕',
          data: `action=remove_item&id=${item.id}`,
        },
        style: 'secondary',
        height: 'sm',
        flex: 1,
      },
    ],
    paddingBottom: '5px',
  }));

  return {
    type: 'flex',
    altText: `ตะกร้า ${cart.length} ชิ้น รวม ฿${total}`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [
          { type: 'text', text: '🛒 ตะกร้าสินค้า', weight: 'bold', size: 'xl', color: '#FFFFFF' },
          { type: 'text', text: `${cart.length} รายการ`, size: 'sm', color: '#FFFFFFAA' },
        ],
        backgroundColor: '#2C5282',
        paddingAll: '15px',
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          ...cartItems,
          { type: 'separator', margin: 'md' },
          {
            type: 'box',
            layout: 'horizontal',
            contents: [
              { type: 'text', text: 'รวมทั้งสิ้น', weight: 'bold', size: 'lg' },
              { type: 'text', text: `฿${total}`, weight: 'bold', size: 'lg', align: 'end', color: '#E74C3C' },
            ],
            margin: 'md',
          },
        ],
      },
      footer: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '💳 ดำเนินการชำระเงิน',
              data: 'action=checkout',
            },
            style: 'primary',
            color: '#27AE60',
          },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '🛍️ ช้อปต่อ',
              data: 'action=products',
            },
            style: 'secondary',
            margin: 'sm',
          },
        ],
      },
    },
  };
}

module.exports = { ShoppingCart, createCartMessage };
```

---

## 10. State Machine Pattern

```javascript
// stateMachine.js

class StateMachine {
  constructor(transitions) {
    this.transitions = transitions;
  }

  canTransition(currentState, event) {
    const state = this.transitions[currentState];
    if (!state) return false;
    return event in state.transitions;
  }

  getNextState(currentState, event) {
    const state = this.transitions[currentState];
    if (!state) return null;
    return state.transitions[event] || null;
  }
}

// Order State Machine
const orderStateMachine = new StateMachine({
  idle: {
    transitions: {
      start_order: 'browsing',
    },
  },
  browsing: {
    transitions: {
      add_to_cart: 'cart',
      cancel: 'idle',
    },
  },
  cart: {
    transitions: {
      checkout: 'checkout',
      continue_shopping: 'browsing',
      clear_cart: 'idle',
    },
  },
  checkout: {
    transitions: {
      enter_address: 'address',
      back_to_cart: 'cart',
      cancel: 'idle',
    },
  },
  address: {
    transitions: {
      select_payment: 'payment',
      back: 'checkout',
    },
  },
  payment: {
    transitions: {
      confirm_payment: 'confirming',
      back: 'address',
      cancel: 'idle',
    },
  },
  confirming: {
    transitions: {
      payment_success: 'completed',
      payment_failed: 'payment',
    },
  },
  completed: {
    transitions: {
      new_order: 'idle',
    },
  },
});

// Postback Handler with State Machine
async function handleOrderStateMachine(event) {
  const userId = event.source.userId;
  const data = new URLSearchParams(event.postback.data);
  const action = data.get('action');

  // ดึง state ปัจจุบัน
  const state = await stateManager.getState(userId) || { currentState: 'idle', data: {} };
  const currentState = state.currentState || 'idle';

  // ตรวจสอบว่า transition ถูกต้อง
  if (!orderStateMachine.canTransition(currentState, action)) {
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: `⚠️ ไม่สามารถดำเนินการได้ในขั้นตอนนี้ (สถานะ: ${currentState})`,
    });
    return;
  }

  // เปลี่ยน state
  const nextState = orderStateMachine.getNextState(currentState, action);

  // อัพเดท state
  await stateManager.setState(userId, {
    ...state,
    currentState: nextState,
    previousState: currentState,
  });

  // ทำงานตาม state ใหม่
  await executeStateAction(event, nextState, data, state);
}

async function executeStateAction(event, state, data, previousStateData) {
  switch (state) {
    case 'browsing':
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: '🛍️ กำลังแสดงสินค้า...',
      });
      break;
    case 'cart':
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: '🛒 แสดงตะกร้าสินค้า...',
      });
      break;
    case 'completed':
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: '✅ สั่งซื้อสำเร็จ! ขอบคุณที่ใช้บริการ',
      });
      break;
    // ... other states
  }
}
```

---

## 11. Error Handling และ Validation

```javascript
// postbackErrorHandler.js

class PostbackValidator {
  static validateData(data, rules) {
    const errors = [];

    for (const [field, rule] of Object.entries(rules)) {
      const value = data.get(field);

      if (rule.required && !value) {
        errors.push(`Missing required field: ${field}`);
        continue;
      }

      if (value && rule.type === 'number') {
        const num = parseInt(value);
        if (isNaN(num)) {
          errors.push(`${field} must be a number`);
        } else if (rule.min !== undefined && num < rule.min) {
          errors.push(`${field} must be >= ${rule.min}`);
        } else if (rule.max !== undefined && num > rule.max) {
          errors.push(`${field} must be <= ${rule.max}`);
        }
      }

      if (value && rule.enum && !rule.enum.includes(value)) {
        errors.push(`${field} must be one of: ${rule.enum.join(', ')}`);
      }
    }

    return errors;
  }
}

// ตัวอย่างการใช้ Validator
async function handleValidatedPostback(event) {
  const data = new URLSearchParams(event.postback.data);
  const action = data.get('action');

  if (action === 'set_guests') {
    const errors = PostbackValidator.validateData(data, {
      count: { required: true, type: 'number', min: 1, max: 20 },
    });

    if (errors.length > 0) {
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: `❌ ข้อมูลไม่ถูกต้อง:\n${errors.join('\n')}`,
      });
      return;
    }

    const count = parseInt(data.get('count'));
    // ดำเนินการต่อ...
  }
}

// Retry Logic
async function handlePostbackWithRetry(event, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      await handlePostback(event);
      return;
    } catch (error) {
      if (attempt === maxRetries) {
        console.error(`Failed after ${maxRetries} attempts:`, error);
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: '❌ เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง',
        });
      } else {
        console.warn(`Attempt ${attempt} failed, retrying...`);
        await new Promise(resolve => setTimeout(resolve, 1000 * attempt));
      }
    }
  }
}
```

---

## 12. Best Practices

### Data Format ที่แนะนำ

```javascript
// ใช้ URLSearchParams format
const data = 'action=confirm_order&orderId=123&amount=500';

// ใช้ JSON (ถ้า data ไม่เกิน 300 chars)
const data2 = JSON.stringify({ action: 'confirm', id: 123 });

// ✅ ดี - สั้น เข้าใจง่าย
'action=view_product&id=P001'

// ❌ ไม่ดี - ยาวเกินไป
'action=view_a_specific_product_details&productIdentifier=PROD-00123456'
```

### Security Best Practices

```javascript
// ✅ ตรวจสอบ userId เสมอ
async function handlePostback(event) {
  const userId = event.source.userId;
  
  // ตรวจสอบว่า User มีสิทธิ์ทำ action นี้
  const hasPermission = await checkUserPermission(userId, action);
  if (!hasPermission) {
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: '⛔ คุณไม่มีสิทธิ์ดำเนินการนี้',
    });
    return;
  }
}

// ✅ ไม่ฝัง Sensitive Data ใน postback data
// ❌ ผิด
'action=pay&creditCardNumber=1234567890123456'

// ✅ ถูก - ใช้ reference ID แทน
'action=pay&paymentSessionId=sess_abc123'
```

---

## สรุป

Postback Events เป็นหัวใจสำคัญของ Interactive LINE Bot ที่ช่วย:
- สร้าง Stateful Conversations
- สร้าง Multi-step Forms
- ควบคุม Flow การสนทนา
- ปกป้องข้อมูล Sensitive

**Key Points:**
- Data สูงสุด 300 ตัวอักษร
- ใช้ URLSearchParams format
- จัดการ State ด้วย Redis หรือ In-memory
- Validate Input เสมอ
- ใช้ State Machine สำหรับ Flow ที่ซับซ้อน
