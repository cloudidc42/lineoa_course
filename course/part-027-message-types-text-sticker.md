# ส่วนที่ 27: ประเภทข้อความ - Text, Sticker, Location, Image, Video, Audio, File, Imagemap

---

## สารบัญ

1. [ภาพรวมประเภทข้อความ](#1-ภาพรวมประเภทข้อความ)
2. [Text Message พร้อม Emojis](#2-text-message-พร้อม-emojis)
3. [Sticker Message](#3-sticker-message)
4. [Location Message](#4-location-message)
5. [Image Message](#5-image-message)
6. [Video Message](#6-video-message)
7. [Audio Message](#7-audio-message)
8. [File Message](#8-file-message)
9. [Imagemap Message](#9-imagemap-message)
10. [Code Examples ใน Node.js](#10-code-examples-ใน-nodejs)
11. [Code Examples ใน Python](#11-code-examples-ใน-python)
12. [Best Practices](#12-best-practices)
13. [แบบฝึกหัดท้ายบท](#13-แบบฝึกหัดท้ายบท)

---

## 1. ภาพรวมประเภทข้อความ

LINE Messaging API รองรับข้อความหลายประเภท แต่ละประเภทมีโครงสร้าง JSON และข้อจำกัดที่แตกต่างกัน

### ตารางสรุปประเภทข้อความ

| ประเภท | ใช้ใน Reply/Push | ส่งจาก Webhook | รองรับ Reply |
|--------|----------------|---------------|-------------|
| text | ✅ | ✅ | ✅ |
| sticker | ✅ | ✅ | ✅ |
| location | ✅ | ✅ | ✅ |
| image | ✅ | ✅ (URL เท่านั้น) | ✅ |
| video | ✅ | ✅ (URL เท่านั้น) | ✅ |
| audio | ✅ | ✅ (URL เท่านั้น) | ✅ |
| file | ❌ (ผู้ใช้เท่านั้น) | ✅ | ❌ |
| imagemap | ✅ | ❌ | ✅ |
| template | ✅ | ❌ | ✅ |
| flex | ✅ | ❌ | ✅ |

---

## 2. Text Message พร้อม Emojis

### โครงสร้าง Text Message พื้นฐาน

```json
{
  "type": "text",
  "text": "สวัสดีครับ! ยินดีให้บริการ"
}
```

### Text Message พร้อม LINE Emojis

LINE มี Emoji พิเศษของตัวเองที่แตกต่างจาก Unicode emoji ทั่วไป

```json
{
  "type": "text",
  "text": "สวัสดี $ ยินดีต้อนรับสู่ร้านของเรา $",
  "emojis": [
    {
      "index": 7,
      "productId": "5ac1bfd5040ab15980c9b435",
      "emojiId": "001"
    },
    {
      "index": 30,
      "productId": "5ac1bfd5040ab15980c9b435",
      "emojiId": "002"
    }
  ]
}
```

**หมายเหตุ:** `$` เป็น placeholder สำหรับ emoji ใน text โดย `index` คือตำแหน่งของ `$` ใน string (นับจาก 0)

### การหา LINE Emoji Product IDs

```
ค้นหาได้จาก: https://developers.line.biz/en/docs/messaging-api/emoji-list/
```

### Emoji Package สำคัญ

| Package | productId | ตัวอย่าง |
|---------|-----------|---------|
| LINE Characters | 5ac1bfd5040ab15980c9b435 | 001-019 |
| LINE Stickers Vol.2 | 5ac21a18040ab15980c9b43e | หลากหลาย |
| LINE Characters & Friends | 5ac1bfd5040ab15980c9b435 | ตัวละคร LINE |

### Text ที่มี Newline และ Special Characters

```json
{
  "type": "text",
  "text": "📋 สรุปคำสั่งซื้อ\n━━━━━━━━━━━━━━━\n📦 สินค้า: เสื้อยืด XL\n💰 ราคา: 599 บาท\n🚚 สถานะ: กำลังจัดส่ง\n━━━━━━━━━━━━━━━\nขอบคุณที่ใช้บริการ!"
}
```

### ข้อจำกัด Text Message

| ข้อจำกัด | ค่า |
|---------|-----|
| ความยาวสูงสุด | 5,000 ตัวอักษร |
| จำนวน Emoji สูงสุด | 20 ตัวต่อข้อความ |
| รองรับ Unicode | ✅ |

---

## 3. Sticker Message

### โครงสร้าง Sticker Message

```json
{
  "type": "sticker",
  "packageId": "1",
  "stickerId": "1"
}
```

### Sticker Packages ที่รองรับ

LINE มี Sticker packages หลายชุด แต่ Bot สามารถส่งได้เฉพาะ packages ที่กำหนดไว้เท่านั้น

#### Sticker Packages สำหรับ API (แบบฟรี)

| Package ID | ชื่อ Package | Sticker IDs |
|-----------|-------------|------------|
| 1 | With Brown & Friends | 1-17 |
| 2 | Brown's Bear's Store | 180-307 |
| 3 | Cherry Hearts | 180-259 |
| 4 | LINE Characters | 300-399 |
| 11537 | BROWN & FRIENDS New Year 2019 | 52002734-52002773 |
| 11538 | Brown & Cony's Lovey Dovey | 51626494-51626533 |

#### ตัวอย่าง Sticker แต่ละ Package

```json
// Package 1 - With Brown
{
  "type": "sticker",
  "packageId": "1",
  "stickerId": "1"
}

// Package 1 - Happy
{
  "type": "sticker",
  "packageId": "1",
  "stickerId": "2"
}

// Package 1 - Sad
{
  "type": "sticker",
  "packageId": "1",
  "stickerId": "3"
}

// Package 2 - Bear's Store
{
  "type": "sticker",
  "packageId": "2",
  "stickerId": "180"
}
```

### การหา Sticker IDs ที่รองรับ

```
GET https://developers.line.biz/en/docs/messaging-api/sticker-list/
```

หรือตรวจสอบผ่าน API:
```
GET https://api.line.me/v2/bot/message/sticker/list/packages
```

### Sticker Message ที่รองรับการ Reply

```json
{
  "type": "sticker",
  "packageId": "446",
  "stickerId": "1988"
}
```

### ตัวอย่าง Sticker Packages ยอดนิยม

```javascript
const POPULAR_STICKERS = [
  // Happy
  { packageId: '1', stickerId: '1' },
  { packageId: '1', stickerId: '2' },
  // Love
  { packageId: '2', stickerId: '180' },
  // Wink
  { packageId: '3', stickerId: '240' },
  // Thumbs up
  { packageId: '4', stickerId: '305' },
  // OK
  { packageId: '4', stickerId: '306' },
  // Party
  { packageId: '11537', stickerId: '52002734' },
];

// ส่ง random sticker
function getRandomSticker() {
  const randomIndex = Math.floor(Math.random() * POPULAR_STICKERS.length);
  return {
    type: 'sticker',
    ...POPULAR_STICKERS[randomIndex]
  };
}
```

---

## 4. Location Message

### โครงสร้าง Location Message

```json
{
  "type": "location",
  "title": "ร้านอาหารของเรา",
  "address": "123 ถนนสุขุมวิท แขวงคลองเตย เขตคลองเตย กรุงเทพมหานคร 10110",
  "latitude": 13.7563,
  "longitude": 100.5018
}
```

### Parameters ของ Location Message

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| type | String | Yes | "location" |
| title | String | Yes | ชื่อสถานที่ (สูงสุด 100 ตัวอักษร) |
| address | String | Yes | ที่อยู่ |
| latitude | Number | Yes | ละติจูด (-90 ถึง 90) |
| longitude | Number | Yes | ลองจิจูด (-180 ถึง 180) |

### ตัวอย่าง Location Messages สำหรับธุรกิจ

```json
// สาขาหลัก
{
  "type": "location",
  "title": "สาขาใหญ่ CentralWorld",
  "address": "999/9 ถนนพระราม 1 แขวงปทุมวัน เขตปทุมวัน กรุงเทพฯ 10330",
  "latitude": 13.7467,
  "longitude": 100.5394
}

// โรงแรม
{
  "type": "location",
  "title": "โรงแรม Grand Hyatt Erawan",
  "address": "494 ถนนพลอยชมพู แขวงลุมพินี เขตปทุมวัน กรุงเทพฯ 10330",
  "latitude": 13.7431,
  "longitude": 100.5397
}
```

### การหาพิกัด GPS จากที่อยู่

```javascript
const axios = require('axios');

async function getCoordinates(address) {
  const API_KEY = process.env.GOOGLE_MAPS_API_KEY;
  const encodedAddress = encodeURIComponent(address);
  
  const response = await axios.get(
    `https://maps.googleapis.com/maps/api/geocode/json?address=${encodedAddress}&key=${API_KEY}`
  );
  
  if (response.data.status === 'OK') {
    const location = response.data.results[0].geometry.location;
    return {
      latitude: location.lat,
      longitude: location.lng
    };
  }
  
  throw new Error('ไม่พบพิกัดสำหรับที่อยู่นี้');
}

// ใช้งาน
async function sendLocationMessage(replyToken, address, title) {
  const coords = await getCoordinates(address);
  
  return client.replyMessage({
    replyToken,
    messages: [{
      type: 'location',
      title,
      address,
      latitude: coords.latitude,
      longitude: coords.longitude
    }]
  });
}
```

---

## 5. Image Message

### โครงสร้าง Image Message

```json
{
  "type": "image",
  "originalContentUrl": "https://example.com/original.jpg",
  "previewImageUrl": "https://example.com/preview.jpg"
}
```

### Parameters ของ Image Message

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| type | String | Yes | "image" |
| originalContentUrl | String | Yes | URL รูปต้นฉบับ |
| previewImageUrl | String | Yes | URL รูป preview |

### ข้อจำกัด Image Message

| ข้อจำกัด | ค่า |
|---------|-----|
| ขนาดไฟล์สูงสุด | 10 MB |
| รูปแบบที่รองรับ | JPEG, PNG |
| ความละเอียดแนะนำ | สูงสุด 4096x4096 pixels |
| URL | ต้อง HTTPS |

### ตัวอย่าง Image Message ที่สมบูรณ์

```json
{
  "type": "image",
  "originalContentUrl": "https://cdn.example.com/products/shirt-xl.jpg",
  "previewImageUrl": "https://cdn.example.com/products/shirt-xl-thumb.jpg"
}
```

### การอัปโหลดและส่งรูปภาพ

```javascript
const axios = require('axios');
const FormData = require('form-data');
const fs = require('fs');

/**
 * อัปโหลดรูปภาพไปยัง LINE (Content API)
 * หมายเหตุ: ฟีเจอร์นี้ใช้ได้กับ Message ที่ผู้ใช้ส่งมาเท่านั้น
 */
async function downloadImageFromMessage(messageId) {
  const response = await axios.get(
    `https://api-data.line.me/v2/bot/message/${messageId}/content`,
    {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`
      },
      responseType: 'arraybuffer'
    }
  );
  
  return response.data;
}

/**
 * ส่ง Image Message โดยใช้ URL ที่เป็น public
 */
async function sendImageMessage(replyToken, imageUrl, previewUrl) {
  return client.replyMessage({
    replyToken,
    messages: [{
      type: 'image',
      originalContentUrl: imageUrl,
      previewImageUrl: previewUrl || imageUrl
    }]
  });
}

/**
 * ส่งรูปภาพหลายรูป
 */
async function sendMultipleImages(replyToken, images) {
  const messages = images.slice(0, 5).map(img => ({
    type: 'image',
    originalContentUrl: img.url,
    previewImageUrl: img.preview || img.url
  }));
  
  return client.replyMessage({ replyToken, messages });
}
```

### Image Message ใน Context ต่างๆ

```json
// Product showcase
{
  "type": "image",
  "originalContentUrl": "https://shop.example.com/products/watch-001.jpg",
  "previewImageUrl": "https://shop.example.com/products/watch-001-thumb.jpg"
}

// Promotional banner
{
  "type": "image",
  "originalContentUrl": "https://cdn.example.com/banners/sale-2024.jpg",
  "previewImageUrl": "https://cdn.example.com/banners/sale-2024-preview.jpg"
}

// Receipt/QR Code
{
  "type": "image",
  "originalContentUrl": "https://api.example.com/receipts/12345.png",
  "previewImageUrl": "https://api.example.com/receipts/12345-thumb.png"
}
```

---

## 6. Video Message

### โครงสร้าง Video Message

```json
{
  "type": "video",
  "originalContentUrl": "https://example.com/video.mp4",
  "previewImageUrl": "https://example.com/video-preview.jpg",
  "trackingId": "track001"
}
```

### Parameters ของ Video Message

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| type | String | Yes | "video" |
| originalContentUrl | String | Yes | URL วิดีโอ |
| previewImageUrl | String | Yes | URL รูป thumbnail |
| trackingId | String | No | ID สำหรับ tracking การดู |

### ข้อจำกัด Video Message

| ข้อจำกัด | ค่า |
|---------|-----|
| ขนาดไฟล์สูงสุด | 200 MB |
| รูปแบบ | MP4 |
| ระยะเวลาสูงสุด | 60 วินาที |
| URL | ต้อง HTTPS |

### ตัวอย่าง Video Message

```javascript
// ส่ง Product demo video
async function sendProductVideo(replyToken, productId) {
  const product = await getProduct(productId);
  
  return client.replyMessage({
    replyToken,
    messages: [
      {
        type: 'text',
        text: `🎥 วิดีโอสาธิต: ${product.name}`
      },
      {
        type: 'video',
        originalContentUrl: product.videoUrl,
        previewImageUrl: product.thumbnailUrl,
        trackingId: `product_${productId}`
      }
    ]
  });
}
```

### Video แบบ External Link

```json
{
  "type": "video",
  "originalContentUrl": "https://cdn.example.com/videos/tutorial.mp4",
  "previewImageUrl": "https://cdn.example.com/videos/tutorial-thumb.jpg"
}
```

---

## 7. Audio Message

### โครงสร้าง Audio Message

```json
{
  "type": "audio",
  "originalContentUrl": "https://example.com/audio.m4a",
  "duration": 60000
}
```

### Parameters ของ Audio Message

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| type | String | Yes | "audio" |
| originalContentUrl | String | Yes | URL ไฟล์เสียง |
| duration | Number | Yes | ระยะเวลา (มิลลิวินาที) |

### ข้อจำกัด Audio Message

| ข้อจำกัด | ค่า |
|---------|-----|
| ขนาดไฟล์สูงสุด | 200 MB |
| รูปแบบ | M4A |
| URL | ต้อง HTTPS |

### ตัวอย่าง Audio Message

```json
{
  "type": "audio",
  "originalContentUrl": "https://cdn.example.com/audio/greeting.m4a",
  "duration": 5000
}
```

### Use Cases ของ Audio Message

1. **ข้อความเสียงสำหรับลูกค้า VIP** - บันทึกข้อความต้อนรับพิเศษ
2. **Podcast snippets** - แชร์เสียงบางส่วน
3. **Voice notifications** - แจ้งเตือนด้วยเสียง

```javascript
// ส่ง Audio greeting
async function sendAudioGreeting(replyToken, userName) {
  return client.replyMessage({
    replyToken,
    messages: [
      {
        type: 'text',
        text: `สวัสดีคุณ ${userName}! มีข้อความเสียงจากเรา:`
      },
      {
        type: 'audio',
        originalContentUrl: 'https://cdn.example.com/audio/welcome.m4a',
        duration: 8000
      }
    ]
  });
}
```

---

## 8. File Message

### ข้อสำคัญเกี่ยวกับ File Message

**File Message สามารถส่งได้เฉพาะผู้ใช้ส่งมาหา Bot เท่านั้น** Bot ไม่สามารถส่ง File Message หาผู้ใช้ได้ แต่ Bot สามารถดาวน์โหลดไฟล์ที่ผู้ใช้ส่งมาได้

### รับไฟล์จากผู้ใช้

```json
// Webhook event เมื่อผู้ใช้ส่งไฟล์
{
  "type": "message",
  "message": {
    "type": "file",
    "id": "325708",
    "fileName": "document.pdf",
    "fileSize": 2138
  },
  "replyToken": "nHuyWiB7...",
  "source": {
    "type": "user",
    "userId": "U4af4980..."
  }
}
```

### ดาวน์โหลดไฟล์ที่ผู้ใช้ส่งมา

```javascript
const axios = require('axios');
const fs = require('fs');
const path = require('path');

/**
 * ดาวน์โหลดไฟล์จากผู้ใช้
 */
async function downloadUserFile(messageId, fileName, saveDir = './downloads') {
  // สร้าง directory ถ้ายังไม่มี
  if (!fs.existsSync(saveDir)) {
    fs.mkdirSync(saveDir, { recursive: true });
  }
  
  const response = await axios.get(
    `https://api-data.line.me/v2/bot/message/${messageId}/content`,
    {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`
      },
      responseType: 'arraybuffer'
    }
  );
  
  const savePath = path.join(saveDir, fileName);
  fs.writeFileSync(savePath, response.data);
  
  return savePath;
}

/**
 * จัดการเมื่อผู้ใช้ส่งไฟล์
 */
async function handleFileMessage(event) {
  const { replyToken, message } = event;
  const { id, fileName, fileSize } = message;
  
  try {
    const savedPath = await downloadUserFile(id, fileName);
    
    const fileSizeKB = (fileSize / 1024).toFixed(1);
    
    await client.replyMessage({
      replyToken,
      messages: [{
        type: 'text',
        text: `✅ ได้รับไฟล์แล้ว!\n\n📄 ชื่อไฟล์: ${fileName}\n📊 ขนาด: ${fileSizeKB} KB\n\nกำลังประมวลผล...`
      }]
    });
    
    // ประมวลผลไฟล์ตามต้องการ
    await processFile(savedPath, fileName);
    
  } catch (error) {
    console.error('Error handling file:', error);
    
    await client.replyMessage({
      replyToken,
      messages: [{
        type: 'text',
        text: '❌ เกิดข้อผิดพลาดในการรับไฟล์ กรุณาลองใหม่อีกครั้ง'
      }]
    });
  }
}

/**
 * ประมวลผลไฟล์ (ตัวอย่าง)
 */
async function processFile(filePath, fileName) {
  const ext = path.extname(fileName).toLowerCase();
  
  switch (ext) {
    case '.pdf':
      await processPDF(filePath);
      break;
    case '.xlsx':
    case '.xls':
      await processExcel(filePath);
      break;
    case '.csv':
      await processCSV(filePath);
      break;
    default:
      console.log('Unsupported file type:', ext);
  }
}
```

---

## 9. Imagemap Message

Imagemap Message ช่วยให้สามารถสร้างรูปภาพที่มีพื้นที่คลิกได้ (clickable areas) ซึ่งแต่ละพื้นที่สามารถกำหนด action ที่ต่างกันได้

### โครงสร้าง Imagemap Message

```json
{
  "type": "imagemap",
  "baseUrl": "https://example.com/imagemap",
  "altText": "แผนที่สินค้า",
  "baseSize": {
    "width": 1040,
    "height": 1040
  },
  "video": {
    "originalContentUrl": "https://example.com/video.mp4",
    "previewImageUrl": "https://example.com/preview.jpg",
    "area": {
      "x": 0,
      "y": 0,
      "width": 520,
      "height": 520
    },
    "externalLink": {
      "linkUri": "https://example.com",
      "label": "ดูเพิ่มเติม"
    }
  },
  "actions": [
    {
      "type": "uri",
      "linkUri": "https://example.com/product/1",
      "area": {
        "x": 0,
        "y": 0,
        "width": 520,
        "height": 520
      }
    },
    {
      "type": "uri",
      "linkUri": "https://example.com/product/2",
      "area": {
        "x": 520,
        "y": 0,
        "width": 520,
        "height": 520
      }
    },
    {
      "type": "message",
      "text": "สนใจสินค้าในส่วนล่างซ้าย",
      "area": {
        "x": 0,
        "y": 520,
        "width": 520,
        "height": 520
      }
    },
    {
      "type": "postback",
      "data": "action=view&item=4",
      "area": {
        "x": 520,
        "y": 520,
        "width": 520,
        "height": 520
      }
    }
  ]
}
```

### Parameters ของ Imagemap Message

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| type | String | Yes | "imagemap" |
| baseUrl | String | Yes | Base URL ของรูปภาพ |
| altText | String | Yes | ข้อความ alt |
| baseSize | Object | Yes | ขนาดฐาน (width x height) |
| video | Object | No | วิดีโอที่แนบ |
| actions | Array | Yes | Actions สำหรับแต่ละพื้นที่ |

### Action Types ใน Imagemap

| Type | คำอธิบาย |
|------|---------|
| uri | เปิด URL |
| message | ส่งข้อความ |
| postback | ส่ง postback |

### URL ของ Imagemap

LINE จะดึงรูปภาพจาก URL ที่กำหนดโดยต่อท้ายด้วยขนาดรูปภาพ:
- `${baseUrl}/1040` - สำหรับ non-Retina
- `${baseUrl}/2080` - สำหรับ Retina (2x)

ดังนั้น server ของคุณต้องตอบสนองที่:
- `https://example.com/imagemap/1040`
- `https://example.com/imagemap/2080`

### ตัวอย่าง Imagemap สำหรับ Product Grid

```javascript
/**
 * สร้าง Imagemap Message สำหรับ Product Grid 2x2
 */
function createProductGridImagemap(products, baseUrl) {
  const actions = products.slice(0, 4).map((product, index) => {
    const col = index % 2;
    const row = Math.floor(index / 2);
    
    return {
      type: 'uri',
      linkUri: `https://shop.example.com/products/${product.id}`,
      area: {
        x: col * 520,
        y: row * 520,
        width: 520,
        height: 520
      }
    };
  });
  
  return {
    type: 'imagemap',
    baseUrl,
    altText: 'สินค้าแนะนำ',
    baseSize: {
      width: 1040,
      height: 1040
    },
    actions
  };
}

// ใช้งาน
const products = [
  { id: 1, name: 'เสื้อยืด' },
  { id: 2, name: 'กางเกง' },
  { id: 3, name: 'รองเท้า' },
  { id: 4, name: 'กระเป๋า' }
];

const imagemap = createProductGridImagemap(
  products,
  'https://cdn.example.com/imagemaps/products-grid'
);

await client.replyMessage({
  replyToken,
  messages: [imagemap]
});
```

### Imagemap Server ที่ต้องสร้าง

```javascript
// Express server สำหรับ serve imagemap images
app.get('/imagemap/:name/:size', async (req, res) => {
  const { name, size } = req.params;
  const validSizes = ['240', '300', '460', '700', '1040', '2080'];
  
  if (!validSizes.includes(size)) {
    return res.status(400).json({ error: 'Invalid size' });
  }
  
  try {
    // ดึงรูปภาพตามขนาดที่ต้องการ
    const imagePath = `./public/imagemaps/${name}/${size}.jpg`;
    
    if (!fs.existsSync(imagePath)) {
      // Resize รูปภาพถ้ายังไม่มี
      await resizeImage(`./public/imagemaps/${name}/original.jpg`, imagePath, parseInt(size));
    }
    
    res.sendFile(path.resolve(imagePath));
  } catch (error) {
    res.status(500).json({ error: 'Image not found' });
  }
});

async function resizeImage(inputPath, outputPath, width) {
  const sharp = require('sharp');
  await sharp(inputPath)
    .resize(width)
    .jpeg({ quality: 85 })
    .toFile(outputPath);
}
```

---

## 10. Code Examples ใน Node.js

### helpers/messageBuilder.js - สร้าง Message Objects

```javascript
'use strict';

/**
 * สร้าง Text Message
 */
function textMessage(text, emojis = []) {
  const msg = { type: 'text', text };
  if (emojis.length > 0) {
    msg.emojis = emojis;
  }
  return msg;
}

/**
 * สร้าง Sticker Message
 */
function stickerMessage(packageId, stickerId) {
  return {
    type: 'sticker',
    packageId: String(packageId),
    stickerId: String(stickerId)
  };
}

/**
 * สร้าง Location Message
 */
function locationMessage(title, address, latitude, longitude) {
  return {
    type: 'location',
    title,
    address,
    latitude,
    longitude
  };
}

/**
 * สร้าง Image Message
 */
function imageMessage(originalUrl, previewUrl) {
  return {
    type: 'image',
    originalContentUrl: originalUrl,
    previewImageUrl: previewUrl || originalUrl
  };
}

/**
 * สร้าง Video Message
 */
function videoMessage(videoUrl, thumbnailUrl, trackingId) {
  const msg = {
    type: 'video',
    originalContentUrl: videoUrl,
    previewImageUrl: thumbnailUrl
  };
  
  if (trackingId) {
    msg.trackingId = trackingId;
  }
  
  return msg;
}

/**
 * สร้าง Audio Message
 */
function audioMessage(audioUrl, durationMs) {
  return {
    type: 'audio',
    originalContentUrl: audioUrl,
    duration: durationMs
  };
}

/**
 * สร้าง Imagemap Message
 */
function imagemapMessage(baseUrl, altText, width, height, actions, video = null) {
  const msg = {
    type: 'imagemap',
    baseUrl,
    altText,
    baseSize: { width, height },
    actions
  };
  
  if (video) {
    msg.video = video;
  }
  
  return msg;
}

/**
 * สร้าง URI Action สำหรับ Imagemap
 */
function imagemapUriAction(uri, x, y, width, height) {
  return {
    type: 'uri',
    linkUri: uri,
    area: { x, y, width, height }
  };
}

/**
 * สร้าง Message Action สำหรับ Imagemap
 */
function imagemapMessageAction(text, x, y, width, height) {
  return {
    type: 'message',
    text,
    area: { x, y, width, height }
  };
}

module.exports = {
  textMessage,
  stickerMessage,
  locationMessage,
  imageMessage,
  videoMessage,
  audioMessage,
  imagemapMessage,
  imagemapUriAction,
  imagemapMessageAction
};
```

### handlers/messageHandler.js - Handler ครบทุกประเภท

```javascript
'use strict';

const {
  textMessage,
  stickerMessage,
  locationMessage,
  imageMessage,
  videoMessage
} = require('../helpers/messageBuilder');

/**
 * จัดการ Message Events ทุกประเภท
 */
async function handleMessageEvent(event, client) {
  const { replyToken, message, source } = event;
  
  switch (message.type) {
    case 'text':
      return handleTextMessage(event, client);
    
    case 'image':
      return handleIncomingImage(event, client);
    
    case 'video':
      return handleIncomingVideo(event, client);
    
    case 'audio':
      return handleIncomingAudio(event, client);
    
    case 'file':
      return handleIncomingFile(event, client);
    
    case 'location':
      return handleIncomingLocation(event, client);
    
    case 'sticker':
      return handleIncomingSticker(event, client);
    
    default:
      return client.replyMessage({
        replyToken,
        messages: [textMessage('ขอโทษครับ ไม่รองรับประเภทข้อความนี้')]
      });
  }
}

/**
 * จัดการ Text Message
 */
async function handleTextMessage(event, client) {
  const { replyToken, message } = event;
  const text = message.text.trim();
  
  // Commands
  const commands = {
    'รูปภาพ': () => sendSampleImage(replyToken, client),
    'วิดีโอ': () => sendSampleVideo(replyToken, client),
    'สติกเกอร์': () => sendSampleSticker(replyToken, client),
    'แผนที่': () => sendSampleLocation(replyToken, client),
    'เสียง': () => sendSampleAudio(replyToken, client),
    'imagemap': () => sendSampleImagemap(replyToken, client),
  };
  
  const handler = commands[text.toLowerCase()];
  if (handler) return handler();
  
  // Default response
  return client.replyMessage({
    replyToken,
    messages: [
      textMessage(`📨 ได้รับข้อความ: "${text}"`),
      textMessage('พิมพ์: รูปภาพ, วิดีโอ, สติกเกอร์, แผนที่, เสียง, imagemap')
    ]
  });
}

/**
 * ส่ง Sample Image
 */
async function sendSampleImage(replyToken, client) {
  return client.replyMessage({
    replyToken,
    messages: [
      textMessage('🖼️ ตัวอย่างรูปภาพ:'),
      imageMessage(
        'https://picsum.photos/id/237/1040/1040',
        'https://picsum.photos/id/237/400/400'
      )
    ]
  });
}

/**
 * ส่ง Sample Video
 */
async function sendSampleVideo(replyToken, client) {
  return client.replyMessage({
    replyToken,
    messages: [
      textMessage('🎥 ตัวอย่างวิดีโอ:'),
      videoMessage(
        'https://sample-videos.com/video321/mp4/720/big_buck_bunny_720p_1mb.mp4',
        'https://picsum.photos/id/100/1040/585',
        'sample_video_001'
      )
    ]
  });
}

/**
 * ส่ง Sample Sticker
 */
async function sendSampleSticker(replyToken, client) {
  return client.replyMessage({
    replyToken,
    messages: [
      textMessage('😊 ตัวอย่าง Sticker:'),
      stickerMessage('1', '1'),
      stickerMessage('1', '2')
    ]
  });
}

/**
 * ส่ง Sample Location
 */
async function sendSampleLocation(replyToken, client) {
  return client.replyMessage({
    replyToken,
    messages: [
      textMessage('📍 ที่ตั้งของเรา:'),
      locationMessage(
        'สำนักงานใหญ่',
        '123 ถนนสุขุมวิท กรุงเทพฯ',
        13.7563,
        100.5018
      )
    ]
  });
}

/**
 * ส่ง Sample Audio
 */
async function sendSampleAudio(replyToken, client) {
  return client.replyMessage({
    replyToken,
    messages: [
      textMessage('🔊 ตัวอย่างไฟล์เสียง:'),
      {
        type: 'audio',
        originalContentUrl: 'https://www.soundjay.com/misc/sounds/bell-ringing-05.m4a',
        duration: 3000
      }
    ]
  });
}

/**
 * ส่ง Sample Imagemap
 */
async function sendSampleImagemap(replyToken, client) {
  return client.replyMessage({
    replyToken,
    messages: [
      textMessage('🗺️ ตัวอย่าง Imagemap (คลิกแต่ละส่วนได้):'),
      {
        type: 'imagemap',
        baseUrl: 'https://picsum.photos/id/1015',
        altText: 'Clickable Image',
        baseSize: { width: 1040, height: 1040 },
        actions: [
          {
            type: 'uri',
            linkUri: 'https://line.me',
            area: { x: 0, y: 0, width: 520, height: 520 }
          },
          {
            type: 'message',
            text: 'คลิกส่วนบนขวา',
            area: { x: 520, y: 0, width: 520, height: 520 }
          },
          {
            type: 'message',
            text: 'คลิกส่วนล่างซ้าย',
            area: { x: 0, y: 520, width: 520, height: 520 }
          },
          {
            type: 'message',
            text: 'คลิกส่วนล่างขวา',
            area: { x: 520, y: 520, width: 520, height: 520 }
          }
        ]
      }
    ]
  });
}

/**
 * จัดการรูปภาพที่ผู้ใช้ส่งมา
 */
async function handleIncomingImage(event, client) {
  const { replyToken, message } = event;
  
  return client.replyMessage({
    replyToken,
    messages: [
      textMessage('📸 ได้รับรูปภาพแล้ว!'),
      textMessage(`ID: ${message.id}\nกำลังประมวลผล...`)
    ]
  });
}

/**
 * จัดการวิดีโอที่ผู้ใช้ส่งมา
 */
async function handleIncomingVideo(event, client) {
  const { replyToken, message } = event;
  
  return client.replyMessage({
    replyToken,
    messages: [textMessage('🎥 ได้รับวิดีโอแล้ว! กำลังประมวลผล...')]
  });
}

/**
 * จัดการไฟล์เสียงที่ผู้ใช้ส่งมา
 */
async function handleIncomingAudio(event, client) {
  const { replyToken } = event;
  
  return client.replyMessage({
    replyToken,
    messages: [textMessage('🎵 ได้รับไฟล์เสียงแล้ว! กำลังประมวลผล...')]
  });
}

/**
 * จัดการไฟล์ที่ผู้ใช้ส่งมา
 */
async function handleIncomingFile(event, client) {
  const { replyToken, message } = event;
  
  return client.replyMessage({
    replyToken,
    messages: [
      textMessage(`📄 ได้รับไฟล์: ${message.fileName} (${message.fileSize} bytes)`)
    ]
  });
}

/**
 * จัดการ Location ที่ผู้ใช้ส่งมา
 */
async function handleIncomingLocation(event, client) {
  const { replyToken, message } = event;
  const { title, address, latitude, longitude } = message;
  
  return client.replyMessage({
    replyToken,
    messages: [
      textMessage(`📍 ได้รับตำแหน่ง:\n${title || 'ไม่มีชื่อ'}\n${address || ''}\n(${latitude}, ${longitude})`)
    ]
  });
}

/**
 * จัดการ Sticker ที่ผู้ใช้ส่งมา
 */
async function handleIncomingSticker(event, client) {
  const { replyToken, message } = event;
  
  // ตอบกลับด้วย sticker
  return client.replyMessage({
    replyToken,
    messages: [
      stickerMessage('1', '2')  // Happy sticker
    ]
  });
}

module.exports = {
  handleMessageEvent
};
```

---

## 11. Code Examples ใน Python

### utils/message_builder.py

```python
from typing import Optional, List, Dict, Any, Tuple
from linebot.v3.messaging import (
    TextMessage, StickerMessage, LocationMessage,
    ImageMessage, VideoMessage, AudioMessage,
    ImagemapMessage, ImagemapBaseSize,
    URIImagemapAction, MessageImagemapAction, PostbackImagemapAction,
    ImagemapArea, ImagemapExternalLink, ImagemapVideo
)


def text_message(text: str, emojis: Optional[List[Dict]] = None) -> TextMessage:
    """สร้าง Text Message"""
    msg = TextMessage(type='text', text=text)
    if emojis:
        msg.emojis = emojis
    return msg


def sticker_message(package_id: str, sticker_id: str) -> StickerMessage:
    """สร้าง Sticker Message"""
    return StickerMessage(
        type='sticker',
        package_id=str(package_id),
        sticker_id=str(sticker_id)
    )


def location_message(
    title: str,
    address: str,
    latitude: float,
    longitude: float
) -> LocationMessage:
    """สร้าง Location Message"""
    return LocationMessage(
        type='location',
        title=title,
        address=address,
        latitude=latitude,
        longitude=longitude
    )


def image_message(
    original_url: str,
    preview_url: Optional[str] = None
) -> ImageMessage:
    """สร้าง Image Message"""
    return ImageMessage(
        type='image',
        original_content_url=original_url,
        preview_image_url=preview_url or original_url
    )


def video_message(
    video_url: str,
    thumbnail_url: str,
    tracking_id: Optional[str] = None
) -> VideoMessage:
    """สร้าง Video Message"""
    msg = VideoMessage(
        type='video',
        original_content_url=video_url,
        preview_image_url=thumbnail_url
    )
    if tracking_id:
        msg.tracking_id = tracking_id
    return msg


def audio_message(audio_url: str, duration_ms: int) -> AudioMessage:
    """สร้าง Audio Message"""
    return AudioMessage(
        type='audio',
        original_content_url=audio_url,
        duration=duration_ms
    )


def imagemap_message(
    base_url: str,
    alt_text: str,
    width: int,
    height: int,
    actions: List
) -> ImagemapMessage:
    """สร้าง Imagemap Message"""
    return ImagemapMessage(
        type='imagemap',
        base_url=base_url,
        alt_text=alt_text,
        base_size=ImagemapBaseSize(width=width, height=height),
        actions=actions
    )


def uri_imagemap_action(
    uri: str,
    x: int, y: int, width: int, height: int
) -> URIImagemapAction:
    """สร้าง URI Action สำหรับ Imagemap"""
    return URIImagemapAction(
        type='uri',
        link_uri=uri,
        area=ImagemapArea(x=x, y=y, width=width, height=height)
    )


def message_imagemap_action(
    text: str,
    x: int, y: int, width: int, height: int
) -> MessageImagemapAction:
    """สร้าง Message Action สำหรับ Imagemap"""
    return MessageImagemapAction(
        type='message',
        text=text,
        area=ImagemapArea(x=x, y=y, width=width, height=height)
    )
```

### handlers/message_handler.py

```python
import logging
from linebot.v3.webhooks import (
    MessageEvent, TextMessageContent, ImageMessageContent,
    VideoMessageContent, AudioMessageContent, FileMessageContent,
    LocationMessageContent, StickerMessageContent
)
from utils.message_builder import (
    text_message, sticker_message, location_message,
    image_message, video_message, audio_message, imagemap_message,
    uri_imagemap_action, message_imagemap_action
)
from services.line_service import reply_message

logger = logging.getLogger(__name__)


def handle_message_event(event: MessageEvent):
    """จัดการ Message Events ทุกประเภท"""
    reply_token = event.reply_token
    message = event.message
    
    handlers = {
        TextMessageContent: handle_text_message,
        ImageMessageContent: handle_image_message,
        VideoMessageContent: handle_video_message,
        AudioMessageContent: handle_audio_message,
        FileMessageContent: handle_file_message,
        LocationMessageContent: handle_location_message,
        StickerMessageContent: handle_sticker_message,
    }
    
    handler = handlers.get(type(message))
    
    if handler:
        return handler(event)
    else:
        return reply_message(reply_token, 'ไม่รองรับประเภทข้อความนี้')


def handle_text_message(event: MessageEvent):
    """จัดการ Text Message"""
    reply_token = event.reply_token
    text = event.message.text.strip()
    
    command_map = {
        'รูปภาพ': send_sample_image,
        'วิดีโอ': send_sample_video,
        'สติกเกอร์': send_sample_sticker,
        'แผนที่': send_sample_location,
        'เสียง': send_sample_audio,
        'imagemap': send_sample_imagemap,
    }
    
    handler = command_map.get(text.lower())
    if handler:
        return handler(reply_token)
    
    # Default echo
    return reply_message(reply_token, [
        text_message(f'📨 ได้รับข้อความ: "{text}"'),
        text_message('พิมพ์: รูปภาพ, วิดีโอ, สติกเกอร์, แผนที่, เสียง, imagemap')
    ])


def send_sample_image(reply_token: str):
    """ส่ง Sample Image"""
    return reply_message(reply_token, [
        text_message('🖼️ ตัวอย่างรูปภาพ:'),
        image_message(
            'https://picsum.photos/id/237/1040/1040',
            'https://picsum.photos/id/237/400/400'
        )
    ])


def send_sample_sticker(reply_token: str):
    """ส่ง Sample Sticker"""
    return reply_message(reply_token, [
        text_message('😊 ตัวอย่าง Sticker:'),
        sticker_message('1', '1'),
        sticker_message('1', '2')
    ])


def send_sample_location(reply_token: str):
    """ส่ง Sample Location"""
    return reply_message(reply_token, [
        text_message('📍 ที่ตั้งของเรา:'),
        location_message(
            'สำนักงานใหญ่',
            '123 ถนนสุขุมวิท กรุงเทพฯ',
            13.7563,
            100.5018
        )
    ])


def send_sample_imagemap(reply_token: str):
    """ส่ง Sample Imagemap"""
    actions = [
        uri_imagemap_action('https://line.me', 0, 0, 520, 520),
        message_imagemap_action('คลิกส่วนบนขวา', 520, 0, 520, 520),
        message_imagemap_action('คลิกส่วนล่างซ้าย', 0, 520, 520, 520),
        message_imagemap_action('คลิกส่วนล่างขวา', 520, 520, 520, 520),
    ]
    
    return reply_message(reply_token, [
        text_message('🗺️ ตัวอย่าง Imagemap:'),
        imagemap_message(
            base_url='https://picsum.photos/id/1015',
            alt_text='Clickable Image',
            width=1040,
            height=1040,
            actions=actions
        )
    ])


def handle_image_message(event: MessageEvent):
    """จัดการรูปภาพที่ผู้ใช้ส่งมา"""
    return reply_message(event.reply_token, [
        text_message('📸 ได้รับรูปภาพแล้ว! กำลังประมวลผล...')
    ])


def handle_video_message(event: MessageEvent):
    """จัดการวิดีโอที่ผู้ใช้ส่งมา"""
    return reply_message(event.reply_token, [
        text_message('🎥 ได้รับวิดีโอแล้ว! กำลังประมวลผล...')
    ])


def handle_audio_message(event: MessageEvent):
    """จัดการเสียงที่ผู้ใช้ส่งมา"""
    return reply_message(event.reply_token, [
        text_message('🎵 ได้รับไฟล์เสียงแล้ว! กำลังประมวลผล...')
    ])


def handle_file_message(event: MessageEvent):
    """จัดการไฟล์ที่ผู้ใช้ส่งมา"""
    msg = event.message
    reply_message(event.reply_token, [
        text_message(f'📄 ได้รับไฟล์: {msg.file_name} ({msg.file_size:,} bytes)')
    ])


def handle_location_message(event: MessageEvent):
    """จัดการ Location ที่ผู้ใช้ส่งมา"""
    msg = event.message
    reply_message(event.reply_token, [
        text_message(
            f'📍 ได้รับตำแหน่ง:\n'
            f'{msg.title or "ไม่มีชื่อ"}\n'
            f'{msg.address or ""}\n'
            f'({msg.latitude:.6f}, {msg.longitude:.6f})'
        )
    ])


def handle_sticker_message(event: MessageEvent):
    """จัดการ Sticker ที่ผู้ใช้ส่งมา"""
    reply_message(event.reply_token, [
        sticker_message('1', '2')  # Happy sticker
    ])
```

---

## 12. Best Practices

### การเลือกประเภทข้อความ

| สถานการณ์ | ประเภทที่แนะนำ | เหตุผล |
|----------|-------------|-------|
| ข่าวสารทั่วไป | Text | เรียบง่าย ตรงไปตรงมา |
| โปรโมชั่น | Image + Text | ดึงดูดสายตา |
| Tutorial | Video | อธิบายได้ชัดเจน |
| ที่ตั้งสาขา | Location | นำทางได้ทันที |
| สินค้าหลายรายการ | Imagemap | Interactive, compact |
| ตอบรับ/สัมพันธ์ | Sticker | เป็นมิตร |

### ข้อควรระวัง

1. **HTTPS เท่านั้น** - URL ทั้งหมดต้องเป็น HTTPS
2. **ขนาดไฟล์** - ตรวจสอบขนาดไฟล์ก่อนส่ง
3. **Format** - ใช้ format ที่รองรับเท่านั้น
4. **CDN** - ใช้ CDN สำหรับไฟล์ขนาดใหญ่
5. **Preview URL** - สร้าง thumbnail ที่เหมาะสมสำหรับ preview

### Performance Tips

```javascript
// Cache message objects ที่ใช้บ่อย
const MESSAGES = {
  welcome: [
    { type: 'text', text: '👋 ยินดีต้อนรับ!' },
    { type: 'sticker', packageId: '1', stickerId: '1' }
  ],
  
  help: [
    { type: 'text', text: '📖 วิธีใช้งาน...' }
  ],
  
  error: [
    { type: 'text', text: '❌ เกิดข้อผิดพลาด กรุณาลองใหม่' }
  ]
};

// ใช้ Object.freeze() เพื่อป้องกันการแก้ไข
Object.freeze(MESSAGES);
```

---

## 13. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Multi-type Response Bot
สร้าง Bot ที่ตอบสนองต่อคำสั่ง:
- "รูป" → ส่งรูปภาพสินค้า
- "แผนที่" → ส่ง location สาขา
- "sticker" → ส่ง sticker ต้อนรับ
- "วิดีโอ" → ส่งวิดีโอแนะนำ

### แบบฝึกหัดที่ 2: Photo Bot
สร้าง Bot ที่:
- รับรูปภาพจากผู้ใช้
- ดาวน์โหลดและบันทึกลงเซิร์ฟเวอร์
- ตอบกลับด้วยรายละเอียดไฟล์

### แบบฝึกหัดที่ 3: Product Imagemap
สร้าง Imagemap ที่มี:
- รูปสินค้า 4 รายการในกริด 2x2
- แต่ละส่วนเมื่อคลิกเปิดหน้าสินค้าใน browser
- มีข้อความกำกับแต่ละสินค้า

### เฉลยแบบฝึกหัดที่ 1 (Python)

```python
# handlers/product_handler.py
from utils.message_builder import text_message, image_message, location_message, sticker_message
from services.line_service import reply_message

PRODUCTS = {
    'เสื้อยืด': {
        'url': 'https://picsum.photos/id/200/1040/1040',
        'preview': 'https://picsum.photos/id/200/400/400'
    },
    'กางเกง': {
        'url': 'https://picsum.photos/id/201/1040/1040',
        'preview': 'https://picsum.photos/id/201/400/400'
    }
}

BRANCHES = [
    {
        'name': 'สาขา CentralWorld',
        'address': '999/9 ถนนพระราม 1 กรุงเทพฯ',
        'lat': 13.7467,
        'lng': 100.5394
    },
    {
        'name': 'สาขา Siam Paragon',
        'address': '991 ถนนพระราม 1 กรุงเทพฯ',
        'lat': 13.7461,
        'lng': 100.5347
    }
]

def handle_product_command(reply_token: str, command: str):
    if command == 'รูป':
        messages = [text_message('📸 สินค้าของเรา:')]
        for name, product in PRODUCTS.items():
            messages.append(text_message(name))
            messages.append(image_message(product['url'], product['preview']))
        reply_message(reply_token, messages[:5])  # สูงสุด 5 ข้อความ
    
    elif command == 'แผนที่':
        messages = [text_message('📍 สาขาของเรา:')]
        for branch in BRANCHES:
            messages.append(location_message(
                branch['name'], branch['address'],
                branch['lat'], branch['lng']
            ))
        reply_message(reply_token, messages[:5])
    
    elif command == 'sticker':
        reply_message(reply_token, [
            sticker_message('1', '1'),
            text_message('ยินดีต้อนรับ! 😊')
        ])
```

---

**สรุปบทเรียน:**

ในบทนี้เราได้เรียนรู้ประเภทข้อความหลักของ LINE Messaging API:

- **Text** - ข้อความธรรมดา + LINE Emojis
- **Sticker** - สติกเกอร์จาก packages ที่รองรับ
- **Location** - แชร์ตำแหน่ง GPS
- **Image** - รูปภาพ JPEG/PNG จาก HTTPS URL
- **Video** - วิดีโอ MP4 จาก HTTPS URL
- **Audio** - ไฟล์เสียง M4A จาก HTTPS URL
- **File** - รับได้อย่างเดียว (ผู้ใช้ส่งมา)
- **Imagemap** - รูปภาพที่มี clickable areas
