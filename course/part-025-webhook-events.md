# Part 25: Webhook Events ทุกประเภท

## สารบัญ

1. [Overview ของ Webhook Events](#overview)
2. [Message Events](#message-events)
   - [Text Message](#text-message)
   - [Image Message](#image-message)
   - [Video Message](#video-message)
   - [Audio Message](#audio-message)
   - [File Message](#file-message)
   - [Location Message](#location-message)
   - [Sticker Message](#sticker-message)
3. [Follow / Unfollow Events](#follow-unfollow-events)
4. [Postback Events](#postback-events)
5. [Join / Leave Events](#join-leave-events)
6. [MemberJoined / MemberLeft Events](#member-events)
7. [Beacon Events](#beacon-events)
8. [AccountLink Events](#accountlink-events)
9. [Things Events](#things-events)
10. [โครงสร้าง Event JSON](#event-json-structure)
11. [การจัดการ Error](#error-handling)
12. [Webhook Verification](#webhook-verification)
13. [ตัวอย่างโค้ด Node.js และ Python](#code-examples)
14. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Overview ของ Webhook Events

### ประเภท Events ทั้งหมด

| Event Type | ความหมาย | Source |
|-----------|---------|--------|
| `message` | ผู้ใช้ส่งข้อความ | User, Group, Room |
| `follow` | ผู้ใช้กด Follow | User |
| `unfollow` | ผู้ใช้กด Block/Unfollow | User |
| `postback` | ผู้ใช้กด Postback Action | User, Group, Room |
| `join` | Bot ถูกเชิญเข้า Group/Room | Group, Room |
| `leave` | Bot ถูกลบออกจาก Group/Room | Group, Room |
| `memberJoined` | สมาชิกใหม่เข้า Group/Room | Group, Room |
| `memberLeft` | สมาชิกออกจาก Group/Room | Group, Room |
| `beacon` | ผู้ใช้อยู่ใกล้ LINE Beacon | User |
| `accountLink` | การเชื่อมบัญชี | User |
| `things` | LINE Things device event | User |
| `videoViewingComplete` | ผู้ใช้ดูวิดีโอจนจบ | User, Group |
| `unsend` | ผู้ใช้ unsend ข้อความ | User, Group, Room |

### โครงสร้าง Webhook Request

```json
{
  "destination": "xxxxxxxxxx",
  "events": [
    {
      "type": "message",
      "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
      "deliveryContext": {
        "isRedelivery": false
      },
      "timestamp": 1625665242211,
      "source": {
        "type": "user",
        "userId": "U4af4980629..."
      },
      "replyToken": "0f3779fba3b349968c5d07db31eab56f",
      "mode": "active",
      "message": {
        "type": "text",
        "id": "14353798921116",
        "quoteToken": "q3Plxr4AgKd...",
        "text": "Hello!"
      }
    }
  ]
}
```

### Source Types

```json
// User source
{
  "type": "user",
  "userId": "U4af4980629..."
}

// Group source
{
  "type": "group",
  "groupId": "Ca56f94637cc4347f90a6...",
  "userId": "U4af4980629..."
}

// Room source
{
  "type": "room",
  "roomId": "Ra8dbf4673c4c812cd491...",
  "userId": "U4af4980629..."
}
```

---

## 2. Message Events

Message Events เกิดขึ้นเมื่อผู้ใช้ส่งข้อความมายัง Bot

### 2.1 Text Message

**JSON:**
```json
{
  "type": "message",
  "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
  "deliveryContext": {
    "isRedelivery": false
  },
  "timestamp": 1625665242211,
  "source": {
    "type": "user",
    "userId": "U4af4980629..."
  },
  "replyToken": "0f3779fba3b349968c5d07db31eab56f",
  "mode": "active",
  "message": {
    "type": "text",
    "id": "14353798921116",
    "quoteToken": "q3Plxr4AgKd...",
    "text": "สวัสดีครับ",
    "emojis": [
      {
        "index": 0,
        "length": 6,
        "productId": "5ac1bfd5040ab15980c9b435",
        "emojiId": "001"
      }
    ],
    "mention": {
      "mentionees": [
        {
          "index": 0,
          "length": 7,
          "userId": "U49585cd0d5...",
          "isSelf": false,
          "type": "user"
        }
      ]
    },
    "quotedMessageId": "14353798921112"
  }
}
```

**Node.js Handler:**
```javascript
// handlers/textMessage.js
async function handleTextMessage(event) {
  const { replyToken, message, source } = event;
  const text = message.text;
  const userId = source.userId;
  
  console.log(`[TEXT] User ${userId}: "${text}"`);
  
  // ตรวจสอบ mention
  if (message.mention) {
    const mentionees = message.mention.mentionees;
    console.log('Mentioned users:', mentionees.map(m => m.userId));
  }
  
  // ตรวจสอบ quote
  if (message.quotedMessageId) {
    console.log('Quoted message ID:', message.quotedMessageId);
  }
  
  // ประมวลผล
  const response = processTextCommand(text);
  
  await client.replyMessage({
    replyToken,
    messages: [{
      type: 'text',
      text: response
    }]
  });
}

function processTextCommand(text) {
  const lower = text.toLowerCase().trim();
  
  const commands = {
    'สวัสดี': 'สวัสดีครับ! 😊',
    'hello': 'Hello! 👋',
    'help': getHelpText(),
    'time': getCurrentTime(),
    'date': getCurrentDate(),
  };
  
  if (commands[lower]) return commands[lower];
  if (lower.startsWith('/echo ')) return text.substring(6);
  
  return `คุณพิมพ์ว่า: "${text}"`;
}

function getHelpText() {
  return [
    '📋 คำสั่งที่รองรับ:',
    'สวัสดี - ทักทาย',
    'help - ช่วยเหลือ',
    'time - เวลาปัจจุบัน',
    'date - วันที่ปัจจุบัน',
    '/echo <ข้อความ> - Echo',
  ].join('\n');
}

function getCurrentTime() {
  return `🕐 ${new Date().toLocaleTimeString('th-TH', {timeZone: 'Asia/Bangkok'})}`;
}

function getCurrentDate() {
  return `📅 ${new Date().toLocaleDateString('th-TH', {
    timeZone: 'Asia/Bangkok',
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    weekday: 'long'
  })}`;
}

module.exports = { handleTextMessage };
```

**Python Handler:**
```python
# handlers/text_message.py
from datetime import datetime
import pytz
from linebot.v3.messaging import TextMessage
from app.services import line_service


def handle_text_message(event) -> None:
    """จัดการ Text Message"""
    text = event.message.text
    user_id = event.source.user_id
    reply_token = event.reply_token
    
    print(f"[TEXT] User {user_id}: {text!r}")
    
    # ตรวจสอบ mention
    if hasattr(event.message, 'mention') and event.message.mention:
        mentionees = event.message.mention.mentionees
        print(f"Mentioned: {[m.user_id for m in mentionees]}")
    
    response = process_command(text)
    
    line_service.reply_message(
        reply_token,
        TextMessage(text=response)
    )


def process_command(text: str) -> str:
    """ประมวลผลคำสั่ง"""
    lower = text.lower().strip()
    
    commands = {
        'สวัสดี': 'สวัสดีครับ! 😊',
        'hello': 'Hello! 👋',
        'help': get_help_text(),
        'time': get_current_time(),
        'date': get_current_date(),
    }
    
    if lower in commands:
        return commands[lower]
    
    if lower.startswith('/echo '):
        return text[6:]
    
    return f'คุณพิมพ์ว่า: "{text}"'


def get_help_text() -> str:
    return '\n'.join([
        '📋 คำสั่งที่รองรับ:',
        'สวัสดี - ทักทาย',
        'help - ช่วยเหลือ',
        'time - เวลาปัจจุบัน',
        'date - วันที่ปัจจุบัน',
        '/echo <ข้อความ> - Echo',
    ])


def get_current_time() -> str:
    tz = pytz.timezone('Asia/Bangkok')
    now = datetime.now(tz)
    return f'🕐 {now.strftime("%H:%M:%S")}'


def get_current_date() -> str:
    tz = pytz.timezone('Asia/Bangkok')
    now = datetime.now(tz)
    return f'📅 {now.strftime("%A, %d %B %Y")}'
```

---

### 2.2 Image Message

**JSON:**
```json
{
  "type": "message",
  "timestamp": 1625665242211,
  "source": {
    "type": "user",
    "userId": "U4af4980629..."
  },
  "replyToken": "0f3779fba3b349968c5d07db31eab56f",
  "message": {
    "type": "image",
    "id": "14353798921116",
    "quoteToken": "q3Plxr4AgKd...",
    "contentProvider": {
      "type": "line"
    },
    "imageSet": {
      "id": "E005D41A7288F41B65593ED38FF6E9834B046AB966BC6369D3B5145F4B39898C",
      "index": 1,
      "total": 2
    }
  }
}
```

**Node.js Handler:**
```javascript
// handlers/imageMessage.js
const fs = require('fs');
const path = require('path');

async function handleImageMessage(event) {
  const { replyToken, message } = event;
  
  console.log(`[IMAGE] Message ID: ${message.id}`);
  console.log(`[IMAGE] Content Provider: ${message.contentProvider?.type}`);
  
  // ดาวน์โหลดรูปภาพ (ถ้าต้องการ)
  if (message.contentProvider?.type === 'line') {
    try {
      const content = await client.getMessageContent(message.id);
      
      // บันทึกไฟล์
      const fileName = `uploads/image_${message.id}.jpg`;
      const writeStream = fs.createWriteStream(fileName);
      
      await new Promise((resolve, reject) => {
        content.pipe(writeStream);
        content.on('end', resolve);
        content.on('error', reject);
      });
      
      console.log(`Image saved: ${fileName}`);
    } catch (error) {
      console.error('Failed to download image:', error);
    }
  }
  
  // ตรวจสอบว่าเป็นส่วนหนึ่งของ Image Set
  if (message.imageSet) {
    const { id, index, total } = message.imageSet;
    console.log(`Image ${index}/${total} in set ${id}`);
  }
  
  await client.replyMessage({
    replyToken,
    messages: [{
      type: 'text',
      text: '📷 ขอบคุณสำหรับรูปภาพ!'
    }]
  });
}

module.exports = { handleImageMessage };
```

**Python Handler:**
```python
# handlers/image_message.py
import os
from linebot.v3.messaging import TextMessage
from app.services import line_service


def handle_image_message(event) -> None:
    """จัดการ Image Message"""
    message = event.message
    reply_token = event.reply_token
    
    print(f"[IMAGE] ID: {message.id}")
    
    # ตรวจสอบ image set
    if hasattr(message, 'image_set') and message.image_set:
        print(f"Image {message.image_set.index}/{message.image_set.total}")
    
    # ดาวน์โหลดรูปภาพ
    if message.content_provider.type == 'line':
        try:
            content = line_service.line_bot_api.get_message_content(message.id)
            
            os.makedirs('uploads', exist_ok=True)
            file_path = f'uploads/image_{message.id}.jpg'
            
            with open(file_path, 'wb') as f:
                for chunk in content:
                    f.write(chunk)
            
            print(f"Image saved: {file_path}")
        except Exception as e:
            print(f"Failed to download: {e}")
    
    line_service.reply_message(
        reply_token,
        TextMessage(text='📷 ขอบคุณสำหรับรูปภาพ!')
    )
```

---

### 2.3 Video Message

**JSON:**
```json
{
  "type": "message",
  "message": {
    "type": "video",
    "id": "14353798921116",
    "quoteToken": "q3Plxr4AgKd...",
    "duration": 60000,
    "contentProvider": {
      "type": "external",
      "originalContentUrl": "https://example.com/video.mp4",
      "previewImageUrl": "https://example.com/thumbnail.jpg"
    }
  }
}
```

**Node.js Handler:**
```javascript
async function handleVideoMessage(event) {
  const { replyToken, message } = event;
  
  console.log(`[VIDEO] Duration: ${message.duration}ms`);
  
  if (message.contentProvider?.type === 'external') {
    console.log(`External video: ${message.contentProvider.originalContentUrl}`);
  }
  
  await client.replyMessage({
    replyToken,
    messages: [{
      type: 'text',
      text: `🎥 ขอบคุณสำหรับวิดีโอ!\n⏱️ ความยาว: ${Math.round(message.duration / 1000)} วินาที`
    }]
  });
}
```

---

### 2.4 Audio Message

**JSON:**
```json
{
  "type": "message",
  "message": {
    "type": "audio",
    "id": "14353798921116",
    "quoteToken": "q3Plxr4AgKd...",
    "duration": 10000,
    "contentProvider": {
      "type": "line"
    }
  }
}
```

**Node.js Handler:**
```javascript
async function handleAudioMessage(event) {
  const { replyToken, message } = event;
  
  const durationSec = Math.round((message.duration || 0) / 1000);
  
  await client.replyMessage({
    replyToken,
    messages: [{
      type: 'text',
      text: `🎵 ได้รับไฟล์เสียง (${durationSec} วินาที)`
    }]
  });
}
```

---

### 2.5 File Message

**JSON:**
```json
{
  "type": "message",
  "message": {
    "type": "file",
    "id": "14353798921116",
    "quoteToken": "q3Plxr4AgKd...",
    "fileName": "document.pdf",
    "fileSize": 2048
  }
}
```

**Node.js Handler:**
```javascript
async function handleFileMessage(event) {
  const { replyToken, message } = event;
  const { fileName, fileSize } = message;
  
  const fileSizeKB = Math.round(fileSize / 1024);
  
  console.log(`[FILE] Name: ${fileName}, Size: ${fileSizeKB}KB`);
  
  // ดาวน์โหลดไฟล์ (ถ้าต้องการ)
  // const content = await client.getMessageContent(message.id);
  // const writeStream = fs.createWriteStream(`uploads/${fileName}`);
  // content.pipe(writeStream);
  
  await client.replyMessage({
    replyToken,
    messages: [{
      type: 'text',
      text: `📎 ได้รับไฟล์:\n📄 ${fileName}\n💾 ขนาด: ${fileSizeKB} KB`
    }]
  });
}
```

**Python Handler:**
```python
def handle_file_message(event) -> None:
    message = event.message
    reply_token = event.reply_token
    
    file_size_kb = round((message.file_size or 0) / 1024, 1)
    
    line_service.reply_message(
        reply_token,
        TextMessage(
            text=f'📎 ได้รับไฟล์:\n📄 {message.file_name}\n💾 {file_size_kb} KB'
        )
    )
```

---

### 2.6 Location Message

**JSON:**
```json
{
  "type": "message",
  "message": {
    "type": "location",
    "id": "14353798921116",
    "quoteToken": "q3Plxr4AgKd...",
    "title": "บริษัท ABC",
    "address": "123 ถนนสุขุมวิท แขวงคลองเตย เขตคลองเตย กรุงเทพมหานคร 10110",
    "latitude": 13.7563,
    "longitude": 100.5018
  }
}
```

**Node.js Handler:**
```javascript
async function handleLocationMessage(event) {
  const { replyToken, message } = event;
  const { title, address, latitude, longitude } = message;
  
  console.log(`[LOCATION] ${title}: ${latitude}, ${longitude}`);
  
  // ส่ง Location กลับ
  await client.replyMessage({
    replyToken,
    messages: [
      {
        type: 'text',
        text: `📍 ได้รับตำแหน่ง:\n🏢 ${title || 'ไม่มีชื่อ'}\n📮 ${address || ''}\n\n🗺️ พิกัด: ${latitude.toFixed(4)}, ${longitude.toFixed(4)}`
      },
      {
        type: 'location',
        title: title || 'ตำแหน่งที่ส่งมา',
        address: address || `${latitude}, ${longitude}`,
        latitude,
        longitude
      }
    ]
  });
}
```

**Python Handler:**
```python
from linebot.v3.messaging import TextMessage, LocationMessage

def handle_location_message(event) -> None:
    message = event.message
    reply_token = event.reply_token
    
    title = message.title or 'ตำแหน่งที่ส่งมา'
    address = message.address or f'{message.latitude}, {message.longitude}'
    
    line_service.reply_message(
        reply_token,
        [
            TextMessage(
                text=f'📍 ได้รับตำแหน่ง:\n🏢 {title}\n📮 {address}'
            ),
            LocationMessage(
                title=title,
                address=address,
                latitude=message.latitude,
                longitude=message.longitude
            )
        ]
    )
```

---

### 2.7 Sticker Message

**JSON:**
```json
{
  "type": "message",
  "message": {
    "type": "sticker",
    "id": "14353798921116",
    "quoteToken": "q3Plxr4AgKd...",
    "packageId": "446",
    "stickerId": "1988",
    "stickerResourceType": "STATIC",
    "keywords": ["happy", "smile"]
  }
}
```

**Sticker Resource Types:**
- `STATIC` - ภาพนิ่ง
- `ANIMATION` - Animated
- `SOUND` - มีเสียง
- `ANIMATION_SOUND` - Animated + เสียง
- `POPUP` - Pop-up
- `POPUP_SOUND` - Pop-up + เสียง
- `NAME_TEXT` - มีชื่อ
- `PER_STICKER_TEXT` - มีข้อความในแต่ละ Sticker

**Node.js Handler:**
```javascript
async function handleStickerMessage(event) {
  const { replyToken, message } = event;
  const { packageId, stickerId, stickerResourceType, keywords } = message;
  
  console.log(`[STICKER] Package: ${packageId}, Sticker: ${stickerId}, Type: ${stickerResourceType}`);
  
  if (keywords) {
    console.log(`[STICKER] Keywords: ${keywords.join(', ')}`);
  }
  
  // ส่ง Sticker กลับ (random จากรายการ)
  const responses = [
    { packageId: '446', stickerId: '1988' },
    { packageId: '446', stickerId: '2012' },
    { packageId: '789', stickerId: '10855' },
  ];
  
  const random = responses[Math.floor(Math.random() * responses.length)];
  
  await client.replyMessage({
    replyToken,
    messages: [{
      type: 'sticker',
      packageId: random.packageId,
      stickerId: random.stickerId
    }]
  });
}
```

---

## 3. Follow / Unfollow Events

### Follow Event

เกิดขึ้นเมื่อ:
- ผู้ใช้กด Follow
- ผู้ใช้ Unblock Bot

**JSON:**
```json
{
  "type": "follow",
  "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
  "deliveryContext": {
    "isRedelivery": false
  },
  "timestamp": 1625665242211,
  "source": {
    "type": "user",
    "userId": "U4af4980629..."
  },
  "replyToken": "0f3779fba3b349968c5d07db31eab56f",
  "mode": "active"
}
```

**Node.js Handler:**
```javascript
async function handleFollow(event) {
  const { replyToken, source } = event;
  const userId = source.userId;
  
  console.log(`[FOLLOW] New follower: ${userId}`);
  
  // บันทึกใน Database
  // await db.users.upsert({ userId, followedAt: new Date(), isActive: true });
  
  // ดึง Profile
  let displayName = 'คุณ';
  let pictureUrl = null;
  
  try {
    const profile = await client.getProfile(userId);
    displayName = profile.displayName;
    pictureUrl = profile.pictureUrl;
    
    // บันทึก Profile
    // await db.users.update(userId, { displayName, pictureUrl });
  } catch (error) {
    console.error('Cannot get profile:', error);
  }
  
  // Flex Message ต้อนรับ
  const welcomeMessage = createWelcomeFlexMessage(displayName);
  
  await client.replyMessage({
    replyToken,
    messages: [
      {
        type: 'text',
        text: `ยินดีต้อนรับ ${displayName}! 🎉`
      },
      welcomeMessage
    ]
  });
}

function createWelcomeFlexMessage(displayName) {
  return {
    type: 'flex',
    altText: 'ยินดีต้อนรับ',
    contents: {
      type: 'bubble',
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: '🎉 ยินดีต้อนรับ!',
            weight: 'bold',
            size: 'xl'
          },
          {
            type: 'text',
            text: `สวัสดี ${displayName}`,
            size: 'lg',
            margin: 'md'
          },
          {
            type: 'separator',
            margin: 'xl'
          },
          {
            type: 'text',
            text: 'สิ่งที่ Bot ทำได้:',
            weight: 'bold',
            margin: 'xl'
          },
          {
            type: 'text',
            text: '• ตอบคำถามอัตโนมัติ\n• แสดงสินค้า\n• รับออเดอร์',
            wrap: true,
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
            style: 'primary',
            action: {
              type: 'message',
              label: 'เริ่มต้นใช้งาน',
              text: 'help'
            }
          }
        ]
      }
    }
  };
}
```

### Unfollow Event

**JSON:**
```json
{
  "type": "unfollow",
  "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
  "deliveryContext": {
    "isRedelivery": false
  },
  "timestamp": 1625665242211,
  "source": {
    "type": "user",
    "userId": "U4af4980629..."
  },
  "mode": "active"
}
```

> **หมายเหตุ:** Unfollow Event ไม่มี `replyToken` เพราะไม่สามารถ Reply ได้

**Node.js Handler:**
```javascript
async function handleUnfollow(event) {
  const userId = event.source.userId;
  
  console.log(`[UNFOLLOW] User ${userId} unfollowed`);
  
  // อัปเดตสถานะใน Database
  // await db.users.update(userId, { isActive: false, unfollowedAt: new Date() });
  
  // ไม่สามารถส่งข้อความได้ เพราะ Block แล้ว
}
```

**Python Handler:**
```python
def handle_follow(event) -> None:
    """จัดการ Follow Event"""
    user_id = event.source.user_id
    reply_token = event.reply_token
    
    print(f"[FOLLOW] New follower: {user_id}")
    
    # ดึง Profile
    display_name = 'คุณ'
    try:
        profile = line_service.get_user_profile(user_id)
        display_name = profile['displayName']
    except Exception as e:
        print(f"Cannot get profile: {e}")
    
    # ส่งข้อความต้อนรับ
    line_service.reply_message(
        reply_token,
        [
            TextMessage(text=f'ยินดีต้อนรับ {display_name}! 🎉'),
            TextMessage(text='พิมพ์ "help" เพื่อดูคำสั่งทั้งหมด'),
        ]
    )


def handle_unfollow(event) -> None:
    """จัดการ Unfollow Event"""
    user_id = event.source.user_id
    print(f"[UNFOLLOW] {user_id}")
    # ไม่สามารถ Reply ได้
```

---

## 4. Postback Events

Postback เกิดขึ้นเมื่อผู้ใช้กด Button ที่มี `postback` Action

**JSON:**
```json
{
  "type": "postback",
  "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
  "deliveryContext": {
    "isRedelivery": false
  },
  "timestamp": 1625665242211,
  "source": {
    "type": "user",
    "userId": "U4af4980629..."
  },
  "replyToken": "0f3779fba3b349968c5d07db31eab56f",
  "postback": {
    "data": "action=buy&productId=12345&qty=1",
    "params": {
      "date": "2023-12-25",
      "time": "10:00",
      "datetime": "2023-12-25T10:00"
    }
  },
  "mode": "active"
}
```

**Postback Data Formats:**

```javascript
// Simple key-value
"data": "action=buy&id=123"

// JSON encoded
"data": "{\"action\":\"confirm\",\"orderId\":\"ORD001\"}"

// Plain text
"data": "cancel_order"
```

**Node.js Handler:**
```javascript
async function handlePostback(event) {
  const { replyToken, postback, source } = event;
  const userId = source.userId;
  
  console.log(`[POSTBACK] Data: ${postback.data}`);
  
  // Parse postback data
  let data;
  try {
    // ลอง parse เป็น JSON ก่อน
    data = JSON.parse(postback.data);
  } catch {
    // ถ้าไม่ใช่ JSON ให้ parse เป็น query string
    const params = new URLSearchParams(postback.data);
    data = Object.fromEntries(params.entries());
  }
  
  console.log(`[POSTBACK] Parsed data:`, data);
  
  // ตรวจสอบ date/time params
  if (postback.params?.date) {
    console.log(`[POSTBACK] Date selected: ${postback.params.date}`);
  }
  if (postback.params?.time) {
    console.log(`[POSTBACK] Time selected: ${postback.params.time}`);
  }
  
  // Dispatch ตาม action
  const action = data.action || data;
  
  const handlers = {
    buy: () => handleBuyAction(data, userId),
    cancel: () => handleCancelAction(data, userId),
    confirm: () => handleConfirmAction(data, userId),
    menu: () => handleMenuAction(data, userId),
  };
  
  const handler = handlers[action];
  let responseText;
  
  if (handler) {
    responseText = await handler();
  } else {
    responseText = `ได้รับ Postback: ${postback.data}`;
  }
  
  await client.replyMessage({
    replyToken,
    messages: [{
      type: 'text',
      text: responseText
    }]
  });
}

async function handleBuyAction(data, userId) {
  const { productId, qty = 1 } = data;
  // บันทึกออเดอร์
  // await orderService.create({ userId, productId, qty });
  return `✅ สั่งซื้อสำเร็จ!\n📦 สินค้า ID: ${productId}\n🔢 จำนวน: ${qty}`;
}

async function handleCancelAction(data, userId) {
  const { orderId } = data;
  // ยกเลิกออเดอร์
  return `❌ ยกเลิกออเดอร์ ${orderId} แล้ว`;
}

async function handleConfirmAction(data, userId) {
  return `✅ ยืนยันแล้ว!`;
}

async function handleMenuAction(data, userId) {
  const { menuItem } = data;
  return `📋 เลือก: ${menuItem}`;
}

module.exports = { handlePostback };
```

**Python Handler:**
```python
import json
from urllib.parse import parse_qs, urlencode
from linebot.v3.messaging import TextMessage
from app.services import line_service


def handle_postback(event) -> None:
    """จัดการ Postback Event"""
    postback = event.postback
    reply_token = event.reply_token
    user_id = event.source.user_id
    
    data_str = postback.data
    print(f"[POSTBACK] Data: {data_str}")
    
    # Parse data
    data = parse_postback_data(data_str)
    
    # ตรวจสอบ date/time params
    if hasattr(postback, 'params') and postback.params:
        if hasattr(postback.params, 'date') and postback.params.date:
            print(f"[POSTBACK] Date: {postback.params.date}")
        if hasattr(postback.params, 'time') and postback.params.time:
            print(f"[POSTBACK] Time: {postback.params.time}")
    
    # Dispatch
    action = data.get('action', data_str)
    
    response = dispatch_action(action, data, user_id)
    
    line_service.reply_message(
        reply_token,
        TextMessage(text=response)
    )


def parse_postback_data(data_str: str) -> dict:
    """Parse postback data string"""
    # ลอง parse JSON
    try:
        return json.loads(data_str)
    except (json.JSONDecodeError, ValueError):
        pass
    
    # ลอง parse query string
    try:
        parsed = parse_qs(data_str)
        return {k: v[0] if len(v) == 1 else v for k, v in parsed.items()}
    except Exception:
        pass
    
    return {'raw': data_str}


def dispatch_action(action: str, data: dict, user_id: str) -> str:
    """Dispatch action to handler"""
    handlers = {
        'buy': lambda: f"✅ สั่งซื้อสำเร็จ! สินค้า: {data.get('productId')}",
        'cancel': lambda: f"❌ ยกเลิกแล้ว",
        'confirm': lambda: "✅ ยืนยันแล้ว!",
        'help': lambda: "📋 ช่วยเหลือ",
    }
    
    handler = handlers.get(action)
    if handler:
        return handler()
    
    return f"Postback: {action}"
```

---

## 5. Join / Leave Events

### Join Event

เกิดขึ้นเมื่อ Bot ถูกเชิญเข้า Group หรือ Room

**JSON:**
```json
{
  "type": "join",
  "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
  "deliveryContext": {
    "isRedelivery": false
  },
  "timestamp": 1625665242211,
  "source": {
    "type": "group",
    "groupId": "Ca56f94637cc4347f90a6..."
  },
  "replyToken": "0f3779fba3b349968c5d07db31eab56f",
  "mode": "active"
}
```

**Node.js Handler:**
```javascript
async function handleJoin(event) {
  const { replyToken, source } = event;
  const sourceType = source.type; // 'group' หรือ 'room'
  const sourceId = source.groupId || source.roomId;
  
  console.log(`[JOIN] Bot joined ${sourceType}: ${sourceId}`);
  
  // ส่งข้อความแนะนำตัว
  await client.replyMessage({
    replyToken,
    messages: [{
      type: 'text',
      text: `สวัสดีทุกคน! 👋\nฉันคือ Bot ช่วยเหลือ\n\nพิมพ์ "help" เพื่อดูสิ่งที่ฉันทำได้`
    }]
  });
}
```

### Leave Event

**JSON:**
```json
{
  "type": "leave",
  "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
  "timestamp": 1625665242211,
  "source": {
    "type": "group",
    "groupId": "Ca56f94637cc4347f90a6..."
  },
  "mode": "active"
}
```

> **หมายเหตุ:** Leave Event ไม่มี `replyToken`

**Node.js Handler:**
```javascript
async function handleLeave(event) {
  const { source } = event;
  const sourceId = source.groupId || source.roomId;
  
  console.log(`[LEAVE] Bot left ${source.type}: ${sourceId}`);
  
  // บันทึกใน Database
  // await db.groups.deactivate(sourceId);
}
```

**Python:**
```python
from linebot.v3.messaging import TextMessage
from app.services import line_service


def handle_join(event) -> None:
    """Bot เข้า Group/Room"""
    source = event.source
    source_type = source.type
    source_id = getattr(source, 'group_id', None) or getattr(source, 'room_id', None)
    
    print(f"[JOIN] Bot joined {source_type}: {source_id}")
    
    line_service.reply_message(
        event.reply_token,
        TextMessage(text='สวัสดีทุกคน! 👋\nพิมพ์ "help" เพื่อดูคำสั่ง')
    )


def handle_leave(event) -> None:
    """Bot ออกจาก Group/Room"""
    source = event.source
    source_id = getattr(source, 'group_id', None) or getattr(source, 'room_id', None)
    print(f"[LEAVE] Bot left {source.type}: {source_id}")
```

---

## 6. MemberJoined / MemberLeft Events

### MemberJoined Event

**JSON:**
```json
{
  "type": "memberJoined",
  "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
  "deliveryContext": {
    "isRedelivery": false
  },
  "timestamp": 1625665242211,
  "source": {
    "type": "group",
    "groupId": "Ca56f94637cc4347f90a6..."
  },
  "replyToken": "0f3779fba3b349968c5d07db31eab56f",
  "joined": {
    "members": [
      {
        "type": "user",
        "userId": "U4af4980629..."
      },
      {
        "type": "user",
        "userId": "U5bf4980630..."
      }
    ]
  },
  "mode": "active"
}
```

**Node.js Handler:**
```javascript
async function handleMemberJoined(event) {
  const { replyToken, joined, source } = event;
  const members = joined.members;
  
  console.log(`[MEMBER_JOIN] ${members.length} members joined ${source.groupId}`);
  
  // ดึง Profile ของสมาชิกใหม่
  const profiles = await Promise.allSettled(
    members.map(member => client.getProfile(member.userId))
  );
  
  const names = profiles
    .filter(p => p.status === 'fulfilled')
    .map(p => p.value.displayName);
  
  const welcomeText = names.length > 0
    ? `ยินดีต้อนรับ ${names.join(', ')}! 🎉`
    : `ยินดีต้อนรับสมาชิกใหม่! 🎉`;
  
  await client.replyMessage({
    replyToken,
    messages: [{
      type: 'text',
      text: welcomeText
    }]
  });
}
```

### MemberLeft Event

**JSON:**
```json
{
  "type": "memberLeft",
  "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
  "timestamp": 1625665242211,
  "source": {
    "type": "group",
    "groupId": "Ca56f94637cc4347f90a6..."
  },
  "left": {
    "members": [
      {
        "type": "user",
        "userId": "U4af4980629..."
      }
    ]
  },
  "mode": "active"
}
```

> **หมายเหตุ:** MemberLeft Event ไม่มี `replyToken`

**Node.js Handler:**
```javascript
async function handleMemberLeft(event) {
  const { left, source } = event;
  const members = left.members;
  
  console.log(`[MEMBER_LEFT] ${members.length} members left ${source.groupId}`);
  
  for (const member of members) {
    console.log(`Member left: ${member.userId}`);
  }
}
```

---

## 7. Beacon Events

Beacon Events เกิดขึ้นเมื่อผู้ใช้อยู่ใกล้ LINE Beacon Device

**JSON:**
```json
{
  "type": "beacon",
  "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
  "deliveryContext": {
    "isRedelivery": false
  },
  "timestamp": 1625665242211,
  "source": {
    "type": "user",
    "userId": "U4af4980629..."
  },
  "replyToken": "0f3779fba3b349968c5d07db31eab56f",
  "beacon": {
    "hwid": "d41d8cd98f",
    "type": "enter",
    "deviceMessage": "7469702031"
  },
  "mode": "active"
}
```

**Beacon Types:**
- `enter` - ผู้ใช้เข้าใกล้ Beacon
- `leave` - ผู้ใช้ออกห่าง Beacon
- `banner` - ผู้ใช้แตะ Banner

**Node.js Handler:**
```javascript
async function handleBeacon(event) {
  const { replyToken, beacon, source } = event;
  const userId = source.userId;
  const { hwid, type, deviceMessage } = beacon;
  
  console.log(`[BEACON] User ${userId}, HWID: ${hwid}, Type: ${type}`);
  
  switch (type) {
    case 'enter':
      // ผู้ใช้เข้าใกล้
      await client.replyMessage({
        replyToken,
        messages: [{
          type: 'text',
          text: `🔵 ยินดีต้อนรับสู่ร้านของเรา!\nมีโปรโมชันพิเศษสำหรับคุณ!`
        }]
      });
      
      // บันทึกการเข้าร้าน
      // await analytics.trackStoreVisit(userId, hwid);
      break;
      
    case 'leave':
      // ผู้ใช้ออกไป
      console.log(`User ${userId} left beacon ${hwid}`);
      // ไม่มี replyToken ใน leave event
      break;
      
    case 'banner':
      // ผู้ใช้แตะ Banner
      await client.replyMessage({
        replyToken,
        messages: [{
          type: 'text',
          text: '🎁 ขอบคุณที่แตะ Banner!\nรับส่วนลด 10% วันนี้!'
        }]
      });
      break;
  }
  
  // Decode device message ถ้ามี
  if (deviceMessage) {
    const decoded = Buffer.from(deviceMessage, 'hex').toString('utf8');
    console.log(`Device message: ${decoded}`);
  }
}
```

**Python Handler:**
```python
def handle_beacon(event) -> None:
    """จัดการ Beacon Event"""
    beacon = event.beacon
    user_id = event.source.user_id
    reply_token = event.reply_token
    
    print(f"[BEACON] HWID: {beacon.hwid}, Type: {beacon.type}")
    
    if beacon.type == 'enter':
        line_service.reply_message(
            reply_token,
            TextMessage(text='🔵 ยินดีต้อนรับสู่ร้านของเรา!')
        )
    elif beacon.type == 'banner':
        line_service.reply_message(
            reply_token,
            TextMessage(text='🎁 ขอบคุณที่แตะ Banner!')
        )
    
    # Decode device message
    if beacon.device_message:
        import binascii
        decoded = binascii.unhexlify(beacon.device_message).decode('utf-8')
        print(f"Device message: {decoded}")
```

---

## 8. AccountLink Events

AccountLink Events เกิดขึ้นเมื่อผู้ใช้ทำ Account Linking

**JSON:**
```json
{
  "type": "accountLink",
  "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
  "deliveryContext": {
    "isRedelivery": false
  },
  "timestamp": 1625665242211,
  "source": {
    "type": "user",
    "userId": "U4af4980629..."
  },
  "replyToken": "0f3779fba3b349968c5d07db31eab56f",
  "link": {
    "result": "ok",
    "nonce": "xxxxxxxxxxxxxxx"
  },
  "mode": "active"
}
```

**Node.js Handler:**
```javascript
async function handleAccountLink(event) {
  const { replyToken, link, source } = event;
  const userId = source.userId;
  
  console.log(`[ACCOUNT_LINK] User: ${userId}, Result: ${link.result}`);
  
  if (link.result === 'ok') {
    // เชื่อมบัญชีสำเร็จ
    // nonce ใช้ระบุ user ในระบบของคุณ
    const nonce = link.nonce;
    
    // ดึงข้อมูล user จาก nonce (ที่บันทึกไว้ก่อนหน้า)
    // const internalUser = await db.getLinkNonce(nonce);
    // await db.linkAccounts(userId, internalUser.id);
    
    await client.replyMessage({
      replyToken,
      messages: [{
        type: 'text',
        text: '✅ เชื่อมบัญชีสำเร็จแล้ว!\nตอนนี้คุณสามารถใช้งานฟีเจอร์พิเศษได้แล้ว'
      }]
    });
  } else {
    // เชื่อมบัญชีไม่สำเร็จ
    await client.replyMessage({
      replyToken,
      messages: [{
        type: 'text',
        text: '❌ เชื่อมบัญชีไม่สำเร็จ กรุณาลองใหม่อีกครั้ง'
      }]
    });
  }
}
```

---

## 9. Things Events

Things Events เกี่ยวกับ LINE Things (IoT devices)

**JSON - Device Link:**
```json
{
  "type": "things",
  "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
  "timestamp": 1625665242211,
  "source": {
    "type": "user",
    "userId": "U4af4980629..."
  },
  "replyToken": "0f3779fba3b349968c5d07db31eab56f",
  "things": {
    "deviceId": "t2c449A29",
    "type": "link"
  },
  "mode": "active"
}
```

**JSON - Device Unlink:**
```json
{
  "things": {
    "deviceId": "t2c449A29",
    "type": "unlink"
  }
}
```

**JSON - Scenario Result:**
```json
{
  "things": {
    "deviceId": "t2c449A29",
    "type": "scenarioResult",
    "result": {
      "scenarioId": "SCENARIO_12345",
      "revision": 2,
      "startTime": 1547817900,
      "endTime": 1547817925,
      "resultCode": "success",
      "bleNotificationPayload": "AQ==",
      "actionResults": [
        {
          "type": "binary",
          "data": "/w=="
        }
      ]
    }
  }
}
```

**Node.js Handler:**
```javascript
async function handleThings(event) {
  const { replyToken, things, source } = event;
  const userId = source.userId;
  
  console.log(`[THINGS] Device: ${things.deviceId}, Type: ${things.type}`);
  
  switch (things.type) {
    case 'link':
      await client.replyMessage({
        replyToken,
        messages: [{
          type: 'text',
          text: `✅ เชื่อมต่ออุปกรณ์ ${things.deviceId} สำเร็จ!`
        }]
      });
      break;
      
    case 'unlink':
      // ไม่มี replyToken
      console.log(`Device ${things.deviceId} unlinked from user ${userId}`);
      break;
      
    case 'scenarioResult':
      const { result } = things;
      console.log(`Scenario ${result.scenarioId}: ${result.resultCode}`);
      
      if (result.bleNotificationPayload) {
        const data = Buffer.from(result.bleNotificationPayload, 'base64');
        console.log('BLE Notification:', data.toString('hex'));
      }
      break;
  }
}
```

---

## 10. โครงสร้าง Event JSON

### Common Fields

ทุก Event มี fields เหล่านี้:

```json
{
  "type": "string",              // ประเภท event
  "webhookEventId": "string",   // unique ID
  "deliveryContext": {
    "isRedelivery": false        // เป็นการส่งซ้ำหรือไม่
  },
  "timestamp": 1625665242211,   // Unix timestamp (milliseconds)
  "source": { ... },            // แหล่งที่มา
  "mode": "active"              // active | standby
}
```

### Mode Values

| Mode | ความหมาย |
|------|----------|
| `active` | Bot กำลังทำงาน (ปกติ) |
| `standby` | Bot อยู่ใน Standby (ใน Group ที่มีหลาย Bot) |

### Source Objects

```json
// User
{"type": "user", "userId": "U..."}

// Group
{"type": "group", "groupId": "C...", "userId": "U..."}

// Room
{"type": "room", "roomId": "R...", "userId": "U..."}
```

### Events ที่ไม่มี replyToken

| Event | มี replyToken |
|-------|--------------|
| message | ✅ |
| follow | ✅ |
| unfollow | ❌ |
| postback | ✅ |
| join | ✅ |
| leave | ❌ |
| memberJoined | ✅ |
| memberLeft | ❌ |
| beacon (enter/banner) | ✅ |
| beacon (leave) | ❌ |
| accountLink | ✅ |
| things (link) | ✅ |
| things (unlink) | ❌ |

---

## 11. การจัดการ Error

### ประเภท Error ที่พบบ่อย

```javascript
// 1. Invalid Reply Token
{
  "message": "Invalid reply token"
}
// สาเหตุ: ใช้ replyToken หลังจาก 30 วินาที หรือใช้ซ้ำ

// 2. Rate Limit
// HTTP 429
{
  "message": "Too many requests"
}

// 3. Invalid Access Token
// HTTP 401
{
  "message": "The channel access token is invalid"
}
```

### Error Handling Strategy

**Node.js:**
```javascript
// utils/errorHandler.js

class LineError extends Error {
  constructor(message, statusCode, originalError) {
    super(message);
    this.name = 'LineError';
    this.statusCode = statusCode;
    this.originalError = originalError;
  }
}

async function safeReply(client, replyToken, messages) {
  try {
    await client.replyMessage({ replyToken, messages });
    return true;
  } catch (error) {
    const statusCode = error.statusCode || error.response?.status;
    
    if (statusCode === 400) {
      // Invalid reply token หรือ expired
      console.warn('Reply token expired or invalid');
      return false;
    }
    
    if (statusCode === 429) {
      // Rate limited
      console.warn('Rate limited, will retry...');
      const retryAfter = parseInt(error.response?.headers?.['retry-after'] || '5');
      await new Promise(r => setTimeout(r, retryAfter * 1000));
      return safeReply(client, replyToken, messages);
    }
    
    // Log และ rethrow สำหรับ error อื่น
    console.error('Unexpected error:', error);
    throw new LineError('Failed to reply', statusCode, error);
  }
}

// Webhook Event Error Handler
async function handleEventWithErrorBoundary(event, handler) {
  const startTime = Date.now();
  
  try {
    await handler(event);
    
    const duration = Date.now() - startTime;
    console.debug(`Event processed in ${duration}ms`);
  } catch (error) {
    const duration = Date.now() - startTime;
    
    console.error({
      message: 'Event handler failed',
      eventType: event.type,
      userId: event.source?.userId,
      duration,
      error: {
        name: error.name,
        message: error.message,
        statusCode: error.statusCode,
        stack: process.env.NODE_ENV !== 'production' ? error.stack : undefined,
      }
    });
    
    // ไม่ throw error เพื่อให้ประมวลผล event ต่อไปได้
  }
}

module.exports = { safeReply, handleEventWithErrorBoundary, LineError };
```

**Python:**
```python
# utils/error_handler.py
import logging
import time
from typing import Callable
from functools import wraps

logger = logging.getLogger(__name__)


class LineAPIError(Exception):
    def __init__(self, message: str, status_code: int = None):
        super().__init__(message)
        self.status_code = status_code


def with_error_boundary(handler: Callable) -> Callable:
    """Decorator สำหรับ Event Handlers"""
    
    @wraps(handler)
    def wrapper(event, *args, **kwargs):
        start = time.time()
        
        try:
            result = handler(event, *args, **kwargs)
            duration = time.time() - start
            logger.debug(f"Event processed in {duration:.3f}s")
            return result
        except Exception as e:
            duration = time.time() - start
            logger.error(
                f"Event handler failed: {type(e).__name__}: {e}",
                extra={
                    'event_type': getattr(event, 'type', 'unknown'),
                    'user_id': getattr(event.source, 'user_id', None),
                    'duration': duration,
                }
            )
            # ไม่ raise เพื่อให้ประมวลผล event อื่นต่อได้
    
    return wrapper


def safe_reply(reply_token: str, messages, max_retries: int = 3):
    """Reply with retry logic"""
    from app.services import line_service
    
    for attempt in range(max_retries):
        try:
            line_service.reply_message(reply_token, messages)
            return True
        except Exception as e:
            status_code = getattr(e, 'status', None)
            
            if status_code == 400:
                logger.warning(f"Reply token expired (attempt {attempt + 1})")
                return False
            
            if status_code == 429:
                retry_after = 5
                logger.warning(f"Rate limited, waiting {retry_after}s...")
                time.sleep(retry_after)
                continue
            
            if attempt < max_retries - 1:
                wait = 2 ** attempt
                logger.warning(f"Retry {attempt + 1}/{max_retries} after {wait}s")
                time.sleep(wait)
                continue
            
            raise
    
    return False
```

---

## 12. Webhook Verification

### การ Verify Webhook จาก LINE Console

เมื่อกด **Verify** บน LINE Console จะส่ง Request พิเศษ:

```json
{
  "destination": "xxxxxxxxxx",
  "events": []
}
```

เซิร์ฟเวอร์ต้องตอบกลับ 200 เมื่อได้รับ Request นี้

**Node.js:**
```javascript
app.post('/webhook', verifySignature, async (req, res) => {
  const { events } = req.body;
  
  // ตอบกลับ 200 ทันที (รวมถึง verify request ที่ events = [])
  res.status(200).json({ message: 'OK' });
  
  // ประมวลผลเฉพาะเมื่อมี events
  if (!events || events.length === 0) {
    console.log('Verify request received');
    return;
  }
  
  // Process events...
});
```

### Webhook Delivery Guarantee

LINE มี **At-least-once delivery** - อาจส่ง Event ซ้ำได้

ตรวจสอบด้วย `webhookEventId`:

```javascript
// Node.js - Idempotency ด้วย Redis
const Redis = require('ioredis');
const redis = new Redis(process.env.REDIS_URL);

async function processEventOnce(event) {
  const eventId = event.webhookEventId;
  
  // ตรวจสอบว่าเคยประมวลผลแล้วหรือยัง
  const processed = await redis.get(`event:${eventId}`);
  
  if (processed) {
    console.log(`Event ${eventId} already processed, skipping`);
    return;
  }
  
  // ประมวลผล
  await handleEvent(event);
  
  // บันทึกว่าประมวลผลแล้ว (เก็บ 24 ชั่วโมง)
  await redis.setex(`event:${eventId}`, 86400, '1');
}
```

```python
# Python - Idempotency ด้วย Redis
import redis

r = redis.from_url(os.environ.get('REDIS_URL', 'redis://localhost:6379'))


def process_event_once(event, handler):
    """ประมวลผล event เพียงครั้งเดียว"""
    event_id = event.webhook_event_id
    
    # ตรวจสอบ
    if r.get(f'event:{event_id}'):
        print(f"Event {event_id} already processed")
        return
    
    # ประมวลผล
    handler(event)
    
    # บันทึก (TTL 24 ชั่วโมง)
    r.setex(f'event:{event_id}', 86400, '1')
```

### Webhook Timeout

LINE expects Response ภายใน **1 วินาที**

ถ้าไม่ได้รับ Response LINE จะ Retry:

```javascript
// ✅ ตอบกลับทันที แล้วประมวลผลทีหลัง
app.post('/webhook', async (req, res) => {
  // ตอบกลับทันที
  res.status(200).json({ message: 'OK' });
  
  // ประมวลผลแบบ Async (หลังส่ง Response แล้ว)
  const events = req.body.events || [];
  
  // ใช้ setImmediate เพื่อให้ Response ออกไปก่อน
  setImmediate(async () => {
    for (const event of events) {
      await handleEvent(event);
    }
  });
});
```

```python
# FastAPI - ตอบกลับทันที ประมวลผลทีหลัง
from fastapi import BackgroundTasks

@app.post('/webhook')
async def webhook(
    request: Request,
    background_tasks: BackgroundTasks,
    x_line_signature: str = Header(alias="X-Line-Signature", default=None)
):
    body = await request.body()
    
    try:
        events = parser.parse(body.decode(), x_line_signature)
    except InvalidSignatureError:
        raise HTTPException(403)
    
    # เพิ่ม background tasks
    for event in events:
        background_tasks.add_task(process_event, event)
    
    # ตอบกลับทันที
    return {"message": "OK"}
```

---

## 13. ตัวอย่างโค้ด Node.js และ Python

### Event Dispatcher (Node.js)

```javascript
// src/eventDispatcher.js
const handlers = {
  message: require('./handlers/messageHandler'),
  follow: require('./handlers/followHandler'),
  unfollow: require('./handlers/followHandler'),
  postback: require('./handlers/postbackHandler'),
  join: require('./handlers/groupHandler'),
  leave: require('./handlers/groupHandler'),
  memberJoined: require('./handlers/memberHandler'),
  memberLeft: require('./handlers/memberHandler'),
  beacon: require('./handlers/beaconHandler'),
  accountLink: require('./handlers/accountLinkHandler'),
  things: require('./handlers/thingsHandler'),
  videoViewingComplete: require('./handlers/videoHandler'),
  unsend: require('./handlers/unsendHandler'),
};

const eventMethodMap = {
  message: 'handleMessage',
  follow: 'handleFollow',
  unfollow: 'handleUnfollow',
  postback: 'handlePostback',
  join: 'handleJoin',
  leave: 'handleLeave',
  memberJoined: 'handleMemberJoined',
  memberLeft: 'handleMemberLeft',
  beacon: 'handleBeacon',
  accountLink: 'handleAccountLink',
  things: 'handleThings',
  videoViewingComplete: 'handleVideoComplete',
  unsend: 'handleUnsend',
};

async function dispatch(event) {
  const { type } = event;
  const handler = handlers[type];
  const method = eventMethodMap[type];
  
  if (!handler || !method || typeof handler[method] !== 'function') {
    console.warn(`No handler for event type: ${type}`);
    return;
  }
  
  try {
    await handler[method](event);
  } catch (error) {
    console.error(`Error in ${type} handler:`, {
      message: error.message,
      userId: event.source?.userId,
    });
  }
}

module.exports = { dispatch };
```

### Event Dispatcher (Python)

```python
# app/event_dispatcher.py
from typing import Dict, Callable
from linebot.v3.webhooks import (
    MessageEvent, FollowEvent, UnfollowEvent,
    PostbackEvent, JoinEvent, LeaveEvent,
    MemberJoinedEvent, MemberLeftEvent,
    BeaconEvent, AccountLinkEvent, ThingsEvent,
)
import logging

logger = logging.getLogger(__name__)


class EventDispatcher:
    """Dispatch Webhook Events to Handlers"""
    
    def __init__(self):
        self._handlers: Dict[type, Callable] = {}
    
    def register(self, event_type: type, handler: Callable):
        """Register handler for event type"""
        self._handlers[event_type] = handler
        return self
    
    def dispatch(self, event) -> None:
        """Dispatch event to registered handler"""
        event_class = type(event)
        handler = self._handlers.get(event_class)
        
        if not handler:
            logger.warning(f"No handler for {event_class.__name__}")
            return
        
        try:
            handler(event)
        except Exception as e:
            logger.error(
                f"Error in {event_class.__name__} handler: {e}",
                exc_info=True
            )


# สร้าง dispatcher
from app.handlers import (
    message_handler,
    follow_handler,
    postback_handler,
    group_handler,
    member_handler,
    beacon_handler,
    account_link_handler,
    things_handler,
)

dispatcher = EventDispatcher()
dispatcher \
    .register(MessageEvent, message_handler.handle_message) \
    .register(FollowEvent, follow_handler.handle_follow) \
    .register(UnfollowEvent, follow_handler.handle_unfollow) \
    .register(PostbackEvent, postback_handler.handle_postback) \
    .register(JoinEvent, group_handler.handle_join) \
    .register(LeaveEvent, group_handler.handle_leave) \
    .register(MemberJoinedEvent, member_handler.handle_member_joined) \
    .register(MemberLeftEvent, member_handler.handle_member_left) \
    .register(BeaconEvent, beacon_handler.handle_beacon) \
    .register(AccountLinkEvent, account_link_handler.handle_account_link) \
    .register(ThingsEvent, things_handler.handle_things)
```

### Complete Webhook Handler

```javascript
// src/routes/webhook.js - Complete Handler
const express = require('express');
const router = express.Router();
const { dispatch } = require('../eventDispatcher');
const { verifySignature } = require('../middleware/auth');
const logger = require('../utils/logger');

router.post('/', async (req, res) => {
  // Verify Signature
  const signature = req.headers['x-line-signature'];
  const rawBody = req.rawBody;
  
  if (!signature || !verifySignature(rawBody, signature)) {
    return res.status(403).json({ error: 'Invalid signature' });
  }
  
  const { destination, events } = req.body;
  
  logger.info(`Webhook: destination=${destination}, events=${events?.length || 0}`);
  
  // ตอบกลับทันที
  res.status(200).json({ message: 'OK' });
  
  // Process events
  if (!events || events.length === 0) return;
  
  const promises = events.map(event => 
    dispatch(event).catch(err => {
      logger.error(`Failed to dispatch event: ${err.message}`);
    })
  );
  
  await Promise.allSettled(promises);
});

module.exports = router;
```

---

## 14. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Event Logger
สร้าง Webhook Handler ที่:
- บันทึกทุก Event ลงไฟล์ JSON
- แสดง Summary เมื่อพิมพ์ "stats"
- รองรับทุก Event Type

### แบบฝึกหัดที่ 2: Group Manager Bot
สร้าง Bot สำหรับ Group ที่:
- ต้อนรับสมาชิกใหม่ด้วย Flex Message
- ส่งข้อความเมื่อมีคนออก
- แนะนำตัวเมื่อเข้า Group ใหม่

### แบบฝึกหัดที่ 3: Postback Menu
สร้าง Bot ที่มีเมนูซับซ้อน:
- เมนูหลักด้วย Quick Reply
- Sub-menu ด้วย Postback
- Back Button กลับ

### แบบฝึกหัดที่ 4: Idempotent Webhook
สร้าง Webhook Handler ที่:
- ตรวจสอบ webhookEventId ก่อนประมวลผล
- ใช้ Redis หรือ In-memory Map บันทึก processed events
- Log เมื่อพบ duplicate event

### แบบฝึกหัดที่ 5: Event Analytics
สร้างระบบวิเคราะห์ Events:
- นับจำนวน Events แต่ละประเภท
- Track User Activity (follow/unfollow/message)
- สร้าง API endpoint ดู Analytics

---

## สรุป

ในบทนี้เราได้เรียนรู้ Webhook Events ทุกประเภท:

1. **Message Events** - Text, Image, Video, Audio, File, Location, Sticker
2. **Follow/Unfollow** - การจัดการผู้ติดตาม
3. **Postback** - การจัดการ Button Actions ที่ซับซ้อน
4. **Join/Leave** - Bot เข้า/ออก Groups
5. **MemberJoined/Left** - สมาชิก Group เข้า/ออก
6. **Beacon** - LINE Beacon IoT Events
7. **AccountLink** - การเชื่อมบัญชี
8. **Things** - LINE Things IoT Events
9. **Error Handling** - Retry, Rate Limit, Timeout
10. **Webhook Verification** - Signature, Idempotency, Timeout

### Key Takeaways

- ✅ ตอบกลับ 200 ทันที ก่อนประมวลผล Events
- ✅ ตรวจสอบ Signature ทุก Request
- ✅ ใช้ `webhookEventId` ป้องกัน Duplicate Processing
- ✅ Handle Error ทุก Event โดยไม่ให้กระทบ Event อื่น
- ✅ Log ทุก Event สำหรับ Debug
- ✅ ไม่ Block Main Thread ด้วย Heavy Processing

ยินดีด้วย! คุณเรียนรู้พื้นฐาน LINE Messaging API ครบถ้วนแล้ว!

---

*เอกสารอ้างอิง: https://developers.line.biz/en/docs/messaging-api/receiving-messages/*
