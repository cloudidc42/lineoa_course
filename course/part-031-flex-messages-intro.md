# Part 31: Flex Messages เบื้องต้น

## สารบัญ

1. [Flex Message คืออะไร?](#flex-message-คืออะไร)
2. [Flex Message vs Template Messages](#flex-message-vs-template-messages)
3. [Flex Message Simulator](#flex-message-simulator)
4. [โครงสร้างพื้นฐาน](#โครงสร้างพื้นฐาน)
5. [Container Types: Bubble และ Carousel](#container-types)
6. [การส่ง Flex Message ครั้งแรก](#การส่ง-flex-message-ครั้งแรก)
7. [JSON Structure Overview](#json-structure-overview)
8. [Node.js Examples](#nodejs-examples)
9. [Python Examples](#python-examples)
10. [Common Mistakes และ Tips](#common-mistakes-และ-tips)
11. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Flex Message คืออะไร?

Flex Message เป็นรูปแบบข้อความที่ทรงพลังที่สุดใน LINE Messaging API ที่ช่วยให้นักพัฒนาสามารถสร้าง UI แบบกำหนดเองได้อย่างอิสระ โดยใช้ระบบ layout ที่คล้ายกับ CSS Flexbox

### ทำไมต้องใช้ Flex Message?

ก่อนที่จะมี Flex Message นักพัฒนาต้องใช้ Template Messages ซึ่งมีข้อจำกัดหลายอย่าง:

- Layout ตายตัว ปรับแต่งได้น้อย
- สีและ font ไม่สามารถเปลี่ยนได้
- จำนวน action ถูกจำกัด
- ไม่รองรับ layout ที่ซับซ้อน

Flex Message แก้ปัญหาเหล่านี้ด้วยการให้ความยืดหยุ่นในการออกแบบ UI เหมือนกับการสร้าง HTML/CSS โดยใช้ JSON

### ลักษณะเด่นของ Flex Message

```
✅ ออกแบบ Layout ได้อิสระ
✅ กำหนดสีพื้นหลัง, สีตัวอักษร ได้เอง
✅ รองรับรูปภาพ, วิดีโอ, ไอคอน
✅ Layout Horizontal/Vertical/Baseline
✅ รองรับ Action หลายประเภท
✅ แสดงผลสวยงามทั้ง iOS และ Android
✅ สร้าง Carousel ได้สูงสุด 12 cards
✅ รองรับ Animated GIF
```

### ตัวอย่างการใช้งานจริง

**1. Product Card**
```
┌─────────────────────────┐
│  [รูปสินค้า]            │
├─────────────────────────┤
│ ชื่อสินค้า              │
│ ราคา: ฿1,290           │
│ ⭐⭐⭐⭐⭐ (4.9)         │
├─────────────────────────┤
│ [เพิ่มลงตะกร้า] [ดูเพิ่มเติม] │
└─────────────────────────┘
```

**2. Order Receipt**
```
┌─────────────────────────┐
│ 🧾 ใบเสร็จคำสั่งซื้อ    │
├─────────────────────────┤
│ รายการ         ราคา    │
│ ─────────────────────── │
│ กาแฟลาเต้ x1    ฿65   │
│ ขนมปัง x2       ฿80   │
│ ─────────────────────── │
│ รวม             ฿145  │
│ ส่วนลด           -฿15  │
│ สุทธิ            ฿130  │
├─────────────────────────┤
│ [ดูสถานะคำสั่งซื้อ]     │
└─────────────────────────┘
```

**3. Event Invitation**
```
┌─────────────────────────┐
│  [Banner งาน]           │
├─────────────────────────┤
│ 🎉 Tech Conference 2025 │
│ 📅 15 มกราคม 2025      │
│ 📍 BITEC บางนา          │
│ 👥 ลงทะเบียนแล้ว 250 คน │
├─────────────────────────┤
│ [สมัครเข้าร่วม]         │
└─────────────────────────┘
```

---

## 2. Flex Message vs Template Messages

### ตารางเปรียบเทียบโดยละเอียด

| คุณสมบัติ | Template Messages | Flex Messages |
|-----------|-------------------|---------------|
| ความยืดหยุ่น Layout | จำกัด | อิสระ |
| การกำหนดสี | ไม่ได้ | ได้ทุกสี |
| จำนวน Actions | จำกัดตาม template | ได้มากกว่า |
| รูปภาพ | 1 ภาพต่อ card | หลายภาพ |
| ความซับซ้อน JSON | น้อย | สูง |
| Learning Curve | ง่าย | ยากกว่า |
| Simulator | ไม่มี | มี (Flex Message Simulator) |
| รองรับ Video | ไม่ | ได้ |
| รองรับ GIF | ไม่ | ได้ |
| Custom Font Size | ไม่ | ได้ |
| Background Image | ไม่ | ได้ |
| Nested Layout | ไม่ | ได้ |

### เมื่อไหรควรใช้อะไร?

**ใช้ Template Messages เมื่อ:**
- ต้องการความเรียบง่าย
- ไม่ต้องการ custom design
- เวลาในการพัฒนาจำกัด
- เนื้อหาไม่ซับซ้อน

**ใช้ Flex Messages เมื่อ:**
- ต้องการ UI ที่สวยงามและเป็นแบรนด์
- ต้องการ layout ที่ซับซ้อน
- แสดงข้อมูลหลายประเภทในการ์ดเดียว
- ต้องการ interactive UI ที่ดี
- สร้าง product catalog, receipt, dashboard

### ตัวอย่าง Template Message (Buttons Template)

```json
{
  "type": "template",
  "altText": "สวัสดี",
  "template": {
    "type": "buttons",
    "thumbnailImageUrl": "https://example.com/image.jpg",
    "title": "ชื่อสินค้า",
    "text": "รายละเอียดสินค้า",
    "actions": [
      {
        "type": "uri",
        "label": "ดูเพิ่มเติม",
        "uri": "https://example.com"
      },
      {
        "type": "postback",
        "label": "ซื้อเลย",
        "data": "action=buy&id=123"
      }
    ]
  }
}
```

**ข้อจำกัด**: title ยาวสูงสุด 40 ตัวอักษร, text สูงสุด 160 ตัวอักษร, actions สูงสุด 4

### ตัวอย่าง Flex Message แบบเดียวกัน

```json
{
  "type": "flex",
  "altText": "สวัสดี",
  "contents": {
    "type": "bubble",
    "hero": {
      "type": "image",
      "url": "https://example.com/image.jpg",
      "size": "full",
      "aspectRatio": "20:13",
      "aspectMode": "cover"
    },
    "body": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        {
          "type": "text",
          "text": "ชื่อสินค้า",
          "weight": "bold",
          "size": "xl",
          "color": "#1a1a1a"
        },
        {
          "type": "text",
          "text": "รายละเอียดสินค้าที่สามารถเขียนได้ยาวกว่า 160 ตัวอักษร และสามารถเพิ่มข้อมูลได้มากกว่าเดิม",
          "size": "sm",
          "color": "#666666",
          "wrap": true,
          "margin": "md"
        },
        {
          "type": "text",
          "text": "฿1,290",
          "weight": "bold",
          "size": "xxl",
          "color": "#FF5722",
          "margin": "lg"
        }
      ]
    },
    "footer": {
      "type": "box",
      "layout": "horizontal",
      "spacing": "sm",
      "contents": [
        {
          "type": "button",
          "style": "secondary",
          "action": {
            "type": "uri",
            "label": "ดูเพิ่มเติม",
            "uri": "https://example.com"
          }
        },
        {
          "type": "button",
          "style": "primary",
          "action": {
            "type": "postback",
            "label": "ซื้อเลย",
            "data": "action=buy&id=123"
          }
        }
      ]
    }
  }
}
```

**ข้อดี**: กำหนดสี, ขนาด font, ไม่จำกัด text length, สามารถเพิ่ม element ได้เสรี

---

## 3. Flex Message Simulator

Flex Message Simulator เป็นเครื่องมือออนไลน์ที่ LINE จัดให้สำหรับ preview และทดสอบ Flex Message ก่อนที่จะ deploy จริง

### วิธีเข้าใช้งาน

1. ไปที่ https://developers.line.biz/flex-simulator/
2. Login ด้วย LINE Account

### ส่วนประกอบของ Simulator

```
┌──────────────────────────────────────────────────────┐
│  Flex Message Simulator                               │
├────────────────────┬─────────────────────────────────┤
│                    │                                 │
│   JSON Editor      │        Preview Panel            │
│   (ซ้าย)          │        (ขวา)                   │
│                    │                                 │
│  {                 │   ┌─────────────────┐          │
│    "type": "flex", │   │                 │          │
│    "altText": ..., │   │  [Flex Message] │          │
│    "contents": {   │   │    Preview      │          │
│      ...           │   │                 │          │
│    }               │   └─────────────────┘          │
│  }                 │                                 │
│                    │   [iOS] [Android] [Desktop]     │
├────────────────────┴─────────────────────────────────┤
│  [Preview] [Send to LINE] [Share]                   │
└──────────────────────────────────────────────────────┘
```

### ฟีเจอร์หลักของ Simulator

**1. Real-time Preview**
- เห็นผลลัพธ์ทันทีที่แก้ไข JSON
- แสดงผลทั้ง iOS และ Android style
- ตรวจสอบ error ใน JSON ได้

**2. Templates**
- มี template สำเร็จรูปให้เลือก
- เช่น Shopping, Receipt, Review, Date time, Transport

**3. Send to LINE**
- ส่ง Flex Message ไปยัง LINE Account จริงได้
- ต้อง connect กับ LINE Account ก่อน

**4. Share**
- แชร์ Flex Message เป็น URL ได้
- เหมาะสำหรับการ collaborate กับทีม

### การใช้งาน Simulator Step by Step

**ขั้นที่ 1**: เปิด Simulator และ clear JSON เดิมออก

**ขั้นที่ 2**: วาง JSON ของคุณ หรือเลือก Template ที่ต้องการ

**ขั้นที่ 3**: ดู Preview และแก้ไข JSON จนพอใจ

**ขั้นที่ 4**: คลิก "Send to LINE" เพื่อทดสอบใน LINE จริง

**ขั้นที่ 5**: นำ JSON ที่สมบูรณ์ไปใช้ใน code

### Tips การใช้ Simulator

```
💡 Tip 1: ใช้ Ctrl+Z เพื่อ Undo
💡 Tip 2: JSON ต้องถูกต้องก่อนจึงจะ Preview ได้
💡 Tip 3: เปลี่ยน device type เพื่อดู responsive behavior
💡 Tip 4: ใช้ Share link เพื่อ share กับ designer
💡 Tip 5: บันทึก JSON ไว้ใน version control เสมอ
```

---

## 4. โครงสร้างพื้นฐาน

### ภาพรวมโครงสร้าง Flex Message

```
Flex Message
├── type: "flex"
├── altText: "ข้อความ fallback"
└── contents (Container)
    ├── Bubble Container
    │   ├── size
    │   ├── direction  
    │   ├── header (Box)
    │   ├── hero (Image/Box/Video)
    │   ├── body (Box)
    │   ├── footer (Box)
    │   └── styles
    └── Carousel Container
        └── contents[]
            ├── Bubble 1
            ├── Bubble 2
            └── Bubble 3
```

### Message Level Structure

ทุก Flex Message เริ่มต้นด้วยโครงสร้างนี้:

```json
{
  "type": "flex",
  "altText": "ข้อความที่แสดงเมื่อ device ไม่รองรับ Flex Message",
  "contents": {
    // Container object (bubble หรือ carousel)
  }
}
```

| Property | Required | Description |
|----------|----------|-------------|
| type | ✅ | ต้องเป็น "flex" เสมอ |
| altText | ✅ | ข้อความ fallback (สูงสุด 400 ตัวอักษร) |
| contents | ✅ | Container object |

### altText คืออะไรและสำคัญแค่ไหน?

altText จะแสดงในกรณีต่อไปนี้:
- **LINE notification** บนหน้าจอล็อก
- **LINE desktop** เวอร์ชันเก่า
- **Device ที่ไม่รองรับ** Flex Message
- **Push notification preview**

```
❌ altText ที่ไม่ดี:
"Flex Message"

✅ altText ที่ดี:
"สินค้า: Nike Air Max 90 ราคา ฿3,500 - แตะเพื่อดูรายละเอียด"
```

### Component Hierarchy

```
Container (bubble/carousel)
└── Sections (header/hero/body/footer)
    └── Box Component
        ├── Text Component
        ├── Image Component
        ├── Button Component
        ├── Icon Component
        ├── Video Component
        ├── Separator Component
        ├── Filler Component
        └── Box Component (nested)
            └── ... (can nest further)
```

### การอ้างอิง Properties

ทุก Component มี properties หลัก:

```json
{
  "type": "box",        // ประเภทของ component
  "layout": "vertical", // วิธีจัด layout (สำหรับ box)
  "margin": "md",       // margin รอบ component
  "padding": "lg",      // padding ภายใน component  
  "flex": 1,            // proportion ใน flexbox
  "contents": []        // child components
}
```

---

## 5. Container Types

### Bubble Container

Bubble คือ container พื้นฐานที่สุด แสดงเป็นการ์ดเดียว

```json
{
  "type": "flex",
  "altText": "ตัวอย่าง Bubble",
  "contents": {
    "type": "bubble",
    "size": "mega",
    "header": { /* ... */ },
    "hero": { /* ... */ },
    "body": { /* ... */ },
    "footer": { /* ... */ },
    "styles": { /* ... */ }
  }
}
```

**ส่วนประกอบของ Bubble:**

| Section | Required | Description |
|---------|----------|-------------|
| header | ❌ | ส่วนหัว มักใช้แสดง title |
| hero | ❌ | รูปภาพหลักหรือ video |
| body | ❌ | เนื้อหาหลัก |
| footer | ❌ | ส่วนล่าง มักใช้ปุ่ม action |
| styles | ❌ | กำหนด style ของแต่ละ section |

**ขนาด Bubble:**

```
nano  → ขนาดเล็กมาก
micro → ขนาดเล็ก  
kilo  → ขนาดกลาง (default)
mega  → ขนาดใหญ่
giga  → ขนาดใหญ่มาก
```

### Carousel Container

Carousel คือ container ที่รวม Bubble หลายอันเข้าด้วยกัน ผู้ใช้สามารถ scroll ดูได้

```json
{
  "type": "flex",
  "altText": "ตัวอย่าง Carousel",
  "contents": {
    "type": "carousel",
    "contents": [
      {
        "type": "bubble",
        "body": { /* bubble 1 */ }
      },
      {
        "type": "bubble",
        "body": { /* bubble 2 */ }
      },
      {
        "type": "bubble",
        "body": { /* bubble 3 */ }
      }
    ]
  }
}
```

**ข้อจำกัด Carousel:**
- สูงสุด 12 bubbles ต่อ carousel
- ทุก bubble ควรมีขนาดเท่ากัน
- Carousel ไม่รองรับ styles property

---

## 6. การส่ง Flex Message ครั้งแรก

### ข้อกำหนดเบื้องต้น

1. **LINE Developer Account** - ลงทะเบียนที่ https://developers.line.biz
2. **LINE Channel** - สร้าง Messaging API Channel
3. **Channel Access Token** - สำหรับ authenticate API calls
4. **Webhook Server** - สำหรับรับ events (optional สำหรับ Push Message)

### ตัวอย่าง: ส่ง Flex Message อย่างง่าย (cURL)

```bash
curl -X POST https://api.line.me/v2/bot/message/push \
  -H 'Content-Type: application/json' \
  -H 'Authorization: Bearer YOUR_CHANNEL_ACCESS_TOKEN' \
  -d '{
    "to": "USER_ID",
    "messages": [
      {
        "type": "flex",
        "altText": "Hello Flex!",
        "contents": {
          "type": "bubble",
          "body": {
            "type": "box",
            "layout": "vertical",
            "contents": [
              {
                "type": "text",
                "text": "สวัสดี Flex Message!",
                "weight": "bold",
                "size": "xl"
              }
            ]
          }
        }
      }
    ]
  }'
```

### Response ที่คาดหวัง

```json
{}
```

Response เป็น empty object หมายถึง success (HTTP 200)

### Error Responses ที่พบบ่อย

```json
// 400 Bad Request - JSON ไม่ถูกต้อง
{
  "message": "The request body has 1 error(s)",
  "details": [
    {
      "message": "Property 'type' is missing",
      "property": "messages[0].contents.body"
    }
  ]
}

// 401 Unauthorized - Token ไม่ถูกต้อง
{
  "message": "Authentication failed due to the following reason: invalid token",
  "details": []
}

// 429 Too Many Requests - Rate limit
{
  "message": "Too Many Requests"
}
```

---

## 7. JSON Structure Overview

### โครงสร้าง JSON แบบสมบูรณ์

```json
{
  "type": "flex",
  "altText": "สินค้าแนะนำ: Nike Air Max",
  "contents": {
    "type": "bubble",
    "size": "mega",
    "direction": "ltr",
    
    "header": {
      "type": "box",
      "layout": "vertical",
      "backgroundColor": "#FF5722",
      "contents": [
        {
          "type": "text",
          "text": "🔥 สินค้าแนะนำ",
          "color": "#FFFFFF",
          "weight": "bold",
          "size": "sm"
        }
      ]
    },
    
    "hero": {
      "type": "image",
      "url": "https://example.com/nike-air-max.jpg",
      "size": "full",
      "aspectRatio": "20:13",
      "aspectMode": "cover",
      "action": {
        "type": "uri",
        "uri": "https://example.com/product/123"
      }
    },
    
    "body": {
      "type": "box",
      "layout": "vertical",
      "spacing": "md",
      "contents": [
        {
          "type": "text",
          "text": "Nike Air Max 90",
          "weight": "bold",
          "size": "xl",
          "color": "#1a1a1a"
        },
        {
          "type": "box",
          "layout": "baseline",
          "spacing": "sm",
          "contents": [
            {
              "type": "icon",
              "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
              "size": "sm"
            },
            {
              "type": "text",
              "text": "4.9",
              "size": "sm",
              "color": "#999999",
              "flex": 0
            },
            {
              "type": "text",
              "text": "(2,341 reviews)",
              "size": "sm",
              "color": "#aaaaaa"
            }
          ]
        },
        {
          "type": "separator",
          "margin": "lg"
        },
        {
          "type": "box",
          "layout": "horizontal",
          "margin": "lg",
          "contents": [
            {
              "type": "text",
              "text": "ราคา",
              "size": "sm",
              "color": "#555555",
              "flex": 0
            },
            {
              "type": "text",
              "text": "฿3,500",
              "size": "xl",
              "weight": "bold",
              "color": "#FF5722",
              "align": "end"
            }
          ]
        }
      ]
    },
    
    "footer": {
      "type": "box",
      "layout": "horizontal",
      "spacing": "sm",
      "contents": [
        {
          "type": "button",
          "style": "secondary",
          "height": "sm",
          "action": {
            "type": "uri",
            "label": "ดูรายละเอียด",
            "uri": "https://example.com/product/123"
          }
        },
        {
          "type": "button",
          "style": "primary",
          "height": "sm",
          "action": {
            "type": "postback",
            "label": "ซื้อเลย",
            "data": "action=buy&productId=123"
          }
        }
      ]
    },
    
    "styles": {
      "header": {
        "backgroundColor": "#FF5722"
      },
      "hero": {
        "separator": false
      },
      "body": {
        "backgroundColor": "#FFFFFF"
      },
      "footer": {
        "separator": true,
        "separatorColor": "#eeeeee"
      }
    }
  }
}
```

### Component Reference Map

```
"type" values ที่ใช้บ่อย:

Container types:
  "bubble"    - single card container
  "carousel"  - multiple cards container

Component types:
  "box"       - layout container
  "text"      - text element
  "image"     - image element  
  "button"    - button element
  "icon"      - small icon
  "video"     - video element
  "separator" - horizontal line
  "filler"    - empty space
  "spacer"    - deprecated, use margin แทน
  "span"      - inline text (ใช้ใน text.contents)

Action types:
  "uri"       - เปิด URL
  "postback"  - ส่ง postback event
  "message"   - ส่ง text message
  "datetimepicker" - เปิด date picker
  "camera"    - เปิดกล้อง
  "cameraRoll" - เปิด gallery
  "location"  - ส่ง location
```

---

## 8. Node.js Examples

### ติดตั้ง LINE Bot SDK

```bash
npm install @line/bot-sdk
```

### ตัวอย่างพื้นฐาน: ส่ง Flex Message

```javascript
// app.js
const line = require('@line/bot-sdk');

const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET,
};

const client = new line.Client(config);

// สร้าง Flex Message JSON
function createSimpleFlexMessage() {
  return {
    type: 'flex',
    altText: 'สินค้าแนะนำ',
    contents: {
      type: 'bubble',
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: 'สวัสดี!',
            weight: 'bold',
            size: 'xl',
            color: '#1a1a1a',
          },
          {
            type: 'text',
            text: 'นี่คือ Flex Message แรกของฉัน',
            size: 'md',
            color: '#666666',
            wrap: true,
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
            style: 'primary',
            action: {
              type: 'uri',
              label: 'เรียนรู้เพิ่มเติม',
              uri: 'https://developers.line.biz/en/docs/messaging-api/flex-message-elements/',
            },
          },
        ],
      },
    },
  };
}

// ส่ง Flex Message
async function sendFlexMessage(userId) {
  try {
    const flexMessage = createSimpleFlexMessage();
    
    await client.pushMessage(userId, flexMessage);
    console.log('Flex Message ส่งสำเร็จ!');
  } catch (error) {
    console.error('เกิดข้อผิดพลาด:', error);
    throw error;
  }
}

// ตัวอย่างการใช้งาน
sendFlexMessage('Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx');
```

### ตัวอย่าง: สร้าง Product Card

```javascript
// product-card.js

/**
 * สร้าง Flex Message สำหรับแสดงสินค้า
 * @param {Object} product - ข้อมูลสินค้า
 * @returns {Object} Flex Message object
 */
function createProductCard(product) {
  const {
    name,
    description,
    price,
    originalPrice,
    imageUrl,
    rating,
    reviewCount,
    productUrl,
    productId,
  } = product;

  // คำนวณส่วนลด
  const discount = originalPrice
    ? Math.round(((originalPrice - price) / originalPrice) * 100)
    : 0;

  return {
    type: 'flex',
    altText: `สินค้า: ${name} - ราคา ฿${price.toLocaleString()}`,
    contents: {
      type: 'bubble',
      size: 'mega',
      hero: {
        type: 'image',
        url: imageUrl,
        size: 'full',
        aspectRatio: '20:13',
        aspectMode: 'cover',
        action: {
          type: 'uri',
          uri: productUrl,
        },
      },
      body: {
        type: 'box',
        layout: 'vertical',
        spacing: 'sm',
        contents: [
          // ชื่อสินค้า
          {
            type: 'text',
            text: name,
            weight: 'bold',
            size: 'xl',
            color: '#1a1a1a',
            maxLines: 2,
            wrap: true,
          },
          // Rating
          {
            type: 'box',
            layout: 'baseline',
            spacing: 'sm',
            margin: 'sm',
            contents: [
              {
                type: 'icon',
                url: 'https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png',
                size: 'sm',
              },
              {
                type: 'text',
                text: `${rating}`,
                size: 'sm',
                color: '#FF9800',
                flex: 0,
                weight: 'bold',
              },
              {
                type: 'text',
                text: `(${reviewCount.toLocaleString()} รีวิว)`,
                size: 'sm',
                color: '#aaaaaa',
                margin: 'sm',
              },
            ],
          },
          // Description
          {
            type: 'text',
            text: description,
            size: 'sm',
            color: '#666666',
            wrap: true,
            margin: 'sm',
            maxLines: 3,
          },
          // Separator
          {
            type: 'separator',
            margin: 'md',
          },
          // Price section
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'md',
            contents: [
              {
                type: 'box',
                layout: 'vertical',
                flex: 2,
                contents: [
                  {
                    type: 'text',
                    text: `฿${price.toLocaleString()}`,
                    weight: 'bold',
                    size: 'xl',
                    color: '#FF5722',
                  },
                  ...(originalPrice
                    ? [
                        {
                          type: 'text',
                          text: `฿${originalPrice.toLocaleString()}`,
                          size: 'sm',
                          color: '#aaaaaa',
                          decoration: 'line-through',
                        },
                      ]
                    : []),
                ],
              },
              ...(discount > 0
                ? [
                    {
                      type: 'box',
                      layout: 'vertical',
                      flex: 0,
                      contents: [
                        {
                          type: 'text',
                          text: `-${discount}%`,
                          size: 'sm',
                          color: '#FFFFFF',
                          weight: 'bold',
                          align: 'center',
                        },
                      ],
                      backgroundColor: '#F44336',
                      cornerRadius: '4px',
                      padding: '4px 8px',
                    },
                  ]
                : []),
            ],
          },
        ],
      },
      footer: {
        type: 'box',
        layout: 'horizontal',
        spacing: 'sm',
        contents: [
          {
            type: 'button',
            style: 'secondary',
            height: 'sm',
            flex: 1,
            action: {
              type: 'uri',
              label: 'ดูรายละเอียด',
              uri: productUrl,
            },
          },
          {
            type: 'button',
            style: 'primary',
            height: 'sm',
            flex: 1,
            color: '#FF5722',
            action: {
              type: 'postback',
              label: 'เพิ่มลงตะกร้า',
              data: `action=addToCart&productId=${productId}`,
              displayText: `เพิ่ม ${name} ลงตะกร้าแล้ว`,
            },
          },
        ],
      },
    },
  };
}

// ตัวอย่างการใช้งาน
const product = {
  name: 'Nike Air Max 90 Essential',
  description: 'รองเท้าวิ่งรุ่นยอดนิยม ดีไซน์คลาสสิก สวมใส่สบาย เหมาะกับทุกโอกาส',
  price: 3500,
  originalPrice: 4500,
  imageUrl: 'https://example.com/nike-air-max.jpg',
  rating: 4.9,
  reviewCount: 2341,
  productUrl: 'https://example.com/products/nike-air-max-90',
  productId: 'NIKE-AM90-001',
};

const flexMessage = createProductCard(product);
console.log(JSON.stringify(flexMessage, null, 2));
```

### ตัวอย่าง: Webhook Handler

```javascript
// webhook.js
const express = require('express');
const line = require('@line/bot-sdk');

const app = express();

const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET,
};

const client = new line.Client(config);

// Webhook endpoint
app.post('/webhook', line.middleware(config), async (req, res) => {
  try {
    const events = req.body.events;
    
    await Promise.all(events.map(handleEvent));
    
    res.json({ success: true });
  } catch (error) {
    console.error(error);
    res.status(500).json({ error: error.message });
  }
});

async function handleEvent(event) {
  // รองรับเฉพาะ message event ประเภท text
  if (event.type !== 'message' || event.message.type !== 'text') {
    return;
  }

  const userMessage = event.message.text.toLowerCase();
  
  if (userMessage === 'สินค้า' || userMessage === 'product') {
    // ส่ง product card
    const flexMessage = createProductCard({
      name: 'สินค้าตัวอย่าง',
      description: 'รายละเอียดสินค้าตัวอย่าง',
      price: 299,
      originalPrice: 399,
      imageUrl: 'https://via.placeholder.com/400x300',
      rating: 4.5,
      reviewCount: 100,
      productUrl: 'https://example.com',
      productId: 'P001',
    });
    
    return client.replyMessage(event.replyToken, flexMessage);
  }
  
  // Default reply
  return client.replyMessage(event.replyToken, {
    type: 'text',
    text: 'พิมพ์ "สินค้า" เพื่อดูตัวอย่าง Flex Message',
  });
}

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

## 9. Python Examples

### ติดตั้ง LINE Bot SDK Python

```bash
pip install line-bot-sdk
```

### ตัวอย่างพื้นฐาน: ส่ง Flex Message

```python
# send_flex.py
import os
from linebot import LineBotApi
from linebot.models import FlexSendMessage, BubbleContainer, BoxComponent, TextComponent

# Initialize
line_bot_api = LineBotApi(os.environ.get('LINE_CHANNEL_ACCESS_TOKEN'))

def create_simple_flex():
    """สร้าง Flex Message อย่างง่าย"""
    return FlexSendMessage(
        alt_text='สวัสดี Flex Message!',
        contents=BubbleContainer(
            body=BoxComponent(
                layout='vertical',
                contents=[
                    TextComponent(
                        text='สวัสดี!',
                        weight='bold',
                        size='xl',
                        color='#1a1a1a'
                    ),
                    TextComponent(
                        text='นี่คือ Flex Message แรกของฉัน',
                        size='md',
                        color='#666666',
                        wrap=True,
                        margin='md'
                    )
                ]
            )
        )
    )

# ส่ง Flex Message
def send_flex_message(user_id: str):
    try:
        flex_message = create_simple_flex()
        line_bot_api.push_message(user_id, flex_message)
        print('ส่ง Flex Message สำเร็จ!')
    except Exception as e:
        print(f'เกิดข้อผิดพลาด: {e}')
        raise

# ใช้งาน
send_flex_message('Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx')
```

### ตัวอย่าง: ใช้ Raw JSON (แนะนำสำหรับ Flex Message ที่ซับซ้อน)

```python
# send_flex_raw.py
import os
import json
from linebot import LineBotApi
from linebot.models import FlexSendMessage, RawBubbleContainer

line_bot_api = LineBotApi(os.environ.get('LINE_CHANNEL_ACCESS_TOKEN'))

def create_product_card(product: dict) -> FlexSendMessage:
    """
    สร้าง Flex Message สำหรับแสดงสินค้า
    
    Args:
        product: dict ที่มี keys: name, description, price, original_price, 
                 image_url, rating, review_count, product_url, product_id
    
    Returns:
        FlexSendMessage object
    """
    discount = 0
    if product.get('original_price'):
        discount = round(
            ((product['original_price'] - product['price']) / product['original_price']) * 100
        )
    
    # สร้าง price section contents
    price_contents = [
        {
            "type": "text",
            "text": f"฿{product['price']:,}",
            "weight": "bold",
            "size": "xl",
            "color": "#FF5722"
        }
    ]
    
    if product.get('original_price'):
        price_contents.append({
            "type": "text",
            "text": f"฿{product['original_price']:,}",
            "size": "sm",
            "color": "#aaaaaa",
            "decoration": "line-through"
        })
    
    price_box_contents = [
        {
            "type": "box",
            "layout": "vertical",
            "flex": 2,
            "contents": price_contents
        }
    ]
    
    if discount > 0:
        price_box_contents.append({
            "type": "box",
            "layout": "vertical",
            "flex": 0,
            "contents": [
                {
                    "type": "text",
                    "text": f"-{discount}%",
                    "size": "sm",
                    "color": "#FFFFFF",
                    "weight": "bold",
                    "align": "center"
                }
            ],
            "backgroundColor": "#F44336",
            "cornerRadius": "4px",
            "padding": "4px 8px"
        })
    
    bubble = {
        "type": "bubble",
        "size": "mega",
        "hero": {
            "type": "image",
            "url": product['image_url'],
            "size": "full",
            "aspectRatio": "20:13",
            "aspectMode": "cover",
            "action": {
                "type": "uri",
                "uri": product['product_url']
            }
        },
        "body": {
            "type": "box",
            "layout": "vertical",
            "spacing": "sm",
            "contents": [
                {
                    "type": "text",
                    "text": product['name'],
                    "weight": "bold",
                    "size": "xl",
                    "color": "#1a1a1a",
                    "maxLines": 2,
                    "wrap": True
                },
                {
                    "type": "box",
                    "layout": "baseline",
                    "spacing": "sm",
                    "margin": "sm",
                    "contents": [
                        {
                            "type": "icon",
                            "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
                            "size": "sm"
                        },
                        {
                            "type": "text",
                            "text": f"{product['rating']}",
                            "size": "sm",
                            "color": "#FF9800",
                            "flex": 0,
                            "weight": "bold"
                        },
                        {
                            "type": "text",
                            "text": f"({product['review_count']:,} รีวิว)",
                            "size": "sm",
                            "color": "#aaaaaa",
                            "margin": "sm"
                        }
                    ]
                },
                {
                    "type": "text",
                    "text": product['description'],
                    "size": "sm",
                    "color": "#666666",
                    "wrap": True,
                    "margin": "sm",
                    "maxLines": 3
                },
                {
                    "type": "separator",
                    "margin": "md"
                },
                {
                    "type": "box",
                    "layout": "horizontal",
                    "margin": "md",
                    "contents": price_box_contents
                }
            ]
        },
        "footer": {
            "type": "box",
            "layout": "horizontal",
            "spacing": "sm",
            "contents": [
                {
                    "type": "button",
                    "style": "secondary",
                    "height": "sm",
                    "flex": 1,
                    "action": {
                        "type": "uri",
                        "label": "ดูรายละเอียด",
                        "uri": product['product_url']
                    }
                },
                {
                    "type": "button",
                    "style": "primary",
                    "height": "sm",
                    "flex": 1,
                    "color": "#FF5722",
                    "action": {
                        "type": "postback",
                        "label": "เพิ่มลงตะกร้า",
                        "data": f"action=addToCart&productId={product['product_id']}",
                        "displayText": f"เพิ่ม {product['name']} ลงตะกร้าแล้ว"
                    }
                }
            ]
        }
    }
    
    return FlexSendMessage(
        alt_text=f"สินค้า: {product['name']} - ราคา ฿{product['price']:,}",
        contents=RawBubbleContainer(bubble)
    )


# Flask Webhook Handler
from flask import Flask, request, abort
from linebot import WebhookHandler
from linebot.models import MessageEvent, TextMessage

app = Flask(__name__)
handler = WebhookHandler(os.environ.get('LINE_CHANNEL_SECRET'))

@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('X-Line-Signature', '')
    body = request.get_data(as_text=True)
    
    try:
        handler.handle(body, signature)
    except Exception as e:
        print(f'Error: {e}')
        abort(400)
    
    return 'OK'

@handler.add(MessageEvent, message=TextMessage)
def handle_message(event):
    text = event.message.text.lower()
    
    if text in ['สินค้า', 'product']:
        product = {
            'name': 'Nike Air Max 90 Essential',
            'description': 'รองเท้าวิ่งรุ่นยอดนิยม ดีไซน์คลาสสิก',
            'price': 3500,
            'original_price': 4500,
            'image_url': 'https://via.placeholder.com/400x300',
            'rating': 4.9,
            'review_count': 2341,
            'product_url': 'https://example.com/products/nike-am90',
            'product_id': 'NIKE-AM90-001'
        }
        
        flex_message = create_product_card(product)
        line_bot_api.reply_message(event.reply_token, flex_message)
    else:
        line_bot_api.reply_message(
            event.reply_token,
            TextSendMessage(text='พิมพ์ "สินค้า" เพื่อดูตัวอย่าง Flex Message')
        )

if __name__ == '__main__':
    app.run(port=3000, debug=True)
```

---

## 10. Common Mistakes และ Tips

### ข้อผิดพลาดที่พบบ่อย

**1. ลืม altText**
```json
// ❌ ผิด
{
  "type": "flex",
  "contents": { ... }
}

// ✅ ถูก
{
  "type": "flex",
  "altText": "ข้อความอธิบาย",
  "contents": { ... }
}
```

**2. ใส่ contents ผิดระดับ**
```json
// ❌ ผิด - contents อยู่ที่ bubble level
{
  "type": "flex",
  "altText": "...",
  "contents": {
    "type": "bubble",
    "contents": [  // ← ผิด! bubble ไม่มี contents
      { ... }
    ]
  }
}

// ✅ ถูก - contents อยู่ที่ box level
{
  "type": "flex",
  "altText": "...",
  "contents": {
    "type": "bubble",
    "body": {
      "type": "box",
      "layout": "vertical",
      "contents": [  // ← ถูก! box มี contents
        { ... }
      ]
    }
  }
}
```

**3. URL รูปภาพไม่รองรับ HTTPS**
```json
// ❌ ผิด - ต้องใช้ HTTPS เท่านั้น
"url": "http://example.com/image.jpg"

// ✅ ถูก
"url": "https://example.com/image.jpg"
```

**4. Button action ไม่มี label**
```json
// ❌ ผิด
{
  "type": "button",
  "action": {
    "type": "uri",
    "uri": "https://example.com"
  }
}

// ✅ ถูก
{
  "type": "button",
  "action": {
    "type": "uri",
    "label": "เปิดเว็บไซต์",  // ← ต้องมี label
    "uri": "https://example.com"
  }
}
```

**5. ใช้ color code ผิดรูปแบบ**
```json
// ❌ ผิด
"color": "red"
"color": "rgb(255, 0, 0)"

// ✅ ถูก
"color": "#FF0000"
"color": "#FF000080"  // รองรับ alpha
```

### Best Practices

```
✅ เสมอใช้ HTTPS สำหรับรูปภาพ
✅ กำหนด aspectRatio และ aspectMode สำหรับรูปภาพ
✅ ใส่ altText ที่อธิบายเนื้อหาชัดเจน
✅ ทดสอบใน Flex Message Simulator ก่อน deploy
✅ ทดสอบบนทั้ง iOS และ Android
✅ ใช้ wrap: true สำหรับ text ที่อาจยาว
✅ กำหนด maxLines เพื่อป้องกัน overflow
✅ ใช้ separator แบ่งส่วนเนื้อหา
✅ button label ควรกระชับ ไม่เกิน 20 ตัวอักษร
✅ รักษา JSON structure ให้ clean และ readable
```

### Performance Tips

```
⚡ ลดขนาดรูปภาพก่อน upload (สูงสุด 10MB)
⚡ ใช้ CDN สำหรับรูปภาพเพื่อโหลดเร็ว
⚡ Carousel ไม่ควรเกิน 6-8 cards (UX)
⚡ หลีกเลี่ยง nesting box ลึกเกินไป (ช้า render)
⚡ ใช้ aspectMode: "cover" แทน "fit" เพื่อ layout ที่คงที่
```

---

## 11. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Hello Flex (ง่าย)

สร้าง Flex Message แบบ bubble อย่างง่ายที่แสดง:
- ชื่อของคุณ (ขนาด xl, bold)
- ตำแหน่งงาน (ขนาด md, สีเทา)
- ปุ่ม "ติดต่อฉัน" ที่เปิด URL

```json
// เฉลย - โครงสร้างพื้นฐาน
{
  "type": "flex",
  "altText": "นามบัตร",
  "contents": {
    "type": "bubble",
    "body": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        // เติมเนื้อหาที่นี่
      ]
    },
    "footer": {
      // เติมปุ่มที่นี่
    }
  }
}
```

### แบบฝึกหัดที่ 2: Weather Card (ปานกลาง)

สร้าง Flex Message แสดงสภาพอากาศที่มี:
- Icon สภาพอากาศ (รูปภาพ)
- อุณหภูมิ ขนาดใหญ่ (สี)
- ความชื้น, ความเร็วลม
- ชื่อเมือง
- ปุ่ม "ดูพยากรณ์ 7 วัน"

### แบบฝึกหัดที่ 3: Restaurant Menu Card (ยาก)

สร้าง Carousel ที่มี 3 การ์ดเมนูอาหาร แต่ละการ์ดมี:
- รูปอาหาร
- ชื่ออาหาร
- ราคา
- คะแนน
- ปุ่มสั่ง

---

## สรุป

Flex Message เป็น feature ที่ทรงพลังมากใน LINE Messaging API ที่ช่วยให้สร้าง UI ที่สวยงามและ interactive ได้ ใน Part นี้เราได้เรียนรู้:

1. **Flex Message คืออะไร** และแตกต่างจาก Template Messages อย่างไร
2. **Flex Message Simulator** เครื่องมือสำหรับ design และ test
3. **โครงสร้างพื้นฐาน** ของ Flex Message (type, altText, contents)
4. **Container types** ทั้ง Bubble และ Carousel
5. **การส่ง Flex Message** ด้วย cURL, Node.js, และ Python
6. **Common Mistakes** และวิธีแก้ไข

ใน Part ต่อไปเราจะเจาะลึก Bubble Container และ component ต่างๆ อย่างละเอียด

---

## Resources

- [Flex Message Official Docs](https://developers.line.biz/en/docs/messaging-api/flex-message-elements/)
- [Flex Message Simulator](https://developers.line.biz/flex-simulator/)
- [LINE Bot SDK Node.js](https://github.com/line/line-bot-sdk-nodejs)
- [LINE Bot SDK Python](https://github.com/line/line-bot-sdk-python)
- [Flex Message JSON Schema](https://developers.line.biz/en/reference/messaging-api/#flex-message)
