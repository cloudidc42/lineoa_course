# ส่วนที่ 29: User Profile API และการจัดการข้อมูลผู้ใช้

---

## สารบัญ

1. [ภาพรวม User Profile API](#1-ภาพรวม-user-profile-api)
2. [Get User Profile API](#2-get-user-profile-api)
3. [Get Group Member Profile API](#3-get-group-member-profile-api)
4. [Get Room Member Profile API](#4-get-room-member-profile-api)
5. [Get Followers API](#5-get-followers-api)
6. [โครงสร้างข้อมูล Profile](#6-โครงสร้างข้อมูล-profile)
7. [การจัดเก็บข้อมูลผู้ใช้](#7-การจัดเก็บข้อมูลผู้ใช้)
8. [User Permission Concepts](#8-user-permission-concepts)
9. [PDPA และการปฏิบัติตามกฎหมาย](#9-pdpa-และการปฏิบัติตามกฎหมาย)
10. [Building User Database](#10-building-user-database)
11. [Code Examples ใน Node.js](#11-code-examples-ใน-nodejs)
12. [Code Examples ใน Python](#12-code-examples-ใน-python)
13. [Best Practices](#13-best-practices)
14. [แบบฝึกหัดท้ายบท](#14-แบบฝึกหัดท้ายบท)

---

## 1. ภาพรวม User Profile API

LINE Messaging API ให้ Bot สามารถดึงข้อมูลโปรไฟล์ของผู้ใช้ได้ทั้งใน 1:1 chat, Group และ Room ข้อมูลเหล่านี้มีความสำคัญสำหรับการให้บริการแบบ personalized และการวิเคราะห์ผู้ใช้

### สิ่งที่ Bot สามารถดึงได้

| ข้อมูล | API | เงื่อนไข |
|--------|-----|--------|
| Profile ผู้ใช้ (1:1) | getProfile | ผู้ใช้ต้อง Follow Bot |
| Profile ผู้ใช้ใน Group | getGroupMemberProfile | Bot ต้องอยู่ในกลุ่ม |
| Profile ผู้ใช้ใน Room | getRoomMemberProfile | Bot ต้องอยู่ใน Room |
| รายชื่อ Followers | getFollowers | ต้องใช้ Bot account |
| รายชื่อสมาชิก Group | getGroupMemberIds | Bot ต้องอยู่ในกลุ่ม |
| รายชื่อสมาชิก Room | getRoomMemberIds | Bot ต้องอยู่ใน Room |

### ข้อจำกัดที่สำคัญ

- **Bot ไม่สามารถดูเบอร์โทรหรืออีเมล** จาก LINE API
- **userId เปลี่ยนได้** ถ้าผู้ใช้ Revoke permissions
- **รูปโปรไฟล์** มี URL ที่หมดอายุ ไม่ควรเก็บ URL ไว้นาน
- **displayName** ผู้ใช้สามารถเปลี่ยนได้ตลอดเวลา

---

## 2. Get User Profile API

### Endpoint

```
GET https://api.line.me/v2/bot/profile/{userId}
```

### Headers

```
Authorization: Bearer {channel access token}
```

### Response

```json
{
  "displayName": "สมชาย ใจดี",
  "userId": "U4af4980629...",
  "language": "th",
  "pictureUrl": "https://profile.line-scdn.net/0h...",
  "statusMessage": "ยินดีให้บริการ",
  "followedAt": "20230101T000000.000Z"
}
```

### Response Fields

| Field | Type | Description |
|-------|------|-------------|
| displayName | String | ชื่อที่แสดงใน LINE |
| userId | String | User ID (ไม่ซ้ำกัน) |
| language | String | ภาษาที่ตั้งค่าไว้ |
| pictureUrl | String | URL รูปโปรไฟล์ |
| statusMessage | String | Status message |
| followedAt | String | วันที่ Follow (ISO 8601) |

### ตัวอย่าง cURL

```bash
curl -X GET "https://api.line.me/v2/bot/profile/U4af4980629..." \
  -H "Authorization: Bearer YOUR_CHANNEL_ACCESS_TOKEN"
```

---

## 3. Get Group Member Profile API

### Endpoint

```
GET https://api.line.me/v2/bot/group/{groupId}/member/{userId}
```

### Response

```json
{
  "displayName": "สมชาย ใจดี",
  "userId": "U4af4980629...",
  "pictureUrl": "https://profile.line-scdn.net/0h..."
}
```

### หมายเหตุ

- ข้อมูลที่ได้จาก Group Profile จะน้อยกว่า 1:1 Profile
- ไม่มี `language`, `statusMessage`, `followedAt`
- groupId ได้จาก Webhook event source

---

## 4. Get Room Member Profile API

### Endpoint

```
GET https://api.line.me/v2/bot/room/{roomId}/member/{userId}
```

### Response

```json
{
  "displayName": "สมหญิง รักเรียน",
  "userId": "Uf8d5d3b2c...",
  "pictureUrl": "https://profile.line-scdn.net/0h..."
}
```

---

## 5. Get Followers API

### Endpoint

```
GET https://api.line.me/v2/bot/followers/ids
```

### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| start | String | No | Continuation token |
| limit | Number | No | จำนวนต่อ page (สูงสุด 1000) |

### Response

```json
{
  "userIds": [
    "U4af4980629...",
    "Uf8d5d3b2c...",
    "U0d1c0d4e5..."
  ],
  "next": "continuation_token_here"
}
```

### วนดึง Followers ทั้งหมด

```javascript
async function getAllFollowers(client) {
  const allUserIds = [];
  let start = undefined;
  
  do {
    const response = await client.getFollowers(start);
    allUserIds.push(...response.userIds);
    start = response.next;
    
    console.log(`Fetched ${allUserIds.length} followers so far...`);
    
    // Rate limiting
    if (start) {
      await new Promise(resolve => setTimeout(resolve, 200));
    }
  } while (start);
  
  return allUserIds;
}
```

---

## 6. โครงสร้างข้อมูล Profile

### User Profile Object

```javascript
// โครงสร้างที่แนะนำสำหรับการเก็บข้อมูล
const UserProfile = {
  // จาก LINE API
  userId: String,           // Primary Key
  displayName: String,      // ชื่อ LINE (อาจเปลี่ยนได้)
  pictureUrl: String,       // URL รูปโปรไฟล์ (อาจ expired)
  statusMessage: String,    // Status message
  language: String,         // ภาษา
  followedAt: Date,         // วันที่ Follow
  
  // ข้อมูลที่เราเพิ่มเอง
  isFollowing: Boolean,     // กำลัง Follow อยู่หรือเปล่า
  unfollowedAt: Date,       // วันที่ Unfollow
  lastSeen: Date,           // ครั้งล่าสุดที่ส่งข้อความ
  messageCount: Number,     // จำนวนข้อความที่ส่ง
  tags: Array,              // Tags ที่กำหนดเอง
  notes: String,            // หมายเหตุจาก Admin
  
  // PDPA Consent
  pdpaConsent: {
    accepted: Boolean,
    acceptedAt: Date,
    version: String
  },
  
  // Metadata
  createdAt: Date,
  updatedAt: Date
};
```

### Group Member Profile Object

```javascript
const GroupMemberProfile = {
  userId: String,
  groupId: String,
  displayName: String,
  pictureUrl: String,
  role: String,           // 'owner', 'admin', 'member'
  joinedAt: Date,
  lastActiveAt: Date
};
```

---

## 7. การจัดเก็บข้อมูลผู้ใช้

### Database Schema (MySQL)

```sql
-- Users table
CREATE TABLE line_users (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id VARCHAR(50) NOT NULL UNIQUE,
  display_name VARCHAR(200),
  picture_url TEXT,
  status_message TEXT,
  language VARCHAR(10),
  followed_at TIMESTAMP,
  is_following BOOLEAN DEFAULT TRUE,
  unfollowed_at TIMESTAMP,
  last_seen TIMESTAMP,
  message_count INT DEFAULT 0,
  pdpa_accepted BOOLEAN DEFAULT FALSE,
  pdpa_accepted_at TIMESTAMP,
  pdpa_version VARCHAR(20),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  
  INDEX idx_is_following (is_following),
  INDEX idx_last_seen (last_seen),
  INDEX idx_followed_at (followed_at)
);

-- User Tags table
CREATE TABLE user_tags (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id VARCHAR(50) NOT NULL,
  tag VARCHAR(100) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  FOREIGN KEY (user_id) REFERENCES line_users(user_id),
  UNIQUE KEY unique_user_tag (user_id, tag)
);

-- User Interactions table
CREATE TABLE user_interactions (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id VARCHAR(50) NOT NULL,
  event_type VARCHAR(50) NOT NULL,
  event_data JSON,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  FOREIGN KEY (user_id) REFERENCES line_users(user_id),
  INDEX idx_user_event (user_id, event_type),
  INDEX idx_created_at (created_at)
);

-- PDPA Consent Log
CREATE TABLE pdpa_consent_log (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id VARCHAR(50) NOT NULL,
  action ENUM('accept', 'revoke') NOT NULL,
  version VARCHAR(20) NOT NULL,
  ip_address VARCHAR(45),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  FOREIGN KEY (user_id) REFERENCES line_users(user_id)
);
```

### Database Schema (MongoDB)

```javascript
// MongoDB Schema
const userSchema = {
  userId: { type: String, required: true, unique: true, index: true },
  displayName: { type: String, maxLength: 200 },
  pictureUrl: { type: String },
  statusMessage: { type: String },
  language: { type: String, maxLength: 10 },
  followedAt: { type: Date },
  isFollowing: { type: Boolean, default: true, index: true },
  unfollowedAt: { type: Date },
  lastSeen: { type: Date, index: true },
  messageCount: { type: Number, default: 0 },
  tags: [{ type: String }],
  notes: { type: String },
  
  pdpaConsent: {
    accepted: { type: Boolean, default: false },
    acceptedAt: { type: Date },
    version: { type: String },
    revokedAt: { type: Date }
  },
  
  metadata: {
    source: { type: String, enum: ['follow', 'message', 'import'] },
    segment: { type: String },
    customFields: { type: Map, of: String }
  },
  
  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now }
};
```

---

## 8. User Permission Concepts

### ระดับการเข้าถึงข้อมูล

```
ระดับ 1: Anonymous User
- Bot รู้จัก userId เท่านั้น
- ไม่มีข้อมูลอื่น

ระดับ 2: Follower
- Bot ดึง Profile ได้
- เห็น displayName, picture, language

ระดับ 3: Registered User
- ผู้ใช้ให้ข้อมูลเพิ่มเติม (ชื่อ, เบอร์, อีเมล)
- ผ่าน LINE Login หรือ Form

ระดับ 4: Verified User
- ยืนยันตัวตนแล้ว (OTP, บัตรประชาชน)
- เข้าถึงบริการระดับสูงได้
```

### การขอ Permission

LINE Bot ไม่ได้ขอ permission แบบ OAuth โดยตรง แต่สามารถใช้:

1. **LINE Login** - สำหรับ LIFF app เพื่อขอ profile + email
2. **Messaging API** - ดึง Profile ได้โดยตรงจาก Follow event
3. **User Form** - ให้ผู้ใช้กรอกข้อมูลผ่าน Bot หรือ Web

### LIFF (LINE Front-end Framework) Login

```javascript
// ใน LIFF App
liff.init({ liffId: 'YOUR_LIFF_ID' }).then(() => {
  if (!liff.isLoggedIn()) {
    liff.login();
  }
  
  // ดึงข้อมูล Profile
  liff.getProfile().then(profile => {
    console.log('User ID:', profile.userId);
    console.log('Display Name:', profile.displayName);
    console.log('Picture URL:', profile.pictureUrl);
    
    // ส่งข้อมูลไปยัง Backend
    sendProfileToBackend(profile);
  });
  
  // ดึง Email (ต้องขอ permission เพิ่มเติม)
  const decodedToken = liff.getDecodedIDToken();
  if (decodedToken) {
    console.log('Email:', decodedToken.email);
  }
});
```

---

## 9. PDPA และการปฏิบัติตามกฎหมาย

### พระราชบัญญัติคุ้มครองข้อมูลส่วนบุคคล (PDPA)

PDPA บังคับใช้ในไทยตั้งแต่ 1 มิถุนายน 2022 มีผลต่อการเก็บและใช้ข้อมูลของผู้ใช้ LINE

### ข้อมูลที่อยู่ภายใต้ PDPA จาก LINE

| ข้อมูล | ประเภท | ต้องขอ Consent |
|--------|--------|----------------|
| userId | Identifier | มีข้อถกเถียง |
| displayName | Personal | ใช่ |
| pictureUrl | Personal | ใช่ |
| statusMessage | Personal | ใช่ |
| เบอร์โทร (จาก form) | Personal | ใช่ |
| อีเมล (จาก LIFF) | Personal | ใช่ |
| พฤติกรรมการใช้งาน | Behavioral | ใช่ |

### หลักการสำคัญของ PDPA

```
1. Lawful Basis (ฐานทางกฎหมาย)
   - Consent (ความยินยอม)
   - Contract (สัญญา)
   - Legal obligation
   - Legitimate interests

2. Purpose Limitation (จำกัดวัตถุประสงค์)
   - เก็บเพื่อวัตถุประสงค์ที่แจ้งเท่านั้น

3. Data Minimization (เก็บเท่าที่จำเป็น)
   - ไม่เก็บข้อมูลที่ไม่จำเป็น

4. Accuracy (ความถูกต้อง)
   - อัปเดตข้อมูลให้เป็นปัจจุบัน

5. Storage Limitation (จำกัดระยะเวลาเก็บ)
   - ลบเมื่อหมดความจำเป็น

6. Data Subject Rights (สิทธิของเจ้าของข้อมูล)
   - Right to Access
   - Right to Rectification
   - Right to Erasure
   - Right to Portability
```

### Implementation ของ PDPA Consent

```javascript
// 1. ขอ Consent เมื่อผู้ใช้ Follow
async function requestPDPAConsent(replyToken, userId) {
  const consentMessage = {
    type: 'template',
    altText: 'นโยบายความเป็นส่วนตัว',
    template: {
      type: 'buttons',
      title: '📋 นโยบายความเป็นส่วนตัว',
      text: 'เราเก็บข้อมูล: ชื่อ, รูปโปรไฟล์, ประวัติการสนทนา เพื่อปรับปรุงบริการ\n\nอ่านรายละเอียดได้ที่ลิงค์ด้านล่าง',
      actions: [
        {
          type: 'postback',
          label: '✅ ยินยอม',
          data: 'action=pdpa_consent&version=1.0',
          displayText: 'ฉันยินยอมให้เก็บข้อมูล'
        },
        {
          type: 'postback',
          label: '❌ ไม่ยินยอม',
          data: 'action=pdpa_decline',
          displayText: 'ฉันไม่ยินยอม'
        },
        {
          type: 'uri',
          label: '📄 อ่านนโยบายเต็ม',
          uri: 'https://example.com/privacy-policy'
        }
      ]
    }
  };
  
  return client.replyMessage({
    replyToken,
    messages: [consentMessage]
  });
}

// 2. บันทึก Consent
async function recordPDPAConsent(userId, accepted, version = '1.0') {
  await db.users.update(
    { userId },
    {
      $set: {
        'pdpaConsent.accepted': accepted,
        'pdpaConsent.acceptedAt': accepted ? new Date() : null,
        'pdpaConsent.version': version,
        updatedAt: new Date()
      }
    },
    { upsert: true }
  );
  
  // บันทึก audit log
  await db.pdpaLogs.insert({
    userId,
    action: accepted ? 'accept' : 'decline',
    version,
    timestamp: new Date()
  });
}

// 3. ตรวจสอบก่อนเข้าถึงข้อมูล
async function checkPDPAConsent(userId) {
  const user = await db.users.findOne({ userId });
  return user?.pdpaConsent?.accepted === true;
}

// 4. จัดการสิทธิ "ขอลบข้อมูล" (Right to Erasure)
async function handleDeleteRequest(userId) {
  // Anonymize data แทนการลบทั้งหมด (เพื่อ audit trail)
  await db.users.update(
    { userId },
    {
      $set: {
        displayName: 'DELETED',
        pictureUrl: null,
        statusMessage: null,
        'pdpaConsent.revokedAt': new Date(),
        'pdpaConsent.accepted': false,
        deletedAt: new Date()
      }
    }
  );
  
  // ลบข้อมูล interaction history
  await db.interactions.deleteMany({ userId });
  
  return true;
}
```

---

## 10. Building User Database

### User Service ที่ครบถ้วน

```javascript
'use strict';

const axios = require('axios');

class UserService {
  constructor(db, lineClient) {
    this.db = db;
    this.lineClient = lineClient;
  }
  
  /**
   * ดึงและบันทึก User Profile
   */
  async fetchAndStoreProfile(userId) {
    try {
      const profile = await this.lineClient.getProfile(userId);
      
      const userData = {
        userId: profile.userId,
        displayName: profile.displayName,
        pictureUrl: profile.pictureUrl,
        statusMessage: profile.statusMessage || null,
        language: profile.language || null,
        followedAt: profile.followedAt ? new Date(profile.followedAt) : null,
        isFollowing: true,
        lastSeen: new Date(),
        updatedAt: new Date()
      };
      
      // Upsert - สร้างถ้าไม่มี, อัปเดตถ้ามี
      await this.db.users.findOneAndUpdate(
        { userId },
        {
          $set: userData,
          $setOnInsert: {
            messageCount: 0,
            createdAt: new Date()
          }
        },
        { upsert: true, returnDocument: 'after' }
      );
      
      return userData;
    } catch (error) {
      if (error.statusCode === 404) {
        // User ไม่ได้ Follow Bot แล้ว
        await this.markAsUnfollowed(userId);
        return null;
      }
      throw error;
    }
  }
  
  /**
   * บันทึก Follow Event
   */
  async handleFollow(userId) {
    try {
      const profile = await this.lineClient.getProfile(userId);
      
      const now = new Date();
      
      await this.db.users.findOneAndUpdate(
        { userId },
        {
          $set: {
            displayName: profile.displayName,
            pictureUrl: profile.pictureUrl,
            statusMessage: profile.statusMessage || null,
            language: profile.language || null,
            isFollowing: true,
            lastSeen: now,
            updatedAt: now
          },
          $setOnInsert: {
            followedAt: now,
            messageCount: 0,
            createdAt: now
          }
        },
        { upsert: true }
      );
      
      // บันทึก interaction
      await this.logInteraction(userId, 'follow', {
        displayName: profile.displayName
      });
      
      console.log(`New follower: ${profile.displayName} (${userId})`);
      
    } catch (error) {
      console.error('Error handling follow:', error);
      throw error;
    }
  }
  
  /**
   * บันทึก Unfollow Event
   */
  async handleUnfollow(userId) {
    await this.db.users.updateOne(
      { userId },
      {
        $set: {
          isFollowing: false,
          unfollowedAt: new Date(),
          updatedAt: new Date()
        }
      }
    );
    
    await this.logInteraction(userId, 'unfollow');
  }
  
  /**
   * อัปเดต Last Seen
   */
  async updateLastSeen(userId) {
    await this.db.users.updateOne(
      { userId },
      {
        $set: { lastSeen: new Date(), updatedAt: new Date() },
        $inc: { messageCount: 1 }
      }
    );
  }
  
  /**
   * Mark as Unfollowed
   */
  async markAsUnfollowed(userId) {
    await this.db.users.updateOne(
      { userId },
      {
        $set: {
          isFollowing: false,
          updatedAt: new Date()
        }
      }
    );
  }
  
  /**
   * ดึงข้อมูลผู้ใช้จาก DB
   */
  async getUser(userId) {
    return this.db.users.findOne({ userId });
  }
  
  /**
   * ดึงรายชื่อ Active Followers
   */
  async getActiveFollowers(daysBack = 7) {
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - daysBack);
    
    return this.db.users.find({
      isFollowing: true,
      lastSeen: { $gte: cutoffDate }
    }).toArray();
  }
  
  /**
   * เพิ่ม Tag ให้ผู้ใช้
   */
  async addTag(userId, tag) {
    await this.db.users.updateOne(
      { userId },
      {
        $addToSet: { tags: tag },
        $set: { updatedAt: new Date() }
      }
    );
  }
  
  /**
   * ลบ Tag ของผู้ใช้
   */
  async removeTag(userId, tag) {
    await this.db.users.updateOne(
      { userId },
      {
        $pull: { tags: tag },
        $set: { updatedAt: new Date() }
      }
    );
  }
  
  /**
   * ดึงผู้ใช้ที่มี Tag
   */
  async getUsersByTag(tag) {
    return this.db.users.find({
      isFollowing: true,
      tags: tag
    }).toArray();
  }
  
  /**
   * บันทึก Interaction
   */
  async logInteraction(userId, eventType, data = {}) {
    await this.db.interactions.insertOne({
      userId,
      eventType,
      data,
      timestamp: new Date()
    });
  }
  
  /**
   * ดึง Followers ทั้งหมดจาก LINE API
   */
  async syncFollowers() {
    console.log('Starting followers sync...');
    
    let start = undefined;
    let syncedCount = 0;
    
    do {
      const response = await this.lineClient.getFollowers(start);
      
      for (const userId of response.userIds) {
        try {
          await this.fetchAndStoreProfile(userId);
          syncedCount++;
          
          // Rate limiting
          await new Promise(resolve => setTimeout(resolve, 50));
        } catch (error) {
          console.error(`Failed to sync user ${userId}:`, error.message);
        }
      }
      
      start = response.next;
      console.log(`Synced ${syncedCount} users...`);
      
    } while (start);
    
    console.log(`Sync complete. Total: ${syncedCount} users`);
    return syncedCount;
  }
  
  /**
   * ส่งออกข้อมูลผู้ใช้ (สำหรับ PDPA Right to Portability)
   */
  async exportUserData(userId) {
    const user = await this.getUser(userId);
    if (!user) return null;
    
    const interactions = await this.db.interactions
      .find({ userId })
      .sort({ timestamp: -1 })
      .limit(1000)
      .toArray();
    
    return {
      profile: {
        userId: user.userId,
        displayName: user.displayName,
        language: user.language,
        followedAt: user.followedAt,
        isFollowing: user.isFollowing
      },
      statistics: {
        messageCount: user.messageCount,
        firstSeen: user.createdAt,
        lastSeen: user.lastSeen
      },
      interactions: interactions.map(i => ({
        type: i.eventType,
        timestamp: i.timestamp
      })),
      exportedAt: new Date()
    };
  }
  
  /**
   * ลบข้อมูลผู้ใช้ (Right to Erasure)
   */
  async deleteUserData(userId) {
    // Anonymize แทนการลบ
    await this.db.users.updateOne(
      { userId },
      {
        $set: {
          displayName: 'ANONYMIZED',
          pictureUrl: null,
          statusMessage: null,
          isFollowing: false,
          deletedAt: new Date(),
          updatedAt: new Date()
        },
        $unset: {
          language: '',
          tags: '',
          notes: ''
        }
      }
    );
    
    // ลบ interaction log (ไม่เกี่ยวกับ audit)
    await this.db.interactions.deleteMany({ userId });
    
    return true;
  }
}

module.exports = UserService;
```

---

## 11. Code Examples ใน Node.js

### services/profileService.js

```javascript
'use strict';

require('dotenv').config();
const line = require('@line/bot-sdk');

const client = new line.messagingApi.MessagingApiClient({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

/**
 * ดึง User Profile
 */
async function getUserProfile(userId) {
  try {
    const profile = await client.getProfile(userId);
    return {
      userId: profile.userId,
      displayName: profile.displayName,
      pictureUrl: profile.pictureUrl,
      statusMessage: profile.statusMessage,
      language: profile.language,
      followedAt: profile.followedAt
    };
  } catch (error) {
    if (error.statusCode === 404) {
      console.log(`User ${userId} not found or not following`);
      return null;
    }
    throw error;
  }
}

/**
 * ดึง Group Member Profile
 */
async function getGroupMemberProfile(groupId, userId) {
  try {
    const profile = await client.getGroupMemberProfile(groupId, userId);
    return {
      userId: profile.userId,
      displayName: profile.displayName,
      pictureUrl: profile.pictureUrl
    };
  } catch (error) {
    console.error(`Error getting group member profile:`, error);
    return null;
  }
}

/**
 * ดึง Room Member Profile
 */
async function getRoomMemberProfile(roomId, userId) {
  try {
    const profile = await client.getRoomMemberProfile(roomId, userId);
    return {
      userId: profile.userId,
      displayName: profile.displayName,
      pictureUrl: profile.pictureUrl
    };
  } catch (error) {
    console.error(`Error getting room member profile:`, error);
    return null;
  }
}

/**
 * ดึงรายชื่อสมาชิกกลุ่ม
 */
async function getGroupMemberIds(groupId) {
  const allIds = [];
  let start = undefined;
  
  do {
    const response = await client.getGroupMemberIds(groupId, start);
    allIds.push(...response.memberIds);
    start = response.next;
    
    if (start) {
      await new Promise(resolve => setTimeout(resolve, 200));
    }
  } while (start);
  
  return allIds;
}

/**
 * ดึง Profile แบบ Batch
 */
async function getBatchProfiles(userIds, delayMs = 100) {
  const profiles = [];
  
  for (const userId of userIds) {
    try {
      const profile = await getUserProfile(userId);
      if (profile) profiles.push(profile);
      await new Promise(resolve => setTimeout(resolve, delayMs));
    } catch (error) {
      console.error(`Failed to get profile for ${userId}:`, error.message);
    }
  }
  
  return profiles;
}

/**
 * ตรวจสอบว่าผู้ใช้ยัง Follow อยู่หรือเปล่า
 */
async function isUserFollowing(userId) {
  try {
    await client.getProfile(userId);
    return true;
  } catch (error) {
    if (error.statusCode === 404) return false;
    throw error;
  }
}

module.exports = {
  getUserProfile,
  getGroupMemberProfile,
  getRoomMemberProfile,
  getGroupMemberIds,
  getBatchProfiles,
  isUserFollowing
};
```

### handlers/profileHandler.js

```javascript
'use strict';

const profileService = require('../services/profileService');
const userService = require('../services/userService');
const lineService = require('../services/lineService');

/**
 * จัดการ Follow Event - ดึงและบันทึก Profile
 */
async function handleFollowWithProfile(event) {
  const { replyToken, source } = event;
  const userId = source.userId;
  
  try {
    // ดึง Profile
    const profile = await profileService.getUserProfile(userId);
    
    if (!profile) {
      return lineService.replyMessage(replyToken, 'ขอบคุณที่ติดตาม!');
    }
    
    // บันทึกลง DB
    await userService.handleFollow(userId, profile);
    
    // ส่งข้อความต้อนรับแบบ personalized
    const welcomeText = `🎉 สวัสดีคุณ ${profile.displayName}!\n\nขอบคุณที่ติดตาม LINE ของเรานะครับ\nเรายินดีให้บริการคุณเสมอ`;
    
    // ขอ PDPA Consent
    const pdpaTemplate = {
      type: 'template',
      altText: 'นโยบายความเป็นส่วนตัว',
      template: {
        type: 'confirm',
        text: 'เราขอเก็บข้อมูลโปรไฟล์ของคุณเพื่อปรับปรุงบริการ\n\nคุณยินยอมหรือไม่?',
        actions: [
          {
            type: 'postback',
            label: '✅ ยินยอม',
            data: `action=pdpa_consent&userId=${userId}&version=1.0`
          },
          {
            type: 'postback',
            label: '❌ ไม่ยินยอม',
            data: `action=pdpa_decline&userId=${userId}`
          }
        ]
      }
    };
    
    return lineService.replyMessage(replyToken, [
      { type: 'text', text: welcomeText },
      pdpaTemplate
    ]);
    
  } catch (error) {
    console.error('Error in handleFollowWithProfile:', error);
    return lineService.replyMessage(replyToken, '🎉 ยินดีต้อนรับ!');
  }
}

/**
 * แสดงข้อมูลโปรไฟล์ผู้ใช้
 */
async function showUserProfile(replyToken, userId) {
  try {
    const profile = await profileService.getUserProfile(userId);
    const dbUser = await userService.getUser(userId);
    
    if (!profile) {
      return lineService.replyMessage(replyToken, 'ไม่สามารถดึงข้อมูลโปรไฟล์ได้');
    }
    
    const followDate = profile.followedAt
      ? new Date(profile.followedAt).toLocaleDateString('th-TH')
      : 'ไม่ทราบ';
    
    const profileText = `👤 ข้อมูลโปรไฟล์ของคุณ\n\n`
      + `ชื่อ: ${profile.displayName}\n`
      + `ภาษา: ${profile.language || 'ไม่ระบุ'}\n`
      + `Status: ${profile.statusMessage || '-'}\n`
      + `ติดตามเมื่อ: ${followDate}\n`
      + `ข้อความทั้งหมด: ${dbUser?.messageCount || 0} ข้อความ`;
    
    return lineService.replyMessage(replyToken, [
      { type: 'text', text: profileText }
    ]);
    
  } catch (error) {
    console.error('Error showing profile:', error);
    return lineService.replyMessage(replyToken, '❌ เกิดข้อผิดพลาด กรุณาลองใหม่');
  }
}

/**
 * จัดการ Postback PDPA Consent
 */
async function handlePDPAConsent(event) {
  const { replyToken, postback, source } = event;
  const params = new URLSearchParams(postback.data);
  const action = params.get('action');
  const userId = source.userId;
  
  if (action === 'pdpa_consent') {
    const version = params.get('version');
    
    await userService.recordPDPAConsent(userId, true, version);
    
    return lineService.replyMessage(replyToken, 
      '✅ ขอบคุณที่ยินยอม!\n\nเราจะดูแลข้อมูลของคุณอย่างปลอดภัย\nคุณสามารถขอลบข้อมูลได้ตลอดเวลา'
    );
  }
  
  if (action === 'pdpa_decline') {
    await userService.recordPDPAConsent(userId, false);
    
    return lineService.replyMessage(replyToken,
      '📋 รับทราบแล้วครับ\n\nเราจะเก็บเฉพาะข้อมูลที่จำเป็นเท่านั้น\nคุณยังสามารถใช้บริการพื้นฐานได้ตามปกติ'
    );
  }
}

module.exports = {
  handleFollowWithProfile,
  showUserProfile,
  handlePDPAConsent
};
```

---

## 12. Code Examples ใน Python

### services/profile_service.py

```python
import logging
import asyncio
from typing import Optional, List, Dict
from linebot.v3.messaging import (
    Configuration, ApiClient, MessagingApi
)
from linebot.v3.exceptions import ApiException
from dotenv import load_dotenv
import os

load_dotenv()

logger = logging.getLogger(__name__)

configuration = Configuration(
    access_token=os.environ.get('LINE_CHANNEL_ACCESS_TOKEN')
)


def get_messaging_api() -> MessagingApi:
    return MessagingApi(ApiClient(configuration))


def get_user_profile(user_id: str) -> Optional[Dict]:
    """
    ดึง User Profile
    
    Args:
        user_id: LINE userId
    
    Returns:
        Dict หรือ None ถ้าไม่พบ
    """
    try:
        api = get_messaging_api()
        profile = api.get_profile(user_id)
        
        return {
            'user_id': profile.user_id,
            'display_name': profile.display_name,
            'picture_url': profile.picture_url,
            'status_message': profile.status_message,
            'language': profile.language,
            'followed_at': profile.followed_at
        }
    except ApiException as e:
        if e.status == 404:
            logger.info(f"User {user_id} not found or not following")
            return None
        logger.error(f"Error getting profile for {user_id}: {e}")
        raise


def get_group_member_profile(group_id: str, user_id: str) -> Optional[Dict]:
    """ดึง Group Member Profile"""
    try:
        api = get_messaging_api()
        profile = api.get_group_member_profile(group_id, user_id)
        
        return {
            'user_id': profile.user_id,
            'display_name': profile.display_name,
            'picture_url': profile.picture_url
        }
    except ApiException as e:
        logger.error(f"Error getting group member profile: {e}")
        return None


def get_room_member_profile(room_id: str, user_id: str) -> Optional[Dict]:
    """ดึง Room Member Profile"""
    try:
        api = get_messaging_api()
        profile = api.get_room_member_profile(room_id, user_id)
        
        return {
            'user_id': profile.user_id,
            'display_name': profile.display_name,
            'picture_url': profile.picture_url
        }
    except ApiException as e:
        logger.error(f"Error getting room member profile: {e}")
        return None


def get_group_member_ids(group_id: str) -> List[str]:
    """ดึงรายชื่อสมาชิกกลุ่มทั้งหมด"""
    api = get_messaging_api()
    all_ids = []
    start = None
    
    while True:
        response = api.get_group_member_ids(group_id, start=start)
        all_ids.extend(response.member_ids)
        
        start = response.next
        if not start:
            break
        
        import time
        time.sleep(0.2)
    
    return all_ids


def get_batch_profiles(user_ids: List[str], delay_ms: int = 100) -> List[Dict]:
    """ดึง Profile หลายคนพร้อมกัน"""
    import time
    
    profiles = []
    
    for user_id in user_ids:
        try:
            profile = get_user_profile(user_id)
            if profile:
                profiles.append(profile)
            time.sleep(delay_ms / 1000)
        except Exception as e:
            logger.error(f"Failed to get profile for {user_id}: {e}")
    
    return profiles


def is_user_following(user_id: str) -> bool:
    """ตรวจสอบว่าผู้ใช้ยัง Follow อยู่หรือเปล่า"""
    try:
        get_user_profile(user_id)
        return True
    except ApiException as e:
        if e.status == 404:
            return False
        raise
```

### services/user_db_service.py

```python
import logging
from datetime import datetime, timedelta
from typing import Optional, List, Dict
from pymongo import MongoClient, DESCENDING

logger = logging.getLogger(__name__)


class UserDBService:
    def __init__(self, mongo_uri: str, db_name: str):
        self.client = MongoClient(mongo_uri)
        self.db = self.client[db_name]
        self.users = self.db.users
        self.interactions = self.db.interactions
        self.pdpa_logs = self.db.pdpa_logs
        
        # สร้าง indexes
        self.users.create_index('user_id', unique=True)
        self.users.create_index('is_following')
        self.users.create_index('last_seen')
    
    def upsert_user(self, user_id: str, profile: Dict) -> Dict:
        """เพิ่มหรืออัปเดต User"""
        now = datetime.now()
        
        result = self.users.find_one_and_update(
            {'user_id': user_id},
            {
                '$set': {
                    'display_name': profile.get('display_name'),
                    'picture_url': profile.get('picture_url'),
                    'status_message': profile.get('status_message'),
                    'language': profile.get('language'),
                    'is_following': True,
                    'last_seen': now,
                    'updated_at': now
                },
                '$setOnInsert': {
                    'user_id': user_id,
                    'followed_at': profile.get('followed_at', now),
                    'message_count': 0,
                    'tags': [],
                    'created_at': now
                }
            },
            upsert=True,
            return_document=True
        )
        
        return result
    
    def handle_follow(self, user_id: str, profile: Dict):
        """บันทึก Follow Event"""
        now = datetime.now()
        
        self.users.update_one(
            {'user_id': user_id},
            {
                '$set': {
                    'display_name': profile.get('display_name'),
                    'picture_url': profile.get('picture_url'),
                    'status_message': profile.get('status_message'),
                    'language': profile.get('language'),
                    'is_following': True,
                    'last_seen': now,
                    'updated_at': now
                },
                '$setOnInsert': {
                    'user_id': user_id,
                    'followed_at': now,
                    'message_count': 0,
                    'tags': [],
                    'created_at': now,
                    'pdpa_consent': {
                        'accepted': False,
                        'accepted_at': None,
                        'version': None
                    }
                }
            },
            upsert=True
        )
        
        self.log_interaction(user_id, 'follow', {'display_name': profile.get('display_name')})
    
    def handle_unfollow(self, user_id: str):
        """บันทึก Unfollow Event"""
        self.users.update_one(
            {'user_id': user_id},
            {
                '$set': {
                    'is_following': False,
                    'unfollowed_at': datetime.now(),
                    'updated_at': datetime.now()
                }
            }
        )
        
        self.log_interaction(user_id, 'unfollow')
    
    def update_last_seen(self, user_id: str):
        """อัปเดต Last Seen"""
        self.users.update_one(
            {'user_id': user_id},
            {
                '$set': {'last_seen': datetime.now(), 'updated_at': datetime.now()},
                '$inc': {'message_count': 1}
            }
        )
    
    def record_pdpa_consent(self, user_id: str, accepted: bool, version: str = '1.0'):
        """บันทึก PDPA Consent"""
        now = datetime.now()
        
        self.users.update_one(
            {'user_id': user_id},
            {
                '$set': {
                    'pdpa_consent.accepted': accepted,
                    'pdpa_consent.accepted_at': now if accepted else None,
                    'pdpa_consent.version': version,
                    'updated_at': now
                }
            }
        )
        
        # บันทึก audit log
        self.pdpa_logs.insert_one({
            'user_id': user_id,
            'action': 'accept' if accepted else 'decline',
            'version': version,
            'timestamp': now
        })
    
    def get_user(self, user_id: str) -> Optional[Dict]:
        """ดึงข้อมูล User"""
        return self.users.find_one({'user_id': user_id})
    
    def get_active_followers(self, days_back: int = 7) -> List[Dict]:
        """ดึง Active Followers"""
        cutoff = datetime.now() - timedelta(days=days_back)
        return list(self.users.find({
            'is_following': True,
            'last_seen': {'$gte': cutoff}
        }))
    
    def add_tag(self, user_id: str, tag: str):
        """เพิ่ม Tag"""
        self.users.update_one(
            {'user_id': user_id},
            {'$addToSet': {'tags': tag}, '$set': {'updated_at': datetime.now()}}
        )
    
    def get_users_by_tag(self, tag: str) -> List[Dict]:
        """ดึงผู้ใช้ที่มี Tag"""
        return list(self.users.find({'is_following': True, 'tags': tag}))
    
    def log_interaction(self, user_id: str, event_type: str, data: Dict = None):
        """บันทึก Interaction"""
        self.interactions.insert_one({
            'user_id': user_id,
            'event_type': event_type,
            'data': data or {},
            'timestamp': datetime.now()
        })
    
    def delete_user_data(self, user_id: str):
        """ลบข้อมูล User (PDPA Right to Erasure)"""
        self.users.update_one(
            {'user_id': user_id},
            {
                '$set': {
                    'display_name': 'ANONYMIZED',
                    'picture_url': None,
                    'status_message': None,
                    'is_following': False,
                    'deleted_at': datetime.now(),
                    'updated_at': datetime.now()
                },
                '$unset': {
                    'language': '',
                    'tags': '',
                    'notes': ''
                }
            }
        )
        
        self.interactions.delete_many({'user_id': user_id})
        logger.info(f"User data deleted/anonymized: {user_id}")
```

---

## 13. Best Practices

### การเก็บข้อมูล Profile

1. **เก็บ userId เป็น Primary Key** - userId คงที่ตลอด
2. **อัปเดต displayName สม่ำเสมอ** - ผู้ใช้อาจเปลี่ยนชื่อ
3. **ไม่เก็บ pictureUrl ระยะยาว** - URL มี expiration
4. **เก็บ followedAt** - มีประโยชน์สำหรับ analytics
5. **Index อย่างเหมาะสม** - isFollowing, lastSeen

### Security

```javascript
// ✅ ดี: ตรวจสอบ userId ก่อนใช้
async function getProfile(userId) {
  // ตรวจสอบ format ของ userId
  if (!/^U[0-9a-f]{32}$/.test(userId)) {
    throw new Error('Invalid userId format');
  }
  
  return await lineClient.getProfile(userId);
}

// ❌ ไม่ดี: ใช้ userId โดยตรง
async function getProfile(userId) {
  return await lineClient.getProfile(userId);  // อาจมี injection
}
```

### PDPA Checklist

- [ ] มี Privacy Policy ที่อ่านได้ง่าย
- [ ] ขอ Consent ก่อนเก็บข้อมูล
- [ ] บันทึก Consent log พร้อม timestamp
- [ ] มีระบบให้ผู้ใช้ขอดูข้อมูลของตัวเอง
- [ ] มีระบบให้ผู้ใช้ขอลบข้อมูล
- [ ] จำกัดการเข้าถึงข้อมูล (Role-based)
- [ ] เข้ารหัสข้อมูลที่ sensitive
- [ ] มี Data Breach response plan

---

## 14. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Profile Fetcher
สร้าง Bot ที่:
- เมื่อมีคน Follow → ดึง Profile และแสดงข้อความต้อนรับพร้อมชื่อ
- เมื่อผู้ใช้พิมพ์ "โปรไฟล์" → แสดงข้อมูลของตัวเอง
- เก็บข้อมูลลงใน JSON file หรือ SQLite

### แบบฝึกหัดที่ 2: User Segmentation
สร้างระบบแบ่ง Segment ผู้ใช้:
- Active (ส่งข้อความใน 7 วัน)
- Inactive (ไม่ส่งข้อความนาน 30 วัน)
- New (Follow ภายใน 7 วัน)
- ส่ง Multicast ไปยังแต่ละ segment

### แบบฝึกหัดที่ 3: PDPA Consent Flow
สร้าง PDPA Flow ที่:
- แสดง Privacy Policy
- ขอ Consent
- บันทึก Consent พร้อม timestamp
- มีคำสั่ง "ลบข้อมูลของฉัน"

---

**สรุปบทเรียน:**

User Profile API ช่วยให้ Bot สามารถ:

1. **ดึงข้อมูล Profile** จาก 1:1, Group, Room
2. **เก็บข้อมูลผู้ใช้** อย่างเป็นระบบ
3. **Personalize บริการ** ด้วยชื่อและข้อมูลของผู้ใช้
4. **ปฏิบัติตาม PDPA** ด้วยการขอ Consent และให้สิทธิ์ผู้ใช้

การเก็บข้อมูลต้องเป็นไปตาม PDPA ซึ่งต้องขอ Consent ก่อน และต้องให้สิทธิ์ผู้ใช้ในการดู/ลบข้อมูลของตัวเอง
