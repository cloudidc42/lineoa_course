# Part 55: Audience API

## สารบัญ

1. [Audience API Overview](#audience-api-overview)
2. [Audience Types](#audience-types)
3. [Upload Audience (User IDs)](#upload-audience-user-ids)
4. [Click Audience](#click-audience)
5. [Impression Audience](#impression-audience)
6. [Video View Audience](#video-view-audience)
7. [Chat Tag Audience](#chat-tag-audience)
8. [Add Friend Audience](#add-friend-audience)
9. [สร้าง Audience ผ่าน API](#สร้าง-audience-ผ่าน-api)
10. [อัพเดต Audience](#อัพเดต-audience)
11. [ลบ Audience](#ลบ-audience)
12. [ใช้ Audience ใน Narrowcast](#ใช้-audience-ใน-narrowcast)
13. [Audience Size และ Limits](#audience-size-และ-limits)
14. [Node.js Examples](#nodejs-examples)

---

## 1. Audience API Overview

Audience API ช่วยให้สามารถกำหนดกลุ่มเป้าหมายในการส่งข้อความผ่าน Narrowcast ได้อย่างแม่นยำ

### Audience คืออะไร?

Audience คือกลุ่มผู้ใช้ที่ถูกจัดกลุ่มตามเกณฑ์ต่างๆ เช่น:
- ผู้ใช้ที่กดปุ่มบางปุ่ม
- ผู้ใช้ที่เคยดูข้อความนั้น
- ผู้ใช้ที่เพิ่มเพื่อนในช่วงเวลาหนึ่ง
- ผู้ใช้ที่มี Tag กำหนด

### การใช้งาน Audience

```
Audience → Narrowcast → เฉพาะกลุ่มที่กำหนด

ตัวอย่าง:
- ส่งโปรโมชัน VIP → Audience: Users ที่ซื้อ > 3 ครั้ง
- ส่ง Re-engagement → Audience: Users ที่ไม่ Active 30 วัน
- ส่ง Product Update → Audience: Users ที่คลิก Product ในข้อความก่อน
```

### Endpoints

```
Base URL: https://api.line.me/v2/bot

GET    /audienceGroup/{audienceGroupId}        - ดู Audience
POST   /audienceGroup/upload                   - สร้าง Upload Audience
PUT    /audienceGroup/{id}/uploadAudience       - เพิ่ม Users
DELETE /audienceGroup/{audienceGroupId}        - ลบ Audience
GET    /audienceGroups                         - ดูรายการ
PUT    /audienceGroup/{id}                     - อัพเดตชื่อ/คำอธิบาย
POST   /audienceGroup/click                    - สร้าง Click Audience
POST   /audienceGroup/imp                      - สร้าง Impression Audience
POST   /audienceGroup/video                    - สร้าง Video Audience
PUT    /audienceGroup/{id}/share               - แชร์กับ Account อื่น
```

---

## 2. Audience Types

### ประเภท Audience ทั้งหมด

| ประเภท | วิธีสร้าง | ใช้งาน |
|--------|----------|--------|
| `UPLOAD` | อัพโหลด User IDs | กลุ่มที่กำหนดเอง |
| `CLICK` | จาก Message Click | ผู้ใช้ที่คลิก URL |
| `IMP` | จาก Message View | ผู้ใช้ที่เห็นข้อความ |
| `VIDEO` | จาก Video View | ผู้ใช้ที่ดู Video |
| `CHAT_TAG` | จาก Chat Tag | ผู้ใช้ที่ถูก Tag |
| `FRIEND_PATH` | จาก Add Friend | ผู้ใช้ที่เพิ่มเพื่อน |

---

## 3. Upload Audience (User IDs)

### สร้าง Upload Audience

```javascript
const axios = require('axios');
require('dotenv').config();

const LINE_API = 'https://api.line.me/v2/bot';
const headers = {
  'Authorization': `Bearer ${process.env.CHANNEL_ACCESS_TOKEN}`,
  'Content-Type': 'application/json'
};

// สร้าง Audience จาก User IDs
async function createUploadAudience(name, userIds, description = '') {
  const payload = {
    description: name,
    isIfaAudience: false, // false = User ID, true = IFA (iOS Advertising ID)
    audiences: userIds.map(id => ({ id }))
  };

  try {
    const response = await axios.post(
      `${LINE_API}/audienceGroup/upload`,
      payload,
      { headers }
    );
    return response.data;
  } catch (error) {
    console.error('Create audience error:', error.response?.data);
    throw error;
  }
}

// Response
/*
{
  "audienceGroupId": 4389303728991,
  "type": "UPLOAD",
  "description": "VIP Customers",
  "created": 1609459200,
  "permission": "READ_WRITE",
  "createRoute": "OA_MANAGER",
  "status": "READY",
  "audienceCount": 1500
}
*/
```

### Upload Audience แบบ Batch

```javascript
// สร้าง Audience พร้อมกำหนดจำนวน Batch
async function createLargeUploadAudience(name, allUserIds) {
  const BATCH_SIZE = 10000; // สูงสุด 10,000 ต่อ Request

  // สร้าง Audience ก่อนด้วย userIds แรก
  const firstBatch = allUserIds.slice(0, BATCH_SIZE);
  const audience = await createUploadAudience(name, firstBatch);

  console.log(`Created audience: ${audience.audienceGroupId}`);

  // เพิ่ม userIds ที่เหลือ
  for (let i = BATCH_SIZE; i < allUserIds.length; i += BATCH_SIZE) {
    const batch = allUserIds.slice(i, i + BATCH_SIZE);
    await addUsersToAudience(audience.audienceGroupId, batch);
    console.log(`Added batch ${Math.ceil(i/BATCH_SIZE) + 1}: ${batch.length} users`);
    await new Promise(r => setTimeout(r, 1000)); // Rate limiting
  }

  return audience;
}

// เพิ่ม Users เข้า Audience ที่มีอยู่
async function addUsersToAudience(audienceGroupId, userIds) {
  const payload = {
    audienceGroupId,
    audiences: userIds.map(id => ({ id }))
  };

  const response = await axios.put(
    `${LINE_API}/audienceGroup/${audienceGroupId}/uploadAudience`,
    payload,
    { headers }
  );
  return response.data;
}
```

### Upload Audience จาก Database

```javascript
async function syncVIPAudience() {
  try {
    // ดึง VIP Users จาก Database
    const vipUsers = await User.find({
      memberStatus: 'vip',
      lineUserId: { $exists: true, $ne: null },
      notificationConsent: true
    }).select('lineUserId');

    const userIds = vipUsers.map(u => u.lineUserId);
    console.log(`Found ${userIds.length} VIP users`);

    if (userIds.length === 0) {
      console.log('No VIP users to sync');
      return;
    }

    // ตรวจสอบว่า Audience มีอยู่แล้วหรือไม่
    const existingAudience = await findAudienceByDescription('VIP Customers');

    if (existingAudience) {
      // อัพเดต Audience ที่มีอยู่
      console.log(`Updating existing audience: ${existingAudience.audienceGroupId}`);
      // ลบเก่า สร้างใหม่ (เพราะไม่มี "replace all" API)
      await deleteAudience(existingAudience.audienceGroupId);
    }

    // สร้าง Audience ใหม่
    const audience = await createLargeUploadAudience('VIP Customers', userIds);
    console.log(`VIP Audience synced: ${audience.audienceGroupId}`);

    // บันทึก Audience ID ลง config
    await SystemConfig.set('vipAudienceId', audience.audienceGroupId.toString());

    return audience;
  } catch (error) {
    console.error('Sync audience error:', error);
    throw error;
  }
}

async function findAudienceByDescription(description) {
  const audiences = await listAllAudiences();
  return audiences.find(a => a.description === description);
}
```

---

## 4. Click Audience

Click Audience สร้างจากผู้ใช้ที่คลิก URL ใน Narrowcast/Broadcast message

### สร้าง Click Audience

```javascript
async function createClickAudience(name, requestId, clickUrl = null) {
  const payload = {
    description: name,
    requestId, // ID ของ Narrowcast/Broadcast Message
  };

  if (clickUrl) {
    payload.clickUrl = clickUrl; // URL เฉพาะที่ต้องการ track
  }

  const response = await axios.post(
    `${LINE_API}/audienceGroup/click`,
    payload,
    { headers }
  );
  return response.data;
}

// ตัวอย่าง: สร้าง Click Audience หลังส่ง Narrowcast
async function sendNarrowcastAndTrack(audienceGroupId, message) {
  // ส่ง Narrowcast
  const narrowcastResult = await sendNarrowcast(audienceGroupId, message);
  const requestId = narrowcastResult.requestId;

  // รอให้ข้อความส่งสำเร็จก่อน (ประมาณ 1-2 ชั่วโมง)
  console.log(`Narrowcast sent with requestId: ${requestId}`);
  console.log('Creating click audience after 2 hours...');

  // Delay 2 ชั่วโมง (ใน Production ควรใช้ Scheduled Job)
  setTimeout(async () => {
    const clickAudience = await createClickAudience(
      `Clicked - Campaign ${Date.now()}`,
      requestId
    );
    console.log('Click Audience created:', clickAudience.audienceGroupId);
  }, 2 * 60 * 60 * 1000);

  return requestId;
}
```

### ดู Click Audience Details

```javascript
async function getAudienceDetails(audienceGroupId) {
  const response = await axios.get(
    `${LINE_API}/audienceGroup/${audienceGroupId}`,
    { headers }
  );
  return response.data;
}

// Response สำหรับ Click Audience
/*
{
  "audienceGroup": {
    "audienceGroupId": 4389303728992,
    "type": "CLICK",
    "description": "Clicked - Campaign",
    "status": "READY",
    "audienceCount": 350,
    "created": 1609459200,
    "requestId": "5b59bac5-c800-1234-5678-9abc12de3456",
    "clickUrl": "https://example.com/product"
  }
}
*/
```

---

## 5. Impression Audience

Impression Audience สร้างจากผู้ใช้ที่เห็น (receive) Narrowcast message

```javascript
async function createImpressionAudience(name, requestId) {
  const payload = {
    description: name,
    requestId // ID ของ Narrowcast Message
  };

  const response = await axios.post(
    `${LINE_API}/audienceGroup/imp`,
    payload,
    { headers }
  );
  return response.data;
}

// ใช้งาน: Re-target ผู้ใช้ที่เห็น Promo แล้วแต่ไม่ซื้อ
async function createRemarketingAudiences(campaignRequestId) {
  try {
    // สร้าง Impression Audience (คนที่เห็น)
    const impressionAudience = await createImpressionAudience(
      'Saw Promo Campaign',
      campaignRequestId
    );

    // สร้าง Click Audience (คนที่คลิก)
    const clickAudience = await createClickAudience(
      'Clicked Promo Campaign',
      campaignRequestId
    );

    console.log('Impression Audience:', impressionAudience.audienceGroupId);
    console.log('Click Audience:', clickAudience.audienceGroupId);

    // กลยุทธ์:
    // - ส่ง Follow-up ให้คนที่เห็นแต่ไม่คลิก
    // - ส่ง Reminder ให้คนที่คลิกแต่ไม่ซื้อ

    return { impressionAudience, clickAudience };
  } catch (error) {
    console.error('Error creating remarketing audiences:', error);
    throw error;
  }
}
```

---

## 6. Video View Audience

Video View Audience สร้างจากผู้ใช้ที่ดู Video Message

```javascript
async function createVideoViewAudience(name, requestId, viewedMilliseconds) {
  const payload = {
    description: name,
    requestId,
    viewedMilliseconds // ดูนานแค่ไหน (milliseconds)
  };

  const response = await axios.post(
    `${LINE_API}/audienceGroup/video`,
    payload,
    { headers }
  );
  return response.data;
}

// ตัวอย่าง: สร้าง Video Audience ตามเปอร์เซ็นที่ดู
async function createVideoEngagementAudiences(videoRequestId, videoDurationMs) {
  const audiences = {};

  // 25% ของวีดีโอ (Engaged viewers)
  audiences.quarter = await createVideoViewAudience(
    'Video 25% Viewed',
    videoRequestId,
    Math.floor(videoDurationMs * 0.25)
  );

  // 50% ของวีดีโอ (Mid viewers)
  audiences.half = await createVideoViewAudience(
    'Video 50% Viewed',
    videoRequestId,
    Math.floor(videoDurationMs * 0.5)
  );

  // 75% ของวีดีโอ (Highly engaged)
  audiences.threeQuarter = await createVideoViewAudience(
    'Video 75% Viewed',
    videoRequestId,
    Math.floor(videoDurationMs * 0.75)
  );

  // 95% ของวีดีโอ (Completed viewers)
  audiences.completed = await createVideoViewAudience(
    'Video Completed',
    videoRequestId,
    Math.floor(videoDurationMs * 0.95)
  );

  return audiences;
}
```

---

## 7. Chat Tag Audience

Chat Tag Audience สร้างจากผู้ใช้ที่มี Chat Tag กำหนด

```javascript
// Chat Tag ต้องสร้างใน LINE OA Manager ก่อน
// Chat Tag ใช้ระบุประเภทของผู้ใช้

// หมายเหตุ: Chat Tag Audience สร้างผ่าน LINE OA Manager (Web Console)
// ไม่มี API โดยตรงสำหรับสร้าง Chat Tag Audience แต่สามารถ
// ดึงมาใช้ใน Narrowcast ได้

// สร้าง Chat Tag ผ่าน API (Tagging Users)
async function tagUser(userId, chatTag) {
  const response = await axios.post(
    `https://api.line.me/v2/bot/message/tag`,
    {
      userId,
      tagId: chatTag.id
    },
    { headers }
  );
  return response.data;
}

// ดู Chat Tags ทั้งหมด
async function getChatTags() {
  const response = await axios.get(
    'https://api.line.me/v2/bot/message/tags',
    { headers }
  );
  return response.data;
}

// ตัวอย่างระบบ Auto-tagging
async function autoTagUserByBehavior(userId, behavior) {
  const tags = await getChatTags();

  const behaviorTagMap = {
    'purchased': 'tag-purchased',
    'vip': 'tag-vip',
    'inactive_30d': 'tag-inactive',
    'high_value': 'tag-high-value',
    'new_user': 'tag-new'
  };

  const tagName = behaviorTagMap[behavior];
  const tag = tags.tags?.find(t => t.name === tagName);

  if (tag) {
    await tagUser(userId, tag);
    console.log(`Tagged ${userId} as ${tagName}`);
  }
}
```

---

## 8. Add Friend Audience

Add Friend Audience สร้างจากผู้ใช้ที่เพิ่มเพื่อนในช่วงเวลาหนึ่ง

```javascript
// Add Friend Audience สร้างผ่าน LINE OA Manager (Console)
// แต่สามารถดึงมาใช้ใน API ได้

// ดูรายการ Friend Path (แหล่งที่ผู้ใช้เพิ่มเพื่อน)
async function getFriendPaths() {
  const audiences = await listAllAudiences();
  return audiences.filter(a => a.type === 'FRIEND_PATH');
}

// ตัวอย่าง: ส่ง Welcome message ให้กับ New Friends
async function sendWelcomeToNewFriends() {
  const friendPathAudiences = await getFriendPaths();

  // หา Audience ของ New Friends ใน 7 วันที่ผ่านมา
  const sevenDaysAgo = Date.now() - 7 * 24 * 60 * 60 * 1000;
  const recentAudiences = friendPathAudiences.filter(
    a => a.created * 1000 >= sevenDaysAgo
  );

  if (recentAudiences.length === 0) {
    console.log('No recent Friend Path audiences');
    return;
  }

  for (const audience of recentAudiences) {
    await sendNarrowcast(
      audience.audienceGroupId,
      buildWelcomeMessage()
    );
    console.log(`Sent welcome to ${audience.audienceGroupId}`);
  }
}

function buildWelcomeMessage() {
  return {
    type: 'flex',
    altText: 'ยินดีต้อนรับ!',
    contents: {
      type: 'bubble',
      hero: {
        type: 'image',
        url: 'https://example.com/welcome-banner.jpg',
        size: 'full',
        aspectRatio: '20:13',
        aspectMode: 'cover'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: 'ยินดีต้อนรับ! 🎉',
            weight: 'bold',
            size: 'xl'
          },
          {
            type: 'text',
            text: 'ขอบคุณที่เพิ่มเป็นเพื่อน รับของขวัญต้อนรับได้เลย!',
            wrap: true,
            color: '#666666',
            margin: 'sm'
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
              label: 'รับของขวัญ',
              uri: 'https://yourshop.com/welcome-gift'
            },
            style: 'primary',
            color: '#00B900'
          }
        ]
      }
    }
  };
}
```

---

## 9. สร้าง Audience ผ่าน API

### Audience Manager Class

```javascript
// AudienceManager.js
class AudienceManager {
  constructor(channelAccessToken) {
    this.token = channelAccessToken || process.env.CHANNEL_ACCESS_TOKEN;
    this.apiUrl = 'https://api.line.me/v2/bot';
    this.headers = {
      'Authorization': `Bearer ${this.token}`,
      'Content-Type': 'application/json'
    };
  }

  async _request(method, path, data = null) {
    const config = {
      method,
      url: `${this.apiUrl}${path}`,
      headers: this.headers
    };
    if (data) config.data = data;

    try {
      const response = await axios(config);
      return response.data;
    } catch (error) {
      const errMsg = error.response?.data?.message || error.message;
      throw new Error(`Audience API Error: ${errMsg}`);
    }
  }

  // ─── Create Methods ────────────────────────────────────────────

  async createUpload(description, userIds = []) {
    if (userIds.length > 10000) {
      throw new Error('Maximum 10,000 users per request');
    }

    return await this._request('POST', '/audienceGroup/upload', {
      description,
      isIfaAudience: false,
      audiences: userIds.map(id => ({ id }))
    });
  }

  async createClick(description, requestId, clickUrl = null) {
    const payload = { description, requestId };
    if (clickUrl) payload.clickUrl = clickUrl;
    return await this._request('POST', '/audienceGroup/click', payload);
  }

  async createImpression(description, requestId) {
    return await this._request('POST', '/audienceGroup/imp', {
      description,
      requestId
    });
  }

  async createVideoView(description, requestId, viewedMilliseconds) {
    return await this._request('POST', '/audienceGroup/video', {
      description,
      requestId,
      viewedMilliseconds
    });
  }

  // ─── Read Methods ────────────────────────────────────────────

  async get(audienceGroupId) {
    return await this._request('GET', `/audienceGroup/${audienceGroupId}`);
  }

  async list(options = {}) {
    const {
      page = 1,
      description,
      status,
      size = 40,
      includeExternalPublicGroups = true,
      createRoute
    } = options;

    const params = new URLSearchParams({ page, size, includeExternalPublicGroups });
    if (description) params.append('description', description);
    if (status) params.append('status', status);
    if (createRoute) params.append('createRoute', createRoute);

    return await this._request('GET', `/audienceGroups?${params}`);
  }

  async listAll(options = {}) {
    const allAudiences = [];
    let page = 1;
    let hasMore = true;

    while (hasMore) {
      const result = await this.list({ ...options, page });
      allAudiences.push(...(result.audienceGroups || []));

      hasMore = result.hasNextPage || false;
      page++;

      if (page > 100) break; // Safety limit
    }

    return allAudiences;
  }

  // ─── Update Methods ────────────────────────────────────────────

  async addUsers(audienceGroupId, userIds) {
    const batchSize = 10000;
    const results = [];

    for (let i = 0; i < userIds.length; i += batchSize) {
      const batch = userIds.slice(i, i + batchSize);
      const result = await this._request(
        'PUT',
        `/audienceGroup/${audienceGroupId}/uploadAudience`,
        {
          audienceGroupId,
          audiences: batch.map(id => ({ id }))
        }
      );
      results.push(result);

      if (i + batchSize < userIds.length) {
        await new Promise(r => setTimeout(r, 500));
      }
    }

    return results;
  }

  async updateDescription(audienceGroupId, description) {
    return await this._request('PUT', `/audienceGroup/${audienceGroupId}`, {
      description
    });
  }

  // ─── Delete Methods ────────────────────────────────────────────

  async delete(audienceGroupId) {
    return await this._request('DELETE', `/audienceGroup/${audienceGroupId}`);
  }

  async deleteAll(options = {}) {
    const audiences = await this.listAll(options);
    const results = [];

    for (const audience of audiences) {
      try {
        await this.delete(audience.audienceGroupId);
        results.push({ id: audience.audienceGroupId, success: true });
      } catch (error) {
        results.push({
          id: audience.audienceGroupId,
          success: false,
          error: error.message
        });
      }
    }

    return results;
  }

  // ─── Share Methods ────────────────────────────────────────────

  async share(audienceGroupId) {
    return await this._request('PUT', `/audienceGroup/${audienceGroupId}/share`);
  }

  // ─── Utility Methods ────────────────────────────────────────────

  async findByDescription(description) {
    const audiences = await this.listAll();
    return audiences.find(a => a.description === description);
  }

  async upsert(description, userIds) {
    const existing = await this.findByDescription(description);

    if (existing) {
      // ลบเก่า สร้างใหม่ (เพราะไม่มี replace API)
      await this.delete(existing.audienceGroupId);
    }

    return await this.createUpload(description, userIds.slice(0, 10000));
  }

  // ตรวจสอบสถานะ Audience
  async waitForReady(audienceGroupId, maxWait = 30 * 60 * 1000) {
    const startTime = Date.now();
    const checkInterval = 30000; // 30 วินาที

    while (Date.now() - startTime < maxWait) {
      const audience = await this.get(audienceGroupId);
      const status = audience.audienceGroup?.status;

      if (status === 'READY') return audience;
      if (status === 'FAILED') throw new Error(`Audience ${audienceGroupId} failed`);

      console.log(`Audience status: ${status}. Waiting...`);
      await new Promise(r => setTimeout(r, checkInterval));
    }

    throw new Error(`Audience ${audienceGroupId} not ready after ${maxWait/60000} minutes`);
  }
}

module.exports = AudienceManager;
```

---

## 10. อัพเดต Audience

```javascript
const audienceManager = new AudienceManager();

// อัพเดตชื่อ/คำอธิบาย
async function renameAudience(audienceGroupId, newDescription) {
  try {
    await audienceManager.updateDescription(audienceGroupId, newDescription);
    console.log(`Renamed audience ${audienceGroupId} to "${newDescription}"`);
  } catch (error) {
    console.error('Rename error:', error);
  }
}

// อัพเดต Users ใน Audience
async function refreshAudience(audienceGroupId, newUserIds) {
  try {
    console.log(`Adding ${newUserIds.length} users to audience ${audienceGroupId}`);
    const results = await audienceManager.addUsers(audienceGroupId, newUserIds);
    console.log(`Added users successfully`);
    return results;
  } catch (error) {
    console.error('Refresh audience error:', error);
    throw error;
  }
}

// Sync Audience กับ Database ทุกวัน
const cron = require('node-cron');

cron.schedule('0 2 * * *', async () => {
  console.log('Daily audience sync started...');

  try {
    // Sync VIP Audience
    const vipUsers = await User.find({
      memberStatus: 'vip',
      lineUserId: { $exists: true }
    }).select('lineUserId');
    const vipIds = vipUsers.map(u => u.lineUserId);

    const vipAudienceId = process.env.VIP_AUDIENCE_ID;
    if (vipAudienceId) {
      await refreshAudience(vipAudienceId, vipIds);
    } else {
      const audience = await audienceManager.createUpload('VIP Members', vipIds);
      process.env.VIP_AUDIENCE_ID = audience.audienceGroupId;
    }

    // Sync Inactive Users (ไม่ active 30 วัน)
    const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
    const inactiveUsers = await User.find({
      lastActiveAt: { $lt: thirtyDaysAgo },
      lineUserId: { $exists: true }
    }).select('lineUserId');
    const inactiveIds = inactiveUsers.map(u => u.lineUserId);

    await audienceManager.upsert('Inactive 30 Days', inactiveIds);

    console.log('Daily audience sync completed');
  } catch (error) {
    console.error('Audience sync failed:', error);
  }
});
```

---

## 11. ลบ Audience

```javascript
// ลบ Audience เดียว
async function deleteAudience(audienceGroupId) {
  try {
    await audienceManager.delete(audienceGroupId);
    console.log(`Deleted audience: ${audienceGroupId}`);
    return true;
  } catch (error) {
    console.error('Delete error:', error.message);
    return false;
  }
}

// ลบ Audiences ที่หมดอายุ
async function cleanupOldAudiences(maxAgeDays = 90) {
  const audiences = await audienceManager.listAll();
  const maxAgeMs = maxAgeDays * 24 * 60 * 60 * 1000;
  const cutoff = Date.now() - maxAgeMs;

  const oldAudiences = audiences.filter(
    a => a.created * 1000 < cutoff && a.status === 'READY'
  );

  console.log(`Found ${oldAudiences.length} audiences older than ${maxAgeDays} days`);

  let deleted = 0;
  for (const audience of oldAudiences) {
    const success = await deleteAudience(audience.audienceGroupId);
    if (success) deleted++;
    await new Promise(r => setTimeout(r, 200));
  }

  console.log(`Deleted ${deleted}/${oldAudiences.length} old audiences`);
  return deleted;
}

// ลบ Audience ที่ล้มเหลว
async function deleteFailedAudiences() {
  const audiences = await audienceManager.list({ status: 'FAILED' });
  const results = [];

  for (const a of (audiences.audienceGroups || [])) {
    const success = await deleteAudience(a.audienceGroupId);
    results.push({ id: a.audienceGroupId, success });
  }

  return results;
}
```

---

## 12. ใช้ Audience ใน Narrowcast

### Narrowcast กับ Audience

```javascript
// Narrowcast Service
async function sendNarrowcast(audienceGroupId, messages, options = {}) {
  const payload = {
    messages: Array.isArray(messages) ? messages : [messages],
    recipient: {
      type: 'audience',
      audienceGroupId
    },
    notificationDisabled: options.silent || false
  };

  // เพิ่ม Demographic Filter ถ้ามี
  if (options.demographic) {
    payload.demographic = options.demographic;
  }

  const response = await axios.post(
    'https://api.line.me/v2/bot/message/narrowcast',
    payload,
    { headers }
  );

  return response.data;
}

// ตัวอย่าง: Narrowcast ให้ VIP Members
async function sendVIPPromotion(promotion) {
  const vipAudienceId = await getAudienceId('VIP Members');

  if (!vipAudienceId) {
    throw new Error('VIP Audience not found');
  }

  const message = {
    type: 'flex',
    altText: `VIP Exclusive: ${promotion.title}`,
    contents: buildPromotionFlex(promotion, 'vip')
  };

  const result = await sendNarrowcast(vipAudienceId, message);
  console.log(`VIP Narrowcast sent: ${result.requestId}`);
  return result.requestId;
}

async function getAudienceId(description) {
  const audience = await audienceManager.findByDescription(description);
  return audience?.audienceGroupId;
}

// Narrowcast กับ Multiple Audiences (Union)
async function sendToMultipleAudiences(audienceIds, message) {
  const results = [];

  for (const audienceId of audienceIds) {
    try {
      const result = await sendNarrowcast(audienceId, message);
      results.push({ audienceId, requestId: result.requestId, success: true });
    } catch (error) {
      results.push({ audienceId, success: false, error: error.message });
    }
    await new Promise(r => setTimeout(r, 1000));
  }

  return results;
}

// Narrowcast พร้อม Demographic Filter
async function sendTargetedCampaign(audienceGroupId, message, demographic) {
  /*
  demographic = {
    age: { gte: 'age_35', lte: 'age_44' },
    gender: { oneOf: ['male'] },
    region: { oneOf: ['TH-10'] }, // Bangkok
    appType: { oneOf: ['IOS', 'ANDROID'] }
  }
  */

  return await sendNarrowcast(audienceGroupId, message, { demographic });
}
```

### Narrowcast + Audience + A/B Testing

```javascript
async function runABTest(audienceGroupId, messageA, messageB) {
  // แบ่ง Audience เป็น 2 กลุ่ม (50/50)
  const audiences = await audienceManager.get(audienceGroupId);
  const totalUsers = audiences.audienceGroup?.audienceCount || 0;

  console.log(`Running A/B test on ${totalUsers} users`);

  // สร้าง Narrowcast 2 รอบ พร้อม ratio
  const resultA = await axios.post(
    'https://api.line.me/v2/bot/message/narrowcast',
    {
      messages: [messageA],
      recipient: { type: 'audience', audienceGroupId },
      limit: {
        type: 'percentage',
        percentage: 50
      }
    },
    { headers }
  );

  const resultB = await axios.post(
    'https://api.line.me/v2/bot/message/narrowcast',
    {
      messages: [messageB],
      recipient: { type: 'audience', audienceGroupId },
      limit: {
        type: 'percentage',
        percentage: 50
      }
    },
    { headers }
  );

  return {
    testA: { requestId: resultA.data.requestId, message: 'A' },
    testB: { requestId: resultB.data.requestId, message: 'B' }
  };
}

// ดู Narrowcast Progress
async function getNarrowcastProgress(requestId) {
  const response = await axios.get(
    `https://api.line.me/v2/bot/message/progress/narrowcast?requestId=${requestId}`,
    { headers }
  );
  return response.data;
}

// รอจนกว่า Narrowcast จะส่งเสร็จ
async function waitForNarrowcastComplete(requestId, maxWait = 3600000) {
  const startTime = Date.now();

  while (Date.now() - startTime < maxWait) {
    const progress = await getNarrowcastProgress(requestId);

    if (progress.phase === 'SUCCEEDED') {
      return progress;
    }
    if (progress.phase === 'FAILED') {
      throw new Error(`Narrowcast failed: ${progress.errorCode}`);
    }

    console.log(`Phase: ${progress.phase}, Sent: ${progress.successCount || 0}`);
    await new Promise(r => setTimeout(r, 30000));
  }

  throw new Error('Narrowcast timeout');
}
```

---

## 13. Audience Size และ Limits

### Limits

| ข้อจำกัด | ค่า |
|---------|-----|
| จำนวน Audience Groups สูงสุด | 1,000 |
| Users ต่อ Upload Audience | สูงสุด 100,000 |
| Users ต่อ Request (Upload) | สูงสุด 10,000 |
| ขนาดต่ำสุดสำหรับ Narrowcast | 50 users |
| Audience Name | สูงสุด 120 ตัวอักษร |
| Audience อายุสูงสุด | 180 วัน |

### ตรวจสอบขนาด Audience

```javascript
async function checkAudienceSize(audienceGroupId) {
  const audience = await audienceManager.get(audienceGroupId);
  const count = audience.audienceGroup?.audienceCount || 0;

  const MIN_SIZE = 50;

  if (count < MIN_SIZE) {
    console.warn(`⚠️ Audience ${audienceGroupId} has only ${count} users (minimum: ${MIN_SIZE})`);
    return { ready: false, count, minRequired: MIN_SIZE };
  }

  return { ready: true, count };
}

// ตรวจสอบก่อน Narrowcast
async function safeNarrowcast(audienceGroupId, messages) {
  const sizeCheck = await checkAudienceSize(audienceGroupId);

  if (!sizeCheck.ready) {
    throw new Error(
      `Audience too small: ${sizeCheck.count} users (minimum: ${sizeCheck.minRequired})`
    );
  }

  const audience = await audienceManager.get(audienceGroupId);
  if (audience.audienceGroup?.status !== 'READY') {
    throw new Error(`Audience not ready: status = ${audience.audienceGroup?.status}`);
  }

  return await sendNarrowcast(audienceGroupId, messages);
}

// สรุปข้อมูล Audience ทั้งหมด
async function getAudienceSummary() {
  const audiences = await audienceManager.listAll();

  const summary = {
    total: audiences.length,
    byType: {},
    byStatus: {},
    totalUsers: 0
  };

  for (const a of audiences) {
    // By Type
    summary.byType[a.type] = (summary.byType[a.type] || 0) + 1;

    // By Status
    summary.byStatus[a.status] = (summary.byStatus[a.status] || 0) + 1;

    // Total Users
    summary.totalUsers += a.audienceCount || 0;
  }

  return summary;
}
```

---

## 14. Node.js Examples

### Complete Audience Workflow

```javascript
// audience-workflow.js
const AudienceManager = require('./AudienceManager');
const User = require('./models/User');
const Order = require('./models/Order');

const manager = new AudienceManager();

// ─── สร้าง Audience ทุกประเภทสำหรับ E-commerce ─────────────────────

async function setupEcommerceAudiences() {
  console.log('Setting up E-commerce Audiences...\n');

  const audiences = {};

  // 1. New Users (เพิ่มเพื่อนใน 30 วัน)
  const newUsers = await User.find({
    createdAt: { $gte: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000) },
    lineUserId: { $exists: true }
  }).select('lineUserId');

  audiences.newUsers = await manager.upsert(
    'New Users - 30 Days',
    newUsers.map(u => u.lineUserId)
  );
  console.log(`1. New Users: ${audiences.newUsers.audienceGroupId} (${newUsers.length} users)`);

  // 2. Active Buyers (ซื้อใน 90 วัน)
  const activeBuyers = await Order.distinct('userId', {
    createdAt: { $gte: new Date(Date.now() - 90 * 24 * 60 * 60 * 1000) },
    status: 'delivered'
  });
  const activeBuyerUsers = await User.find({
    _id: { $in: activeBuyers },
    lineUserId: { $exists: true }
  }).select('lineUserId');

  audiences.activeBuyers = await manager.upsert(
    'Active Buyers - 90 Days',
    activeBuyerUsers.map(u => u.lineUserId)
  );
  console.log(`2. Active Buyers: ${audiences.activeBuyers.audienceGroupId}`);

  // 3. High Value Customers (ซื้อ > 5,000 บาทใน 90 วัน)
  const highValueData = await Order.aggregate([
    {
      $match: {
        createdAt: { $gte: new Date(Date.now() - 90 * 24 * 60 * 60 * 1000) },
        status: 'delivered'
      }
    },
    {
      $group: {
        _id: '$userId',
        totalSpent: { $sum: '$totalAmount' }
      }
    },
    {
      $match: { totalSpent: { $gte: 5000 } }
    }
  ]);

  const highValueIds = highValueData.map(d => d._id);
  const highValueUsers = await User.find({
    _id: { $in: highValueIds },
    lineUserId: { $exists: true }
  }).select('lineUserId');

  audiences.highValue = await manager.upsert(
    'High Value Customers',
    highValueUsers.map(u => u.lineUserId)
  );
  console.log(`3. High Value: ${audiences.highValue.audienceGroupId}`);

  // 4. Inactive Users (ไม่ซื้อใน 90 วัน)
  const activeBuyerUserIds = activeBuyerUsers.map(u => u._id.toString());
  const inactiveUsers = await User.find({
    _id: { $nin: activeBuyerUserIds },
    createdAt: { $lt: new Date(Date.now() - 90 * 24 * 60 * 60 * 1000) },
    lineUserId: { $exists: true }
  }).select('lineUserId');

  audiences.inactive = await manager.upsert(
    'Inactive Users - 90 Days',
    inactiveUsers.map(u => u.lineUserId)
  );
  console.log(`4. Inactive: ${audiences.inactive.audienceGroupId}`);

  // 5. VIP Members
  const vipMembers = await User.find({
    memberStatus: 'vip',
    lineUserId: { $exists: true }
  }).select('lineUserId');

  audiences.vip = await manager.upsert(
    'VIP Members',
    vipMembers.map(u => u.lineUserId)
  );
  console.log(`5. VIP: ${audiences.vip.audienceGroupId}`);

  console.log('\nAudience setup complete!');
  return audiences;
}

// ─── Campaign Functions ────────────────────────────────────────────

// Re-engagement Campaign
async function runReEngagementCampaign() {
  const inactiveAudience = await manager.findByDescription('Inactive Users - 90 Days');

  if (!inactiveAudience) {
    console.error('Inactive audience not found');
    return;
  }

  const sizeCheck = await checkAudienceSize(inactiveAudience.audienceGroupId);
  if (!sizeCheck.ready) {
    console.log('Audience too small for campaign');
    return;
  }

  console.log(`Sending re-engagement to ${sizeCheck.count} users...`);

  const message = {
    type: 'flex',
    altText: 'เราคิดถึงคุณ! ส่วนลด 15%',
    contents: {
      type: 'bubble',
      hero: {
        type: 'image',
        url: 'https://example.com/comeback.jpg',
        size: 'full',
        aspectRatio: '20:13',
        aspectMode: 'cover'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          { type: 'text', text: '😊 เราคิดถึงคุณ!', weight: 'bold', size: 'xl' },
          { type: 'text', text: 'กลับมาช้อปกับเราได้เลย รับส่วนลด 15%', wrap: true, color: '#666666', margin: 'sm' },
          {
            type: 'text',
            text: 'ใช้โค้ด: COMEBACK15',
            size: 'lg',
            weight: 'bold',
            color: '#E74C3C',
            margin: 'md',
            align: 'center'
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
              label: 'ช้อปเลย',
              uri: 'https://yourshop.com?coupon=COMEBACK15'
            },
            style: 'primary',
            color: '#E74C3C'
          }
        ]
      }
    }
  };

  const result = await safeNarrowcast(inactiveAudience.audienceGroupId, message);
  console.log(`Re-engagement campaign sent: ${result.requestId}`);
  return result.requestId;
}

// VIP Exclusive Campaign
async function runVIPExclusiveSale(saleData) {
  const vipAudience = await manager.findByDescription('VIP Members');

  if (!vipAudience) throw new Error('VIP audience not found');

  const message = {
    type: 'flex',
    altText: `VIP EXCLUSIVE: ${saleData.title}`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        backgroundColor: '#2C3E50',
        paddingAll: 'lg',
        contents: [
          { type: 'text', text: '⭐ VIP EXCLUSIVE', color: '#FFD700', size: 'sm' },
          { type: 'text', text: saleData.title, color: '#FFFFFF', weight: 'bold', size: 'xl' }
        ]
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          { type: 'text', text: saleData.description, wrap: true },
          {
            type: 'text',
            text: `ลด ${saleData.discountPercent}%`,
            size: 'xxl',
            weight: 'bold',
            color: '#E74C3C',
            margin: 'lg',
            align: 'center'
          },
          {
            type: 'text',
            text: `สิ้นสุด: ${new Date(saleData.endDate).toLocaleDateString('th-TH')}`,
            size: 'sm',
            color: '#999999',
            margin: 'md',
            align: 'center'
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
              label: 'ช้อปสินค้า VIP',
              uri: saleData.shopUrl
            },
            style: 'primary',
            color: '#2C3E50'
          }
        ]
      }
    }
  };

  return await safeNarrowcast(vipAudience.audienceGroupId, message);
}

// ─── Analytics ────────────────────────────────────────────

async function getAudienceAnalytics() {
  const audiences = await manager.listAll();

  const report = audiences.map(a => ({
    id: a.audienceGroupId,
    name: a.description,
    type: a.type,
    status: a.status,
    size: a.audienceCount || 0,
    createdAt: new Date(a.created * 1000).toLocaleDateString('th-TH'),
    age: Math.floor((Date.now() - a.created * 1000) / (24 * 60 * 60 * 1000)) + ' วัน'
  }));

  console.table(report);

  const summary = {
    totalAudiences: report.length,
    totalUniqueUsers: report.reduce((s, a) => s + a.size, 0),
    byType: report.reduce((acc, a) => {
      acc[a.type] = (acc[a.type] || 0) + 1;
      return acc;
    }, {}),
    readyAudiences: report.filter(a => a.status === 'READY').length
  };

  return { report, summary };
}

module.exports = {
  setupEcommerceAudiences,
  runReEngagementCampaign,
  runVIPExclusiveSale,
  getAudienceAnalytics
};
```

### ตัวอย่างการใช้งาน

```javascript
// main.js
const {
  setupEcommerceAudiences,
  runReEngagementCampaign,
  runVIPExclusiveSale,
  getAudienceAnalytics
} = require('./audience-workflow');

async function main() {
  // 1. ตั้งค่า Audiences ทั้งหมด
  console.log('=== Setting up Audiences ===');
  const audiences = await setupEcommerceAudiences();

  // 2. รัน Re-engagement Campaign
  console.log('\n=== Running Re-engagement Campaign ===');
  const reEngagementId = await runReEngagementCampaign();
  if (reEngagementId) {
    console.log(`Campaign sent: ${reEngagementId}`);
  }

  // 3. ส่ง VIP Sale
  console.log('\n=== VIP Exclusive Sale ===');
  const vipResult = await runVIPExclusiveSale({
    title: 'YEAR END SALE 2024',
    description: 'สินค้าคัดสรรสำหรับสมาชิก VIP เท่านั้น',
    discountPercent: 30,
    endDate: new Date('2024-12-31'),
    shopUrl: 'https://yourshop.com/vip-sale'
  });
  console.log('VIP Sale sent:', vipResult?.requestId);

  // 4. ดู Analytics
  console.log('\n=== Audience Analytics ===');
  const analytics = await getAudienceAnalytics();
  console.log('\nSummary:', analytics.summary);
}

main().catch(console.error);
```

---

## สรุป Audience API

### กลยุทธ์การใช้ Audience

```
1. Upload Audience
   └─> ควบคุมได้เต็มที่ (Custom Segments)
   └─> เหมาะกับ: VIP, Members, Custom Lists

2. Click Audience
   └─> Re-target คนที่สนใจ
   └─> เหมาะกับ: Remarketing

3. Impression Audience
   └─> คนที่เห็นข้อความ
   └─> เหมาะกับ: Follow-up campaigns

4. Video Audience
   └─> คนที่ดู Video
   └─> เหมาะกับ: Video marketing

5. Chat Tag Audience
   └─> ใช้ Tags จัดกลุ่ม
   └─> เหมาะกับ: Customer segments

6. Friend Path Audience
   └─> คนที่เพิ่งเพิ่มเพื่อน
   └─> เหมาะกับ: Welcome campaigns
```

| สิ่งที่ต้องจำ | ความสำคัญ |
|-------------|---------|
| ขั้นต่ำ 50 users ถึง Narrowcast ได้ | สูงมาก |
| อัพเดต Audience ก่อน Campaign | สูง |
| ตรวจสอบ Status = READY | สูง |
| ลบ Audience เก่าเพื่อประหยัด Quota | ปานกลาง |
| ใช้ Click/Impression สำหรับ Remarketing | ปานกลาง |

---

*จบ Part 55: Audience API*

*ดูส่วนอื่นๆ:*
- *[Part 51: Rich Menu API ขั้นสูง](part-051-rich-menu-api-advanced.md)*
- *[Part 52: LINE Login](part-052-line-login.md)*
- *[Part 53: LINE Pay](part-053-line-pay.md)*
- *[Part 54: Notification Channel](part-054-notification-channel.md)*
