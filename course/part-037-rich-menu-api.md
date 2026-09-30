# Part 37: Rich Menu API - การสร้าง Rich Menu ผ่าน API

## บทนำ

Rich Menu คือเมนูที่แสดงอยู่ด้านล่างหน้าจอแชท LINE สามารถกำหนดเองได้ทั้งภาพและ Action ผ่าน Messaging API ช่วยให้ผู้ใช้เข้าถึงฟีเจอร์ต่างๆ ได้อย่างรวดเร็ว

---

## 1. Rich Menu API vs OA Manager Rich Menu

### เปรียบเทียบวิธีสร้าง Rich Menu

| Feature | OA Manager | Messaging API |
|---------|------------|---------------|
| สร้างผ่าน | Web Interface | Code/API |
| ความยืดหยุ่น | จำกัด | สูงมาก |
| Per-user Menu | ไม่รองรับ | รองรับ |
| Dynamic switching | ไม่รองรับ | รองรับ |
| ต้องมี coding skill | ไม่ต้องการ | ต้องการ |
| Scheduled changes | ไม่รองรับ | รองรับ |
| Tabbed Rich Menu | ไม่รองรับ | รองรับ |

### เมื่อไรควรใช้ API

```
ใช้ OA Manager เมื่อ:
├── ต้องการ Rich Menu พื้นฐาน
├── ไม่มี developer
├── เนื้อหาไม่เปลี่ยนแปลง
└── ใช้สำหรับทุกคนเหมือนกัน

ใช้ API เมื่อ:
├── ต้องการ Rich Menu แตกต่างกันต่อผู้ใช้
├── ต้องเปลี่ยน Menu ตามสถานการณ์
├── ต้องการ Tabbed Rich Menu
├── ต้องเปลี่ยน Menu อัตโนมัติ
└── ต้องการ analytics ละเอียด
```

---

## 2. Rich Menu Flow Overview

```
┌─────────────────────────────────────────────────────────┐
│                   Rich Menu API Flow                     │
│                                                          │
│  1. สร้าง Rich Menu Object (POST /v2/bot/richmenu)      │
│                    │                                     │
│                    ▼                                     │
│  2. Upload ภาพ Rich Menu                                │
│     (POST /v2/bot/richmenu/{richMenuId}/content)        │
│                    │                                     │
│                    ▼                                     │
│  3. (ทางเลือก) ตั้งเป็น Default Menu                   │
│     (POST /v2/bot/user/all/richmenu/{richMenuId})       │
│                    │                                     │
│                    ▼                                     │
│  4. (ทางเลือก) Link ให้ User เฉพาะราย                  │
│     (POST /v2/bot/user/{userId}/richmenu/{richMenuId})  │
└─────────────────────────────────────────────────────────┘
```

---

## 3. โครงสร้าง Rich Menu Object

### Rich Menu JSON Structure

```json
{
  "size": {
    "width": 2500,
    "height": 1686
  },
  "selected": true,
  "name": "Main Menu",
  "chatBarText": "เปิดเมนู",
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
        "label": "สินค้า",
        "text": "ดูสินค้า"
      }
    },
    {
      "bounds": {
        "x": 833,
        "y": 0,
        "width": 834,
        "height": 843
      },
      "action": {
        "type": "uri",
        "label": "เว็บไซต์",
        "uri": "https://example.com"
      }
    }
  ]
}
```

### Field Descriptions

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `size` | Object | Yes | ขนาดของ Rich Menu |
| `size.width` | Integer | Yes | ความกว้าง (2500) |
| `size.height` | Integer | Yes | ความสูง (843 หรือ 1686) |
| `selected` | Boolean | Yes | แสดงเมนูทันทีหรือไม่ |
| `name` | String | Yes | ชื่อ Rich Menu (แสดงใน OA Manager) |
| `chatBarText` | String | Yes | ข้อความบน Chat Bar |
| `areas` | Array | Yes | พื้นที่และ Action |

---

## 4. ขนาดภาพ Rich Menu

### Size Options

```
Full Size (2 rows):
┌──────────────────────────────────┐
│                                  │ ← Height: 1686px
│         Rich Menu Full           │
│                                  │
└──────────────────────────────────┘
   Width: 2500px

Half Size (1 row):
┌──────────────────────────────────┐
│       Rich Menu Half             │ ← Height: 843px
└──────────────────────────────────┘
   Width: 2500px
```

| Size | Width | Height | Rows |
|------|-------|--------|------|
| Full | 2500 | 1686 | 2 |
| Half | 2500 | 843 | 1 |

### ข้อกำหนดของภาพ

| ข้อกำหนด | รายละเอียด |
|-----------|------------|
| รูปแบบ | JPEG หรือ PNG |
| ขนาดไฟล์ | ไม่เกิน 1MB |
| Resolution | 2500x1686 หรือ 2500x843 |
| Color Space | RGB |

---

## 5. Area Bounds และ Layout

### การกำหนด Grid Layout

```
Full Size (2500 x 1686) - 6 Cells:
┌──────────┬──────────┬──────────┐
│  (0,0)   │ (833,0)  │(1667,0)  │ Row 1
│ 833x843  │ 834x843  │ 833x843  │
├──────────┼──────────┼──────────┤
│  (0,843) │(833,843) │(1667,843)│ Row 2
│ 833x843  │ 834x843  │ 833x843  │
└──────────┴──────────┴──────────┘

Half Size (2500 x 843) - 3 Cells:
┌──────────┬──────────┬──────────┐
│  (0,0)   │ (833,0)  │(1667,0)  │
│ 833x843  │ 834x843  │ 833x843  │
└──────────┴──────────┴──────────┘
```

### ตัวอย่าง 6-Cell Layout

```json
{
  "areas": [
    {
      "bounds": { "x": 0, "y": 0, "width": 833, "height": 843 },
      "action": { "type": "message", "text": "เซลล์ 1" }
    },
    {
      "bounds": { "x": 833, "y": 0, "width": 834, "height": 843 },
      "action": { "type": "message", "text": "เซลล์ 2" }
    },
    {
      "bounds": { "x": 1667, "y": 0, "width": 833, "height": 843 },
      "action": { "type": "message", "text": "เซลล์ 3" }
    },
    {
      "bounds": { "x": 0, "y": 843, "width": 833, "height": 843 },
      "action": { "type": "message", "text": "เซลล์ 4" }
    },
    {
      "bounds": { "x": 833, "y": 843, "width": 834, "height": 843 },
      "action": { "type": "message", "text": "เซลล์ 5" }
    },
    {
      "bounds": { "x": 1667, "y": 843, "width": 833, "height": 843 },
      "action": { "type": "message", "text": "เซลล์ 6" }
    }
  ]
}
```

---

## 6. Action Types สำหรับ Rich Menu

Rich Menu รองรับ Action ดังนี้:

### 6.1 Message Action
```json
{
  "type": "message",
  "label": "สินค้า",
  "text": "ดูสินค้าทั้งหมด"
}
```

### 6.2 URI Action
```json
{
  "type": "uri",
  "label": "เว็บไซต์",
  "uri": "https://example.com/shop"
}
```

### 6.3 Postback Action
```json
{
  "type": "postback",
  "label": "โปรไฟล์",
  "data": "action=view_profile",
  "displayText": "ดูโปรไฟล์ของฉัน"
}
```

### 6.4 Richmenu Switch Action (Tabbed Menu)
```json
{
  "type": "richmenuswitch",
  "richMenuAliasId": "richmenu-alias-shopping",
  "data": "tab=shopping"
}
```

### 6.5 Datetime Picker Action
```json
{
  "type": "datetimepicker",
  "label": "จอง",
  "data": "action=booking",
  "mode": "date"
}
```

### 6.6 Camera/CameraRoll/Location Actions
```json
{
  "type": "camera",
  "label": "ถ่ายภาพ"
}
```

---

## 7. การสร้าง Rich Menu ด้วย Node.js

### 7.1 Setup

```javascript
// richMenuManager.js
const axios = require('axios');
const fs = require('fs');
const FormData = require('form-data');
const path = require('path');

const BASE_URL = 'https://api.line.me/v2/bot';
const headers = {
  Authorization: `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
  'Content-Type': 'application/json',
};

class RichMenuManager {
  constructor(accessToken) {
    this.accessToken = accessToken;
    this.baseHeaders = {
      Authorization: `Bearer ${accessToken}`,
      'Content-Type': 'application/json',
    };
  }

  // สร้าง Rich Menu
  async createRichMenu(richMenuData) {
    try {
      const response = await axios.post(
        `${BASE_URL}/richmenu`,
        richMenuData,
        { headers: this.baseHeaders }
      );
      console.log('✅ Rich Menu created:', response.data.richMenuId);
      return response.data.richMenuId;
    } catch (error) {
      console.error('❌ Error creating rich menu:', error.response?.data);
      throw error;
    }
  }

  // Upload ภาพ Rich Menu
  async uploadRichMenuImage(richMenuId, imagePath) {
    try {
      const form = new FormData();
      form.append('image', fs.createReadStream(imagePath));

      const ext = path.extname(imagePath).toLowerCase();
      const contentType = ext === '.png' ? 'image/png' : 'image/jpeg';

      const response = await axios.post(
        `${BASE_URL}/richmenu/${richMenuId}/content`,
        form,
        {
          headers: {
            Authorization: `Bearer ${this.accessToken}`,
            'Content-Type': contentType,
            ...form.getHeaders(),
          },
        }
      );
      console.log('✅ Image uploaded successfully');
      return response.data;
    } catch (error) {
      console.error('❌ Error uploading image:', error.response?.data);
      throw error;
    }
  }

  // ตั้ง Default Rich Menu
  async setDefaultRichMenu(richMenuId) {
    try {
      await axios.post(
        `${BASE_URL}/user/all/richmenu/${richMenuId}`,
        {},
        { headers: this.baseHeaders }
      );
      console.log('✅ Default Rich Menu set:', richMenuId);
    } catch (error) {
      console.error('❌ Error setting default rich menu:', error.response?.data);
      throw error;
    }
  }

  // Link Rich Menu ให้ User
  async linkRichMenuToUser(userId, richMenuId) {
    try {
      await axios.post(
        `${BASE_URL}/user/${userId}/richmenu/${richMenuId}`,
        {},
        { headers: this.baseHeaders }
      );
      console.log(`✅ Rich Menu ${richMenuId} linked to user ${userId}`);
    } catch (error) {
      console.error('❌ Error linking rich menu:', error.response?.data);
      throw error;
    }
  }

  // ยกเลิก Link Rich Menu จาก User
  async unlinkRichMenuFromUser(userId) {
    try {
      await axios.delete(
        `${BASE_URL}/user/${userId}/richmenu`,
        { headers: this.baseHeaders }
      );
      console.log(`✅ Rich Menu unlinked from user ${userId}`);
    } catch (error) {
      console.error('❌ Error unlinking rich menu:', error.response?.data);
      throw error;
    }
  }

  // ดูรายการ Rich Menu ทั้งหมด
  async listRichMenus() {
    try {
      const response = await axios.get(
        `${BASE_URL}/richmenu/list`,
        { headers: this.baseHeaders }
      );
      return response.data.richmenus;
    } catch (error) {
      console.error('❌ Error listing rich menus:', error.response?.data);
      throw error;
    }
  }

  // ดู Rich Menu ของ User
  async getUserRichMenu(userId) {
    try {
      const response = await axios.get(
        `${BASE_URL}/user/${userId}/richmenu`,
        { headers: this.baseHeaders }
      );
      return response.data;
    } catch (error) {
      if (error.response?.status === 404) {
        return null; // ไม่มี Rich Menu
      }
      throw error;
    }
  }

  // ลบ Rich Menu
  async deleteRichMenu(richMenuId) {
    try {
      await axios.delete(
        `${BASE_URL}/richmenu/${richMenuId}`,
        { headers: this.baseHeaders }
      );
      console.log('✅ Rich Menu deleted:', richMenuId);
    } catch (error) {
      console.error('❌ Error deleting rich menu:', error.response?.data);
      throw error;
    }
  }

  // ดู Default Rich Menu
  async getDefaultRichMenu() {
    try {
      const response = await axios.get(
        `${BASE_URL}/user/all/richmenu`,
        { headers: this.baseHeaders }
      );
      return response.data;
    } catch (error) {
      if (error.response?.status === 404) {
        return null;
      }
      throw error;
    }
  }

  // ยกเลิก Default Rich Menu
  async cancelDefaultRichMenu() {
    try {
      await axios.delete(
        `${BASE_URL}/user/all/richmenu`,
        { headers: this.baseHeaders }
      );
      console.log('✅ Default Rich Menu cancelled');
    } catch (error) {
      console.error('❌ Error cancelling default rich menu:', error.response?.data);
      throw error;
    }
  }
}

module.exports = RichMenuManager;
```

---

## 8. สร้าง Rich Menu แบบ Full Implementation

```javascript
// createMainRichMenu.js
const RichMenuManager = require('./richMenuManager');

const manager = new RichMenuManager(process.env.LINE_CHANNEL_ACCESS_TOKEN);

// กำหนด Rich Menu Data สำหรับเมนูหลัก
const mainMenuData = {
  size: {
    width: 2500,
    height: 1686,
  },
  selected: true,
  name: 'Main Menu - Full',
  chatBarText: '📋 เปิดเมนู',
  areas: [
    // Row 1, Column 1: สินค้า
    {
      bounds: { x: 0, y: 0, width: 833, height: 843 },
      action: {
        type: 'postback',
        label: 'สินค้า',
        data: 'action=products',
        displayText: 'ดูสินค้าทั้งหมด',
      },
    },
    // Row 1, Column 2: โปรโมชั่น
    {
      bounds: { x: 833, y: 0, width: 834, height: 843 },
      action: {
        type: 'postback',
        label: 'โปรโมชั่น',
        data: 'action=promotions',
        displayText: 'ดูโปรโมชั่น',
      },
    },
    // Row 1, Column 3: คำสั่งซื้อ
    {
      bounds: { x: 1667, y: 0, width: 833, height: 843 },
      action: {
        type: 'postback',
        label: 'คำสั่งซื้อ',
        data: 'action=orders',
        displayText: 'ดูคำสั่งซื้อของฉัน',
      },
    },
    // Row 2, Column 1: ติดต่อเรา
    {
      bounds: { x: 0, y: 843, width: 833, height: 843 },
      action: {
        type: 'uri',
        label: 'ติดต่อเรา',
        uri: 'https://example.com/contact',
      },
    },
    // Row 2, Column 2: เว็บไซต์
    {
      bounds: { x: 833, y: 843, width: 834, height: 843 },
      action: {
        type: 'uri',
        label: 'เว็บไซต์',
        uri: 'https://example.com',
      },
    },
    // Row 2, Column 3: โปรไฟล์
    {
      bounds: { x: 1667, y: 843, width: 833, height: 843 },
      action: {
        type: 'postback',
        label: 'โปรไฟล์',
        data: 'action=profile',
        displayText: 'ดูโปรไฟล์ของฉัน',
      },
    },
  ],
};

async function setupMainRichMenu() {
  try {
    console.log('🚀 Creating Main Rich Menu...');

    // Step 1: สร้าง Rich Menu
    const richMenuId = await manager.createRichMenu(mainMenuData);
    console.log('Rich Menu ID:', richMenuId);

    // Step 2: Upload ภาพ
    await manager.uploadRichMenuImage(richMenuId, './assets/main-menu.png');

    // Step 3: ตั้งเป็น Default
    await manager.setDefaultRichMenu(richMenuId);

    console.log('✅ Main Rich Menu setup complete!');
    return richMenuId;
  } catch (error) {
    console.error('❌ Setup failed:', error.message);
    process.exit(1);
  }
}

setupMainRichMenu().then(id => {
  console.log(`Rich Menu ID: ${id}`);
});
```

---

## 9. Per-User Rich Menu (ตามประเภทผู้ใช้)

### แนวคิด Per-User Rich Menu

```
ผู้ใช้ทั่วไป (Guest):
┌────────┬────────┬────────┐
│ สมัคร │  FAQ   │ ติดต่อ │
└────────┴────────┴────────┘

สมาชิก (Member):
┌────────┬────────┬────────┐
│สินค้า │ คำสั่ง │ คะแนน  │
└────────┴────────┴────────┘

VIP Member:
┌────────┬────────┬────────┐
│ VIP   │ คะแนน │ ของรางวัล│
└────────┴────────┴────────┘
```

### Implementation

```javascript
// userRichMenuManager.js
const RichMenuManager = require('./richMenuManager');
const db = require('./database'); // สมมติมี database module

const manager = new RichMenuManager(process.env.LINE_CHANNEL_ACCESS_TOKEN);

// เก็บ ID ของ Rich Menu แต่ละประเภท
let richMenuIds = {
  guest: null,
  member: null,
  vip: null,
};

// สร้าง Rich Menu สำหรับ Guest
const guestMenuData = {
  size: { width: 2500, height: 843 },
  selected: true,
  name: 'Guest Menu',
  chatBarText: '📋 เมนู',
  areas: [
    {
      bounds: { x: 0, y: 0, width: 833, height: 843 },
      action: {
        type: 'postback',
        label: 'สมัครสมาชิก',
        data: 'action=register',
      },
    },
    {
      bounds: { x: 833, y: 0, width: 834, height: 843 },
      action: {
        type: 'postback',
        label: 'FAQ',
        data: 'action=faq',
      },
    },
    {
      bounds: { x: 1667, y: 0, width: 833, height: 843 },
      action: {
        type: 'uri',
        label: 'ติดต่อเรา',
        uri: 'https://example.com/contact',
      },
    },
  ],
};

// สร้าง Rich Menu สำหรับ Member
const memberMenuData = {
  size: { width: 2500, height: 1686 },
  selected: true,
  name: 'Member Menu',
  chatBarText: '🛍️ เมนูสมาชิก',
  areas: [
    {
      bounds: { x: 0, y: 0, width: 833, height: 843 },
      action: { type: 'postback', label: 'สินค้า', data: 'action=products' },
    },
    {
      bounds: { x: 833, y: 0, width: 834, height: 843 },
      action: { type: 'postback', label: 'คำสั่งซื้อ', data: 'action=orders' },
    },
    {
      bounds: { x: 1667, y: 0, width: 833, height: 843 },
      action: { type: 'postback', label: 'คะแนน', data: 'action=points' },
    },
    {
      bounds: { x: 0, y: 843, width: 833, height: 843 },
      action: { type: 'postback', label: 'โปรไฟล์', data: 'action=profile' },
    },
    {
      bounds: { x: 833, y: 843, width: 834, height: 843 },
      action: { type: 'postback', label: 'โปรโมชั่น', data: 'action=promotions' },
    },
    {
      bounds: { x: 1667, y: 843, width: 833, height: 843 },
      action: { type: 'postback', label: 'ติดต่อ', data: 'action=support' },
    },
  ],
};

// สร้าง Rich Menu สำหรับ VIP
const vipMenuData = {
  size: { width: 2500, height: 1686 },
  selected: true,
  name: 'VIP Menu',
  chatBarText: '⭐ VIP Menu',
  areas: [
    {
      bounds: { x: 0, y: 0, width: 1250, height: 843 },
      action: { type: 'postback', label: 'VIP Special', data: 'action=vip_special' },
    },
    {
      bounds: { x: 1250, y: 0, width: 1250, height: 843 },
      action: { type: 'postback', label: 'คะแนน VIP', data: 'action=vip_points' },
    },
    {
      bounds: { x: 0, y: 843, width: 833, height: 843 },
      action: { type: 'postback', label: 'ของรางวัล', data: 'action=rewards' },
    },
    {
      bounds: { x: 833, y: 843, width: 834, height: 843 },
      action: { type: 'postback', label: 'Priority Support', data: 'action=priority_support' },
    },
    {
      bounds: { x: 1667, y: 843, width: 833, height: 843 },
      action: { type: 'postback', label: 'บัญชี', data: 'action=account' },
    },
  ],
};

// เริ่มต้น สร้าง Rich Menu ทั้งหมด
async function initializeRichMenus() {
  console.log('🚀 Initializing all Rich Menus...');

  // สร้าง Guest Menu
  richMenuIds.guest = await manager.createRichMenu(guestMenuData);
  await manager.uploadRichMenuImage(richMenuIds.guest, './assets/guest-menu.png');
  console.log('✅ Guest menu created:', richMenuIds.guest);

  // สร้าง Member Menu
  richMenuIds.member = await manager.createRichMenu(memberMenuData);
  await manager.uploadRichMenuImage(richMenuIds.member, './assets/member-menu.png');
  console.log('✅ Member menu created:', richMenuIds.member);

  // สร้าง VIP Menu
  richMenuIds.vip = await manager.createRichMenu(vipMenuData);
  await manager.uploadRichMenuImage(richMenuIds.vip, './assets/vip-menu.png');
  console.log('✅ VIP menu created:', richMenuIds.vip);

  // ตั้ง Guest เป็น Default
  await manager.setDefaultRichMenu(richMenuIds.guest);
  console.log('✅ Guest menu set as default');

  return richMenuIds;
}

// อัปเดต Rich Menu เมื่อ User เปลี่ยน Tier
async function updateUserRichMenu(userId, newTier) {
  const menuId = richMenuIds[newTier];

  if (!menuId) {
    console.error(`❌ No rich menu found for tier: ${newTier}`);
    return;
  }

  await manager.linkRichMenuToUser(userId, menuId);
  console.log(`✅ User ${userId} assigned to ${newTier} menu`);
}

// ตรวจสอบและตั้ง Rich Menu เมื่อ User Follow
async function onUserFollow(userId) {
  const userTier = await db.getUserTier(userId);

  if (userTier && richMenuIds[userTier]) {
    await manager.linkRichMenuToUser(userId, richMenuIds[userTier]);
  }
  // ถ้าไม่มี tier ให้ใช้ default (guest)
}

module.exports = {
  initializeRichMenus,
  updateUserRichMenu,
  onUserFollow,
  richMenuIds,
};
```

---

## 10. Tabbed Rich Menu (Rich Menu Alias)

Tabbed Rich Menu ช่วยให้สามารถสลับระหว่างเมนูได้

### สร้าง Rich Menu Alias

```javascript
// tabbedRichMenu.js

// สร้าง Rich Menu Alias
async function createRichMenuAlias(richMenuId, aliasId) {
  try {
    const response = await axios.post(
      'https://api.line.me/v2/bot/richmenu/alias',
      {
        richMenuAliasId: aliasId,
        richMenuId: richMenuId,
      },
      {
        headers: {
          Authorization: `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
          'Content-Type': 'application/json',
        },
      }
    );
    console.log('✅ Alias created:', aliasId);
    return response.data;
  } catch (error) {
    console.error('❌ Error creating alias:', error.response?.data);
    throw error;
  }
}

// อัปเดต Rich Menu Alias
async function updateRichMenuAlias(aliasId, newRichMenuId) {
  try {
    await axios.put(
      `https://api.line.me/v2/bot/richmenu/alias/${aliasId}`,
      { richMenuId: newRichMenuId },
      {
        headers: {
          Authorization: `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
          'Content-Type': 'application/json',
        },
      }
    );
    console.log('✅ Alias updated:', aliasId);
  } catch (error) {
    console.error('❌ Error updating alias:', error.response?.data);
    throw error;
  }
}

// Rich Menu แบบ Tabbed (แท็บ 1: หน้าหลัก, แท็บ 2: สินค้า)
const tab1MenuData = {
  size: { width: 2500, height: 1686 },
  selected: true,
  name: 'Tab 1 - Home',
  chatBarText: '🏠 หน้าหลัก',
  areas: [
    // แท็บนำทาง - Tab 1 (Active)
    {
      bounds: { x: 0, y: 0, width: 1250, height: 250 },
      action: {
        type: 'richmenuswitch',
        richMenuAliasId: 'tab1-menu',
        data: 'tab=1',
      },
    },
    // แท็บนำทาง - Tab 2
    {
      bounds: { x: 1250, y: 0, width: 1250, height: 250 },
      action: {
        type: 'richmenuswitch',
        richMenuAliasId: 'tab2-menu',
        data: 'tab=2',
      },
    },
    // เนื้อหา Row 1
    {
      bounds: { x: 0, y: 250, width: 833, height: 716 },
      action: { type: 'postback', label: 'สินค้าใหม่', data: 'action=new_products' },
    },
    {
      bounds: { x: 833, y: 250, width: 834, height: 716 },
      action: { type: 'postback', label: 'โปรโมชั่น', data: 'action=promotions' },
    },
    {
      bounds: { x: 1667, y: 250, width: 833, height: 716 },
      action: { type: 'postback', label: 'ติดต่อ', data: 'action=contact' },
    },
    // เนื้อหา Row 2
    {
      bounds: { x: 0, y: 966, width: 833, height: 720 },
      action: { type: 'postback', label: 'คำสั่งซื้อ', data: 'action=orders' },
    },
    {
      bounds: { x: 833, y: 966, width: 834, height: 720 },
      action: { type: 'postback', label: 'คะแนน', data: 'action=points' },
    },
    {
      bounds: { x: 1667, y: 966, width: 833, height: 720 },
      action: { type: 'postback', label: 'บัญชี', data: 'action=account' },
    },
  ],
};

// Setup Tabbed Menu
async function setupTabbedMenu() {
  const manager = new RichMenuManager(process.env.LINE_CHANNEL_ACCESS_TOKEN);

  // สร้าง Tab 1
  const tab1Id = await manager.createRichMenu(tab1MenuData);
  await manager.uploadRichMenuImage(tab1Id, './assets/tab1-menu.png');

  // สร้าง Tab 2 (คล้ายกัน แต่เนื้อหาต่างกัน)
  const tab2MenuData = { ...tab1MenuData, name: 'Tab 2 - Products' };
  const tab2Id = await manager.createRichMenu(tab2MenuData);
  await manager.uploadRichMenuImage(tab2Id, './assets/tab2-menu.png');

  // สร้าง Aliases
  await createRichMenuAlias(tab1Id, 'tab1-menu');
  await createRichMenuAlias(tab2Id, 'tab2-menu');

  // ตั้ง Tab 1 เป็น Default
  await manager.setDefaultRichMenu(tab1Id);

  console.log('✅ Tabbed menu setup complete!');
}
```

---

## 11. Dynamic Rich Menu Switching

```javascript
// dynamicMenuSwitcher.js
const line = require('@line/bot-sdk');

const client = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
});

// เปลี่ยน Rich Menu ตาม User State
async function switchMenuByState(userId, state) {
  const menuMapping = {
    browsing: process.env.RICHMENU_ID_BROWSING,
    shopping: process.env.RICHMENU_ID_SHOPPING,
    checkout: process.env.RICHMENU_ID_CHECKOUT,
    member: process.env.RICHMENU_ID_MEMBER,
    guest: process.env.RICHMENU_ID_GUEST,
  };

  const menuId = menuMapping[state];
  if (!menuId) {
    console.warn(`⚠️ No menu found for state: ${state}`);
    return;
  }

  await client.linkRichMenuToUser(userId, menuId);
  console.log(`✅ Switched user ${userId} to ${state} menu`);
}

// เปลี่ยนเมนูกลุ่มผู้ใช้ (Bulk)
async function bulkSwitchMenu(userIds, menuId) {
  const promises = userIds.map(userId =>
    client.linkRichMenuToUser(userId, menuId).catch(err => {
      console.error(`❌ Failed for user ${userId}:`, err.message);
    })
  );

  await Promise.all(promises);
  console.log(`✅ Bulk menu switch complete for ${userIds.length} users`);
}

// เปลี่ยน Menu ตามเวลา (Time-based)
async function scheduleMenuChange() {
  const now = new Date();
  const hour = now.getHours();

  let menuId;
  if (hour >= 9 && hour < 12) {
    menuId = process.env.RICHMENU_MORNING;
  } else if (hour >= 12 && hour < 14) {
    menuId = process.env.RICHMENU_LUNCH;
  } else if (hour >= 14 && hour < 18) {
    menuId = process.env.RICHMENU_AFTERNOON;
  } else {
    menuId = process.env.RICHMENU_DEFAULT;
  }

  // ตั้ง Default Menu ตามเวลา
  if (menuId) {
    const manager = new RichMenuManager(process.env.LINE_CHANNEL_ACCESS_TOKEN);
    await manager.setDefaultRichMenu(menuId);
    console.log(`✅ Menu changed to ${menuId} for hour ${hour}`);
  }
}

// Webhook Handler
const express = require('express');
const app = express();

app.post('/webhook', line.middleware({
  channelSecret: process.env.LINE_CHANNEL_SECRET,
}), async (req, res) => {
  const events = req.body.events;

  await Promise.all(events.map(async (event) => {
    const userId = event.source.userId;

    if (event.type === 'follow') {
      // ตรวจสอบ user tier แล้วเลือก menu
      await switchMenuByState(userId, 'guest');

    } else if (event.type === 'postback') {
      const data = new URLSearchParams(event.postback.data);
      const action = data.get('action');

      if (action === 'start_checkout') {
        await switchMenuByState(userId, 'checkout');
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: '🛒 เริ่มกระบวนการชำระเงิน...',
        });
      } else if (action === 'cancel_checkout') {
        await switchMenuByState(userId, 'shopping');
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: '↩️ ยกเลิกการชำระเงิน กลับสู่หน้าร้าน',
        });
      }
    }
  }));

  res.json({ status: 'ok' });
});
```

---

## 12. การสร้างภาพ Rich Menu ด้วย Canvas (Node.js)

```javascript
// generateRichMenuImage.js
const { createCanvas, loadImage } = require('canvas');
const fs = require('fs');

async function generateRichMenuImage(options) {
  const { width = 2500, height = 1686, outputPath = './output/richmenu.png' } = options;

  const canvas = createCanvas(width, height);
  const ctx = canvas.getContext('2d');

  // พื้นหลัง
  ctx.fillStyle = '#FFFFFF';
  ctx.fillRect(0, 0, width, height);

  // วาด Grid
  const cellWidth = Math.floor(width / 3);
  const cellHeight = Math.floor(height / 2);

  const cells = [
    { x: 0,          y: 0,          text: '🛍️ สินค้า',    color: '#FF6B6B' },
    { x: cellWidth,  y: 0,          text: '🎁 โปรโมชั่น', color: '#4ECDC4' },
    { x: cellWidth*2,y: 0,          text: '📦 คำสั่งซื้อ', color: '#45B7D1' },
    { x: 0,          y: cellHeight, text: '📞 ติดต่อ',     color: '#96CEB4' },
    { x: cellWidth,  y: cellHeight, text: '🌐 เว็บไซต์',   color: '#FFEAA7' },
    { x: cellWidth*2,y: cellHeight, text: '👤 โปรไฟล์',    color: '#DDA0DD' },
  ];

  for (const cell of cells) {
    // วาดพื้นหลัง cell
    ctx.fillStyle = cell.color;
    ctx.fillRect(cell.x + 5, cell.y + 5, cellWidth - 10, cellHeight - 10);

    // วาดข้อความ
    ctx.fillStyle = '#FFFFFF';
    ctx.font = 'bold 80px Arial';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';
    ctx.fillText(
      cell.text,
      cell.x + cellWidth / 2,
      cell.y + cellHeight / 2
    );
  }

  // วาดเส้นแบ่ง
  ctx.strokeStyle = '#FFFFFF';
  ctx.lineWidth = 10;

  // เส้นแนวตั้ง
  ctx.beginPath();
  ctx.moveTo(cellWidth, 0);
  ctx.lineTo(cellWidth, height);
  ctx.stroke();

  ctx.beginPath();
  ctx.moveTo(cellWidth * 2, 0);
  ctx.lineTo(cellWidth * 2, height);
  ctx.stroke();

  // เส้นแนวนอน
  ctx.beginPath();
  ctx.moveTo(0, cellHeight);
  ctx.lineTo(width, cellHeight);
  ctx.stroke();

  // บันทึกไฟล์
  const buffer = canvas.toBuffer('image/png');
  fs.writeFileSync(outputPath, buffer);

  console.log(`✅ Rich Menu image generated: ${outputPath}`);
  return outputPath;
}

// สร้างภาพพร้อม Icon จาก URL
async function generateRichMenuWithIcons(iconUrls, outputPath) {
  const width = 2500;
  const height = 1686;
  const canvas = createCanvas(width, height);
  const ctx = canvas.getContext('2d');

  ctx.fillStyle = '#1E3A5F';
  ctx.fillRect(0, 0, width, height);

  const cellWidth = Math.floor(width / 3);
  const cellHeight = Math.floor(height / 2);

  for (let i = 0; i < 6; i++) {
    const col = i % 3;
    const row = Math.floor(i / 3);
    const x = col * cellWidth;
    const y = row * cellHeight;

    // วาด cell background
    const gradient = ctx.createLinearGradient(x, y, x + cellWidth, y + cellHeight);
    gradient.addColorStop(0, '#2C5282');
    gradient.addColorStop(1, '#1A365D');
    ctx.fillStyle = gradient;
    ctx.fillRect(x + 8, y + 8, cellWidth - 16, cellHeight - 16);

    // โหลดและวาด icon
    if (iconUrls[i]) {
      try {
        const img = await loadImage(iconUrls[i]);
        const iconSize = 200;
        ctx.drawImage(
          img,
          x + (cellWidth - iconSize) / 2,
          y + cellHeight / 4,
          iconSize,
          iconSize
        );
      } catch (err) {
        console.warn(`⚠️ Failed to load icon ${i}:`, err.message);
      }
    }
  }

  const buffer = canvas.toBuffer('image/png');
  fs.writeFileSync(outputPath, buffer);
  console.log(`✅ Rich Menu with icons generated: ${outputPath}`);
}

module.exports = { generateRichMenuImage, generateRichMenuWithIcons };
```

---

## 13. Complete Setup Script

```javascript
// setup.js - Script สำหรับ setup ทุกอย่าง
require('dotenv').config();
const RichMenuManager = require('./richMenuManager');
const { generateRichMenuImage } = require('./generateRichMenuImage');

async function fullSetup() {
  const manager = new RichMenuManager(process.env.LINE_CHANNEL_ACCESS_TOKEN);

  console.log('🚀 Starting Full Rich Menu Setup...\n');

  // Step 1: ลบ Rich Menu เก่า
  console.log('📋 Step 1: Listing existing rich menus...');
  const existing = await manager.listRichMenus();
  console.log(`Found ${existing.length} existing rich menus`);

  for (const menu of existing) {
    await manager.deleteRichMenu(menu.richMenuId);
    console.log(`🗑️ Deleted: ${menu.richMenuId}`);
  }

  // Step 2: สร้างภาพ Rich Menu
  console.log('\n📋 Step 2: Generating rich menu images...');
  await generateRichMenuImage({
    width: 2500,
    height: 1686,
    outputPath: './assets/main-menu.png',
  });

  // Step 3: สร้าง Rich Menu
  console.log('\n📋 Step 3: Creating rich menus...');

  const mainMenuId = await manager.createRichMenu({
    size: { width: 2500, height: 1686 },
    selected: true,
    name: 'Main Menu v2',
    chatBarText: '📋 เปิดเมนู',
    areas: [
      { bounds: { x: 0, y: 0, width: 833, height: 843 }, action: { type: 'postback', label: 'สินค้า', data: 'action=products' } },
      { bounds: { x: 833, y: 0, width: 834, height: 843 }, action: { type: 'postback', label: 'โปรโมชั่น', data: 'action=promotions' } },
      { bounds: { x: 1667, y: 0, width: 833, height: 843 }, action: { type: 'postback', label: 'คำสั่งซื้อ', data: 'action=orders' } },
      { bounds: { x: 0, y: 843, width: 833, height: 843 }, action: { type: 'uri', label: 'ติดต่อ', uri: 'https://example.com/contact' } },
      { bounds: { x: 833, y: 843, width: 834, height: 843 }, action: { type: 'uri', label: 'เว็บไซต์', uri: 'https://example.com' } },
      { bounds: { x: 1667, y: 843, width: 833, height: 843 }, action: { type: 'postback', label: 'โปรไฟล์', data: 'action=profile' } },
    ],
  });

  // Step 4: Upload ภาพ
  console.log('\n📋 Step 4: Uploading images...');
  await manager.uploadRichMenuImage(mainMenuId, './assets/main-menu.png');

  // Step 5: ตั้ง Default
  console.log('\n📋 Step 5: Setting default rich menu...');
  await manager.setDefaultRichMenu(mainMenuId);

  console.log('\n✅ Setup Complete!');
  console.log(`Main Menu ID: ${mainMenuId}`);

  // บันทึก IDs
  const config = {
    mainMenuId,
    setupDate: new Date().toISOString(),
  };

  require('fs').writeFileSync(
    './richmenu-config.json',
    JSON.stringify(config, null, 2)
  );

  console.log('📄 Config saved to richmenu-config.json');
}

fullSetup().catch(console.error);
```

---

## 14. Troubleshooting

### ปัญหาที่พบบ่อย

**1. ภาพอัปโหลดไม่ได้**
```javascript
// ตรวจสอบขนาดและรูปแบบภาพ
const sharp = require('sharp');

async function validateImage(imagePath) {
  const metadata = await sharp(imagePath).metadata();
  
  console.log('Image info:', {
    width: metadata.width,
    height: metadata.height,
    format: metadata.format,
    size: metadata.size,
  });

  const issues = [];
  if (metadata.width !== 2500) issues.push(`Width should be 2500, got ${metadata.width}`);
  if (![843, 1686].includes(metadata.height)) issues.push(`Height should be 843 or 1686, got ${metadata.height}`);
  if (!['jpeg', 'png'].includes(metadata.format)) issues.push(`Format should be jpeg or png, got ${metadata.format}`);

  if (issues.length > 0) {
    console.error('❌ Image issues:', issues);
    return false;
  }

  console.log('✅ Image is valid');
  return true;
}
```

**2. Area Bounds ผิด**
```javascript
// ตรวจสอบว่า areas ไม่ overlap กัน
function validateAreas(areas, menuWidth, menuHeight) {
  // สร้าง grid เพื่อตรวจสอบ
  const grid = Array(menuHeight).fill(null).map(() => Array(menuWidth).fill(0));

  for (let i = 0; i < areas.length; i++) {
    const { x, y, width, height } = areas[i].bounds;
    
    for (let row = y; row < y + height; row++) {
      for (let col = x; col < x + width; col++) {
        if (grid[row]?.[col] > 0) {
          console.error(`❌ Area ${i} overlaps with area ${grid[row][col] - 1}`);
          return false;
        }
        if (grid[row]) grid[row][col] = i + 1;
      }
    }
  }
  
  console.log('✅ All areas are valid');
  return true;
}
```

---

## สรุป

Rich Menu API ให้ความยืดหยุ่นสูงในการสร้างและจัดการ Rich Menu:

1. **Default Rich Menu** - ใช้สำหรับผู้ใช้ทั่วไป
2. **Per-user Rich Menu** - กำหนดเฉพาะรายบุคคล
3. **Tabbed Rich Menu** - สลับระหว่างเมนูได้
4. **Dynamic Switching** - เปลี่ยนเมนูตามสถานการณ์

**Key Points:**
- ภาพต้องขนาด 2500x1686 หรือ 2500x843
- รูปแบบ PNG หรือ JPEG ไม่เกิน 1MB
- กำหนด Areas ได้อย่างอิสระ
- รองรับ Action หลายประเภท
