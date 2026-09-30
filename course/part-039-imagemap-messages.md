# Part 39: Imagemap Messages - การสร้างภาพแบบ Interactive

## บทนำ

Imagemap Message คือรูปแบบข้อความพิเศษใน LINE ที่ช่วยให้สามารถกำหนดพื้นที่บนภาพ (Tap Areas) ที่เมื่อผู้ใช้กดแล้วจะเกิด Action ต่างๆ ทำให้ภาพธรรมดากลายเป็น Interactive Menu ได้อย่างมีประสิทธิภาพ

---

## 1. Imagemap Message Structure

### โครงสร้างพื้นฐาน

```json
{
  "type": "imagemap",
  "baseUrl": "https://example.com/images/imagemap",
  "altText": "This is an imagemap",
  "baseSize": {
    "width": 1040,
    "height": 800
  },
  "actions": [
    {
      "type": "uri",
      "label": "Visit website",
      "linkUri": "https://example.com",
      "area": {
        "x": 0,
        "y": 0,
        "width": 520,
        "height": 800
      }
    },
    {
      "type": "message",
      "label": "Send message",
      "text": "Hello World",
      "area": {
        "x": 520,
        "y": 0,
        "width": 520,
        "height": 800
      }
    }
  ]
}
```

### Field Descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `type` | String | Yes | ต้องเป็น `"imagemap"` |
| `baseUrl` | String | Yes | Base URL ของภาพ |
| `altText` | String | Yes | ข้อความแทนภาพ (สูงสุด 400 chars) |
| `baseSize` | Object | Yes | ขนาดพื้นฐาน |
| `baseSize.width` | Integer | Yes | ต้องเป็น 1040 เสมอ |
| `baseSize.height` | Integer | Yes | ความสูงของภาพ |
| `actions` | Array | Yes | รายการ Action บนภาพ |
| `video` | Object | No | วิดีโอบนภาพ (Optional) |

---

## 2. ข้อกำหนดของภาพ

### Image Requirements

```
Imagemap Image Requirements:
─────────────────────────────────
Width:  1040 pixels (fixed, required)
Height: ไม่จำกัด แต่แนะนำสัดส่วน 16:9 หรือ 4:3
Format: JPEG หรือ PNG
Size:   ไม่เกิน 1MB
URL:    HTTPS เท่านั้น
─────────────────────────────────
```

### Multi-Resolution Support

LINE จะโหลดภาพตามขนาดหน้าจอ โดย URL จะต้องรองรับหลาย resolution:

```
baseUrl: "https://example.com/images/imagemap"

LINE จะโหลด:
├── .../imagemap/1040    (Full resolution: 1040px)
├── .../imagemap/700     (Medium: 700px)
├── .../imagemap/460     (Small: 460px)
└── .../imagemap/300     (Thumbnail: 300px)
```

### การ Setup Server ให้รองรับ Multi-Resolution

```javascript
// imageServer.js
const express = require('express');
const sharp = require('sharp');
const path = require('path');
const fs = require('fs');

const app = express();

// Route สำหรับ Imagemap
app.get('/images/imagemap/:size', async (req, res) => {
  const { size } = req.params;
  const allowedSizes = [1040, 700, 460, 300];
  const targetSize = parseInt(size);

  if (!allowedSizes.includes(targetSize)) {
    return res.status(404).send('Not found');
  }

  const originalImagePath = path.join(__dirname, 'assets', 'imagemap-original.png');

  try {
    const resizedImage = await sharp(originalImagePath)
      .resize(targetSize)
      .png()
      .toBuffer();

    res.setHeader('Content-Type', 'image/png');
    res.setHeader('Cache-Control', 'public, max-age=86400'); // Cache 1 วัน
    res.send(resizedImage);
  } catch (error) {
    console.error('Error resizing image:', error);
    res.status(500).send('Error processing image');
  }
});

app.listen(3000, () => console.log('Image server running on port 3000'));
```

### Pre-generate ภาพทุกขนาด

```javascript
// generateImagemapImages.js
const sharp = require('sharp');
const path = require('path');
const fs = require('fs');

const SIZES = [1040, 700, 460, 300];

async function generateAllSizes(inputPath, outputDir, baseName) {
  // สร้าง directory ถ้ายังไม่มี
  if (!fs.existsSync(outputDir)) {
    fs.mkdirSync(outputDir, { recursive: true });
  }

  for (const size of SIZES) {
    const outputPath = path.join(outputDir, `${baseName}-${size}.jpg`);

    await sharp(inputPath)
      .resize(size)
      .jpeg({ quality: 85 })
      .toFile(outputPath);

    console.log(`✅ Generated ${size}px: ${outputPath}`);
  }
}

// ใช้งาน
generateAllSizes(
  './assets/menu-original.jpg',
  './public/images/menu',
  'menu'
);
```

---

## 3. Tap Areas (x, y, width, height)

### การกำหนดพื้นที่

```
Coordinate System:
┌──────────────────────────────────┐
│ (0,0)              (1040,0)      │ ← y=0 (top)
│  ┌────────┐                     │
│  │  Area  │ (x:0, y:0,          │
│  │        │  w:300, h:200)      │
│  └────────┘                     │
│ (0,200)            (1040,200)   │
│                                 │
│                    ┌────────┐   │
│                    │  Area  │   │ (x:740, y:200)
│                    │        │   │
│                    └────────┘   │
│                    (1040,400)   │
└──────────────────────────────────┘
```

### ตัวอย่าง Layout 2x2 Grid

```javascript
// กำหนด Grid 2x2
const baseWidth = 1040;
const baseHeight = 780;
const halfWidth = baseWidth / 2;  // 520
const halfHeight = baseHeight / 2; // 390

const areas = [
  // Top-Left
  {
    x: 0,
    y: 0,
    width: halfWidth,
    height: halfHeight,
  },
  // Top-Right
  {
    x: halfWidth,
    y: 0,
    width: halfWidth,
    height: halfHeight,
  },
  // Bottom-Left
  {
    x: 0,
    y: halfHeight,
    width: halfWidth,
    height: halfHeight,
  },
  // Bottom-Right
  {
    x: halfWidth,
    y: halfHeight,
    width: halfWidth,
    height: halfHeight,
  },
];
```

### ตัวอย่าง Layout ที่ซับซ้อน

```
Custom Layout (1040 x 800):
┌─────────────────────────────────────┐
│                                     │ ← ส่วนหัว (full width, h:200)
│         HEADER AREA                 │
├──────────────┬──────────────────────┤
│              │                      │
│   LEFT (520) │   RIGHT (520)        │ ← ส่วนกลาง (h:400)
│              │                      │
├──────────────┴──────────────────────┤
│   BOTTOM LEFT  │  BOTTOM RIGHT      │ ← ส่วนล่าง (h:200)
└────────────────┴────────────────────┘

// Corresponding JSON
areas: [
  { area: {x:0, y:0, width:1040, height:200}, action: {...} },    // Header
  { area: {x:0, y:200, width:520, height:400}, action: {...} },   // Left
  { area: {x:520, y:200, width:520, height:400}, action: {...} }, // Right
  { area: {x:0, y:600, width:520, height:200}, action: {...} },   // Bottom-Left
  { area: {x:520, y:600, width:520, height:200}, action: {...} }, // Bottom-Right
]
```

---

## 4. URI Actions

```json
{
  "type": "uri",
  "label": "Visit our website",
  "linkUri": "https://example.com/products",
  "area": {
    "x": 0,
    "y": 0,
    "width": 520,
    "height": 400
  }
}
```

### URI Action Fields

| Field | Type | Required | Max Length | Description |
|-------|------|----------|------------|-------------|
| `type` | String | Yes | - | `"uri"` |
| `label` | String | Yes | 50 chars | Label สำหรับ accessibility |
| `linkUri` | String | Yes | 1000 chars | URL ที่จะเปิด |
| `area` | Object | Yes | - | พื้นที่ที่กด |

### URI ที่รองรับ

```
https://     - External URL
http://      - External URL (บางอุปกรณ์อาจไม่รองรับ)
line://      - LINE Internal URI
tel:         - เบอร์โทรศัพท์
mailto:      - อีเมล
```

### Line URI Actions

```json
// เปิด Line App
{ "linkUri": "line://app/1645278921-kWRPP32q" }

// เปิดโปรไฟล์
{ "linkUri": "line://nv/profileOther/U1234567890" }

// โทรออก
{ "linkUri": "tel:+66812345678" }

// ส่งอีเมล
{ "linkUri": "mailto:contact@example.com" }
```

---

## 5. Message Actions

```json
{
  "type": "message",
  "label": "Order Coffee",
  "text": "สั่งกาแฟ",
  "area": {
    "x": 0,
    "y": 0,
    "width": 520,
    "height": 400
  }
}
```

### Message Action Fields

| Field | Type | Required | Max Length | Description |
|-------|------|----------|------------|-------------|
| `type` | String | Yes | - | `"message"` |
| `label` | String | Yes | 50 chars | Label สำหรับ accessibility |
| `text` | String | Yes | 300 chars | ข้อความที่ส่ง |
| `area` | Object | Yes | - | พื้นที่ที่กด |

---

## 6. Video ใน Imagemap

สามารถเพิ่มวิดีโอบน Imagemap ได้:

```json
{
  "type": "imagemap",
  "baseUrl": "https://example.com/images/video-thumbnail",
  "altText": "Video Imagemap",
  "baseSize": {
    "width": 1040,
    "height": 585
  },
  "video": {
    "originalContentUrl": "https://example.com/videos/intro.mp4",
    "previewImageUrl": "https://example.com/images/video-preview.jpg",
    "area": {
      "x": 0,
      "y": 0,
      "width": 1040,
      "height": 585
    },
    "externalLink": {
      "linkUri": "https://example.com/full-video",
      "label": "ดูวิดีโอเต็ม"
    }
  },
  "actions": [
    {
      "type": "uri",
      "label": "Visit website",
      "linkUri": "https://example.com",
      "area": {
        "x": 0,
        "y": 0,
        "width": 1040,
        "height": 585
      }
    }
  ]
}
```

### Video Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `originalContentUrl` | String | Yes | URL ของวิดีโอ (HTTPS) |
| `previewImageUrl` | String | Yes | Thumbnail ของวิดีโอ |
| `area` | Object | Yes | พื้นที่แสดงวิดีโอ |
| `externalLink` | Object | No | ลิงก์เพิ่มเติม |

### ข้อกำหนดวิดีโอ

```
Video Requirements:
─────────────────────
Format:  MP4
Codec:   H.264
Size:    ไม่เกิน 200MB
Length:  ไม่เกิน 1 นาที
URL:     HTTPS เท่านั้น
─────────────────────
```

---

## 7. Use Case 1: Interactive Menu Image

### ตัวอย่างร้านอาหาร

```javascript
// restaurantMenu.js
const line = require('@line/bot-sdk');

const client = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
});

function createRestaurantMenuImagemap() {
  // ภาพเมนูร้านอาหาร ขนาด 1040 x 780
  // แบ่งเป็น 6 ช่อง (3x2 grid)
  const cellW = Math.floor(1040 / 3); // ~347
  const cellH = Math.floor(780 / 2);  // 390

  const menuItems = [
    {
      name: 'กาแฟ',
      message: 'สั่งกาแฟ',
      col: 0, row: 0,
    },
    {
      name: 'ชา',
      message: 'สั่งชา',
      col: 1, row: 0,
    },
    {
      name: 'น้ำผลไม้',
      message: 'สั่งน้ำผลไม้',
      col: 2, row: 0,
    },
    {
      name: 'เค้ก',
      message: 'สั่งเค้ก',
      col: 0, row: 1,
    },
    {
      name: 'แซนวิช',
      message: 'สั่งแซนวิช',
      col: 1, row: 1,
    },
    {
      name: 'ดูเมนูทั้งหมด',
      uri: 'https://restaurant.example.com/menu',
      col: 2, row: 1,
    },
  ];

  const actions = menuItems.map(item => {
    const x = item.col * cellW;
    const y = item.row * cellH;

    const action = item.uri
      ? {
          type: 'uri',
          label: item.name,
          linkUri: item.uri,
          area: { x, y, width: cellW, height: cellH },
        }
      : {
          type: 'message',
          label: item.name,
          text: item.message,
          area: { x, y, width: cellW, height: cellH },
        };

    return action;
  });

  return {
    type: 'imagemap',
    baseUrl: 'https://restaurant.example.com/images/menu-board',
    altText: 'เมนูร้านอาหาร - กดเลือกสินค้า',
    baseSize: {
      width: 1040,
      height: 780,
    },
    actions,
  };
}

// Handler
async function handleMenuRequest(event) {
  await client.replyMessage(event.replyToken, createRestaurantMenuImagemap());
}

module.exports = { createRestaurantMenuImagemap, handleMenuRequest };
```

---

## 8. Use Case 2: Product Catalog Image

```javascript
// productCatalog.js

function createProductCatalogImagemap(products) {
  // ภาพ Catalog สินค้า ขนาด 1040 x 1040
  // จัดเป็น Grid 3x3 (หรือ 2x2 ขึ้นอยู่กับจำนวนสินค้า)
  
  const gridCols = products.length <= 4 ? 2 : 3;
  const gridRows = Math.ceil(products.length / gridCols);
  
  const cellW = Math.floor(1040 / gridCols);
  const cellH = Math.floor(1040 / gridRows);
  
  const actions = products.map((product, index) => {
    const col = index % gridCols;
    const row = Math.floor(index / gridCols);
    
    return {
      type: 'postback',
      label: product.name,
      data: `action=view_product&id=${product.id}`,
      area: {
        x: col * cellW,
        y: row * cellH,
        width: cellW,
        height: cellH,
      },
    };
  });

  return {
    type: 'imagemap',
    baseUrl: `https://shop.example.com/catalog/page-1`,
    altText: 'แคตาล็อกสินค้า',
    baseSize: {
      width: 1040,
      height: 1040,
    },
    actions,
  };
}

// ตัวอย่างการใช้งาน
const sampleProducts = [
  { id: 'P001', name: 'เสื้อยืด' },
  { id: 'P002', name: 'กางเกงยีนส์' },
  { id: 'P003', name: 'รองเท้า' },
  { id: 'P004', name: 'กระเป๋า' },
  { id: 'P005', name: 'หมวก' },
  { id: 'P006', name: 'แว่นตา' },
];

const catalogMessage = createProductCatalogImagemap(sampleProducts);

// Webhook Handler สำหรับ Product Catalog
async function handleProductCatalogPostback(event) {
  const data = new URLSearchParams(event.postback.data);
  const action = data.get('action');
  const productId = data.get('id');

  if (action === 'view_product') {
    // ดึงข้อมูลสินค้า
    const product = await getProductById(productId);
    
    if (!product) {
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: '❌ ไม่พบสินค้า',
      });
      return;
    }

    // แสดงรายละเอียดสินค้าด้วย Flex Message
    await client.replyMessage(event.replyToken, {
      type: 'flex',
      altText: product.name,
      contents: {
        type: 'bubble',
        hero: {
          type: 'image',
          url: product.imageUrl,
          size: 'full',
          aspectRatio: '4:3',
          action: {
            type: 'uri',
            uri: product.productUrl,
          },
        },
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            { type: 'text', text: product.name, weight: 'bold', size: 'xl' },
            { type: 'text', text: `฿${product.price}`, size: 'lg', color: '#E74C3C', margin: 'sm' },
            { type: 'text', text: product.description, size: 'sm', color: '#666', wrap: true, margin: 'sm' },
          ],
        },
        footer: {
          type: 'box',
          layout: 'horizontal',
          contents: [
            {
              type: 'button',
              action: {
                type: 'postback',
                label: '🛒 เพิ่มในตะกร้า',
                data: `action=add_to_cart&id=${productId}`,
              },
              style: 'primary',
            },
          ],
        },
      },
    });
  }
}

async function getProductById(id) {
  // Placeholder - ดึงจาก Database จริง
  return { id, name: 'Sample Product', price: 299, imageUrl: 'https://example.com/product.jpg', productUrl: 'https://example.com/product', description: 'รายละเอียดสินค้า' };
}
```

---

## 9. Use Case 3: Event Flyer with Links

```javascript
// eventFlyer.js

function createEventFlyerImagemap(eventData) {
  // Event Flyer ขนาด 1040 x 1560
  // Layout:
  // [0,0] → [1040,200]  : Event Title & Date (click to add calendar)
  // [0,200] → [1040,900] : Event Image (click to see details)
  // [0,900] → [520,1200] : Location (click to open maps)
  // [520,900] → [1040,1200] : Register (click to register)
  // [0,1200] → [1040,1560] : Social Share & More Info

  const {
    id,
    title,
    date,
    locationUrl,
    registerUrl,
    detailUrl,
    calendarUrl,
    socialUrl,
  } = eventData;

  return {
    type: 'imagemap',
    baseUrl: `https://events.example.com/flyers/${id}`,
    altText: `${title} - ${date}`,
    baseSize: {
      width: 1040,
      height: 1560,
    },
    actions: [
      // Event Title & Date - Click to Add Calendar
      {
        type: 'uri',
        label: `${title} - เพิ่มใน Calendar`,
        linkUri: calendarUrl || `https://calendar.google.com/calendar/r/eventedit?text=${encodeURIComponent(title)}&dates=${date}`,
        area: { x: 0, y: 0, width: 1040, height: 200 },
      },
      // Event Main Image - Click to See Details
      {
        type: 'uri',
        label: 'รายละเอียดงาน',
        linkUri: detailUrl,
        area: { x: 0, y: 200, width: 1040, height: 700 },
      },
      // Location - Click to Open Maps
      {
        type: 'uri',
        label: 'ดูแผนที่',
        linkUri: locationUrl,
        area: { x: 0, y: 900, width: 520, height: 300 },
      },
      // Register Button
      {
        type: 'uri',
        label: 'ลงทะเบียน',
        linkUri: registerUrl,
        area: { x: 520, y: 900, width: 520, height: 300 },
      },
      // Social Share
      {
        type: 'message',
        label: 'แชร์งาน',
        text: `📢 แชร์งาน: ${title} - ${detailUrl}`,
        area: { x: 0, y: 1200, width: 520, height: 360 },
      },
      // More Info
      {
        type: 'uri',
        label: 'ข้อมูลเพิ่มเติม',
        linkUri: socialUrl || detailUrl,
        area: { x: 520, y: 1200, width: 520, height: 360 },
      },
    ],
  };
}

// ตัวอย่างการใช้
const concertEvent = {
  id: 'concert-2024-01',
  title: 'BIG CONCERT 2024',
  date: '20240315T190000/20240315T220000',
  locationUrl: 'https://maps.google.com/?q=Impact+Arena+Bangkok',
  registerUrl: 'https://tickets.example.com/big-concert-2024',
  detailUrl: 'https://events.example.com/big-concert-2024',
  calendarUrl: null,
  socialUrl: 'https://facebook.com/events/bigconcert2024',
};

const flyerMessage = createEventFlyerImagemap(concertEvent);
```

---

## 10. Use Case 4: Interactive Infographic

```javascript
// interactiveInfographic.js

function createStatisticsInfographic(stats) {
  // Infographic ขนาด 1040 x 1300
  // แสดงสถิติ 5 รายการ พร้อม Tap Areas

  const statItems = stats.map((stat, index) => ({
    ...stat,
    y: 100 + index * 240,
    height: 220,
  }));

  const actions = statItems.map(stat => ({
    type: 'uri',
    label: stat.label,
    linkUri: stat.detailUrl,
    area: {
      x: 0,
      y: stat.y,
      width: 1040,
      height: stat.height,
    },
  }));

  return {
    type: 'imagemap',
    baseUrl: 'https://analytics.example.com/infographic/monthly',
    altText: 'สถิติประจำเดือน - กดเพื่อดูรายละเอียด',
    baseSize: {
      width: 1040,
      height: 1300,
    },
    actions,
  };
}

// Infographic ที่มี Call-to-Action หลายจุด
function createSalesInfographic() {
  return {
    type: 'imagemap',
    baseUrl: 'https://sales.example.com/report/q4-2024',
    altText: 'รายงานยอดขาย Q4 2024',
    baseSize: {
      width: 1040,
      height: 2000,
    },
    actions: [
      // Click Title - Go to Full Report
      {
        type: 'uri',
        label: 'รายงานเต็ม',
        linkUri: 'https://sales.example.com/report/q4-2024/full',
        area: { x: 0, y: 0, width: 1040, height: 200 },
      },
      // Revenue Chart - Go to Revenue Detail
      {
        type: 'uri',
        label: 'รายละเอียดรายได้',
        linkUri: 'https://sales.example.com/report/revenue',
        area: { x: 0, y: 200, width: 1040, height: 500 },
      },
      // Product Performance - Go to Products
      {
        type: 'message',
        label: 'ดูสินค้าขายดี',
        text: 'แสดงสินค้าขายดีไตรมาส 4',
        area: { x: 0, y: 700, width: 520, height: 400 },
      },
      // Regional Sales - Go to Regions
      {
        type: 'uri',
        label: 'ยอดขายตามภูมิภาค',
        linkUri: 'https://sales.example.com/report/regions',
        area: { x: 520, y: 700, width: 520, height: 400 },
      },
      // Customer Growth - Message
      {
        type: 'message',
        label: 'ข้อมูลลูกค้า',
        text: 'ดูข้อมูลการเติบโตของลูกค้า',
        area: { x: 0, y: 1100, width: 1040, height: 400 },
      },
      // Download Report - URI
      {
        type: 'uri',
        label: 'ดาวน์โหลดรายงาน',
        linkUri: 'https://sales.example.com/report/q4-2024/download',
        area: { x: 0, y: 1500, width: 1040, height: 500 },
      },
    ],
  };
}
```

---

## 11. การสร้างภาพ Imagemap ด้วย Canvas

```javascript
// createImagemapCanvas.js
const { createCanvas, loadImage, registerFont } = require('canvas');
const fs = require('fs');
const path = require('path');

// ลงทะเบียน Font ภาษาไทย
registerFont('./fonts/NotoSansThai-Bold.ttf', { family: 'Noto Sans Thai', weight: 'bold' });
registerFont('./fonts/NotoSansThai-Regular.ttf', { family: 'Noto Sans Thai' });

async function createMenuImagemap(menuItems, outputDir) {
  const WIDTH = 1040;
  const COLS = 3;
  const ROWS = Math.ceil(menuItems.length / COLS);
  const CELL_W = Math.floor(WIDTH / COLS);
  const CELL_H = 300;
  const HEIGHT = CELL_H * ROWS;

  const canvas = createCanvas(WIDTH, HEIGHT);
  const ctx = canvas.getContext('2d');

  // พื้นหลัง
  ctx.fillStyle = '#F8F9FA';
  ctx.fillRect(0, 0, WIDTH, HEIGHT);

  // วาดแต่ละ Cell
  for (let i = 0; i < menuItems.length; i++) {
    const item = menuItems[i];
    const col = i % COLS;
    const row = Math.floor(i / COLS);
    const x = col * CELL_W;
    const y = row * CELL_H;

    // วาดพื้นหลัง Cell
    const gradient = ctx.createLinearGradient(x, y, x + CELL_W, y + CELL_H);
    gradient.addColorStop(0, item.color || '#667EEA');
    gradient.addColorStop(1, item.colorEnd || '#764BA2');
    ctx.fillStyle = gradient;
    ctx.fillRect(x + 5, y + 5, CELL_W - 10, CELL_H - 10);

    // โหลดและวาดภาพสินค้า
    if (item.imageUrl) {
      try {
        const img = await loadImage(item.imageUrl);
        const imgSize = 160;
        ctx.save();
        ctx.beginPath();
        ctx.arc(x + CELL_W / 2, y + 100, imgSize / 2, 0, Math.PI * 2);
        ctx.clip();
        ctx.drawImage(img, x + (CELL_W - imgSize) / 2, y + 20, imgSize, imgSize);
        ctx.restore();
      } catch (err) {
        // วาด placeholder
        ctx.fillStyle = 'rgba(255,255,255,0.3)';
        ctx.fillRect(x + (CELL_W - 100) / 2, y + 30, 100, 100);
      }
    }

    // วาดข้อความชื่อสินค้า
    ctx.fillStyle = '#FFFFFF';
    ctx.font = 'bold 36px "Noto Sans Thai"';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(item.name, x + CELL_W / 2, y + CELL_H - 90);

    // วาดราคา
    if (item.price) {
      ctx.font = '28px "Noto Sans Thai"';
      ctx.fillStyle = '#FFE66D';
      ctx.fillText(`฿${item.price}`, x + CELL_W / 2, y + CELL_H - 50);
    }

    // วาดกรอบ Cell
    ctx.strokeStyle = 'rgba(255,255,255,0.3)';
    ctx.lineWidth = 2;
    ctx.strokeRect(x + 5, y + 5, CELL_W - 10, CELL_H - 10);
  }

  // สร้าง Output Directory ถ้ายังไม่มี
  if (!fs.existsSync(outputDir)) {
    fs.mkdirSync(outputDir, { recursive: true });
  }

  // บันทึกภาพทุกขนาด
  const sizes = [1040, 700, 460, 300];
  for (const size of sizes) {
    const resizedCanvas = createCanvas(size, Math.round(HEIGHT * size / WIDTH));
    const resizedCtx = resizedCanvas.getContext('2d');
    resizedCtx.drawImage(canvas, 0, 0, size, Math.round(HEIGHT * size / WIDTH));

    const outputPath = path.join(outputDir, `${size}`);
    const buffer = resizedCanvas.toBuffer('image/jpeg', { quality: 0.85 });
    fs.writeFileSync(outputPath, buffer);
    console.log(`✅ Saved: ${outputPath}`);
  }

  console.log('✅ All imagemap sizes generated!');
  return { width: WIDTH, height: HEIGHT };
}

// ตัวอย่างการใช้
const coffeeMenu = [
  { name: 'เอสเปรสโซ', price: 60, color: '#2C3E50', colorEnd: '#3498DB', imageUrl: 'https://example.com/espresso.jpg' },
  { name: 'ลาเต้', price: 75, color: '#E67E22', colorEnd: '#F39C12', imageUrl: 'https://example.com/latte.jpg' },
  { name: 'คาปูชิโน', price: 75, color: '#8E44AD', colorEnd: '#9B59B6', imageUrl: 'https://example.com/cappuccino.jpg' },
  { name: 'มอคค่า', price: 85, color: '#27AE60', colorEnd: '#2ECC71', imageUrl: 'https://example.com/mocha.jpg' },
  { name: 'อเมริกาโน', price: 65, color: '#2980B9', colorEnd: '#3498DB', imageUrl: 'https://example.com/americano.jpg' },
  { name: 'ดูเมนูเพิ่ม', price: null, color: '#C0392B', colorEnd: '#E74C3C', imageUrl: null },
];

createMenuImagemap(coffeeMenu, './public/imagemap/coffee-menu');
```

---

## 12. Complete Example: Interactive Shop Banner

```javascript
// shopBanner.js
const line = require('@line/bot-sdk');

const client = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
});

// สร้าง Imagemap สำหรับ Banner ร้านค้า
function createShopBanner(bannerData) {
  const { bannerId, promotions, categories, socialLinks } = bannerData;

  // Layout:
  // Row 1 (y:0-400): Main Banner / Hero Image - Click to visit
  // Row 2 (y:400-700): 3 Promotion Categories
  // Row 3 (y:700-900): Social Links & Contact

  return {
    type: 'imagemap',
    baseUrl: `https://shop.example.com/banners/${bannerId}`,
    altText: 'ร้านค้าออนไลน์ - คลิกเพื่อดูสินค้าและโปรโมชั่น',
    baseSize: {
      width: 1040,
      height: 900,
    },
    actions: [
      // Main Banner
      {
        type: 'uri',
        label: 'ดูโปรโมชั่นหลัก',
        linkUri: 'https://shop.example.com/main-promotion',
        area: { x: 0, y: 0, width: 1040, height: 400 },
      },
      // Promotion 1
      {
        type: 'postback',
        label: promotions[0]?.name || 'โปรโมชั่น 1',
        data: `action=view_promotion&id=${promotions[0]?.id || 'promo1'}`,
        area: { x: 0, y: 400, width: 347, height: 300 },
      },
      // Promotion 2
      {
        type: 'postback',
        label: promotions[1]?.name || 'โปรโมชั่น 2',
        data: `action=view_promotion&id=${promotions[1]?.id || 'promo2'}`,
        area: { x: 347, y: 400, width: 346, height: 300 },
      },
      // Promotion 3
      {
        type: 'postback',
        label: promotions[2]?.name || 'โปรโมชั่น 3',
        data: `action=view_promotion&id=${promotions[2]?.id || 'promo3'}`,
        area: { x: 693, y: 400, width: 347, height: 300 },
      },
      // Facebook
      {
        type: 'uri',
        label: 'Facebook',
        linkUri: socialLinks?.facebook || 'https://facebook.com',
        area: { x: 0, y: 700, width: 260, height: 200 },
      },
      // Instagram
      {
        type: 'uri',
        label: 'Instagram',
        linkUri: socialLinks?.instagram || 'https://instagram.com',
        area: { x: 260, y: 700, width: 260, height: 200 },
      },
      // Website
      {
        type: 'uri',
        label: 'Website',
        linkUri: socialLinks?.website || 'https://example.com',
        area: { x: 520, y: 700, width: 260, height: 200 },
      },
      // Contact
      {
        type: 'message',
        label: 'ติดต่อเรา',
        text: 'ต้องการติดต่อพนักงาน',
        area: { x: 780, y: 700, width: 260, height: 200 },
      },
    ],
  };
}

// Webhook Handler
const express = require('express');
const app = express();

app.post('/webhook', line.middleware({
  channelSecret: process.env.LINE_CHANNEL_SECRET,
}), async (req, res) => {
  const events = req.body.events;

  await Promise.all(events.map(async (event) => {
    if (event.type === 'message' && event.message.type === 'text') {
      const text = event.message.text.toLowerCase();

      if (text.includes('banner') || text.includes('เมนู') || text.includes('สินค้า')) {
        const banner = createShopBanner({
          bannerId: 'main-2024',
          promotions: [
            { id: 'summer', name: 'Summer Sale' },
            { id: 'new', name: 'สินค้าใหม่' },
            { id: 'hot', name: 'ขายดี' },
          ],
          socialLinks: {
            facebook: 'https://facebook.com/shopexample',
            instagram: 'https://instagram.com/shopexample',
            website: 'https://shop.example.com',
          },
        });

        await client.replyMessage(event.replyToken, banner);
      }
    } else if (event.type === 'postback') {
      const data = new URLSearchParams(event.postback.data);
      const action = data.get('action');

      if (action === 'view_promotion') {
        const promoId = data.get('id');
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: `🎁 กำลังโหลดโปรโมชั่น: ${promoId}`,
        });
      }
    }
  }));

  res.json({ status: 'ok' });
});

app.listen(3000, () => console.log('Server on port 3000'));
```

---

## 13. Common Pitfalls และวิธีแก้

### ปัญหา 1: ภาพโหลดไม่ขึ้น

```javascript
// ❌ ผิด - URL ไม่มี size suffix
baseUrl: "https://example.com/images/menu.jpg"

// ✅ ถูก - URL ไม่มีนามสกุล (LINE จะเพิ่ม /1040, /700 เอง)
baseUrl: "https://example.com/images/menu"

// ✅ ถูก - Server ต้องรองรับ paths เหล่านี้:
// GET /images/menu/1040
// GET /images/menu/700
// GET /images/menu/460
// GET /images/menu/300
```

### ปัญหา 2: Area ออกนอกขอบภาพ

```javascript
// ❌ ผิด - Area กว้างกว่าภาพ
{
  baseSize: { width: 1040, height: 600 },
  actions: [{
    area: { x: 500, y: 0, width: 600, height: 600 } // x+width = 1100 > 1040
  }]
}

// ✅ ถูก - Area อยู่ภายในภาพ
{
  baseSize: { width: 1040, height: 600 },
  actions: [{
    area: { x: 520, y: 0, width: 520, height: 600 } // x+width = 1040 = width
  }]
}
```

### ปัญหา 3: baseSize.width ไม่ใช่ 1040

```javascript
// ❌ ผิด - width ต้องเป็น 1040 เสมอ
baseSize: { width: 800, height: 600 }

// ✅ ถูก
baseSize: { width: 1040, height: 600 }
```

### Validation Function

```javascript
function validateImagemap(imagemapMessage) {
  const errors = [];

  // ตรวจสอบ baseSize
  if (imagemapMessage.baseSize.width !== 1040) {
    errors.push('baseSize.width must be 1040');
  }

  // ตรวจสอบ Areas
  const { width, height } = imagemapMessage.baseSize;
  
  for (let i = 0; i < imagemapMessage.actions.length; i++) {
    const action = imagemapMessage.actions[i];
    const { x, y, width: w, height: h } = action.area;

    if (x < 0 || y < 0) {
      errors.push(`Action ${i}: x,y cannot be negative`);
    }
    if (x + w > width) {
      errors.push(`Action ${i}: area extends beyond image width`);
    }
    if (y + h > height) {
      errors.push(`Action ${i}: area extends beyond image height`);
    }
    if (w <= 0 || h <= 0) {
      errors.push(`Action ${i}: area dimensions must be positive`);
    }
  }

  return errors;
}

// ใช้งาน
const errors = validateImagemap(imagemapMessage);
if (errors.length > 0) {
  console.error('Imagemap validation failed:', errors);
} else {
  console.log('✅ Imagemap is valid');
}
```

---

## สรุป

Imagemap Message เป็นเครื่องมือที่ทรงพลังในการสร้าง Interactive Visual Content ใน LINE โดย:
- ทำให้ภาพธรรมดากลายเป็น Interactive ได้
- รองรับ Area แบบกำหนดเองได้อย่างอิสระ
- รองรับ Action หลายประเภท (URI, Message, Video)
- เหมาะสำหรับ Menu, Catalog, Flyer, Infographic

**Key Points:**
- `baseSize.width` ต้องเป็น 1040 เสมอ
- URL ต้องรองรับหลาย Resolution (/1040, /700, /460, /300)
- Area ต้องอยู่ภายในขนาดภาพ
- ใช้ HTTPS เท่านั้น
- ภาพต้องไม่เกิน 1MB
