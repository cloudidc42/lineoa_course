# ส่วนที่ 96: WebSocket และ Real-time Communication สำหรับ LINE Bot

---

## สารบัญ

1. [WebSocket ใน LINE Bot Ecosystem](#1-websocket-ใน-line-bot-ecosystem)
2. [Real-time Chat ระหว่าง LINE Users](#2-real-time-chat-ระหว่าง-line-users)
3. [Socket.io สำหรับ Admin Dashboard](#3-socketio-สำหรับ-admin-dashboard)
4. [Real-time Order Tracking](#4-real-time-order-tracking)
5. [Live Auction System บน LINE](#5-live-auction-system-บน-line)
6. [Real-time Queue System](#6-real-time-queue-system)
7. [Push Notifications ไปยัง LIFF ผ่าน WebSocket](#7-push-notifications-ไปยัง-liff-ผ่าน-websocket)
8. [WebSocket + Redis Pub/Sub สำหรับการ Scaling](#8-websocket--redis-pubsub-สำหรับการ-scaling)
9. [WebSocket Security](#9-websocket-security)
10. [Load Balancing WebSockets](#10-load-balancing-websockets)
11. [Complete Real-time Order Tracking System](#11-complete-real-time-order-tracking-system)

---

## 1. WebSocket ใน LINE Bot Ecosystem

### 1.1 ภาพรวมของ WebSocket

WebSocket เป็นโปรโตคอลการสื่อสารแบบ full-duplex ที่ทำงานบน TCP connection เดียว ต่างจาก HTTP ที่เป็น request-response แบบ one-way WebSocket ช่วยให้ server และ client สามารถส่งข้อมูลหากันได้ตลอดเวลาโดยไม่ต้องรอ request

```
HTTP Traditional:
Client ──[Request]──> Server
Client <──[Response]── Server
(ต้องส่ง request ทุกครั้งที่ต้องการข้อมูลใหม่)

WebSocket:
Client <──[Handshake]──> Server
Client <══[Persistent Connection]══> Server
(ส่งข้อมูลได้สองทางตลอดเวลา)
```

### 1.2 บทบาทของ WebSocket ใน LINE Bot

```
LINE Platform Architecture with WebSocket:

┌─────────────────────────────────────────────────────────────┐
│                         LINE Platform                        │
│                                                             │
│  ┌──────────────┐    Webhook    ┌─────────────────────┐    │
│  │  LINE Users  │ ─────────────>│   Your Bot Server   │    │
│  │              │               │                     │    │
│  │  LINE App    │ <─────────────│   WebSocket Server  │    │
│  │  (LIFF)      │  Push/Reply   │                     │    │
│  └──────────────┘               └─────────┬───────────┘    │
│                                           │                 │
│                               ┌───────────▼───────────┐    │
│                               │    Admin Dashboard    │    │
│                               │   (WebSocket Client)  │    │
│                               └───────────────────────┘    │
└─────────────────────────────────────────────────────────────┘
```

### 1.3 Use Cases หลักของ WebSocket ใน LINE Bot

| Use Case | ประโยชน์ | ความซับซ้อน |
|----------|---------|------------|
| Admin Dashboard | ดูข้อความ real-time | ต่ำ |
| Order Tracking | ติดตามสถานะคำสั่งซื้อ | ปานกลาง |
| Live Auction | ประมูลแบบ real-time | สูง |
| Queue System | คิวรอแบบ real-time | ปานกลาง |
| Live Chat Support | แชทกับ agent | ปานกลาง |
| Real-time Analytics | สถิติแบบ live | สูง |

### 1.4 การติดตั้งพื้นฐาน

```bash
# สร้าง project
mkdir line-websocket-system
cd line-websocket-system
npm init -y

# ติดตั้ง dependencies หลัก
npm install @line/bot-sdk express socket.io redis ioredis
npm install dotenv cors helmet express-rate-limit
npm install jsonwebtoken bcryptjs
npm install --save-dev nodemon jest

# โครงสร้าง project
mkdir -p src/{controllers,services,models,middleware,utils}
mkdir -p src/{websocket,redis,config}
```

```javascript
// src/config/index.js
module.exports = {
  line: {
    channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
    channelSecret: process.env.LINE_CHANNEL_SECRET,
  },
  server: {
    port: process.env.PORT || 3000,
    nodeEnv: process.env.NODE_ENV || 'development',
  },
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: process.env.REDIS_PORT || 6379,
    password: process.env.REDIS_PASSWORD,
  },
  jwt: {
    secret: process.env.JWT_SECRET || 'your-secret-key',
    expiresIn: '24h',
  },
  websocket: {
    cors: {
      origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
      methods: ['GET', 'POST'],
    },
    pingTimeout: 60000,
    pingInterval: 25000,
  },
};
```

---

## 2. Real-time Chat ระหว่าง LINE Users

### 2.1 สถาปัตยกรรมระบบ Chat

```
Real-time Chat Architecture:

LINE User A                Bot Server              LINE User B
    │                          │                        │
    │──[Send Message]──────────>│                       │
    │                          │──[Store in DB]         │
    │                          │──[Emit to Socket]──────>│ (Admin)
    │                          │──[Push to LINE B]──────>│
    │                          │                        │
    │<──[Delivery Receipt]──────│                       │
    │                          │                        │
```

### 2.2 Chat Service Implementation

```javascript
// src/services/chatService.js
const { Client } = require('@line/bot-sdk');
const config = require('../config');
const redisClient = require('../redis/client');

class ChatService {
  constructor(io) {
    this.io = io;
    this.lineClient = new Client({
      channelAccessToken: config.line.channelAccessToken,
    });
    this.activeSessions = new Map();
  }

  // บันทึกข้อความใน Redis
  async saveMessage(roomId, message) {
    const key = `chat:${roomId}:messages`;
    const messageStr = JSON.stringify({
      ...message,
      timestamp: Date.now(),
    });
    
    await redisClient.rpush(key, messageStr);
    await redisClient.expire(key, 86400 * 7); // เก็บ 7 วัน
    
    // เก็บ metadata ของ room
    await redisClient.hset(`chat:${roomId}:meta`, {
      lastMessage: message.text || '[media]',
      lastTimestamp: Date.now(),
      messageCount: await redisClient.llen(key),
    });
  }

  // ดึงประวัติการสนทนา
  async getChatHistory(roomId, limit = 50) {
    const key = `chat:${roomId}:messages`;
    const messages = await redisClient.lrange(key, -limit, -1);
    return messages.map(m => JSON.parse(m));
  }

  // ส่งข้อความไปยัง LINE user
  async sendToLineUser(userId, message) {
    try {
      await this.lineClient.pushMessage(userId, message);
      return { success: true };
    } catch (error) {
      console.error('Error sending to LINE:', error);
      throw error;
    }
  }

  // Broadcast ไปยัง admin dashboard
  broadcastToAdmins(event, data) {
    this.io.to('admin-room').emit(event, data);
  }

  // จัดการ incoming message จาก LINE
  async handleIncomingMessage(event) {
    const { source, message, timestamp } = event;
    const userId = source.userId;
    const roomId = source.groupId || source.roomId || userId;

    // สร้าง message object
    const messageData = {
      id: `${userId}-${timestamp}`,
      roomId,
      userId,
      direction: 'incoming',
      type: message.type,
      text: message.text,
      timestamp,
    };

    // บันทึกข้อความ
    await this.saveMessage(roomId, messageData);

    // Broadcast ไปยัง admin dashboard
    this.broadcastToAdmins('new-message', messageData);

    // Publish ไปยัง Redis channel สำหรับ multi-instance
    await redisClient.publish('chat:new-message', JSON.stringify(messageData));

    return messageData;
  }

  // Agent ตอบกลับผ่าน dashboard
  async agentReply(roomId, userId, text, agentId) {
    const message = { type: 'text', text };
    
    // ส่งไปยัง LINE
    await this.sendToLineUser(userId, message);

    // บันทึกข้อความตอบกลับ
    const replyData = {
      id: `agent-${Date.now()}`,
      roomId,
      agentId,
      direction: 'outgoing',
      type: 'text',
      text,
      timestamp: Date.now(),
    };

    await this.saveMessage(roomId, replyData);

    // Broadcast การตอบกลับ
    this.broadcastToAdmins('message-sent', replyData);

    return replyData;
  }
}

module.exports = ChatService;
```

### 2.3 WebSocket Server Setup

```javascript
// src/websocket/server.js
const { Server } = require('socket.io');
const jwt = require('jsonwebtoken');
const config = require('../config');
const ChatService = require('../services/chatService');

function setupWebSocket(httpServer) {
  const io = new Server(httpServer, {
    cors: config.websocket.cors,
    pingTimeout: config.websocket.pingTimeout,
    pingInterval: config.websocket.pingInterval,
  });

  // Authentication Middleware
  io.use((socket, next) => {
    const token = socket.handshake.auth.token;
    if (!token) {
      return next(new Error('Authentication required'));
    }

    try {
      const decoded = jwt.verify(token, config.jwt.secret);
      socket.userId = decoded.userId;
      socket.role = decoded.role;
      next();
    } catch (err) {
      next(new Error('Invalid token'));
    }
  });

  const chatService = new ChatService(io);

  // Connection handler
  io.on('connection', (socket) => {
    console.log(`Client connected: ${socket.id}, role: ${socket.role}`);

    // Admin joins admin room
    if (socket.role === 'admin' || socket.role === 'agent') {
      socket.join('admin-room');
      socket.emit('joined-admin-room', { status: 'connected' });
    }

    // Admin requests chat history
    socket.on('get-chat-history', async ({ roomId, limit }) => {
      try {
        const history = await chatService.getChatHistory(roomId, limit);
        socket.emit('chat-history', { roomId, messages: history });
      } catch (error) {
        socket.emit('error', { message: error.message });
      }
    });

    // Agent sends reply
    socket.on('agent-reply', async ({ roomId, userId, text }) => {
      try {
        if (socket.role !== 'admin' && socket.role !== 'agent') {
          return socket.emit('error', { message: 'Unauthorized' });
        }
        const result = await chatService.agentReply(
          roomId, userId, text, socket.userId
        );
        socket.emit('reply-sent', result);
      } catch (error) {
        socket.emit('error', { message: error.message });
      }
    });

    // Admin joins specific chat room
    socket.on('join-chat-room', ({ roomId }) => {
      socket.join(`chat-${roomId}`);
      socket.emit('joined-chat-room', { roomId });
    });

    // Typing indicator
    socket.on('agent-typing', ({ roomId, isTyping }) => {
      socket.to(`chat-${roomId}`).emit('agent-typing', {
        agentId: socket.userId,
        isTyping,
      });
    });

    socket.on('disconnect', () => {
      console.log(`Client disconnected: ${socket.id}`);
    });
  });

  return { io, chatService };
}

module.exports = setupWebSocket;
```

### 2.4 LINE Webhook Integration

```javascript
// src/controllers/webhookController.js
const line = require('@line/bot-sdk');
const config = require('../config');

class WebhookController {
  constructor(chatService) {
    this.chatService = chatService;
    this.lineClient = new line.Client({
      channelAccessToken: config.line.channelAccessToken,
    });
  }

  async handleWebhook(req, res) {
    const events = req.body.events;
    
    try {
      await Promise.all(events.map(event => this.processEvent(event)));
      res.json({ status: 'ok' });
    } catch (error) {
      console.error('Webhook error:', error);
      res.status(500).json({ error: error.message });
    }
  }

  async processEvent(event) {
    switch (event.type) {
      case 'message':
        await this.handleMessage(event);
        break;
      case 'follow':
        await this.handleFollow(event);
        break;
      case 'unfollow':
        await this.handleUnfollow(event);
        break;
      case 'join':
        await this.handleJoin(event);
        break;
      default:
        console.log(`Unhandled event type: ${event.type}`);
    }
  }

  async handleMessage(event) {
    // บันทึกและ broadcast ข้อความ
    const messageData = await this.chatService.handleIncomingMessage(event);
    
    // ตอบกลับอัตโนมัติถ้าไม่มี agent online
    const hasOnlineAgent = await this.checkOnlineAgents(messageData.roomId);
    
    if (!hasOnlineAgent && event.message.type === 'text') {
      await this.lineClient.replyMessage(event.replyToken, {
        type: 'text',
        text: 'ขอบคุณที่ติดต่อมา เจ้าหน้าที่จะตอบกลับโดยเร็วที่สุด 🙏',
      });
    }
  }

  async handleFollow(event) {
    const userId = event.source.userId;
    
    // บันทึก follower ใหม่
    await this.chatService.broadcastToAdmins('new-follower', { userId });
    
    // ส่งข้อความต้อนรับ
    await this.lineClient.replyMessage(event.replyToken, {
      type: 'text',
      text: 'ยินดีต้อนรับ! 🎉 ขอบคุณที่ติดตามเรา',
    });
  }

  async handleUnfollow(event) {
    await this.chatService.broadcastToAdmins('user-unfollowed', {
      userId: event.source.userId,
    });
  }

  async handleJoin(event) {
    await this.lineClient.replyMessage(event.replyToken, {
      type: 'text',
      text: 'ขอบคุณที่เพิ่มเข้ากลุ่ม! 🙏',
    });
  }

  async checkOnlineAgents(roomId) {
    // ตรวจสอบว่ามี agent online อยู่หรือไม่
    const agents = await redisClient.smembers(`room:${roomId}:agents`);
    return agents.length > 0;
  }
}

module.exports = WebhookController;
```

---

## 3. Socket.io สำหรับ Admin Dashboard

### 3.1 Admin Dashboard Architecture

```
Admin Dashboard Layout:

┌─────────────────────────────────────────────────────────────┐
│  LINE OA Admin Dashboard                    [Online: 3 ●]  │
├──────────────────┬──────────────────────────────────────────┤
│  Active Chats    │  Chat Window                             │
│ ─────────────── │ ─────────────────────────────────────── │
│ 👤 สมชาย (3)   │  Room: สมชาย_12345                       │
│ 👤 สมหญิง (1)  │                                           │
│ 👤 สมศรี (5)   │  [User]: สวัสดีครับ                      │
│                  │  [Agent]: สวัสดีค่ะ มีอะไรให้ช่วย?      │
│  Unassigned: 2  │                                           │
│                  │  [พิมพ์ข้อความ...]         [ส่ง]         │
├──────────────────┴──────────────────────────────────────────┤
│  Live Stats: Messages Today: 234 | Active Users: 45        │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Admin Dashboard Server

```javascript
// src/websocket/adminDashboard.js
const Redis = require('ioredis');
const config = require('../config');

class AdminDashboard {
  constructor(io) {
    this.io = io;
    this.redis = new Redis(config.redis);
    this.stats = {
      messagesToday: 0,
      activeUsers: new Set(),
      activeSessions: new Map(),
    };
    
    this.setupRedisSubscription();
    this.startStatsUpdater();
  }

  setupRedisSubscription() {
    const subscriber = new Redis(config.redis);
    
    subscriber.subscribe('chat:new-message', 'order:updated', 'user:event');
    
    subscriber.on('message', (channel, message) => {
      const data = JSON.parse(message);
      
      switch (channel) {
        case 'chat:new-message':
          this.handleNewMessage(data);
          break;
        case 'order:updated':
          this.handleOrderUpdate(data);
          break;
        case 'user:event':
          this.handleUserEvent(data);
          break;
      }
    });
  }

  handleNewMessage(data) {
    this.stats.messagesToday++;
    this.stats.activeUsers.add(data.userId);
    
    // Broadcast ไปยัง admin room
    this.io.to('admin-room').emit('dashboard:new-message', {
      ...data,
      stats: this.getStats(),
    });
  }

  handleOrderUpdate(data) {
    this.io.to('admin-room').emit('dashboard:order-updated', data);
  }

  handleUserEvent(data) {
    if (data.type === 'follow') {
      this.io.to('admin-room').emit('dashboard:new-follower', data);
    }
  }

  getStats() {
    return {
      messagesToday: this.stats.messagesToday,
      activeUsers: this.stats.activeUsers.size,
      activeSessions: this.stats.activeSessions.size,
      timestamp: Date.now(),
    };
  }

  startStatsUpdater() {
    setInterval(() => {
      this.io.to('admin-room').emit('dashboard:stats-update', this.getStats());
    }, 5000); // อัพเดตทุก 5 วินาที
  }

  // ดึง live stats จาก Redis
  async getLiveStats() {
    const pipeline = this.redis.pipeline();
    pipeline.get('stats:messages:today');
    pipeline.get('stats:active:users');
    pipeline.get('stats:orders:pending');
    pipeline.get('stats:revenue:today');
    
    const results = await pipeline.exec();
    
    return {
      messagesToday: parseInt(results[0][1] || 0),
      activeUsers: parseInt(results[1][1] || 0),
      pendingOrders: parseInt(results[2][1] || 0),
      revenueToday: parseFloat(results[3][1] || 0),
    };
  }
}

module.exports = AdminDashboard;
```

### 3.3 Admin Dashboard Frontend (HTML + Socket.io Client)

```html
<!-- public/admin/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>LINE OA Admin Dashboard</title>
  <script src="/socket.io/socket.io.js"></script>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Sarabun', sans-serif; background: #f0f2f5; }
    
    .dashboard {
      display: grid;
      grid-template-columns: 300px 1fr;
      grid-template-rows: 60px 1fr 50px;
      height: 100vh;
    }
    
    .header {
      grid-column: 1 / -1;
      background: #00B900;
      color: white;
      display: flex;
      align-items: center;
      padding: 0 20px;
      justify-content: space-between;
    }
    
    .sidebar {
      background: white;
      border-right: 1px solid #e0e0e0;
      overflow-y: auto;
    }
    
    .chat-area {
      display: flex;
      flex-direction: column;
      background: #f0f2f5;
    }
    
    .chat-messages {
      flex: 1;
      overflow-y: auto;
      padding: 20px;
    }
    
    .message {
      margin-bottom: 10px;
      max-width: 70%;
    }
    
    .message.incoming {
      align-self: flex-start;
    }
    
    .message.outgoing {
      align-self: flex-end;
      margin-left: auto;
    }
    
    .message .bubble {
      padding: 10px 15px;
      border-radius: 15px;
      display: inline-block;
    }
    
    .message.incoming .bubble {
      background: white;
    }
    
    .message.outgoing .bubble {
      background: #00B900;
      color: white;
    }
    
    .chat-input {
      padding: 15px;
      background: white;
      display: flex;
      gap: 10px;
      border-top: 1px solid #e0e0e0;
    }
    
    .chat-input input {
      flex: 1;
      padding: 10px;
      border: 1px solid #ddd;
      border-radius: 20px;
      outline: none;
    }
    
    .chat-input button {
      background: #00B900;
      color: white;
      border: none;
      padding: 10px 20px;
      border-radius: 20px;
      cursor: pointer;
    }
    
    .stats-bar {
      grid-column: 1 / -1;
      background: #333;
      color: white;
      display: flex;
      align-items: center;
      padding: 0 20px;
      gap: 30px;
      font-size: 13px;
    }
    
    .chat-room-item {
      padding: 12px 15px;
      border-bottom: 1px solid #f0f0f0;
      cursor: pointer;
      display: flex;
      align-items: center;
      gap: 10px;
    }
    
    .chat-room-item:hover, .chat-room-item.active {
      background: #f0fff0;
    }
    
    .unread-badge {
      background: #ff4444;
      color: white;
      border-radius: 50%;
      width: 20px;
      height: 20px;
      display: flex;
      align-items: center;
      justify-content: center;
      font-size: 11px;
      margin-left: auto;
    }
    
    .status-indicator {
      width: 10px;
      height: 10px;
      border-radius: 50%;
      background: #00B900;
    }
  </style>
</head>
<body>
  <div class="dashboard">
    <header class="header">
      <h1>LINE OA Admin Dashboard</h1>
      <div id="connection-status">
        <span class="status-indicator"></span>
        <span id="online-count">Online: 0</span>
      </div>
    </header>
    
    <aside class="sidebar">
      <div style="padding: 10px; background: #f8f9fa; border-bottom: 1px solid #ddd;">
        <strong>Active Chats</strong>
      </div>
      <div id="chat-rooms"></div>
    </aside>
    
    <main class="chat-area">
      <div class="chat-messages" id="chat-messages">
        <div style="text-align: center; color: #999; margin-top: 50px;">
          เลือก Chat Room เพื่อเริ่มสนทนา
        </div>
      </div>
      <div class="chat-input">
        <input type="text" id="message-input" placeholder="พิมพ์ข้อความ..." />
        <button onclick="sendMessage()">ส่ง</button>
      </div>
    </main>
    
    <div class="stats-bar">
      <span>ข้อความวันนี้: <strong id="stat-messages">0</strong></span>
      <span>ผู้ใช้ active: <strong id="stat-users">0</strong></span>
      <span>คำสั่งซื้อรอ: <strong id="stat-orders">0</strong></span>
      <span>รายได้วันนี้: ฿<strong id="stat-revenue">0</strong></span>
      <span style="margin-left: auto">อัพเดต: <span id="last-update">-</span></span>
    </div>
  </div>

  <script>
    // เชื่อมต่อ Socket.io
    const token = localStorage.getItem('admin_token');
    const socket = io({
      auth: { token }
    });

    let currentRoom = null;
    const rooms = new Map();

    // Connection events
    socket.on('connect', () => {
      console.log('Connected to server');
    });

    socket.on('connect_error', (err) => {
      console.error('Connection error:', err.message);
      // Redirect to login if auth error
      if (err.message === 'Authentication required' || err.message === 'Invalid token') {
        window.location.href = '/admin/login';
      }
    });

    // Receive new messages
    socket.on('new-message', (data) => {
      updateRoomList(data);
      if (currentRoom === data.roomId) {
        appendMessage(data);
      }
      updateStats(data.stats);
    });

    // Stats update
    socket.on('dashboard:stats-update', (stats) => {
      updateStats(stats);
    });

    // Message sent confirmation
    socket.on('message-sent', (data) => {
      appendMessage(data);
    });

    // Chat history
    socket.on('chat-history', ({ roomId, messages }) => {
      const container = document.getElementById('chat-messages');
      container.innerHTML = '';
      messages.forEach(msg => appendMessage(msg));
    });

    function updateRoomList(messageData) {
      let room = rooms.get(messageData.roomId) || {
        id: messageData.roomId,
        userId: messageData.userId,
        unread: 0,
        lastMessage: '',
      };
      
      room.unread++;
      room.lastMessage = messageData.text || '[media]';
      rooms.set(messageData.roomId, room);
      
      renderRoomList();
    }

    function renderRoomList() {
      const container = document.getElementById('chat-rooms');
      container.innerHTML = '';
      
      rooms.forEach((room, roomId) => {
        const div = document.createElement('div');
        div.className = `chat-room-item${currentRoom === roomId ? ' active' : ''}`;
        div.innerHTML = `
          <div>👤</div>
          <div style="flex: 1">
            <div style="font-weight: bold; font-size: 13px">${room.userId?.substring(0, 10)}...</div>
            <div style="font-size: 12px; color: #666">${room.lastMessage}</div>
          </div>
          ${room.unread > 0 ? `<div class="unread-badge">${room.unread}</div>` : ''}
        `;
        div.onclick = () => openChatRoom(roomId, room.userId);
        container.appendChild(div);
      });
    }

    function openChatRoom(roomId, userId) {
      currentRoom = roomId;
      
      // Reset unread
      const room = rooms.get(roomId);
      if (room) {
        room.unread = 0;
        rooms.set(roomId, room);
      }
      
      renderRoomList();
      
      // Join room via socket
      socket.emit('join-chat-room', { roomId });
      
      // Request history
      socket.emit('get-chat-history', { roomId, limit: 50 });
    }

    function appendMessage(data) {
      const container = document.getElementById('chat-messages');
      const div = document.createElement('div');
      div.className = `message ${data.direction}`;
      div.innerHTML = `
        <div class="bubble">
          ${data.text || '[media]'}
          <div style="font-size: 10px; opacity: 0.7; margin-top: 4px">
            ${new Date(data.timestamp).toLocaleTimeString('th-TH')}
          </div>
        </div>
      `;
      container.appendChild(div);
      container.scrollTop = container.scrollHeight;
    }

    function sendMessage() {
      if (!currentRoom) {
        alert('กรุณาเลือก Chat Room ก่อน');
        return;
      }
      
      const input = document.getElementById('message-input');
      const text = input.value.trim();
      if (!text) return;
      
      const room = rooms.get(currentRoom);
      socket.emit('agent-reply', {
        roomId: currentRoom,
        userId: room?.userId,
        text,
      });
      
      input.value = '';
    }

    function updateStats(stats) {
      if (!stats) return;
      document.getElementById('stat-messages').textContent = stats.messagesToday || 0;
      document.getElementById('stat-users').textContent = stats.activeUsers || 0;
      document.getElementById('stat-orders').textContent = stats.pendingOrders || 0;
      document.getElementById('stat-revenue').textContent = 
        (stats.revenueToday || 0).toLocaleString('th-TH');
      document.getElementById('last-update').textContent = 
        new Date().toLocaleTimeString('th-TH');
    }

    // Enter key to send
    document.getElementById('message-input').addEventListener('keypress', (e) => {
      if (e.key === 'Enter') sendMessage();
    });
  </script>
</body>
</html>
```

---

## 4. Real-time Order Tracking

### 4.1 Order Status Flow

```
Order Status Flow:

[สร้างคำสั่งซื้อ] ──> [รอยืนยัน] ──> [กำลังเตรียม] ──> [กำลังจัดส่ง] ──> [ส่งแล้ว]
       │                  │                  │                  │               │
       ▼                  ▼                  ▼                  ▼               ▼
  WebSocket            WebSocket          WebSocket          WebSocket      WebSocket
  + Push Msg          + Push Msg         + Push Msg         + Push Msg     + Push Msg
  ไปยัง User          ไปยัง User         ไปยัง User         ไปยัง User     ไปยัง User
```

### 4.2 Order Tracking Service

```javascript
// src/services/orderTrackingService.js
const Redis = require('ioredis');
const { Client } = require('@line/bot-sdk');
const config = require('../config');

class OrderTrackingService {
  constructor(io) {
    this.io = io;
    this.redis = new Redis(config.redis);
    this.lineClient = new Client({
      channelAccessToken: config.line.channelAccessToken,
    });
    
    this.statuses = {
      CREATED: { label: 'รับคำสั่งซื้อแล้ว', emoji: '📝', color: '#FFA500' },
      CONFIRMED: { label: 'ยืนยันคำสั่งซื้อ', emoji: '✅', color: '#00B900' },
      PREPARING: { label: 'กำลังเตรียมสินค้า', emoji: '🔧', color: '#2196F3' },
      SHIPPING: { label: 'กำลังจัดส่ง', emoji: '🚚', color: '#9C27B0' },
      DELIVERED: { label: 'ส่งสำเร็จ', emoji: '🎉', color: '#00B900' },
      CANCELLED: { label: 'ยกเลิกแล้ว', emoji: '❌', color: '#F44336' },
    };
  }

  // สร้าง order ใหม่
  async createOrder(orderData) {
    const orderId = `ORD-${Date.now()}-${Math.random().toString(36).substr(2, 5).toUpperCase()}`;
    
    const order = {
      id: orderId,
      ...orderData,
      status: 'CREATED',
      createdAt: Date.now(),
      updatedAt: Date.now(),
      statusHistory: [{
        status: 'CREATED',
        timestamp: Date.now(),
        note: 'รับคำสั่งซื้อเรียบร้อยแล้ว',
      }],
    };

    // บันทึกใน Redis
    await this.redis.set(
      `order:${orderId}`,
      JSON.stringify(order),
      'EX',
      86400 * 30 // เก็บ 30 วัน
    );

    // เพิ่มใน user's orders list
    await this.redis.lpush(`user:${orderData.userId}:orders`, orderId);
    
    // ลิงค์ order กับ room ใน Redis
    await this.redis.set(`order:${orderId}:room`, orderData.userId);

    // แจ้งเตือน
    await this.notifyOrderUpdate(order);
    
    return order;
  }

  // อัพเดทสถานะ order
  async updateOrderStatus(orderId, newStatus, note = '', metadata = {}) {
    const orderKey = `order:${orderId}`;
    const orderStr = await this.redis.get(orderKey);
    
    if (!orderStr) {
      throw new Error(`Order ${orderId} not found`);
    }

    const order = JSON.parse(orderStr);
    const oldStatus = order.status;
    
    // อัพเดทข้อมูล
    order.status = newStatus;
    order.updatedAt = Date.now();
    order.statusHistory.push({
      status: newStatus,
      timestamp: Date.now(),
      note,
      metadata,
    });

    // ถ้าเป็นการจัดส่ง บันทึกข้อมูล tracking
    if (newStatus === 'SHIPPING' && metadata.trackingNumber) {
      order.tracking = {
        number: metadata.trackingNumber,
        provider: metadata.provider || 'ไทยไปรษณีย์',
        url: metadata.trackingUrl,
      };
    }

    // บันทึกกลับ
    await this.redis.set(orderKey, JSON.stringify(order), 'EX', 86400 * 30);

    // Publish event
    await this.redis.publish('order:updated', JSON.stringify({
      orderId,
      oldStatus,
      newStatus,
      order,
    }));

    // แจ้งเตือน
    await this.notifyOrderUpdate(order, note);

    return order;
  }

  // ส่งการแจ้งเตือนผ่าน WebSocket และ LINE
  async notifyOrderUpdate(order, note = '') {
    const statusInfo = this.statuses[order.status];
    
    // Broadcast ผ่าน WebSocket ไปยัง LIFF หรือ web client
    this.io.to(`order:${order.id}`).emit('order:status-updated', {
      orderId: order.id,
      status: order.status,
      statusLabel: statusInfo.label,
      statusEmoji: statusInfo.emoji,
      note,
      timestamp: Date.now(),
    });

    // Broadcast ไปยัง admin dashboard
    this.io.to('admin-room').emit('admin:order-updated', {
      order,
      statusInfo,
    });

    // ส่ง LINE push message
    await this.sendLineOrderUpdate(order, statusInfo, note);
  }

  // ส่ง LINE message แจ้งสถานะ
  async sendLineOrderUpdate(order, statusInfo, note) {
    const messages = [
      {
        type: 'flex',
        altText: `คำสั่งซื้อ ${order.id}: ${statusInfo.label}`,
        contents: {
          type: 'bubble',
          header: {
            type: 'box',
            layout: 'vertical',
            backgroundColor: statusInfo.color,
            contents: [
              {
                type: 'text',
                text: `${statusInfo.emoji} ${statusInfo.label}`,
                color: '#FFFFFF',
                weight: 'bold',
                size: 'lg',
              },
            ],
          },
          body: {
            type: 'box',
            layout: 'vertical',
            contents: [
              {
                type: 'text',
                text: `คำสั่งซื้อ #${order.id}`,
                weight: 'bold',
                size: 'md',
              },
              {
                type: 'separator',
                margin: 'md',
              },
              {
                type: 'box',
                layout: 'vertical',
                margin: 'md',
                contents: order.items?.map(item => ({
                  type: 'box',
                  layout: 'horizontal',
                  contents: [
                    { type: 'text', text: item.name, flex: 3, size: 'sm' },
                    { type: 'text', text: `x${item.quantity}`, flex: 1, size: 'sm', align: 'center' },
                    { type: 'text', text: `฿${item.price}`, flex: 2, size: 'sm', align: 'end' },
                  ],
                })) || [],
              },
              ...(note ? [{
                type: 'text',
                text: `หมายเหตุ: ${note}`,
                size: 'sm',
                color: '#888888',
                margin: 'md',
                wrap: true,
              }] : []),
              ...(order.tracking ? [{
                type: 'box',
                layout: 'vertical',
                margin: 'md',
                backgroundColor: '#f0f8ff',
                cornerRadius: 'sm',
                paddingAll: 'sm',
                contents: [
                  {
                    type: 'text',
                    text: '📦 ข้อมูลการจัดส่ง',
                    weight: 'bold',
                    size: 'sm',
                  },
                  {
                    type: 'text',
                    text: `หมายเลขพัสดุ: ${order.tracking.number}`,
                    size: 'sm',
                  },
                  {
                    type: 'text',
                    text: `บริษัทขนส่ง: ${order.tracking.provider}`,
                    size: 'sm',
                  },
                ],
              }] : []),
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
                  label: '📱 ดูรายละเอียดเพิ่มเติม',
                  uri: `${process.env.LIFF_URL}/orders/${order.id}`,
                },
                style: 'primary',
                color: '#00B900',
              },
            ],
          },
        },
      },
    ];

    try {
      await this.lineClient.pushMessage(order.userId, messages);
    } catch (error) {
      console.error('Error sending LINE message:', error);
    }
  }

  // ดึงข้อมูล order
  async getOrder(orderId) {
    const orderStr = await this.redis.get(`order:${orderId}`);
    if (!orderStr) return null;
    return JSON.parse(orderStr);
  }

  // ดึงรายการ orders ของ user
  async getUserOrders(userId, limit = 10) {
    const orderIds = await this.redis.lrange(`user:${userId}:orders`, 0, limit - 1);
    const orders = await Promise.all(
      orderIds.map(id => this.getOrder(id))
    );
    return orders.filter(Boolean);
  }
}

module.exports = OrderTrackingService;
```

### 4.3 LIFF Order Tracking Interface

```javascript
// public/liff/order-tracking.js
const liff = window.liff;
let socket;
let currentOrderId;

async function initLIFF() {
  await liff.init({ liffId: process.env.LIFF_ID });
  
  if (!liff.isLoggedIn()) {
    liff.login();
    return;
  }

  const profile = await liff.getProfile();
  const token = await getAuthToken(profile.userId);
  
  // เชื่อมต่อ WebSocket
  socket = io(process.env.SERVER_URL, {
    auth: { token },
    transports: ['websocket'],
  });

  socket.on('connect', () => {
    console.log('Connected to real-time server');
    if (currentOrderId) {
      socket.emit('join-order-room', { orderId: currentOrderId });
    }
  });

  socket.on('order:status-updated', (data) => {
    updateOrderUI(data);
    showStatusNotification(data);
  });

  // โหลด URL params
  const urlParams = new URLSearchParams(window.location.search);
  currentOrderId = urlParams.get('orderId');
  
  if (currentOrderId) {
    loadOrderDetails(currentOrderId);
  }
}

async function loadOrderDetails(orderId) {
  try {
    const response = await fetch(`/api/orders/${orderId}`, {
      headers: { Authorization: `Bearer ${localStorage.getItem('token')}` },
    });
    const order = await response.json();
    renderOrderDetails(order);
    
    // Subscribe ไปยัง order room
    if (socket.connected) {
      socket.emit('join-order-room', { orderId });
    }
  } catch (error) {
    console.error('Error loading order:', error);
  }
}

function renderOrderDetails(order) {
  const statuses = ['CREATED', 'CONFIRMED', 'PREPARING', 'SHIPPING', 'DELIVERED'];
  const currentIndex = statuses.indexOf(order.status);
  
  const statusLabels = {
    CREATED: 'รับออเดอร์',
    CONFIRMED: 'ยืนยันแล้ว',
    PREPARING: 'กำลังเตรียม',
    SHIPPING: 'กำลังจัดส่ง',
    DELIVERED: 'ส่งแล้ว',
  };

  document.getElementById('order-id').textContent = `#${order.id}`;
  
  // Progress bar
  const progressHTML = statuses.map((status, index) => `
    <div class="status-step ${index <= currentIndex ? 'completed' : ''} ${index === currentIndex ? 'current' : ''}">
      <div class="step-circle">${index <= currentIndex ? '✓' : index + 1}</div>
      <div class="step-label">${statusLabels[status]}</div>
    </div>
  `).join('<div class="step-line"></div>');
  
  document.getElementById('status-progress').innerHTML = progressHTML;
  
  // Items
  const itemsHTML = order.items?.map(item => `
    <div class="order-item">
      <img src="${item.imageUrl}" alt="${item.name}" />
      <div class="item-details">
        <div class="item-name">${item.name}</div>
        <div class="item-price">฿${item.price.toLocaleString()} x ${item.quantity}</div>
      </div>
      <div class="item-total">฿${(item.price * item.quantity).toLocaleString()}</div>
    </div>
  `).join('');
  
  document.getElementById('order-items').innerHTML = itemsHTML;
  
  // Tracking info
  if (order.tracking) {
    document.getElementById('tracking-section').style.display = 'block';
    document.getElementById('tracking-number').textContent = order.tracking.number;
    document.getElementById('tracking-provider').textContent = order.tracking.provider;
  }
}

function updateOrderUI(data) {
  // อัพเดทสถานะแบบ real-time
  document.getElementById('current-status').textContent = 
    `${data.statusEmoji} ${data.statusLabel}`;
  
  // อัพเดท progress
  loadOrderDetails(data.orderId);
}

function showStatusNotification(data) {
  // แสดง notification ใน LIFF
  const notification = document.createElement('div');
  notification.className = 'status-notification';
  notification.innerHTML = `
    <div class="notification-content">
      <span class="notification-emoji">${data.statusEmoji}</span>
      <div>
        <strong>${data.statusLabel}</strong>
        <p>${data.note || ''}</p>
      </div>
    </div>
  `;
  
  document.body.appendChild(notification);
  setTimeout(() => notification.remove(), 5000);
}

initLIFF();
```

---

## 5. Live Auction System บน LINE

### 5.1 Auction System Architecture

```
Live Auction Flow:

Admin Creates Auction ──> LINE Bot Broadcasts ──> Users Place Bids
         │                        │                       │
         ▼                        ▼                       ▼
   WebSocket Room          LIFF Auction Page        WebSocket Events
   "auction-{id}"          (Real-time updates)      bid:placed
                                                    bid:outbid
                                                    auction:ended
```

### 5.2 Auction Service

```javascript
// src/services/auctionService.js
class AuctionService {
  constructor(io, redis, lineClient) {
    this.io = io;
    this.redis = redis;
    this.lineClient = lineClient;
    this.activeAuctions = new Map();
  }

  async createAuction(auctionData) {
    const auctionId = `AUCTION-${Date.now()}`;
    
    const auction = {
      id: auctionId,
      title: auctionData.title,
      description: auctionData.description,
      imageUrl: auctionData.imageUrl,
      startingPrice: auctionData.startingPrice,
      currentPrice: auctionData.startingPrice,
      minimumBidIncrement: auctionData.minimumBidIncrement || 100,
      startTime: auctionData.startTime || Date.now(),
      endTime: auctionData.endTime,
      status: 'PENDING', // PENDING, ACTIVE, ENDED
      bids: [],
      winnerId: null,
      winnerName: null,
    };

    await this.redis.set(
      `auction:${auctionId}`,
      JSON.stringify(auction),
      'EX',
      86400 * 7
    );

    // กำหนด timer สำหรับการเริ่มและจบ auction
    this.scheduleAuctionTimer(auction);

    return auction;
  }

  scheduleAuctionTimer(auction) {
    const now = Date.now();
    const startDelay = auction.startTime - now;
    const endDelay = auction.endTime - now;

    if (startDelay > 0) {
      setTimeout(() => this.startAuction(auction.id), startDelay);
    } else {
      this.startAuction(auction.id);
    }

    if (endDelay > 0) {
      setTimeout(() => this.endAuction(auction.id), endDelay);
    }
  }

  async startAuction(auctionId) {
    const auction = await this.getAuction(auctionId);
    if (!auction) return;

    auction.status = 'ACTIVE';
    auction.startTime = Date.now();
    await this.saveAuction(auction);

    // Broadcast ไปยัง LINE users
    await this.broadcastAuctionStart(auction);

    // Emit ไปยัง WebSocket clients
    this.io.emit('auction:started', {
      auctionId,
      title: auction.title,
      startingPrice: auction.startingPrice,
      endTime: auction.endTime,
    });

    // เริ่ม countdown timer
    this.startCountdown(auctionId, auction.endTime);
  }

  startCountdown(auctionId, endTime) {
    const interval = setInterval(async () => {
      const remaining = endTime - Date.now();
      
      if (remaining <= 0) {
        clearInterval(interval);
        return;
      }

      // Emit countdown ทุก 1 วินาที
      this.io.to(`auction:${auctionId}`).emit('auction:countdown', {
        auctionId,
        remaining: Math.max(0, remaining),
        formattedTime: this.formatTime(remaining),
      });

      // แจ้งเตือนพิเศษเมื่อเหลือ 5 นาที, 1 นาที, 30 วินาที
      if (remaining <= 30000 && remaining > 29000) {
        this.io.to(`auction:${auctionId}`).emit('auction:warning', {
          message: '⚠️ เหลือเวลาอีก 30 วินาที!',
        });
      } else if (remaining <= 60000 && remaining > 59000) {
        this.io.to(`auction:${auctionId}`).emit('auction:warning', {
          message: '⚠️ เหลือเวลาอีก 1 นาที!',
        });
      }
    }, 1000);
  }

  async placeBid(auctionId, userId, userName, amount) {
    const auction = await this.getAuction(auctionId);
    
    if (!auction) throw new Error('ไม่พบการประมูล');
    if (auction.status !== 'ACTIVE') throw new Error('การประมูลไม่ได้เปิดอยู่');
    if (Date.now() > auction.endTime) throw new Error('การประมูลสิ้นสุดแล้ว');
    
    const minimumBid = auction.currentPrice + auction.minimumBidIncrement;
    if (amount < minimumBid) {
      throw new Error(`ราคาประมูลขั้นต่ำต้องเป็น ฿${minimumBid.toLocaleString()}`);
    }

    const previousBidder = auction.bids.length > 0
      ? auction.bids[auction.bids.length - 1]
      : null;

    // บันทึก bid
    const bid = {
      userId,
      userName,
      amount,
      timestamp: Date.now(),
    };

    auction.bids.push(bid);
    auction.currentPrice = amount;
    auction.leadingBidderId = userId;
    auction.leadingBidderName = userName;

    await this.saveAuction(auction);

    // Emit bid update
    this.io.to(`auction:${auctionId}`).emit('auction:bid-placed', {
      auctionId,
      bid,
      currentPrice: amount,
      totalBids: auction.bids.length,
    });

    // แจ้งผู้ที่ถูก outbid
    if (previousBidder && previousBidder.userId !== userId) {
      await this.notifyOutbid(previousBidder.userId, auction);
    }

    return { success: true, currentPrice: amount };
  }

  async endAuction(auctionId) {
    const auction = await this.getAuction(auctionId);
    if (!auction || auction.status === 'ENDED') return;

    auction.status = 'ENDED';
    auction.endedAt = Date.now();

    if (auction.bids.length > 0) {
      const winningBid = auction.bids[auction.bids.length - 1];
      auction.winnerId = winningBid.userId;
      auction.winnerName = winningBid.userName;
      auction.finalPrice = winningBid.amount;
    }

    await this.saveAuction(auction);

    // Emit auction ended
    this.io.to(`auction:${auctionId}`).emit('auction:ended', {
      auctionId,
      winner: auction.winnerId ? {
        userId: auction.winnerId,
        name: auction.winnerName,
        amount: auction.finalPrice,
      } : null,
    });

    // แจ้งเตือน LINE users
    await this.broadcastAuctionResult(auction);
  }

  async broadcastAuctionStart(auction) {
    const message = {
      type: 'flex',
      altText: `🔨 การประมูล: ${auction.title} เริ่มแล้ว!`,
      contents: {
        type: 'bubble',
        hero: {
          type: 'image',
          url: auction.imageUrl,
          size: 'full',
          aspectRatio: '20:13',
          aspectMode: 'cover',
        },
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: '🔨 การประมูลเริ่มแล้ว!',
              weight: 'bold',
              size: 'xl',
              color: '#FF4444',
            },
            {
              type: 'text',
              text: auction.title,
              weight: 'bold',
              size: 'lg',
              margin: 'md',
              wrap: true,
            },
            {
              type: 'text',
              text: auction.description,
              size: 'sm',
              color: '#888888',
              wrap: true,
              margin: 'sm',
            },
            {
              type: 'separator',
              margin: 'md',
            },
            {
              type: 'box',
              layout: 'horizontal',
              margin: 'md',
              contents: [
                {
                  type: 'box',
                  layout: 'vertical',
                  contents: [
                    { type: 'text', text: 'ราคาเริ่มต้น', size: 'xs', color: '#888888' },
                    {
                      type: 'text',
                      text: `฿${auction.startingPrice.toLocaleString()}`,
                      weight: 'bold',
                      size: 'lg',
                      color: '#FF4444',
                    },
                  ],
                },
                {
                  type: 'box',
                  layout: 'vertical',
                  contents: [
                    { type: 'text', text: 'สิ้นสุดใน', size: 'xs', color: '#888888' },
                    {
                      type: 'text',
                      text: this.formatRemainingTime(auction.endTime),
                      weight: 'bold',
                      size: 'sm',
                    },
                  ],
                },
              ],
            },
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
                label: '🔨 เข้าร่วมการประมูล',
                uri: `${process.env.LIFF_URL}/auction/${auction.id}`,
              },
              style: 'primary',
              color: '#FF4444',
            },
          ],
        },
      },
    };

    // Broadcast ไปยังทุก follower
    // ในระบบจริงควรใช้ Multicast กับรายชื่อ followers
    await this.lineClient.broadcast(message);
  }

  async broadcastAuctionResult(auction) {
    if (!auction.winnerId) {
      // ไม่มีผู้ชนะ
      await this.lineClient.broadcast({
        type: 'text',
        text: `การประมูล "${auction.title}" สิ้นสุดแล้ว โดยไม่มีผู้เข้าร่วม`,
      });
      return;
    }

    // แจ้งผู้ชนะ
    await this.lineClient.pushMessage(auction.winnerId, {
      type: 'flex',
      altText: '🎉 ยินดีด้วย! คุณชนะการประมูล',
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: '🎉 ยินดีด้วย!',
              weight: 'bold',
              size: 'xxl',
              align: 'center',
            },
            {
              type: 'text',
              text: 'คุณชนะการประมูล',
              size: 'lg',
              align: 'center',
              margin: 'sm',
            },
            {
              type: 'text',
              text: auction.title,
              weight: 'bold',
              align: 'center',
              margin: 'md',
              wrap: true,
            },
            {
              type: 'text',
              text: `ราคาสุดท้าย: ฿${auction.finalPrice.toLocaleString()}`,
              size: 'xl',
              color: '#FF4444',
              weight: 'bold',
              align: 'center',
              margin: 'md',
            },
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
                label: '💳 ดำเนินการชำระเงิน',
                uri: `${process.env.LIFF_URL}/auction/${auction.id}/payment`,
              },
              style: 'primary',
              color: '#00B900',
            },
          ],
        },
      },
    });

    // Broadcast ผลการประมูลไปยังทุกคน
    await this.lineClient.broadcast({
      type: 'text',
      text: `🔨 การประมูล "${auction.title}" สิ้นสุดแล้ว\n` +
            `ผู้ชนะ: ${auction.winnerName}\n` +
            `ราคาสุดท้าย: ฿${auction.finalPrice.toLocaleString()}`,
    });
  }

  async notifyOutbid(userId, auction) {
    await this.lineClient.pushMessage(userId, {
      type: 'text',
      text: `⚠️ มีผู้ประมูลสูงกว่าคุณในรายการ "${auction.title}"\n` +
            `ราคาปัจจุบัน: ฿${auction.currentPrice.toLocaleString()}\n` +
            `กลับมาประมูลได้เลย! 🔨`,
    });
  }

  async getAuction(auctionId) {
    const data = await this.redis.get(`auction:${auctionId}`);
    return data ? JSON.parse(data) : null;
  }

  async saveAuction(auction) {
    await this.redis.set(
      `auction:${auction.id}`,
      JSON.stringify(auction),
      'EX',
      86400 * 7
    );
  }

  formatTime(ms) {
    const seconds = Math.floor(ms / 1000) % 60;
    const minutes = Math.floor(ms / 60000) % 60;
    const hours = Math.floor(ms / 3600000);
    
    if (hours > 0) return `${hours}:${String(minutes).padStart(2, '0')}:${String(seconds).padStart(2, '0')}`;
    return `${minutes}:${String(seconds).padStart(2, '0')}`;
  }

  formatRemainingTime(endTime) {
    const remaining = endTime - Date.now();
    if (remaining <= 0) return 'สิ้นสุดแล้ว';
    
    const hours = Math.floor(remaining / 3600000);
    const minutes = Math.floor((remaining % 3600000) / 60000);
    
    if (hours > 0) return `${hours} ชั่วโมง ${minutes} นาที`;
    return `${minutes} นาที`;
  }
}

module.exports = AuctionService;
```

---

## 6. Real-time Queue System

### 6.1 Queue System Design

```
Queue System Flow:

User ──[เข้าคิว]──> Bot ──[ออกตั๋วคิว]──> User ได้หมายเลข
                                │
                                ▼
                    ┌─────────────────────┐
                    │    Queue Service    │
                    │  Q001, Q002, Q003   │
                    └──────────┬──────────┘
                               │ WebSocket
                               ▼
                    ┌─────────────────────┐
                    │   Staff Interface   │──[เรียก Q001]──> LINE Push
                    └─────────────────────┘
```

### 6.2 Queue Service Implementation

```javascript
// src/services/queueService.js
class QueueService {
  constructor(io, redis, lineClient) {
    this.io = io;
    this.redis = redis;
    this.lineClient = lineClient;
  }

  async joinQueue(userId, serviceType = 'general') {
    const queueKey = `queue:${serviceType}`;
    const counterKey = `queue:${serviceType}:counter`;
    
    // ตรวจสอบว่า user อยู่ในคิวแล้วหรือไม่
    const existingTicket = await this.redis.hget(`queue:${serviceType}:users`, userId);
    if (existingTicket) {
      const position = await this.getQueuePosition(serviceType, existingTicket);
      return {
        status: 'already_in_queue',
        ticketNumber: existingTicket,
        position,
        estimatedWait: position * 5, // 5 นาทีต่อคิว
      };
    }

    // ออกหมายเลขคิว
    const ticketNumber = await this.redis.incr(counterKey);
    const ticketId = `Q${String(ticketNumber).padStart(3, '0')}`;
    
    // เพิ่มเข้าคิว
    await this.redis.rpush(queueKey, JSON.stringify({
      ticketId,
      userId,
      joinedAt: Date.now(),
      serviceType,
    }));

    // Map userId -> ticketId
    await this.redis.hset(`queue:${serviceType}:users`, userId, ticketId);

    const position = await this.redis.llen(queueKey);
    
    // แจ้งเตือน admin
    this.io.to('admin-room').emit('queue:new-customer', {
      ticketId,
      userId,
      serviceType,
      position,
      totalInQueue: position,
    });

    // ส่ง LINE message ยืนยัน
    await this.sendQueueConfirmation(userId, ticketId, position, serviceType);

    return {
      status: 'queued',
      ticketId,
      position,
      estimatedWait: position * 5,
    };
  }

  async callNextCustomer(serviceType, staffId) {
    const queueKey = `queue:${serviceType}`;
    
    const customerStr = await this.redis.lpop(queueKey);
    if (!customerStr) {
      return { status: 'empty', message: 'ไม่มีลูกค้าในคิว' };
    }

    const customer = JSON.parse(customerStr);
    
    // ลบออกจาก user map
    await this.redis.hdel(`queue:${serviceType}:users`, customer.userId);

    // บันทึก serving info
    await this.redis.hset(`queue:${serviceType}:serving`, {
      ticketId: customer.ticketId,
      userId: customer.userId,
      staffId,
      calledAt: Date.now(),
    });

    // คำนวณเวลารอ
    const waitTime = Date.now() - customer.joinedAt;

    // อัพเดท queue ที่เหลือ
    await this.updateQueuePositions(serviceType);

    // แจ้ง WebSocket
    this.io.to('admin-room').emit('queue:customer-called', {
      ticketId: customer.ticketId,
      staffId,
      waitTime,
    });

    // ส่ง LINE push แจ้งลูกค้า
    await this.notifyCustomerCalled(customer.userId, customer.ticketId, staffId);

    return {
      status: 'called',
      customer,
      waitTime,
    };
  }

  async updateQueuePositions(serviceType) {
    const queueKey = `queue:${serviceType}`;
    const queue = await this.redis.lrange(queueKey, 0, -1);
    
    queue.forEach(async (item, index) => {
      const customer = JSON.parse(item);
      const position = index + 1;
      const estimatedWait = position * 5;
      
      // แจ้ง user ผ่าน WebSocket (ถ้า user มี LIFF เปิดอยู่)
      this.io.to(`user:${customer.userId}`).emit('queue:position-updated', {
        ticketId: customer.ticketId,
        position,
        estimatedWait,
      });
    });
  }

  async sendQueueConfirmation(userId, ticketId, position, serviceType) {
    await this.lineClient.pushMessage(userId, {
      type: 'flex',
      altText: `คิวหมายเลข ${ticketId}`,
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: '🎫 ตั๋วคิวของคุณ',
              weight: 'bold',
              size: 'lg',
              align: 'center',
            },
            {
              type: 'text',
              text: ticketId,
              weight: 'bold',
              size: 'xxl',
              align: 'center',
              color: '#00B900',
              margin: 'md',
            },
            {
              type: 'separator',
              margin: 'md',
            },
            {
              type: 'box',
              layout: 'horizontal',
              margin: 'md',
              contents: [
                {
                  type: 'box',
                  layout: 'vertical',
                  contents: [
                    { type: 'text', text: 'ลำดับที่', size: 'sm', color: '#888' },
                    { type: 'text', text: `${position}`, weight: 'bold', size: 'xl' },
                  ],
                  alignItems: 'center',
                },
                {
                  type: 'separator',
                },
                {
                  type: 'box',
                  layout: 'vertical',
                  contents: [
                    { type: 'text', text: 'รอประมาณ', size: 'sm', color: '#888' },
                    { type: 'text', text: `${position * 5} นาที`, weight: 'bold', size: 'xl' },
                  ],
                  alignItems: 'center',
                },
              ],
            },
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
                label: '📱 ติดตามคิว Real-time',
                uri: `${process.env.LIFF_URL}/queue/${ticketId}`,
              },
              style: 'primary',
              color: '#00B900',
            },
            {
              type: 'button',
              action: {
                type: 'message',
                label: '❌ ยกเลิกคิว',
                text: `ยกเลิกคิว ${ticketId}`,
              },
              style: 'secondary',
              margin: 'sm',
            },
          ],
        },
      },
    });
  }

  async notifyCustomerCalled(userId, ticketId, staffId) {
    await this.lineClient.pushMessage(userId, {
      type: 'text',
      text: `🔔 ถึงคิวของคุณแล้ว!\nหมายเลขคิว: ${ticketId}\nกรุณามาที่เคาน์เตอร์ได้เลยค่ะ`,
    });
  }

  async getQueueStatus(serviceType) {
    const queueKey = `queue:${serviceType}`;
    const queue = await this.redis.lrange(queueKey, 0, -1);
    
    return {
      serviceType,
      totalInQueue: queue.length,
      customers: queue.map((item, index) => {
        const customer = JSON.parse(item);
        return {
          ticketId: customer.ticketId,
          position: index + 1,
          waitTime: Date.now() - customer.joinedAt,
          estimatedWait: (index + 1) * 5,
        };
      }),
    };
  }

  async getQueuePosition(serviceType, ticketId) {
    const queue = await this.redis.lrange(`queue:${serviceType}`, 0, -1);
    const index = queue.findIndex(item => JSON.parse(item).ticketId === ticketId);
    return index >= 0 ? index + 1 : -1;
  }
}

module.exports = QueueService;
```

---

## 7. Push Notifications ไปยัง LIFF ผ่าน WebSocket

### 7.1 LIFF WebSocket Integration

```javascript
// public/liff/websocket-client.js
class LIFFWebSocketClient {
  constructor(serverUrl, liffId) {
    this.serverUrl = serverUrl;
    this.liffId = liffId;
    this.socket = null;
    this.userId = null;
    this.handlers = new Map();
  }

  async initialize() {
    // Init LIFF
    await liff.init({ liffId: this.liffId });
    
    if (!liff.isLoggedIn()) {
      liff.login({ redirectUri: window.location.href });
      return;
    }

    const profile = await liff.getProfile();
    this.userId = profile.userId;

    // รับ token จาก server
    const token = await this.getAuthToken(profile.userId);
    
    // เชื่อมต่อ WebSocket
    this.socket = io(this.serverUrl, {
      auth: { token },
      transports: ['websocket', 'polling'],
      reconnection: true,
      reconnectionAttempts: 5,
      reconnectionDelay: 1000,
    });

    this.setupEventListeners();
  }

  setupEventListeners() {
    this.socket.on('connect', () => {
      console.log('LIFF WebSocket connected');
      this.emit('connected', { userId: this.userId });
      
      // Join user-specific room
      this.socket.emit('join-user-room', { userId: this.userId });
    });

    this.socket.on('disconnect', (reason) => {
      console.log('Disconnected:', reason);
      if (reason === 'io server disconnect') {
        // Server forcibly disconnected, reconnect manually
        this.socket.connect();
      }
    });

    this.socket.on('reconnect', () => {
      console.log('Reconnected');
      this.socket.emit('join-user-room', { userId: this.userId });
    });

    // Order updates
    this.socket.on('order:status-updated', (data) => {
      this.handleOrderUpdate(data);
    });

    // Queue updates
    this.socket.on('queue:position-updated', (data) => {
      this.handleQueueUpdate(data);
    });

    // Custom notifications
    this.socket.on('notification', (data) => {
      this.showNotification(data);
    });

    // Point updates
    this.socket.on('points:updated', (data) => {
      this.handlePointsUpdate(data);
    });
  }

  handleOrderUpdate(data) {
    // อัพเดท UI
    this.triggerHandler('order-update', data);
    
    // แสดง toast notification
    this.showToast(`สถานะคำสั่งซื้อ: ${data.statusLabel}`, 'success');
    
    // Vibrate ถ้าอยู่บน mobile
    if (navigator.vibrate) {
      navigator.vibrate([200, 100, 200]);
    }
  }

  handleQueueUpdate(data) {
    this.triggerHandler('queue-update', data);
    
    if (data.position === 1) {
      this.showToast('🔔 คุณอยู่ถัดไป! เตรียมตัวได้เลย', 'warning');
      if (navigator.vibrate) {
        navigator.vibrate([500, 200, 500, 200, 500]);
      }
    } else {
      this.showToast(`ลำดับคิวของคุณ: ${data.position}`, 'info');
    }
  }

  handlePointsUpdate(data) {
    this.triggerHandler('points-update', data);
    
    if (data.earned > 0) {
      this.showToast(`🎉 ได้รับ ${data.earned} คะแนน! ยอดรวม: ${data.total}`, 'success');
    }
  }

  showToast(message, type = 'info') {
    const toast = document.createElement('div');
    toast.className = `liff-toast liff-toast-${type}`;
    toast.textContent = message;
    toast.style.cssText = `
      position: fixed;
      top: 20px;
      left: 50%;
      transform: translateX(-50%);
      background: ${type === 'success' ? '#00B900' : type === 'warning' ? '#FFA500' : '#2196F3'};
      color: white;
      padding: 12px 20px;
      border-radius: 8px;
      z-index: 9999;
      font-size: 14px;
      box-shadow: 0 4px 12px rgba(0,0,0,0.3);
      max-width: 90%;
      text-align: center;
      transition: opacity 0.3s;
    `;
    
    document.body.appendChild(toast);
    
    setTimeout(() => {
      toast.style.opacity = '0';
      setTimeout(() => toast.remove(), 300);
    }, 4000);
  }

  showNotification(data) {
    if (Notification.permission === 'granted') {
      new Notification(data.title, {
        body: data.body,
        icon: data.icon || '/icons/line-oa.png',
      });
    }
    
    this.showToast(data.body, data.type || 'info');
  }

  on(event, handler) {
    if (!this.handlers.has(event)) {
      this.handlers.set(event, []);
    }
    this.handlers.get(event).push(handler);
  }

  triggerHandler(event, data) {
    const handlers = this.handlers.get(event) || [];
    handlers.forEach(h => h(data));
  }

  async getAuthToken(userId) {
    const response = await fetch('/api/auth/liff-token', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ userId }),
    });
    const data = await response.json();
    return data.token;
  }

  disconnect() {
    if (this.socket) {
      this.socket.disconnect();
    }
  }
}

// ใช้งาน
const wsClient = new LIFFWebSocketClient(
  process.env.SERVER_URL,
  process.env.LIFF_ID
);

wsClient.on('order-update', (data) => {
  // อัพเดท order UI ในหน้า
  document.getElementById('order-status').textContent = data.statusLabel;
});

wsClient.on('queue-update', (data) => {
  document.getElementById('queue-position').textContent = data.position;
  document.getElementById('est-wait').textContent = `${data.estimatedWait} นาที`;
});

wsClient.initialize();
```

---

## 8. WebSocket + Redis Pub/Sub สำหรับการ Scaling

### 8.1 Multi-Instance Architecture

```
Scaling Architecture:

Load Balancer (Nginx)
        │
   ┌────┴─────┐
   │          │
Server 1    Server 2    Server 3
   │          │          │
   └────┬─────┘──────────┘
        │
   Redis Pub/Sub
   (Message Broker)
        │
   ┌────┴──────┐
   │           │
Subscriber  Subscriber
(all servers listen)
```

### 8.2 Redis Adapter สำหรับ Socket.io

```javascript
// src/redis/socketAdapter.js
const { createAdapter } = require('@socket.io/redis-adapter');
const { createClient } = require('redis');
const config = require('../config');

async function setupRedisAdapter(io) {
  const pubClient = createClient({
    socket: {
      host: config.redis.host,
      port: config.redis.port,
    },
    password: config.redis.password,
  });

  const subClient = pubClient.duplicate();

  await Promise.all([pubClient.connect(), subClient.connect()]);

  io.adapter(createAdapter(pubClient, subClient));
  
  console.log('Redis adapter configured for Socket.io');
  
  return { pubClient, subClient };
}

module.exports = setupRedisAdapter;
```

### 8.3 Redis Pub/Sub Manager

```javascript
// src/redis/pubSubManager.js
const Redis = require('ioredis');
const config = require('../config');

class PubSubManager {
  constructor() {
    this.publisher = new Redis(config.redis);
    this.subscriber = new Redis(config.redis);
    this.handlers = new Map();
    
    this.setupSubscriber();
  }

  setupSubscriber() {
    this.subscriber.on('message', (channel, message) => {
      const handlers = this.handlers.get(channel) || [];
      const data = JSON.parse(message);
      handlers.forEach(handler => handler(data));
    });

    this.subscriber.on('pmessage', (pattern, channel, message) => {
      const handlers = this.handlers.get(pattern) || [];
      const data = JSON.parse(message);
      handlers.forEach(handler => handler({ channel, data }));
    });
  }

  async subscribe(channel, handler) {
    if (!this.handlers.has(channel)) {
      this.handlers.set(channel, []);
      await this.subscriber.subscribe(channel);
    }
    this.handlers.get(channel).push(handler);
  }

  async psubscribe(pattern, handler) {
    if (!this.handlers.has(pattern)) {
      this.handlers.set(pattern, []);
      await this.subscriber.psubscribe(pattern);
    }
    this.handlers.get(pattern).push(handler);
  }

  async publish(channel, data) {
    await this.publisher.publish(channel, JSON.stringify(data));
  }

  async unsubscribe(channel) {
    await this.subscriber.unsubscribe(channel);
    this.handlers.delete(channel);
  }

  // ส่ง message ไปยัง specific user ทุก server
  async sendToUser(userId, event, data) {
    await this.publish(`user:${userId}`, { event, data });
  }

  // ส่ง message ไปยัง order room ทุก server
  async sendToOrder(orderId, event, data) {
    await this.publish(`order:${orderId}`, { event, data });
  }

  // ส่ง message ไปยัง admin room ทุก server
  async sendToAdmins(event, data) {
    await this.publish('admin:broadcast', { event, data });
  }
}

// Singleton
const pubSubManager = new PubSubManager();

module.exports = pubSubManager;
```

### 8.4 Integration ใน Main Server

```javascript
// src/server.js
const express = require('express');
const http = require('http');
const { Server } = require('socket.io');
const line = require('@line/bot-sdk');
const config = require('./config');
const setupWebSocket = require('./websocket/server');
const setupRedisAdapter = require('./redis/socketAdapter');
const pubSubManager = require('./redis/pubSubManager');

const app = express();
const httpServer = http.createServer(app);
const io = new Server(httpServer, {
  cors: config.websocket.cors,
});

// Setup Redis adapter for scaling
setupRedisAdapter(io);

// Setup WebSocket handlers
const { chatService, orderTrackingService } = setupWebSocket(io);

// Subscribe to Redis channels for cross-server communication
pubSubManager.subscribe('admin:broadcast', ({ event, data }) => {
  io.to('admin-room').emit(event, data);
});

pubSubManager.psubscribe('user:*', ({ channel, data }) => {
  const userId = channel.replace('user:', '');
  io.to(`user:${userId}`).emit(data.event, data.data);
});

pubSubManager.psubscribe('order:*', ({ channel, data }) => {
  const orderId = channel.replace('order:', '');
  io.to(`order:${orderId}`).emit(data.event, data.data);
});

// LINE Webhook
app.post('/webhook', line.middleware({
  channelSecret: config.line.channelSecret,
}), (req, res) => {
  const controller = new WebhookController(chatService);
  controller.handleWebhook(req, res);
});

// API Routes
app.use('/api', require('./routes/api'));

const PORT = config.server.port;
httpServer.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

## 9. WebSocket Security

### 9.1 Authentication & Authorization

```javascript
// src/middleware/websocketAuth.js
const jwt = require('jsonwebtoken');
const Redis = require('ioredis');
const config = require('../config');
const redis = new Redis(config.redis);

// Middleware สำหรับ Socket.io
const socketAuthMiddleware = async (socket, next) => {
  try {
    const token = socket.handshake.auth.token || 
                  socket.handshake.headers.authorization?.replace('Bearer ', '');
    
    if (!token) {
      throw new Error('No token provided');
    }

    // ตรวจสอบ blacklisted tokens
    const isBlacklisted = await redis.get(`blacklist:token:${token}`);
    if (isBlacklisted) {
      throw new Error('Token has been revoked');
    }

    // Verify JWT
    const decoded = jwt.verify(token, config.jwt.secret);
    
    // ตรวจสอบว่า user ยังมีสิทธิ์อยู่
    const userSession = await redis.get(`session:${decoded.userId}`);
    if (!userSession) {
      throw new Error('Session expired');
    }

    socket.userId = decoded.userId;
    socket.role = decoded.role;
    socket.permissions = decoded.permissions || [];
    
    next();
  } catch (err) {
    next(new Error(`Authentication failed: ${err.message}`));
  }
};

// Rate limiting สำหรับ WebSocket events
const socketRateLimiter = (maxEvents = 100, windowMs = 60000) => {
  return (socket, next) => {
    const key = `ratelimit:socket:${socket.id}`;
    
    socket.use(async ([event, ...args], next) => {
      try {
        const count = await redis.incr(key);
        
        if (count === 1) {
          await redis.expire(key, Math.ceil(windowMs / 1000));
        }
        
        if (count > maxEvents) {
          socket.emit('error', {
            code: 'RATE_LIMIT_EXCEEDED',
            message: 'Too many events. Please slow down.',
          });
          return;
        }
        
        next();
      } catch (err) {
        next(err);
      }
    });
    
    next();
  };
};

// Input validation middleware
const socketInputValidator = (socket, next) => {
  socket.use(([event, data, callback], next) => {
    // Sanitize input data
    if (data && typeof data === 'object') {
      // ลบ dangerous fields
      const dangerous = ['__proto__', 'constructor', 'prototype'];
      dangerous.forEach(key => delete data[key]);
      
      // Limit string length
      Object.keys(data).forEach(key => {
        if (typeof data[key] === 'string' && data[key].length > 10000) {
          data[key] = data[key].substring(0, 10000);
        }
      });
    }
    
    next();
  });
  
  next();
};

module.exports = {
  socketAuthMiddleware,
  socketRateLimiter,
  socketInputValidator,
};
```

### 9.2 Security Best Practices

```javascript
// src/websocket/security.js
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');

// CORS configuration
const corsOptions = {
  origin: (origin, callback) => {
    const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') || [];
    
    if (!origin || allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error(`Origin ${origin} not allowed by CORS`));
    }
  },
  methods: ['GET', 'POST'],
  credentials: true,
};

// Namespace-based access control
function setupSecureNamespaces(io) {
  // Admin namespace - เฉพาะ admin/agent
  const adminNs = io.of('/admin');
  adminNs.use(async (socket, next) => {
    if (!['admin', 'agent', 'supervisor'].includes(socket.role)) {
      return next(new Error('Access denied: Admin only'));
    }
    next();
  });

  // Customer namespace - เฉพาะ authenticated users
  const customerNs = io.of('/customer');
  customerNs.use(async (socket, next) => {
    if (!socket.userId) {
      return next(new Error('Access denied: Login required'));
    }
    next();
  });

  // Public namespace - ทุกคนเข้าได้ แต่จำกัด events
  const publicNs = io.of('/public');
  publicNs.use((socket, next) => {
    // Rate limit สูงกว่า authenticated users
    next();
  });

  return { adminNs, customerNs, publicNs };
}

// Event validation schemas
const eventSchemas = {
  'agent-reply': {
    required: ['roomId', 'userId', 'text'],
    types: { roomId: 'string', userId: 'string', text: 'string' },
    maxLength: { text: 5000 },
  },
  'join-order-room': {
    required: ['orderId'],
    types: { orderId: 'string' },
  },
  'place-bid': {
    required: ['auctionId', 'amount'],
    types: { auctionId: 'string', amount: 'number' },
    min: { amount: 1 },
    max: { amount: 100000000 },
  },
};

function validateEventData(event, data) {
  const schema = eventSchemas[event];
  if (!schema) return { valid: true };

  for (const field of schema.required || []) {
    if (data[field] === undefined || data[field] === null) {
      return { valid: false, error: `Missing required field: ${field}` };
    }
  }

  for (const [field, type] of Object.entries(schema.types || {})) {
    if (data[field] !== undefined && typeof data[field] !== type) {
      return { valid: false, error: `Invalid type for ${field}: expected ${type}` };
    }
  }

  for (const [field, maxLen] of Object.entries(schema.maxLength || {})) {
    if (data[field] && data[field].length > maxLen) {
      return { valid: false, error: `${field} exceeds maximum length of ${maxLen}` };
    }
  }

  for (const [field, min] of Object.entries(schema.min || {})) {
    if (data[field] !== undefined && data[field] < min) {
      return { valid: false, error: `${field} must be at least ${min}` };
    }
  }

  return { valid: true };
}

module.exports = {
  corsOptions,
  setupSecureNamespaces,
  validateEventData,
};
```

---

## 10. Load Balancing WebSockets

### 10.1 Nginx Configuration

```nginx
# /etc/nginx/nginx.conf
upstream websocket_servers {
    # Sticky sessions based on IP
    ip_hash;
    
    server ws-server-1:3000;
    server ws-server-2:3000;
    server ws-server-3:3000;
    
    keepalive 64;
}

server {
    listen 80;
    listen 443 ssl;
    server_name api.yourlinebot.com;
    
    ssl_certificate /etc/ssl/certs/cert.pem;
    ssl_certificate_key /etc/ssl/private/key.pem;
    
    # WebSocket upgrade
    location /socket.io/ {
        proxy_pass http://websocket_servers;
        proxy_http_version 1.1;
        
        # WebSocket headers
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # Timeouts
        proxy_read_timeout 86400s;
        proxy_send_timeout 86400s;
        proxy_connect_timeout 60s;
        
        # Buffer settings
        proxy_buffering off;
        proxy_buffer_size 128k;
        proxy_buffers 4 256k;
        proxy_busy_buffers_size 256k;
    }
    
    # Regular API
    location /api/ {
        proxy_pass http://websocket_servers;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
    
    # LINE Webhook
    location /webhook {
        proxy_pass http://websocket_servers;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 10.2 Health Check & Monitoring

```javascript
// src/health/websocketHealth.js
class WebSocketHealthMonitor {
  constructor(io, redis) {
    this.io = io;
    this.redis = redis;
    this.metrics = {
      connections: 0,
      messagesPerSecond: 0,
      errors: 0,
      rooms: 0,
    };
    
    this.startMetricsCollection();
  }

  startMetricsCollection() {
    // อัพเดท metrics ทุก 30 วินาที
    setInterval(async () => {
      await this.collectMetrics();
    }, 30000);
  }

  async collectMetrics() {
    const sockets = await this.io.fetchSockets();
    
    this.metrics = {
      connections: sockets.length,
      rooms: this.io.sockets.adapter.rooms.size,
      timestamp: Date.now(),
      serverId: process.env.SERVER_ID || 'unknown',
    };

    // บันทึกใน Redis สำหรับ monitoring
    await this.redis.hset(
      `server:metrics:${process.env.SERVER_ID || 'default'}`,
      this.metrics
    );

    // Set expiry
    await this.redis.expire(
      `server:metrics:${process.env.SERVER_ID || 'default'}`,
      120
    );
  }

  async getClusterMetrics() {
    const keys = await this.redis.keys('server:metrics:*');
    const metrics = await Promise.all(
      keys.map(async key => {
        const data = await this.redis.hgetall(key);
        return { server: key.replace('server:metrics:', ''), ...data };
      })
    );
    
    return {
      servers: metrics,
      totalConnections: metrics.reduce((sum, s) => sum + parseInt(s.connections || 0), 0),
      totalRooms: metrics.reduce((sum, s) => sum + parseInt(s.rooms || 0), 0),
    };
  }

  getHealthStatus() {
    return {
      status: 'healthy',
      connections: this.metrics.connections,
      uptime: process.uptime(),
      memory: process.memoryUsage(),
    };
  }
}

module.exports = WebSocketHealthMonitor;
```

---

## 11. Complete Real-time Order Tracking System

### 11.1 สถาปัตยกรรมระบบสมบูรณ์

```
Complete System Architecture:

┌─────────────────────────────────────────────────────────────┐
│                    LINE Platform                            │
│                                                             │
│   LINE Users ──[webhook]──> Bot Server ──[push]──> Users   │
│       │                         │                          │
│    LIFF App                WebSocket                       │
│       │                    Server                          │
│       └──[ws connect]──────────┘                           │
└─────────────────────────────────────────────────────────────┘
                               │
                    ┌──────────┴──────────┐
                    │                     │
             Redis Cluster          PostgreSQL
             (Real-time)            (Persistent)
                    │                     │
             ┌──────┴──────┐       ┌─────┴──────┐
             │             │       │            │
           Pub/Sub      Cache    Orders       Users
           (Events)   (Sessions) (Data)       (Data)
```

### 11.2 Complete Server Setup

```javascript
// src/app.js - Main Application
const express = require('express');
const http = require('http');
const path = require('path');
const cors = require('cors');
const helmet = require('helmet');
const rateLimit = require('express-rate-limit');
const line = require('@line/bot-sdk');
const { Server } = require('socket.io');

const config = require('./config');
const setupWebSocket = require('./websocket/server');
const setupRedisAdapter = require('./redis/socketAdapter');
const pubSubManager = require('./redis/pubSubManager');
const OrderTrackingService = require('./services/orderTrackingService');
const ChatService = require('./services/chatService');
const AuctionService = require('./services/auctionService');
const QueueService = require('./services/queueService');
const WebhookController = require('./controllers/webhookController');
const { socketAuthMiddleware, socketRateLimiter } = require('./middleware/websocketAuth');

async function createApp() {
  const app = express();
  const httpServer = http.createServer(app);

  // Security middleware
  app.use(helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", 'https://static.line-scdn.net'],
        connectSrc: ["'self'", 'wss:'],
      },
    },
  }));

  app.use(cors({
    origin: process.env.ALLOWED_ORIGINS?.split(','),
    credentials: true,
  }));

  // Rate limiting
  app.use('/api', rateLimit({
    windowMs: 15 * 60 * 1000,
    max: 1000,
    message: 'Too many requests, please try again later.',
  }));

  app.use(express.json({ limit: '1mb' }));
  app.use(express.static(path.join(__dirname, '../public')));

  // Setup Socket.io
  const io = new Server(httpServer, {
    cors: config.websocket.cors,
    adapter: await setupRedisAdapter(),
  });

  // Apply Socket.io middleware
  io.use(socketAuthMiddleware);
  io.use(socketRateLimiter(200, 60000));

  // Initialize services
  const lineClient = new line.Client({
    channelAccessToken: config.line.channelAccessToken,
  });
  const redis = require('./redis/client');

  const chatService = new ChatService(io);
  const orderTrackingService = new OrderTrackingService(io);
  const auctionService = new AuctionService(io, redis, lineClient);
  const queueService = new QueueService(io, redis, lineClient);

  // Setup Socket.io handlers
  io.on('connection', (socket) => {
    console.log(`Client connected: ${socket.id}, user: ${socket.userId}`);

    // Join user room
    socket.join(`user:${socket.userId}`);

    if (['admin', 'agent'].includes(socket.role)) {
      socket.join('admin-room');
    }

    // Order tracking
    socket.on('join-order-room', async ({ orderId }) => {
      // ตรวจสอบสิทธิ์
      const order = await orderTrackingService.getOrder(orderId);
      if (order && (order.userId === socket.userId || socket.role === 'admin')) {
        socket.join(`order:${orderId}`);
        const orderData = await orderTrackingService.getOrder(orderId);
        socket.emit('order:details', orderData);
      }
    });

    // Auction
    socket.on('join-auction', async ({ auctionId }) => {
      socket.join(`auction:${auctionId}`);
      const auction = await auctionService.getAuction(auctionId);
      socket.emit('auction:details', auction);
    });

    socket.on('place-bid', async ({ auctionId, amount }) => {
      try {
        const result = await auctionService.placeBid(
          auctionId, socket.userId, socket.userName, amount
        );
        socket.emit('bid:success', result);
      } catch (error) {
        socket.emit('bid:error', { message: error.message });
      }
    });

    // Queue
    socket.on('join-queue', async ({ serviceType }) => {
      try {
        const result = await queueService.joinQueue(socket.userId, serviceType);
        socket.emit('queue:joined', result);
      } catch (error) {
        socket.emit('queue:error', { message: error.message });
      }
    });

    // Admin: call next
    socket.on('queue:call-next', async ({ serviceType }) => {
      if (!['admin', 'agent'].includes(socket.role)) {
        return socket.emit('error', { message: 'Unauthorized' });
      }
      try {
        const result = await queueService.callNextCustomer(serviceType, socket.userId);
        socket.emit('queue:called', result);
        io.to('admin-room').emit('queue:updated', await queueService.getQueueStatus(serviceType));
      } catch (error) {
        socket.emit('queue:error', { message: error.message });
      }
    });

    socket.on('disconnect', () => {
      console.log(`Client disconnected: ${socket.id}`);
    });
  });

  // LINE Webhook endpoint
  app.post('/webhook',
    line.middleware({ channelSecret: config.line.channelSecret }),
    (req, res) => {
      const controller = new WebhookController(chatService, orderTrackingService, lineClient);
      controller.handleWebhook(req, res);
    }
  );

  // API Routes
  app.use('/api/orders', require('./routes/orderRoutes')(orderTrackingService));
  app.use('/api/auctions', require('./routes/auctionRoutes')(auctionService));
  app.use('/api/queue', require('./routes/queueRoutes')(queueService));
  app.use('/api/auth', require('./routes/authRoutes'));

  // Health check
  app.get('/health', (req, res) => {
    res.json({
      status: 'ok',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
    });
  });

  return { app, httpServer, io };
}

// Start server
createApp().then(({ httpServer }) => {
  const PORT = config.server.port;
  httpServer.listen(PORT, () => {
    console.log(`🚀 Server running on port ${PORT}`);
    console.log(`🌍 Environment: ${config.server.nodeEnv}`);
  });
}).catch(console.error);
```

### 11.3 Docker Compose Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  app-1:
    build: .
    environment:
      - SERVER_ID=app-1
      - PORT=3000
      - REDIS_HOST=redis
      - NODE_ENV=production
    depends_on:
      - redis
      - postgres
    networks:
      - app-network

  app-2:
    build: .
    environment:
      - SERVER_ID=app-2
      - PORT=3000
      - REDIS_HOST=redis
      - NODE_ENV=production
    depends_on:
      - redis
      - postgres
    networks:
      - app-network

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/ssl
    depends_on:
      - app-1
      - app-2
    networks:
      - app-network

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    networks:
      - app-network

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: linebot_db
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
    networks:
      - app-network

volumes:
  redis-data:
  postgres-data:

networks:
  app-network:
    driver: bridge
```

### 11.4 การ Test ระบบ

```javascript
// tests/websocket.test.js
const { createServer } = require('http');
const { Server } = require('socket.io');
const Client = require('socket.io-client');
const jwt = require('jsonwebtoken');
const config = require('../src/config');

describe('WebSocket Tests', () => {
  let io, serverSocket, clientSocket;
  let httpServer;

  beforeAll((done) => {
    httpServer = createServer();
    io = new Server(httpServer);
    httpServer.listen(() => {
      const port = httpServer.address().port;
      
      const token = jwt.sign(
        { userId: 'test-user', role: 'admin' },
        config.jwt.secret
      );
      
      clientSocket = Client(`http://localhost:${port}`, {
        auth: { token },
      });
      
      io.on('connection', (socket) => {
        serverSocket = socket;
      });
      
      clientSocket.on('connect', done);
    });
  });

  afterAll(() => {
    io.close();
    clientSocket.close();
  });

  test('should authenticate successfully', (done) => {
    clientSocket.on('connect', () => {
      expect(clientSocket.connected).toBe(true);
      done();
    });
  });

  test('should receive order status update', (done) => {
    clientSocket.emit('join-order-room', { orderId: 'test-order-1' });
    
    clientSocket.on('order:status-updated', (data) => {
      expect(data.orderId).toBe('test-order-1');
      expect(data.status).toBeDefined();
      done();
    });

    // Simulate server sending update
    setTimeout(() => {
      serverSocket.emit('order:status-updated', {
        orderId: 'test-order-1',
        status: 'SHIPPING',
        statusLabel: 'กำลังจัดส่ง',
      });
    }, 100);
  });

  test('should handle bid placement', (done) => {
    clientSocket.emit('place-bid', {
      auctionId: 'test-auction-1',
      amount: 1000,
    });

    clientSocket.on('bid:success', (data) => {
      expect(data.success).toBe(true);
      done();
    });

    clientSocket.on('bid:error', (data) => {
      // Handle expected errors
      done();
    });
  });

  test('should join queue successfully', (done) => {
    clientSocket.emit('join-queue', { serviceType: 'general' });
    
    clientSocket.on('queue:joined', (data) => {
      expect(data.status).toBeDefined();
      expect(data.ticketId).toBeDefined();
      done();
    });
  });
});
```

### 11.5 Performance Monitoring

```javascript
// src/monitoring/websocketMetrics.js
const prometheus = require('prom-client');

// Metrics
const wsConnections = new prometheus.Gauge({
  name: 'websocket_active_connections',
  help: 'Number of active WebSocket connections',
  labelNames: ['server_id'],
});

const wsMessages = new prometheus.Counter({
  name: 'websocket_messages_total',
  help: 'Total WebSocket messages processed',
  labelNames: ['event_type', 'direction'],
});

const wsLatency = new prometheus.Histogram({
  name: 'websocket_message_latency_seconds',
  help: 'WebSocket message processing latency',
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1],
});

// Middleware สำหรับ tracking metrics
function metricsMiddleware(socket, next) {
  wsConnections.inc({ server_id: process.env.SERVER_ID });
  
  socket.on('disconnect', () => {
    wsConnections.dec({ server_id: process.env.SERVER_ID });
  });

  const originalEmit = socket.emit.bind(socket);
  socket.emit = (event, ...args) => {
    wsMessages.inc({ event_type: event, direction: 'outbound' });
    return originalEmit(event, ...args);
  };

  socket.use(([event, ...args], next) => {
    const start = Date.now();
    wsMessages.inc({ event_type: event, direction: 'inbound' });
    
    const done = () => {
      wsLatency.observe((Date.now() - start) / 1000);
    };
    
    next();
    done();
  });

  next();
}

module.exports = { metricsMiddleware, wsConnections, wsMessages, wsLatency };
```

---

## สรุปบทที่ 96

ในบทนี้เราได้เรียนรู้:

1. **WebSocket พื้นฐาน** - โปรโตคอล, ความแตกต่างจาก HTTP, use cases
2. **Real-time Chat** - การสร้างระบบ chat ระหว่าง LINE users และ agents
3. **Admin Dashboard** - Socket.io สำหรับ real-time monitoring
4. **Order Tracking** - ติดตามสถานะคำสั่งซื้อแบบ real-time
5. **Live Auction** - ระบบประมูลสดผ่าน LINE + LIFF
6. **Queue System** - ระบบคิวแบบ real-time
7. **LIFF Integration** - การเชื่อมต่อ WebSocket ใน LIFF
8. **Redis Pub/Sub** - การ scale WebSocket หลาย instances
9. **Security** - การรักษาความปลอดภัย WebSocket
10. **Load Balancing** - Nginx configuration สำหรับ WebSocket

### Checklist การ Implement

- [ ] ติดตั้ง Socket.io และ Redis adapter
- [ ] Setup authentication middleware
- [ ] สร้าง admin dashboard
- [ ] Implement order tracking
- [ ] Setup Redis Pub/Sub
- [ ] Configure Nginx load balancer
- [ ] เพิ่ม rate limiting
- [ ] เขียน tests
- [ ] Setup monitoring
- [ ] Deploy ด้วย Docker Compose

---

*หมายเหตุ: ในการ production ควรเพิ่ม SSL/TLS, monitoring, และ alerting ที่สมบูรณ์*
