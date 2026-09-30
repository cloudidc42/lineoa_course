# Part 51: Rich Menu API ขั้นสูง

## สารบัญ

1. [Rich Menu API Overview](#rich-menu-api-overview)
2. [โครงสร้าง Rich Menu Object](#โครงสร้าง-rich-menu-object)
3. [Area Object: Bounds และ Action](#area-object-bounds-และ-action)
4. [Action Types ทั้งหมด](#action-types-ทั้งหมด)
5. [สร้าง Rich Menu ผ่าน API](#สร้าง-rich-menu-ผ่าน-api)
6. [อัพโหลด Rich Menu Image (Binary)](#อัพโหลด-rich-menu-image-binary)
7. [ตั้งค่า Default Rich Menu](#ตั้งค่า-default-rich-menu)
8. [Get Current Rich Menu](#get-current-rich-menu)
9. [Cancel Default Rich Menu](#cancel-default-rich-menu)
10. [Link/Unlink Rich Menu กับ User](#linkunlink-rich-menu-กับ-user)
11. [Bulk Link Rich Menu](#bulk-link-rich-menu)
12. [Rich Menu Alias](#rich-menu-alias)
13. [Switching Rich Menu ด้วย Alias](#switching-rich-menu-ด้วย-alias)
14. [สร้าง Switching Animation](#สร้าง-switching-animation)
15. [Node.js Rich Menu Manager Class](#nodejs-rich-menu-manager-class)
16. [Python Rich Menu Manager](#python-rich-menu-manager)
17. [Best Practices และ Security](#best-practices-และ-security)

---

## 1. Rich Menu API Overview

Rich Menu คือเมนูที่แสดงอยู่ด้านล่างหน้าจอแชทใน LINE แบ่งออกเป็น 2 แบบ:

1. **Default Rich Menu** - แสดงให้ผู้ใช้ทุกคนที่ยังไม่มี user-specific rich menu
2. **User-specific Rich Menu** - กำหนดเฉพาะผู้ใช้แต่ละคน

### Endpoint หลัก

```
Base URL: https://api.line.me/v2/bot/
```

| Method | Endpoint | คำอธิบาย |
|--------|----------|-----------|
| POST | `/richmenu` | สร้าง Rich Menu |
| GET | `/richmenu/{richMenuId}` | ดู Rich Menu |
| DELETE | `/richmenu/{richMenuId}` | ลบ Rich Menu |
| POST | `/richmenu/{richMenuId}/content` | อัพโหลดรูป |
| GET | `/richmenu/{richMenuId}/content` | ดาวน์โหลดรูป |
| GET | `/richmenu/list` | ดูรายการ Rich Menu ทั้งหมด |
| POST | `/user/all/richmenu/{richMenuId}` | ตั้งค่า Default |
| GET | `/user/all/richmenu` | ดู Default Rich Menu |
| DELETE | `/user/all/richmenu` | ยกเลิก Default |
| POST | `/user/{userId}/richmenu/{richMenuId}` | Link ให้ User |
| DELETE | `/user/{userId}/richmenu` | Unlink จาก User |
| GET | `/user/{userId}/richmenu` | ดู Rich Menu ของ User |
| POST | `/richmenu/bulk/link` | Bulk Link |
| POST | `/richmenu/bulk/unlink` | Bulk Unlink |
| POST | `/richmenu/alias` | สร้าง Alias |
| GET | `/richmenu/alias/{aliasId}` | ดู Alias |
| DELETE | `/richmenu/alias/{aliasId}` | ลบ Alias |
| GET | `/richmenu/alias/list` | ดูรายการ Alias |
| POST | `/richmenu/alias/{aliasId}` | อัพเดต Alias |

### การตั้งค่า Headers

```javascript
const headers = {
  'Authorization': `Bearer ${process.env.CHANNEL_ACCESS_TOKEN}`,
  'Content-Type': 'application/json'
};

// สำหรับอัพโหลดรูป
const imageHeaders = {
  'Authorization': `Bearer ${process.env.CHANNEL_ACCESS_TOKEN}`,
  'Content-Type': 'image/jpeg' // หรือ image/png
};
```

---

## 2. โครงสร้าง Rich Menu Object

```json
{
  "size": {
    "width": 2500,
    "height": 1686
  },
  "selected": false,
  "name": "ชื่อ Rich Menu (ไม่แสดงให้ผู้ใช้เห็น)",
  "chatBarText": "ข้อความที่แสดงในแถบเมนู",
  "areas": [
    {
      "bounds": {
        "x": 0,
        "y": 0,
        "width": 833,
        "height": 843
      },
      "action": {
        "type": "message",
        "label": "สั่งซื้อสินค้า",
        "text": "สั่งซื้อสินค้า"
      }
    }
  ]
}
```

### ขนาดรูปภาพที่รองรับ

| ขนาด | Width | Height | หมายเหตุ |
|------|-------|--------|----------|
| Full Size | 2500 | 1686 | มาตรฐาน - สูงเต็ม |
| Half Size | 2500 | 843 | มาตรฐาน - สูงครึ่ง |
| Custom | 2500 | 843-2500 | ขนาดที่กำหนดเอง |

### ขีดจำกัดที่สำคัญ

```
- จำนวน areas สูงสุด: 20 areas ต่อ Rich Menu
- ขนาดไฟล์รูป: สูงสุด 1 MB
- จำนวน Rich Menu ที่สร้างได้: สูงสุด 1000 Rich Menu
- จำนวน Alias: สูงสุด 100 aliases
- ขนาด chatBarText: สูงสุด 14 ตัวอักษร
- ขนาด name: สูงสุด 300 ตัวอักษร
```

---

## 3. Area Object: Bounds และ Action

### Bounds Object

bounds กำหนดตำแหน่งและขนาดของพื้นที่ที่กดได้ใน Rich Menu

```json
{
  "bounds": {
    "x": 0,      // ตำแหน่ง X (pixel จากซ้าย)
    "y": 0,      // ตำแหน่ง Y (pixel จากบน)
    "width": 833,  // ความกว้าง (pixel)
    "height": 843  // ความสูง (pixel)
  }
}
```

### Layout แบบ 3x2 (มาตรฐาน)

```
+--------+--------+--------+
| (0,0)  | (833,0)|(1666,0)|
| 833x843|833x843 | 833x843|
+--------+--------+--------+
| (0,843)|(833,843)|(1666,843)|
| 833x843|833x843 | 833x843|
+--------+--------+--------+
```

```javascript
// Helper สำหรับคำนวณ bounds แบบ Grid
function calculateGridBounds(cols, rows, menuWidth = 2500, menuHeight = 1686) {
  const cellWidth = Math.floor(menuWidth / cols);
  const cellHeight = Math.floor(menuHeight / rows);
  const areas = [];

  for (let row = 0; row < rows; row++) {
    for (let col = 0; col < cols; col++) {
      areas.push({
        bounds: {
          x: col * cellWidth,
          y: row * cellHeight,
          width: cellWidth,
          height: cellHeight
        }
      });
    }
  }
  return areas;
}

// ใช้งาน: สร้าง Grid 3x2
const gridAreas = calculateGridBounds(3, 2);
console.log(gridAreas);
/*
[
  { bounds: { x: 0, y: 0, width: 833, height: 843 } },
  { bounds: { x: 833, y: 0, width: 833, height: 843 } },
  { bounds: { x: 1666, y: 0, width: 833, height: 843 } },
  { bounds: { x: 0, y: 843, width: 833, height: 843 } },
  { bounds: { x: 833, y: 843, width: 833, height: 843 } },
  { bounds: { x: 1666, y: 843, width: 833, height: 843 } }
]
*/
```

### Layout แบบ Custom

```javascript
// Layout แบบ 2 คอลัมน์ บน + 3 คอลัมน์ ล่าง
const customLayout = [
  // แถวบน: 2 ปุ่มใหญ่
  { bounds: { x: 0, y: 0, width: 1250, height: 843 } },
  { bounds: { x: 1250, y: 0, width: 1250, height: 843 } },
  // แถวล่าง: 3 ปุ่มเล็ก
  { bounds: { x: 0, y: 843, width: 833, height: 843 } },
  { bounds: { x: 833, y: 843, width: 833, height: 843 } },
  { bounds: { x: 1666, y: 843, width: 834, height: 843 } }
];
```

---

## 4. Action Types ทั้งหมด

### 4.1 Message Action

ส่งข้อความแทนผู้ใช้เมื่อกดปุ่ม

```json
{
  "type": "message",
  "label": "สั่งซื้อ",
  "text": "ฉันต้องการสั่งซื้อสินค้า"
}
```

```javascript
// ตัวอย่างการใช้งาน
const messageAction = {
  type: 'message',
  label: 'ดูสินค้า',
  text: 'ดูสินค้าทั้งหมด'
};

// ใช้ใน Webhook Handler
app.post('/webhook', (req, res) => {
  const events = req.body.events;
  events.forEach(event => {
    if (event.type === 'message' && event.message.type === 'text') {
      const text = event.message.text;
      if (text === 'ดูสินค้าทั้งหมด') {
        // ส่งรายการสินค้า
        sendProductList(event.replyToken);
      }
    }
  });
  res.sendStatus(200);
});
```

### 4.2 URI Action

เปิด URL เมื่อกดปุ่ม

```json
{
  "type": "uri",
  "label": "เว็บไซต์",
  "uri": "https://example.com/products",
  "altUri": {
    "desktop": "https://example.com/products?platform=desktop"
  }
}
```

```javascript
// รองรับ LINE URL Scheme
const uriActions = [
  // เปิดเว็บ
  { type: 'uri', label: 'Website', uri: 'https://example.com' },
  // โทรศัพท์
  { type: 'uri', label: 'โทร', uri: 'tel:+6612345678' },
  // อีเมล
  { type: 'uri', label: 'Email', uri: 'mailto:info@example.com' },
  // LINE URL Scheme
  { type: 'uri', label: 'แชท', uri: 'https://line.me/R/ti/p/@username' },
  // LIFF
  { type: 'uri', label: 'เปิดแอป', uri: 'https://liff.line.me/1234567890-abcdefgh' }
];
```

### 4.3 Postback Action

ส่ง postback data ไปยัง webhook (ไม่แสดงข้อความในแชท)

```json
{
  "type": "postback",
  "label": "ประวัติคำสั่งซื้อ",
  "data": "action=order_history&page=1",
  "displayText": "ดูประวัติคำสั่งซื้อ",
  "inputOption": "openKeyboard",
  "fillInText": "กรอก Order ID: "
}
```

```javascript
// ตัวอย่างการ Handle Postback
app.post('/webhook', (req, res) => {
  const events = req.body.events;
  events.forEach(async event => {
    if (event.type === 'postback') {
      const data = new URLSearchParams(event.postback.data);
      const action = data.get('action');

      switch (action) {
        case 'order_history':
          const page = parseInt(data.get('page')) || 1;
          await sendOrderHistory(event.replyToken, event.source.userId, page);
          break;
        case 'view_product':
          const productId = data.get('id');
          await sendProductDetail(event.replyToken, productId);
          break;
        case 'add_cart':
          const itemId = data.get('item');
          await addToCart(event.source.userId, itemId);
          await client.replyMessage(event.replyToken, {
            type: 'text',
            text: 'เพิ่มสินค้าในตะกร้าแล้ว!'
          });
          break;
      }
    }
  });
  res.sendStatus(200);
});
```

### 4.4 Datetimepicker Action

เปิด Date/Time picker

```json
{
  "type": "datetimepicker",
  "label": "เลือกวันนัดหมาย",
  "data": "action=appointment&type=date",
  "mode": "datetime",
  "initial": "2024-01-15t10:00",
  "max": "2024-12-31t23:59",
  "min": "2024-01-01t00:00"
}
```

```javascript
// mode options: date, time, datetime
const dateActions = {
  dateOnly: {
    type: 'datetimepicker',
    label: 'เลือกวันที่',
    data: 'action=select_date',
    mode: 'date',
    initial: '2024-01-15',
    max: '2024-12-31',
    min: '2024-01-01'
  },
  timeOnly: {
    type: 'datetimepicker',
    label: 'เลือกเวลา',
    data: 'action=select_time',
    mode: 'time',
    initial: '10:00',
    max: '18:00',
    min: '09:00'
  },
  dateTime: {
    type: 'datetimepicker',
    label: 'เลือกวันและเวลา',
    data: 'action=select_datetime',
    mode: 'datetime',
    initial: '2024-01-15t10:00',
    max: '2024-12-31t23:59',
    min: '2024-01-01t00:00'
  }
};

// Handle Postback จาก DatetimePicker
app.post('/webhook', (req, res) => {
  const events = req.body.events;
  events.forEach(async event => {
    if (event.type === 'postback' && event.postback.params) {
      const params = event.postback.params;
      // params.date, params.time, params.datetime
      if (params.datetime) {
        const selectedDate = new Date(params.datetime);
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: `นัดหมายวันที่ ${selectedDate.toLocaleDateString('th-TH')} เวลา ${selectedDate.toLocaleTimeString('th-TH')}`
        });
      }
    }
  });
  res.sendStatus(200);
});
```

### 4.5 Camera Action

เปิดกล้องถ่ายภาพ

```json
{
  "type": "camera",
  "label": "ถ่ายรูป"
}
```

### 4.6 Camera Roll Action

เปิดคลังรูปภาพ

```json
{
  "type": "cameraRoll",
  "label": "เลือกรูปภาพ"
}
```

### 4.7 Location Action

เปิดตัวเลือกส่งตำแหน่ง

```json
{
  "type": "location",
  "label": "ส่งที่อยู่"
}
```

### 4.8 RichMenu Switch Action

สลับ Rich Menu โดยใช้ Alias

```json
{
  "type": "richmenuswitch",
  "richMenuAliasId": "richmenu-alias-shop",
  "data": "action=switch_to_shop"
}
```

---

## 5. สร้าง Rich Menu ผ่าน API

### ตัวอย่างการสร้าง Rich Menu แบบสมบูรณ์

```javascript
const axios = require('axios');
require('dotenv').config();

const LINE_API_URL = 'https://api.line.me/v2/bot';
const CHANNEL_ACCESS_TOKEN = process.env.CHANNEL_ACCESS_TOKEN;

const headers = {
  'Authorization': `Bearer ${CHANNEL_ACCESS_TOKEN}`,
  'Content-Type': 'application/json'
};

// สร้าง Rich Menu แบบ 3x2 Grid
async function createMainRichMenu() {
  const richMenu = {
    size: {
      width: 2500,
      height: 1686
    },
    selected: true,
    name: 'main-menu',
    chatBarText: 'เมนูหลัก',
    areas: [
      {
        bounds: { x: 0, y: 0, width: 833, height: 843 },
        action: {
          type: 'message',
          label: 'สั่งซื้อสินค้า',
          text: 'สั่งซื้อสินค้า'
        }
      },
      {
        bounds: { x: 833, y: 0, width: 833, height: 843 },
        action: {
          type: 'uri',
          label: 'โปรโมชัน',
          uri: 'https://example.com/promotions'
        }
      },
      {
        bounds: { x: 1666, y: 0, width: 834, height: 843 },
        action: {
          type: 'postback',
          label: 'ติดต่อเรา',
          data: 'action=contact',
          displayText: 'ติดต่อเรา'
        }
      },
      {
        bounds: { x: 0, y: 843, width: 833, height: 843 },
        action: {
          type: 'postback',
          label: 'ตะกร้าสินค้า',
          data: 'action=view_cart',
          displayText: 'ดูตะกร้าสินค้า'
        }
      },
      {
        bounds: { x: 833, y: 843, width: 833, height: 843 },
        action: {
          type: 'postback',
          label: 'ประวัติคำสั่งซื้อ',
          data: 'action=order_history',
          displayText: 'ดูประวัติคำสั่งซื้อ'
        }
      },
      {
        bounds: { x: 1666, y: 843, width: 834, height: 843 },
        action: {
          type: 'postback',
          label: 'บัญชีของฉัน',
          data: 'action=my_account',
          displayText: 'บัญชีของฉัน'
        }
      }
    ]
  };

  try {
    const response = await axios.post(
      `${LINE_API_URL}/richmenu`,
      richMenu,
      { headers }
    );
    console.log('Rich Menu created:', response.data.richMenuId);
    return response.data.richMenuId;
  } catch (error) {
    console.error('Error creating rich menu:', error.response?.data);
    throw error;
  }
}
```

### Response จาก API

```json
{
  "richMenuId": "richmenu-88c05ef6921ae535095e21d9f4e4c9e5"
}
```

---

## 6. อัพโหลด Rich Menu Image (Binary)

### อัพโหลดด้วย Node.js

```javascript
const fs = require('fs');
const axios = require('axios');
const FormData = require('form-data');

async function uploadRichMenuImage(richMenuId, imagePath) {
  const imageBuffer = fs.readFileSync(imagePath);
  const mimeType = imagePath.endsWith('.png') ? 'image/png' : 'image/jpeg';

  try {
    const response = await axios.post(
      `${LINE_API_URL}/richmenu/${richMenuId}/content`,
      imageBuffer,
      {
        headers: {
          'Authorization': `Bearer ${CHANNEL_ACCESS_TOKEN}`,
          'Content-Type': mimeType
        }
      }
    );
    console.log('Image uploaded successfully');
    return response.data;
  } catch (error) {
    console.error('Error uploading image:', error.response?.data);
    throw error;
  }
}

// ใช้งาน
async function setupRichMenu() {
  try {
    // 1. สร้าง Rich Menu
    const richMenuId = await createMainRichMenu();
    console.log('Rich Menu ID:', richMenuId);

    // 2. อัพโหลดรูป
    await uploadRichMenuImage(richMenuId, './images/rich-menu.jpg');
    console.log('Image uploaded');

    // 3. ตั้งค่าเป็น Default
    await setDefaultRichMenu(richMenuId);
    console.log('Set as default');

    return richMenuId;
  } catch (error) {
    console.error('Setup failed:', error);
    throw error;
  }
}
```

### อัพโหลดด้วย Sharp สำหรับ Resize อัตโนมัติ

```javascript
const sharp = require('sharp');
const axios = require('axios');

async function uploadRichMenuImageWithResize(richMenuId, imagePath) {
  try {
    // Resize รูปให้ได้ขนาดที่ต้องการ
    const resizedBuffer = await sharp(imagePath)
      .resize(2500, 1686, {
        fit: 'cover',
        position: 'center'
      })
      .jpeg({ quality: 90 })
      .toBuffer();

    // ตรวจสอบขนาดไฟล์ (ต้องไม่เกิน 1MB)
    const fileSizeKB = resizedBuffer.length / 1024;
    if (fileSizeKB > 1024) {
      throw new Error(`ไฟล์ใหญ่เกินไป: ${fileSizeKB.toFixed(0)}KB (สูงสุด 1024KB)`);
    }

    console.log(`ขนาดไฟล์: ${fileSizeKB.toFixed(0)}KB`);

    const response = await axios.post(
      `${LINE_API_URL}/richmenu/${richMenuId}/content`,
      resizedBuffer,
      {
        headers: {
          'Authorization': `Bearer ${CHANNEL_ACCESS_TOKEN}`,
          'Content-Type': 'image/jpeg'
        }
      }
    );

    return response.data;
  } catch (error) {
    console.error('Error:', error.message);
    throw error;
  }
}
```

### ดาวน์โหลดรูป Rich Menu

```javascript
async function downloadRichMenuImage(richMenuId, savePath) {
  try {
    const response = await axios.get(
      `${LINE_API_URL}/richmenu/${richMenuId}/content`,
      {
        headers: { 'Authorization': `Bearer ${CHANNEL_ACCESS_TOKEN}` },
        responseType: 'arraybuffer'
      }
    );

    fs.writeFileSync(savePath, response.data);
    console.log(`รูปบันทึกที่: ${savePath}`);
    return savePath;
  } catch (error) {
    console.error('Error downloading image:', error.response?.data);
    throw error;
  }
}
```

---

## 7. ตั้งค่า Default Rich Menu

```javascript
async function setDefaultRichMenu(richMenuId) {
  try {
    const response = await axios.post(
      `${LINE_API_URL}/user/all/richmenu/${richMenuId}`,
      {},
      { headers }
    );
    console.log('Default rich menu set:', richMenuId);
    return response.data;
  } catch (error) {
    console.error('Error setting default:', error.response?.data);
    throw error;
  }
}
```

### Response

```json
{}
```

(Response เป็น empty object เมื่อสำเร็จ)

---

## 8. Get Current Rich Menu

### ดู Default Rich Menu

```javascript
async function getDefaultRichMenu() {
  try {
    const response = await axios.get(
      `${LINE_API_URL}/user/all/richmenu`,
      { headers }
    );
    return response.data;
  } catch (error) {
    if (error.response?.status === 404) {
      console.log('ไม่มี Default Rich Menu');
      return null;
    }
    throw error;
  }
}
```

### Response

```json
{
  "richMenuId": "richmenu-88c05ef6921ae535095e21d9f4e4c9e5"
}
```

### ดู Rich Menu ของ User

```javascript
async function getUserRichMenu(userId) {
  try {
    const response = await axios.get(
      `${LINE_API_URL}/user/${userId}/richmenu`,
      { headers }
    );
    return response.data;
  } catch (error) {
    if (error.response?.status === 404) {
      return null;
    }
    throw error;
  }
}
```

### ดูข้อมูล Rich Menu ทั้งหมด

```javascript
async function getRichMenuDetails(richMenuId) {
  try {
    const response = await axios.get(
      `${LINE_API_URL}/richmenu/${richMenuId}`,
      { headers }
    );
    return response.data;
  } catch (error) {
    console.error('Error:', error.response?.data);
    throw error;
  }
}

// ดูรายการ Rich Menu ทั้งหมด
async function getAllRichMenus() {
  try {
    const response = await axios.get(
      `${LINE_API_URL}/richmenu/list`,
      { headers }
    );
    return response.data.richmenus;
  } catch (error) {
    console.error('Error:', error.response?.data);
    throw error;
  }
}
```

### Response ของ getRichMenuDetails

```json
{
  "richMenuId": "richmenu-88c05ef6921ae535095e21d9f4e4c9e5",
  "size": {
    "width": 2500,
    "height": 1686
  },
  "selected": true,
  "name": "main-menu",
  "chatBarText": "เมนูหลัก",
  "areas": [
    {
      "bounds": {
        "x": 0,
        "y": 0,
        "width": 833,
        "height": 843
      },
      "action": {
        "type": "message",
        "label": "สั่งซื้อสินค้า",
        "text": "สั่งซื้อสินค้า"
      }
    }
  ]
}
```

---

## 9. Cancel Default Rich Menu

```javascript
async function cancelDefaultRichMenu() {
  try {
    const response = await axios.delete(
      `${LINE_API_URL}/user/all/richmenu`,
      { headers }
    );
    console.log('Default rich menu cancelled');
    return response.data;
  } catch (error) {
    console.error('Error:', error.response?.data);
    throw error;
  }
}

// ลบ Rich Menu
async function deleteRichMenu(richMenuId) {
  try {
    const response = await axios.delete(
      `${LINE_API_URL}/richmenu/${richMenuId}`,
      { headers }
    );
    console.log('Rich Menu deleted:', richMenuId);
    return response.data;
  } catch (error) {
    console.error('Error:', error.response?.data);
    throw error;
  }
}
```

---

## 10. Link/Unlink Rich Menu กับ User

### Link Rich Menu ให้ User

```javascript
async function linkRichMenuToUser(userId, richMenuId) {
  try {
    const response = await axios.post(
      `${LINE_API_URL}/user/${userId}/richmenu/${richMenuId}`,
      {},
      { headers }
    );
    console.log(`Linked ${richMenuId} to ${userId}`);
    return response.data;
  } catch (error) {
    console.error('Error linking:', error.response?.data);
    throw error;
  }
}

// Unlink Rich Menu จาก User
async function unlinkRichMenuFromUser(userId) {
  try {
    const response = await axios.delete(
      `${LINE_API_URL}/user/${userId}/richmenu`,
      { headers }
    );
    console.log(`Unlinked rich menu from ${userId}`);
    return response.data;
  } catch (error) {
    console.error('Error unlinking:', error.response?.data);
    throw error;
  }
}
```

### ตัวอย่างการ Link Rich Menu ตามสถานะผู้ใช้

```javascript
// ระบบ Rich Menu ตามสถานะ Member
async function updateUserRichMenu(userId, userStatus) {
  const richMenuMap = {
    'guest': process.env.RICH_MENU_GUEST_ID,
    'member': process.env.RICH_MENU_MEMBER_ID,
    'vip': process.env.RICH_MENU_VIP_ID,
    'admin': process.env.RICH_MENU_ADMIN_ID
  };

  const richMenuId = richMenuMap[userStatus] || richMenuMap['guest'];

  try {
    await linkRichMenuToUser(userId, richMenuId);
    console.log(`User ${userId} upgraded to ${userStatus} menu`);
  } catch (error) {
    console.error(`Failed to update menu for ${userId}:`, error);
  }
}

// Hook เมื่อผู้ใช้ Follow
app.post('/webhook', async (req, res) => {
  const events = req.body.events;

  for (const event of events) {
    if (event.type === 'follow') {
      const userId = event.source.userId;

      // ดูว่า user เป็น member หรือยัง
      const user = await db.users.findOne({ lineUserId: userId });
      const status = user ? user.memberStatus : 'guest';

      await updateUserRichMenu(userId, status);
    }
  }

  res.sendStatus(200);
});
```

---

## 11. Bulk Link Rich Menu

### Link Rich Menu ให้หลาย User พร้อมกัน

```javascript
async function bulkLinkRichMenu(userIds, richMenuId) {
  // LINE API รองรับสูงสุด 150,000 users ต่อ request
  const BATCH_SIZE = 500; // แบ่งเป็น batches ย่อยๆ
  const results = [];

  for (let i = 0; i < userIds.length; i += BATCH_SIZE) {
    const batch = userIds.slice(i, i + BATCH_SIZE);

    try {
      const response = await axios.post(
        `${LINE_API_URL}/richmenu/bulk/link`,
        {
          richMenuId: richMenuId,
          userIds: batch
        },
        { headers }
      );
      results.push({ success: true, count: batch.length });
      console.log(`Linked ${batch.length} users (batch ${Math.floor(i/BATCH_SIZE) + 1})`);
    } catch (error) {
      results.push({ success: false, error: error.response?.data });
      console.error(`Batch ${Math.floor(i/BATCH_SIZE) + 1} failed:`, error.response?.data);
    }

    // หน่วงเวลาเพื่อป้องกัน Rate Limit
    if (i + BATCH_SIZE < userIds.length) {
      await new Promise(resolve => setTimeout(resolve, 1000));
    }
  }

  return results;
}

// Bulk Unlink
async function bulkUnlinkRichMenu(userIds) {
  try {
    const response = await axios.post(
      `${LINE_API_URL}/richmenu/bulk/unlink`,
      { userIds },
      { headers }
    );
    return response.data;
  } catch (error) {
    console.error('Error bulk unlinking:', error.response?.data);
    throw error;
  }
}

// ตัวอย่างการใช้ Bulk Link ในระบบจริง
async function upgradeAllMembersRichMenu() {
  try {
    // ดึงรายชื่อ VIP members ทั้งหมด
    const vipMembers = await db.users.find({
      memberStatus: 'vip',
      lineUserId: { $exists: true }
    }).select('lineUserId');

    const userIds = vipMembers.map(m => m.lineUserId);
    console.log(`Upgrading ${userIds.length} VIP members`);

    const results = await bulkLinkRichMenu(
      userIds,
      process.env.RICH_MENU_VIP_ID
    );

    const successCount = results.filter(r => r.success).reduce((sum, r) => sum + r.count, 0);
    console.log(`Successfully linked: ${successCount}/${userIds.length}`);
  } catch (error) {
    console.error('Upgrade failed:', error);
  }
}
```

---

## 12. Rich Menu Alias

Alias คือชื่อที่ใช้แทน richMenuId เพื่อให้ง่ายต่อการจัดการ และรองรับ switching animation

### สร้าง Rich Menu Alias

```javascript
async function createRichMenuAlias(richMenuId, aliasId) {
  try {
    const response = await axios.post(
      `${LINE_API_URL}/richmenu/alias`,
      {
        richMenuId: richMenuId,
        richMenuAliasId: aliasId
      },
      { headers }
    );
    console.log(`Alias created: ${aliasId} -> ${richMenuId}`);
    return response.data;
  } catch (error) {
    console.error('Error creating alias:', error.response?.data);
    throw error;
  }
}

// อัพเดต Alias (เปลี่ยน Rich Menu ที่ Alias ชี้ไป)
async function updateRichMenuAlias(aliasId, newRichMenuId) {
  try {
    const response = await axios.post(
      `${LINE_API_URL}/richmenu/alias/${aliasId}`,
      {
        richMenuId: newRichMenuId
      },
      { headers }
    );
    console.log(`Alias updated: ${aliasId} -> ${newRichMenuId}`);
    return response.data;
  } catch (error) {
    console.error('Error updating alias:', error.response?.data);
    throw error;
  }
}

// ดู Alias
async function getRichMenuAlias(aliasId) {
  try {
    const response = await axios.get(
      `${LINE_API_URL}/richmenu/alias/${aliasId}`,
      { headers }
    );
    return response.data;
  } catch (error) {
    console.error('Error getting alias:', error.response?.data);
    throw error;
  }
}

// ดูรายการ Alias ทั้งหมด
async function getAllRichMenuAliases() {
  try {
    const response = await axios.get(
      `${LINE_API_URL}/richmenu/alias/list`,
      { headers }
    );
    return response.data.aliases;
  } catch (error) {
    console.error('Error getting aliases:', error.response?.data);
    throw error;
  }
}

// ลบ Alias
async function deleteRichMenuAlias(aliasId) {
  try {
    const response = await axios.delete(
      `${LINE_API_URL}/richmenu/alias/${aliasId}`,
      { headers }
    );
    console.log(`Alias deleted: ${aliasId}`);
    return response.data;
  } catch (error) {
    console.error('Error deleting alias:', error.response?.data);
    throw error;
  }
}
```

### Response ของ getRichMenuAlias

```json
{
  "richMenuAliasId": "richmenu-alias-main",
  "richMenuId": "richmenu-88c05ef6921ae535095e21d9f4e4c9e5"
}
```

---

## 13. Switching Rich Menu ด้วย Alias

### การตั้งค่า richmenuswitch Action

Rich Menu สามารถสลับระหว่างกันได้โดยใช้ `richmenuswitch` action ร่วมกับ Alias

```javascript
// Rich Menu A (หน้าหลัก) - มีปุ่มไปหน้า Shop
const mainMenu = {
  size: { width: 2500, height: 1686 },
  selected: true,
  name: 'main-menu',
  chatBarText: 'เมนูหลัก',
  areas: [
    {
      bounds: { x: 0, y: 0, width: 1250, height: 1686 },
      action: {
        type: 'richmenuswitch',
        richMenuAliasId: 'richmenu-alias-shop',
        data: 'action=switch_to_shop'
      }
    },
    {
      bounds: { x: 1250, y: 0, width: 1250, height: 843 },
      action: {
        type: 'message',
        label: 'ติดต่อเรา',
        text: 'ติดต่อเรา'
      }
    },
    {
      bounds: { x: 1250, y: 843, width: 1250, height: 843 },
      action: {
        type: 'postback',
        label: 'โปรโมชัน',
        data: 'action=promotions'
      }
    }
  ]
};

// Rich Menu B (หน้า Shop) - มีปุ่มกลับหน้าหลัก
const shopMenu = {
  size: { width: 2500, height: 1686 },
  selected: true,
  name: 'shop-menu',
  chatBarText: 'เมนูช้อป',
  areas: [
    {
      bounds: { x: 0, y: 0, width: 833, height: 843 },
      action: {
        type: 'message',
        label: 'ดูสินค้า',
        text: 'ดูสินค้าทั้งหมด'
      }
    },
    {
      bounds: { x: 833, y: 0, width: 833, height: 843 },
      action: {
        type: 'postback',
        label: 'ตะกร้า',
        data: 'action=cart'
      }
    },
    {
      bounds: { x: 1666, y: 0, width: 834, height: 843 },
      action: {
        type: 'postback',
        label: 'ประวัติ',
        data: 'action=history'
      }
    },
    {
      bounds: { x: 0, y: 843, width: 2500, height: 843 },
      action: {
        type: 'richmenuswitch',
        richMenuAliasId: 'richmenu-alias-main',
        data: 'action=switch_to_main'
      }
    }
  ]
};
```

### ขั้นตอนการตั้งค่า Switching

```javascript
async function setupSwitchableRichMenus() {
  try {
    // 1. สร้าง Rich Menu ทั้งสอง
    const [mainMenuId, shopMenuId] = await Promise.all([
      createRichMenuFromObject(mainMenu),
      createRichMenuFromObject(shopMenu)
    ]);

    console.log('Main Menu ID:', mainMenuId);
    console.log('Shop Menu ID:', shopMenuId);

    // 2. อัพโหลดรูปทั้งสอง
    await Promise.all([
      uploadRichMenuImage(mainMenuId, './images/main-menu.jpg'),
      uploadRichMenuImage(shopMenuId, './images/shop-menu.jpg')
    ]);

    // 3. สร้าง Aliases
    await createRichMenuAlias(mainMenuId, 'richmenu-alias-main');
    await createRichMenuAlias(shopMenuId, 'richmenu-alias-shop');

    // 4. ตั้งค่า Default เป็นหน้าหลัก
    await setDefaultRichMenu(mainMenuId);

    console.log('Switchable Rich Menus setup complete!');
    return { mainMenuId, shopMenuId };
  } catch (error) {
    console.error('Setup failed:', error);
    throw error;
  }
}

async function createRichMenuFromObject(menuObject) {
  const response = await axios.post(
    `${LINE_API_URL}/richmenu`,
    menuObject,
    { headers }
  );
  return response.data.richMenuId;
}
```

---

## 14. สร้าง Switching Animation

เมื่อใช้ `richmenuswitch` action จะมี animation การเปลี่ยนเมนูโดยอัตโนมัติ แต่เราสามารถสร้าง "ภาพลวงตา" ของ animation ที่ซับซ้อนขึ้นได้

### แนวคิด Animated Rich Menu

```
+------------------+     กด "ดูสินค้า"    +------------------+
|    Main Menu     |  ─────────────────>  |    Shop Menu     |
| [บ้าน] [ช้อป] [?]|                      | [สินค้า][ตะกร้า]  |
| [ส่วนลด][บัญชี]  |  <─────────────────  | [ประวัติ] [กลับ]  |
+------------------+   กด "กลับหน้าหลัก"  +------------------+
```

### Multi-level Rich Menu System

```javascript
class RichMenuSystem {
  constructor(channelAccessToken) {
    this.token = channelAccessToken;
    this.headers = {
      'Authorization': `Bearer ${channelAccessToken}`,
      'Content-Type': 'application/json'
    };
    this.menuMap = {
      'main': null,
      'shop': null,
      'account': null,
      'support': null
    };
    this.aliasMap = {
      'main': 'richmenu-alias-main',
      'shop': 'richmenu-alias-shop',
      'account': 'richmenu-alias-account',
      'support': 'richmenu-alias-support'
    };
  }

  // สร้าง Rich Menu พร้อม navigation
  createMenuConfig(name, sections) {
    const areas = [];
    const gridCols = 3;
    const gridRows = 2;
    const cellWidth = Math.floor(2500 / gridCols);
    const cellHeight = Math.floor(1686 / gridRows);

    sections.forEach((section, index) => {
      const row = Math.floor(index / gridCols);
      const col = index % gridCols;
      areas.push({
        bounds: {
          x: col * cellWidth,
          y: row * cellHeight,
          width: col === gridCols - 1 ? 2500 - (gridCols - 1) * cellWidth : cellWidth,
          height: cellHeight
        },
        action: section.action
      });
    });

    return {
      size: { width: 2500, height: 1686 },
      selected: true,
      name,
      chatBarText: this.getMenuLabel(name),
      areas
    };
  }

  getMenuLabel(name) {
    const labels = {
      'main': 'เมนูหลัก',
      'shop': 'ช้อปปิ้ง',
      'account': 'บัญชีของฉัน',
      'support': 'ช่วยเหลือ'
    };
    return labels[name] || name;
  }

  // สร้าง Navigation Action ไปยัง Menu อื่น
  switchAction(targetMenu, data = '') {
    return {
      type: 'richmenuswitch',
      richMenuAliasId: this.aliasMap[targetMenu],
      data: data || `action=switch_to_${targetMenu}`
    };
  }

  // สร้าง Menu ทั้งหมดพร้อมกัน
  async initializeAllMenus(menuImages) {
    const menuConfigs = {
      main: this.createMenuConfig('main', [
        { action: { type: 'message', label: 'สั่งซื้อ', text: 'สั่งซื้อสินค้า' } },
        { action: { type: 'uri', label: 'โปรโมชัน', uri: 'https://example.com/promo' } },
        { action: this.switchAction('support', 'action=goto_support') },
        { action: this.switchAction('shop', 'action=goto_shop') },
        { action: this.switchAction('account', 'action=goto_account') },
        { action: { type: 'postback', label: 'เกี่ยวกับเรา', data: 'action=about' } }
      ]),
      shop: this.createMenuConfig('shop', [
        { action: { type: 'message', label: 'ดูสินค้า', text: 'แสดงสินค้าทั้งหมด' } },
        { action: { type: 'postback', label: 'ตะกร้า', data: 'action=cart' } },
        { action: { type: 'postback', label: 'ประวัติ', data: 'action=history' } },
        { action: { type: 'message', label: 'ค้นหาสินค้า', text: 'ค้นหาสินค้า' } },
        { action: { type: 'postback', label: 'ส่วนลด', data: 'action=discounts' } },
        { action: this.switchAction('main', 'action=goto_main') }
      ]),
      account: this.createMenuConfig('account', [
        { action: { type: 'postback', label: 'ข้อมูลส่วนตัว', data: 'action=profile' } },
        { action: { type: 'postback', label: 'คะแนนสะสม', data: 'action=points' } },
        { action: { type: 'postback', label: 'ที่อยู่จัดส่ง', data: 'action=addresses' } },
        { action: { type: 'postback', label: 'วิธีชำระเงิน', data: 'action=payment' } },
        { action: { type: 'postback', label: 'การแจ้งเตือน', data: 'action=notifications' } },
        { action: this.switchAction('main', 'action=goto_main') }
      ]),
      support: this.createMenuConfig('support', [
        { action: { type: 'message', label: 'FAQ', text: 'คำถามที่พบบ่อย' } },
        { action: { type: 'uri', label: 'ติดต่อเรา', uri: 'https://example.com/contact' } },
        { action: { type: 'postback', label: 'แจ้งปัญหา', data: 'action=report_issue' } },
        { action: { type: 'uri', label: 'นโยบาย', uri: 'https://example.com/policy' } },
        { action: { type: 'postback', label: 'Live Chat', data: 'action=live_chat' } },
        { action: this.switchAction('main', 'action=goto_main') }
      ])
    };

    // สร้างทุก Menu พร้อมกัน
    const createPromises = Object.entries(menuConfigs).map(async ([name, config]) => {
      const id = await this.createMenu(config);
      this.menuMap[name] = id;
      return { name, id };
    });

    await Promise.all(createPromises);

    // อัพโหลดรูปทุก Menu
    const uploadPromises = Object.entries(menuImages).map(([name, imagePath]) => {
      const menuId = this.menuMap[name];
      if (menuId) {
        return this.uploadImage(menuId, imagePath);
      }
    });

    await Promise.all(uploadPromises);

    // สร้าง Aliases
    for (const [name, aliasId] of Object.entries(this.aliasMap)) {
      const menuId = this.menuMap[name];
      if (menuId) {
        await this.createOrUpdateAlias(aliasId, menuId);
      }
    }

    // ตั้งค่า Default
    await this.setDefault('main');

    return this.menuMap;
  }

  async createMenu(config) {
    const response = await axios.post(
      `${LINE_API_URL}/richmenu`,
      config,
      { headers: this.headers }
    );
    return response.data.richMenuId;
  }

  async uploadImage(menuId, imagePath) {
    const buffer = fs.readFileSync(imagePath);
    const mimeType = imagePath.endsWith('.png') ? 'image/png' : 'image/jpeg';

    await axios.post(
      `${LINE_API_URL}/richmenu/${menuId}/content`,
      buffer,
      {
        headers: {
          ...this.headers,
          'Content-Type': mimeType
        }
      }
    );
  }

  async createOrUpdateAlias(aliasId, menuId) {
    try {
      // ลองสร้างใหม่ก่อน
      await axios.post(
        `${LINE_API_URL}/richmenu/alias`,
        { richMenuId: menuId, richMenuAliasId: aliasId },
        { headers: this.headers }
      );
    } catch (error) {
      if (error.response?.status === 400) {
        // Alias มีอยู่แล้ว - อัพเดต
        await axios.post(
          `${LINE_API_URL}/richmenu/alias/${aliasId}`,
          { richMenuId: menuId },
          { headers: this.headers }
        );
      } else {
        throw error;
      }
    }
  }

  async setDefault(menuName) {
    const menuId = this.menuMap[menuName];
    if (!menuId) throw new Error(`Menu '${menuName}' not found`);

    await axios.post(
      `${LINE_API_URL}/user/all/richmenu/${menuId}`,
      {},
      { headers: this.headers }
    );
  }
}
```

---

## 15. Node.js Rich Menu Manager Class

```javascript
// rich-menu-manager.js
const axios = require('axios');
const fs = require('fs');
const sharp = require('sharp');
require('dotenv').config();

class RichMenuManager {
  constructor(options = {}) {
    this.channelAccessToken = options.token || process.env.CHANNEL_ACCESS_TOKEN;
    this.baseUrl = 'https://api.line.me/v2/bot';
    this.defaultHeaders = {
      'Authorization': `Bearer ${this.channelAccessToken}`,
      'Content-Type': 'application/json'
    };
    this.cache = new Map();
  }

  // ─── Core Methods ────────────────────────────────────────────

  async create(richMenuConfig) {
    this._validateConfig(richMenuConfig);

    const response = await this._request('POST', '/richmenu', richMenuConfig);
    const { richMenuId } = response;

    this.cache.set(`menu:${richMenuId}`, richMenuConfig);
    return richMenuId;
  }

  async get(richMenuId) {
    if (this.cache.has(`menu:${richMenuId}`)) {
      return this.cache.get(`menu:${richMenuId}`);
    }

    const data = await this._request('GET', `/richmenu/${richMenuId}`);
    this.cache.set(`menu:${richMenuId}`, data);
    return data;
  }

  async delete(richMenuId) {
    await this._request('DELETE', `/richmenu/${richMenuId}`);
    this.cache.delete(`menu:${richMenuId}`);
    return true;
  }

  async list() {
    const data = await this._request('GET', '/richmenu/list');
    return data.richmenus || [];
  }

  // ─── Image Methods ────────────────────────────────────────────

  async uploadImage(richMenuId, imageSource, options = {}) {
    let imageBuffer;
    let mimeType = options.mimeType || 'image/jpeg';

    if (Buffer.isBuffer(imageSource)) {
      imageBuffer = imageSource;
    } else if (typeof imageSource === 'string') {
      // File path
      imageBuffer = fs.readFileSync(imageSource);
      mimeType = imageSource.endsWith('.png') ? 'image/png' : 'image/jpeg';
    } else {
      throw new Error('imageSource must be a Buffer or file path string');
    }

    // Auto-resize หากระบุ
    if (options.resize) {
      const { width = 2500, height = 1686, quality = 90 } = options.resize;
      imageBuffer = await sharp(imageBuffer)
        .resize(width, height, { fit: 'cover' })
        .jpeg({ quality })
        .toBuffer();
    }

    // ตรวจสอบขนาด
    const fileSizeKB = imageBuffer.length / 1024;
    if (fileSizeKB > 1024) {
      throw new Error(`Image too large: ${fileSizeKB.toFixed(0)}KB (max 1024KB)`);
    }

    await axios.post(
      `${this.baseUrl}/richmenu/${richMenuId}/content`,
      imageBuffer,
      {
        headers: {
          'Authorization': `Bearer ${this.channelAccessToken}`,
          'Content-Type': mimeType
        }
      }
    );

    return { richMenuId, fileSizeKB: fileSizeKB.toFixed(0) };
  }

  async downloadImage(richMenuId, savePath) {
    const response = await axios.get(
      `${this.baseUrl}/richmenu/${richMenuId}/content`,
      {
        headers: { 'Authorization': `Bearer ${this.channelAccessToken}` },
        responseType: 'arraybuffer'
      }
    );

    if (savePath) {
      fs.writeFileSync(savePath, response.data);
    }

    return response.data;
  }

  // ─── Default Methods ────────────────────────────────────────────

  async setDefault(richMenuId) {
    await this._request('POST', `/user/all/richmenu/${richMenuId}`, {});
    return true;
  }

  async getDefault() {
    try {
      return await this._request('GET', '/user/all/richmenu');
    } catch (error) {
      if (error.response?.status === 404) return null;
      throw error;
    }
  }

  async cancelDefault() {
    await this._request('DELETE', '/user/all/richmenu');
    return true;
  }

  // ─── User Link Methods ────────────────────────────────────────────

  async linkToUser(userId, richMenuId) {
    await this._request('POST', `/user/${userId}/richmenu/${richMenuId}`, {});
    return true;
  }

  async unlinkFromUser(userId) {
    await this._request('DELETE', `/user/${userId}/richmenu`);
    return true;
  }

  async getUserMenu(userId) {
    try {
      return await this._request('GET', `/user/${userId}/richmenu`);
    } catch (error) {
      if (error.response?.status === 404) return null;
      throw error;
    }
  }

  // ─── Bulk Methods ────────────────────────────────────────────

  async bulkLink(userIds, richMenuId, options = {}) {
    const batchSize = options.batchSize || 500;
    const delay = options.delay || 1000;
    const results = [];

    for (let i = 0; i < userIds.length; i += batchSize) {
      const batch = userIds.slice(i, i + batchSize);

      try {
        await this._request('POST', '/richmenu/bulk/link', {
          richMenuId,
          userIds: batch
        });
        results.push({ success: true, start: i, count: batch.length });
      } catch (error) {
        results.push({
          success: false,
          start: i,
          count: batch.length,
          error: error.response?.data
        });
      }

      if (options.onProgress) {
        options.onProgress({
          processed: Math.min(i + batchSize, userIds.length),
          total: userIds.length,
          percentage: Math.round(Math.min((i + batchSize) / userIds.length * 100, 100))
        });
      }

      if (i + batchSize < userIds.length && delay > 0) {
        await this._sleep(delay);
      }
    }

    return {
      total: userIds.length,
      success: results.filter(r => r.success).reduce((s, r) => s + r.count, 0),
      failed: results.filter(r => !r.success).reduce((s, r) => s + r.count, 0),
      batches: results
    };
  }

  async bulkUnlink(userIds, options = {}) {
    const batchSize = options.batchSize || 500;
    const results = [];

    for (let i = 0; i < userIds.length; i += batchSize) {
      const batch = userIds.slice(i, i + batchSize);
      try {
        await this._request('POST', '/richmenu/bulk/unlink', { userIds: batch });
        results.push({ success: true, count: batch.length });
      } catch (error) {
        results.push({ success: false, error: error.response?.data });
      }

      if (i + batchSize < userIds.length) {
        await this._sleep(options.delay || 1000);
      }
    }

    return results;
  }

  // ─── Alias Methods ────────────────────────────────────────────

  async createAlias(aliasId, richMenuId) {
    await this._request('POST', '/richmenu/alias', {
      richMenuAliasId: aliasId,
      richMenuId
    });
    return true;
  }

  async updateAlias(aliasId, richMenuId) {
    await this._request('POST', `/richmenu/alias/${aliasId}`, { richMenuId });
    return true;
  }

  async getAlias(aliasId) {
    return await this._request('GET', `/richmenu/alias/${aliasId}`);
  }

  async deleteAlias(aliasId) {
    await this._request('DELETE', `/richmenu/alias/${aliasId}`);
    return true;
  }

  async listAliases() {
    const data = await this._request('GET', '/richmenu/alias/list');
    return data.aliases || [];
  }

  async upsertAlias(aliasId, richMenuId) {
    try {
      await this.createAlias(aliasId, richMenuId);
    } catch (error) {
      if (error.response?.status === 400) {
        await this.updateAlias(aliasId, richMenuId);
      } else {
        throw error;
      }
    }
    return true;
  }

  // ─── High-level Methods ────────────────────────────────────────────

  // สร้าง Rich Menu พร้อมรูปภาพในขั้นตอนเดียว
  async createWithImage(richMenuConfig, imagePath, options = {}) {
    const richMenuId = await this.create(richMenuConfig);

    try {
      await this.uploadImage(richMenuId, imagePath, options.imageOptions);
    } catch (error) {
      // ลบ Rich Menu ที่สร้างไปแล้วหากอัพโหลดรูปล้มเหลว
      await this.delete(richMenuId).catch(() => {});
      throw error;
    }

    return richMenuId;
  }

  // สร้างหรืออัพเดต Rich Menu (idempotent)
  async upsert(name, richMenuConfig, imagePath) {
    const existingMenus = await this.list();
    const existing = existingMenus.find(m => m.name === name);

    let richMenuId;
    if (existing) {
      // ลบเก่า สร้างใหม่
      await this.delete(existing.richMenuId);
    }

    richMenuId = await this.createWithImage(richMenuConfig, imagePath);
    return richMenuId;
  }

  // ล้าง Rich Menu ทั้งหมด
  async deleteAll(options = {}) {
    const menus = await this.list();

    if (options.cancelDefault) {
      await this.cancelDefault().catch(() => {});
    }

    if (options.deleteAliases) {
      const aliases = await this.listAliases();
      await Promise.all(aliases.map(a => this.deleteAlias(a.richMenuAliasId).catch(() => {})));
    }

    const results = await Promise.all(
      menus.map(menu => this.delete(menu.richMenuId).then(() => ({ id: menu.richMenuId, success: true })).catch(err => ({ id: menu.richMenuId, success: false, error: err.message })))
    );

    this.cache.clear();
    return results;
  }

  // ─── Private Methods ────────────────────────────────────────────

  async _request(method, path, data) {
    try {
      const config = {
        method,
        url: `${this.baseUrl}${path}`,
        headers: this.defaultHeaders
      };

      if (data !== undefined) {
        config.data = data;
      }

      const response = await axios(config);
      return response.data;
    } catch (error) {
      const err = new Error(error.response?.data?.message || error.message);
      err.statusCode = error.response?.status;
      err.details = error.response?.data;
      throw err;
    }
  }

  _validateConfig(config) {
    if (!config.size || !config.size.width || !config.size.height) {
      throw new Error('Rich Menu must have size.width and size.height');
    }
    if (!config.areas || config.areas.length === 0) {
      throw new Error('Rich Menu must have at least one area');
    }
    if (config.areas.length > 20) {
      throw new Error('Rich Menu can have at most 20 areas');
    }
    if (config.chatBarText && config.chatBarText.length > 14) {
      throw new Error('chatBarText must be 14 characters or less');
    }
  }

  _sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

module.exports = RichMenuManager;
```

### ตัวอย่างการใช้ Manager Class

```javascript
const RichMenuManager = require('./rich-menu-manager');

async function main() {
  const manager = new RichMenuManager();

  // สร้าง Rich Menu พร้อมรูป
  const richMenuId = await manager.createWithImage(
    {
      size: { width: 2500, height: 1686 },
      selected: true,
      name: 'main-v2',
      chatBarText: 'เมนูหลัก',
      areas: [
        {
          bounds: { x: 0, y: 0, width: 833, height: 843 },
          action: { type: 'message', label: 'ช้อป', text: 'ดูสินค้า' }
        },
        {
          bounds: { x: 833, y: 0, width: 833, height: 843 },
          action: { type: 'postback', label: 'ตะกร้า', data: 'action=cart' }
        },
        {
          bounds: { x: 1666, y: 0, width: 834, height: 843 },
          action: { type: 'postback', label: 'บัญชี', data: 'action=account' }
        },
        {
          bounds: { x: 0, y: 843, width: 1250, height: 843 },
          action: { type: 'uri', label: 'โปรโมชัน', uri: 'https://example.com/promo' }
        },
        {
          bounds: { x: 1250, y: 843, width: 1250, height: 843 },
          action: { type: 'message', label: 'ติดต่อ', text: 'ติดต่อเรา' }
        }
      ]
    },
    './images/main-menu.jpg',
    { imageOptions: { resize: { width: 2500, height: 1686 } } }
  );

  // ตั้งค่าเป็น Default
  await manager.setDefault(richMenuId);

  // สร้าง Alias
  await manager.upsertAlias('richmenu-alias-main', richMenuId);

  console.log('Rich Menu setup complete:', richMenuId);

  // Bulk Link ให้ VIP Members
  const vipUserIds = ['U123...', 'U456...', 'U789...'];
  const result = await manager.bulkLink(vipUserIds, richMenuId, {
    onProgress: ({ processed, total, percentage }) => {
      console.log(`Progress: ${processed}/${total} (${percentage}%)`);
    }
  });

  console.log('Bulk link result:', result);
}

main().catch(console.error);
```

---

## 16. Python Rich Menu Manager

```python
# rich_menu_manager.py
import os
import json
import time
from typing import Optional, List, Dict, Any
import requests
from pathlib import Path

class RichMenuManager:
    """Python Rich Menu Manager for LINE Messaging API"""

    BASE_URL = "https://api.line.me/v2/bot"

    def __init__(self, channel_access_token: Optional[str] = None):
        self.token = channel_access_token or os.getenv('CHANNEL_ACCESS_TOKEN')
        if not self.token:
            raise ValueError("Channel access token is required")

        self.headers = {
            'Authorization': f'Bearer {self.token}',
            'Content-Type': 'application/json'
        }
        self._cache = {}

    # ─── Core Methods ────────────────────────────────────────────

    def create(self, config: Dict[str, Any]) -> str:
        """สร้าง Rich Menu ใหม่"""
        self._validate_config(config)
        response = self._request('POST', '/richmenu', json=config)
        rich_menu_id = response['richMenuId']
        self._cache[f'menu:{rich_menu_id}'] = config
        return rich_menu_id

    def get(self, rich_menu_id: str) -> Dict[str, Any]:
        """ดูข้อมูล Rich Menu"""
        cache_key = f'menu:{rich_menu_id}'
        if cache_key in self._cache:
            return self._cache[cache_key]

        data = self._request('GET', f'/richmenu/{rich_menu_id}')
        self._cache[cache_key] = data
        return data

    def delete(self, rich_menu_id: str) -> bool:
        """ลบ Rich Menu"""
        self._request('DELETE', f'/richmenu/{rich_menu_id}')
        self._cache.pop(f'menu:{rich_menu_id}', None)
        return True

    def list_menus(self) -> List[Dict[str, Any]]:
        """ดูรายการ Rich Menu ทั้งหมด"""
        data = self._request('GET', '/richmenu/list')
        return data.get('richmenus', [])

    # ─── Image Methods ────────────────────────────────────────────

    def upload_image(self, rich_menu_id: str, image_path: str,
                     mime_type: Optional[str] = None) -> Dict[str, Any]:
        """อัพโหลดรูปภาพ Rich Menu"""
        image_path = Path(image_path)

        if not image_path.exists():
            raise FileNotFoundError(f"Image not found: {image_path}")

        if mime_type is None:
            mime_type = 'image/png' if image_path.suffix.lower() == '.png' else 'image/jpeg'

        with open(image_path, 'rb') as f:
            image_data = f.read()

        # ตรวจสอบขนาดไฟล์
        file_size_kb = len(image_data) / 1024
        if file_size_kb > 1024:
            raise ValueError(f"Image too large: {file_size_kb:.0f}KB (max 1024KB)")

        response = requests.post(
            f'{self.BASE_URL}/richmenu/{rich_menu_id}/content',
            data=image_data,
            headers={
                'Authorization': f'Bearer {self.token}',
                'Content-Type': mime_type
            }
        )
        response.raise_for_status()
        return {'richMenuId': rich_menu_id, 'fileSizeKB': f'{file_size_kb:.0f}'}

    # ─── Default Methods ────────────────────────────────────────────

    def set_default(self, rich_menu_id: str) -> bool:
        """ตั้งค่า Default Rich Menu"""
        self._request('POST', f'/user/all/richmenu/{rich_menu_id}', json={})
        return True

    def get_default(self) -> Optional[Dict[str, Any]]:
        """ดู Default Rich Menu"""
        try:
            return self._request('GET', '/user/all/richmenu')
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 404:
                return None
            raise

    def cancel_default(self) -> bool:
        """ยกเลิก Default Rich Menu"""
        self._request('DELETE', '/user/all/richmenu')
        return True

    # ─── User Link Methods ────────────────────────────────────────────

    def link_to_user(self, user_id: str, rich_menu_id: str) -> bool:
        """Link Rich Menu ให้ User"""
        self._request('POST', f'/user/{user_id}/richmenu/{rich_menu_id}', json={})
        return True

    def unlink_from_user(self, user_id: str) -> bool:
        """Unlink Rich Menu จาก User"""
        self._request('DELETE', f'/user/{user_id}/richmenu')
        return True

    def get_user_menu(self, user_id: str) -> Optional[Dict[str, Any]]:
        """ดู Rich Menu ของ User"""
        try:
            return self._request('GET', f'/user/{user_id}/richmenu')
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 404:
                return None
            raise

    # ─── Bulk Methods ────────────────────────────────────────────

    def bulk_link(self, user_ids: List[str], rich_menu_id: str,
                  batch_size: int = 500, delay: float = 1.0) -> Dict[str, Any]:
        """Bulk Link Rich Menu ให้หลาย User"""
        results = []

        for i in range(0, len(user_ids), batch_size):
            batch = user_ids[i:i + batch_size]
            try:
                self._request('POST', '/richmenu/bulk/link', json={
                    'richMenuId': rich_menu_id,
                    'userIds': batch
                })
                results.append({'success': True, 'count': len(batch)})
                print(f"Linked {len(batch)} users (batch {i//batch_size + 1})")
            except Exception as e:
                results.append({'success': False, 'error': str(e), 'count': len(batch)})
                print(f"Batch {i//batch_size + 1} failed: {e}")

            if i + batch_size < len(user_ids) and delay > 0:
                time.sleep(delay)

        return {
            'total': len(user_ids),
            'success': sum(r['count'] for r in results if r['success']),
            'failed': sum(r['count'] for r in results if not r['success']),
            'batches': results
        }

    def bulk_unlink(self, user_ids: List[str],
                    batch_size: int = 500, delay: float = 1.0) -> List[Dict]:
        """Bulk Unlink Rich Menu จากหลาย User"""
        results = []
        for i in range(0, len(user_ids), batch_size):
            batch = user_ids[i:i + batch_size]
            try:
                self._request('POST', '/richmenu/bulk/unlink', json={'userIds': batch})
                results.append({'success': True, 'count': len(batch)})
            except Exception as e:
                results.append({'success': False, 'error': str(e)})
            if i + batch_size < len(user_ids):
                time.sleep(delay)
        return results

    # ─── Alias Methods ────────────────────────────────────────────

    def create_alias(self, alias_id: str, rich_menu_id: str) -> bool:
        """สร้าง Rich Menu Alias"""
        self._request('POST', '/richmenu/alias', json={
            'richMenuAliasId': alias_id,
            'richMenuId': rich_menu_id
        })
        return True

    def update_alias(self, alias_id: str, rich_menu_id: str) -> bool:
        """อัพเดต Rich Menu Alias"""
        self._request('POST', f'/richmenu/alias/{alias_id}', json={
            'richMenuId': rich_menu_id
        })
        return True

    def upsert_alias(self, alias_id: str, rich_menu_id: str) -> bool:
        """สร้างหรืออัพเดต Alias"""
        try:
            return self.create_alias(alias_id, rich_menu_id)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 400:
                return self.update_alias(alias_id, rich_menu_id)
            raise

    def get_alias(self, alias_id: str) -> Dict[str, Any]:
        """ดูข้อมูล Alias"""
        return self._request('GET', f'/richmenu/alias/{alias_id}')

    def delete_alias(self, alias_id: str) -> bool:
        """ลบ Alias"""
        self._request('DELETE', f'/richmenu/alias/{alias_id}')
        return True

    def list_aliases(self) -> List[Dict[str, Any]]:
        """ดูรายการ Alias ทั้งหมด"""
        data = self._request('GET', '/richmenu/alias/list')
        return data.get('aliases', [])

    # ─── High-level Methods ────────────────────────────────────────────

    def create_with_image(self, config: Dict[str, Any], image_path: str) -> str:
        """สร้าง Rich Menu พร้อมอัพโหลดรูปในขั้นตอนเดียว"""
        rich_menu_id = self.create(config)
        try:
            self.upload_image(rich_menu_id, image_path)
        except Exception as e:
            self.delete(rich_menu_id)
            raise RuntimeError(f"Failed to upload image: {e}") from e
        return rich_menu_id

    def delete_all(self, cancel_default: bool = True,
                   delete_aliases: bool = True) -> Dict[str, Any]:
        """ลบ Rich Menu ทั้งหมด"""
        if cancel_default:
            try:
                self.cancel_default()
            except Exception:
                pass

        if delete_aliases:
            for alias in self.list_aliases():
                try:
                    self.delete_alias(alias['richMenuAliasId'])
                except Exception:
                    pass

        menus = self.list_menus()
        results = []
        for menu in menus:
            try:
                self.delete(menu['richMenuId'])
                results.append({'id': menu['richMenuId'], 'success': True})
            except Exception as e:
                results.append({'id': menu['richMenuId'], 'success': False, 'error': str(e)})

        self._cache.clear()
        return {
            'deleted': sum(1 for r in results if r['success']),
            'failed': sum(1 for r in results if not r['success']),
            'results': results
        }

    # ─── Private Methods ────────────────────────────────────────────

    def _request(self, method: str, path: str, **kwargs) -> Any:
        """HTTP Request helper"""
        url = f'{self.BASE_URL}{path}'
        if 'json' in kwargs:
            headers = self.headers
        else:
            headers = self.headers

        response = requests.request(method, url, headers=headers, **kwargs)

        try:
            response.raise_for_status()
        except requests.exceptions.HTTPError as e:
            error_data = {}
            try:
                error_data = response.json()
            except Exception:
                pass
            raise requests.exceptions.HTTPError(
                f"HTTP {response.status_code}: {error_data.get('message', response.text)}",
                response=response
            ) from e

        if response.content:
            return response.json()
        return {}

    def _validate_config(self, config: Dict[str, Any]) -> None:
        """ตรวจสอบ config ก่อนสร้าง"""
        if 'size' not in config:
            raise ValueError("Config must have 'size'")
        if 'areas' not in config or len(config['areas']) == 0:
            raise ValueError("Config must have at least one 'area'")
        if len(config['areas']) > 20:
            raise ValueError("Config can have at most 20 areas")
        if 'chatBarText' in config and len(config['chatBarText']) > 14:
            raise ValueError("chatBarText must be 14 characters or less")


# ─── ตัวอย่างการใช้งาน ────────────────────────────────────────────

if __name__ == '__main__':
    manager = RichMenuManager()

    # สร้าง Rich Menu config
    config = {
        'size': {'width': 2500, 'height': 1686},
        'selected': True,
        'name': 'main-menu-py',
        'chatBarText': 'เมนูหลัก',
        'areas': [
            {
                'bounds': {'x': 0, 'y': 0, 'width': 833, 'height': 843},
                'action': {'type': 'message', 'label': 'ช้อปปิ้ง', 'text': 'ดูสินค้า'}
            },
            {
                'bounds': {'x': 833, 'y': 0, 'width': 833, 'height': 843},
                'action': {'type': 'postback', 'label': 'ตะกร้า', 'data': 'action=cart'}
            },
            {
                'bounds': {'x': 1666, 'y': 0, 'width': 834, 'height': 843},
                'action': {'type': 'postback', 'label': 'บัญชี', 'data': 'action=account'}
            },
            {
                'bounds': {'x': 0, 'y': 843, 'width': 1250, 'height': 843},
                'action': {'type': 'uri', 'label': 'โปรโมชัน', 'uri': 'https://example.com/promo'}
            },
            {
                'bounds': {'x': 1250, 'y': 843, 'width': 1250, 'height': 843},
                'action': {'type': 'message', 'label': 'ติดต่อเรา', 'text': 'ติดต่อเรา'}
            }
        ]
    }

    # สร้างและอัพโหลดรูป
    rich_menu_id = manager.create_with_image(config, './images/main-menu.jpg')
    print(f'Created: {rich_menu_id}')

    # ตั้งค่า Default
    manager.set_default(rich_menu_id)
    print('Set as default')

    # สร้าง Alias
    manager.upsert_alias('richmenu-alias-main', rich_menu_id)
    print('Alias created')
```

---

## 17. Best Practices และ Security

### ข้อแนะนำที่สำคัญ

```javascript
// 1. เก็บ ID ใน Environment Variables
const RICH_MENU_IDS = {
  main: process.env.RICH_MENU_MAIN_ID,
  vip: process.env.RICH_MENU_VIP_ID,
  member: process.env.RICH_MENU_MEMBER_ID,
  guest: process.env.RICH_MENU_GUEST_ID
};

// 2. ใช้ Cache เพื่อลด API Calls
const menuCache = new Map();
const CACHE_TTL = 5 * 60 * 1000; // 5 นาที

async function getCachedRichMenu(richMenuId) {
  const cacheKey = `menu:${richMenuId}`;
  const cached = menuCache.get(cacheKey);

  if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
    return cached.data;
  }

  const data = await getRichMenuDetails(richMenuId);
  menuCache.set(cacheKey, { data, timestamp: Date.now() });
  return data;
}

// 3. Error Handling ที่ดี
async function safeSetDefaultRichMenu(richMenuId) {
  try {
    await setDefaultRichMenu(richMenuId);
    console.log(`Default set to: ${richMenuId}`);
    return { success: true };
  } catch (error) {
    const statusCode = error.response?.status;

    if (statusCode === 400) {
      console.error('Invalid rich menu ID or structure');
    } else if (statusCode === 401) {
      console.error('Invalid access token');
    } else if (statusCode === 404) {
      console.error('Rich menu not found');
    } else if (statusCode === 429) {
      console.error('Rate limit exceeded - retry later');
    }

    return { success: false, error: error.message, statusCode };
  }
}

// 4. Rate Limiting
const rateLimit = {
  requests: 0,
  resetTime: Date.now() + 60000,
  maxRequests: 2000 // ต่อนาที
};

async function rateLimitedRequest(fn) {
  if (Date.now() > rateLimit.resetTime) {
    rateLimit.requests = 0;
    rateLimit.resetTime = Date.now() + 60000;
  }

  if (rateLimit.requests >= rateLimit.maxRequests) {
    const waitTime = rateLimit.resetTime - Date.now();
    console.log(`Rate limit reached. Waiting ${waitTime}ms...`);
    await new Promise(resolve => setTimeout(resolve, waitTime));
  }

  rateLimit.requests++;
  return fn();
}

// 5. Retry Logic
async function retryRequest(fn, maxRetries = 3, delay = 1000) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries) throw error;

      const statusCode = error.response?.status;
      if (statusCode === 429 || statusCode >= 500) {
        const waitTime = delay * Math.pow(2, attempt - 1); // Exponential backoff
        console.log(`Attempt ${attempt} failed. Retrying in ${waitTime}ms...`);
        await new Promise(resolve => setTimeout(resolve, waitTime));
      } else {
        throw error; // ไม่ต้อง Retry สำหรับ 4xx errors อื่นๆ
      }
    }
  }
}
```

### Security Best Practices

```javascript
// 1. ตรวจสอบ Webhook Signature
const crypto = require('crypto');

function verifySignature(body, signature, channelSecret) {
  const hash = crypto
    .createHmac('SHA256', channelSecret)
    .update(body)
    .digest('base64');
  return signature === hash;
}

app.post('/webhook', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-line-signature'];
  if (!verifySignature(req.body, signature, process.env.CHANNEL_SECRET)) {
    return res.status(403).json({ error: 'Invalid signature' });
  }
  // Process events...
  res.sendStatus(200);
});

// 2. ตรวจสอบ postback data
function parsePostbackData(data) {
  const params = new URLSearchParams(data);
  const action = params.get('action');

  // Whitelist ของ actions ที่อนุญาต
  const allowedActions = ['view_cart', 'order_history', 'contact', 'promotions'];
  if (!allowedActions.includes(action)) {
    throw new Error(`Unknown action: ${action}`);
  }

  return Object.fromEntries(params.entries());
}
```

---

## สรุป

Rich Menu API ขั้นสูงช่วยให้เราสามารถสร้างประสบการณ์ที่ดีสำหรับผู้ใช้:

| ฟีเจอร์ | ประโยชน์ |
|---------|---------|
| Multiple Rich Menus | เมนูที่แตกต่างกันสำหรับผู้ใช้แต่ละกลุ่ม |
| Alias | จัดการ Menu โดยใช้ชื่อที่จำง่าย |
| Switching | ให้ผู้ใช้สลับเมนูได้ทันทีโดยไม่ต้องรอ |
| Bulk Link | อัพเดตเมนูให้ผู้ใช้จำนวนมากพร้อมกัน |
| Manager Class | จัดการ Menu อย่างมีระบบ |

---

*ส่วนถัดไป: [Part 52: LINE Login](part-052-line-login.md)*
