# Part 22: การสร้าง LINE Bot ด้วย Node.js

## สารบัญ

1. [Prerequisites](#prerequisites)
2. [การสร้างโปรเจกต์ Node.js](#การสร้างโปรเจกต์)
3. [การติดตั้ง @line/bot-sdk](#การติดตั้ง-linebotsdk)
4. [Project Structure](#project-structure)
5. [Express.js Setup สำหรับ Webhook](#expressjs-setup)
6. [การจัดการ Webhook Events](#การจัดการ-webhook-events)
7. [Signature Verification](#signature-verification)
8. [การ Reply ข้อความ](#การ-reply-ขอความ)
9. [Echo Bot สมบูรณ์](#echo-bot-สมบูรณ)
10. [Environment Variables](#environment-variables)
11. [Ngrok สำหรับ Local Development](#ngrok)
12. [Deploy บน Railway/Render](#deployment)
13. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Prerequisites

### สิ่งที่ต้องมีก่อน

**ซอฟต์แวร์ที่ต้องติดตั้ง:**

| ซอฟต์แวร์ | Version | Download |
|-----------|---------|----------|
| Node.js | 18.x หรือใหม่กว่า | https://nodejs.org |
| npm | 8.x+ (มากับ Node.js) | - |
| Git | 2.x+ | https://git-scm.com |
| VS Code | Latest | https://code.visualstudio.com |
| Ngrok | Latest | https://ngrok.com |

**ตรวจสอบการติดตั้ง:**

```bash
# ตรวจสอบ Node.js
node --version
# v20.11.0

# ตรวจสอบ npm
npm --version
# 10.2.4

# ตรวจสอบ Git
git --version
# git version 2.43.0
```

**ความรู้ที่ควรมี:**
- JavaScript พื้นฐาน
- HTTP/REST API เบื้องต้น
- JSON format
- Command Line เบื้องต้น

### LINE Developer Account

1. บัญชี LINE Personal
2. สร้าง Channel บน LINE Developers Console แล้ว (ดู Part 21)
3. มี Channel Access Token
4. มี Channel Secret

---

## 2. การสร้างโปรเจกต์ Node.js

### ขั้นตอนการสร้างโปรเจกต์

```bash
# สร้างโฟลเดอร์โปรเจกต์
mkdir my-line-bot
cd my-line-bot

# เริ่มต้น npm project
npm init -y
```

**package.json ที่ได้:**

```json
{
  "name": "my-line-bot",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```

**แก้ไข package.json:**

```json
{
  "name": "my-line-bot",
  "version": "1.0.0",
  "description": "LINE Bot สำหรับธุรกิจ",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "test": "jest"
  },
  "keywords": ["line", "bot", "chatbot"],
  "author": "Your Name",
  "license": "MIT",
  "engines": {
    "node": ">=18.0.0"
  }
}
```

---

## 3. การติดตั้ง @line/bot-sdk

### Dependencies หลัก

```bash
# ติดตั้ง LINE Bot SDK
npm install @line/bot-sdk

# ติดตั้ง Express.js
npm install express

# ติดตั้ง dotenv สำหรับ Environment Variables
npm install dotenv

# ติดตั้ง axios สำหรับ HTTP requests (optional)
npm install axios
```

### Dev Dependencies

```bash
# ติดตั้ง nodemon สำหรับ Auto-reload ระหว่าง Development
npm install --save-dev nodemon

# ติดตั้ง Jest สำหรับ Testing
npm install --save-dev jest

# ติดตั้ง ESLint สำหรับ Code Quality
npm install --save-dev eslint
```

### ตรวจสอบ package.json หลังติดตั้ง

```json
{
  "name": "my-line-bot",
  "version": "1.0.0",
  "description": "LINE Bot สำหรับธุรกิจ",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "test": "jest --coverage"
  },
  "dependencies": {
    "@line/bot-sdk": "^8.0.0",
    "axios": "^1.6.0",
    "dotenv": "^16.3.0",
    "express": "^4.18.0"
  },
  "devDependencies": {
    "eslint": "^8.57.0",
    "jest": "^29.7.0",
    "nodemon": "^3.0.0"
  }
}
```

---

## 4. Project Structure

### โครงสร้างโปรเจกต์ที่แนะนำ

```
my-line-bot/
├── src/
│   ├── index.js              # Entry point
│   ├── config.js             # Configuration
│   ├── middleware/
│   │   ├── auth.js           # Signature verification
│   │   └── error.js          # Error handling
│   ├── handlers/
│   │   ├── messageHandler.js  # Handle message events
│   │   ├── followHandler.js   # Handle follow/unfollow
│   │   └── postbackHandler.js # Handle postback events
│   ├── services/
│   │   ├── lineService.js     # LINE API wrapper
│   │   └── userService.js     # User management
│   ├── utils/
│   │   ├── logger.js          # Logging utility
│   │   └── helpers.js         # Helper functions
│   └── routes/
│       ├── webhook.js         # Webhook routes
│       └── health.js          # Health check
├── tests/
│   ├── handlers/
│   │   └── messageHandler.test.js
│   └── services/
│       └── lineService.test.js
├── .env                      # Environment variables (ไม่ push ขึ้น Git)
├── .env.example              # Template สำหรับ .env
├── .gitignore
├── package.json
└── README.md
```

### สร้างโครงสร้างโฟลเดอร์

```bash
mkdir -p src/{middleware,handlers,services,utils,routes}
mkdir -p tests/{handlers,services}
touch src/index.js
touch src/config.js
touch src/middleware/auth.js
touch src/middleware/error.js
touch src/handlers/messageHandler.js
touch src/handlers/followHandler.js
touch src/handlers/postbackHandler.js
touch src/services/lineService.js
touch src/utils/logger.js
touch src/routes/webhook.js
touch .env
touch .env.example
touch .gitignore
```

### .gitignore

```gitignore
# Dependencies
node_modules/

# Environment variables
.env
.env.local
.env.*.local

# Logs
logs/
*.log
npm-debug.log*

# Runtime data
pids
*.pid
*.seed
*.pid.lock

# Build output
dist/
build/

# OS files
.DS_Store
Thumbs.db

# IDE files
.idea/
.vscode/settings.json
*.swp
*.swo
```

---

## 5. Express.js Setup สำหรับ Webhook

### src/config.js

```javascript
// src/config.js
require('dotenv').config();

const config = {
  // LINE Configuration
  line: {
    channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
    channelSecret: process.env.LINE_CHANNEL_SECRET,
  },
  
  // Server Configuration
  server: {
    port: parseInt(process.env.PORT || '3000', 10),
    env: process.env.NODE_ENV || 'development',
  },
  
  // Webhook
  webhook: {
    path: process.env.WEBHOOK_PATH || '/webhook',
  }
};

// Validate required config
function validateConfig() {
  const required = [
    'line.channelAccessToken',
    'line.channelSecret',
  ];
  
  const missing = required.filter(key => {
    const value = key.split('.').reduce((obj, k) => obj?.[k], config);
    return !value;
  });
  
  if (missing.length > 0) {
    throw new Error(`Missing required config: ${missing.join(', ')}`);
  }
}

validateConfig();

module.exports = config;
```

### src/utils/logger.js

```javascript
// src/utils/logger.js
const LOG_LEVELS = {
  ERROR: 0,
  WARN: 1,
  INFO: 2,
  DEBUG: 3,
};

const currentLevel = LOG_LEVELS[
  (process.env.LOG_LEVEL || 'INFO').toUpperCase()
] ?? LOG_LEVELS.INFO;

function formatMessage(level, message, data) {
  const timestamp = new Date().toISOString();
  const dataStr = data ? ` ${JSON.stringify(data)}` : '';
  return `[${timestamp}] [${level}] ${message}${dataStr}`;
}

const logger = {
  error: (message, data) => {
    if (currentLevel >= LOG_LEVELS.ERROR) {
      console.error(formatMessage('ERROR', message, data));
    }
  },
  warn: (message, data) => {
    if (currentLevel >= LOG_LEVELS.WARN) {
      console.warn(formatMessage('WARN', message, data));
    }
  },
  info: (message, data) => {
    if (currentLevel >= LOG_LEVELS.INFO) {
      console.log(formatMessage('INFO', message, data));
    }
  },
  debug: (message, data) => {
    if (currentLevel >= LOG_LEVELS.DEBUG) {
      console.log(formatMessage('DEBUG', message, data));
    }
  },
};

module.exports = logger;
```

### src/middleware/auth.js

```javascript
// src/middleware/auth.js
const crypto = require('crypto');
const config = require('../config');
const logger = require('../utils/logger');

function verifySignature(body, signature) {
  const hmac = crypto.createHmac('SHA256', config.line.channelSecret);
  hmac.update(body);
  const digest = hmac.digest('base64');
  
  return crypto.timingSafeEqual(
    Buffer.from(digest),
    Buffer.from(signature || '')
  );
}

function webhookAuth(req, res, next) {
  const signature = req.headers['x-line-signature'];
  
  if (!signature) {
    logger.warn('Missing X-Line-Signature header');
    return res.status(401).json({ error: 'Missing signature' });
  }
  
  const rawBody = req.rawBody;
  
  if (!rawBody) {
    logger.error('Raw body not available');
    return res.status(500).json({ error: 'Internal server error' });
  }
  
  try {
    if (!verifySignature(rawBody, signature)) {
      logger.warn('Invalid signature', { signature });
      return res.status(403).json({ error: 'Invalid signature' });
    }
    
    next();
  } catch (error) {
    logger.error('Signature verification error', { error: error.message });
    return res.status(500).json({ error: 'Internal server error' });
  }
}

module.exports = { webhookAuth, verifySignature };
```

### src/middleware/error.js

```javascript
// src/middleware/error.js
const logger = require('../utils/logger');

function errorHandler(err, req, res, next) {
  logger.error('Unhandled error', {
    message: err.message,
    stack: err.stack,
    url: req.url,
    method: req.method,
  });
  
  if (res.headersSent) {
    return next(err);
  }
  
  const status = err.status || err.statusCode || 500;
  const message = process.env.NODE_ENV === 'production'
    ? 'Internal Server Error'
    : err.message;
  
  res.status(status).json({ error: message });
}

function notFoundHandler(req, res) {
  res.status(404).json({ error: 'Not Found' });
}

module.exports = { errorHandler, notFoundHandler };
```

### src/routes/health.js

```javascript
// src/routes/health.js
const express = require('express');
const router = express.Router();

router.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    memory: process.memoryUsage(),
  });
});

router.get('/', (req, res) => {
  res.json({ message: 'LINE Bot Server is running' });
});

module.exports = router;
```

---

## 6. การจัดการ Webhook Events

### ประเภท Events ที่ต้องจัดการ

```javascript
// src/handlers/eventDispatcher.js
const messageHandler = require('./messageHandler');
const followHandler = require('./followHandler');
const postbackHandler = require('./postbackHandler');
const logger = require('../utils/logger');

const eventHandlers = {
  message: messageHandler.handle,
  follow: followHandler.handleFollow,
  unfollow: followHandler.handleUnfollow,
  postback: postbackHandler.handle,
  join: (event) => logger.info('Bot joined group/room', { source: event.source }),
  leave: (event) => logger.info('Bot left group/room', { source: event.source }),
  memberJoined: (event) => logger.info('Member joined', { members: event.joined.members }),
  memberLeft: (event) => logger.info('Member left', { members: event.left.members }),
  beacon: (event) => logger.info('Beacon event', { beacon: event.beacon }),
  accountLink: (event) => logger.info('Account linked', { link: event.link }),
};

async function dispatchEvent(event) {
  const handler = eventHandlers[event.type];
  
  if (!handler) {
    logger.warn(`No handler for event type: ${event.type}`);
    return;
  }
  
  try {
    await handler(event);
  } catch (error) {
    logger.error(`Error handling ${event.type} event`, {
      error: error.message,
      event: JSON.stringify(event),
    });
  }
}

module.exports = { dispatchEvent };
```

### src/handlers/messageHandler.js

```javascript
// src/handlers/messageHandler.js
const lineService = require('../services/lineService');
const logger = require('../utils/logger');

async function handle(event) {
  if (!event.replyToken) {
    logger.warn('No reply token in message event');
    return;
  }
  
  const { type } = event.message;
  
  switch (type) {
    case 'text':
      await handleTextMessage(event);
      break;
    case 'image':
      await handleImageMessage(event);
      break;
    case 'video':
      await handleVideoMessage(event);
      break;
    case 'audio':
      await handleAudioMessage(event);
      break;
    case 'file':
      await handleFileMessage(event);
      break;
    case 'location':
      await handleLocationMessage(event);
      break;
    case 'sticker':
      await handleStickerMessage(event);
      break;
    default:
      logger.warn(`Unknown message type: ${type}`);
      await lineService.replyMessage(event.replyToken, {
        type: 'text',
        text: 'ขออภัย ฉันยังไม่รองรับข้อความประเภทนี้'
      });
  }
}

async function handleTextMessage(event) {
  const { replyToken, message, source } = event;
  const text = message.text;
  const userId = source.userId;
  
  logger.info('Received text message', { userId, text });
  
  // Echo the message back
  await lineService.replyMessage(replyToken, {
    type: 'text',
    text: `คุณพิมพ์ว่า: "${text}"`
  });
}

async function handleImageMessage(event) {
  const { replyToken, message } = event;
  
  logger.info('Received image message', { messageId: message.id });
  
  await lineService.replyMessage(replyToken, {
    type: 'text',
    text: '📷 ขอบคุณสำหรับรูปภาพ! ฉันได้รับแล้ว'
  });
}

async function handleVideoMessage(event) {
  const { replyToken, message } = event;
  
  await lineService.replyMessage(replyToken, {
    type: 'text',
    text: '🎥 ขอบคุณสำหรับวิดีโอ!'
  });
}

async function handleAudioMessage(event) {
  const { replyToken, message } = event;
  
  await lineService.replyMessage(replyToken, {
    type: 'text',
    text: '🎵 ขอบคุณสำหรับไฟล์เสียง!'
  });
}

async function handleFileMessage(event) {
  const { replyToken, message } = event;
  
  await lineService.replyMessage(replyToken, {
    type: 'text',
    text: `📎 ขอบคุณสำหรับไฟล์: ${message.fileName}`
  });
}

async function handleLocationMessage(event) {
  const { replyToken, message } = event;
  const { title, address, latitude, longitude } = message;
  
  await lineService.replyMessage(replyToken, [
    {
      type: 'text',
      text: `📍 ได้รับตำแหน่ง:\n${title || 'ไม่มีชื่อ'}\n${address}`
    },
    {
      type: 'location',
      title: title || 'ตำแหน่งที่ส่งมา',
      address: address,
      latitude: latitude,
      longitude: longitude
    }
  ]);
}

async function handleStickerMessage(event) {
  const { replyToken, message } = event;
  const { packageId, stickerId } = message;
  
  // ส่ง Sticker กลับ
  await lineService.replyMessage(replyToken, {
    type: 'sticker',
    packageId: '446',
    stickerId: '1988'
  });
}

module.exports = { handle };
```

### src/handlers/followHandler.js

```javascript
// src/handlers/followHandler.js
const lineService = require('../services/lineService');
const logger = require('../utils/logger');

async function handleFollow(event) {
  const { replyToken, source } = event;
  const userId = source.userId;
  
  logger.info('New follower', { userId });
  
  // ดึงข้อมูล User Profile
  let userName = 'คุณ';
  try {
    const profile = await lineService.getUserProfile(userId);
    userName = profile.displayName;
  } catch (error) {
    logger.error('Failed to get user profile', { error: error.message });
  }
  
  // ส่งข้อความต้อนรับ
  await lineService.replyMessage(replyToken, [
    {
      type: 'text',
      text: `ยินดีต้อนรับ ${userName}! 🎉\n\nฉันคือ LINE Bot ของเรา ยินดีที่ได้รู้จัก!`
    },
    {
      type: 'text',
      text: '📋 สิ่งที่ฉันทำได้:\n• ตอบคำถาม\n• ให้ข้อมูลสินค้า\n• รับออเดอร์\n\nพิมพ์ "ช่วยเหลือ" เพื่อดูคำสั่งทั้งหมด'
    }
  ]);
}

async function handleUnfollow(event) {
  const userId = event.source.userId;
  
  logger.info('User unfollowed', { userId });
  
  // บันทึกในฐานข้อมูล (ถ้ามี)
  // await userService.markAsUnfollowed(userId);
}

module.exports = { handleFollow, handleUnfollow };
```

---

## 7. Signature Verification (อธิบายละเอียด)

### วิธีการทำงานของ Signature

1. LINE สร้าง HMAC-SHA256 จาก Request Body โดยใช้ Channel Secret
2. LINE ใส่ค่านี้ใน HTTP Header `X-Line-Signature`
3. เซิร์ฟเวอร์ของคุณสร้าง HMAC-SHA256 ด้วยวิธีเดียวกัน
4. เปรียบเทียบทั้งสอง ถ้าเท่ากัน = ของจริง

```
LINE Platform → [Request Body + Channel Secret] → HMAC-SHA256 → base64 → X-Line-Signature
Your Server   → [Request Body + Channel Secret] → HMAC-SHA256 → base64 → Compare
```

### ข้อสำคัญของการ Verify Signature

```javascript
// ❌ ผิด - ใช้ Buffer.from ธรรมดา อาจเจอ Timing Attack
function verifyBad(body, signature, secret) {
  const expected = crypto.createHmac('SHA256', secret)
    .update(body)
    .digest('base64');
  return expected === signature; // String comparison ไม่ปลอดภัย!
}

// ✅ ถูก - ใช้ timingSafeEqual เพื่อป้องกัน Timing Attack
function verifyGood(body, signature, secret) {
  const expected = crypto.createHmac('SHA256', secret)
    .update(body)
    .digest('base64');
  
  try {
    return crypto.timingSafeEqual(
      Buffer.from(expected),
      Buffer.from(signature)
    );
  } catch {
    return false; // length mismatch
  }
}
```

### Middleware สำหรับ Raw Body

```javascript
// สำคัญ: ต้องเก็บ Raw Body ก่อนที่ Express จะ Parse JSON
app.use(express.json({
  verify: (req, res, buf, encoding) => {
    req.rawBody = buf.toString(encoding || 'utf8');
  }
}));
```

---

## 8. การ Reply ข้อความ

### src/services/lineService.js

```javascript
// src/services/lineService.js
const line = require('@line/bot-sdk');
const config = require('../config');
const logger = require('../utils/logger');

// สร้าง LINE Client
const client = new line.messagingApi.MessagingApiClient({
  channelAccessToken: config.line.channelAccessToken,
});

// Reply Message
async function replyMessage(replyToken, messages) {
  if (!Array.isArray(messages)) {
    messages = [messages];
  }
  
  if (messages.length > 5) {
    throw new Error('Cannot send more than 5 messages at once');
  }
  
  try {
    const result = await client.replyMessage({
      replyToken,
      messages,
    });
    
    logger.debug('Reply sent successfully', { sentMessages: result.sentMessages });
    return result;
  } catch (error) {
    logger.error('Failed to reply message', {
      error: error.message,
      statusCode: error.statusCode,
      originalError: error.originalError?.message,
    });
    throw error;
  }
}

// Push Message
async function pushMessage(to, messages) {
  if (!Array.isArray(messages)) {
    messages = [messages];
  }
  
  try {
    const result = await client.pushMessage({ to, messages });
    logger.debug('Push message sent', { to, messageCount: messages.length });
    return result;
  } catch (error) {
    logger.error('Failed to push message', { error: error.message, to });
    throw error;
  }
}

// Multicast Message
async function multicastMessage(to, messages) {
  if (!Array.isArray(to)) {
    to = [to];
  }
  if (!Array.isArray(messages)) {
    messages = [messages];
  }
  
  // LINE รองรับสูงสุด 500 recipients ต่อครั้ง
  const results = [];
  for (let i = 0; i < to.length; i += 500) {
    const batch = to.slice(i, i + 500);
    const result = await client.multicast({ to: batch, messages });
    results.push(result);
  }
  
  return results;
}

// Get User Profile
async function getUserProfile(userId) {
  try {
    return await client.getProfile(userId);
  } catch (error) {
    logger.error('Failed to get user profile', { error: error.message, userId });
    throw error;
  }
}

// Get Group Member Profile
async function getGroupMemberProfile(groupId, userId) {
  try {
    return await client.getGroupMemberProfile(groupId, userId);
  } catch (error) {
    logger.error('Failed to get group member profile', { error: error.message });
    throw error;
  }
}

// Get Bot Info
async function getBotInfo() {
  try {
    return await client.getBotInfo();
  } catch (error) {
    logger.error('Failed to get bot info', { error: error.message });
    throw error;
  }
}

// Get Message Content (image, video, audio, file)
async function getMessageContent(messageId) {
  try {
    return await client.getMessageContent(messageId);
  } catch (error) {
    logger.error('Failed to get message content', { error: error.message, messageId });
    throw error;
  }
}

// Set Typing Indicator
async function showLoadingAnimation(chatId, loadingSeconds = 5) {
  try {
    await client.showLoadingAnimation({ chatId, loadingSeconds });
  } catch (error) {
    logger.error('Failed to show loading animation', { error: error.message });
  }
}

module.exports = {
  client,
  replyMessage,
  pushMessage,
  multicastMessage,
  getUserProfile,
  getGroupMemberProfile,
  getBotInfo,
  getMessageContent,
  showLoadingAnimation,
};
```

### ประเภทของ Messages

```javascript
// Text Message
const textMessage = {
  type: 'text',
  text: 'สวัสดีครับ!',
  emojis: [
    {
      index: 0,
      productId: '5ac1bfd5040ab15980c9b435',
      emojiId: '001'
    }
  ]
};

// Image Message
const imageMessage = {
  type: 'image',
  originalContentUrl: 'https://example.com/image.jpg',  // max 10MB
  previewImageUrl: 'https://example.com/preview.jpg'    // max 1MB
};

// Video Message
const videoMessage = {
  type: 'video',
  originalContentUrl: 'https://example.com/video.mp4',  // max 200MB
  previewImageUrl: 'https://example.com/thumbnail.jpg'
};

// Audio Message
const audioMessage = {
  type: 'audio',
  originalContentUrl: 'https://example.com/audio.m4a',
  duration: 60000  // milliseconds
};

// Location Message
const locationMessage = {
  type: 'location',
  title: 'บริษัทของเรา',
  address: '123 ถนนสุขุมวิท กรุงเทพฯ',
  latitude: 13.7563,
  longitude: 100.5018
};

// Sticker Message
const stickerMessage = {
  type: 'sticker',
  packageId: '446',
  stickerId: '1988'
};

// Quick Reply
const quickReplyMessage = {
  type: 'text',
  text: 'เลือกตัวเลือก:',
  quickReply: {
    items: [
      {
        type: 'action',
        action: {
          type: 'message',
          label: 'ใช่',
          text: 'ใช่'
        }
      },
      {
        type: 'action',
        action: {
          type: 'message',
          label: 'ไม่ใช่',
          text: 'ไม่ใช่'
        }
      },
      {
        type: 'action',
        action: {
          type: 'location',
          label: 'ส่งตำแหน่ง'
        }
      }
    ]
  }
};

// Flex Message
const flexMessage = {
  type: 'flex',
  altText: 'สินค้าแนะนำ',
  contents: {
    type: 'bubble',
    hero: {
      type: 'image',
      url: 'https://example.com/product.jpg',
      size: 'full',
      aspectRatio: '20:13',
      action: {
        type: 'uri',
        uri: 'https://example.com/product'
      }
    },
    body: {
      type: 'box',
      layout: 'vertical',
      contents: [
        {
          type: 'text',
          text: 'สินค้าพิเศษ',
          weight: 'bold',
          size: 'xl'
        },
        {
          type: 'text',
          text: 'ราคา ฿999',
          size: 'lg',
          color: '#FF0000'
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
            type: 'uri',
            label: 'ซื้อเลย',
            uri: 'https://example.com/buy'
          }
        }
      ]
    }
  }
};
```

---

## 9. Echo Bot สมบูรณ์

### src/routes/webhook.js

```javascript
// src/routes/webhook.js
const express = require('express');
const router = express.Router();
const { webhookAuth } = require('../middleware/auth');
const { dispatchEvent } = require('../handlers/eventDispatcher');
const logger = require('../utils/logger');

router.post('/', webhookAuth, async (req, res) => {
  const { destination, events } = req.body;
  
  logger.info('Webhook received', {
    destination,
    eventCount: events?.length || 0,
  });
  
  // ตอบกลับ 200 ทันที ก่อนประมวลผล
  res.status(200).json({ message: 'OK' });
  
  if (!events || !Array.isArray(events)) {
    return;
  }
  
  // ประมวลผล events แบบ Async
  const promises = events.map(event => dispatchEvent(event));
  
  try {
    await Promise.allSettled(promises);
  } catch (error) {
    logger.error('Error processing webhook events', { error: error.message });
  }
});

module.exports = router;
```

### src/index.js (Entry Point)

```javascript
// src/index.js
require('dotenv').config();
const express = require('express');
const config = require('./config');
const logger = require('./utils/logger');
const webhookRouter = require('./routes/webhook');
const healthRouter = require('./routes/health');
const { errorHandler, notFoundHandler } = require('./middleware/error');

const app = express();

// Raw body parser (ต้องก่อน JSON parser)
app.use(express.json({
  verify: (req, res, buf, encoding) => {
    req.rawBody = buf.toString(encoding || 'utf8');
  }
}));

app.use(express.urlencoded({ extended: true }));

// Request logging middleware
app.use((req, res, next) => {
  logger.info(`${req.method} ${req.path}`, {
    ip: req.ip,
    userAgent: req.headers['user-agent'],
  });
  next();
});

// Routes
app.use('/', healthRouter);
app.use(config.webhook.path, webhookRouter);

// Error handlers
app.use(notFoundHandler);
app.use(errorHandler);

// Start server
const port = config.server.port;
app.listen(port, () => {
  logger.info(`LINE Bot server started`, {
    port,
    env: config.server.env,
    webhookPath: config.webhook.path,
  });
});

// Graceful shutdown
process.on('SIGTERM', () => {
  logger.info('SIGTERM received, shutting down gracefully');
  process.exit(0);
});

process.on('unhandledRejection', (reason, promise) => {
  logger.error('Unhandled Rejection', { reason, promise });
});

module.exports = app;
```

### ตัวอย่าง Echo Bot แบบ Simple (ไฟล์เดียว)

```javascript
// simple-echo-bot.js
require('dotenv').config();
const express = require('express');
const line = require('@line/bot-sdk');
const crypto = require('crypto');

const app = express();

// LINE SDK Config
const lineConfig = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET,
};

const client = new line.messagingApi.MessagingApiClient({
  channelAccessToken: lineConfig.channelAccessToken,
});

// Raw body parser สำหรับ Signature Verification
app.use(express.json({
  verify: (req, res, buf) => {
    req.rawBody = buf;
  }
}));

// Signature Verification Middleware
function verifyLineSignature(req, res, next) {
  const signature = req.headers['x-line-signature'];
  
  if (!signature) {
    return res.status(401).send('Unauthorized');
  }
  
  const body = req.rawBody?.toString() || '';
  const hmac = crypto.createHmac('SHA256', lineConfig.channelSecret);
  hmac.update(body);
  const expectedSignature = hmac.digest('base64');
  
  try {
    if (!crypto.timingSafeEqual(
      Buffer.from(expectedSignature),
      Buffer.from(signature)
    )) {
      return res.status(403).send('Forbidden');
    }
  } catch (e) {
    return res.status(403).send('Forbidden');
  }
  
  next();
}

// Webhook endpoint
app.post('/webhook', verifyLineSignature, async (req, res) => {
  res.status(200).json({ message: 'OK' });
  
  const events = req.body.events || [];
  
  for (const event of events) {
    try {
      await handleEvent(event);
    } catch (error) {
      console.error('Error handling event:', error);
    }
  }
});

// Event handler
async function handleEvent(event) {
  console.log('Event:', JSON.stringify(event, null, 2));
  
  switch (event.type) {
    case 'message':
      await handleMessage(event);
      break;
    case 'follow':
      await handleFollow(event);
      break;
    case 'unfollow':
      console.log('User unfollowed:', event.source.userId);
      break;
    case 'postback':
      await handlePostback(event);
      break;
    default:
      console.log('Unknown event type:', event.type);
  }
}

// Message handler
async function handleMessage(event) {
  const { replyToken, message } = event;
  
  switch (message.type) {
    case 'text':
      // Echo text message
      await client.replyMessage({
        replyToken,
        messages: [{
          type: 'text',
          text: `Echo: ${message.text}`
        }]
      });
      break;
      
    case 'image':
      await client.replyMessage({
        replyToken,
        messages: [{
          type: 'text',
          text: '📷 ได้รับรูปภาพแล้ว!'
        }]
      });
      break;
      
    case 'sticker':
      await client.replyMessage({
        replyToken,
        messages: [{
          type: 'sticker',
          packageId: '446',
          stickerId: '1988'
        }]
      });
      break;
      
    default:
      await client.replyMessage({
        replyToken,
        messages: [{
          type: 'text',
          text: 'ขออภัย ฉันยังไม่รองรับข้อความประเภทนี้'
        }]
      });
  }
}

// Follow handler
async function handleFollow(event) {
  const { replyToken, source } = event;
  
  let displayName = 'คุณ';
  try {
    const profile = await client.getProfile(source.userId);
    displayName = profile.displayName;
  } catch (err) {
    console.error('Cannot get profile:', err);
  }
  
  await client.replyMessage({
    replyToken,
    messages: [
      {
        type: 'text',
        text: `ยินดีต้อนรับ ${displayName}! 🎉`
      },
      {
        type: 'text',
        text: 'พิมพ์อะไรก็ได้ แล้วฉันจะ Echo กลับไปให้!',
        quickReply: {
          items: [
            {
              type: 'action',
              action: {
                type: 'message',
                label: 'สวัสดี!',
                text: 'สวัสดี!'
              }
            },
            {
              type: 'action',
              action: {
                type: 'message',
                label: 'ช่วยเหลือ',
                text: 'ช่วยเหลือ'
              }
            }
          ]
        }
      }
    ]
  });
}

// Postback handler
async function handlePostback(event) {
  const { replyToken, postback } = event;
  
  await client.replyMessage({
    replyToken,
    messages: [{
      type: 'text',
      text: `Postback data: ${postback.data}`
    }]
  });
}

// Health check
app.get('/', (req, res) => {
  res.json({ status: 'LINE Bot is running!' });
});

// Start server
const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`LINE Echo Bot started on port ${PORT}`);
  console.log(`Webhook URL: http://localhost:${PORT}/webhook`);
});
```

---

## 10. Environment Variables

### .env.example

```env
# LINE Configuration
LINE_CHANNEL_ACCESS_TOKEN=your_channel_access_token_here
LINE_CHANNEL_SECRET=your_channel_secret_here

# Server Configuration
PORT=3000
NODE_ENV=development

# Webhook
WEBHOOK_PATH=/webhook

# Logging
LOG_LEVEL=INFO

# Database (ถ้ามี)
# DATABASE_URL=postgresql://user:password@localhost:5432/linebot

# Redis (ถ้ามี)
# REDIS_URL=redis://localhost:6379
```

### .env (จริง - ห้าม commit)

```env
LINE_CHANNEL_ACCESS_TOKEN=eyJhbGciOiJIUzI1NiJ9.xxxxx
LINE_CHANNEL_SECRET=a1b2c3d4e5f6789012345678901234ab
PORT=3000
NODE_ENV=development
```

### การโหลด Environment Variables

```javascript
// ต้อง require dotenv ก่อน require อื่นๆ
require('dotenv').config();

// ตรวจสอบว่าโหลดสำเร็จ
if (!process.env.LINE_CHANNEL_ACCESS_TOKEN) {
  console.error('LINE_CHANNEL_ACCESS_TOKEN is not set!');
  process.exit(1);
}
```

### การใช้ Environment Variables อย่างปลอดภัย

```javascript
// config/env.js
const requiredEnvVars = [
  'LINE_CHANNEL_ACCESS_TOKEN',
  'LINE_CHANNEL_SECRET',
];

// ตรวจสอบ required variables
const missing = requiredEnvVars.filter(key => !process.env[key]);
if (missing.length > 0) {
  console.error(`Missing required environment variables: ${missing.join(', ')}`);
  console.error('Please check your .env file');
  process.exit(1);
}

module.exports = {
  LINE_CHANNEL_ACCESS_TOKEN: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  LINE_CHANNEL_SECRET: process.env.LINE_CHANNEL_SECRET,
  PORT: parseInt(process.env.PORT || '3000', 10),
  NODE_ENV: process.env.NODE_ENV || 'development',
};
```

---

## 11. Ngrok สำหรับ Local Development

### ติดตั้งและตั้งค่า Ngrok

```bash
# ติดตั้ง ngrok
# macOS
brew install ngrok

# Windows (ใช้ winget)
winget install ngrok

# Linux
snap install ngrok

# หรือ download จาก https://ngrok.com/download
```

### สมัคร Account และ Authtoken

```bash
# 1. สมัครที่ https://ngrok.com
# 2. คัดลอก Authtoken จาก Dashboard
# 3. ตั้งค่า
ngrok authtoken YOUR_AUTH_TOKEN
```

### รัน Ngrok

```bash
# Terminal 1: รัน Bot Server
npm run dev

# Terminal 2: รัน Ngrok
ngrok http 3000
```

**Output จาก Ngrok:**
```
ngrok by @inconshreveable

Session Status   online
Account          your@email.com
Version          3.5.0
Region           Southeast Asia (ap)
Latency          28ms
Web Interface    http://127.0.0.1:4040
Forwarding       https://abc123.ngrok-free.app -> http://localhost:3000

Connections      ttl     opn     rt1     rt5     p50     p90
                 0       0       0.00    0.00    0.00    0.00
```

### ตั้งค่า Webhook URL บน LINE Console

1. คัดลอก URL จาก Ngrok: `https://abc123.ngrok-free.app`
2. ไปที่ LINE Developers Console
3. Messaging API Tab → Webhook URL
4. ใส่: `https://abc123.ngrok-free.app/webhook`
5. คลิก **Verify**

### Script สำหรับ Auto-update Webhook URL

```javascript
// scripts/update-webhook.js
require('dotenv').config();
const axios = require('axios');

async function getNgrokUrl() {
  const response = await axios.get('http://localhost:4040/api/tunnels');
  const tunnel = response.data.tunnels.find(t => t.proto === 'https');
  return tunnel?.public_url;
}

async function updateWebhookUrl(webhookUrl) {
  const response = await axios.put(
    'https://api.line.me/v2/bot/channel/webhook/endpoint',
    { webhook: webhookUrl },
    {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
        'Content-Type': 'application/json',
      }
    }
  );
  return response.data;
}

async function main() {
  try {
    const ngrokUrl = await getNgrokUrl();
    if (!ngrokUrl) {
      throw new Error('Ngrok is not running');
    }
    
    const webhookUrl = `${ngrokUrl}/webhook`;
    console.log(`Updating webhook URL to: ${webhookUrl}`);
    
    await updateWebhookUrl(webhookUrl);
    console.log('Webhook URL updated successfully!');
  } catch (error) {
    console.error('Failed to update webhook URL:', error.message);
  }
}

main();
```

```json
// เพิ่มใน package.json scripts
{
  "scripts": {
    "dev": "nodemon src/index.js",
    "ngrok": "ngrok http 3000",
    "update-webhook": "node scripts/update-webhook.js"
  }
}
```

### ngrok.yml Configuration

```yaml
# ~/.config/ngrok/ngrok.yml
version: "2"
authtoken: YOUR_AUTH_TOKEN

tunnels:
  linebot:
    proto: http
    addr: 3000
    subdomain: your-linebot  # ต้องมี paid plan
```

---

## 12. Deployment

### Deploy บน Railway

**ขั้นตอน:**

1. สร้าง account บน https://railway.app
2. ติดตั้ง Railway CLI:
   ```bash
   npm install -g @railway/cli
   ```
3. Login:
   ```bash
   railway login
   ```
4. สร้างโปรเจกต์:
   ```bash
   railway init
   ```
5. ตั้งค่า Environment Variables:
   ```bash
   railway variables set LINE_CHANNEL_ACCESS_TOKEN=your_token
   railway variables set LINE_CHANNEL_SECRET=your_secret
   ```
6. Deploy:
   ```bash
   railway up
   ```

**Procfile (สำหรับ Railway):**
```
web: node src/index.js
```

### Deploy บน Render

**render.yaml:**
```yaml
services:
  - type: web
    name: line-bot
    env: node
    plan: free
    buildCommand: npm install
    startCommand: node src/index.js
    envVars:
      - key: LINE_CHANNEL_ACCESS_TOKEN
        sync: false
      - key: LINE_CHANNEL_SECRET
        sync: false
      - key: NODE_ENV
        value: production
```

### Deploy บน Heroku

```bash
# ติดตั้ง Heroku CLI
npm install -g heroku

# Login
heroku login

# สร้าง App
heroku create my-line-bot

# ตั้งค่า Environment Variables
heroku config:set LINE_CHANNEL_ACCESS_TOKEN=your_token
heroku config:set LINE_CHANNEL_SECRET=your_secret

# Deploy
git push heroku main
```

**Procfile:**
```
web: node src/index.js
```

### Docker Deployment

**Dockerfile:**
```dockerfile
FROM node:20-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm ci --only=production

# Copy source
COPY src/ ./src/

# Expose port
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost:3000/health || exit 1

# Start
CMD ["node", "src/index.js"]
```

**docker-compose.yml:**
```yaml
version: '3.8'

services:
  linebot:
    build: .
    ports:
      - "3000:3000"
    environment:
      - LINE_CHANNEL_ACCESS_TOKEN=${LINE_CHANNEL_ACCESS_TOKEN}
      - LINE_CHANNEL_SECRET=${LINE_CHANNEL_SECRET}
      - NODE_ENV=production
    restart: unless-stopped
```

### Production Checklist

```markdown
# Production Deployment Checklist

## Security
- [ ] ไม่มี Secrets ใน Code
- [ ] ใช้ HTTPS เท่านั้น
- [ ] ตรวจสอบ Signature ทุก Request
- [ ] ตั้งค่า Rate Limiting
- [ ] ปิด Debug Logging

## Performance
- [ ] ใช้ PM2 หรือ Process Manager
- [ ] ตั้งค่า Health Check
- [ ] ตั้งค่า Auto-restart
- [ ] Monitor Memory Usage

## Monitoring
- [ ] Setup Error Tracking (Sentry)
- [ ] Setup Log Aggregation
- [ ] Setup Uptime Monitoring
- [ ] Alert on High Error Rate
```

---

## 13. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Echo Bot พื้นฐาน
สร้าง Echo Bot ที่:
- รับข้อความ Text แล้ว Echo กลับพร้อมเวลา
- รับรูปภาพแล้วตอบ "ขอบคุณสำหรับรูปภาพ"
- ส่ง Sticker กลับเมื่อได้รับ Sticker

### แบบฝึกหัดที่ 2: Command Bot
สร้าง Bot ที่รองรับคำสั่ง:
- `/help` - แสดงคำสั่งทั้งหมด
- `/time` - แสดงเวลาปัจจุบัน
- `/weather` - แสดงสภาพอากาศ (mock data)
- `/joke` - บอกมุกตลก

### แบบฝึกหัดที่ 3: Calculator Bot
สร้าง Bot คิดเลข:
- รับ Expression เช่น "2 + 2"
- คำนวณและส่งผลลัพธ์กลับ
- Handle ข้อผิดพลาด เช่น หารด้วยศูนย์

### แบบฝึกหัดที่ 4: Quick Reply Menu
สร้าง Bot ที่มีเมนู Quick Reply:
- เมนูหลัก: สินค้า, ติดต่อ, เกี่ยวกับ
- เมนูสินค้า: รายการสินค้า, ค้นหาสินค้า

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Project Setup** - สร้างโปรเจกต์ Node.js อย่างถูกต้อง
2. **@line/bot-sdk** - การใช้ LINE Official SDK
3. **Express.js** - ตั้งค่า Webhook Server
4. **Event Handling** - จัดการ Events ต่างๆ
5. **Signature Verification** - ป้องกัน Request ปลอม
6. **Message Types** - ส่งข้อความหลากหลายรูปแบบ
7. **Echo Bot** - ตัวอย่างสมบูรณ์
8. **Environment Variables** - จัดการ Config อย่างปลอดภัย
9. **Ngrok** - ทดสอบบน Local
10. **Deployment** - Deploy บน Railway/Render/Heroku

ในบทต่อไปเราจะเรียนรู้การสร้าง LINE Bot ด้วย Python!
