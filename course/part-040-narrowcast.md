# Part 40: Narrowcast - การส่งข้อความแบบกำหนดกลุ่มเป้าหมาย

## บทนำ

Narrowcast คือการส่งข้อความไปยังกลุ่มผู้ใช้เฉพาะ โดยสามารถกรองตามข้อมูลประชากร (Demographic), พฤติกรรม, หรือ Custom Audience ทำให้การส่งข้อความมีประสิทธิภาพและตรงกลุ่มเป้าหมายมากกว่า Broadcast ทั่วไป

---

## 1. เปรียบเทียบ Narrowcast vs Multicast vs Broadcast

### ตารางเปรียบเทียบ

| ฟีเจอร์ | Broadcast | Multicast | Narrowcast |
|---------|-----------|-----------|------------|
| เป้าหมาย | ทุกคน | ระบุ User IDs | กรองตาม Criteria |
| จำนวนผู้รับ | ทุก Follower | สูงสุด 500 users/request | กรองจากทั้งหมด |
| ข้อกำหนด Followers | ไม่จำกัด | ไม่จำกัด | ต้องมี ≥100 Followers |
| Demographic Filter | ❌ | ❌ | ✅ |
| Custom Audience | ❌ | ❌ | ✅ |
| ค่าใช้จ่าย | ตาม Message Quota | ตาม Message Quota | ตาม Message Quota |
| Async/Sync | Sync | Sync | Async |
| ตรวจสอบความคืบหน้า | ❌ | ❌ | ✅ |

### When to Use What

```
Broadcast → ส่งให้ทุกคน (ข่าวสาร, ประกาศ)
    ↓
Multicast → ส่งให้กลุ่มเฉพาะที่รู้จัก User IDs
    ↓
Narrowcast → ส่งตาม Demographic/Audience
```

---

## 2. โครงสร้าง Narrowcast Request

### Basic Request Structure

```json
{
  "messages": [
    {
      "type": "text",
      "text": "สวัสดีครับ! นี่คือข้อความสำหรับกลุ่มเป้าหมาย"
    }
  ],
  "recipient": {
    "type": "all"
  },
  "filter": {
    "demographic": {
      "type": "age",
      "ageMin": "age_18",
      "ageMax": "age_35"
    }
  },
  "limit": {
    "upTo": 1000
  },
  "notificationDisabled": false
}
```

### Request Fields

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `messages` | Array | Yes | ข้อความที่ส่ง (สูงสุด 5 messages) |
| `recipient` | Object | No | กลุ่มผู้รับ |
| `filter` | Object | No | Demographic Filter |
| `limit` | Object | No | จำกัดจำนวนผู้รับ |
| `notificationDisabled` | Boolean | No | ปิด notification |

---

## 3. Demographic Filters

### 3.1 Age Filter

```json
{
  "filter": {
    "demographic": {
      "type": "age",
      "ageMin": "age_18",
      "ageMax": "age_34"
    }
  }
}
```

#### Age Range Values

| Value | Age |
|-------|-----|
| `age_15` | 15 ปี |
| `age_20` | 20 ปี |
| `age_25` | 25 ปี |
| `age_30` | 30 ปี |
| `age_35` | 35 ปี |
| `age_40` | 40 ปี |
| `age_45` | 45 ปี |
| `age_50` | 50 ปี |

**หมายเหตุ:** ต้องมีผู้ใช้ใน Age Range นั้นอย่างน้อย 100 คน

### 3.2 Gender Filter

```json
{
  "filter": {
    "demographic": {
      "type": "gender",
      "oneOf": ["male"]
    }
  }
}
```

#### Gender Values

| Value | Description |
|-------|-------------|
| `male` | เพศชาย |
| `female` | เพศหญิง |

### 3.3 OS Type Filter

```json
{
  "filter": {
    "demographic": {
      "type": "appType",
      "oneOf": ["ios"]
    }
  }
}
```

#### OS Type Values

| Value | Description |
|-------|-------------|
| `ios` | iPhone/iPad |
| `android` | Android |

### 3.4 Region Filter

```json
{
  "filter": {
    "demographic": {
      "type": "region",
      "oneOf": ["TH"]
    }
  }
}
```

#### Region Codes (Thailand)

| Code | Region |
|------|--------|
| `TH` | Thailand |
| `TH01` | กรุงเทพมหานคร |
| `TH02` | เชียงใหม่ |
| `TH03` | ขอนแก่น |
| `TH04` | ชลบุรี |
| `TH05` | สงขลา |
| `TH06` | นครราชสีมา |

### 3.5 Subscription Period Filter

```json
{
  "filter": {
    "demographic": {
      "type": "subscriptionPeriod",
      "gte": "day_7",
      "lte": "day_30"
    }
  }
}
```

#### Subscription Period Values

| Value | Description |
|-------|-------------|
| `day_7` | 7 วัน |
| `day_30` | 30 วัน |
| `day_90` | 90 วัน |
| `day_180` | 180 วัน |
| `day_365` | 365 วัน |

---

## 4. Audience-Based Narrowcast

### 4.1 ประเภท Audience

```
Audience Types:
├── Upload Audience    - อัปโหลด User IDs หรือ IFA
├── Click Audience     - คนที่คลิก URL ใน Message
├── Impression Audience - คนที่เห็น Message
├── Video View Audience - คนที่ดูวิดีโอ
└── Chat Tag Audience  - คนที่มี Chat Tag เฉพาะ
```

### 4.2 สร้าง Upload Audience

```javascript
// createAudience.js
const axios = require('axios');

const headers = {
  Authorization: `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
  'Content-Type': 'application/json',
};

// สร้าง Audience จาก User IDs
async function createAudienceByUserIds(description, userIds) {
  try {
    const response = await axios.post(
      'https://api.line.me/v2/bot/audienceGroup/upload',
      {
        description,
        isIfaAudience: false,
        audiences: userIds.map(userId => ({ id: userId })),
      },
      { headers }
    );

    console.log('✅ Audience created:', response.data);
    return response.data.audienceGroupId;
  } catch (error) {
    console.error('❌ Error creating audience:', error.response?.data);
    throw error;
  }
}

// ดูรายการ Audience
async function listAudiences() {
  try {
    const response = await axios.get(
      'https://api.line.me/v2/bot/audienceGroup/list',
      { headers }
    );
    return response.data.audienceGroups;
  } catch (error) {
    console.error('❌ Error listing audiences:', error.response?.data);
    throw error;
  }
}

// ดูรายละเอียด Audience
async function getAudience(audienceGroupId) {
  try {
    const response = await axios.get(
      `https://api.line.me/v2/bot/audienceGroup/${audienceGroupId}`,
      { headers }
    );
    return response.data;
  } catch (error) {
    console.error('❌ Error getting audience:', error.response?.data);
    throw error;
  }
}

module.exports = { createAudienceByUserIds, listAudiences, getAudience };
```

### 4.3 Narrowcast โดยใช้ Audience

```json
{
  "messages": [
    {
      "type": "text",
      "text": "สวัสดีสมาชิก VIP! นี่คือ Offer พิเศษสำหรับคุณ"
    }
  ],
  "recipient": {
    "type": "audienceGroup",
    "audienceGroupId": 5614991017776
  },
  "filter": {
    "demographic": {
      "type": "gender",
      "oneOf": ["female"]
    }
  }
}
```

---

## 5. Combined Filters

### การรวม Filters ด้วย AND/OR

```json
{
  "filter": {
    "demographic": {
      "type": "and",
      "demographicFilterConditions": [
        {
          "type": "age",
          "ageMin": "age_25",
          "ageMax": "age_34"
        },
        {
          "type": "gender",
          "oneOf": ["female"]
        }
      ]
    }
  }
}
```

### OR Condition

```json
{
  "filter": {
    "demographic": {
      "type": "or",
      "demographicFilterConditions": [
        {
          "type": "region",
          "oneOf": ["TH01"]
        },
        {
          "type": "region",
          "oneOf": ["TH02"]
        }
      ]
    }
  }
}
```

### NOT Condition

```json
{
  "filter": {
    "demographic": {
      "type": "not",
      "demographicFilterCondition": {
        "type": "subscriptionPeriod",
        "gte": "day_7"
      }
    }
  }
}
```

### Complex Combined Filter (AND + OR)

```json
{
  "filter": {
    "demographic": {
      "type": "and",
      "demographicFilterConditions": [
        {
          "type": "age",
          "ageMin": "age_20",
          "ageMax": "age_35"
        },
        {
          "type": "or",
          "demographicFilterConditions": [
            {
              "type": "region",
              "oneOf": ["TH01", "TH02"]
            },
            {
              "type": "appType",
              "oneOf": ["ios"]
            }
          ]
        }
      ]
    }
  }
}
```

---

## 6. Narrowcast with Limit

```json
{
  "messages": [
    {
      "type": "text",
      "text": "โปรโมชั่นด่วน! 100 คนแรกเท่านั้น"
    }
  ],
  "recipient": {
    "type": "all"
  },
  "limit": {
    "upTo": 100
  }
}
```

### Limit Object

| Field | Type | Description |
|-------|------|-------------|
| `upTo` | Integer | จำนวนสูงสุดที่จะส่ง (1 - 9999999) |

---

## 7. API Request Structure

### HTTP Request

```
POST https://api.line.me/v2/bot/message/narrowcast

Headers:
  Authorization: Bearer {Channel Access Token}
  Content-Type: application/json

Body: {Narrowcast Request Object}
```

### Response

```json
{
  "requestId": "5b59509c-c57b-11e9-aa8c-2a2ae2dbcce4"
}
```

**หมายเหตุ:** Response จะมีเฉพาะ `requestId` เท่านั้น การส่งเป็น Asynchronous

---

## 8. Complete Node.js Implementation

### 8.1 Basic Narrowcast Service

```javascript
// narrowcastService.js
const axios = require('axios');

class NarrowcastService {
  constructor(accessToken) {
    this.accessToken = accessToken;
    this.baseUrl = 'https://api.line.me/v2/bot';
    this.headers = {
      Authorization: `Bearer ${accessToken}`,
      'Content-Type': 'application/json',
    };
  }

  // ส่ง Narrowcast
  async send(options) {
    const {
      messages,
      recipient = { type: 'all' },
      filter,
      limit,
      notificationDisabled = false,
    } = options;

    const body = { messages, recipient, notificationDisabled };
    if (filter) body.filter = filter;
    if (limit) body.limit = limit;

    try {
      const response = await axios.post(
        `${this.baseUrl}/message/narrowcast`,
        body,
        { headers: this.headers }
      );

      console.log('✅ Narrowcast sent! Request ID:', response.data.requestId);
      return response.data.requestId;
    } catch (error) {
      const errData = error.response?.data;
      console.error('❌ Narrowcast failed:', errData);
      throw new Error(`Narrowcast failed: ${JSON.stringify(errData)}`);
    }
  }

  // ตรวจสอบสถานะ
  async getProgress(requestId) {
    try {
      const response = await axios.get(
        `${this.baseUrl}/message/progress/narrowcast?requestId=${requestId}`,
        { headers: this.headers }
      );
      return response.data;
    } catch (error) {
      console.error('❌ Error getting progress:', error.response?.data);
      throw error;
    }
  }

  // รอจนกว่าจะเสร็จ
  async waitForCompletion(requestId, intervalMs = 5000, maxWaitMs = 300000) {
    const startTime = Date.now();

    while (Date.now() - startTime < maxWaitMs) {
      const progress = await this.getProgress(requestId);
      console.log(`Progress: ${progress.phase} - Success: ${progress.successCount}, Failed: ${progress.failureCount}`);

      if (progress.phase === 'succeeded') {
        console.log('✅ Narrowcast completed!');
        return progress;
      }

      if (progress.phase === 'failed') {
        throw new Error(`Narrowcast failed: ${progress.errorCode}`);
      }

      // รอก่อน check ครั้งต่อไป
      await new Promise(resolve => setTimeout(resolve, intervalMs));
    }

    throw new Error('Narrowcast timed out');
  }

  // ส่งตาม Age Range
  async sendToAgeRange(messages, ageMin, ageMax, additionalOptions = {}) {
    return this.send({
      messages,
      filter: {
        demographic: {
          type: 'age',
          ageMin,
          ageMax,
        },
      },
      ...additionalOptions,
    });
  }

  // ส่งตาม Gender
  async sendToGender(messages, gender, additionalOptions = {}) {
    return this.send({
      messages,
      filter: {
        demographic: {
          type: 'gender',
          oneOf: Array.isArray(gender) ? gender : [gender],
        },
      },
      ...additionalOptions,
    });
  }

  // ส่งตาม Region
  async sendToRegion(messages, regions, additionalOptions = {}) {
    return this.send({
      messages,
      filter: {
        demographic: {
          type: 'region',
          oneOf: Array.isArray(regions) ? regions : [regions],
        },
      },
      ...additionalOptions,
    });
  }

  // ส่งตาม OS Type
  async sendToOSType(messages, osType, additionalOptions = {}) {
    return this.send({
      messages,
      filter: {
        demographic: {
          type: 'appType',
          oneOf: Array.isArray(osType) ? osType : [osType],
        },
      },
      ...additionalOptions,
    });
  }

  // ส่งไปยัง Audience Group
  async sendToAudience(messages, audienceGroupId, additionalOptions = {}) {
    return this.send({
      messages,
      recipient: {
        type: 'audienceGroup',
        audienceGroupId,
      },
      ...additionalOptions,
    });
  }
}

module.exports = NarrowcastService;
```

### 8.2 ตัวอย่างการใช้งาน

```javascript
// examples.js
const NarrowcastService = require('./narrowcastService');

const service = new NarrowcastService(process.env.LINE_CHANNEL_ACCESS_TOKEN);

// ตัวอย่าง 1: ส่งโปรโมชั่นให้ผู้หญิง อายุ 20-34 ปี
async function sendFashionPromotion() {
  const requestId = await service.send({
    messages: [
      {
        type: 'flex',
        altText: 'โปรโมชั่นเสื้อผ้าผู้หญิง ลด 30%!',
        contents: {
          type: 'bubble',
          hero: {
            type: 'image',
            url: 'https://shop.example.com/promo/fashion-sale.jpg',
            size: 'full',
            aspectRatio: '4:3',
          },
          body: {
            type: 'box',
            layout: 'vertical',
            contents: [
              { type: 'text', text: '👗 Fashion Sale', weight: 'bold', size: 'xl' },
              { type: 'text', text: 'ลด 30% ทุกรายการ', size: 'lg', color: '#E74C3C', margin: 'sm' },
              { type: 'text', text: 'วันนี้ - 31 ธ.ค. เท่านั้น!', size: 'sm', color: '#888', margin: 'sm' },
            ],
          },
          footer: {
            type: 'box',
            layout: 'vertical',
            contents: [
              {
                type: 'button',
                action: {
                  type: 'uri',
                  label: '🛍️ ช้อปเลย',
                  uri: 'https://shop.example.com/fashion-sale',
                },
                style: 'primary',
                color: '#FF69B4',
              },
            ],
          },
        },
      },
    ],
    filter: {
      demographic: {
        type: 'and',
        demographicFilterConditions: [
          { type: 'gender', oneOf: ['female'] },
          { type: 'age', ageMin: 'age_20', ageMax: 'age_34' },
        ],
      },
    },
    limit: { upTo: 5000 },
  });

  // รอผลลัพธ์
  const result = await service.waitForCompletion(requestId);
  console.log(`✅ Sent to ${result.successCount} users`);
}

// ตัวอย่าง 2: ส่งโปรโมชั่นเกมให้ผู้ใช้ iOS อายุ 18-30
async function sendGamePromotion() {
  const requestId = await service.send({
    messages: [
      {
        type: 'text',
        text: '🎮 เกมใหม่มาแล้ว! ดาวน์โหลดฟรีบน iOS วันนี้เท่านั้น',
      },
    ],
    filter: {
      demographic: {
        type: 'and',
        demographicFilterConditions: [
          { type: 'appType', oneOf: ['ios'] },
          { type: 'age', ageMin: 'age_18', ageMax: 'age_30' },
        ],
      },
    },
  });

  console.log('Game promotion sent! Request ID:', requestId);
}

// ตัวอย่าง 3: ส่งให้สมาชิกใหม่ (สมัครน้อยกว่า 7 วัน)
async function sendWelcomeToNewFollowers() {
  const requestId = await service.send({
    messages: [
      {
        type: 'text',
        text: '🎉 ยินดีต้อนรับสมาชิกใหม่!\n\nรับคูปองส่วนลด 15% สำหรับการสั่งซื้อครั้งแรก\n\nใช้โค้ด: WELCOME15',
      },
    ],
    filter: {
      demographic: {
        type: 'subscriptionPeriod',
        lte: 'day_7',
      },
    },
  });

  console.log('Welcome message sent! Request ID:', requestId);
}

// ตัวอย่าง 4: ส่งตาม Region (กรุงเทพ + เชียงใหม่)
async function sendBangkokChiangMaiPromotion() {
  const requestId = await service.send({
    messages: [
      {
        type: 'text',
        text: '🏙️ โปรโมชั่นพิเศษสำหรับลูกค้า กทม. และเชียงใหม่\n\nส่งฟรี! ไม่มีขั้นต่ำ วันนี้วันเดียว',
      },
    ],
    filter: {
      demographic: {
        type: 'region',
        oneOf: ['TH01', 'TH02'],
      },
    },
  });

  console.log('Regional promotion sent! Request ID:', requestId);
}

// รันตัวอย่าง
async function runExamples() {
  console.log('🚀 Running Narrowcast examples...\n');

  try {
    await sendFashionPromotion();
    await sendGamePromotion();
    await sendWelcomeToNewFollowers();
    await sendBangkokChiangMaiPromotion();
    console.log('\n✅ All examples completed!');
  } catch (error) {
    console.error('❌ Error:', error.message);
  }
}

runExamples();
```

---

## 9. ตรวจสอบ Narrowcast Progress

### Progress API

```
GET https://api.line.me/v2/bot/message/progress/narrowcast?requestId={requestId}
```

### Progress Response

```json
{
  "phase": "succeeded",
  "successCount": 4521,
  "failureCount": 12,
  "errorCode": 0,
  "acceptedTime": "2024-01-15T10:00:00.000Z",
  "completedTime": "2024-01-15T10:05:30.000Z"
}
```

### Phase Values

| Phase | Description |
|-------|-------------|
| `waiting` | กำลังรอการประมวลผล |
| `sending` | กำลังส่ง |
| `succeeded` | ส่งสำเร็จ |
| `failed` | ส่งล้มเหลว |

### Progress Monitoring

```javascript
// progressMonitor.js
const NarrowcastService = require('./narrowcastService');

class ProgressMonitor {
  constructor(service) {
    this.service = service;
  }

  // Monitor ด้วย Callback
  async monitor(requestId, callbacks = {}) {
    const {
      onProgress,
      onComplete,
      onError,
      intervalMs = 5000,
      maxWaitMs = 600000,
    } = callbacks;

    const startTime = Date.now();

    return new Promise((resolve, reject) => {
      const check = async () => {
        try {
          const progress = await this.service.getProgress(requestId);

          if (onProgress) {
            onProgress(progress);
          }

          if (progress.phase === 'succeeded') {
            if (onComplete) onComplete(progress);
            resolve(progress);
            return;
          }

          if (progress.phase === 'failed') {
            const error = new Error(`Failed with error code: ${progress.errorCode}`);
            if (onError) onError(error, progress);
            reject(error);
            return;
          }

          if (Date.now() - startTime > maxWaitMs) {
            const error = new Error('Progress monitoring timed out');
            if (onError) onError(error);
            reject(error);
            return;
          }

          setTimeout(check, intervalMs);
        } catch (error) {
          if (onError) onError(error);
          reject(error);
        }
      };

      check();
    });
  }

  // Monitor หลาย Requests พร้อมกัน
  async monitorMultiple(requestIds, callbacks = {}) {
    return Promise.all(
      requestIds.map(id => this.monitor(id, callbacks))
    );
  }
}

// ตัวอย่างการใช้
const service = new NarrowcastService(process.env.LINE_CHANNEL_ACCESS_TOKEN);
const monitor = new ProgressMonitor(service);

async function sendAndMonitor() {
  const requestId = await service.sendToGender(
    [{ type: 'text', text: 'Test message' }],
    'female'
  );

  await monitor.monitor(requestId, {
    onProgress: (progress) => {
      console.log(`📊 Status: ${progress.phase}`);
      if (progress.successCount !== undefined) {
        console.log(`   ✅ Success: ${progress.successCount}`);
        console.log(`   ❌ Failed: ${progress.failureCount}`);
      }
    },
    onComplete: (result) => {
      console.log(`\n🎉 Complete!`);
      console.log(`Total sent: ${result.successCount}`);
      console.log(`Duration: ${new Date(result.completedTime) - new Date(result.acceptedTime)}ms`);
    },
    onError: (error) => {
      console.error('Monitor error:', error.message);
    },
    intervalMs: 3000,
  });
}

sendAndMonitor();
```

---

## 10. Error Handling

### Common Error Codes

| Error Code | Description | Solution |
|------------|-------------|----------|
| 400 | Bad request - Invalid parameters | ตรวจสอบ Request body |
| 401 | Unauthorized | ตรวจสอบ Access Token |
| 403 | Forbidden | ตรวจสอบ Permission ของ Channel |
| 429 | Too Many Requests | Rate limit เกิน |

### Narrowcast Error Codes (in Progress)

| Code | Description |
|------|-------------|
| 0 | Success |
| 1 | Internal error |
| 2 | No recipients matched |

### Error Handling Implementation

```javascript
// errorHandler.js
const NarrowcastService = require('./narrowcastService');

class ResilientNarrowcastService extends NarrowcastService {
  async sendWithRetry(options, maxRetries = 3) {
    let lastError;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await this.send(options);
      } catch (error) {
        lastError = error;
        const statusCode = error.response?.status;

        // ไม่ Retry สำหรับ Client Errors (4xx) ยกเว้น 429
        if (statusCode >= 400 && statusCode < 500 && statusCode !== 429) {
          throw error;
        }

        // Rate limit - รอนานขึ้น
        if (statusCode === 429) {
          const retryAfter = parseInt(error.response?.headers?.['retry-after'] || '60');
          console.warn(`Rate limited. Waiting ${retryAfter}s before retry ${attempt}/${maxRetries}`);
          await new Promise(resolve => setTimeout(resolve, retryAfter * 1000));
        } else {
          // Exponential backoff
          const delay = Math.min(1000 * Math.pow(2, attempt - 1), 30000);
          console.warn(`Attempt ${attempt} failed. Retrying in ${delay}ms...`);
          await new Promise(resolve => setTimeout(resolve, delay));
        }
      }
    }

    throw lastError;
  }

  // ตรวจสอบว่า Narrowcast จะมีผู้รับเพียงพอ
  async validateRecipientCount(filter) {
    // ตรวจสอบว่ามีผู้ใช้ตาม filter อย่างน้อย 100 คน
    // (จำเป็นต้องมีการประมาณ เนื่องจาก API ไม่ได้ให้ตรวจสอบล่วงหน้า)
    console.warn('Note: Narrowcast requires at least 100 matching recipients');
    return true;
  }
}

// Graceful Error Handling
async function safeSend(service, options) {
  try {
    const requestId = await service.send(options);
    console.log('✅ Narrowcast initiated:', requestId);
    return { success: true, requestId };
  } catch (error) {
    const response = error.response?.data;
    
    if (response?.message?.includes('recipient')) {
      console.error('❌ No recipients matched the criteria');
      return { success: false, error: 'NO_RECIPIENTS' };
    }

    if (response?.message?.includes('quota')) {
      console.error('❌ Message quota exceeded');
      return { success: false, error: 'QUOTA_EXCEEDED' };
    }

    console.error('❌ Narrowcast failed:', response?.message || error.message);
    return { success: false, error: response?.message || 'UNKNOWN_ERROR' };
  }
}
```

---

## 11. Cost Considerations

### ต้นทุนของ Narrowcast

```
Narrowcast ใช้ Message Credits เหมือนกับ Broadcast:
─────────────────────────────────────────────────────
Plan             Messages/Month    ราคา
─────────────────────────────────────────────────────
Free             500               ฟรี
Light            5,000             ฟรี
Standard         30,000            ฟรี
เกินโควต้า        ทุก 1,000 messages  ≈ ฿4-8 บาท
─────────────────────────────────────────────────────
```

### การคำนวณต้นทุน

```javascript
// costCalculator.js

function estimateNarrowcastCost(estimatedRecipients, pricePerMessage = 0.006) {
  const totalCost = estimatedRecipients * pricePerMessage;

  return {
    recipients: estimatedRecipients,
    costPerMessage: pricePerMessage,
    totalCost: totalCost.toFixed(2),
    currency: 'USD',
    thbEstimate: (totalCost * 35).toFixed(2), // ประมาณ THB
  };
}

// ตัวอย่าง
const estimate = estimateNarrowcastCost(10000);
console.log('Cost estimate:', estimate);
// Output: { recipients: 10000, costPerMessage: 0.006, totalCost: '60.00', ... }

// Tips ลดต้นทุน
const costTips = [
  '1. ใช้ Demographic Filter เพื่อส่งเฉพาะกลุ่มเป้าหมาย',
  '2. ใช้ limit.upTo เพื่อจำกัดจำนวนผู้รับ',
  '3. A/B Test ด้วย limit ก่อนส่งจริง',
  '4. รวมข้อความหลายอย่างในการส่งครั้งเดียว (สูงสุด 5 messages)',
  '5. ติดตามผลและปรับ criteria เพื่อเพิ่ม Conversion Rate',
];

console.log('\nCost Saving Tips:');
costTips.forEach(tip => console.log(tip));
```

### A/B Testing ด้วย Narrowcast

```javascript
// abTest.js
const NarrowcastService = require('./narrowcastService');

async function runABTest(messageA, messageB, targetFilter) {
  const service = new NarrowcastService(process.env.LINE_CHANNEL_ACCESS_TOKEN);

  console.log('🧪 Starting A/B Test...');

  // ส่ง Version A (จำกัด 500 คน)
  const requestIdA = await service.send({
    messages: [messageA],
    filter: targetFilter,
    limit: { upTo: 500 },
  });

  // รอสักครู่แล้วส่ง Version B
  await new Promise(resolve => setTimeout(resolve, 2000));

  const requestIdB = await service.send({
    messages: [messageB],
    filter: targetFilter,
    limit: { upTo: 500 },
  });

  console.log('A/B Test IDs:');
  console.log('  Version A:', requestIdA);
  console.log('  Version B:', requestIdB);

  // รอผลทั้งคู่
  const [resultA, resultB] = await Promise.all([
    service.waitForCompletion(requestIdA),
    service.waitForCompletion(requestIdB),
  ]);

  console.log('\n📊 A/B Test Results:');
  console.log(`Version A: ${resultA.successCount} sent`);
  console.log(`Version B: ${resultB.successCount} sent`);
  
  return { resultA, resultB };
}

// ตัวอย่างการใช้งาน A/B Test
const messageVariantA = {
  type: 'text',
  text: '🎁 รับส่วนลด 20% วันนี้เท่านั้น! กด Shop Now',
};

const messageVariantB = {
  type: 'text',
  text: '⏰ เหลืออีก 3 ชั่วโมง! ลด 20% ทุกสินค้า คลิกเลย',
};

const targetFilter = {
  demographic: {
    type: 'age',
    ageMin: 'age_25',
    ageMax: 'age_34',
  },
};

runABTest(messageVariantA, messageVariantB, targetFilter);
```

---

## 12. Scheduling Narrowcast

```javascript
// scheduledNarrowcast.js
const cron = require('node-cron');
const NarrowcastService = require('./narrowcastService');

const service = new NarrowcastService(process.env.LINE_CHANNEL_ACCESS_TOKEN);

// ส่งข้อความทุกเช้าวันจันทร์ (Weekly Newsletter)
cron.schedule('0 9 * * 1', async () => {
  console.log('📧 Sending weekly newsletter...');

  try {
    const requestId = await service.send({
      messages: [
        {
          type: 'text',
          text: `📰 ข่าวสารประจำสัปดาห์\n\n${getWeeklyNewsletter()}`,
        },
      ],
      recipient: { type: 'all' },
    });

    console.log('✅ Weekly newsletter sent:', requestId);
    
    // บันทึก Log
    await logNarrowcast(requestId, 'weekly_newsletter');
  } catch (error) {
    console.error('❌ Failed to send newsletter:', error.message);
    await notifyAdmin(error);
  }
}, {
  timezone: 'Asia/Bangkok',
});

// ส่งโปรโมชั่นวันหยุดสุดสัปดาห์
cron.schedule('0 10 * * 5', async () => {
  console.log('🎉 Sending weekend promotion...');

  const requestId = await service.send({
    messages: [
      {
        type: 'text',
        text: '🎊 Happy Weekend! ลด 15% ทุกรายการ ใช้โค้ด: WEEKEND15',
      },
    ],
    filter: {
      demographic: {
        type: 'subscriptionPeriod',
        gte: 'day_30',
      },
    },
  });

  console.log('Weekend promo sent:', requestId);
}, {
  timezone: 'Asia/Bangkok',
});

// ส่งข้อความ Re-engagement ให้ inactive users
cron.schedule('0 14 1 * *', async () => {
  console.log('💌 Sending re-engagement message...');

  const requestId = await service.send({
    messages: [
      {
        type: 'text',
        text: '👋 สวัสดี! นานแล้วที่เราไม่ได้คุยกัน\n\nมีโปรโมชั่นพิเศษรอคุณอยู่ เข้ามาดูนะ!',
      },
    ],
    filter: {
      demographic: {
        type: 'subscriptionPeriod',
        gte: 'day_90',
      },
    },
    limit: { upTo: 10000 },
  });

  console.log('Re-engagement sent:', requestId);
}, {
  timezone: 'Asia/Bangkok',
});

function getWeeklyNewsletter() {
  return `สัปดาห์ที่ ${new Date().toLocaleDateString('th-TH')}\n\n` +
    '• สินค้าใหม่ประจำสัปดาห์\n' +
    '• โปรโมชั่นที่ไม่ควรพลาด\n' +
    '• เคล็ดลับและข่าวสาร';
}

async function logNarrowcast(requestId, type) {
  console.log(`Log: ${type} - ${requestId}`);
}

async function notifyAdmin(error) {
  console.error('Admin notification:', error.message);
}

console.log('📅 Scheduled Narrowcast service started');
```

---

## 13. Best Practices

### Checklist ก่อนส่ง Narrowcast

```
✅ Pre-Send Checklist:
───────────────────────────────────────────
□ ตรวจสอบเนื้อหาข้อความ
□ ตรวจสอบ Link ทั้งหมดทำงานได้
□ ตรวจสอบ Filter ถูกต้อง
□ ประมาณจำนวนผู้รับ (ต้องมี ≥100)
□ ตั้งค่า Limit ถ้าต้องการจำกัด
□ เตรียม Landing Page รองรับ Traffic
□ Monitor Progress หลังส่ง
□ วางแผน Follow-up Message
───────────────────────────────────────────
```

### Dos and Don'ts

```
✅ DOs:
- ใช้ Filter ที่ตรงกลุ่มเป้าหมาย
- ทดสอบด้วย Limit ก่อนส่งจริง
- Monitor Progress เสมอ
- ส่งในเวลาที่เหมาะสม
- Personalize ข้อความตาม Segment

❌ DON'Ts:
- ส่งบ่อยเกินไป (spam)
- ส่งข้อความที่ไม่เกี่ยวข้อง
- ใช้ Filter แบบ Broad เกินไป
- ละเลย Error Handling
- ไม่ตรวจสอบ Progress
```

### Timing Best Practices

```javascript
// bestTimes.js
const BEST_SEND_TIMES = {
  // กลุ่ม Young Adults (18-30)
  youngAdults: {
    weekday: { hour: 20, minute: 0 }, // 20:00 น.
    weekend: { hour: 11, minute: 0 }, // 11:00 น.
  },
  // กลุ่ม Adults (31-50)
  adults: {
    weekday: { hour: 12, minute: 0 }, // 12:00 น. (พักกลางวัน)
    weekend: { hour: 10, minute: 0 }, // 10:00 น.
  },
  // กลุ่ม Senior (50+)
  senior: {
    weekday: { hour: 9, minute: 0 },  // 9:00 น.
    weekend: { hour: 9, minute: 0 },  // 9:00 น.
  },
};

function getBestSendTime(ageGroup) {
  const isWeekend = [0, 6].includes(new Date().getDay());
  const times = BEST_SEND_TIMES[ageGroup];
  return isWeekend ? times.weekend : times.weekday;
}
```

---

## 14. Complete Campaign Manager

```javascript
// campaignManager.js
const NarrowcastService = require('./narrowcastService');

class CampaignManager {
  constructor(accessToken) {
    this.service = new NarrowcastService(accessToken);
    this.activeCampaigns = new Map();
  }

  async launchCampaign(campaign) {
    const {
      id,
      name,
      messages,
      targetSegment,
      limit,
      schedule,
    } = campaign;

    console.log(`🚀 Launching campaign: ${name}`);

    const filter = this.buildFilter(targetSegment);

    const requestId = await this.service.send({
      messages,
      filter,
      limit: limit ? { upTo: limit } : undefined,
    });

    this.activeCampaigns.set(requestId, {
      id,
      name,
      requestId,
      startedAt: new Date(),
      status: 'running',
    });

    // Monitor ใน Background
    this.monitorCampaign(requestId, name);

    return requestId;
  }

  buildFilter(segment) {
    const conditions = [];

    if (segment.age) {
      conditions.push({
        type: 'age',
        ageMin: segment.age.min,
        ageMax: segment.age.max,
      });
    }

    if (segment.gender) {
      conditions.push({
        type: 'gender',
        oneOf: Array.isArray(segment.gender) ? segment.gender : [segment.gender],
      });
    }

    if (segment.region) {
      conditions.push({
        type: 'region',
        oneOf: Array.isArray(segment.region) ? segment.region : [segment.region],
      });
    }

    if (segment.os) {
      conditions.push({
        type: 'appType',
        oneOf: Array.isArray(segment.os) ? segment.os : [segment.os],
      });
    }

    if (segment.subscriptionDays) {
      conditions.push({
        type: 'subscriptionPeriod',
        gte: `day_${segment.subscriptionDays.min || 0}`,
        lte: segment.subscriptionDays.max ? `day_${segment.subscriptionDays.max}` : undefined,
      });
    }

    if (conditions.length === 0) return undefined;
    if (conditions.length === 1) return { demographic: conditions[0] };

    return {
      demographic: {
        type: 'and',
        demographicFilterConditions: conditions,
      },
    };
  }

  async monitorCampaign(requestId, name) {
    try {
      const result = await this.service.waitForCompletion(requestId);
      
      const campaign = this.activeCampaigns.get(requestId);
      if (campaign) {
        campaign.status = 'completed';
        campaign.result = result;
        campaign.completedAt = new Date();
      }

      console.log(`✅ Campaign "${name}" completed:`);
      console.log(`   Sent: ${result.successCount}`);
      console.log(`   Failed: ${result.failureCount}`);

      // แจ้งเตือน (เช่น ส่ง Email หรือ Slack)
      await this.notifyCompletion(name, result);
    } catch (error) {
      console.error(`❌ Campaign "${name}" failed:`, error.message);
    }
  }

  async notifyCompletion(campaignName, result) {
    console.log(`📢 Campaign "${campaignName}" notification sent`);
  }

  getCampaignStatus(requestId) {
    return this.activeCampaigns.get(requestId);
  }

  getAllCampaigns() {
    return Array.from(this.activeCampaigns.values());
  }
}

// ตัวอย่างการใช้งาน
const manager = new CampaignManager(process.env.LINE_CHANNEL_ACCESS_TOKEN);

// Campaign 1: โปรโมชั่นสำหรับผู้หญิงวัยทำงาน
manager.launchCampaign({
  id: 'camp001',
  name: 'Women Fashion Summer Sale',
  messages: [
    {
      type: 'text',
      text: '👗 Summer Sale 2024! ลด 30% ทุกรายการแฟชั่นผู้หญิง',
    },
  ],
  targetSegment: {
    gender: 'female',
    age: { min: 'age_25', max: 'age_40' },
    region: ['TH01', 'TH02', 'TH04'],
  },
  limit: 10000,
});

// Campaign 2: Re-engagement สำหรับสมาชิกเก่า
manager.launchCampaign({
  id: 'camp002',
  name: 'Re-engagement Campaign',
  messages: [
    {
      type: 'text',
      text: '💌 เราคิดถึงคุณ! กลับมาช้อปและรับโค้ดส่วนลด 25%',
    },
  ],
  targetSegment: {
    subscriptionDays: { min: 180 },
  },
  limit: 5000,
});
```

---

## สรุป

Narrowcast เป็นเครื่องมือที่ทรงพลังสำหรับการทำ Targeted Marketing ใน LINE:

1. **Demographic Filters** - กรองตาม Age, Gender, OS, Region, Subscription Period
2. **Audience-based** - ส่งตาม Custom Audience ที่สร้างเอง
3. **Combined Filters** - AND, OR, NOT Conditions
4. **Limit** - จำกัดจำนวนผู้รับ
5. **Async Processing** - Monitor Progress ได้

**Key Points:**
- ต้องมี Followers อย่างน้อย 100 คนที่ match criteria
- Narrowcast เป็น Async ต้องตรวจสอบ Progress แยก
- ใช้ Limit เพื่อควบคุมต้นทุน
- Test ด้วย Limit เล็กน้อยก่อนส่งจริง
- Monitor และวิเคราะห์ผลลัพธ์เสมอ
