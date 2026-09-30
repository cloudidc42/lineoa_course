# ส่วนที่ 55: Audience API - การจัดการกลุ่มเป้าหมายอย่างครบวงจร

---

## สารบัญ

1. [Audience API คืออะไร?](#1-audience-api-คืออะไร)
2. [ทำไมต้องใช้ Audience API แทนการสร้างกลุ่มเป้าหมายด้วยตนเอง](#2-ทำไมต้องใช้-audience-api)
3. [ประเภทของ Audience](#3-ประเภทของ-audience)
4. [การสร้าง Audience ผ่าน API](#4-การสร้าง-audience-ผ่าน-api)
5. [การดึงรายการ Audience](#5-การดึงรายการ-audience)
6. [การดูรายละเอียด Audience](#6-การดูรายละเอียด-audience)
7. [การอัปเดต Audience](#7-การอัปเดต-audience)
8. [การลบ Audience](#8-การลบ-audience)
9. [ขนาดและข้อกำหนดของ Audience](#9-ขนาดและข้อกำหนดของ-audience)
10. [การรวม Audience กับ Filter](#10-การรวม-audience-กับ-filter)
11. [การใช้ Audience ใน Narrowcast API](#11-การใช้-audience-ใน-narrowcast-api)
12. [Audience สำหรับ LINE Ads (Retargeting)](#12-audience-สำหรับ-line-ads)
13. [PDPA และข้อพิจารณาด้านความเป็นส่วนตัว](#13-pdpa-และข้อพิจารณา)
14. [AudienceManager Class (Node.js)](#14-audiencemanager-class-nodejs)
15. [AudienceService Class (Python)](#15-audienceservice-class-python)
16. [ตัวอย่างการใช้งานจริง](#16-ตัวอย่างการใช้งานจริง)
17. [แบบฝึกหัดท้ายบท](#17-แบบฝึกหัดท้ายบท)

---

## 1. Audience API คืออะไร?

**Audience API** คือชุด API ของ LINE Messaging API ที่ช่วยให้ธุรกิจสามารถสร้าง จัดการ และใช้งาน "กลุ่มผู้รับข้อความ (Audience)" ได้โดยอัตโนมัติผ่านโปรแกรม แทนการทำงานด้วยมือในหน้า LINE Official Account Manager

### ภาพรวมระบบ Audience

```
LINE Audience API
├── Audience Management
│   ├── Create Audience (สร้างกลุ่ม)
│   ├── Get Audience List (ดูรายการกลุ่ม)
│   ├── Get Audience Details (ดูรายละเอียดกลุ่ม)
│   ├── Update Audience (อัปเดตกลุ่ม)
│   └── Delete Audience (ลบกลุ่ม)
├── Audience Types (ประเภทกลุ่ม)
│   ├── Upload Audience (อัปโหลด User ID / IFA)
│   ├── Click Audience (คลิกข้อความ)
│   ├── Impression Audience (เห็นข้อความ)
│   ├── Video View Audience (ดูวิดีโอ)
│   ├── Chat Tag Audience (แท็กแชท)
│   ├── Add Friend Audience (เพิ่มเพื่อน)
│   └── Reserved Audience (สำรองไว้)
└── Integration
    ├── Narrowcast API (ส่งข้อความเจาะจง)
    └── LINE Ads (โฆษณา)
```

### Base URL

```
https://api.line.me/v2/bot/audienceGroup
```

### Authentication Header

```http
Authorization: Bearer {Channel Access Token}
Content-Type: application/json
```

### ขีดจำกัดของ API

| รายการ | ขีดจำกัด |
|--------|----------|
| จำนวน Audience สูงสุดต่อ Channel | 1,000 กลุ่ม |
| ขนาด Upload ต่อครั้ง | 10,000 User IDs |
| ขนาด Audience สูงสุด | ไม่จำกัด (รวมหลายครั้ง) |
| Audience ขั้นต่ำสำหรับ Narrowcast | 50 คน |
| Rate Limit | 100 requests/วินาที |

---

## 2. ทำไมต้องใช้ Audience API

### เปรียบเทียบการสร้างกลุ่มเป้าหมายด้วยมือ vs API

| หัวข้อ | สร้างด้วยมือ (LINE OA Manager) | ใช้ Audience API |
|--------|-------------------------------|-----------------|
| ความเร็ว | ช้า (ต้องคลิกหลายขั้นตอน) | เร็ว (Automate ได้) |
| ขนาดข้อมูล | จำกัด (อัปโหลด CSV) | รองรับข้อมูลขนาดใหญ่ |
| การอัปเดต | ต้องทำเองทุกครั้ง | Real-time อัตโนมัติ |
| การรวมระบบ | ยาก | ง่าย (REST API) |
| ความแม่นยำ | ขึ้นกับมนุษย์ | 100% แม่นยำ |
| เวลาทำงาน | เฉพาะเวลาทำงาน | 24/7 อัตโนมัติ |
| Audit Trail | ไม่มี | มี Log ครบ |
| Dynamic Update | ไม่รองรับ | รองรับแบบ Real-time |

### กรณีที่ควรใช้ Audience API

**1. กรณีข้อมูลขนาดใหญ่**
```
สถานการณ์: มีลูกค้า 500,000 คนที่ซื้อสินค้าในช่วง 30 วันที่ผ่านมา
ปัญหาถ้าทำด้วยมือ: ต้องส่งออก CSV → อัปโหลดทีละไฟล์ → ใช้เวลาหลายชั่วโมง
แก้ด้วย API: ดึงข้อมูลจาก DB → สร้าง Audience อัตโนมัติ → เสร็จใน 5 นาที
```

**2. กรณี Dynamic Audience**
```
สถานการณ์: ต้องการส่งข้อความหาลูกค้าที่ยังไม่ได้ชำระเงินค้างอยู่
ปัญหาถ้าทำด้วยมือ: รายชื่อเปลี่ยนทุกวัน ต้องอัปเดตทุกวัน
แก้ด้วย API: Cron Job รัน Query DB → สร้าง/อัปเดต Audience ทุกคืน
```

**3. กรณี A/B Testing อัตโนมัติ**
```
สถานการณ์: ต้องการทดสอบข้อความ 3 แบบกับลูกค้ากลุ่มเดิม
ปัญหาถ้าทำด้วยมือ: ต้องแบ่งกลุ่มและสร้างทีละกลุ่ม
แก้ด้วย API: แบ่งข้อมูล → สร้าง 3 Audience พร้อมกัน → ส่งข้อความต่างกัน
```

**4. กรณี Re-targeting อัตโนมัติ**
```
สถานการณ์: ต้องการส่ง Follow-up ให้ผู้ที่คลิก Coupon แต่ยังไม่ซื้อ
ปัญหาถ้าทำด้วยมือ: ต้องรอรายงาน → Export → กรอง → อัปโหลด
แก้ด้วย API: ใช้ Click Audience → กรองใน CRM → Narrowcast อัตโนมัติ
```

---

## 3. ประเภทของ Audience

### 3.1 Upload Audience (กลุ่มจาก User ID / IFA)

**อธิบาย:** สร้างกลุ่มจากรายการ LINE User ID หรือ IFA (Identifier for Advertisers) ที่คุณอัปโหลดเองโดยตรง

**Use Case:**
- ลูกค้าที่ลงทะเบียนในระบบ CRM
- ผู้ใช้แอปที่ Link กับ LINE Login
- รายชื่อลูกค้า VIP จากระบบ POS

**ข้อกำหนด:**
```
- ต้องมี User ID จาก LINE (ไม่ใช่ชื่อหรืออีเมล)
- User ID ต้องอยู่ในรูปแบบ Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
- IFA ใช้สำหรับ LINE Ads เท่านั้น (iOS: IDFA, Android: GAID)
- ขั้นต่ำ 50 User IDs สำหรับ Narrowcast
```

**JSON Request ตัวอย่าง:**
```json
{
  "description": "VIP Customers - September 2024",
  "isIfaAudience": false,
  "audiences": [
    { "id": "U4af4980629..." },
    { "id": "U8edia980629..." },
    { "id": "U1bdia980629..." }
  ]
}
```

---

### 3.2 Click Audience (กลุ่มจากการคลิกข้อความ)

**อธิบาย:** สร้างกลุ่มโดยอัตโนมัติจากผู้ที่คลิก URL ใน Broadcast Message หรือ Narrowcast Message ที่ระบุ

**Use Case:**
- ผู้ที่สนใจโปรโมชั่น (คลิก URL ใน Broadcast)
- ลูกค้าที่กดดู Product Landing Page
- Re-targeting ผู้ที่คลิกแต่ยังไม่ซื้อ

**การทำงาน:**
```
1. ส่ง Broadcast Message ที่มี URL
2. LINE Platform ติดตามว่าใครคลิก URL นั้น
3. สร้าง Click Audience โดยระบุ requestId ของ Broadcast นั้น
4. Audience จะมีผู้ที่คลิก URL ทุกคนในนั้น
```

**JSON Request ตัวอย่าง:**
```json
{
  "description": "Clicked Promo October 2024",
  "requestId": "5b59509c-c57b-11e9-aa8c-2a86e4085a59",
  "clickUrl": "https://example.com/promo"
}
```

> **หมายเหตุ:** `requestId` คือ ID ของข้อความที่ส่งไป ได้มาจาก response ของ Broadcast API

---

### 3.3 Impression Audience (กลุ่มจากการเห็นข้อความ)

**อธิบาย:** สร้างกลุ่มจากผู้ที่เปิดดูข้อความ (Impression) ไม่จำเป็นต้องคลิก เพียงแค่ข้อความถูกแสดงในหน้าจอ

**Use Case:**
- ผู้ที่เห็นแคมเปญแต่ยังไม่ตอบสนอง
- วัดว่าใครอ่านข่าวสารแต่ไม่คลิก
- ส่ง Follow-up ให้ผู้ที่เห็นแต่ไม่ดำเนินการ

**ความแตกต่าง Click vs Impression:**
```
Impression Audience = คนที่เปิดดูข้อความ (ใหญ่กว่า)
Click Audience = คนที่คลิก URL (เล็กกว่า แต่ Intent สูงกว่า)

Impression ⊇ Click (ทุกคนที่คลิกต้องเห็นก่อน)
```

**JSON Request ตัวอย่าง:**
```json
{
  "description": "Saw October Newsletter",
  "requestId": "5b59509c-c57b-11e9-aa8c-2a86e4085a59"
}
```

---

### 3.4 Video View Audience (กลุ่มจากการดูวิดีโอ)

**อธิบาย:** สร้างกลุ่มจากผู้ที่ดูวิดีโอใน Broadcast Message ถึง % ที่กำหนด เช่น ดูถึง 25%, 50%, 75%, 95%

**Use Case:**
- ผู้ที่ดูโฆษณาวิดีโอจบ (High Intent)
- Re-targeting ผู้ที่ดูวิดีโอสินค้าแล้วยังไม่ซื้อ
- วัด Engagement ของวิดีโอ Content

**Percentage Threshold ที่รองรับ:**
```
RATE_25  = ดูวิดีโอถึง 25%
RATE_50  = ดูวิดีโอถึง 50%
RATE_75  = ดูวิดีโอถึง 75%
RATE_95  = ดูวิดีโอถึง 95% (เกือบจบ)
```

**JSON Request ตัวอย่าง:**
```json
{
  "description": "Watched Product Video 95%",
  "requestId": "5b59509c-c57b-11e9-aa8c-2a86e4085a59",
  "viewedRate": "RATE_95"
}
```

---

### 3.5 Chat Tag Audience (กลุ่มจาก Chat Tag)

**อธิบาย:** สร้างกลุ่มจากผู้ที่มี Tag ที่แปะไว้ในหน้าแชทของ LINE OA Manager เช่น "VIP", "ซื้อแล้ว", "สนใจโปรดักส์ A"

**Use Case:**
- ส่งข้อความเฉพาะลูกค้าที่ Staff แท็กว่า "Hot Lead"
- Broadcast ให้เฉพาะลูกค้าที่แท็กว่า "VIP"
- ติดตามผลลูกค้าที่แท็กว่า "ต้องติดตาม"

**ข้อกำหนด:**
```
- Tag ต้องถูกสร้างในหน้า LINE OA Manager ก่อน
- ใช้ Chat Tag ID (ตัวเลข) ไม่ใช่ชื่อ Tag
- ต้องมีผู้ใช้ในกลุ่มนี้อย่างน้อย 50 คน
```

**JSON Request ตัวอย่าง:**
```json
{
  "description": "Chat Tag: VIP Customers",
  "tagNames": ["VIP", "Gold Member"]
}
```

---

### 3.6 Add Friend Audience (กลุ่มจากการเพิ่มเพื่อน)

**อธิบาย:** สร้างกลุ่มจากผู้ที่เพิ่มเพื่อนจากแหล่งที่มาที่ระบุ เช่น QR Code, URL เฉพาะ, LINE Ads Campaign

**Add Friend Paths (ช่องทางการเพิ่มเพื่อน):**
```
FRIEND_PATH_QR         = สแกน QR Code
FRIEND_PATH_URL        = กด URL เพิ่มเพื่อน
FRIEND_PATH_SEARCH     = ค้นหาใน LINE App
FRIEND_PATH_CARD       = ได้รับ Business Card
FRIEND_PATH_SMS        = คลิกจาก SMS
FRIEND_PATH_PROMOTION  = จาก Promotion
FRIEND_PATH_CHAT_SCREEN = จากหน้าจอแชท
FRIEND_PATH_LINE_AD    = จาก LINE Ads
```

**Use Case:**
- ส่งข้อความต้อนรับเฉพาะผู้ที่เพิ่มเพื่อนจาก Campaign QR
- วิเคราะห์คุณภาพลูกค้าตามแหล่งที่มา
- ส่งข้อความตาม Journey ที่ต่างกัน

**JSON Request ตัวอย่าง:**
```json
{
  "description": "Added via QR Code - Store Bangkok",
  "audienceGroupType": "FRIEND_PATH",
  "description": "QR Code Campaign Sep 2024"
}
```

---

### 3.7 Reserved Audience (กลุ่มสำรอง)

**อธิบาย:** Audience ที่จองพื้นที่ไว้ก่อน แล้วค่อยเพิ่มข้อมูลในภายหลัง ใช้สำหรับ Workflow ที่ต้องการสร้าง Audience ล่วงหน้า

**Use Case:**
- สร้าง Audience ไว้ก่อนสำหรับ Campaign ที่จะถึง
- Placeholder สำหรับ Audience ที่ข้อมูลยังไม่พร้อม
- ใช้ใน Automation Pipeline ที่สร้าง → เติมข้อมูล → ส่ง

---

## 4. การสร้าง Audience ผ่าน API

### 4.1 สร้าง Upload Audience

**Endpoint:**
```
POST https://api.line.me/v2/bot/audienceGroup/upload
```

**Request Headers:**
```http
Authorization: Bearer {Channel Access Token}
Content-Type: application/json
```

**Request Body:**
```json
{
  "description": "VIP Customers Q4 2024",
  "isIfaAudience": false,
  "audiences": [
    { "id": "U4af4980629b0fcfcbf22adf96359dbf8" },
    { "id": "U8edia98062abcdef1234567890abcdef" },
    { "id": "U1bdia980629ccccdddd1111222233334" }
  ]
}
```

**Response:**
```json
{
  "audienceGroupId": 5614991017776,
  "type": "UPLOAD",
  "description": "VIP Customers Q4 2024",
  "created": 1713686400,
  "permission": "READ_WRITE",
  "createRoute": "OA_MANAGER",
  "status": "READY",
  "audienceCount": 3,
  "isIfaAudience": false
}
```

**Node.js ตัวอย่าง:**
```javascript
const axios = require('axios');

async function createUploadAudience(description, userIds) {
  const audiences = userIds.map(id => ({ id }));
  
  const response = await axios.post(
    'https://api.line.me/v2/bot/audienceGroup/upload',
    {
      description,
      isIfaAudience: false,
      audiences
    },
    {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
        'Content-Type': 'application/json'
      }
    }
  );
  
  return response.data;
}

// การใช้งาน
const userIds = [
  'U4af4980629b0fcfcbf22adf96359dbf8',
  'U8edia98062abcdef1234567890abcdef'
];

createUploadAudience('VIP Customers Q4 2024', userIds)
  .then(result => console.log('Created:', result.audienceGroupId))
  .catch(err => console.error('Error:', err.response?.data));
```

**Python ตัวอย่าง:**
```python
import requests
import os

def create_upload_audience(description: str, user_ids: list[str]) -> dict:
    audiences = [{"id": uid} for uid in user_ids]
    
    response = requests.post(
        'https://api.line.me/v2/bot/audienceGroup/upload',
        json={
            "description": description,
            "isIfaAudience": False,
            "audiences": audiences
        },
        headers={
            "Authorization": f"Bearer {os.environ['LINE_CHANNEL_ACCESS_TOKEN']}",
            "Content-Type": "application/json"
        }
    )
    response.raise_for_status()
    return response.json()

# การใช้งาน
user_ids = [
    "U4af4980629b0fcfcbf22adf96359dbf8",
    "U8edia98062abcdef1234567890abcdef"
]

result = create_upload_audience("VIP Customers Q4 2024", user_ids)
print(f"Created Audience ID: {result['audienceGroupId']}")
```

---

### 4.2 สร้าง Click Audience

**Endpoint:**
```
POST https://api.line.me/v2/bot/audienceGroup/click
```

**Request Body:**
```json
{
  "description": "Clicked Promo Oct 2024",
  "requestId": "5b59509c-c57b-11e9-aa8c-2a86e4085a59",
  "clickUrl": "https://example.com/promo-october"
}
```

> `clickUrl` เป็น optional - ถ้าไม่ระบุจะรวมทุก URL ที่ถูกคลิกในข้อความนั้น

**Response:**
```json
{
  "audienceGroupId": 5614991017777,
  "type": "CLICK",
  "description": "Clicked Promo Oct 2024",
  "created": 1713686400,
  "requestId": "5b59509c-c57b-11e9-aa8c-2a86e4085a59",
  "clickUrl": "https://example.com/promo-october",
  "status": "IN_PROGRESS"
}
```

**หมายเหตุ:** Status จะเป็น `IN_PROGRESS` ก่อน LINE Platform จะประมวลผลข้อมูลก่อนเปลี่ยนเป็น `READY`

**Node.js ตัวอย่าง:**
```javascript
async function createClickAudience(description, requestId, clickUrl = null) {
  const body = { description, requestId };
  if (clickUrl) body.clickUrl = clickUrl;
  
  const response = await axios.post(
    'https://api.line.me/v2/bot/audienceGroup/click',
    body,
    {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
        'Content-Type': 'application/json'
      }
    }
  );
  
  return response.data;
}
```

---

### 4.3 สร้าง Impression Audience

**Endpoint:**
```
POST https://api.line.me/v2/bot/audienceGroup/impression
```

**Request Body:**
```json
{
  "description": "Saw October Newsletter",
  "requestId": "5b59509c-c57b-11e9-aa8c-2a86e4085a59"
}
```

**Response:**
```json
{
  "audienceGroupId": 5614991017778,
  "type": "IMPRESSION",
  "description": "Saw October Newsletter",
  "created": 1713686400,
  "requestId": "5b59509c-c57b-11e9-aa8c-2a86e4085a59",
  "status": "IN_PROGRESS"
}
```

---

### 4.4 สร้าง Video View Audience

**Endpoint:**
```
POST https://api.line.me/v2/bot/audienceGroup/click
```

**Request Body:**
```json
{
  "description": "Watched 95% of Product Video",
  "requestId": "5b59509c-c57b-11e9-aa8c-2a86e4085a59",
  "viewedRate": "RATE_95"
}
```

**viewedRate Values:**
- `RATE_25` - ดูถึง 25%
- `RATE_50` - ดูถึง 50%
- `RATE_75` - ดูถึง 75%
- `RATE_95` - ดูถึง 95%

---

## 5. การดึงรายการ Audience

**Endpoint:**
```
GET https://api.line.me/v2/bot/audienceGroup/list
```

**Query Parameters:**
```
page        = หมายเลขหน้า (เริ่มที่ 1)
description = กรองตามชื่อ (optional)
status      = กรองตาม status (optional): IN_PROGRESS | READY | FAILED | EXPIRED
size        = จำนวนต่อหน้า (สูงสุด 40, default 40)
includesExternalPublicGroups = รวม Audience จาก LINE Ads (optional)
createRoute = กรองตามแหล่งสร้าง (optional): OA_MANAGER | MESSAGING_API
```

**Response:**
```json
{
  "audienceGroups": [
    {
      "audienceGroupId": 5614991017776,
      "type": "UPLOAD",
      "description": "VIP Customers Q4 2024",
      "status": "READY",
      "audienceCount": 1523,
      "created": 1713686400,
      "permission": "READ_WRITE",
      "createRoute": "MESSAGING_API",
      "isIfaAudience": false
    },
    {
      "audienceGroupId": 5614991017777,
      "type": "CLICK",
      "description": "Clicked Promo Oct 2024",
      "status": "READY",
      "audienceCount": 342,
      "created": 1713600000,
      "permission": "READ_WRITE",
      "createRoute": "MESSAGING_API",
      "requestId": "5b59509c-c57b-11e9-aa8c-2a86e4085a59"
    }
  ],
  "hasNextPage": true,
  "totalCount": 45,
  "readWriteAudienceGroupTotalCount": 30,
  "page": 1,
  "size": 40
}
```

**Node.js ตัวอย่าง:**
```javascript
async function getAudienceList(page = 1, status = null) {
  const params = { page, size: 40 };
  if (status) params.status = status;
  
  const response = await axios.get(
    'https://api.line.me/v2/bot/audienceGroup/list',
    {
      params,
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`
      }
    }
  );
  
  return response.data;
}

// ดึงทุก Audience ที่พร้อมใช้งาน
async function getAllReadyAudiences() {
  const allAudiences = [];
  let page = 1;
  let hasNextPage = true;
  
  while (hasNextPage) {
    const result = await getAudienceList(page, 'READY');
    allAudiences.push(...result.audienceGroups);
    hasNextPage = result.hasNextPage;
    page++;
  }
  
  return allAudiences;
}
```

**Python ตัวอย่าง:**
```python
def get_audience_list(page: int = 1, status: str = None) -> dict:
    params = {"page": page, "size": 40}
    if status:
        params["status"] = status
    
    response = requests.get(
        'https://api.line.me/v2/bot/audienceGroup/list',
        params=params,
        headers={
            "Authorization": f"Bearer {os.environ['LINE_CHANNEL_ACCESS_TOKEN']}"
        }
    )
    response.raise_for_status()
    return response.json()

def get_all_ready_audiences() -> list:
    all_audiences = []
    page = 1
    
    while True:
        result = get_audience_list(page, status="READY")
        all_audiences.extend(result["audienceGroups"])
        
        if not result.get("hasNextPage"):
            break
        page += 1
    
    return all_audiences
```

---

## 6. การดูรายละเอียด Audience

**Endpoint:**
```
GET https://api.line.me/v2/bot/audienceGroup/{audienceGroupId}
```

**Response:**
```json
{
  "audienceGroup": {
    "audienceGroupId": 5614991017776,
    "type": "UPLOAD",
    "description": "VIP Customers Q4 2024",
    "status": "READY",
    "audienceCount": 1523,
    "created": 1713686400,
    "permission": "READ_WRITE",
    "createRoute": "MESSAGING_API",
    "isIfaAudience": false,
    "expireTimestamp": 1745222400,
    "isLatestAudienceSupported": true
  },
  "jobs": [
    {
      "audienceGroupJobId": 12345,
      "audienceGroupId": 5614991017776,
      "description": "Initial Upload",
      "type": "DIFF_ADD",
      "status": "FINISHED",
      "audienceCount": 1523,
      "created": 1713686400,
      "completed": 1713686450,
      "failedType": null,
      "errorCode": 0
    }
  ]
}
```

**Status Values:**
```
IN_PROGRESS = กำลังประมวลผล
READY       = พร้อมใช้งาน
FAILED      = เกิดข้อผิดพลาด
EXPIRED     = หมดอายุ (Audience อายุ 180 วัน)
```

**Node.js ตัวอย่าง:**
```javascript
async function getAudienceDetails(audienceGroupId) {
  const response = await axios.get(
    `https://api.line.me/v2/bot/audienceGroup/${audienceGroupId}`,
    {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`
      }
    }
  );
  
  return response.data;
}

// ตรวจสอบว่า Audience พร้อมใช้งานหรือยัง
async function waitForAudienceReady(audienceGroupId, maxRetries = 10) {
  for (let i = 0; i < maxRetries; i++) {
    const details = await getAudienceDetails(audienceGroupId);
    const status = details.audienceGroup.status;
    
    if (status === 'READY') {
      console.log(`Audience ${audienceGroupId} is ready with ${details.audienceGroup.audienceCount} members`);
      return details.audienceGroup;
    }
    
    if (status === 'FAILED') {
      throw new Error(`Audience ${audienceGroupId} failed to process`);
    }
    
    console.log(`Waiting... Status: ${status} (attempt ${i + 1}/${maxRetries})`);
    await new Promise(resolve => setTimeout(resolve, 5000)); // รอ 5 วินาที
  }
  
  throw new Error(`Audience ${audienceGroupId} did not become READY within timeout`);
}
```

---

## 7. การอัปเดต Audience

### 7.1 อัปเดตชื่อ (Description)

**Endpoint:**
```
PUT https://api.line.me/v2/bot/audienceGroup/{audienceGroupId}/updateDescription
```

**Request Body:**
```json
{
  "description": "VIP Customers Q4 2024 - Updated"
}
```

**Response:** `200 OK` (empty body)

---

### 7.2 เพิ่ม User IDs เข้า Upload Audience

**Endpoint:**
```
PUT https://api.line.me/v2/bot/audienceGroup/upload
```

**Request Body:**
```json
{
  "audienceGroupId": 5614991017776,
  "audiences": [
    { "id": "Unew1user1234567890abcdef1234567" },
    { "id": "Unew2user1234567890abcdef7654321" }
  ]
}
```

**Response:** `202 Accepted`

**Node.js ตัวอย่าง:**
```javascript
async function addUsersToAudience(audienceGroupId, userIds) {
  const audiences = userIds.map(id => ({ id }));
  
  const response = await axios.put(
    'https://api.line.me/v2/bot/audienceGroup/upload',
    { audienceGroupId, audiences },
    {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
        'Content-Type': 'application/json'
      }
    }
  );
  
  return response.status === 202;
}
```

---

### 7.3 ลบ User IDs ออกจาก Upload Audience

**Endpoint:**
```
PUT https://api.line.me/v2/bot/audienceGroup/upload/byUpload
```

**Request Body:**
```json
{
  "audienceGroupId": 5614991017776,
  "audiences": [
    { "id": "U4af4980629b0fcfcbf22adf96359dbf8" }
  ]
}
```

**Node.js ตัวอย่าง:**
```javascript
async function removeUsersFromAudience(audienceGroupId, userIds) {
  const audiences = userIds.map(id => ({ id }));
  
  const response = await axios.put(
    'https://api.line.me/v2/bot/audienceGroup/upload/byUpload',
    { audienceGroupId, audiences },
    {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
        'Content-Type': 'application/json'
      }
    }
  );
  
  return response.status === 202;
}
```

---

## 8. การลบ Audience

**Endpoint:**
```
DELETE https://api.line.me/v2/bot/audienceGroup/{audienceGroupId}
```

**Response:** `200 OK` (empty body)

**Node.js ตัวอย่าง:**
```javascript
async function deleteAudience(audienceGroupId) {
  const response = await axios.delete(
    `https://api.line.me/v2/bot/audienceGroup/${audienceGroupId}`,
    {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`
      }
    }
  );
  
  return response.status === 200;
}

// ลบ Audience ที่หมดอายุทั้งหมด
async function deleteExpiredAudiences() {
  const result = await getAudienceList(1, 'EXPIRED');
  const expired = result.audienceGroups;
  
  let deletedCount = 0;
  for (const audience of expired) {
    try {
      await deleteAudience(audience.audienceGroupId);
      console.log(`Deleted expired audience: ${audience.description} (${audience.audienceGroupId})`);
      deletedCount++;
    } catch (err) {
      console.error(`Failed to delete ${audience.audienceGroupId}:`, err.message);
    }
  }
  
  return deletedCount;
}
```

---

## 9. ขนาดและข้อกำหนดของ Audience

### ข้อกำหนดขั้นต่ำ

| ประเภท Audience | ขนาดขั้นต่ำ | หมายเหตุ |
|----------------|------------|---------|
| Upload Audience | 50 User IDs | สำหรับ Narrowcast |
| Click Audience | อัตโนมัติ | ต้องมีผู้คลิกอย่างน้อย 50 คน |
| Impression Audience | อัตโนมัติ | ต้องมีผู้เห็นอย่างน้อย 50 คน |
| Video View Audience | อัตโนมัติ | ต้องมีผู้ดูอย่างน้อย 50 คน |
| Chat Tag Audience | 50 คน | ต้องมี Tag ในระบบ |

### ข้อจำกัดสำคัญ

```
1. อายุของ Audience: 180 วัน (6 เดือน)
   - หลังจากนั้นจะเปลี่ยนเป็น EXPIRED
   - ต้องสร้างใหม่

2. จำนวน Audience ต่อ Channel: สูงสุด 1,000 กลุ่ม
   - ควรลบกลุ่มที่ไม่ใช้แล้วออก

3. ขนาดการ Upload ต่อครั้ง: 10,000 User IDs
   - ถ้ามีมากกว่าต้องแบ่งส่งหลายรอบ

4. Rate Limit: 
   - 200 requests ต่อนาที สำหรับ Upload
   - 100 requests ต่อวินาที ทั่วไป

5. เวลาประมวลผล:
   - Upload: ไม่กี่วินาที ถึงไม่กี่นาที
   - Click/Impression: 24-48 ชั่วโมง (ขึ้นกับขนาด Broadcast)
```

### การแบ่ง Upload ขนาดใหญ่

```javascript
async function uploadLargeAudience(description, allUserIds) {
  const BATCH_SIZE = 10000;
  const batches = [];
  
  // แบ่งเป็น chunks ขนาด 10,000
  for (let i = 0; i < allUserIds.length; i += BATCH_SIZE) {
    batches.push(allUserIds.slice(i, i + BATCH_SIZE));
  }
  
  // สร้าง Audience ด้วย batch แรก
  const firstBatch = batches[0];
  const audience = await createUploadAudience(description, firstBatch);
  const audienceGroupId = audience.audienceGroupId;
  
  console.log(`Created Audience ID: ${audienceGroupId}`);
  console.log(`Uploading batch 1/${batches.length} (${firstBatch.length} users)`);
  
  // เพิ่ม batch ถัดไป
  for (let i = 1; i < batches.length; i++) {
    console.log(`Uploading batch ${i + 1}/${batches.length} (${batches[i].length} users)`);
    await addUsersToAudience(audienceGroupId, batches[i]);
    
    // รอเล็กน้อยเพื่อหลีกเลี่ยง Rate Limit
    await new Promise(resolve => setTimeout(resolve, 500));
  }
  
  return audienceGroupId;
}
```

---

## 10. การรวม Audience กับ Filter

### Narrowcast กับ Demographic Filter

คุณสามารถรวม Audience กับ Demographic Filter เพื่อให้ได้กลุ่มเป้าหมายที่แม่นยำยิ่งขึ้น

**ตัวอย่าง: ส่งข้อความหา VIP Customer อายุ 25-40 ปีเท่านั้น**

```json
{
  "messages": [
    {
      "type": "text",
      "text": "สวัสดีลูกค้า VIP! มีโปรโมชั่นพิเศษสำหรับคุณ"
    }
  ],
  "recipient": {
    "type": "audienceGroup",
    "groupId": 5614991017776
  },
  "filter": {
    "demographic": {
      "type": "operator",
      "and": [
        {
          "type": "age",
          "gte": "age_25",
          "lt": "age_40"
        },
        {
          "type": "appType",
          "appType": ["ios", "android"]
        }
      ]
    }
  }
}
```

### การรวมหลาย Audience (Union)

**ตัวอย่าง: ส่งให้ทั้งกลุ่ม VIP และกลุ่มที่คลิก Promotion**

```json
{
  "recipient": {
    "type": "union",
    "audienceGroupIds": [
      5614991017776,
      5614991017777
    ]
  }
}
```

### การแยก Audience (Exclude)

**ตัวอย่าง: ส่งให้ผู้ที่คลิก Coupon แต่ไม่รวมผู้ที่ซื้อแล้ว**

```json
{
  "recipient": {
    "type": "difference",
    "minuend": {
      "type": "audienceGroup",
      "groupId": 5614991017777
    },
    "subtrahend": {
      "type": "audienceGroup",
      "groupId": 5614991017778
    }
  }
}
```

### การตัด Audience (Intersection)

**ตัวอย่าง: ส่งให้เฉพาะคนที่อยู่ทั้งสองกลุ่ม**

```json
{
  "recipient": {
    "type": "intersection",
    "audienceGroupIds": [
      5614991017776,
      5614991017777
    ]
  }
}
```

---

## 11. การใช้ Audience ใน Narrowcast API

### โครงสร้าง Narrowcast Request

```json
POST https://api.line.me/v2/bot/message/narrowcast

{
  "messages": [...],
  "recipient": {
    "type": "audienceGroup",
    "groupId": {audienceGroupId}
  },
  "filter": {
    "demographic": {...}
  },
  "limit": {
    "max": 1000,
    "upToRemainingQuota": false
  },
  "notificationDisabled": false
}
```

### ตัวอย่างครบถ้วน: Narrowcast ไปยัง Audience

**Node.js:**
```javascript
async function narrowcastToAudience(audienceGroupId, messageText, demographicFilter = null) {
  const body = {
    messages: [
      {
        type: 'text',
        text: messageText
      }
    ],
    recipient: {
      type: 'audienceGroup',
      groupId: audienceGroupId
    }
  };
  
  if (demographicFilter) {
    body.filter = { demographic: demographicFilter };
  }
  
  const response = await axios.post(
    'https://api.line.me/v2/bot/message/narrowcast',
    body,
    {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
        'Content-Type': 'application/json',
        'X-Line-Retry-Key': `narrowcast-${audienceGroupId}-${Date.now()}`
      }
    }
  );
  
  // requestId สำหรับติดตามผล
  const requestId = response.headers['x-line-request-id'];
  console.log(`Narrowcast sent, requestId: ${requestId}`);
  
  return requestId;
}

// ตัวอย่างการใช้งาน
const filter = {
  type: 'operator',
  and: [
    { type: 'age', gte: 'age_20', lt: 'age_35' },
    { type: 'gender', oneOf: ['female'] }
  ]
};

narrowcastToAudience(5614991017776, 'สวัสดีลูกค้า VIP! มีข่าวดีสำหรับคุณ', filter);
```

### ตรวจสอบผล Narrowcast

```javascript
async function getNarrowcastProgress(requestId) {
  const response = await axios.get(
    `https://api.line.me/v2/bot/message/progress/narrowcast?requestId=${requestId}`,
    {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`
      }
    }
  );
  
  return response.data;
}

// Response ตัวอย่าง
/*
{
  "phase": "succeeded",
  "successCount": 1234,
  "failureCount": 5,
  "targetCount": 1239,
  "errorCode": null,
  "acceptedTime": "2024-10-01T10:00:00.000Z",
  "completedTime": "2024-10-01T10:05:00.000Z"
}
*/
```

---

## 12. Audience สำหรับ LINE Ads

### การใช้ Audience ใน LINE Ads (Retargeting)

Audience ที่สร้างจาก Messaging API สามารถใช้ใน LINE Ads เพื่อทำ Retargeting กับผู้ใช้กลุ่มเดิม

**ประเภท Audience ที่ใช้กับ LINE Ads:**
```
1. Custom Audience - อัปโหลด User IDs / IFAs
2. Lookalike Audience - LINE หา Similar Users
3. Website Custom Audience - Pixel Tracking
4. App Custom Audience - จาก App Events
```

**การสร้าง IFA Audience สำหรับ Ads:**

```json
POST https://api.line.me/v2/bot/audienceGroup/upload

{
  "description": "App Users for Retargeting",
  "isIfaAudience": true,
  "audiences": [
    { "id": "00000000-0000-0000-0000-000000000001" },
    { "id": "00000000-0000-0000-0000-000000000002" }
  ]
}
```

**หมายเหตุสำคัญ:**
```
- IFA (Identifier for Advertisers):
  * iOS: IDFA (Identifier for Advertisers)
  * Android: GAID (Google Advertising ID)
  
- User ID กับ IFA ต้องใช้คนละ Audience
- IFA Audience ใช้ได้เฉพาะกับ LINE Ads ไม่สามารถ Narrowcast ได้
- ต้องได้รับความยินยอมจากผู้ใช้ก่อนเก็บ IFA (PDPA)
```

**Lookalike Audience:**
```
1. สร้าง Seed Audience (กลุ่มต้นแบบ) อย่างน้อย 100 คน
2. ใน LINE Ads Manager เลือก "สร้าง Lookalike Audience"
3. เลือก Seed Audience
4. LINE Algorithm จะหาคนที่มี Profile คล้ายกัน
5. ใช้ Lookalike Audience ใน LINE Ads Campaign
```

---

## 13. PDPA และข้อพิจารณาด้านความเป็นส่วนตัว

### หลักการ PDPA ที่เกี่ยวข้องกับ Audience API

**พ.ร.บ. คุ้มครองข้อมูลส่วนบุคคล พ.ศ. 2562 (PDPA)** มีผลกระทบโดยตรงต่อการใช้ Audience API ดังนี้:

#### 1. ฐานทางกฎหมายในการประมวลผล (Legal Basis)

```
สำหรับ Upload Audience:
✅ ความยินยอม (Consent) - ผู้ใช้ให้ความยินยอมผ่านนโยบาย Privacy
✅ การปฏิบัติตามสัญญา - ลูกค้าที่ซื้อสินค้าและต้องการรับการแจ้งเตือน
✅ ประโยชน์อันชอบธรรม - การส่งข้อมูลที่เกี่ยวข้องกับบริการที่ใช้อยู่

สำหรับ IFA Audience (LINE Ads):
✅ ความยินยอมเท่านั้น (Explicit Consent Required)
```

#### 2. สิ่งที่ต้องระวัง

```
❌ อย่าใช้ข้อมูลที่ได้มาโดยไม่ได้รับความยินยอม
❌ อย่าเก็บ User ID โดยไม่แจ้งในนโยบาย Privacy
❌ อย่าส่ง Marketing Message โดยไม่มีตัวเลือก Opt-out
❌ อย่าเก็บข้อมูลนานเกินความจำเป็น
❌ อย่าส่ง Audience ให้บุคคลที่สาม (Third Party) โดยไม่ได้รับความยินยอม
```

#### 3. Best Practices สำหรับ PDPA

```javascript
// ตัวอย่าง: เก็บ Consent ก่อนเพิ่มใน Audience
class PDPACompliantAudienceManager {
  
  async addUserToAudience(userId, audienceGroupId, purpose) {
    // 1. ตรวจสอบว่าผู้ใช้ให้ Consent สำหรับ purpose นี้
    const hasConsent = await this.checkUserConsent(userId, purpose);
    
    if (!hasConsent) {
      console.log(`User ${userId} has not consented to ${purpose}. Skipping.`);
      return false;
    }
    
    // 2. บันทึก Log ว่าเพิ่มใน Audience เพื่ออะไร
    await this.logAudienceAddition(userId, audienceGroupId, purpose);
    
    // 3. เพิ่มใน Audience
    await addUsersToAudience(audienceGroupId, [userId]);
    return true;
  }
  
  async handleOptOut(userId) {
    // เมื่อผู้ใช้ Opt-out ต้องลบออกจากทุก Audience
    const allAudiences = await getAllReadyAudiences();
    
    for (const audience of allAudiences) {
      try {
        await removeUsersFromAudience(audience.audienceGroupId, [userId]);
        console.log(`Removed ${userId} from audience ${audience.description}`);
      } catch (err) {
        // อาจไม่ได้อยู่ใน Audience นั้น - ไม่ต้อง error
      }
    }
    
    // บันทึก Opt-out ใน Database
    await this.recordOptOut(userId);
  }
  
  async checkUserConsent(userId, purpose) {
    // ตรวจสอบใน Database ว่าผู้ใช้ให้ Consent หรือไม่
    // Implementation ขึ้นกับระบบ CRM ของแต่ละบริษัท
    return true; // placeholder
  }
  
  async logAudienceAddition(userId, audienceGroupId, purpose) {
    // บันทึก Log เพื่อ Audit Trail
    console.log(`[AUDIT] Added ${userId} to audience ${audienceGroupId} for purpose: ${purpose}`);
  }
  
  async recordOptOut(userId) {
    // บันทึก Opt-out ใน Database
    console.log(`[AUDIT] User ${userId} opted out at ${new Date().toISOString()}`);
  }
}
```

#### 4. Data Retention Policy

```
Audience Data:
- LINE Platform: 180 วัน (อัตโนมัติ)
- ระบบของคุณ: กำหนดตาม Privacy Policy
  * Recommended: ไม่เกิน 1 ปี สำหรับ Marketing Audience
  * ต้องลบเมื่อผู้ใช้ขอลบข้อมูล (Right to Erasure)

IFA / Tracking Data:
- เก็บได้เฉพาะเมื่อ App อยู่ใน Foreground (iOS 14+)
- ต้องขอ Permission อย่างชัดเจน
```

---

## 14. AudienceManager Class (Node.js)

```javascript
/**
 * AudienceManager - จัดการ LINE Audience API ครบวงจร
 * 
 * ความสามารถ:
 * - สร้าง Upload / Click / Impression / Video View Audience
 * - ดึงรายการและรายละเอียด Audience
 * - อัปเดต (เพิ่ม/ลบ User) และลบ Audience
 * - รองรับ Batch Upload ขนาดใหญ่
 * - รอ Audience พร้อมใช้งาน (Polling)
 * - PDPA-aware operations
 */

const axios = require('axios');

class AudienceManager {
  constructor(channelAccessToken) {
    this.token = channelAccessToken;
    this.baseUrl = 'https://api.line.me/v2/bot/audienceGroup';
    this.narrowcastUrl = 'https://api.line.me/v2/bot/message/narrowcast';
    this.BATCH_SIZE = 10000;
    this.POLL_INTERVAL = 5000; // 5 seconds
    this.MAX_POLL_RETRIES = 60; // 5 minutes total
  }
  
  get headers() {
    return {
      'Authorization': `Bearer ${this.token}`,
      'Content-Type': 'application/json'
    };
  }
  
  // ========== CREATE OPERATIONS ==========
  
  /**
   * สร้าง Upload Audience จาก User IDs
   * @param {string} description - ชื่อ Audience
   * @param {string[]} userIds - รายการ User IDs
   * @param {boolean} isIfaAudience - ใช้ IFA (สำหรับ LINE Ads)
   * @returns {Promise<object>} - Audience object
   */
  async createUploadAudience(description, userIds, isIfaAudience = false) {
    if (userIds.length === 0) {
      throw new Error('userIds must not be empty');
    }
    
    // ถ้ามากกว่า BATCH_SIZE ให้ใช้ createLargeUploadAudience
    if (userIds.length > this.BATCH_SIZE) {
      return this.createLargeUploadAudience(description, userIds, isIfaAudience);
    }
    
    const audiences = userIds.map(id => ({ id }));
    
    const response = await axios.post(
      `${this.baseUrl}/upload`,
      { description, isIfaAudience, audiences },
      { headers: this.headers }
    );
    
    return response.data;
  }
  
  /**
   * สร้าง Upload Audience ขนาดใหญ่ (>10,000 users) แบบแบ่ง batch
   */
  async createLargeUploadAudience(description, allUserIds, isIfaAudience = false) {
    const batches = this._chunkArray(allUserIds, this.BATCH_SIZE);
    console.log(`Splitting ${allUserIds.length} users into ${batches.length} batches`);
    
    // สร้าง Audience ด้วย batch แรก
    const audience = await this.createUploadAudience(
      description,
      batches[0],
      isIfaAudience
    );
    const audienceGroupId = audience.audienceGroupId;
    
    // เพิ่ม batches ที่เหลือ
    for (let i = 1; i < batches.length; i++) {
      console.log(`Adding batch ${i + 1}/${batches.length}...`);
      await this.addUsersToAudience(audienceGroupId, batches[i]);
      await this._sleep(500); // Rate limit protection
    }
    
    console.log(`Large audience created: ${audienceGroupId}`);
    return audience;
  }
  
  /**
   * สร้าง Click Audience จาก Broadcast requestId
   * @param {string} description - ชื่อ Audience
   * @param {string} requestId - requestId ของ Broadcast/Narrowcast
   * @param {string|null} clickUrl - URL ที่ต้องการติดตาม (optional)
   */
  async createClickAudience(description, requestId, clickUrl = null) {
    const body = { description, requestId };
    if (clickUrl) body.clickUrl = clickUrl;
    
    const response = await axios.post(
      `${this.baseUrl}/click`,
      body,
      { headers: this.headers }
    );
    
    return response.data;
  }
  
  /**
   * สร้าง Impression Audience จาก Broadcast requestId
   */
  async createImpressionAudience(description, requestId) {
    const response = await axios.post(
      `${this.baseUrl}/impression`,
      { description, requestId },
      { headers: this.headers }
    );
    
    return response.data;
  }
  
  /**
   * สร้าง Video View Audience
   * @param {string} viewedRate - 'RATE_25' | 'RATE_50' | 'RATE_75' | 'RATE_95'
   */
  async createVideoViewAudience(description, requestId, viewedRate = 'RATE_95') {
    const validRates = ['RATE_25', 'RATE_50', 'RATE_75', 'RATE_95'];
    if (!validRates.includes(viewedRate)) {
      throw new Error(`Invalid viewedRate. Must be one of: ${validRates.join(', ')}`);
    }
    
    const response = await axios.post(
      `${this.baseUrl}/click`,
      { description, requestId, viewedRate },
      { headers: this.headers }
    );
    
    return response.data;
  }
  
  // ========== READ OPERATIONS ==========
  
  /**
   * ดึงรายการ Audience ทั้งหมด
   * @param {object} options - { status, description, createRoute }
   */
  async listAudiences(options = {}) {
    const { status, description, createRoute } = options;
    const allAudiences = [];
    let page = 1;
    let hasNextPage = true;
    
    while (hasNextPage) {
      const params = { page, size: 40 };
      if (status) params.status = status;
      if (description) params.description = description;
      if (createRoute) params.createRoute = createRoute;
      
      const response = await axios.get(
        `${this.baseUrl}/list`,
        { params, headers: this.headers }
      );
      
      allAudiences.push(...response.data.audienceGroups);
      hasNextPage = response.data.hasNextPage;
      page++;
    }
    
    return allAudiences;
  }
  
  /**
   * ดูรายละเอียด Audience
   */
  async getAudienceDetails(audienceGroupId) {
    const response = await axios.get(
      `${this.baseUrl}/${audienceGroupId}`,
      { headers: this.headers }
    );
    
    return response.data;
  }
  
  /**
   * รอจนกว่า Audience จะ READY
   * @param {number} audienceGroupId
   * @returns {Promise<object>} - Audience object when ready
   */
  async waitUntilReady(audienceGroupId) {
    for (let i = 0; i < this.MAX_POLL_RETRIES; i++) {
      const details = await this.getAudienceDetails(audienceGroupId);
      const { status, audienceCount, description } = details.audienceGroup;
      
      if (status === 'READY') {
        console.log(`✓ Audience "${description}" is ready with ${audienceCount} members`);
        return details.audienceGroup;
      }
      
      if (status === 'FAILED') {
        throw new Error(`Audience ${audienceGroupId} (${description}) failed to process`);
      }
      
      if (status === 'EXPIRED') {
        throw new Error(`Audience ${audienceGroupId} has expired`);
      }
      
      console.log(`Waiting for audience ${audienceGroupId}... Status: ${status} (${i + 1}/${this.MAX_POLL_RETRIES})`);
      await this._sleep(this.POLL_INTERVAL);
    }
    
    throw new Error(`Timeout waiting for audience ${audienceGroupId} to be ready`);
  }
  
  // ========== UPDATE OPERATIONS ==========
  
  /**
   * อัปเดตชื่อ Audience
   */
  async updateDescription(audienceGroupId, description) {
    await axios.put(
      `${this.baseUrl}/${audienceGroupId}/updateDescription`,
      { description },
      { headers: this.headers }
    );
    
    return true;
  }
  
  /**
   * เพิ่ม User IDs เข้า Audience
   */
  async addUsersToAudience(audienceGroupId, userIds) {
    if (userIds.length > this.BATCH_SIZE) {
      // แบ่ง batch ถ้ามากเกินไป
      const batches = this._chunkArray(userIds, this.BATCH_SIZE);
      for (const batch of batches) {
        await this._addUsersBatch(audienceGroupId, batch);
        await this._sleep(500);
      }
      return true;
    }
    
    return this._addUsersBatch(audienceGroupId, userIds);
  }
  
  async _addUsersBatch(audienceGroupId, userIds) {
    const audiences = userIds.map(id => ({ id }));
    
    await axios.put(
      `${this.baseUrl}/upload`,
      { audienceGroupId, audiences },
      { headers: this.headers }
    );
    
    return true;
  }
  
  /**
   * ลบ User IDs ออกจาก Audience
   */
  async removeUsersFromAudience(audienceGroupId, userIds) {
    const audiences = userIds.map(id => ({ id }));
    
    await axios.put(
      `${this.baseUrl}/upload/byUpload`,
      { audienceGroupId, audiences },
      { headers: this.headers }
    );
    
    return true;
  }
  
  // ========== DELETE OPERATIONS ==========
  
  /**
   * ลบ Audience
   */
  async deleteAudience(audienceGroupId) {
    await axios.delete(
      `${this.baseUrl}/${audienceGroupId}`,
      { headers: this.headers }
    );
    
    return true;
  }
  
  /**
   * ลบ Audience ที่หมดอายุทั้งหมด
   */
  async cleanupExpiredAudiences() {
    const expired = await this.listAudiences({ status: 'EXPIRED' });
    console.log(`Found ${expired.length} expired audiences to delete`);
    
    let deletedCount = 0;
    for (const audience of expired) {
      try {
        await this.deleteAudience(audience.audienceGroupId);
        console.log(`  Deleted: ${audience.description} (${audience.audienceGroupId})`);
        deletedCount++;
      } catch (err) {
        console.error(`  Failed to delete ${audience.audienceGroupId}:`, err.message);
      }
    }
    
    return deletedCount;
  }
  
  // ========== NARROWCAST OPERATIONS ==========
  
  /**
   * ส่ง Narrowcast ไปยัง Audience
   */
  async narrowcastToAudience(audienceGroupId, messages, demographicFilter = null) {
    const body = {
      messages: Array.isArray(messages) ? messages : [messages],
      recipient: {
        type: 'audienceGroup',
        groupId: audienceGroupId
      }
    };
    
    if (demographicFilter) {
      body.filter = { demographic: demographicFilter };
    }
    
    const response = await axios.post(
      this.narrowcastUrl,
      body,
      {
        headers: {
          ...this.headers,
          'X-Line-Retry-Key': `narrowcast-${audienceGroupId}-${Date.now()}`
        }
      }
    );
    
    return response.headers['x-line-request-id'];
  }
  
  /**
   * ตรวจสอบสถานะ Narrowcast
   */
  async getNarrowcastProgress(requestId) {
    const response = await axios.get(
      `https://api.line.me/v2/bot/message/progress/narrowcast?requestId=${requestId}`,
      { headers: this.headers }
    );
    
    return response.data;
  }
  
  // ========== UTILITY METHODS ==========
  
  /**
   * สร้าง Click Audience แล้วรอจนพร้อม
   */
  async createAndWaitClickAudience(description, requestId, clickUrl = null) {
    const audience = await this.createClickAudience(description, requestId, clickUrl);
    return this.waitUntilReady(audience.audienceGroupId);
  }
  
  /**
   * สรุปสถิติ Audience ทั้งหมด
   */
  async getAudienceSummary() {
    const allAudiences = await this.listAudiences();
    
    const summary = {
      total: allAudiences.length,
      byStatus: {},
      byType: {},
      totalMembers: 0
    };
    
    for (const audience of allAudiences) {
      // นับตาม Status
      summary.byStatus[audience.status] = (summary.byStatus[audience.status] || 0) + 1;
      
      // นับตาม Type
      summary.byType[audience.type] = (summary.byType[audience.type] || 0) + 1;
      
      // รวม Members
      summary.totalMembers += audience.audienceCount || 0;
    }
    
    return summary;
  }
  
  _chunkArray(arr, size) {
    const chunks = [];
    for (let i = 0; i < arr.length; i += size) {
      chunks.push(arr.slice(i, i + size));
    }
    return chunks;
  }
  
  _sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

module.exports = AudienceManager;

// ========== ตัวอย่างการใช้งาน ==========

async function main() {
  const manager = new AudienceManager(process.env.LINE_CHANNEL_ACCESS_TOKEN);
  
  // 1. สร้าง Upload Audience
  const uploadAudience = await manager.createUploadAudience(
    'VIP Customers October 2024',
    [
      'U4af4980629b0fcfcbf22adf96359dbf8',
      'U8edia98062abcdef1234567890abcdef'
    ]
  );
  console.log('Upload Audience ID:', uploadAudience.audienceGroupId);
  
  // 2. สรุป Audience ทั้งหมด
  const summary = await manager.getAudienceSummary();
  console.log('Audience Summary:', JSON.stringify(summary, null, 2));
  
  // 3. ลบ Audience หมดอายุ
  const deleted = await manager.cleanupExpiredAudiences();
  console.log(`Cleaned up ${deleted} expired audiences`);
}
```

---

## 15. AudienceService Class (Python)

```python
"""
AudienceService - LINE Audience API Management Service

ความสามารถ:
- สร้าง Audience หลายประเภท (Upload, Click, Impression, Video)
- จัดการ Audience แบบ Batch สำหรับข้อมูลขนาดใหญ่
- Polling จนกว่า Audience พร้อมใช้งาน
- รองรับ PDPA-compliant operations
- Narrowcast ผ่าน Audience
"""

import os
import time
import logging
from typing import Optional, Union
from dataclasses import dataclass

import requests

logger = logging.getLogger(__name__)


@dataclass
class AudienceInfo:
    audience_group_id: int
    audience_type: str
    description: str
    status: str
    audience_count: int
    created: int
    permission: str
    create_route: str
    is_ifa_audience: bool = False


class AudienceServiceError(Exception):
    """Custom exception for AudienceService"""
    pass


class AudienceService:
    """LINE Audience API Service"""
    
    BASE_URL = "https://api.line.me/v2/bot/audienceGroup"
    NARROWCAST_URL = "https://api.line.me/v2/bot/message/narrowcast"
    BATCH_SIZE = 10_000
    POLL_INTERVAL = 5  # seconds
    MAX_POLL_RETRIES = 60  # 5 minutes
    
    def __init__(self, channel_access_token: str):
        self.token = channel_access_token
        self.session = requests.Session()
        self.session.headers.update({
            "Authorization": f"Bearer {channel_access_token}",
            "Content-Type": "application/json"
        })
    
    # ========== CREATE OPERATIONS ==========
    
    def create_upload_audience(
        self,
        description: str,
        user_ids: list[str],
        is_ifa_audience: bool = False
    ) -> AudienceInfo:
        """
        สร้าง Upload Audience จาก User IDs
        
        Args:
            description: ชื่อ Audience
            user_ids: รายการ LINE User IDs
            is_ifa_audience: ใช้ IFA format (สำหรับ LINE Ads)
        
        Returns:
            AudienceInfo object
        
        Raises:
            AudienceServiceError: เมื่อ API ตอบกลับด้วย error
        """
        if not user_ids:
            raise ValueError("user_ids must not be empty")
        
        if len(user_ids) > self.BATCH_SIZE:
            return self._create_large_upload_audience(description, user_ids, is_ifa_audience)
        
        audiences = [{"id": uid} for uid in user_ids]
        
        response = self.session.post(
            f"{self.BASE_URL}/upload",
            json={
                "description": description,
                "isIfaAudience": is_ifa_audience,
                "audiences": audiences
            }
        )
        
        self._handle_error(response)
        data = response.json()
        
        return self._parse_audience_info(data)
    
    def _create_large_upload_audience(
        self,
        description: str,
        all_user_ids: list[str],
        is_ifa_audience: bool = False
    ) -> AudienceInfo:
        """สร้าง Upload Audience ขนาดใหญ่โดยแบ่ง batch"""
        batches = self._chunk_list(all_user_ids, self.BATCH_SIZE)
        logger.info(f"Splitting {len(all_user_ids)} users into {len(batches)} batches")
        
        # สร้างด้วย batch แรก
        audience = self.create_upload_audience(description, batches[0], is_ifa_audience)
        audience_id = audience.audience_group_id
        
        # เพิ่ม batch ที่เหลือ
        for i, batch in enumerate(batches[1:], start=2):
            logger.info(f"Adding batch {i}/{len(batches)} ({len(batch)} users)...")
            self.add_users_to_audience(audience_id, batch)
            time.sleep(0.5)  # Rate limit protection
        
        logger.info(f"Large audience created: {audience_id}")
        return audience
    
    def create_click_audience(
        self,
        description: str,
        request_id: str,
        click_url: Optional[str] = None
    ) -> AudienceInfo:
        """
        สร้าง Click Audience จากผู้ที่คลิก URL ใน Broadcast
        
        Args:
            description: ชื่อ Audience
            request_id: requestId ของ Broadcast Message
            click_url: URL ที่ต้องการติดตาม (None = ทุก URL)
        """
        body = {"description": description, "requestId": request_id}
        if click_url:
            body["clickUrl"] = click_url
        
        response = self.session.post(f"{self.BASE_URL}/click", json=body)
        self._handle_error(response)
        
        return self._parse_audience_info(response.json())
    
    def create_impression_audience(
        self,
        description: str,
        request_id: str
    ) -> AudienceInfo:
        """สร้าง Impression Audience จากผู้ที่เห็นข้อความ"""
        response = self.session.post(
            f"{self.BASE_URL}/impression",
            json={"description": description, "requestId": request_id}
        )
        self._handle_error(response)
        
        return self._parse_audience_info(response.json())
    
    def create_video_view_audience(
        self,
        description: str,
        request_id: str,
        viewed_rate: str = "RATE_95"
    ) -> AudienceInfo:
        """
        สร้าง Video View Audience
        
        Args:
            viewed_rate: 'RATE_25' | 'RATE_50' | 'RATE_75' | 'RATE_95'
        """
        valid_rates = {"RATE_25", "RATE_50", "RATE_75", "RATE_95"}
        if viewed_rate not in valid_rates:
            raise ValueError(f"Invalid viewed_rate. Must be one of: {valid_rates}")
        
        response = self.session.post(
            f"{self.BASE_URL}/click",
            json={
                "description": description,
                "requestId": request_id,
                "viewedRate": viewed_rate
            }
        )
        self._handle_error(response)
        
        return self._parse_audience_info(response.json())
    
    # ========== READ OPERATIONS ==========
    
    def list_audiences(
        self,
        status: Optional[str] = None,
        description: Optional[str] = None,
        create_route: Optional[str] = None
    ) -> list[AudienceInfo]:
        """
        ดึงรายการ Audience ทั้งหมด (auto-paginate)
        
        Args:
            status: กรองตาม status (IN_PROGRESS|READY|FAILED|EXPIRED)
            description: กรองตามชื่อ
            create_route: กรองตาม route (OA_MANAGER|MESSAGING_API)
        """
        all_audiences = []
        page = 1
        
        while True:
            params = {"page": page, "size": 40}
            if status:
                params["status"] = status
            if description:
                params["description"] = description
            if create_route:
                params["createRoute"] = create_route
            
            response = self.session.get(f"{self.BASE_URL}/list", params=params)
            self._handle_error(response)
            
            data = response.json()
            audiences = [self._parse_audience_info(a) for a in data["audienceGroups"]]
            all_audiences.extend(audiences)
            
            if not data.get("hasNextPage"):
                break
            page += 1
        
        return all_audiences
    
    def get_audience_details(self, audience_group_id: int) -> dict:
        """ดูรายละเอียด Audience รวม Jobs"""
        response = self.session.get(f"{self.BASE_URL}/{audience_group_id}")
        self._handle_error(response)
        
        return response.json()
    
    def wait_until_ready(
        self,
        audience_group_id: int,
        timeout_seconds: int = 300
    ) -> AudienceInfo:
        """
        รอจนกว่า Audience จะ READY
        
        Args:
            audience_group_id: Audience ID
            timeout_seconds: Timeout ในวินาที (default 5 นาที)
        
        Returns:
            AudienceInfo เมื่อ READY
        
        Raises:
            AudienceServiceError: เมื่อ FAILED หรือ timeout
        """
        max_retries = timeout_seconds // self.POLL_INTERVAL
        
        for i in range(max_retries):
            details = self.get_audience_details(audience_group_id)
            audience = details["audienceGroup"]
            status = audience["status"]
            
            if status == "READY":
                logger.info(
                    f"Audience {audience_group_id} ({audience['description']}) "
                    f"is ready with {audience.get('audienceCount', 0)} members"
                )
                return self._parse_audience_info(audience)
            
            if status == "FAILED":
                raise AudienceServiceError(
                    f"Audience {audience_group_id} failed to process"
                )
            
            if status == "EXPIRED":
                raise AudienceServiceError(
                    f"Audience {audience_group_id} has expired"
                )
            
            logger.debug(
                f"Waiting for audience {audience_group_id}... "
                f"Status: {status} ({i + 1}/{max_retries})"
            )
            time.sleep(self.POLL_INTERVAL)
        
        raise AudienceServiceError(
            f"Timeout waiting for audience {audience_group_id} to be ready"
        )
    
    # ========== UPDATE OPERATIONS ==========
    
    def update_description(self, audience_group_id: int, description: str) -> bool:
        """อัปเดตชื่อ Audience"""
        response = self.session.put(
            f"{self.BASE_URL}/{audience_group_id}/updateDescription",
            json={"description": description}
        )
        self._handle_error(response)
        
        return True
    
    def add_users_to_audience(
        self,
        audience_group_id: int,
        user_ids: list[str]
    ) -> bool:
        """เพิ่ม User IDs เข้า Upload Audience"""
        if len(user_ids) > self.BATCH_SIZE:
            batches = self._chunk_list(user_ids, self.BATCH_SIZE)
            for batch in batches:
                self._add_users_batch(audience_group_id, batch)
                time.sleep(0.5)
            return True
        
        return self._add_users_batch(audience_group_id, user_ids)
    
    def _add_users_batch(self, audience_group_id: int, user_ids: list[str]) -> bool:
        audiences = [{"id": uid} for uid in user_ids]
        
        response = self.session.put(
            f"{self.BASE_URL}/upload",
            json={"audienceGroupId": audience_group_id, "audiences": audiences}
        )
        self._handle_error(response)
        
        return True
    
    def remove_users_from_audience(
        self,
        audience_group_id: int,
        user_ids: list[str]
    ) -> bool:
        """ลบ User IDs ออกจาก Upload Audience"""
        audiences = [{"id": uid} for uid in user_ids]
        
        response = self.session.put(
            f"{self.BASE_URL}/upload/byUpload",
            json={"audienceGroupId": audience_group_id, "audiences": audiences}
        )
        self._handle_error(response)
        
        return True
    
    # ========== DELETE OPERATIONS ==========
    
    def delete_audience(self, audience_group_id: int) -> bool:
        """ลบ Audience"""
        response = self.session.delete(f"{self.BASE_URL}/{audience_group_id}")
        self._handle_error(response)
        
        return True
    
    def cleanup_expired_audiences(self) -> int:
        """ลบ Audience ที่หมดอายุทั้งหมด"""
        expired = self.list_audiences(status="EXPIRED")
        logger.info(f"Found {len(expired)} expired audiences to delete")
        
        deleted_count = 0
        for audience in expired:
            try:
                self.delete_audience(audience.audience_group_id)
                logger.info(f"  Deleted: {audience.description} ({audience.audience_group_id})")
                deleted_count += 1
            except AudienceServiceError as e:
                logger.error(f"  Failed to delete {audience.audience_group_id}: {e}")
        
        return deleted_count
    
    # ========== NARROWCAST OPERATIONS ==========
    
    def narrowcast_to_audience(
        self,
        audience_group_id: int,
        messages: Union[dict, list[dict]],
        demographic_filter: Optional[dict] = None
    ) -> str:
        """
        ส่ง Narrowcast ไปยัง Audience
        
        Returns:
            request_id สำหรับติดตามผล
        """
        if isinstance(messages, dict):
            messages = [messages]
        
        body = {
            "messages": messages,
            "recipient": {
                "type": "audienceGroup",
                "groupId": audience_group_id
            }
        }
        
        if demographic_filter:
            body["filter"] = {"demographic": demographic_filter}
        
        response = self.session.post(
            self.NARROWCAST_URL,
            json=body,
            headers={
                "X-Line-Retry-Key": f"narrowcast-{audience_group_id}-{int(time.time())}"
            }
        )
        self._handle_error(response)
        
        return response.headers.get("x-line-request-id", "")
    
    def get_narrowcast_progress(self, request_id: str) -> dict:
        """ตรวจสอบสถานะ Narrowcast"""
        response = self.session.get(
            f"https://api.line.me/v2/bot/message/progress/narrowcast",
            params={"requestId": request_id}
        )
        self._handle_error(response)
        
        return response.json()
    
    # ========== STATISTICS ==========
    
    def get_summary(self) -> dict:
        """สรุปสถิติ Audience ทั้งหมด"""
        all_audiences = self.list_audiences()
        
        summary = {
            "total": len(all_audiences),
            "by_status": {},
            "by_type": {},
            "total_members": 0,
            "ready_audiences": []
        }
        
        for audience in all_audiences:
            # นับตาม Status
            summary["by_status"][audience.status] = (
                summary["by_status"].get(audience.status, 0) + 1
            )
            
            # นับตาม Type
            summary["by_type"][audience.audience_type] = (
                summary["by_type"].get(audience.audience_type, 0) + 1
            )
            
            # รวม Members
            summary["total_members"] += audience.audience_count
            
            # รายการที่พร้อมใช้งาน
            if audience.status == "READY":
                summary["ready_audiences"].append({
                    "id": audience.audience_group_id,
                    "description": audience.description,
                    "count": audience.audience_count
                })
        
        return summary
    
    # ========== PRIVATE HELPERS ==========
    
    def _parse_audience_info(self, data: dict) -> AudienceInfo:
        return AudienceInfo(
            audience_group_id=data.get("audienceGroupId", 0),
            audience_type=data.get("type", ""),
            description=data.get("description", ""),
            status=data.get("status", ""),
            audience_count=data.get("audienceCount", 0),
            created=data.get("created", 0),
            permission=data.get("permission", ""),
            create_route=data.get("createRoute", ""),
            is_ifa_audience=data.get("isIfaAudience", False)
        )
    
    def _handle_error(self, response: requests.Response) -> None:
        if response.status_code >= 400:
            try:
                error_data = response.json()
                message = error_data.get("message", response.text)
            except Exception:
                message = response.text
            
            raise AudienceServiceError(
                f"LINE API error {response.status_code}: {message}"
            )
    
    @staticmethod
    def _chunk_list(lst: list, size: int) -> list[list]:
        return [lst[i:i + size] for i in range(0, len(lst), size)]


# ========== ตัวอย่างการใช้งาน ==========

if __name__ == "__main__":
    import json
    
    service = AudienceService(os.environ["LINE_CHANNEL_ACCESS_TOKEN"])
    
    # สร้าง Upload Audience
    audience = service.create_upload_audience(
        "VIP Customers October 2024",
        [
            "U4af4980629b0fcfcbf22adf96359dbf8",
            "U8edia98062abcdef1234567890abcdef"
        ]
    )
    print(f"Created Audience ID: {audience.audience_group_id}")
    
    # สรุป Audience
    summary = service.get_summary()
    print(f"Total Audiences: {summary['total']}")
    print(f"Total Members: {summary['total_members']:,}")
    print(f"By Status: {json.dumps(summary['by_status'], indent=2)}")
```

---

## 16. ตัวอย่างการใช้งานจริง

### 16.1 Re-engage ผู้ใช้ที่ไม่ Active

**สถานการณ์:** ธุรกิจ E-commerce ต้องการส่งข้อความกระตุ้นลูกค้าที่ไม่ได้ซื้อสินค้าในช่วง 90 วันที่ผ่านมา

**ขั้นตอนการทำงาน:**

```
1. Query ข้อมูลจาก Database
   → ดึง User IDs ที่ last_purchase < (today - 90 days)
   → กรองเฉพาะคนที่ยังเป็น Follower ของ LINE OA

2. สร้าง Audience จาก User IDs
   → สร้าง Upload Audience "Inactive 90 Days"
   → รอจนกว่า Status = READY

3. ส่ง Narrowcast พร้อม Personalization
   → ข้อความ: "นานแล้วที่คุณไม่ได้แวะมา! มีโปรโมชั่นพิเศษสำหรับคุณ"

4. ติดตามผลและปรับปรุง
   → ตรวจสอบ Open Rate และ Click Rate
   → สร้าง Click Audience จาก requestId นั้น
   → Follow-up เฉพาะคนที่คลิกแต่ยังไม่ซื้อ
```

**Node.js Implementation:**

```javascript
const AudienceManager = require('./AudienceManager');
const db = require('./database'); // ตัวอย่าง DB connection

async function reEngageInactiveUsers() {
  const manager = new AudienceManager(process.env.LINE_CHANNEL_ACCESS_TOKEN);
  
  // Step 1: ดึง User IDs ที่ไม่ active
  const ninetyDaysAgo = new Date();
  ninetyDaysAgo.setDate(ninetyDaysAgo.getDate() - 90);
  
  const inactiveUsers = await db.query(`
    SELECT line_user_id 
    FROM customers 
    WHERE last_purchase_date < $1 
      AND line_user_id IS NOT NULL
      AND is_blocked = FALSE
      AND marketing_consent = TRUE
    ORDER BY last_purchase_date ASC
  `, [ninetyDaysAgo]);
  
  const userIds = inactiveUsers.rows.map(r => r.line_user_id);
  console.log(`Found ${userIds.length} inactive users`);
  
  if (userIds.length < 50) {
    console.log('Not enough users for Narrowcast (minimum 50)');
    return;
  }
  
  // Step 2: สร้าง Audience
  const today = new Date().toISOString().split('T')[0];
  const audienceDescription = `Inactive Users 90+ Days - ${today}`;
  
  let audienceId;
  try {
    const audience = await manager.createUploadAudience(audienceDescription, userIds);
    audienceId = audience.audienceGroupId;
    console.log(`Created Audience: ${audienceId}`);
    
    // รอจนพร้อม
    await manager.waitUntilReady(audienceId);
  } catch (err) {
    console.error('Failed to create audience:', err);
    return;
  }
  
  // Step 3: ส่ง Narrowcast
  const message = {
    type: 'flex',
    altText: 'มีโปรโมชั่นพิเศษสำหรับคุณ!',
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: 'เราคิดถึงคุณ! 💖',
            size: 'xl',
            weight: 'bold',
            color: '#ffffff'
          }
        ],
        backgroundColor: '#FF5555'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: 'นานแล้วที่คุณไม่ได้แวะมา\nเรามีโปรโมชั่นพิเศษสำหรับคุณ!',
            wrap: true
          },
          {
            type: 'text',
            text: 'ส่วนลด 20% สำหรับการสั่งซื้อครั้งถัดไป',
            weight: 'bold',
            color: '#FF5555',
            margin: 'md'
          }
        ]
      },
      footer: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'button',
            action: {
              type: 'uri',
              label: 'ช้อปเลย!',
              uri: 'https://example.com/promo?utm_source=line&utm_campaign=reengagement'
            },
            style: 'primary',
            color: '#FF5555'
          }
        ]
      }
    }
  };
  
  try {
    const requestId = await manager.narrowcastToAudience(audienceId, message);
    console.log(`Narrowcast sent! requestId: ${requestId}`);
    
    // บันทึก requestId สำหรับ Follow-up
    await db.query(`
      INSERT INTO campaign_tracking (
        campaign_name, audience_id, request_id, sent_at
      ) VALUES ($1, $2, $3, NOW())
    `, [audienceDescription, audienceId, requestId]);
    
    // รอ 2 วันแล้วสร้าง Click Audience สำหรับ Follow-up
    setTimeout(async () => {
      const clickAudience = await manager.createClickAudience(
        `Clicked Re-engage ${today}`,
        requestId,
        'https://example.com/promo'
      );
      console.log(`Click Audience created: ${clickAudience.audienceGroupId}`);
    }, 2 * 24 * 60 * 60 * 1000); // 2 วัน
    
  } catch (err) {
    console.error('Failed to send Narrowcast:', err);
  }
}

// รัน Cron Job ทุกวันอาทิตย์ เวลา 10:00 น.
// cron.schedule('0 10 * * 0', reEngageInactiveUsers);
reEngageInactiveUsers();
```

---

### 16.2 Target ลูกค้าที่ใช้คูปองสำหรับ Follow-up

**สถานการณ์:** ส่ง Coupon ผ่าน Broadcast แล้วต้องการ Follow-up เฉพาะคนที่ใช้คูปองแต่ยังไม่ได้ซื้อซ้ำ

**Python Implementation:**

```python
import os
import time
from datetime import datetime, timedelta
from AudienceService import AudienceService, AudienceServiceError
import psycopg2  # ตัวอย่าง PostgreSQL

def target_coupon_users_for_followup():
    """
    ส่ง Follow-up ให้ลูกค้าที่ใช้คูปองแต่ยังไม่ซื้อซ้ำใน 7 วัน
    """
    service = AudienceService(os.environ["LINE_CHANNEL_ACCESS_TOKEN"])
    
    # Step 1: ดึง request_id ของ Broadcast Coupon ที่ส่งไปเมื่อ 7 วันที่แล้ว
    # (บันทึกไว้ตอนส่ง Broadcast)
    conn = psycopg2.connect(os.environ["DATABASE_URL"])
    cur = conn.cursor()
    
    seven_days_ago = datetime.now() - timedelta(days=7)
    cur.execute("""
        SELECT request_id, campaign_name 
        FROM broadcast_campaigns 
        WHERE sent_at >= %s 
          AND campaign_type = 'COUPON'
          AND followup_sent = FALSE
    """, (seven_days_ago,))
    
    campaigns = cur.fetchall()
    
    for request_id, campaign_name in campaigns:
        print(f"\nProcessing campaign: {campaign_name}")
        
        # Step 2: สร้าง Click Audience (คนที่คลิกดูคูปอง)
        try:
            click_audience = service.create_click_audience(
                description=f"Clicked Coupon: {campaign_name}",
                request_id=request_id
            )
            
            # รอจนพร้อม
            click_audience = service.wait_until_ready(
                click_audience.audience_group_id,
                timeout_seconds=300
            )
            
            print(f"Click Audience: {click_audience.audience_count} users")
            
        except AudienceServiceError as e:
            print(f"Failed to create click audience: {e}")
            continue
        
        # Step 3: ดึงรายชื่อลูกค้าที่ซื้อแล้ว (Exclude)
        cur.execute("""
            SELECT line_user_id 
            FROM orders 
            WHERE created_at >= %s 
              AND coupon_code IS NOT NULL
              AND line_user_id IS NOT NULL
        """, (seven_days_ago,))
        
        purchased_users = [row[0] for row in cur.fetchall()]
        print(f"Users who already purchased: {len(purchased_users)}")
        
        # Step 4: สร้าง Audience คนที่ซื้อแล้ว (เพื่อ Exclude)
        if len(purchased_users) >= 50:
            try:
                purchased_audience = service.create_upload_audience(
                    description=f"Purchased after Coupon: {campaign_name}",
                    user_ids=purchased_users
                )
                purchased_id = purchased_audience.audience_group_id
                service.wait_until_ready(purchased_id)
            except AudienceServiceError as e:
                print(f"Failed to create purchased audience: {e}")
                purchased_id = None
        else:
            purchased_id = None
        
        # Step 5: ส่ง Narrowcast (Click - Purchased)
        if purchased_id:
            # ส่งให้คนที่คลิกแต่ยังไม่ซื้อ
            recipient = {
                "type": "difference",
                "minuend": {
                    "type": "audienceGroup",
                    "groupId": click_audience.audience_group_id
                },
                "subtrahend": {
                    "type": "audienceGroup",
                    "groupId": purchased_id
                }
            }
        else:
            # ถ้าไม่มีคนซื้อ ส่งให้ทุกคนที่คลิก
            recipient = {
                "type": "audienceGroup",
                "groupId": click_audience.audience_group_id
            }
        
        # ส่งข้อความ Follow-up
        message = {
            "type": "text",
            "text": f"สวัสดีครับ! คูปอง {campaign_name} ของคุณกำลังจะหมดอายุในอีก 3 วัน\nอย่าลืมใช้สิทธิ์นะครับ! 🛒"
        }
        
        try:
            # Narrowcast แบบใช้ recipient ที่กำหนดเอง
            import requests
            headers = {
                "Authorization": f"Bearer {os.environ['LINE_CHANNEL_ACCESS_TOKEN']}",
                "Content-Type": "application/json",
                "X-Line-Retry-Key": f"followup-{request_id}-{int(time.time())}"
            }
            
            resp = requests.post(
                "https://api.line.me/v2/bot/message/narrowcast",
                json={
                    "messages": [message],
                    "recipient": recipient
                },
                headers=headers
            )
            resp.raise_for_status()
            
            followup_request_id = resp.headers.get("x-line-request-id")
            print(f"Follow-up sent! requestId: {followup_request_id}")
            
            # อัปเดต DB ว่าส่ง Follow-up แล้ว
            cur.execute("""
                UPDATE broadcast_campaigns 
                SET followup_sent = TRUE, followup_request_id = %s
                WHERE request_id = %s
            """, (followup_request_id, request_id))
            conn.commit()
            
        except Exception as e:
            print(f"Failed to send follow-up: {e}")
            conn.rollback()
    
    cur.close()
    conn.close()

target_coupon_users_for_followup()
```

---

### 16.3 Exclude ผู้ซื้อล่าสุดออกจากโปรโมชั่น

**สถานการณ์:** ต้องการส่ง Broadcast โปรโมชั่น "ลด 30%" แต่ต้องการ Exclude คนที่เพิ่งซื้อในช่วง 14 วันที่ผ่านมา เพื่อไม่ให้ลูกค้ารู้สึกว่าซื้อแพงกว่าคนอื่น

**Node.js Implementation:**

```javascript
const AudienceManager = require('./AudienceManager');
const db = require('./database');

async function sendPromoExcludingRecentBuyers() {
  const manager = new AudienceManager(process.env.LINE_CHANNEL_ACCESS_TOKEN);
  
  // Step 1: ดึงลูกค้าที่ซื้อใน 14 วันที่ผ่านมา
  const fourteenDaysAgo = new Date();
  fourteenDaysAgo.setDate(fourteenDaysAgo.getDate() - 14);
  
  const recentBuyers = await db.query(`
    SELECT DISTINCT line_user_id 
    FROM orders 
    WHERE created_at > $1 
      AND line_user_id IS NOT NULL
  `, [fourteenDaysAgo]);
  
  const recentBuyerIds = recentBuyers.rows.map(r => r.line_user_id);
  console.log(`Recent buyers to exclude: ${recentBuyerIds.length}`);
  
  let excludeAudienceId = null;
  
  // Step 2: สร้าง Exclude Audience (ถ้ามีพอ)
  if (recentBuyerIds.length >= 50) {
    const excludeAudience = await manager.createUploadAudience(
      'Recent Buyers 14 Days - Exclude',
      recentBuyerIds
    );
    excludeAudienceId = excludeAudience.audienceGroupId;
    await manager.waitUntilReady(excludeAudienceId);
    console.log(`Exclude Audience created: ${excludeAudienceId}`);
  }
  
  // Step 3: เตรียม Narrowcast body
  const promotionMessage = {
    type: 'flex',
    altText: 'โปรโมชั่นพิเศษ! ลด 30% ทุกสินค้า',
    contents: {
      type: 'bubble',
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: '⚡ Flash Sale!',
            size: 'xxl',
            weight: 'bold',
            color: '#FF0000'
          },
          {
            type: 'text',
            text: 'ลด 30% ทุกสินค้า',
            size: 'xl',
            weight: 'bold'
          },
          {
            type: 'text',
            text: 'วันนี้เท่านั้น! หมดเที่ยงคืน',
            color: '#999999'
          }
        ]
      },
      footer: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'button',
            action: {
              type: 'uri',
              label: 'ช้อปเลย!',
              uri: 'https://example.com/flash-sale'
            },
            style: 'primary'
          }
        ]
      }
    }
  };
  
  // Step 4: กำหนด Recipient แบบ Broadcast ทุกคน ยกเว้น Recent Buyers
  let requestId;
  
  if (excludeAudienceId) {
    // Broadcast แบบ Narrowcast ที่ Exclude กลุ่ม
    // ส่งให้ทุก Follower ยกเว้น Recent Buyers
    const response = await axios.post(
      'https://api.line.me/v2/bot/message/narrowcast',
      {
        messages: [promotionMessage],
        recipient: {
          type: 'difference',
          minuend: {
            type: 'all'  // ทุก Follower
          },
          subtrahend: {
            type: 'audienceGroup',
            groupId: excludeAudienceId
          }
        }
      },
      {
        headers: {
          'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
          'Content-Type': 'application/json'
        }
      }
    );
    requestId = response.headers['x-line-request-id'];
  } else {
    // ไม่มีคนซื้อล่าสุดพอ → Broadcast ให้ทุกคนเลย
    const response = await axios.post(
      'https://api.line.me/v2/bot/message/broadcast',
      { messages: [promotionMessage] },
      {
        headers: {
          'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
          'Content-Type': 'application/json'
        }
      }
    );
    requestId = response.headers['x-line-request-id'];
  }
  
  console.log(`Promotion sent! requestId: ${requestId}`);
  
  // Step 5: ติดตามผล
  setTimeout(async () => {
    const progress = await manager.getNarrowcastProgress(requestId);
    console.log('Promotion Results:', progress);
    
    // สร้าง Click Audience สำหรับ Retargeting ในอนาคต
    if (progress.phase === 'succeeded') {
      const clickAudience = await manager.createClickAudience(
        'Clicked Flash Sale',
        requestId,
        'https://example.com/flash-sale'
      );
      console.log(`Created Click Audience for retargeting: ${clickAudience.audienceGroupId}`);
    }
  }, 24 * 60 * 60 * 1000); // 1 วัน
  
  return requestId;
}

sendPromoExcludingRecentBuyers()
  .then(id => console.log('Done! requestId:', id))
  .catch(err => console.error('Error:', err));
```

---

### ตาราง Error Codes ที่พบบ่อย

| Error Code | HTTP Status | ความหมาย | วิธีแก้ไข |
|------------|-------------|----------|----------|
| 401 | 401 | Token ไม่ถูกต้องหรือหมดอายุ | ต่ออายุ Channel Access Token |
| 400 | 400 | Request body ไม่ถูกต้อง | ตรวจสอบ JSON format |
| 404 | 404 | ไม่พบ Audience ID | ตรวจสอบ audienceGroupId |
| 409 | 409 | Audience มีสถานะ EXPIRED | สร้าง Audience ใหม่ |
| 429 | 429 | Rate Limit เกิน | รอแล้วส่งใหม่ |
| 500 | 500 | LINE Server Error | ลองใหม่ในภายหลัง |

---

### Best Practices สรุป

```
✅ DO:
  - ตรวจสอบ Status ก่อนใช้ Audience (ต้องเป็น READY)
  - ลบ Audience ที่ไม่ใช้แล้วเป็นประจำ
  - แบ่ง Upload เป็น batch ไม่เกิน 10,000
  - เก็บ requestId ทุกครั้งที่ Broadcast
  - มี try-catch รอบ API calls ทุกที่
  - บันทึก audit log สำหรับ PDPA compliance
  - ตรวจสอบ audience count ก่อน Narrowcast

❌ DON'T:
  - อย่าใช้ Audience ที่ยังเป็น IN_PROGRESS
  - อย่าเก็บ Audience เกิน 1,000 กลุ่มโดยไม่ลบของเก่า
  - อย่า Narrowcast โดยไม่ตรวจสอบขนาด Audience
  - อย่าเพิ่ม User ที่ไม่ได้ให้ Consent
  - อย่า Share Audience กับบุคคลที่สาม
```

---

## 17. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: สร้าง Audience Pipeline อัตโนมัติ

**โจทย์:** สร้าง Node.js script ที่:
1. ดึงรายชื่อลูกค้าที่สมัครสมาชิกใหม่ใน 7 วันที่ผ่านมาจาก Database
2. สร้าง Upload Audience ชื่อ "New Members - [วันที่]"
3. รอจนกว่า Audience พร้อม
4. ส่ง Welcome Narrowcast
5. บันทึก requestId ลง Database

**Hint:**
```javascript
// โครงสร้างที่แนะนำ
async function newMemberCampaign() {
  // 1. Query DB
  const newMembers = await db.query(/* SQL */);
  
  // 2. สร้าง Audience
  const audience = await manager.createUploadAudience(/* ... */);
  
  // 3. Wait for ready
  await manager.waitUntilReady(/* ... */);
  
  // 4. Send Narrowcast
  const requestId = await manager.narrowcastToAudience(/* ... */);
  
  // 5. Save to DB
  await db.query(/* INSERT */);
}
```

---

### แบบฝึกหัดที่ 2: Audience Analytics Dashboard

**โจทย์:** สร้าง Python script ที่แสดงสรุปรายงาน Audience ทั้งหมด:
- จำนวนกลุ่มแยกตาม Type
- กลุ่มที่ใหญ่ที่สุด 5 กลุ่ม
- กลุ่มที่กำลังจะหมดอายุใน 30 วัน
- Total Reach (ผลรวม audience count ทั้งหมด)

---

### แบบฝึกหัดที่ 3: PDPA-Compliant Opt-out Handler

**โจทย์:** เมื่อผู้ใช้ส่งข้อความ "ยกเลิกการรับข่าวสาร" ให้:
1. บันทึก Opt-out ลง Database
2. ลบ User ID ออกจาก Audience ทุกกลุ่มที่เป็น Upload Audience
3. ส่งข้อความยืนยันกลับ

---

### คำถามทบทวน

1. Click Audience และ Impression Audience ต่างกันอย่างไร? ควรใช้เมื่อไหร่?
2. ทำไม Audience ถึงมีอายุแค่ 180 วัน และควรจัดการอย่างไร?
3. การรวม Audience แบบ Union, Difference, Intersection ใช้สำหรับ Use Case อะไร?
4. ข้อกำหนด PDPA ที่สำคัญในการใช้ Audience API มีอะไรบ้าง?
5. ถ้าต้องการ Upload Audience 500,000 คน ควรทำอย่างไร?

---

## สรุป

Audience API เป็นเครื่องมือที่ทรงพลังสำหรับการทำ Targeted Marketing ผ่าน LINE OA สิ่งสำคัญที่ควรจำ:

| หัวข้อ | สรุป |
|--------|------|
| ประเภท Audience | Upload, Click, Impression, Video View, Chat Tag, Add Friend |
| ขนาดขั้นต่ำ | 50 คน สำหรับ Narrowcast |
| อายุ Audience | 180 วัน (ต้องต่ออายุโดยสร้างใหม่) |
| Batch Upload | แบ่งทีละ 10,000 User IDs |
| PDPA | ต้องมี Consent ก่อนเพิ่มใน Audience |
| Best Practice | เก็บ requestId → สร้าง Click/Impression Audience → Follow-up |

การเข้าใจ Audience API อย่างถ่องแท้จะช่วยให้ธุรกิจสามารถส่งข้อความที่ถูกคน ถูกเวลา ในปริมาณที่เหมาะสม เพิ่ม ROI ของแคมเปญ LINE OA ได้อย่างมีนัยสำคัญ

---

**ส่วนถัดไป:** ส่วนที่ 56 - Narrowcast API ขั้นสูงและ Demographic Targeting

---

*หมายเหตุ: ตัวอย่าง User IDs และ requestIds ในเอกสารนี้เป็น placeholder สำหรับการเรียนรู้เท่านั้น ในการใช้งานจริงต้องใช้ค่าที่ได้จากระบบ LINE จริง*
