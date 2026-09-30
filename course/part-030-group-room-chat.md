# ส่วนที่ 30: Group, Room และ Multi-person Chat

---

## สารบัญ

1. [ภาพรวม Group vs Room vs Multi-person Chat](#1-ภาพรวม-group-vs-room-vs-multi-person-chat)
2. [Bot ใน Group](#2-bot-ใน-group)
3. [Bot ใน Room](#3-bot-ใน-room)
4. [Getting Group Member List](#4-getting-group-member-list)
5. [Getting Room Member List](#5-getting-room-member-list)
6. [Leave Group/Room](#6-leave-grouproom)
7. [Group Chat Event Handling](#7-group-chat-event-handling)
8. [Use Cases สำหรับ Group Bots](#8-use-cases-สำหรับ-group-bots)
9. [Notification-only Bots](#9-notification-only-bots)
10. [Administrative Bots](#10-administrative-bots)
11. [Code Examples ใน Node.js](#11-code-examples-ใน-nodejs)
12. [Code Examples ใน Python](#12-code-examples-ใน-python)
13. [Best Practices](#13-best-practices)
14. [แบบฝึกหัดท้ายบท](#14-แบบฝึกหัดท้ายบท)

---

## 1. ภาพรวม Group vs Room vs Multi-person Chat

LINE มีรูปแบบ Chat แบบหลายคนอยู่ 3 ประเภท ซึ่งมีความแตกต่างที่สำคัญ

### เปรียบเทียบ Group vs Room vs Multi-person

| ลักษณะ | Group | Room | Multi-person Chat |
|-------|-------|------|-------------------|
| สร้างโดย | ผู้ใช้ | ผู้ใช้ | ระบบ (เมื่อเชิญคนเข้าแชท) |
| จำนวนสมาชิก | ไม่จำกัด | สูงสุด 50 คน | สูงสุด 50 คน |
| Bot เข้าร่วมได้ | ✅ | ✅ | ❌ |
| ชื่อกลุ่ม | กำหนดได้ | ไม่มี | ไม่มี |
| Link เชิญ | ✅ | ❌ | ❌ |
| การ broadcast | ✅ | ✅ | ❌ |
| ดูสมาชิกได้ | ✅ (API) | ✅ (API) | ❌ |
| Source Type | `group` | `room` | `user` |

### Source Types ใน Webhook

```json
// Group Chat
{
  "source": {
    "type": "group",
    "groupId": "Ca56f94637cc4347f90a6fb46f1c3543e",
    "userId": "U4af4980629..."
  }
}

// Room Chat
{
  "source": {
    "type": "room",
    "roomId": "Ra8dbf4673c4c812cd491258042226c99",
    "userId": "U4af4980629..."
  }
}

// 1:1 Chat
{
  "source": {
    "type": "user",
    "userId": "U4af4980629..."
  }
}
```

---

## 2. Bot ใน Group

### การเพิ่ม Bot เข้ากลุ่ม

1. ผู้ใช้คลิก "เพิ่มเพื่อน" ที่ Bot
2. เลือก "เพิ่มในกลุ่ม"
3. เลือกกลุ่มที่ต้องการ

### Events ที่เกิดขึ้นใน Group

| Event | เกิดเมื่อ | มี replyToken |
|-------|---------|--------------|
| join | Bot ถูกเพิ่มเข้ากลุ่ม | ✅ |
| leave | Bot ถูกลบออกจากกลุ่ม | ❌ |
| message | สมาชิกส่งข้อความ | ✅ |
| memberJoined | สมาชิกใหม่เข้ากลุ่ม | ✅ |
| memberLeft | สมาชิกออกจากกลุ่ม | ❌ |
| postback | ผู้ใช้กดปุ่ม Postback | ✅ |
| follow | - | - |
| unfollow | - | - |

### Webhook Event ตัวอย่าง

```json
// Join Event (Bot ถูกเพิ่มเข้ากลุ่ม)
{
  "type": "join",
  "timestamp": 1625665242211,
  "source": {
    "type": "group",
    "groupId": "Ca56f94637cc4347f90a6fb46f1c3543e"
  },
  "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA",
  "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
  "deliveryContext": {
    "isRedelivery": false
  },
  "mode": "active"
}

// Member Joined Event
{
  "type": "memberJoined",
  "timestamp": 1625665242211,
  "source": {
    "type": "group",
    "groupId": "Ca56f94637cc4347f90a6fb46f1c3543e"
  },
  "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA",
  "joined": {
    "members": [
      {
        "type": "user",
        "userId": "U4af4980629..."
      }
    ]
  }
}

// Message Event ใน Group
{
  "type": "message",
  "timestamp": 1625665242211,
  "source": {
    "type": "group",
    "groupId": "Ca56f94637cc4347f90a6fb46f1c3543e",
    "userId": "U4af4980629..."
  },
  "message": {
    "type": "text",
    "id": "14353798921116",
    "text": "@Bot สวัสดี"
  },
  "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA"
}
```

### การรู้ว่า Bot ถูก Mention

```javascript
/**
 * ตรวจสอบว่า Bot ถูก @ mention หรือเปล่า
 * LINE ไม่มี official @mention detection แต่ใช้ pattern matching
 */
function isBotMentioned(text, botDisplayName) {
  const mentionPatterns = [
    `@${botDisplayName}`,
    `@Bot`,
    `@bot`
  ];
  
  return mentionPatterns.some(pattern => 
    text.toLowerCase().includes(pattern.toLowerCase())
  );
}

/**
 * ตัดคำ @mention ออกจากข้อความ
 */
function removeMention(text, botDisplayName) {
  return text
    .replace(new RegExp(`@${botDisplayName}`, 'gi'), '')
    .replace(/@Bot/gi, '')
    .trim();
}
```

---

## 3. Bot ใน Room

### การสร้าง Room

Room สร้างโดยผู้ใช้เมื่อเชิญคนเข้า 1:1 chat และผู้ใช้อีกคนยอมรับ

### ความแตกต่างจาก Group

- Room ไม่มีชื่อ
- Room จำกัดที่ 50 คน
- Room ไม่มี Link เชิญ
- API สำหรับ Room คล้ายกับ Group แต่ใช้ `roomId` แทน `groupId`

### Room Events

```json
// Join Event ใน Room
{
  "type": "join",
  "timestamp": 1625665242211,
  "source": {
    "type": "room",
    "roomId": "Ra8dbf4673c4c812cd491258042226c99"
  },
  "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA"
}
```

---

## 4. Getting Group Member List

### API Endpoint

```
GET https://api.line.me/v2/bot/group/{groupId}/members/ids
```

### Query Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| start | String | No | Continuation token |

### Response

```json
{
  "memberIds": [
    "U4af4980629...",
    "Uf8d5d3b2c...",
    "U0d1c0d4e5..."
  ],
  "next": "continuation_token"
}
```

### ดึงสมาชิกกลุ่มทั้งหมด

```javascript
async function getAllGroupMembers(client, groupId) {
  const allMemberIds = [];
  let start = undefined;
  
  do {
    const response = await client.getGroupMemberIds(groupId, start);
    allMemberIds.push(...response.memberIds);
    start = response.next;
    
    console.log(`Fetched ${allMemberIds.length} members...`);
    
    if (start) {
      await new Promise(resolve => setTimeout(resolve, 100));
    }
  } while (start);
  
  return allMemberIds;
}
```

### ดึงข้อมูล Profile ของสมาชิกทั้งหมด

```javascript
async function getGroupMembersWithProfile(client, groupId) {
  const memberIds = await getAllGroupMembers(client, groupId);
  const profiles = [];
  
  for (const userId of memberIds) {
    try {
      const profile = await client.getGroupMemberProfile(groupId, userId);
      profiles.push({
        userId: profile.userId,
        displayName: profile.displayName,
        pictureUrl: profile.pictureUrl
      });
      
      // Rate limiting
      await new Promise(resolve => setTimeout(resolve, 50));
    } catch (error) {
      console.error(`Failed to get profile for ${userId}:`, error.message);
    }
  }
  
  return profiles;
}
```

### ตรวจสอบ Group Summary

```
GET https://api.line.me/v2/bot/group/{groupId}/summary
```

```json
{
  "groupId": "Ca56f94637cc4347f90a6fb46f1c3543e",
  "groupName": "กลุ่มทำงาน",
  "pictureUrl": "https://profile.line-scdn.net/..."
}
```

```javascript
async function getGroupSummary(client, groupId) {
  const summary = await client.getGroupSummary(groupId);
  return {
    groupId: summary.groupId,
    groupName: summary.groupName,
    pictureUrl: summary.pictureUrl
  };
}
```

---

## 5. Getting Room Member List

### API Endpoint

```
GET https://api.line.me/v2/bot/room/{roomId}/members/ids
```

### ดึงสมาชิก Room

```javascript
async function getAllRoomMembers(client, roomId) {
  const allMemberIds = [];
  let start = undefined;
  
  do {
    const response = await client.getRoomMemberIds(roomId, start);
    allMemberIds.push(...response.memberIds);
    start = response.next;
    
    if (start) {
      await new Promise(resolve => setTimeout(resolve, 100));
    }
  } while (start);
  
  return allMemberIds;
}
```

---

## 6. Leave Group/Room

### API Endpoint

```
POST https://api.line.me/v2/bot/group/{groupId}/leave
POST https://api.line.me/v2/bot/room/{roomId}/leave
```

### ออกจากกลุ่ม

```javascript
async function leaveGroup(client, groupId) {
  try {
    await client.leaveGroup(groupId);
    console.log(`Bot left group: ${groupId}`);
    return true;
  } catch (error) {
    console.error('Error leaving group:', error);
    return false;
  }
}

async function leaveRoom(client, roomId) {
  try {
    await client.leaveRoom(roomId);
    console.log(`Bot left room: ${roomId}`);
    return true;
  } catch (error) {
    console.error('Error leaving room:', error);
    return false;
  }
}
```

### ออกกลุ่มอัตโนมัติเมื่อถูกใช้ในทางที่ผิด

```javascript
// ตัวอย่าง: ออกกลุ่มถ้าได้รับ spam command
const spamWords = ['ขาย', 'โปรโมท', 'spam'];

async function handleGroupMessage(event, client) {
  const text = event.message?.text?.toLowerCase() || '';
  
  // ตรวจสอบ spam
  if (spamWords.some(word => text.includes(word))) {
    const groupId = event.source.groupId;
    
    await client.replyMessage({
      replyToken: event.replyToken,
      messages: [{
        type: 'text',
        text: '⚠️ ตรวจพบเนื้อหาที่ไม่เหมาะสม Bot จะออกจากกลุ่ม'
      }]
    });
    
    // รอสักครู่แล้วออก
    setTimeout(() => leaveGroup(client, groupId), 2000);
  }
}
```

---

## 7. Group Chat Event Handling

### Event Handler ครบทุกประเภท

```javascript
'use strict';

const lineService = require('../services/lineService');

/**
 * จัดการ Events ทั้งหมดใน Group/Room
 */
async function handleGroupEvent(event, client) {
  const sourceType = event.source.type;
  const groupId = event.source.groupId;
  const roomId = event.source.roomId;
  const chatId = groupId || roomId;
  
  switch (event.type) {
    case 'join':
      return handleJoinEvent(event, client, sourceType, chatId);
    
    case 'leave':
      return handleLeaveEvent(event, sourceType, chatId);
    
    case 'message':
      return handleGroupMessage(event, client, sourceType, chatId);
    
    case 'memberJoined':
      return handleMemberJoined(event, client);
    
    case 'memberLeft':
      return handleMemberLeft(event, sourceType, chatId);
    
    case 'postback':
      return handleGroupPostback(event, client);
    
    default:
      console.log('Unhandled group event:', event.type);
  }
}

/**
 * Bot เข้าร่วมกลุ่ม/room
 */
async function handleJoinEvent(event, client, sourceType, chatId) {
  const { replyToken } = event;
  
  const welcomeText = sourceType === 'group'
    ? '👋 สวัสดีครับ! ผมคือ Bot ผู้ช่วย\n\nพิมพ์ "ช่วยเหลือ" เพื่อดูคำสั่งที่รองรับ'
    : '👋 สวัสดีครับ! ยินดีให้บริการใน Room นี้';
  
  // บันทึก chat ใหม่
  await saveChatToDb(sourceType, chatId);
  
  return client.replyMessage({
    replyToken,
    messages: [{ type: 'text', text: welcomeText }]
  });
}

/**
 * Bot ออกจากกลุ่ม
 */
async function handleLeaveEvent(event, sourceType, chatId) {
  // ลบข้อมูลกลุ่มออกจาก DB
  await removeChatFromDb(sourceType, chatId);
  console.log(`Bot left ${sourceType}: ${chatId}`);
}

/**
 * จัดการข้อความใน Group
 */
async function handleGroupMessage(event, client, sourceType, chatId) {
  const { replyToken, message, source } = event;
  const userId = source.userId;
  
  if (!message || message.type !== 'text') return;
  
  const text = message.text.trim();
  
  // Bot ตอบเฉพาะเมื่อถูก @mention หรือพิมพ์คำสั่งที่รู้จัก
  const shouldRespond = isBotMentioned(text) || isKnownCommand(text);
  
  if (!shouldRespond) return;
  
  // ลบ @mention ออก
  const cleanText = removeMentions(text);
  
  // Router
  if (cleanText === 'ช่วยเหลือ' || cleanText === 'help') {
    return sendGroupHelp(replyToken, client, sourceType);
  }
  
  if (cleanText === 'สมาชิก' || cleanText === 'members') {
    return sendMemberList(replyToken, client, sourceType, chatId);
  }
  
  if (cleanText === 'รายงาน' || cleanText === 'report') {
    return sendGroupReport(replyToken, client, userId, sourceType, chatId);
  }
  
  if (cleanText.startsWith('ประกาศ ')) {
    // เฉพาะ admin เท่านั้น
    return handleAnnouncement(replyToken, client, userId, cleanText.replace('ประกาศ ', ''), sourceType, chatId);
  }
}

/**
 * จัดการเมื่อสมาชิกใหม่เข้ากลุ่ม
 */
async function handleMemberJoined(event, client) {
  const { replyToken, joined, source } = event;
  const newMembers = joined.members;
  
  const names = [];
  
  for (const member of newMembers) {
    if (member.type === 'user') {
      try {
        const profile = await client.getGroupMemberProfile(
          source.groupId,
          member.userId
        );
        names.push(profile.displayName);
      } catch {
        names.push('สมาชิกใหม่');
      }
    }
  }
  
  const welcomeText = names.length === 1
    ? `👋 ยินดีต้อนรับ ${names[0]} เข้าสู่กลุ่ม!`
    : `👋 ยินดีต้อนรับ ${names.join(', ')} เข้าสู่กลุ่ม!`;
  
  return client.replyMessage({
    replyToken,
    messages: [{ type: 'text', text: welcomeText }]
  });
}

/**
 * จัดการเมื่อสมาชิกออกจากกลุ่ม
 */
async function handleMemberLeft(event, sourceType, chatId) {
  const leftMembers = event.left.members;
  console.log(`${leftMembers.length} member(s) left ${sourceType}: ${chatId}`);
}

/**
 * Helper Functions
 */
function isBotMentioned(text) {
  const botName = process.env.BOT_DISPLAY_NAME || 'Bot';
  return text.includes(`@${botName}`) || text.toLowerCase().includes('@bot');
}

function isKnownCommand(text) {
  const commands = ['ช่วยเหลือ', 'help', 'สมาชิก', 'members', 'รายงาน', 'report'];
  return commands.some(cmd => text.toLowerCase().trim() === cmd);
}

function removeMentions(text) {
  return text.replace(/@\w+/g, '').trim();
}

/**
 * ส่ง Help Message ในกลุ่ม
 */
async function sendGroupHelp(replyToken, client, sourceType) {
  const helpText = `📖 คำสั่งที่รองรับใน${sourceType === 'group' ? 'กลุ่ม' : 'Room'}:\n\n`
    + `• @Bot ช่วยเหลือ - แสดงคำสั่ง\n`
    + `• @Bot สมาชิก - ดูรายชื่อสมาชิก\n`
    + `• @Bot รายงาน - ดูสรุปกิจกรรม\n`
    + `• @Bot ประกาศ [ข้อความ] - ประกาศ (Admin only)`;
  
  return client.replyMessage({
    replyToken,
    messages: [{ type: 'text', text: helpText }]
  });
}

/**
 * ส่งรายชื่อสมาชิก
 */
async function sendMemberList(replyToken, client, sourceType, chatId) {
  try {
    let memberIds;
    
    if (sourceType === 'group') {
      memberIds = await getAllGroupMemberIds(client, chatId);
    } else {
      memberIds = await getAllRoomMemberIds(client, chatId);
    }
    
    const memberText = `👥 สมาชิกใน${sourceType === 'group' ? 'กลุ่ม' : 'Room'}: ${memberIds.length} คน`;
    
    return client.replyMessage({
      replyToken,
      messages: [{ type: 'text', text: memberText }]
    });
  } catch (error) {
    return client.replyMessage({
      replyToken,
      messages: [{ type: 'text', text: '❌ ไม่สามารถดึงรายชื่อสมาชิกได้' }]
    });
  }
}

async function getAllGroupMemberIds(client, groupId) {
  const allIds = [];
  let start;
  
  do {
    const res = await client.getGroupMemberIds(groupId, start);
    allIds.push(...res.memberIds);
    start = res.next;
  } while (start);
  
  return allIds;
}

async function getAllRoomMemberIds(client, roomId) {
  const allIds = [];
  let start;
  
  do {
    const res = await client.getRoomMemberIds(roomId, start);
    allIds.push(...res.memberIds);
    start = res.next;
  } while (start);
  
  return allIds;
}

// Placeholder functions
async function saveChatToDb(sourceType, chatId) {
  console.log(`Saving ${sourceType}: ${chatId}`);
}

async function removeChatFromDb(sourceType, chatId) {
  console.log(`Removing ${sourceType}: ${chatId}`);
}

async function sendGroupReport(replyToken, client, userId, sourceType, chatId) {
  return client.replyMessage({
    replyToken,
    messages: [{ type: 'text', text: '📊 รายงานกิจกรรมกลุ่ม:\n\nกำลังดึงข้อมูล...' }]
  });
}

async function handleAnnouncement(replyToken, client, userId, announcement, sourceType, chatId) {
  // TODO: ตรวจสอบว่าเป็น admin หรือเปล่า
  const announcementText = `📣 ประกาศ\n\n${announcement}`;
  
  return client.replyMessage({
    replyToken,
    messages: [{ type: 'text', text: announcementText }]
  });
}

async function handleGroupPostback(event, client) {
  return client.replyMessage({
    replyToken: event.replyToken,
    messages: [{ type: 'text', text: `รับทราบ: ${event.postback.data}` }]
  });
}

module.exports = {
  handleGroupEvent,
  leaveGroup: async (client, groupId) => client.leaveGroup(groupId),
  leaveRoom: async (client, roomId) => client.leaveRoom(roomId)
};
```

---

## 8. Use Cases สำหรับ Group Bots

### ประเภท Bot ที่เหมาะกับ Group

```
Group Bot Use Cases
├── Notification Bot
│   ├── ส่งข่าวสารให้ทีม
│   ├── แจ้งเตือน deadline
│   └── อัปเดต status โปรเจกต์
│
├── Administrative Bot
│   ├── ประกาศกฎกลุ่ม
│   ├── จัดการสมาชิก
│   └── Poll/Vote
│
├── Service Bot
│   ├── ตอบคำถาม FAQ
│   ├── จองห้องประชุม
│   └── แบ่งปันทรัพยากร
│
└── Entertainment Bot
    ├── เกม
    ├── Quiz
    └── Random picker
```

### ตัวอย่าง Use Case: Project Management Bot

```javascript
/**
 * Project Management Bot สำหรับกลุ่มทีม
 */
class ProjectBot {
  constructor(client, db) {
    this.client = client;
    this.db = db;
  }
  
  /**
   * สร้าง Task ใหม่
   * Command: @Bot task เพิ่ม [ชื่องาน]
   */
  async createTask(replyToken, groupId, userId, taskName) {
    const task = await this.db.tasks.insertOne({
      groupId,
      name: taskName,
      assignedTo: null,
      status: 'pending',
      createdBy: userId,
      createdAt: new Date()
    });
    
    return this.client.replyMessage({
      replyToken,
      messages: [{
        type: 'text',
        text: `✅ เพิ่มงานแล้ว: "${taskName}"\nID: ${task.insertedId}`
      }]
    });
  }
  
  /**
   * รับงาน
   * Command: @Bot รับงาน [ID]
   */
  async assignTask(replyToken, groupId, userId, taskId) {
    const result = await this.db.tasks.findOneAndUpdate(
      { _id: taskId, groupId, status: 'pending' },
      {
        $set: {
          assignedTo: userId,
          status: 'in_progress',
          startedAt: new Date()
        }
      }
    );
    
    if (!result.value) {
      return this.client.replyMessage({
        replyToken,
        messages: [{ type: 'text', text: '❌ ไม่พบงานนี้' }]
      });
    }
    
    return this.client.replyMessage({
      replyToken,
      messages: [{
        type: 'text',
        text: `✅ รับงานแล้ว: "${result.value.name}"`
      }]
    });
  }
  
  /**
   * ดูงานทั้งหมด
   * Command: @Bot งาน
   */
  async listTasks(replyToken, groupId) {
    const tasks = await this.db.tasks
      .find({ groupId, status: { $ne: 'done' } })
      .sort({ createdAt: -1 })
      .limit(10)
      .toArray();
    
    if (tasks.length === 0) {
      return this.client.replyMessage({
        replyToken,
        messages: [{ type: 'text', text: '📋 ไม่มีงานที่รอดำเนินการ' }]
      });
    }
    
    const statusEmoji = {
      pending: '⏳',
      in_progress: '🔄',
      done: '✅'
    };
    
    const taskList = tasks.map(t =>
      `${statusEmoji[t.status]} ${t.name}`
    ).join('\n');
    
    return this.client.replyMessage({
      replyToken,
      messages: [{
        type: 'text',
        text: `📋 งานทั้งหมด (${tasks.length} งาน):\n\n${taskList}`
      }]
    });
  }
}
```

### ตัวอย่าง Use Case: Office Announcement Bot

```javascript
/**
 * Announcement Bot สำหรับองค์กร
 */

// สร้างประกาศและส่งไปยังกลุ่มที่กำหนด
async function sendAnnouncement(announcement, targetGroups, lineClient) {
  const messages = [
    {
      type: 'text',
      text: `📣 ประกาศ\n━━━━━━━━━━━━━━━\n${announcement.title}\n\n${announcement.content}\n━━━━━━━━━━━━━━━\nโดย: ${announcement.author}\nวันที่: ${new Date().toLocaleDateString('th-TH')}`
    }
  ];
  
  const results = [];
  
  for (const groupId of targetGroups) {
    try {
      await lineClient.pushMessage({
        to: groupId,
        messages
      });
      results.push({ groupId, success: true });
    } catch (error) {
      results.push({ groupId, success: false, error: error.message });
    }
  }
  
  return results;
}

// ตัวอย่างการใช้งาน
const announcement = {
  title: 'การประชุมประจำเดือน',
  content: 'ขอเชิญทุกท่านเข้าร่วมประชุมประจำเดือน\nวันศุกร์ที่ 15 ม.ค. เวลา 14:00 น.\nห้องประชุม A ชั้น 3',
  author: 'ฝ่ายบุคคล'
};

const targetGroups = [
  'Ca56f94637cc4347f90a6fb46f1c3543e',  // ทีม Marketing
  'Cb7a8e4912ab3458fe9abc123def4567g'   // ทีม Development
];
```

---

## 9. Notification-only Bots

Notification Bot คือ Bot ที่ถูกเพิ่มเข้ากลุ่มเพื่อส่งข้อความแจ้งเตือนอย่างเดียว ไม่ตอบโต้กับผู้ใช้

### การออกแบบ Notification Bot

```javascript
/**
 * Notification-only Bot
 * - ไม่ตอบสนองต่อข้อความ (เว้นแต่คำสั่งพิเศษ)
 * - ส่งข้อความได้ผ่าน Admin API เท่านั้น
 */

// Webhook handler - ตอบสนองน้อยที่สุด
async function handleNotificationBotEvent(event, client) {
  // รับเฉพาะ join/leave events
  if (event.type === 'join') {
    const chatId = event.source.groupId || event.source.roomId;
    await saveChatId(event.source.type, chatId);
    
    return client.replyMessage({
      replyToken: event.replyToken,
      messages: [{
        type: 'text',
        text: '🔔 Bot แจ้งเตือนได้เพิ่มเข้ากลุ่มแล้ว\n\nBot นี้จะส่งข้อความแจ้งเตือนสำคัญโดยอัตโนมัติ'
      }]
    });
  }
  
  if (event.type === 'leave') {
    const chatId = event.source.groupId || event.source.roomId;
    await removeChatId(event.source.type, chatId);
    return;
  }
  
  // ไม่ตอบสนองต่อข้อความอื่น
  // ยกเว้น admin commands
  if (event.type === 'message' && event.message?.text) {
    const text = event.message.text.trim();
    
    if (text === '/stop' || text === '/หยุด') {
      // Admin command: ออกจากกลุ่ม
      const sourceType = event.source.type;
      const chatId = event.source.groupId || event.source.roomId;
      
      await client.replyMessage({
        replyToken: event.replyToken,
        messages: [{ type: 'text', text: '👋 Bot กำลังออกจากกลุ่ม...' }]
      });
      
      setTimeout(async () => {
        if (sourceType === 'group') {
          await client.leaveGroup(chatId);
        } else {
          await client.leaveRoom(chatId);
        }
      }, 1000);
    }
  }
}

// ตัวอย่าง: Monitoring Bot ส่งแจ้งเตือนเมื่อระบบมีปัญหา
class SystemMonitorBot {
  constructor(lineClient, groupIds) {
    this.lineClient = lineClient;
    this.groupIds = groupIds; // กลุ่ม DevOps
  }
  
  async sendAlert(severity, title, message, details = {}) {
    const severityEmoji = {
      critical: '🚨',
      high: '⚠️',
      medium: '📊',
      low: 'ℹ️'
    };
    
    const emoji = severityEmoji[severity] || '📢';
    
    const alertText = [
      `${emoji} [${severity.toUpperCase()}] ${title}`,
      `━━━━━━━━━━━━━━━`,
      message,
      details.host ? `เซิร์ฟเวอร์: ${details.host}` : null,
      details.timestamp ? `เวลา: ${details.timestamp}` : null,
      details.value ? `ค่า: ${details.value}` : null
    ].filter(Boolean).join('\n');
    
    for (const groupId of this.groupIds) {
      try {
        await this.lineClient.pushMessage({
          to: groupId,
          messages: [{ type: 'text', text: alertText }]
        });
      } catch (error) {
        console.error(`Failed to send alert to ${groupId}:`, error);
      }
    }
  }
  
  async sendDailyReport(report) {
    const reportText = [
      `📊 รายงานประจำวัน - ${new Date().toLocaleDateString('th-TH')}`,
      `━━━━━━━━━━━━━━━`,
      `✅ Uptime: ${report.uptime}%`,
      `📈 Requests: ${report.requests.toLocaleString()} ครั้ง`,
      `⚡ Avg Response: ${report.avgResponseMs}ms`,
      `❌ Errors: ${report.errors} ครั้ง`,
      `━━━━━━━━━━━━━━━`,
      report.notes ? `หมายเหตุ: ${report.notes}` : null
    ].filter(Boolean).join('\n');
    
    for (const groupId of this.groupIds) {
      await this.lineClient.pushMessage({
        to: groupId,
        messages: [{ type: 'text', text: reportText }]
      });
    }
  }
}
```

---

## 10. Administrative Bots

### Bot สำหรับจัดการกลุ่ม

```javascript
/**
 * Administrative Bot สำหรับจัดการกลุ่ม
 */
class AdminBot {
  constructor(lineClient, db, adminUserIds) {
    this.lineClient = lineClient;
    this.db = db;
    this.adminUserIds = adminUserIds; // รายชื่อ Admin ที่อนุญาต
  }
  
  isAdmin(userId) {
    return this.adminUserIds.includes(userId);
  }
  
  async handleMessage(event) {
    const { replyToken, source, message } = event;
    const userId = source.userId;
    const groupId = source.groupId;
    const text = message?.text?.trim() || '';
    
    // ตรวจสอบว่าเป็น admin command
    if (!text.startsWith('/admin ') && !text.startsWith('@Bot admin ')) {
      return;
    }
    
    // ตรวจสอบสิทธิ์
    if (!this.isAdmin(userId)) {
      return this.lineClient.replyMessage({
        replyToken,
        messages: [{ type: 'text', text: '❌ คุณไม่มีสิทธิ์ใช้คำสั่งนี้' }]
      });
    }
    
    // Parse command
    const command = text.replace(/^(\/admin |@Bot admin )/, '').trim();
    
    return this.executeAdminCommand(replyToken, groupId, userId, command);
  }
  
  async executeAdminCommand(replyToken, groupId, userId, command) {
    const parts = command.split(' ');
    const action = parts[0];
    const args = parts.slice(1);
    
    switch (action) {
      case 'stats':
        return this.sendGroupStats(replyToken, groupId);
      
      case 'kick':
        // ไม่สามารถ kick สมาชิกผ่าน API ได้ แต่บันทึก log ได้
        return this.handleKickRequest(replyToken, groupId, userId, args[0]);
      
      case 'announce':
        return this.sendGroupAnnouncement(replyToken, groupId, args.join(' '));
      
      case 'poll':
        return this.createPoll(replyToken, groupId, args);
      
      case 'leave':
        return this.handleLeaveRequest(replyToken, groupId);
      
      default:
        return this.lineClient.replyMessage({
          replyToken,
          messages: [{
            type: 'text',
            text: `❓ คำสั่งไม่รู้จัก: ${action}\n\nคำสั่งที่รองรับ: stats, announce, poll, leave`
          }]
        });
    }
  }
  
  async sendGroupStats(replyToken, groupId) {
    try {
      const summary = await this.lineClient.getGroupSummary(groupId);
      const memberIds = await this.getAllMemberIds(groupId);
      
      const statsText = [
        `📊 สถิติกลุ่ม`,
        `━━━━━━━━━━━━━━━`,
        `ชื่อกลุ่ม: ${summary.groupName}`,
        `สมาชิก: ${memberIds.length} คน`,
        `━━━━━━━━━━━━━━━`,
        `อัปเดต: ${new Date().toLocaleString('th-TH')}`
      ].join('\n');
      
      return this.lineClient.replyMessage({
        replyToken,
        messages: [{ type: 'text', text: statsText }]
      });
    } catch (error) {
      return this.lineClient.replyMessage({
        replyToken,
        messages: [{ type: 'text', text: '❌ ไม่สามารถดึงข้อมูลกลุ่มได้' }]
      });
    }
  }
  
  async sendGroupAnnouncement(replyToken, groupId, text) {
    if (!text) {
      return this.lineClient.replyMessage({
        replyToken,
        messages: [{ type: 'text', text: '❌ กรุณาระบุข้อความประกาศ' }]
      });
    }
    
    const announcementText = `📣 ประกาศสำคัญ\n━━━━━━━━━━━━━━━\n${text}`;
    
    return this.lineClient.replyMessage({
      replyToken,
      messages: [{ type: 'text', text: announcementText }]
    });
  }
  
  async createPoll(replyToken, groupId, args) {
    // args[0] = คำถาม, args[1..] = ตัวเลือก
    if (args.length < 3) {
      return this.lineClient.replyMessage({
        replyToken,
        messages: [{ 
          type: 'text', 
          text: '❌ ใช้: /admin poll [คำถาม] [ตัวเลือก1] [ตัวเลือก2] ...' 
        }]
      });
    }
    
    const question = args[0];
    const options = args.slice(1);
    
    // สร้าง Button Template สำหรับ Poll
    const pollTemplate = {
      type: 'template',
      altText: `Poll: ${question}`,
      template: {
        type: 'buttons',
        title: '📊 Poll',
        text: question,
        actions: options.slice(0, 4).map((opt, i) => ({
          type: 'postback',
          label: opt,
          data: `action=vote&question=${encodeURIComponent(question)}&option=${i}`,
          displayText: `ฉันเลือก: ${opt}`
        }))
      }
    };
    
    return this.lineClient.replyMessage({
      replyToken,
      messages: [pollTemplate]
    });
  }
  
  async handleLeaveRequest(replyToken, groupId) {
    const confirmTemplate = {
      type: 'template',
      altText: 'ยืนยันการออกกลุ่ม',
      template: {
        type: 'confirm',
        text: 'ต้องการให้ Bot ออกจากกลุ่มนี้หรือไม่?',
        actions: [
          {
            type: 'postback',
            label: 'ใช่ ออกเลย',
            data: `action=admin_leave&groupId=${groupId}`
          },
          {
            type: 'postback',
            label: 'ยกเลิก',
            data: 'action=cancel_leave'
          }
        ]
      }
    };
    
    return this.lineClient.replyMessage({
      replyToken,
      messages: [confirmTemplate]
    });
  }
  
  async handleKickRequest(replyToken, groupId, requesterId, targetUserId) {
    // ไม่สามารถ kick ผ่าน API แต่บันทึก log ไว้
    console.log(`Admin ${requesterId} requested kick of ${targetUserId} in ${groupId}`);
    
    return this.lineClient.replyMessage({
      replyToken,
      messages: [{
        type: 'text',
        text: '⚠️ หมายเหตุ: LINE API ไม่รองรับการ kick สมาชิกโดยตรง\nต้องทำผ่านแอป LINE เท่านั้น'
      }]
    });
  }
  
  async getAllMemberIds(groupId) {
    const allIds = [];
    let start;
    
    do {
      const res = await this.lineClient.getGroupMemberIds(groupId, start);
      allIds.push(...res.memberIds);
      start = res.next;
    } while (start);
    
    return allIds;
  }
}
```

---

## 11. Code Examples ใน Node.js

### app.js - Full Group/Room Bot

```javascript
'use strict';

require('dotenv').config();
const express = require('express');
const line = require('@line/bot-sdk');
const { handleGroupEvent } = require('./handlers/groupHandler');

const app = express();

const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};

const client = new line.messagingApi.MessagingApiClient({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

app.post('/webhook', line.middleware(config), async (req, res) => {
  try {
    await Promise.all(req.body.events.map(event => processEvent(event)));
    res.json({ status: 'ok' });
  } catch (err) {
    console.error(err);
    res.status(500).end();
  }
});

async function processEvent(event) {
  const sourceType = event.source.type;
  
  // Route ตาม source type
  if (sourceType === 'group' || sourceType === 'room') {
    return handleGroupEvent(event, client);
  }
  
  // 1:1 Chat
  if (sourceType === 'user') {
    return handle1on1Event(event, client);
  }
}

async function handle1on1Event(event, client) {
  if (event.type === 'follow') {
    return client.replyMessage({
      replyToken: event.replyToken,
      messages: [{ type: 'text', text: '🎉 ยินดีต้อนรับ!' }]
    });
  }
  
  if (event.type === 'message' && event.message.type === 'text') {
    return client.replyMessage({
      replyToken: event.replyToken,
      messages: [{ type: 'text', text: `ได้รับ: ${event.message.text}` }]
    });
  }
}

// Admin endpoints สำหรับส่งข้อความในกลุ่ม
app.post('/admin/group/push', express.json(), async (req, res) => {
  const { groupId, message } = req.body;
  
  try {
    await client.pushMessage({
      to: groupId,
      messages: [{ type: 'text', text: message }]
    });
    res.json({ success: true });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.get('/admin/group/:groupId/summary', async (req, res) => {
  try {
    const summary = await client.getGroupSummary(req.params.groupId);
    res.json(summary);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.get('/admin/group/:groupId/members', async (req, res) => {
  try {
    const { groupId } = req.params;
    const allIds = [];
    let start;
    
    do {
      const response = await client.getGroupMemberIds(groupId, start);
      allIds.push(...response.memberIds);
      start = response.next;
    } while (start);
    
    res.json({ groupId, memberCount: allIds.length, memberIds: allIds });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.delete('/admin/group/:groupId/leave', async (req, res) => {
  try {
    await client.leaveGroup(req.params.groupId);
    res.json({ success: true, message: 'Left group successfully' });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.listen(process.env.PORT || 3000, () => {
  console.log('Group Bot server running');
});
```

---

## 12. Code Examples ใน Python

### handlers/group_handler.py

```python
import logging
from typing import Optional
from linebot.v3.webhooks import (
    MessageEvent, JoinEvent, LeaveEvent,
    MemberJoinedEvent, MemberLeftEvent, PostbackEvent,
    TextMessageContent
)
from services.line_service import reply_message, push_message

logger = logging.getLogger(__name__)


def handle_group_event(event, api):
    """จัดการ Events ทั้งหมดใน Group/Room"""
    event_type = event.type
    source = event.source
    source_type = source.type  # 'group' หรือ 'room'
    chat_id = getattr(source, 'group_id', None) or getattr(source, 'room_id', None)
    
    handlers = {
        'join': handle_join,
        'leave': handle_leave,
        'message': handle_message,
        'memberJoined': handle_member_joined,
        'memberLeft': handle_member_left,
        'postback': handle_postback
    }
    
    handler = handlers.get(event_type)
    if handler:
        return handler(event, api, source_type, chat_id)
    else:
        logger.info(f"Unhandled group event: {event_type}")


def handle_join(event, api, source_type, chat_id):
    """Bot เข้าร่วมกลุ่ม/Room"""
    chat_name = 'กลุ่ม' if source_type == 'group' else 'Room'
    
    reply_message(event.reply_token, [
        f'👋 สวัสดีครับ! ยินดีที่ได้เข้าร่วม{chat_name}นี้\n\nพิมพ์ "@Bot ช่วยเหลือ" เพื่อดูคำสั่ง'
    ])


def handle_leave(event, api, source_type, chat_id):
    """Bot ออกจากกลุ่ม"""
    logger.info(f"Bot left {source_type}: {chat_id}")


def handle_message(event, api, source_type, chat_id):
    """จัดการข้อความใน Group/Room"""
    if not isinstance(event.message, TextMessageContent):
        return
    
    text = event.message.text.strip()
    user_id = event.source.user_id
    
    # ตรวจสอบว่าถูก mention
    if not (is_bot_mentioned(text) or is_known_command(text)):
        return
    
    clean_text = remove_mentions(text)
    
    # Command routing
    if clean_text in ['ช่วยเหลือ', 'help']:
        send_group_help(event.reply_token, source_type)
    
    elif clean_text in ['สมาชิก', 'members']:
        send_member_count(event.reply_token, api, source_type, chat_id)
    
    elif clean_text.startswith('ประกาศ '):
        announcement = clean_text.replace('ประกาศ ', '')
        send_announcement(event.reply_token, announcement)
    
    else:
        reply_message(event.reply_token, f'รับทราบ: "{clean_text}"')


def handle_member_joined(event, api, source_type, chat_id):
    """สมาชิกใหม่เข้ากลุ่ม"""
    members = event.joined.members
    
    names = []
    for member in members:
        if member.type == 'user':
            try:
                if source_type == 'group':
                    profile = api.get_group_member_profile(chat_id, member.user_id)
                else:
                    profile = api.get_room_member_profile(chat_id, member.user_id)
                names.append(profile.display_name)
            except Exception:
                names.append('สมาชิกใหม่')
    
    if names:
        welcome = f"👋 ยินดีต้อนรับ {', '.join(names)} เข้าสู่กลุ่ม!"
        reply_message(event.reply_token, welcome)


def handle_member_left(event, api, source_type, chat_id):
    """สมาชิกออกจากกลุ่ม"""
    left_count = len(event.left.members)
    logger.info(f"{left_count} member(s) left {source_type}: {chat_id}")


def handle_postback(event, api, source_type, chat_id):
    """จัดการ Postback ใน Group"""
    from urllib.parse import parse_qs
    
    data = event.postback.data
    params = {k: v[0] for k, v in parse_qs(data).items()}
    action = params.get('action', '')
    
    if action == 'vote':
        question = params.get('question', '')
        option = params.get('option', '')
        reply_message(event.reply_token, f'🗳️ บันทึกการโหวตแล้ว\nคำถาม: {question}\nตัวเลือก: {option}')
    
    elif action == 'admin_leave':
        group_id = params.get('groupId', '')
        reply_message(event.reply_token, '👋 Bot กำลังออกจากกลุ่ม...')
        import threading
        import time
        def leave_after_delay():
            time.sleep(1)
            try:
                api.leave_group(group_id)
            except Exception as e:
                logger.error(f"Error leaving group: {e}")
        threading.Thread(target=leave_after_delay).start()
    
    else:
        reply_message(event.reply_token, f'รับทราบ: {data}')


def send_group_help(reply_token: str, source_type: str):
    """ส่ง Help ในกลุ่ม"""
    chat_name = 'กลุ่ม' if source_type == 'group' else 'Room'
    help_text = f"""📖 คำสั่งสำหรับ{chat_name}:

• @Bot ช่วยเหลือ - แสดงคำสั่ง
• @Bot สมาชิก - จำนวนสมาชิก
• @Bot ประกาศ [ข้อความ] - ส่งประกาศ"""
    reply_message(reply_token, help_text)


def send_member_count(reply_token: str, api, source_type: str, chat_id: str):
    """ส่งจำนวนสมาชิก"""
    try:
        all_ids = get_all_member_ids(api, source_type, chat_id)
        reply_message(reply_token, f'👥 สมาชิกทั้งหมด: {len(all_ids)} คน')
    except Exception as e:
        reply_message(reply_token, f'❌ ไม่สามารถดึงข้อมูลสมาชิกได้')


def send_announcement(reply_token: str, text: str):
    """ส่งประกาศ"""
    announcement = f"📣 ประกาศ\n━━━━━━━━━━━━━━━\n{text}"
    reply_message(reply_token, announcement)


def get_all_member_ids(api, source_type: str, chat_id: str):
    """ดึงรายชื่อสมาชิกทั้งหมด"""
    all_ids = []
    start = None
    
    while True:
        if source_type == 'group':
            response = api.get_group_member_ids(chat_id, start=start)
        else:
            response = api.get_room_member_ids(chat_id, start=start)
        
        all_ids.extend(response.member_ids)
        start = response.next
        
        if not start:
            break
    
    return all_ids


def is_bot_mentioned(text: str) -> bool:
    """ตรวจสอบว่า Bot ถูก mention"""
    import os
    bot_name = os.environ.get('BOT_DISPLAY_NAME', 'Bot')
    return (f'@{bot_name}' in text or
            '@bot' in text.lower() or
            '@Bot' in text)


def is_known_command(text: str) -> bool:
    """ตรวจสอบว่าเป็นคำสั่งที่รู้จัก"""
    commands = ['ช่วยเหลือ', 'help', 'สมาชิก', 'members']
    return text.lower().strip() in commands


def remove_mentions(text: str) -> str:
    """ลบ @mentions"""
    import re
    return re.sub(r'@\w+', '', text).strip()
```

### app.py - Flask App สำหรับ Group Bot

```python
import os
import logging
from flask import Flask, request, abort, jsonify
from linebot.v3 import WebhookHandler
from linebot.v3.exceptions import InvalidSignatureError
from linebot.v3.messaging import Configuration, ApiClient, MessagingApi
from handlers.group_handler import handle_group_event
from dotenv import load_dotenv

load_dotenv()

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

app = Flask(__name__)

handler = WebhookHandler(os.environ.get('LINE_CHANNEL_SECRET'))

configuration = Configuration(
    access_token=os.environ.get('LINE_CHANNEL_ACCESS_TOKEN')
)


def get_api():
    return MessagingApi(ApiClient(configuration))


@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('X-Line-Signature', '')
    body = request.get_data(as_text=True)
    
    try:
        handler.handle(body, signature)
    except InvalidSignatureError:
        abort(400)
    
    return 'OK'


@handler.default()
def handle_event(event):
    """Handle all events"""
    source_type = event.source.type
    
    if source_type in ['group', 'room']:
        api = get_api()
        handle_group_event(event, api)


# Admin endpoints
@app.route('/admin/group/<group_id>/push', methods=['POST'])
def push_to_group(group_id):
    data = request.get_json()
    message = data.get('message')
    
    if not message:
        return jsonify({'error': 'message required'}), 400
    
    try:
        api = get_api()
        from linebot.v3.messaging import PushMessageRequest, TextMessage
        
        api.push_message(PushMessageRequest(
            to=group_id,
            messages=[TextMessage(type='text', text=message)]
        ))
        
        return jsonify({'success': True})
    except Exception as e:
        return jsonify({'error': str(e)}), 500


@app.route('/admin/group/<group_id>/summary')
def group_summary(group_id):
    try:
        api = get_api()
        summary = api.get_group_summary(group_id)
        return jsonify({
            'groupId': summary.group_id,
            'groupName': summary.group_name,
            'pictureUrl': summary.picture_url
        })
    except Exception as e:
        return jsonify({'error': str(e)}), 500


@app.route('/admin/group/<group_id>/leave', methods=['DELETE'])
def leave_group(group_id):
    try:
        api = get_api()
        api.leave_group(group_id)
        return jsonify({'success': True})
    except Exception as e:
        return jsonify({'error': str(e)}), 500


if __name__ == '__main__':
    port = int(os.environ.get('PORT', 3000))
    app.run(host='0.0.0.0', port=port)
```

---

## 13. Best Practices

### ข้อควรระวังสำหรับ Group Bot

1. **ตอบเฉพาะเมื่อจำเป็น** - อย่าตอบทุกข้อความ ใช้ @mention detection
2. **Rate Limiting** - กลุ่มใหญ่อาจมี message flood ต้องมี cooldown
3. **Error Handling** - อย่าให้ Bot crash ใน production
4. **Permission Check** - ตรวจสอบสิทธิ์สำหรับ admin commands
5. **Leave Logic** - มีคำสั่งให้ Bot ออกจากกลุ่มได้

### Anti-spam

```javascript
// Rate Limiter สำหรับกลุ่ม
const groupCooldowns = new Map();

function isRateLimited(groupId, userId, limitMs = 5000) {
  const key = `${groupId}:${userId}`;
  const lastTime = groupCooldowns.get(key);
  
  if (lastTime && Date.now() - lastTime < limitMs) {
    return true;
  }
  
  groupCooldowns.set(key, Date.now());
  return false;
}

// ใช้ใน handler
async function handleGroupMessage(event, client) {
  const { source } = event;
  const groupId = source.groupId;
  const userId = source.userId;
  
  // ตรวจ rate limit
  if (isRateLimited(groupId, userId)) {
    return; // ไม่ตอบสนอง
  }
  
  // ประมวลผลข้อความ...
}
```

### การจัดการกลุ่มหลายกลุ่ม

```javascript
// เก็บข้อมูลกลุ่มทั้งหมด
const groupDatabase = new Map();

function saveGroup(sourceType, chatId, metadata = {}) {
  groupDatabase.set(chatId, {
    type: sourceType,
    chatId,
    joinedAt: new Date(),
    ...metadata
  });
}

function removeGroup(chatId) {
  groupDatabase.delete(chatId);
}

function getAllGroups() {
  return Array.from(groupDatabase.values());
}

// Push ข้อความไปยังกลุ่มทั้งหมด
async function broadcastToAllGroups(client, messages) {
  const groups = getAllGroups();
  
  for (const group of groups) {
    try {
      await client.pushMessage({
        to: group.chatId,
        messages
      });
      await new Promise(resolve => setTimeout(resolve, 100));
    } catch (error) {
      if (error.statusCode === 404) {
        // Bot ถูกเพิ่มออกจากกลุ่มแล้ว
        removeGroup(group.chatId);
      }
      console.error(`Failed to push to ${group.chatId}:`, error);
    }
  }
}
```

---

## 14. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Welcome Bot
สร้าง Bot ที่:
- ส่งข้อความต้อนรับเมื่อเข้ากลุ่ม
- ต้อนรับสมาชิกใหม่พร้อมชื่อ
- มีคำสั่ง "สมาชิก" เพื่อดูจำนวนสมาชิก

### แบบฝึกหัดที่ 2: Poll Bot
สร้าง Bot ที่:
- สร้าง Poll ด้วย Template Message
- เก็บผลโหวต
- แสดงผลสรุปเมื่อพิมพ์ "@Bot ผลโหวต"

### แบบฝึกหัดที่ 3: Team Notification Bot
สร้างระบบที่:
- Bot อยู่ในกลุ่ม DevOps
- มี API endpoint สำหรับส่ง alert เข้ากลุ่ม
- มี format ที่ชัดเจนสำหรับ different severities

### เฉลยแบบฝึกหัดที่ 1 (Node.js)

```javascript
'use strict';

require('dotenv').config();
const express = require('express');
const line = require('@line/bot-sdk');

const app = express();
const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};

const client = new line.messagingApi.MessagingApiClient({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

app.post('/webhook', line.middleware(config), async (req, res) => {
  await Promise.all(req.body.events.map(handleEvent));
  res.json({ ok: true });
});

async function handleEvent(event) {
  if (event.type === 'join') {
    return client.replyMessage({
      replyToken: event.replyToken,
      messages: [{ type: 'text', text: '👋 สวัสดีครับ! พิมพ์ "สมาชิก" เพื่อดูจำนวน' }]
    });
  }
  
  if (event.type === 'memberJoined') {
    const names = [];
    for (const m of event.joined.members) {
      if (m.type === 'user') {
        try {
          const p = await client.getGroupMemberProfile(event.source.groupId, m.userId);
          names.push(p.displayName);
        } catch { names.push('ผู้ใช้ใหม่'); }
      }
    }
    return client.replyMessage({
      replyToken: event.replyToken,
      messages: [{ type: 'text', text: `🎉 ยินดีต้อนรับ ${names.join(', ')}!` }]
    });
  }
  
  if (event.type === 'message' && event.message?.text === 'สมาชิก') {
    const groupId = event.source.groupId;
    let count = 0;
    let start;
    do {
      const res = await client.getGroupMemberIds(groupId, start);
      count += res.memberIds.length;
      start = res.next;
    } while (start);
    
    return client.replyMessage({
      replyToken: event.replyToken,
      messages: [{ type: 'text', text: `👥 สมาชิกในกลุ่ม: ${count} คน` }]
    });
  }
}

app.listen(3000, () => console.log('Welcome Bot running'));
```

---

**สรุปบทเรียน:**

ในบทนี้เราได้เรียนรู้การทำงานของ Bot ใน Group และ Room:

1. **ความแตกต่าง** ของ Group, Room และ Multi-person Chat
2. **Events ใน Group** - join, leave, message, memberJoined, memberLeft
3. **Member Management** - ดึงรายชื่อสมาชิก, ดู profile
4. **Leave API** - Bot ออกจากกลุ่ม/Room
5. **Use Cases** - Notification Bot, Administrative Bot
6. **Best Practices** - Rate limiting, @mention detection, Permission checks

การออกแบบ Group Bot ที่ดีต้องคำนึงถึง UX ของผู้ใช้ในกลุ่ม อย่าตอบทุกข้อความ และมีระบบป้องกัน spam
