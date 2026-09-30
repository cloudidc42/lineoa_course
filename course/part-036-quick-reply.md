# Part 36: Quick Reply - การตอบกลับด่วนใน LINE OA

## บทนำ

Quick Reply เป็นฟีเจอร์ที่ช่วยให้ผู้ใช้สามารถตอบกลับข้อความได้อย่างรวดเร็ว โดยการแสดงปุ่มตัวเลือกที่ด้านล่างหน้าจอแชท ซึ่งจะหายไปหลังจากที่ผู้ใช้กดปุ่มหรือพิมพ์ข้อความ ทำให้ประสบการณ์การสนทนามีความลื่นไหลและสะดวกมากยิ่งขึ้น

---

## 1. Quick Reply คืออะไร?

Quick Reply คือชุดปุ่มที่แสดงบริเวณแถบด้านล่างของหน้าจอแชท LINE ผู้ใช้สามารถกดปุ่มเหล่านี้เพื่อส่งข้อความหรือดำเนินการได้ทันที โดยไม่ต้องพิมพ์ข้อความเอง

### คุณสมบัติหลักของ Quick Reply

```
┌─────────────────────────────────────────┐
│  LINE Chat Window                        │
│                                          │
│  Bot: คุณต้องการบริการอะไร?             │
│                                          │
│  ┌──────────────────────────────────┐   │
│  │  ถามราคา  │  ดูสินค้า  │  ติดต่อ │   │
│  └──────────────────────────────────┘   │
│  [พิมพ์ข้อความ...]              [ส่ง]   │
└─────────────────────────────────────────┘
```

**ข้อดีของ Quick Reply:**
- ลดการพิมพ์ของผู้ใช้
- ควบคุมการสนทนาได้ดีขึ้น
- ลดข้อผิดพลาดจากการพิมพ์
- ทำให้ UX ดีขึ้น
- เหมาะกับการใช้งานบนมือถือ

**ข้อจำกัด:**
- แสดงได้สูงสุด 13 ปุ่มต่อหนึ่งข้อความ
- ปุ่มจะหายไปเมื่อผู้ใช้กดหรือพิมพ์ข้อความ
- ใช้ได้กับ Messaging API เท่านั้น

---

## 2. โครงสร้างของ Quick Reply

Quick Reply ถูกกำหนดไว้ใน property `quickReply` ของ message object ดังนี้:

```json
{
  "type": "text",
  "text": "คุณต้องการบริการอะไร?",
  "quickReply": {
    "items": [
      {
        "type": "action",
        "imageUrl": "https://example.com/icon1.png",
        "action": {
          "type": "message",
          "label": "ถามราคา",
          "text": "ต้องการถามราคา"
        }
      },
      {
        "type": "action",
        "imageUrl": "https://example.com/icon2.png",
        "action": {
          "type": "message",
          "label": "ดูสินค้า",
          "text": "ต้องการดูสินค้า"
        }
      }
    ]
  }
}
```

### โครงสร้าง QuickReply Object

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `items` | Array | Yes | รายการปุ่ม Quick Reply (สูงสุด 13 items) |

### โครงสร้าง QuickReply Item

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | String | Yes | ต้องเป็น `"action"` เสมอ |
| `imageUrl` | String | No | URL ของไอคอน (PNG, JPEG, ขนาด 24x24 px) |
| `action` | Object | Yes | Action ที่จะเกิดขึ้นเมื่อกดปุ่ม |

---

## 3. Action Types สำหรับ Quick Reply

Quick Reply รองรับ Action หลายประเภท:

### 3.1 Message Action

ส่งข้อความเมื่อกดปุ่ม:

```json
{
  "type": "action",
  "action": {
    "type": "message",
    "label": "ยืนยัน",
    "text": "ฉันยืนยันการสั่งซื้อ"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | String | `"message"` |
| `label` | String | ข้อความบนปุ่ม (สูงสุด 20 ตัวอักษร) |
| `text` | String | ข้อความที่จะส่ง (สูงสุด 300 ตัวอักษร) |

### 3.2 Postback Action

ส่ง Postback Event ไปยัง Webhook:

```json
{
  "type": "action",
  "action": {
    "type": "postback",
    "label": "เลือก",
    "data": "action=select&item=coffee",
    "displayText": "เลือกกาแฟแล้ว"
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `type` | String | `"postback"` |
| `label` | String | ข้อความบนปุ่ม |
| `data` | String | Data ที่จะส่งไปยัง Webhook |
| `displayText` | String | ข้อความที่แสดงในแชท (Optional) |

### 3.3 URI Action

เปิด URL เมื่อกดปุ่ม:

```json
{
  "type": "action",
  "action": {
    "type": "uri",
    "label": "ดูเพิ่มเติม",
    "uri": "https://example.com/product/123"
  }
}
```

### 3.4 Datetime Picker Action

เปิดปฏิทินหรือเลือกเวลา:

```json
{
  "type": "action",
  "action": {
    "type": "datetimepicker",
    "label": "เลือกวันที่",
    "data": "action=booking_date",
    "mode": "date",
    "initial": "2024-01-15",
    "min": "2024-01-01",
    "max": "2024-12-31"
  }
}
```

| Mode | Description |
|------|-------------|
| `date` | เลือกวันที่ |
| `time` | เลือกเวลา |
| `datetime` | เลือกวันที่และเวลา |

### 3.5 Camera Action

เปิดกล้องถ่ายภาพ:

```json
{
  "type": "action",
  "action": {
    "type": "camera",
    "label": "ถ่ายภาพ"
  }
}
```

### 3.6 Camera Roll Action

เปิดคลังภาพ:

```json
{
  "type": "action",
  "action": {
    "type": "cameraRoll",
    "label": "เลือกภาพ"
  }
}
```

### 3.7 Location Action

แชร์ตำแหน่ง:

```json
{
  "type": "action",
  "action": {
    "type": "location",
    "label": "แชร์ตำแหน่ง"
  }
}
```

---

## 4. ไอคอนสำหรับ Quick Reply

Quick Reply รองรับการแสดงไอคอนบนปุ่ม ซึ่งช่วยให้ผู้ใช้เข้าใจได้ง่ายขึ้น

### ข้อกำหนดของไอคอน

| ข้อกำหนด | รายละเอียด |
|-----------|------------|
| รูปแบบไฟล์ | HTTPS URL เท่านั้น |
| นามสกุล | PNG หรือ JPEG |
| ขนาดที่แนะนำ | 24 x 24 pixels |
| ขนาดสูงสุด | 1MB |

```json
{
  "type": "action",
  "imageUrl": "https://cdn.example.com/icons/coffee.png",
  "action": {
    "type": "message",
    "label": "กาแฟ",
    "text": "ต้องการสั่งกาแฟ"
  }
}
```

**หมายเหตุ:** ถ้าไม่ใส่ `imageUrl` ปุ่มจะแสดงแค่ข้อความ

### การสร้างไอคอนที่เหมาะสม

```
ขนาดที่แนะนำ: 24x24 px
ขนาดสูงสุด:  1MB
พื้นหลัง:    ควรโปร่งใส (transparent)
สี:          ควรเห็นชัดบนพื้นหลังขาว
```

---

## 5. จำนวนสูงสุด 13 Items

Quick Reply รองรับได้สูงสุด **13 items** ต่อหนึ่งข้อความ

```
┌─────────────────────────────────────────────────────┐
│  ปุ่ม 1  │  ปุ่ม 2  │  ปุ่ม 3  │  ปุ่ม 4  │  ปุ่ม 5 │
│  ปุ่ม 6  │  ปุ่ม 7  │  ปุ่ม 8  │  ปุ่ม 9  │  ปุ่ม 10│
│  ปุ่ม 11 │  ปุ่ม 12 │  ปุ่ม 13 │                   │
└─────────────────────────────────────────────────────┘
```

**คำแนะนำ:** แม้จะรองรับ 13 ปุ่ม แต่ควรใช้ 5-8 ปุ่มเพื่อความสะดวกของผู้ใช้

---

## 6. การสร้าง Quick Reply ที่คำนึงถึง Context

Quick Reply ที่ดีควรสอดคล้องกับบริบทของการสนทนา

### ตัวอย่างการออกแบบตาม Context

```
สถานการณ์: ผู้ใช้ทักมาครั้งแรก
┌─────────────────────────────────────┐
│ ยินดีต้อนรับ! คุณต้องการอะไร?      │
│                                     │
│ [ดูสินค้า] [ติดต่อ] [โปรโมชั่น]   │
└─────────────────────────────────────┘

สถานการณ์: ผู้ใช้เลือกดูสินค้าแล้ว
┌─────────────────────────────────────┐
│ สินค้าประเภทใดที่คุณสนใจ?          │
│                                     │
│ [เสื้อผ้า] [อิเล็กทรอนิกส์] [อาหาร]│
│ [← กลับ]                           │
└─────────────────────────────────────┘

สถานการณ์: ผู้ใช้เลือกสินค้าแล้ว
┌─────────────────────────────────────┐
│ ต้องการดำเนินการอะไรกับสินค้านี้?  │
│                                     │
│ [สั่งซื้อ] [บันทึก] [แชร์] [← กลับ]│
└─────────────────────────────────────┘
```

### หลักการออกแบบ Context-Aware Quick Reply

1. **ติดตาม State** - รู้ว่าผู้ใช้อยู่ที่ขั้นตอนไหน
2. **มีปุ่ม "กลับ"** - เสมอเมื่ออยู่ใน submenu
3. **จำกัดจำนวน** - ไม่ควรมีมากกว่า 6-8 ปุ่ม
4. **ข้อความชัดเจน** - ปุ่มต้องสื่อความหมายชัดเจน
5. **เหมาะกับ Context** - แสดงเฉพาะตัวเลือกที่เกี่ยวข้อง

---

## 7. การออกแบบ Conversation Flow ด้วย Quick Reply

### Flow Diagram ตัวอย่าง

```
[เริ่มต้น]
    │
    ▼
"ยินดีต้อนรับ!"
[ดูสินค้า] [ติดต่อ] [โปรโมชั่น]
    │           │           │
    ▼           ▼           ▼
"เลือกประเภท" "ช่องทางใด?" "โปรของวันนี้"
[เสื้อ][กางเกง] [โทร][LINE] [รับโปร]
    │
    ▼
"เลือกขนาด"
[S][M][L][XL]
    │
    ▼
"ยืนยันการสั่งซื้อ?"
[ยืนยัน] [ยกเลิก]
    │
    ▼
"ขอบคุณ! คำสั่งซื้อ #123"
```

---

## 8. ตัวอย่างสมบูรณ์: Customer Service Menu

### Node.js Implementation

```javascript
// customerServiceBot.js
const line = require('@line/bot-sdk');

const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET,
};

const client = new line.Client(config);

// ฟังก์ชันสร้าง Quick Reply สำหรับ Customer Service
function createCustomerServiceMenu() {
  return {
    type: 'text',
    text: '🎯 สวัสดีครับ! ยินดีต้อนรับสู่บริการลูกค้า\nคุณต้องการความช่วยเหลืออะไร?',
    quickReply: {
      items: [
        {
          type: 'action',
          imageUrl: 'https://cdn.example.com/icons/order.png',
          action: {
            type: 'postback',
            label: '📦 ติดตามคำสั่งซื้อ',
            data: 'action=track_order',
            displayText: 'ติดตามคำสั่งซื้อ',
          },
        },
        {
          type: 'action',
          imageUrl: 'https://cdn.example.com/icons/return.png',
          action: {
            type: 'postback',
            label: '↩️ คืน/เปลี่ยนสินค้า',
            data: 'action=return_item',
            displayText: 'ต้องการคืน/เปลี่ยนสินค้า',
          },
        },
        {
          type: 'action',
          imageUrl: 'https://cdn.example.com/icons/payment.png',
          action: {
            type: 'postback',
            label: '💳 ปัญหาการชำระ',
            data: 'action=payment_issue',
            displayText: 'มีปัญหาเรื่องการชำระเงิน',
          },
        },
        {
          type: 'action',
          imageUrl: 'https://cdn.example.com/icons/product.png',
          action: {
            type: 'postback',
            label: '🛍️ สอบถามสินค้า',
            data: 'action=product_inquiry',
            displayText: 'สอบถามเรื่องสินค้า',
          },
        },
        {
          type: 'action',
          imageUrl: 'https://cdn.example.com/icons/shipping.png',
          action: {
            type: 'postback',
            label: '🚚 ปัญหาการจัดส่ง',
            data: 'action=shipping_issue',
            displayText: 'มีปัญหาเรื่องการจัดส่ง',
          },
        },
        {
          type: 'action',
          imageUrl: 'https://cdn.example.com/icons/agent.png',
          action: {
            type: 'postback',
            label: '👤 พูดคุยกับเจ้าหน้าที่',
            data: 'action=live_agent',
            displayText: 'ต้องการพูดคุยกับเจ้าหน้าที่',
          },
        },
      ],
    },
  };
}

// ฟังก์ชันจัดการ Postback Events
async function handlePostback(event) {
  const userId = event.source.userId;
  const data = new URLSearchParams(event.postback.data);
  const action = data.get('action');

  switch (action) {
    case 'track_order':
      await client.replyMessage(event.replyToken, createTrackOrderFlow());
      break;
    case 'return_item':
      await client.replyMessage(event.replyToken, createReturnFlow());
      break;
    case 'payment_issue':
      await client.replyMessage(event.replyToken, createPaymentFlow());
      break;
    case 'product_inquiry':
      await client.replyMessage(event.replyToken, createProductInquiryFlow());
      break;
    case 'shipping_issue':
      await client.replyMessage(event.replyToken, createShippingFlow());
      break;
    case 'live_agent':
      await client.replyMessage(event.replyToken, createLiveAgentTransfer());
      break;
    default:
      await client.replyMessage(event.replyToken, createCustomerServiceMenu());
  }
}

// ฟังก์ชัน Track Order Flow
function createTrackOrderFlow() {
  return {
    type: 'text',
    text: '📦 กรุณาพิมพ์หมายเลขคำสั่งซื้อของคุณ\n(ตัวอย่าง: ORD-2024-00123)\n\nหรือเลือกตัวเลือกด้านล่าง:',
    quickReply: {
      items: [
        {
          type: 'action',
          action: {
            type: 'message',
            label: '📋 ดูคำสั่งซื้อล่าสุด',
            text: 'ดูคำสั่งซื้อล่าสุดของฉัน',
          },
        },
        {
          type: 'action',
          action: {
            type: 'message',
            label: '❓ ไม่มีหมายเลข',
            text: 'ฉันไม่มีหมายเลขคำสั่งซื้อ',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '← กลับเมนูหลัก',
            data: 'action=main_menu',
            displayText: 'กลับไปเมนูหลัก',
          },
        },
      ],
    },
  };
}

// ฟังก์ชัน Return Flow
function createReturnFlow() {
  return {
    type: 'text',
    text: '↩️ คุณต้องการดำเนินการอะไร?',
    quickReply: {
      items: [
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '🔄 เปลี่ยนสินค้า',
            data: 'action=exchange_item',
            displayText: 'ต้องการเปลี่ยนสินค้า',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '💰 คืนเงิน',
            data: 'action=refund',
            displayText: 'ต้องการคืนเงิน',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '← กลับ',
            data: 'action=main_menu',
            displayText: 'กลับไปเมนูหลัก',
          },
        },
      ],
    },
  };
}

// Webhook Handler
const express = require('express');
const app = express();

app.post('/webhook', line.middleware(config), async (req, res) => {
  const events = req.body.events;

  await Promise.all(
    events.map(async (event) => {
      if (event.type === 'message' && event.message.type === 'text') {
        const text = event.message.text.toLowerCase();

        if (text.includes('สวัสดี') || text.includes('hello') || text.includes('hi')) {
          await client.replyMessage(event.replyToken, createCustomerServiceMenu());
        }
      } else if (event.type === 'postback') {
        await handlePostback(event);
      }
    })
  );

  res.json({ status: 'ok' });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## 9. ตัวอย่างสมบูรณ์: Product Category Selection

```javascript
// productCategoryBot.js

// สร้าง Category Menu หลัก
function createMainCategoryMenu() {
  return {
    type: 'text',
    text: '🛍️ เลือกหมวดหมู่สินค้าที่คุณสนใจ:',
    quickReply: {
      items: [
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '👕 เสื้อผ้าแฟชั่น',
            data: 'category=fashion',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '📱 อิเล็กทรอนิกส์',
            data: 'category=electronics',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '🏠 เฟอร์นิเจอร์',
            data: 'category=furniture',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '🍕 อาหาร & เครื่องดื่ม',
            data: 'category=food',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '💄 ความงาม',
            data: 'category=beauty',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '🐾 สัตว์เลี้ยง',
            data: 'category=pets',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '📚 หนังสือ',
            data: 'category=books',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '⚽ กีฬา',
            data: 'category=sports',
          },
        },
      ],
    },
  };
}

// สร้าง Sub-category Menu
function createSubCategoryMenu(category) {
  const subCategories = {
    fashion: {
      message: '👕 เลือกประเภทเสื้อผ้า:',
      items: [
        { label: '👔 เสื้อผู้ชาย', data: 'sub=mens_shirts' },
        { label: '👗 เสื้อผู้หญิง', data: 'sub=womens_tops' },
        { label: '👖 กางเกง', data: 'sub=pants' },
        { label: '👟 รองเท้า', data: 'sub=shoes' },
        { label: '👜 กระเป๋า', data: 'sub=bags' },
      ],
    },
    electronics: {
      message: '📱 เลือกประเภทอิเล็กทรอนิกส์:',
      items: [
        { label: '📱 โทรศัพท์มือถือ', data: 'sub=phones' },
        { label: '💻 คอมพิวเตอร์', data: 'sub=computers' },
        { label: '📺 ทีวี', data: 'sub=tv' },
        { label: '🎮 เกมมิ่ง', data: 'sub=gaming' },
        { label: '🎧 หูฟัง', data: 'sub=headphones' },
      ],
    },
  };

  const sub = subCategories[category] || {
    message: 'กรุณาเลือกหมวดหมู่:',
    items: [],
  };

  return {
    type: 'text',
    text: sub.message,
    quickReply: {
      items: [
        ...sub.items.map((item) => ({
          type: 'action',
          action: {
            type: 'postback',
            label: item.label,
            data: `category=${category}&${item.data}`,
          },
        })),
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '← กลับ',
            data: 'action=main_categories',
          },
        },
      ],
    },
  };
}

// Postback Handler
async function handleCategoryPostback(event) {
  const params = new URLSearchParams(event.postback.data);
  const category = params.get('category');
  const action = params.get('action');

  if (action === 'main_categories') {
    await client.replyMessage(event.replyToken, createMainCategoryMenu());
    return;
  }

  if (category && !params.get('sub')) {
    // แสดง sub-categories
    await client.replyMessage(event.replyToken, createSubCategoryMenu(category));
    return;
  }

  if (category && params.get('sub')) {
    // แสดงสินค้าในหมวดหมู่นั้น
    const sub = params.get('sub');
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: `🔍 กำลังค้นหาสินค้าในหมวด ${category}/${sub}...`,
    });
  }
}
```

---

## 10. ตัวอย่างสมบูรณ์: Time Slot Picker

```javascript
// timeSlotPicker.js

// สร้างตัวเลือกวันที่ (วันนี้, พรุ่งนี้, 3 วันข้างหน้า)
function createDateSelection() {
  const today = new Date();
  const dates = [];

  for (let i = 0; i < 7; i++) {
    const date = new Date(today);
    date.setDate(today.getDate() + i);

    const dayNames = ['อาทิตย์', 'จันทร์', 'อังคาร', 'พุธ', 'พฤหัส', 'ศุกร์', 'เสาร์'];
    const dayName = i === 0 ? 'วันนี้' : i === 1 ? 'พรุ่งนี้' : `${dayNames[date.getDay()]}`;
    const dateStr = `${date.getDate()}/${date.getMonth() + 1}`;

    dates.push({
      label: `${dayName} ${dateStr}`,
      data: `date=${date.toISOString().split('T')[0]}`,
    });
  }

  return {
    type: 'text',
    text: '📅 กรุณาเลือกวันที่ต้องการนัดหมาย:',
    quickReply: {
      items: dates.map((d) => ({
        type: 'action',
        action: {
          type: 'postback',
          label: d.label,
          data: d.data,
          displayText: `เลือกวันที่ ${d.label}`,
        },
      })),
    },
  };
}

// สร้างตัวเลือกช่วงเวลา
function createTimeSlotSelection(selectedDate) {
  const timeSlots = [
    { label: '🌅 09:00 - 10:00', slot: '09:00-10:00' },
    { label: '☀️ 10:00 - 11:00', slot: '10:00-11:00' },
    { label: '🌤️ 11:00 - 12:00', slot: '11:00-12:00' },
    { label: '🍽️ 13:00 - 14:00', slot: '13:00-14:00' },
    { label: '🌞 14:00 - 15:00', slot: '14:00-15:00' },
    { label: '☁️ 15:00 - 16:00', slot: '15:00-16:00' },
    { label: '🌆 16:00 - 17:00', slot: '16:00-17:00' },
  ];

  return {
    type: 'text',
    text: `📅 วันที่: ${selectedDate}\n⏰ กรุณาเลือกช่วงเวลา:`,
    quickReply: {
      items: [
        ...timeSlots.map((slot) => ({
          type: 'action',
          action: {
            type: 'postback',
            label: slot.label,
            data: `date=${selectedDate}&time=${slot.slot}`,
            displayText: `เลือกเวลา ${slot.slot}`,
          },
        })),
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '← เลือกวันใหม่',
            data: 'action=select_date',
            displayText: 'กลับไปเลือกวันใหม่',
          },
        },
      ],
    },
  };
}

// ยืนยันการนัดหมาย
function createConfirmBooking(date, time) {
  return {
    type: 'text',
    text: `📋 ยืนยันการนัดหมาย\n\n📅 วันที่: ${date}\n⏰ เวลา: ${time}\n\nต้องการยืนยันการนัดหมายนี้?`,
    quickReply: {
      items: [
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '✅ ยืนยัน',
            data: `action=confirm_booking&date=${date}&time=${time}`,
            displayText: 'ยืนยันการนัดหมาย',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '✏️ เปลี่ยนเวลา',
            data: `action=change_time&date=${date}`,
            displayText: 'เปลี่ยนช่วงเวลา',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '🗓️ เปลี่ยนวัน',
            data: 'action=select_date',
            displayText: 'เปลี่ยนวันที่',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '❌ ยกเลิก',
            data: 'action=cancel_booking',
            displayText: 'ยกเลิกการนัดหมาย',
          },
        },
      ],
    },
  };
}

// Handler สำหรับ Time Slot Picker
async function handleTimeSlotPostback(event) {
  const params = new URLSearchParams(event.postback.data);
  const action = params.get('action');
  const date = params.get('date');
  const time = params.get('time');

  if (action === 'select_date' || !date) {
    await client.replyMessage(event.replyToken, createDateSelection());
  } else if (date && !time) {
    await client.replyMessage(event.replyToken, createTimeSlotSelection(date));
  } else if (date && time && action !== 'confirm_booking') {
    await client.replyMessage(event.replyToken, createConfirmBooking(date, time));
  } else if (action === 'confirm_booking') {
    // บันทึกการนัดหมาย
    await saveBooking(event.source.userId, date, time);
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: `✅ ยืนยันการนัดหมายเรียบร้อยแล้ว!\n\n📅 ${date}\n⏰ ${time}\n\nเราจะส่งการแจ้งเตือนให้คุณก่อนถึงวันนัด`,
    });
  } else if (action === 'cancel_booking') {
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: '❌ ยกเลิกการนัดหมายแล้ว หากต้องการนัดใหม่ พิมพ์ "นัดหมาย"',
    });
  }
}

async function saveBooking(userId, date, time) {
  // บันทึกลงฐานข้อมูล
  console.log(`Saving booking: User ${userId}, Date ${date}, Time ${time}`);
}
```

---

## 11. ตัวอย่างสมบูรณ์: Yes/No Confirmation

```javascript
// confirmationDialogs.js

// Simple Yes/No
function createSimpleConfirmation(question, yesData, noData) {
  return {
    type: 'text',
    text: question,
    quickReply: {
      items: [
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '✅ ใช่',
            data: yesData,
            displayText: 'ใช่',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '❌ ไม่ใช่',
            data: noData,
            displayText: 'ไม่ใช่',
          },
        },
      ],
    },
  };
}

// Confirmation สำหรับการสั่งซื้อ
function createOrderConfirmation(orderDetails) {
  const { items, total, address } = orderDetails;

  let itemList = items.map((item) => `• ${item.name} x${item.qty} = ฿${item.price * item.qty}`).join('\n');

  return {
    type: 'text',
    text: `🛒 สรุปคำสั่งซื้อ\n\n${itemList}\n\n💰 ยอดรวม: ฿${total}\n📍 ส่งที่: ${address}\n\nยืนยันการสั่งซื้อ?`,
    quickReply: {
      items: [
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '✅ ยืนยันสั่งซื้อ',
            data: `action=confirm_order&orderId=${orderDetails.id}`,
            displayText: 'ยืนยันการสั่งซื้อ',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '✏️ แก้ไขรายการ',
            data: `action=edit_order&orderId=${orderDetails.id}`,
            displayText: 'แก้ไขรายการสั่งซื้อ',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '🏠 เปลี่ยนที่อยู่',
            data: `action=change_address&orderId=${orderDetails.id}`,
            displayText: 'เปลี่ยนที่อยู่จัดส่ง',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '❌ ยกเลิก',
            data: `action=cancel_order&orderId=${orderDetails.id}`,
            displayText: 'ยกเลิกคำสั่งซื้อ',
          },
        },
      ],
    },
  };
}

// Confirmation สำหรับการลบข้อมูล
function createDeleteConfirmation(itemName) {
  return {
    type: 'text',
    text: `⚠️ คุณแน่ใจหรือไม่ที่จะลบ "${itemName}"?\n\nการดำเนินการนี้ไม่สามารถย้อนกลับได้`,
    quickReply: {
      items: [
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '🗑️ ลบเลย',
            data: `action=confirm_delete&item=${encodeURIComponent(itemName)}`,
            displayText: `ยืนยันการลบ ${itemName}`,
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '↩️ ยกเลิก',
            data: 'action=cancel_delete',
            displayText: 'ยกเลิก ไม่ลบ',
          },
        },
      ],
    },
  };
}

// Rating/Satisfaction Survey
function createRatingQuickReply(serviceType) {
  return {
    type: 'text',
    text: `⭐ ให้คะแนนความพอใจสำหรับ${serviceType}:`,
    quickReply: {
      items: [
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '⭐⭐⭐⭐⭐ ดีมาก',
            data: `action=rating&score=5&service=${serviceType}`,
            displayText: 'พอใจมากที่สุด 5/5',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '⭐⭐⭐⭐ ดี',
            data: `action=rating&score=4&service=${serviceType}`,
            displayText: 'พอใจ 4/5',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '⭐⭐⭐ ปานกลาง',
            data: `action=rating&score=3&service=${serviceType}`,
            displayText: 'พอใจปานกลาง 3/5',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '⭐⭐ ไม่ค่อยดี',
            data: `action=rating&score=2&service=${serviceType}`,
            displayText: 'ไม่ค่อยพอใจ 2/5',
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '⭐ แย่',
            data: `action=rating&score=1&service=${serviceType}`,
            displayText: 'ไม่พอใจ 1/5',
          },
        },
      ],
    },
  };
}
```

---

## 12. Python Implementation

```python
# quick_reply_bot.py
import os
from flask import Flask, request, abort
from linebot import LineBotApi, WebhookHandler
from linebot.exceptions import InvalidSignatureError
from linebot.models import (
    MessageEvent, TextMessage, PostbackEvent,
    TextSendMessage, QuickReply, QuickReplyButton,
    MessageAction, PostbackAction, URIAction,
    DatetimePickerAction, CameraAction, CameraRollAction,
    LocationAction
)

app = Flask(__name__)

line_bot_api = LineBotApi(os.environ['LINE_CHANNEL_ACCESS_TOKEN'])
handler = WebhookHandler(os.environ['LINE_CHANNEL_SECRET'])


def create_main_menu():
    """สร้างเมนูหลักด้วย Quick Reply"""
    return TextSendMessage(
        text='🎯 ยินดีต้อนรับ! คุณต้องการบริการอะไร?',
        quick_reply=QuickReply(
            items=[
                QuickReplyButton(
                    action=PostbackAction(
                        label='📦 สั่งสินค้า',
                        data='action=order',
                        display_text='ต้องการสั่งสินค้า'
                    )
                ),
                QuickReplyButton(
                    action=PostbackAction(
                        label='📞 ติดต่อเรา',
                        data='action=contact',
                        display_text='ต้องการติดต่อ'
                    )
                ),
                QuickReplyButton(
                    action=PostbackAction(
                        label='❓ FAQ',
                        data='action=faq',
                        display_text='ดูคำถามที่พบบ่อย'
                    )
                ),
                QuickReplyButton(
                    action=PostbackAction(
                        label='🎁 โปรโมชั่น',
                        data='action=promotions',
                        display_text='ดูโปรโมชั่น'
                    )
                ),
            ]
        )
    )


def create_product_categories():
    """สร้างเมนูเลือกหมวดสินค้า"""
    categories = [
        ('👕 เสื้อผ้า', 'fashion'),
        ('📱 มือถือ', 'phones'),
        ('💄 ความงาม', 'beauty'),
        ('🍕 อาหาร', 'food'),
        ('🎮 เกม', 'games'),
    ]

    items = [
        QuickReplyButton(
            action=PostbackAction(
                label=label,
                data=f'category={cat}',
                display_text=f'เลือกหมวด {label}'
            )
        )
        for label, cat in categories
    ]

    # เพิ่มปุ่มกลับ
    items.append(
        QuickReplyButton(
            action=PostbackAction(
                label='← กลับ',
                data='action=main_menu',
                display_text='กลับเมนูหลัก'
            )
        )
    )

    return TextSendMessage(
        text='🛍️ เลือกหมวดหมู่สินค้า:',
        quick_reply=QuickReply(items=items)
    )


def create_time_slots(selected_date):
    """สร้างตัวเลือกช่วงเวลา"""
    time_slots = [
        ('🌅 09:00', '09:00'),
        ('☀️ 10:00', '10:00'),
        ('🌤️ 11:00', '11:00'),
        ('🍽️ 13:00', '13:00'),
        ('🌞 14:00', '14:00'),
        ('🌆 16:00', '16:00'),
    ]

    items = [
        QuickReplyButton(
            action=PostbackAction(
                label=label,
                data=f'date={selected_date}&time={time}',
                display_text=f'เลือกเวลา {time}'
            )
        )
        for label, time in time_slots
    ]

    items.append(
        QuickReplyButton(
            action=PostbackAction(
                label='← เลือกวันใหม่',
                data='action=select_date',
                display_text='กลับเลือกวันใหม่'
            )
        )
    )

    return TextSendMessage(
        text=f'📅 วันที่ {selected_date}\n⏰ เลือกช่วงเวลา:',
        quick_reply=QuickReply(items=items)
    )


def create_yes_no_confirmation(question, yes_data, no_data):
    """สร้างปุ่มยืนยัน Yes/No"""
    return TextSendMessage(
        text=question,
        quick_reply=QuickReply(
            items=[
                QuickReplyButton(
                    action=PostbackAction(
                        label='✅ ใช่',
                        data=yes_data,
                        display_text='ใช่'
                    )
                ),
                QuickReplyButton(
                    action=PostbackAction(
                        label='❌ ไม่ใช่',
                        data=no_data,
                        display_text='ไม่ใช่'
                    )
                ),
            ]
        )
    )


def create_location_sharing():
    """Quick Reply ที่รวม Location + Camera"""
    return TextSendMessage(
        text='📍 กรุณาระบุตำแหน่งหรือส่งภาพสถานที่:',
        quick_reply=QuickReply(
            items=[
                QuickReplyButton(
                    action=LocationAction(label='📍 แชร์ตำแหน่ง')
                ),
                QuickReplyButton(
                    action=CameraAction(label='📸 ถ่ายภาพ')
                ),
                QuickReplyButton(
                    action=CameraRollAction(label='🖼️ เลือกภาพ')
                ),
                QuickReplyButton(
                    action=PostbackAction(
                        label='⌨️ พิมพ์ที่อยู่',
                        data='action=type_address',
                        display_text='จะพิมพ์ที่อยู่เอง'
                    )
                ),
            ]
        )
    )


@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers['X-Line-Signature']
    body = request.get_data(as_text=True)

    try:
        handler.handle(body, signature)
    except InvalidSignatureError:
        abort(400)

    return 'OK'


@handler.add(MessageEvent, message=TextMessage)
def handle_message(event):
    text = event.message.text.lower()

    if any(word in text for word in ['สวัสดี', 'hello', 'hi', 'เริ่ม']):
        line_bot_api.reply_message(event.reply_token, create_main_menu())
    elif 'สินค้า' in text or 'ซื้อ' in text:
        line_bot_api.reply_message(event.reply_token, create_product_categories())
    elif 'นัด' in text or 'จอง' in text:
        from datetime import date
        today = date.today().isoformat()
        line_bot_api.reply_message(event.reply_token, create_time_slots(today))
    else:
        line_bot_api.reply_message(
            event.reply_token,
            TextSendMessage(
                text='ไม่เข้าใจคำสั่ง กรุณาเลือกจากเมนูด้านล่าง:',
                quick_reply=QuickReply(
                    items=[
                        QuickReplyButton(
                            action=MessageAction(
                                label='🏠 เมนูหลัก',
                                text='สวัสดี'
                            )
                        )
                    ]
                )
            )
        )


@handler.add(PostbackEvent)
def handle_postback(event):
    from urllib.parse import parse_qs
    params = parse_qs(event.postback.data)

    action = params.get('action', [None])[0]
    category = params.get('category', [None])[0]
    date = params.get('date', [None])[0]
    time = params.get('time', [None])[0]

    if action == 'main_menu':
        line_bot_api.reply_message(event.reply_token, create_main_menu())

    elif action == 'order' or category:
        line_bot_api.reply_message(event.reply_token, create_product_categories())

    elif date and not time:
        line_bot_api.reply_message(event.reply_token, create_time_slots(date))

    elif date and time:
        line_bot_api.reply_message(
            event.reply_token,
            create_yes_no_confirmation(
                f'📋 ยืนยันนัด {date} เวลา {time}?',
                f'action=confirm&date={date}&time={time}',
                'action=cancel_booking'
            )
        )


if __name__ == '__main__':
    app.run(port=5000, debug=True)
```

---

## 13. Best Practices และข้อควรระวัง

### Best Practices

1. **จำกัดจำนวนปุ่ม**
   - ใช้ 3-6 ปุ่มจะดีที่สุด
   - หลีกเลี่ยงการใช้ครบ 13 ปุ่มพร้อมกัน

2. **ข้อความบนปุ่มสั้นกระชับ**
   ```
   ✅ ดี: "สั่งซื้อ"
   ❌ ไม่ดี: "กดที่นี่เพื่อสั่งซื้อสินค้า"
   ```

3. **ใช้ Emoji เสริมความเข้าใจ**
   ```
   ✅ ดี: "🛒 ตะกร้า"
   ❌ ไม่ดี: "ตะกร้าสินค้า"
   ```

4. **มีตัวเลือก "กลับ" เสมอ**
   ```javascript
   {
     type: 'action',
     action: {
       type: 'postback',
       label: '← กลับ',
       data: 'action=back'
     }
   }
   ```

5. **ใช้ Postback แทน Message เมื่อทำได้**
   - Postback ไม่แสดงในแชท (ลดความรก)
   - ควบคุม Flow ได้ดีกว่า

### Common Pitfalls

1. **Label เกิน 20 ตัวอักษร**
   ```javascript
   // ❌ ผิด - เกิน 20 ตัวอักษร
   label: 'สั่งซื้อสินค้าทั้งหมดในตะกร้า'

   // ✅ ถูก
   label: 'สั่งซื้อสินค้า'
   ```

2. **Items เกิน 13**
   ```javascript
   // ❌ จะเกิด error
   items: [...Array(14).keys()].map(i => ({ type: 'action', ... }))
   ```

3. **imageUrl ไม่ใช่ HTTPS**
   ```javascript
   // ❌ ผิด - ต้องเป็น HTTPS
   imageUrl: 'http://example.com/icon.png'

   // ✅ ถูก
   imageUrl: 'https://example.com/icon.png'
   ```

4. **displayText เกิน 300 ตัวอักษร**
   ```javascript
   // ❌ ผิด
   displayText: 'a'.repeat(301)
   ```

---

## 14. การทดสอบ Quick Reply

```javascript
// test_quick_reply.js
const axios = require('axios');

async function testQuickReply() {
  const headers = {
    'Content-Type': 'application/json',
    Authorization: `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
  };

  const userId = 'U1234567890abcdef';

  const payload = {
    to: userId,
    messages: [
      {
        type: 'text',
        text: 'ทดสอบ Quick Reply',
        quickReply: {
          items: [
            {
              type: 'action',
              action: {
                type: 'message',
                label: 'ตัวเลือก 1',
                text: 'เลือกตัวเลือก 1',
              },
            },
            {
              type: 'action',
              action: {
                type: 'message',
                label: 'ตัวเลือก 2',
                text: 'เลือกตัวเลือก 2',
              },
            },
          ],
        },
      },
    ],
  };

  try {
    const response = await axios.post('https://api.line.me/v2/bot/message/push', payload, { headers });
    console.log('✅ Quick Reply sent:', response.data);
  } catch (error) {
    console.error('❌ Error:', error.response?.data);
  }
}

testQuickReply();
```

---

## สรุป

Quick Reply เป็นเครื่องมือที่มีประสิทธิภาพในการออกแบบ Conversational UI ใน LINE OA โดยช่วยให้:
- ผู้ใช้สามารถโต้ตอบได้รวดเร็ว
- ควบคุม Flow การสนทนาได้ดีขึ้น
- ลดข้อผิดพลาดจากการพิมพ์
- ประสบการณ์ผู้ใช้ดีขึ้น

**Key Points:**
- สูงสุด 13 items ต่อข้อความ
- รองรับ 7 Action types
- Label สูงสุด 20 ตัวอักษร
- imageUrl ต้องเป็น HTTPS
- ปุ่มจะหายไปหลังจากกดหรือพิมพ์
