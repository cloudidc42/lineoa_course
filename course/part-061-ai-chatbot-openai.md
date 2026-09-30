# Part 61: AI Chatbot ด้วย OpenAI GPT สำหรับ LINE Bot

## บทนำ

การนำ AI มาใช้กับ LINE Bot เปิดโอกาสให้สร้างประสบการณ์การสนทนาที่ชาญฉลาดและมีประสิทธิภาพสูง OpenAI GPT เป็นหนึ่งในโมเดลภาษาที่ทรงพลังที่สุดในปัจจุบัน บทนี้จะครอบคลุมทุกด้านของการผสาน OpenAI กับ LINE Bot ตั้งแต่การตั้งค่าพื้นฐานจนถึงระบบ Production ที่สมบูรณ์

## เนื้อหาที่จะเรียนรู้

1. การตั้งค่า OpenAI API
2. การส่งข้อความ LINE ไปยัง OpenAI
3. การรับและจัดรูปแบบคำตอบ
4. การจัดการประวัติการสนทนา
5. การจัดการ Context Window
6. System Prompts สำหรับ LINE Bot
7. Rate Limiting
8. กลยุทธ์การจัดการต้นทุน
9. Streaming Responses
10. Function Calling กับ LINE Actions

---

## สถาปัตยกรรมระบบ

```
┌─────────────────────────────────────────────────────────────────┐
│                        LINE Platform                            │
│  User ──→ LINE App ──→ LINE Server ──→ Webhook URL             │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Backend Server (Node.js)                    │
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────┐  │
│  │  LINE SDK    │───▶│  Message     │───▶│  Conversation    │  │
│  │  Middleware  │    │  Processor   │    │  Manager         │  │
│  └──────────────┘    └──────────────┘    └──────────────────┘  │
│                              │                     │            │
│                              ▼                     ▼            │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    OpenAI Service                        │  │
│  │  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐  │  │
│  │  │ Rate Limiter│  │ Token Counter │  │ Cost Tracker  │  │  │
│  │  └─────────────┘  └──────────────┘  └───────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                              │                                  │
│                              ▼                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                      Database                            │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌───────────────┐  │  │
│  │  │  Redis Cache │  │  PostgreSQL  │  │  MongoDB      │  │  │
│  │  │  (Sessions)  │  │  (Users)     │  │  (History)    │  │  │
│  │  └──────────────┘  └──────────────┘  └───────────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────┐
│                        OpenAI API                               │
│  GPT-4o / GPT-4-turbo / GPT-3.5-turbo                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. การตั้งค่า OpenAI API

### 1.1 สมัครและรับ API Key

```bash
# ขั้นตอนการสมัคร OpenAI
# 1. ไปที่ https://platform.openai.com
# 2. สร้างบัญชีหรือล็อกอิน
# 3. ไปที่ API Keys section
# 4. คลิก "Create new secret key"
# 5. เก็บ key ไว้อย่างปลอดภัย
```

### 1.2 โครงสร้างโปรเจค

```bash
mkdir line-openai-bot
cd line-openai-bot

# สร้างโครงสร้างโฟลเดอร์
mkdir -p src/{services,middleware,models,utils,config}
mkdir -p tests/{unit,integration}

# สร้างไฟล์หลัก
touch src/index.js
touch src/config/index.js
touch src/services/openai.js
touch src/services/lineBot.js
touch src/services/conversation.js
touch src/models/user.js
touch src/models/conversation.js
touch src/middleware/auth.js
touch src/utils/tokenCounter.js
touch src/utils/rateLimiter.js
touch .env
touch .env.example
```

### 1.3 ติดตั้ง Dependencies

```bash
npm init -y

# Core dependencies
npm install @line/bot-sdk openai express dotenv

# Database
npm install mongoose redis ioredis

# Utilities
npm install tiktoken winston morgan helmet cors

# Development
npm install -D nodemon jest supertest
```

### 1.4 ไฟล์ .env

```env
# LINE Bot Configuration
LINE_CHANNEL_ACCESS_TOKEN=your_line_channel_access_token_here
LINE_CHANNEL_SECRET=your_line_channel_secret_here

# OpenAI Configuration
OPENAI_API_KEY=sk-your_openai_api_key_here
OPENAI_MODEL=gpt-4o
OPENAI_MAX_TOKENS=1000
OPENAI_TEMPERATURE=0.7

# Database
MONGODB_URI=mongodb://localhost:27017/line-openai-bot
REDIS_URL=redis://localhost:6379

# Server
PORT=3000
NODE_ENV=development

# Rate Limiting
RATE_LIMIT_REQUESTS_PER_MINUTE=10
RATE_LIMIT_TOKENS_PER_MINUTE=40000

# Cost Management
MAX_DAILY_COST_USD=10.00
ALERT_COST_THRESHOLD_USD=8.00

# Conversation Settings
MAX_CONVERSATION_HISTORY=20
MAX_CONTEXT_TOKENS=8000
SESSION_TIMEOUT_MINUTES=60

# Logging
LOG_LEVEL=info
LOG_FILE=./logs/app.log
```

---

## 2. การตั้งค่า Configuration

```javascript
// src/config/index.js
require('dotenv').config();

const config = {
  line: {
    channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
    channelSecret: process.env.LINE_CHANNEL_SECRET,
  },
  openai: {
    apiKey: process.env.OPENAI_API_KEY,
    model: process.env.OPENAI_MODEL || 'gpt-4o',
    maxTokens: parseInt(process.env.OPENAI_MAX_TOKENS) || 1000,
    temperature: parseFloat(process.env.OPENAI_TEMPERATURE) || 0.7,
  },
  database: {
    mongoUri: process.env.MONGODB_URI || 'mongodb://localhost:27017/line-openai-bot',
    redisUrl: process.env.REDIS_URL || 'redis://localhost:6379',
  },
  server: {
    port: parseInt(process.env.PORT) || 3000,
    nodeEnv: process.env.NODE_ENV || 'development',
  },
  rateLimit: {
    requestsPerMinute: parseInt(process.env.RATE_LIMIT_REQUESTS_PER_MINUTE) || 10,
    tokensPerMinute: parseInt(process.env.RATE_LIMIT_TOKENS_PER_MINUTE) || 40000,
  },
  cost: {
    maxDailyCostUsd: parseFloat(process.env.MAX_DAILY_COST_USD) || 10.00,
    alertThresholdUsd: parseFloat(process.env.ALERT_COST_THRESHOLD_USD) || 8.00,
  },
  conversation: {
    maxHistory: parseInt(process.env.MAX_CONVERSATION_HISTORY) || 20,
    maxContextTokens: parseInt(process.env.MAX_CONTEXT_TOKENS) || 8000,
    sessionTimeoutMinutes: parseInt(process.env.SESSION_TIMEOUT_MINUTES) || 60,
  },
  logging: {
    level: process.env.LOG_LEVEL || 'info',
    file: process.env.LOG_FILE || './logs/app.log',
  },
};

// Validate required config
const requiredFields = [
  'line.channelAccessToken',
  'line.channelSecret',
  'openai.apiKey',
];

requiredFields.forEach((field) => {
  const value = field.split('.').reduce((obj, key) => obj?.[key], config);
  if (!value) {
    throw new Error(`Missing required configuration: ${field}`);
  }
});

module.exports = config;
```

---

## 3. Models (MongoDB)

### 3.1 User Model

```javascript
// src/models/user.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  lineUserId: {
    type: String,
    required: true,
    unique: true,
    index: true,
  },
  displayName: String,
  pictureUrl: String,
  statusMessage: String,
  language: {
    type: String,
    default: 'th',
  },
  preferences: {
    responseStyle: {
      type: String,
      enum: ['concise', 'detailed', 'friendly'],
      default: 'friendly',
    },
    language: {
      type: String,
      default: 'th',
    },
  },
  stats: {
    totalMessages: { type: Number, default: 0 },
    totalTokensUsed: { type: Number, default: 0 },
    totalCostUsd: { type: Number, default: 0 },
    firstMessageAt: Date,
    lastMessageAt: Date,
  },
  dailyStats: [{
    date: { type: Date, required: true },
    messages: { type: Number, default: 0 },
    tokensUsed: { type: Number, default: 0 },
    costUsd: { type: Number, default: 0 },
  }],
  isBlocked: {
    type: Boolean,
    default: false,
  },
  blockReason: String,
  systemPromptOverride: String,
  createdAt: {
    type: Date,
    default: Date.now,
  },
  updatedAt: {
    type: Date,
    default: Date.now,
  },
});

userSchema.pre('save', function (next) {
  this.updatedAt = Date.now();
  next();
});

userSchema.methods.updateDailyStats = function (tokens, costUsd) {
  const today = new Date();
  today.setHours(0, 0, 0, 0);

  let dailyStat = this.dailyStats.find(
    (s) => s.date.getTime() === today.getTime()
  );

  if (!dailyStat) {
    this.dailyStats.push({
      date: today,
      messages: 1,
      tokensUsed: tokens,
      costUsd: costUsd,
    });
  } else {
    dailyStat.messages += 1;
    dailyStat.tokensUsed += tokens;
    dailyStat.costUsd += costUsd;
  }

  // Keep only last 30 days
  if (this.dailyStats.length > 30) {
    this.dailyStats = this.dailyStats.slice(-30);
  }
};

module.exports = mongoose.model('User', userSchema);
```

### 3.2 Conversation Model

```javascript
// src/models/conversation.js
const mongoose = require('mongoose');

const messageSchema = new mongoose.Schema({
  role: {
    type: String,
    enum: ['user', 'assistant', 'system'],
    required: true,
  },
  content: {
    type: String,
    required: true,
  },
  timestamp: {
    type: Date,
    default: Date.now,
  },
  tokens: Number,
  metadata: {
    messageType: String, // text, image, audio, etc.
    imageUrl: String,
    processingTime: Number,
    model: String,
    finishReason: String,
  },
});

const conversationSchema = new mongoose.Schema({
  lineUserId: {
    type: String,
    required: true,
    index: true,
  },
  sessionId: {
    type: String,
    required: true,
    index: true,
  },
  messages: [messageSchema],
  totalTokens: {
    type: Number,
    default: 0,
  },
  totalCostUsd: {
    type: Number,
    default: 0,
  },
  startedAt: {
    type: Date,
    default: Date.now,
  },
  lastActivityAt: {
    type: Date,
    default: Date.now,
  },
  isActive: {
    type: Boolean,
    default: true,
  },
  context: {
    type: Map,
    of: mongoose.Schema.Types.Mixed,
  },
});

conversationSchema.index({ lineUserId: 1, lastActivityAt: -1 });
conversationSchema.index({ sessionId: 1 });

conversationSchema.methods.addMessage = function (role, content, metadata = {}) {
  this.messages.push({ role, content, metadata });
  this.lastActivityAt = Date.now();
  return this;
};

conversationSchema.methods.getRecentMessages = function (maxMessages = 20, maxTokens = 8000) {
  // Return recent messages within token limit
  const messages = [...this.messages].reverse();
  const result = [];
  let tokenCount = 0;

  for (const msg of messages) {
    const msgTokens = msg.tokens || Math.ceil(msg.content.length / 4);
    if (tokenCount + msgTokens > maxTokens && result.length > 0) {
      break;
    }
    result.unshift(msg);
    tokenCount += msgTokens;
    if (result.length >= maxMessages) break;
  }

  return result;
};

module.exports = mongoose.model('Conversation', conversationSchema);
```

---

## 4. Utilities

### 4.1 Token Counter

```javascript
// src/utils/tokenCounter.js
const { encoding_for_model } = require('tiktoken');

class TokenCounter {
  constructor(model = 'gpt-4o') {
    this.model = model;
    this._encoder = null;
  }

  getEncoder() {
    if (!this._encoder) {
      try {
        this._encoder = encoding_for_model(this.model);
      } catch {
        // Fallback for models not in tiktoken
        this._encoder = encoding_for_model('gpt-4');
      }
    }
    return this._encoder;
  }

  countTokens(text) {
    try {
      const encoder = this.getEncoder();
      return encoder.encode(text).length;
    } catch (error) {
      // Approximate counting if tiktoken fails
      return Math.ceil(text.length / 4);
    }
  }

  countMessagesTokens(messages) {
    let totalTokens = 0;

    for (const message of messages) {
      // Each message has overhead
      totalTokens += 4; // message overhead
      totalTokens += this.countTokens(message.content);
      totalTokens += this.countTokens(message.role);
    }

    totalTokens += 2; // reply priming

    return totalTokens;
  }

  calculateCost(promptTokens, completionTokens, model) {
    // Prices per 1000 tokens (USD) - update as OpenAI changes pricing
    const pricing = {
      'gpt-4o': {
        prompt: 0.005,
        completion: 0.015,
      },
      'gpt-4o-mini': {
        prompt: 0.000150,
        completion: 0.000600,
      },
      'gpt-4-turbo': {
        prompt: 0.01,
        completion: 0.03,
      },
      'gpt-3.5-turbo': {
        prompt: 0.0005,
        completion: 0.0015,
      },
    };

    const modelPricing = pricing[model] || pricing['gpt-4o'];
    const promptCost = (promptTokens / 1000) * modelPricing.prompt;
    const completionCost = (completionTokens / 1000) * modelPricing.completion;

    return {
      promptCost,
      completionCost,
      totalCost: promptCost + completionCost,
    };
  }

  dispose() {
    if (this._encoder) {
      this._encoder.free();
      this._encoder = null;
    }
  }
}

module.exports = new TokenCounter();
```

### 4.2 Rate Limiter

```javascript
// src/utils/rateLimiter.js
const Redis = require('ioredis');
const config = require('../config');
const logger = require('./logger');

class RateLimiter {
  constructor() {
    this.redis = new Redis(config.database.redisUrl);
    this.windowMs = 60 * 1000; // 1 minute window
  }

  async checkLimit(userId, type = 'requests') {
    const key = `ratelimit:${type}:${userId}`;
    const limit = type === 'requests'
      ? config.rateLimit.requestsPerMinute
      : config.rateLimit.tokensPerMinute;

    const now = Date.now();
    const windowStart = now - this.windowMs;

    // Use sliding window algorithm
    const pipeline = this.redis.pipeline();
    pipeline.zremrangebyscore(key, '-inf', windowStart);
    pipeline.zcard(key);
    pipeline.expire(key, 120);

    const results = await pipeline.exec();
    const currentCount = results[1][1];

    if (currentCount >= limit) {
      const oldestEntry = await this.redis.zrange(key, 0, 0, 'WITHSCORES');
      const resetTime = oldestEntry.length
        ? parseInt(oldestEntry[1]) + this.windowMs
        : now + this.windowMs;

      return {
        allowed: false,
        remaining: 0,
        resetAt: new Date(resetTime),
        retryAfterMs: resetTime - now,
      };
    }

    return {
      allowed: true,
      remaining: limit - currentCount - 1,
      resetAt: new Date(now + this.windowMs),
    };
  }

  async recordUsage(userId, type = 'requests', count = 1) {
    const key = `ratelimit:${type}:${userId}`;
    const now = Date.now();

    const pipeline = this.redis.pipeline();
    for (let i = 0; i < count; i++) {
      pipeline.zadd(key, now + i, `${now}:${i}:${Math.random()}`);
    }
    pipeline.expire(key, 120);

    await pipeline.exec();
  }

  async getUserUsage(userId) {
    const now = Date.now();
    const windowStart = now - this.windowMs;

    const [requestCount, tokenCount] = await Promise.all([
      this.redis.zcount(`ratelimit:requests:${userId}`, windowStart, '+inf'),
      this.redis.zcount(`ratelimit:tokens:${userId}`, windowStart, '+inf'),
    ]);

    return {
      requests: {
        used: requestCount,
        limit: config.rateLimit.requestsPerMinute,
        remaining: Math.max(0, config.rateLimit.requestsPerMinute - requestCount),
      },
      tokens: {
        used: tokenCount,
        limit: config.rateLimit.tokensPerMinute,
        remaining: Math.max(0, config.rateLimit.tokensPerMinute - tokenCount),
      },
    };
  }

  async isGlobalLimitReached() {
    const today = new Date().toISOString().split('T')[0];
    const key = `cost:daily:${today}`;
    const dailyCost = parseFloat(await this.redis.get(key) || '0');

    return {
      reached: dailyCost >= config.cost.maxDailyCostUsd,
      shouldAlert: dailyCost >= config.cost.alertThresholdUsd,
      currentCost: dailyCost,
      maxCost: config.cost.maxDailyCostUsd,
    };
  }

  async recordCost(costUsd) {
    const today = new Date().toISOString().split('T')[0];
    const key = `cost:daily:${today}`;

    await this.redis.incrbyfloat(key, costUsd);
    await this.redis.expire(key, 48 * 3600); // Keep for 48 hours
  }
}

module.exports = new RateLimiter();
```

### 4.3 Logger

```javascript
// src/utils/logger.js
const winston = require('winston');
const config = require('../config');

const logger = winston.createLogger({
  level: config.logging.level,
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'line-openai-bot' },
  transports: [
    new winston.transports.File({
      filename: config.logging.file,
      maxsize: 10 * 1024 * 1024, // 10MB
      maxFiles: 5,
    }),
    new winston.transports.File({
      filename: './logs/error.log',
      level: 'error',
    }),
  ],
});

if (config.server.nodeEnv !== 'production') {
  logger.add(
    new winston.transports.Console({
      format: winston.format.combine(
        winston.format.colorize(),
        winston.format.simple()
      ),
    })
  );
}

module.exports = logger;
```

---

## 5. OpenAI Service หลัก

```javascript
// src/services/openai.js
const { OpenAI } = require('openai');
const config = require('../config');
const tokenCounter = require('../utils/tokenCounter');
const rateLimiter = require('../utils/rateLimiter');
const logger = require('../utils/logger');

class OpenAIService {
  constructor() {
    this.client = new OpenAI({
      apiKey: config.openai.apiKey,
    });
    this.model = config.openai.model;
  }

  // System prompt สำหรับ LINE Bot
  buildSystemPrompt(user = null, customPrompt = null) {
    const basePrompt = `คุณคือ AI Assistant ที่ช่วยเหลือผู้ใช้ผ่าน LINE Official Account
    
คุณมีคุณสมบัติดังนี้:
- ตอบภาษาไทยได้อย่างคล่องแคล่ว
- เป็นมิตรและสุภาพ
- ให้ข้อมูลที่ถูกต้องและเป็นประโยชน์
- ตอบคำถามกระชับและชัดเจน
- ไม่ตอบเรื่องที่ไม่เหมาะสมหรือผิดกฎหมาย

กฎการตอบ:
1. ตอบในภาษาเดียวกับที่ผู้ใช้ถาม
2. ถ้าไม่แน่ใจ ให้บอกว่าไม่แน่ใจ ไม่ใช่เดา
3. รักษาความปลอดภัยและความเป็นส่วนตัวของผู้ใช้
4. ห้ามเปิดเผยข้อมูลส่วนตัวของผู้ใช้คนอื่น`;

    if (customPrompt) {
      return customPrompt;
    }

    if (user?.preferences?.responseStyle === 'concise') {
      return `${basePrompt}\n\nสไตล์การตอบ: กระชับ ตรงประเด็น ไม่เยิ่นเย้อ`;
    }

    if (user?.preferences?.responseStyle === 'detailed') {
      return `${basePrompt}\n\nสไตล์การตอบ: อธิบายละเอียด ครบถ้วน มีตัวอย่าง`;
    }

    return basePrompt;
  }

  // ส่งข้อความไปยัง OpenAI
  async chat(messages, options = {}) {
    const {
      model = this.model,
      maxTokens = config.openai.maxTokens,
      temperature = config.openai.temperature,
      stream = false,
      functions = null,
      functionCall = null,
    } = options;

    const startTime = Date.now();

    try {
      const requestParams = {
        model,
        messages,
        max_tokens: maxTokens,
        temperature,
        stream,
      };

      if (functions) {
        requestParams.tools = functions.map((fn) => ({
          type: 'function',
          function: fn,
        }));
      }

      if (functionCall) {
        requestParams.tool_choice = functionCall;
      }

      const response = await this.client.chat.completions.create(requestParams);

      const processingTime = Date.now() - startTime;

      if (!stream) {
        const usage = response.usage;
        const cost = tokenCounter.calculateCost(
          usage.prompt_tokens,
          usage.completion_tokens,
          model
        );

        logger.info('OpenAI API call completed', {
          model,
          promptTokens: usage.prompt_tokens,
          completionTokens: usage.completion_tokens,
          totalTokens: usage.total_tokens,
          costUsd: cost.totalCost,
          processingTimeMs: processingTime,
        });

        // Record cost
        await rateLimiter.recordCost(cost.totalCost);

        return {
          content: response.choices[0].message.content,
          role: 'assistant',
          usage,
          cost,
          processingTime,
          finishReason: response.choices[0].finish_reason,
          toolCalls: response.choices[0].message.tool_calls,
          model,
        };
      }

      return response; // Stream object
    } catch (error) {
      logger.error('OpenAI API error', {
        error: error.message,
        errorCode: error.code,
        errorType: error.type,
        model,
        processingTimeMs: Date.now() - startTime,
      });
      throw this.handleOpenAIError(error);
    }
  }

  // Streaming response
  async chatStream(messages, onChunk, options = {}) {
    const stream = await this.chat(messages, { ...options, stream: true });

    let fullContent = '';
    let usage = null;

    for await (const chunk of stream) {
      const delta = chunk.choices[0]?.delta;

      if (delta?.content) {
        fullContent += delta.content;
        await onChunk(delta.content, false);
      }

      if (chunk.usage) {
        usage = chunk.usage;
      }

      if (chunk.choices[0]?.finish_reason) {
        await onChunk('', true);
      }
    }

    const cost = usage
      ? tokenCounter.calculateCost(
          usage.prompt_tokens,
          usage.completion_tokens,
          options.model || this.model
        )
      : null;

    if (cost) {
      await rateLimiter.recordCost(cost.totalCost);
    }

    return { content: fullContent, usage, cost };
  }

  // จัดการ Error
  handleOpenAIError(error) {
    if (error.status === 429) {
      return new Error('ขณะนี้มีผู้ใช้งานมาก กรุณาลองใหม่อีกครั้ง');
    }

    if (error.status === 400) {
      return new Error('ข้อความไม่ถูกต้อง กรุณาลองใหม่');
    }

    if (error.status === 401) {
      return new Error('ไม่สามารถเชื่อมต่อกับ AI ได้ในขณะนี้');
    }

    if (error.status === 503) {
      return new Error('บริการ AI ไม่พร้อมใช้งานชั่วคราว กรุณาลองใหม่ในอีกสักครู่');
    }

    if (error.code === 'context_length_exceeded') {
      return new Error('ประวัติการสนทนายาวเกินไป กำลังรีเซ็ต...');
    }

    return new Error('เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง');
  }
}

module.exports = new OpenAIService();
```

---

## 6. Conversation Manager

```javascript
// src/services/conversation.js
const Redis = require('ioredis');
const Conversation = require('../models/conversation');
const tokenCounter = require('../utils/tokenCounter');
const config = require('../config');
const logger = require('../utils/logger');
const { v4: uuidv4 } = require('uuid');

class ConversationManager {
  constructor() {
    this.redis = new Redis(config.database.redisUrl);
    this.sessionPrefix = 'session:';
    this.sessionTTL = config.conversation.sessionTimeoutMinutes * 60;
  }

  // รับหรือสร้าง Session ID
  async getOrCreateSessionId(lineUserId) {
    const sessionKey = `${this.sessionPrefix}${lineUserId}`;
    let sessionId = await this.redis.get(sessionKey);

    if (!sessionId) {
      sessionId = uuidv4();
      await this.redis.setex(sessionKey, this.sessionTTL, sessionId);
      logger.debug('New session created', { lineUserId, sessionId });
    } else {
      // Refresh TTL
      await this.redis.expire(sessionKey, this.sessionTTL);
    }

    return sessionId;
  }

  // รีเซ็ต Session
  async resetSession(lineUserId) {
    const sessionKey = `${this.sessionPrefix}${lineUserId}`;
    const oldSessionId = await this.redis.get(sessionKey);

    if (oldSessionId) {
      // Mark old conversation as inactive
      await Conversation.updateOne(
        { sessionId: oldSessionId },
        { isActive: false }
      );
    }

    const newSessionId = uuidv4();
    await this.redis.setex(sessionKey, this.sessionTTL, newSessionId);

    logger.info('Session reset', { lineUserId, oldSessionId, newSessionId });
    return newSessionId;
  }

  // ดึงประวัติการสนทนา
  async getHistory(lineUserId, sessionId) {
    // Try Redis cache first
    const cacheKey = `history:${sessionId}`;
    const cached = await this.redis.get(cacheKey);

    if (cached) {
      return JSON.parse(cached);
    }

    // Load from MongoDB
    const conversation = await Conversation.findOne({
      lineUserId,
      sessionId,
      isActive: true,
    });

    if (!conversation) {
      return [];
    }

    const messages = conversation.getRecentMessages(
      config.conversation.maxHistory,
      config.conversation.maxContextTokens
    );

    const history = messages.map((msg) => ({
      role: msg.role,
      content: msg.content,
    }));

    // Cache in Redis
    await this.redis.setex(cacheKey, 300, JSON.stringify(history)); // 5 min cache

    return history;
  }

  // บันทึกข้อความ
  async saveMessage(lineUserId, sessionId, role, content, metadata = {}) {
    const tokensUsed = tokenCounter.countTokens(content);

    // Update or create conversation in MongoDB
    const conversation = await Conversation.findOneAndUpdate(
      { lineUserId, sessionId, isActive: true },
      {
        $push: {
          messages: {
            role,
            content,
            tokens: tokensUsed,
            metadata,
          },
        },
        $inc: { totalTokens: tokensUsed },
        $set: { lastActivityAt: new Date() },
      },
      { upsert: true, new: true }
    );

    // Invalidate cache
    const cacheKey = `history:${sessionId}`;
    await this.redis.del(cacheKey);

    return conversation;
  }

  // จัดการ Context Window
  async pruneContext(messages, maxTokens = config.conversation.maxContextTokens) {
    let totalTokens = tokenCounter.countMessagesTokens(messages);

    if (totalTokens <= maxTokens) {
      return messages;
    }

    // Keep system message (first) and remove oldest user/assistant pairs
    const systemMessages = messages.filter((m) => m.role === 'system');
    const conversationMessages = messages.filter((m) => m.role !== 'system');

    const pruned = [...systemMessages];
    let tokenBudget = maxTokens - tokenCounter.countMessagesTokens(systemMessages);

    // Add messages from newest to oldest until budget is used
    for (let i = conversationMessages.length - 1; i >= 0; i--) {
      const msg = conversationMessages[i];
      const msgTokens = tokenCounter.countTokens(msg.content) + 4;

      if (tokenBudget - msgTokens < 0) break;

      pruned.splice(systemMessages.length, 0, msg);
      tokenBudget -= msgTokens;
    }

    logger.debug('Context pruned', {
      originalTokens: totalTokens,
      prunedTokens: tokenCounter.countMessagesTokens(pruned),
      removedMessages: messages.length - pruned.length,
    });

    return pruned;
  }

  // สร้าง Messages Array สำหรับ OpenAI
  async buildMessages(lineUserId, sessionId, newMessage, systemPrompt) {
    const history = await this.getHistory(lineUserId, sessionId);

    const messages = [
      { role: 'system', content: systemPrompt },
      ...history,
      { role: 'user', content: newMessage },
    ];

    return this.pruneContext(messages);
  }

  // ดึงสรุปการสนทนา
  async summarizeConversation(lineUserId, sessionId) {
    const history = await this.getHistory(lineUserId, sessionId);

    if (history.length < 10) {
      return null; // ไม่จำเป็นต้องสรุปถ้าสนทนาสั้น
    }

    const summaryPrompt = [
      {
        role: 'system',
        content: 'สรุปการสนทนาต่อไปนี้เป็นประเด็นสำคัญ ไม่เกิน 3 ประโยค',
      },
      ...history,
      {
        role: 'user',
        content: 'กรุณาสรุปการสนทนาข้างต้น',
      },
    ];

    return summaryPrompt;
  }
}

module.exports = new ConversationManager();
```

---

## 7. LINE Bot Service หลัก

```javascript
// src/services/lineBot.js
const line = require('@line/bot-sdk');
const config = require('../config');
const openaiService = require('./openai');
const conversationManager = require('./conversation');
const rateLimiter = require('../utils/rateLimiter');
const tokenCounter = require('../utils/tokenCounter');
const User = require('../models/user');
const logger = require('../utils/logger');

class LineBotService {
  constructor() {
    this.client = new line.messagingApi.MessagingApiClient({
      channelAccessToken: config.line.channelAccessToken,
    });
  }

  // จัดการ Event หลัก
  async handleEvent(event) {
    const userId = event.source.userId;

    try {
      switch (event.type) {
        case 'message':
          return this.handleMessage(event, userId);
        case 'follow':
          return this.handleFollow(event, userId);
        case 'unfollow':
          return this.handleUnfollow(userId);
        case 'postback':
          return this.handlePostback(event, userId);
        default:
          logger.debug('Unhandled event type', { eventType: event.type });
          return null;
      }
    } catch (error) {
      logger.error('Event handling error', {
        error: error.message,
        userId,
        eventType: event.type,
      });
    }
  }

  // จัดการข้อความ
  async handleMessage(event, userId) {
    const { message, replyToken } = event;

    if (message.type === 'text') {
      return this.handleTextMessage(userId, message.text, replyToken);
    }

    if (message.type === 'image') {
      return this.handleImageMessage(userId, message.id, replyToken);
    }

    // ข้อความประเภทอื่น
    return this.replyText(replyToken, 'ขออภัย ฉันรองรับเฉพาะข้อความตัวอักษรและรูปภาพในขณะนี้');
  }

  // จัดการข้อความ Text
  async handleTextMessage(userId, text, replyToken) {
    // ตรวจสอบคำสั่งพิเศษ
    const specialCommand = this.checkSpecialCommands(text);
    if (specialCommand) {
      return this.handleSpecialCommand(userId, specialCommand, replyToken);
    }

    // ตรวจสอบ Rate Limit
    const rateCheck = await rateLimiter.checkLimit(userId);
    if (!rateCheck.allowed) {
      const resetMinutes = Math.ceil(rateCheck.retryAfterMs / 60000);
      return this.replyText(
        replyToken,
        `คุณส่งข้อความถี่เกินไป กรุณารอ ${resetMinutes} นาที\n\nการใช้งานในนาทีนี้:\n${rateCheck.remaining} ครั้งที่เหลือ`
      );
    }

    // ตรวจสอบ Global Cost Limit
    const costCheck = await rateLimiter.isGlobalLimitReached();
    if (costCheck.reached) {
      return this.replyText(
        replyToken,
        'ขออภัย บริการถึงขีดจำกัดประจำวันแล้ว กรุณาลองใหม่ในวันพรุ่งนี้'
      );
    }

    // แสดง Loading Animation
    await this.client.showLoadingAnimation({ chatId: userId, loadingSeconds: 10 });

    try {
      // โหลดหรือสร้าง User
      const user = await this.getOrCreateUser(userId);

      if (user.isBlocked) {
        return this.replyText(replyToken, 'บัญชีของคุณถูกระงับการใช้งาน');
      }

      // สร้าง Session
      const sessionId = await conversationManager.getOrCreateSessionId(userId);

      // สร้าง System Prompt
      const systemPrompt = openaiService.buildSystemPrompt(
        user,
        user.systemPromptOverride
      );

      // สร้าง Messages
      const messages = await conversationManager.buildMessages(
        userId,
        sessionId,
        text,
        systemPrompt
      );

      // เรียก OpenAI
      const response = await openaiService.chat(messages, {
        model: config.openai.model,
        maxTokens: config.openai.maxTokens,
        temperature: config.openai.temperature,
      });

      // บันทึกการสนทนา
      await Promise.all([
        conversationManager.saveMessage(userId, sessionId, 'user', text, {
          messageType: 'text',
        }),
        conversationManager.saveMessage(
          userId,
          sessionId,
          'assistant',
          response.content,
          {
            model: response.model,
            processingTime: response.processingTime,
            finishReason: response.finishReason,
          }
        ),
      ]);

      // อัพเดทสถิติผู้ใช้
      await this.updateUserStats(user, response.usage, response.cost);

      // บันทึก Rate Limit
      await rateLimiter.recordUsage(userId, 'requests');
      await rateLimiter.recordUsage(
        userId,
        'tokens',
        response.usage.total_tokens
      );

      // ส่งคำตอบ
      return this.replyText(replyToken, response.content);
    } catch (error) {
      logger.error('Text message processing error', {
        error: error.message,
        userId,
      });
      return this.replyText(
        replyToken,
        error.message || 'เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง'
      );
    }
  }

  // ตรวจสอบคำสั่งพิเศษ
  checkSpecialCommands(text) {
    const trimmed = text.trim().toLowerCase();
    const commands = {
      '/reset': 'reset',
      '/ล้างประวัติ': 'reset',
      '/help': 'help',
      '/ช่วยเหลือ': 'help',
      '/status': 'status',
      '/สถานะ': 'status',
      '/style concise': 'style_concise',
      '/style detailed': 'style_detailed',
      '/style friendly': 'style_friendly',
    };

    return commands[trimmed] || null;
  }

  // จัดการคำสั่งพิเศษ
  async handleSpecialCommand(userId, command, replyToken) {
    switch (command) {
      case 'reset': {
        await conversationManager.resetSession(userId);
        return this.replyText(
          replyToken,
          '✅ ล้างประวัติการสนทนาแล้ว เริ่มต้นใหม่ได้เลย!'
        );
      }

      case 'help': {
        const helpText = `🤖 คำสั่งที่ใช้ได้:

/reset หรือ /ล้างประวัติ - ล้างประวัติการสนทนา
/status หรือ /สถานะ - ดูสถานะการใช้งาน
/style concise - ตั้งรูปแบบตอบสั้น
/style detailed - ตั้งรูปแบบตอบละเอียด
/style friendly - ตั้งรูปแบบตอบเป็นมิตร
/help หรือ /ช่วยเหลือ - แสดงคำสั่งนี้`;

        return this.replyText(replyToken, helpText);
      }

      case 'status': {
        const usage = await rateLimiter.getUserUsage(userId);
        const user = await User.findOne({ lineUserId: userId });
        const statusText = `📊 สถานะการใช้งาน:

นาทีนี้:
- ส่งข้อความแล้ว: ${usage.requests.used}/${usage.requests.limit} ครั้ง
- Tokens ใช้แล้ว: ${usage.tokens.used}/${usage.tokens.limit}

รวมทั้งหมด:
- ข้อความทั้งหมด: ${user?.stats?.totalMessages || 0} ครั้ง
- Tokens ทั้งหมด: ${user?.stats?.totalTokensUsed || 0}`;

        return this.replyText(replyToken, statusText);
      }

      case 'style_concise':
      case 'style_detailed':
      case 'style_friendly': {
        const style = command.split('_')[1];
        await User.updateOne(
          { lineUserId: userId },
          { 'preferences.responseStyle': style }
        );
        const styleNames = {
          concise: 'กระชับ',
          detailed: 'ละเอียด',
          friendly: 'เป็นมิตร',
        };
        return this.replyText(
          replyToken,
          `✅ เปลี่ยนรูปแบบการตอบเป็น "${styleNames[style]}" แล้ว`
        );
      }

      default:
        return null;
    }
  }

  // จัดการรูปภาพ
  async handleImageMessage(userId, messageId, replyToken) {
    return this.replyText(
      replyToken,
      'ขอบคุณสำหรับรูปภาพ! ฉันยังไม่สามารถวิเคราะห์รูปภาพได้ในขณะนี้ กรุณาส่งข้อความแทน'
    );
  }

  // Follow Event
  async handleFollow(event, userId) {
    try {
      const profile = await this.client.getProfile(userId);
      const user = await this.getOrCreateUser(userId, profile);

      const welcomeText = `สวัสดี ${profile.displayName}! 🎉

ยินดีต้อนรับสู่ AI Assistant!

ฉันสามารถช่วยคุณได้ในหลายเรื่อง เช่น:
• ตอบคำถามทั่วไป
• ช่วยเขียนและแก้ไขข้อความ
• แนะนำสถานที่และร้านอาหาร
• ให้ข้อมูลเกี่ยวกับสถานที่ต่างๆ

พิมพ์ /help เพื่อดูคำสั่งทั้งหมด
หรือพิมพ์ข้อความที่ต้องการได้เลย! 😊`;

      await this.client.pushMessage({
        to: userId,
        messages: [{ type: 'text', text: welcomeText }],
      });
    } catch (error) {
      logger.error('Follow handling error', { error: error.message, userId });
    }
  }

  // Unfollow Event
  async handleUnfollow(userId) {
    logger.info('User unfollowed', { userId });
    await User.updateOne({ lineUserId: userId }, { isActive: false });
  }

  // Postback Event
  async handlePostback(event, userId) {
    const { data } = event.postback;
    const params = new URLSearchParams(data);
    const action = params.get('action');

    switch (action) {
      case 'reset_conversation':
        await conversationManager.resetSession(userId);
        return this.replyText(event.replyToken, '✅ ล้างประวัติการสนทนาแล้ว');
      default:
        logger.debug('Unhandled postback action', { action, userId });
    }
  }

  // Helper: โหลดหรือสร้าง User
  async getOrCreateUser(userId, profile = null) {
    let user = await User.findOne({ lineUserId: userId });

    if (!user) {
      user = new User({
        lineUserId: userId,
        displayName: profile?.displayName,
        pictureUrl: profile?.pictureUrl,
        statusMessage: profile?.statusMessage,
        stats: { firstMessageAt: new Date() },
      });
      await user.save();
      logger.info('New user created', { userId });
    }

    return user;
  }

  // Helper: อัพเดทสถิติ
  async updateUserStats(user, usage, cost) {
    user.stats.totalMessages += 1;
    user.stats.totalTokensUsed += usage.total_tokens;
    user.stats.totalCostUsd += cost.totalCost;
    user.stats.lastMessageAt = new Date();
    user.updateDailyStats(usage.total_tokens, cost.totalCost);
    await user.save();
  }

  // Helper: ตอบข้อความ
  async replyText(replyToken, text) {
    return this.client.replyMessage({
      replyToken,
      messages: [{ type: 'text', text }],
    });
  }

  // Helper: ส่ง Flex Message
  async replyFlex(replyToken, altText, contents) {
    return this.client.replyMessage({
      replyToken,
      messages: [{ type: 'flex', altText, contents }],
    });
  }
}

module.exports = new LineBotService();
```

---

## 8. Main Application

```javascript
// src/index.js
const express = require('express');
const line = require('@line/bot-sdk');
const mongoose = require('mongoose');
const helmet = require('helmet');
const cors = require('cors');
const morgan = require('morgan');

const config = require('./config');
const lineBotService = require('./services/lineBot');
const logger = require('./utils/logger');

const app = express();

// Middleware
app.use(helmet());
app.use(cors());
app.use(morgan('combined', { stream: { write: (msg) => logger.info(msg.trim()) } }));

// LINE Webhook - ต้องใช้ raw body
app.post('/webhook', line.middleware({
  channelSecret: config.line.channelSecret,
}), async (req, res) => {
  try {
    const events = req.body.events;

    await Promise.all(
      events.map((event) => lineBotService.handleEvent(event))
    );

    res.status(200).json({ status: 'ok' });
  } catch (error) {
    logger.error('Webhook error', { error: error.message });
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Health Check
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
  });
});

// Stats endpoint (สำหรับ Monitoring)
app.get('/stats', async (req, res) => {
  try {
    const User = require('./models/user');
    const totalUsers = await User.countDocuments();
    const activeToday = await User.countDocuments({
      'stats.lastMessageAt': {
        $gte: new Date(Date.now() - 24 * 3600 * 1000),
      },
    });

    res.json({
      totalUsers,
      activeToday,
      uptime: process.uptime(),
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Error Handler
app.use((err, req, res, next) => {
  logger.error('Express error', { error: err.message, stack: err.stack });
  res.status(500).json({ error: 'Internal server error' });
});

// Start Server
async function start() {
  try {
    // Connect to MongoDB
    await mongoose.connect(config.database.mongoUri, {
      useNewUrlParser: true,
      useUnifiedTopology: true,
    });
    logger.info('Connected to MongoDB');

    // Start HTTP server
    app.listen(config.server.port, () => {
      logger.info(`Server started on port ${config.server.port}`);
    });
  } catch (error) {
    logger.error('Failed to start server', { error: error.message });
    process.exit(1);
  }
}

// Graceful shutdown
process.on('SIGTERM', async () => {
  logger.info('SIGTERM received, shutting down gracefully');
  await mongoose.disconnect();
  process.exit(0);
});

process.on('SIGINT', async () => {
  logger.info('SIGINT received, shutting down gracefully');
  await mongoose.disconnect();
  process.exit(0);
});

start();

module.exports = app;
```

---

## 9. Function Calling กับ LINE Actions

```javascript
// src/services/functionCalling.js
const openaiService = require('./openai');
const logger = require('../utils/logger');

// กำหนด Functions สำหรับ LINE Bot
const lineFunctions = [
  {
    name: 'get_weather',
    description: 'ดูข้อมูลสภาพอากาศของเมืองที่ระบุ',
    parameters: {
      type: 'object',
      properties: {
        city: {
          type: 'string',
          description: 'ชื่อเมืองหรือจังหวัด เช่น กรุงเทพ เชียงใหม่',
        },
        unit: {
          type: 'string',
          enum: ['celsius', 'fahrenheit'],
          description: 'หน่วยอุณหภูมิ',
        },
      },
      required: ['city'],
    },
  },
  {
    name: 'search_product',
    description: 'ค้นหาสินค้าในร้าน',
    parameters: {
      type: 'object',
      properties: {
        query: {
          type: 'string',
          description: 'คำค้นหาสินค้า',
        },
        category: {
          type: 'string',
          description: 'หมวดหมู่สินค้า',
        },
        max_price: {
          type: 'number',
          description: 'ราคาสูงสุด (บาท)',
        },
      },
      required: ['query'],
    },
  },
  {
    name: 'create_appointment',
    description: 'สร้างนัดหมาย',
    parameters: {
      type: 'object',
      properties: {
        service: {
          type: 'string',
          description: 'ประเภทบริการที่ต้องการ',
        },
        date: {
          type: 'string',
          description: 'วันที่ต้องการนัด (YYYY-MM-DD)',
        },
        time: {
          type: 'string',
          description: 'เวลาที่ต้องการ (HH:MM)',
        },
        notes: {
          type: 'string',
          description: 'หมายเหตุเพิ่มเติม',
        },
      },
      required: ['service', 'date', 'time'],
    },
  },
];

// ดำเนินการ Function
async function executeFunction(functionName, args) {
  switch (functionName) {
    case 'get_weather':
      return await getWeather(args.city, args.unit || 'celsius');
    case 'search_product':
      return await searchProduct(args.query, args.category, args.max_price);
    case 'create_appointment':
      return await createAppointment(args.service, args.date, args.time, args.notes);
    default:
      throw new Error(`Unknown function: ${functionName}`);
  }
}

// ตัวอย่าง Function implementations
async function getWeather(city, unit) {
  // ในระบบจริงจะเรียก Weather API
  logger.info('Getting weather', { city, unit });

  // Mock response
  return {
    city,
    temperature: 32,
    unit,
    condition: 'ร้อนและชื้น',
    humidity: 75,
    feelsLike: 38,
  };
}

async function searchProduct(query, category, maxPrice) {
  // ในระบบจริงจะค้นหาจาก Database
  logger.info('Searching product', { query, category, maxPrice });

  return {
    products: [
      {
        name: `สินค้าตัวอย่าง - ${query}`,
        price: 299,
        category: category || 'ทั่วไป',
        available: true,
      },
    ],
    total: 1,
  };
}

async function createAppointment(service, date, time, notes) {
  // ในระบบจริงจะบันทึกลง Database
  logger.info('Creating appointment', { service, date, time });

  return {
    success: true,
    appointmentId: `APT-${Date.now()}`,
    service,
    date,
    time,
    confirmationMessage: `นัดหมายของคุณถูกบันทึกแล้ว: ${service} วันที่ ${date} เวลา ${time}`,
  };
}

// Chat with Function Calling
async function chatWithFunctions(messages) {
  const response = await openaiService.chat(messages, {
    functions: lineFunctions,
  });

  // ถ้า AI ต้องการเรียก Function
  if (response.toolCalls && response.toolCalls.length > 0) {
    const updatedMessages = [...messages];

    // เพิ่ม Assistant message พร้อม tool_calls
    updatedMessages.push({
      role: 'assistant',
      content: response.content,
      tool_calls: response.toolCalls,
    });

    // ดำเนินการแต่ละ Function
    for (const toolCall of response.toolCalls) {
      if (toolCall.type === 'function') {
        const { name, arguments: argsStr } = toolCall.function;
        const args = JSON.parse(argsStr);

        let result;
        try {
          result = await executeFunction(name, args);
        } catch (error) {
          result = { error: error.message };
        }

        // เพิ่มผลลัพธ์
        updatedMessages.push({
          role: 'tool',
          tool_call_id: toolCall.id,
          content: JSON.stringify(result),
        });
      }
    }

    // เรียก OpenAI อีกครั้งพร้อมผลลัพธ์
    const finalResponse = await openaiService.chat(updatedMessages);
    return finalResponse.content;
  }

  return response.content;
}

module.exports = {
  lineFunctions,
  chatWithFunctions,
  executeFunction,
};
```

---

## 10. Python Implementation

```python
# app.py - Python Version ด้วย Flask

import os
import json
import logging
from datetime import datetime, timedelta
from typing import Optional, List, Dict, Any

from flask import Flask, request, abort
from linebot.v3 import WebhookHandler
from linebot.v3.messaging import (
    Configuration,
    ApiClient,
    MessagingApi,
    ReplyMessageRequest,
    TextMessage,
)
from linebot.v3.webhooks import MessageEvent, TextMessageContent, FollowEvent
from linebot.v3.exceptions import InvalidSignatureError
from openai import OpenAI
import redis
from pymongo import MongoClient
from dotenv import load_dotenv

load_dotenv()

# ตั้งค่า Logging
logging.basicConfig(
    level=getattr(logging, os.getenv('LOG_LEVEL', 'INFO')),
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)

app = Flask(__name__)

# ตั้งค่า LINE
line_config = Configuration(
    access_token=os.getenv('LINE_CHANNEL_ACCESS_TOKEN')
)
handler = WebhookHandler(os.getenv('LINE_CHANNEL_SECRET'))

# ตั้งค่า OpenAI
openai_client = OpenAI(api_key=os.getenv('OPENAI_API_KEY'))
OPENAI_MODEL = os.getenv('OPENAI_MODEL', 'gpt-4o')
MAX_TOKENS = int(os.getenv('OPENAI_MAX_TOKENS', 1000))
TEMPERATURE = float(os.getenv('OPENAI_TEMPERATURE', 0.7))

# ตั้งค่า Redis
redis_client = redis.from_url(os.getenv('REDIS_URL', 'redis://localhost:6379'))

# ตั้งค่า MongoDB
mongo_client = MongoClient(os.getenv('MONGODB_URI'))
db = mongo_client['line-openai-bot']
users_col = db['users']
conversations_col = db['conversations']


class ConversationManager:
    """จัดการประวัติการสนทนา"""

    SESSION_TTL = int(os.getenv('SESSION_TIMEOUT_MINUTES', 60)) * 60
    MAX_HISTORY = int(os.getenv('MAX_CONVERSATION_HISTORY', 20))

    @classmethod
    def get_or_create_session(cls, user_id: str) -> str:
        """รับหรือสร้าง Session ID"""
        session_key = f"session:{user_id}"
        session_id = redis_client.get(session_key)

        if session_id:
            redis_client.expire(session_key, cls.SESSION_TTL)
            return session_id.decode()

        import uuid
        session_id = str(uuid.uuid4())
        redis_client.setex(session_key, cls.SESSION_TTL, session_id)
        return session_id

    @classmethod
    def reset_session(cls, user_id: str) -> str:
        """รีเซ็ต Session"""
        session_key = f"session:{user_id}"
        old_session = redis_client.get(session_key)

        if old_session:
            conversations_col.update_one(
                {'sessionId': old_session.decode()},
                {'$set': {'isActive': False}}
            )

        import uuid
        new_session_id = str(uuid.uuid4())
        redis_client.setex(session_key, cls.SESSION_TTL, new_session_id)
        return new_session_id

    @classmethod
    def get_history(cls, user_id: str, session_id: str) -> List[Dict]:
        """ดึงประวัติการสนทนา"""
        cache_key = f"history:{session_id}"
        cached = redis_client.get(cache_key)

        if cached:
            return json.loads(cached)

        conversation = conversations_col.find_one({
            'lineUserId': user_id,
            'sessionId': session_id,
            'isActive': True
        })

        if not conversation:
            return []

        messages = conversation.get('messages', [])
        # เอาเฉพาะ N ข้อความล่าสุด
        recent = messages[-cls.MAX_HISTORY:]
        history = [{'role': m['role'], 'content': m['content']} for m in recent]

        # Cache ใน Redis
        redis_client.setex(cache_key, 300, json.dumps(history, ensure_ascii=False))
        return history

    @classmethod
    def save_message(cls, user_id: str, session_id: str,
                     role: str, content: str, metadata: Dict = None):
        """บันทึกข้อความ"""
        message_doc = {
            'role': role,
            'content': content,
            'timestamp': datetime.utcnow(),
            'metadata': metadata or {}
        }

        conversations_col.update_one(
            {'lineUserId': user_id, 'sessionId': session_id, 'isActive': True},
            {
                '$push': {'messages': message_doc},
                '$set': {'lastActivityAt': datetime.utcnow()},
                '$setOnInsert': {
                    'startedAt': datetime.utcnow(),
                    'totalTokens': 0
                }
            },
            upsert=True
        )

        # Invalidate cache
        redis_client.delete(f"history:{session_id}")


class OpenAIService:
    """บริการ OpenAI"""

    SYSTEM_PROMPT = """คุณคือ AI Assistant ที่ช่วยเหลือผู้ใช้ผ่าน LINE Official Account

คุณมีคุณสมบัติดังนี้:
- ตอบภาษาไทยได้อย่างคล่องแคล่ว
- เป็นมิตรและสุภาพ
- ให้ข้อมูลที่ถูกต้องและเป็นประโยชน์
- ตอบคำถามกระชับและชัดเจน

กฎการตอบ:
1. ตอบในภาษาเดียวกับที่ผู้ใช้ถาม
2. ถ้าไม่แน่ใจ ให้บอกว่าไม่แน่ใจ ไม่ใช่เดา
3. รักษาความปลอดภัยและความเป็นส่วนตัวของผู้ใช้"""

    @classmethod
    def chat(cls, messages: List[Dict], **kwargs) -> Dict:
        """ส่งข้อความไปยัง OpenAI"""
        start_time = datetime.utcnow()

        try:
            response = openai_client.chat.completions.create(
                model=kwargs.get('model', OPENAI_MODEL),
                messages=messages,
                max_tokens=kwargs.get('max_tokens', MAX_TOKENS),
                temperature=kwargs.get('temperature', TEMPERATURE),
            )

            processing_time = (datetime.utcnow() - start_time).total_seconds() * 1000

            usage = response.usage
            logger.info(
                f"OpenAI call: {usage.prompt_tokens}+{usage.completion_tokens} tokens, "
                f"{processing_time:.0f}ms"
            )

            return {
                'content': response.choices[0].message.content,
                'usage': {
                    'prompt_tokens': usage.prompt_tokens,
                    'completion_tokens': usage.completion_tokens,
                    'total_tokens': usage.total_tokens,
                },
                'finish_reason': response.choices[0].finish_reason,
                'processing_time': processing_time,
            }

        except Exception as e:
            logger.error(f"OpenAI error: {e}")
            raise cls._handle_error(e)

    @classmethod
    def _handle_error(cls, error) -> Exception:
        """จัดการ Error จาก OpenAI"""
        error_str = str(error).lower()

        if 'rate limit' in error_str:
            return Exception('ขณะนี้มีผู้ใช้งานมาก กรุณาลองใหม่อีกครั้ง')
        if 'context_length' in error_str:
            return Exception('ประวัติการสนทนายาวเกินไป กำลังรีเซ็ต...')
        if 'authentication' in error_str:
            return Exception('ไม่สามารถเชื่อมต่อกับ AI ได้ในขณะนี้')

        return Exception('เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง')

    @classmethod
    def build_messages(cls, user_id: str, session_id: str,
                       user_message: str) -> List[Dict]:
        """สร้าง Messages สำหรับ OpenAI"""
        history = ConversationManager.get_history(user_id, session_id)

        messages = [
            {'role': 'system', 'content': cls.SYSTEM_PROMPT},
            *history,
            {'role': 'user', 'content': user_message},
        ]

        # Prune context ถ้ายาวเกินไป
        return cls._prune_context(messages)

    @classmethod
    def _prune_context(cls, messages: List[Dict],
                       max_tokens: int = 8000) -> List[Dict]:
        """ลดขนาด Context"""
        # ประมาณ tokens (4 chars ≈ 1 token)
        def estimate_tokens(text: str) -> int:
            return len(text) // 4

        total = sum(estimate_tokens(m['content']) for m in messages)

        if total <= max_tokens:
            return messages

        # เก็บ system message และลบข้อความเก่า
        system_msgs = [m for m in messages if m['role'] == 'system']
        other_msgs = [m for m in messages if m['role'] != 'system']

        result = list(system_msgs)
        budget = max_tokens - sum(estimate_tokens(m['content']) for m in system_msgs)

        for msg in reversed(other_msgs):
            tokens = estimate_tokens(msg['content'])
            if budget - tokens < 0:
                break
            result.insert(len(system_msgs), msg)
            budget -= tokens

        return result


class RateLimiter:
    """จำกัด Rate การใช้งาน"""

    REQUESTS_PER_MINUTE = int(os.getenv('RATE_LIMIT_REQUESTS_PER_MINUTE', 10))

    @classmethod
    def check(cls, user_id: str) -> Dict:
        """ตรวจสอบ Rate Limit"""
        key = f"ratelimit:{user_id}"
        pipe = redis_client.pipeline()
        now = int(datetime.utcnow().timestamp() * 1000)
        window_start = now - 60000  # 1 นาทีที่แล้ว

        pipe.zremrangebyscore(key, '-inf', window_start)
        pipe.zcard(key)
        pipe.expire(key, 120)
        results = pipe.execute()
        current_count = results[1]

        if current_count >= cls.REQUESTS_PER_MINUTE:
            return {'allowed': False, 'remaining': 0}

        return {'allowed': True, 'remaining': cls.REQUESTS_PER_MINUTE - current_count - 1}

    @classmethod
    def record(cls, user_id: str):
        """บันทึกการใช้งาน"""
        key = f"ratelimit:{user_id}"
        now = int(datetime.utcnow().timestamp() * 1000)
        redis_client.zadd(key, {f"{now}:{os.urandom(4).hex()}": now})
        redis_client.expire(key, 120)


@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('X-Line-Signature', '')
    body = request.get_data(as_text=True)

    try:
        handler.handle(body, signature)
    except InvalidSignatureError:
        abort(400)

    return 'OK'


@handler.add(MessageEvent, message=TextMessageContent)
def handle_text_message(event):
    user_id = event.source.user_id
    text = event.message.text.strip()
    reply_token = event.reply_token

    # ตรวจสอบคำสั่งพิเศษ
    if text.lower() in ['/reset', '/ล้างประวัติ']:
        ConversationManager.reset_session(user_id)
        reply_text = '✅ ล้างประวัติการสนทนาแล้ว เริ่มต้นใหม่ได้เลย!'
    elif text.lower() in ['/help', '/ช่วยเหลือ']:
        reply_text = """🤖 คำสั่งที่ใช้ได้:

/reset - ล้างประวัติการสนทนา
/help - แสดงคำสั่งนี้

พิมพ์ข้อความที่ต้องการได้เลย!"""
    else:
        # ตรวจสอบ Rate Limit
        rate_check = RateLimiter.check(user_id)
        if not rate_check['allowed']:
            reply_text = 'คุณส่งข้อความถี่เกินไป กรุณารอ 1 นาที'
        else:
            try:
                session_id = ConversationManager.get_or_create_session(user_id)
                messages = OpenAIService.build_messages(user_id, session_id, text)
                response = OpenAIService.chat(messages)

                # บันทึกการสนทนา
                ConversationManager.save_message(user_id, session_id, 'user', text)
                ConversationManager.save_message(
                    user_id, session_id, 'assistant',
                    response['content'],
                    {'model': OPENAI_MODEL, 'processingTime': response['processing_time']}
                )

                # บันทึก Rate Limit
                RateLimiter.record(user_id)

                reply_text = response['content']
            except Exception as e:
                logger.error(f"Message processing error: {e}")
                reply_text = str(e)

    # ตอบกลับ
    with ApiClient(line_config) as api_client:
        line_bot_api = MessagingApi(api_client)
        line_bot_api.reply_message(
            ReplyMessageRequest(
                reply_token=reply_token,
                messages=[TextMessage(text=reply_text)]
            )
        )


@handler.add(FollowEvent)
def handle_follow(event):
    user_id = event.source.user_id
    logger.info(f"New follower: {user_id}")

    welcome_text = """สวัสดี! ยินดีต้อนรับสู่ AI Assistant 🎉

ฉันพร้อมช่วยเหลือคุณในทุกเรื่อง
พิมพ์ /help เพื่อดูคำสั่งทั้งหมด"""

    with ApiClient(line_config) as api_client:
        line_bot_api = MessagingApi(api_client)
        line_bot_api.reply_message(
            ReplyMessageRequest(
                reply_token=event.reply_token,
                messages=[TextMessage(text=welcome_text)]
            )
        )


@app.route('/health')
def health():
    return {'status': 'healthy', 'timestamp': datetime.utcnow().isoformat()}


if __name__ == '__main__':
    port = int(os.getenv('PORT', 3000))
    app.run(host='0.0.0.0', port=port, debug=os.getenv('NODE_ENV') != 'production')
```

---

## 11. Streaming Response Implementation

```javascript
// src/services/streamingResponse.js
// สำหรับ LINE ต้องใช้ Push Message แทน Reply เพราะ Reply มี timeout สั้น

class StreamingResponseService {
  constructor(lineClient, openaiService) {
    this.lineClient = lineClient;
    this.openaiService = openaiService;
  }

  // Stream และส่งแบบ Chunked
  async streamToUser(userId, messages, options = {}) {
    let fullResponse = '';
    let lastSentLength = 0;
    let chunkBuffer = '';
    const MIN_CHUNK_LENGTH = 50; // ส่งทุก 50 ตัวอักษร

    const onChunk = async (chunk, isDone) => {
      if (isDone) {
        // ส่งส่วนที่เหลือ
        if (chunkBuffer.trim()) {
          await this.lineClient.pushMessage({
            to: userId,
            messages: [{ type: 'text', text: chunkBuffer }],
          });
        }
        return;
      }

      fullResponse += chunk;
      chunkBuffer += chunk;

      // ส่งเมื่อ buffer ใหญ่พอ หรือมีจุดสิ้นสุดประโยค
      const shouldSend = chunkBuffer.length >= MIN_CHUNK_LENGTH ||
        chunkBuffer.includes('\n\n') ||
        (chunkBuffer.length > 20 && /[.!?।।]/.test(chunkBuffer.slice(-5)));

      if (shouldSend) {
        try {
          await this.lineClient.pushMessage({
            to: userId,
            messages: [{ type: 'text', text: chunkBuffer }],
          });
          chunkBuffer = '';
        } catch (error) {
          // ถ้าส่งไม่สำเร็จ เก็บไว้ต่อ
        }
      }
    };

    const result = await this.openaiService.chatStream(messages, onChunk, options);

    return {
      fullContent: fullResponse,
      usage: result.usage,
      cost: result.cost,
    };
  }

  // Typing Indicator แบบ Progressive
  async progressiveResponse(userId, messages, options = {}) {
    // ส่ง "กำลังพิมพ์..." ก่อน
    await this.lineClient.pushMessage({
      to: userId,
      messages: [{ type: 'text', text: '⌛ กำลังประมวลผล...' }],
    });

    const response = await this.openaiService.chat(messages, options);

    // ส่งคำตอบจริง
    await this.lineClient.pushMessage({
      to: userId,
      messages: [{ type: 'text', text: response.content }],
    });

    return response;
  }
}

module.exports = StreamingResponseService;
```

---

## 12. Cost Management

```javascript
// src/services/costManager.js
const Redis = require('ioredis');
const config = require('../config');
const logger = require('../utils/logger');

class CostManager {
  constructor() {
    this.redis = new Redis(config.database.redisUrl);
  }

  // คำนวณราคาแบบ Real-time
  calculateCost(promptTokens, completionTokens, model = 'gpt-4o') {
    const pricing = {
      'gpt-4o': { prompt: 0.005, completion: 0.015 },
      'gpt-4o-mini': { prompt: 0.000150, completion: 0.000600 },
      'gpt-4-turbo': { prompt: 0.01, completion: 0.03 },
      'gpt-3.5-turbo': { prompt: 0.0005, completion: 0.0015 },
    };

    const p = pricing[model] || pricing['gpt-4o'];
    return {
      promptCost: (promptTokens / 1000) * p.prompt,
      completionCost: (completionTokens / 1000) * p.completion,
      total: (promptTokens / 1000) * p.prompt + (completionTokens / 1000) * p.completion,
    };
  }

  // เลือก Model ตามต้นทุน
  selectModel(messageLength, requiresReasoning = false) {
    if (requiresReasoning || messageLength > 500) {
      return 'gpt-4o';
    }
    if (messageLength > 200) {
      return 'gpt-4o-mini';
    }
    return 'gpt-4o-mini'; // ใช้ mini สำหรับข้อความสั้น
  }

  // ตรวจสอบงบประมาณรายวัน
  async checkDailyBudget() {
    const today = new Date().toISOString().split('T')[0];
    const used = parseFloat(await this.redis.get(`cost:daily:${today}`) || '0');
    const remaining = config.cost.maxDailyCostUsd - used;

    return {
      used,
      remaining,
      limit: config.cost.maxDailyCostUsd,
      percentage: (used / config.cost.maxDailyCostUsd) * 100,
      isNearLimit: remaining < config.cost.maxDailyCostUsd * 0.2,
      isExceeded: used >= config.cost.maxDailyCostUsd,
    };
  }

  // บันทึกต้นทุน
  async recordCost(userId, costUsd, tokens, model) {
    const today = new Date().toISOString().split('T')[0];

    // อัพเดท Global Daily Cost
    await this.redis.incrbyfloat(`cost:daily:${today}`, costUsd);
    await this.redis.expire(`cost:daily:${today}`, 48 * 3600);

    // อัพเดท User Daily Cost
    await this.redis.incrbyfloat(`cost:user:${userId}:${today}`, costUsd);
    await this.redis.expire(`cost:user:${userId}:${today}`, 48 * 3600);

    logger.info('Cost recorded', { userId, costUsd, tokens, model });
  }

  // รายงานต้นทุน
  async getCostReport(days = 7) {
    const report = [];
    const today = new Date();

    for (let i = 0; i < days; i++) {
      const date = new Date(today);
      date.setDate(today.getDate() - i);
      const dateStr = date.toISOString().split('T')[0];

      const cost = parseFloat(
        await this.redis.get(`cost:daily:${dateStr}`) || '0'
      );

      report.push({ date: dateStr, cost });
    }

    return report;
  }

  // คำแนะนำประหยัดต้นทุน
  getOptimizationTips(stats) {
    const tips = [];

    if (stats.averageResponseTokens > 800) {
      tips.push('ลด max_tokens เพื่อประหยัดต้นทุน');
    }

    if (stats.model === 'gpt-4o' && stats.averageComplexity < 0.5) {
      tips.push('ใช้ gpt-4o-mini สำหรับคำถามง่ายๆ');
    }

    if (stats.cacheHitRate < 0.3) {
      tips.push('เพิ่ม Caching สำหรับคำตอบที่ใช้บ่อย');
    }

    return tips;
  }
}

module.exports = new CostManager();
```

---

## 13. Monitoring และ Logging

```javascript
// src/services/monitoring.js
const logger = require('../utils/logger');

class MonitoringService {
  // บันทึก Metrics
  static trackMetric(name, value, tags = {}) {
    logger.info('metric', { name, value, tags, timestamp: Date.now() });
  }

  // บันทึก API Response Time
  static trackResponseTime(userId, processingTime, model) {
    this.trackMetric('api.response_time', processingTime, {
      userId,
      model,
    });

    if (processingTime > 10000) {
      logger.warn('Slow API response', { userId, processingTime, model });
    }
  }

  // บันทึก Error Rate
  static trackError(error, context = {}) {
    logger.error('Application error', {
      error: error.message,
      stack: error.stack,
      ...context,
    });
  }

  // Dashboard Stats
  static async getDashboardStats(User, Conversation) {
    const today = new Date();
    today.setHours(0, 0, 0, 0);

    const [
      totalUsers,
      activeToday,
      totalConversations,
      recentErrors,
    ] = await Promise.all([
      User.countDocuments(),
      User.countDocuments({
        'stats.lastMessageAt': { $gte: today },
      }),
      Conversation.countDocuments({ isActive: true }),
      Promise.resolve([]), // ใส่ Error tracking ที่นี่
    ]);

    return {
      users: { total: totalUsers, activeToday },
      conversations: { active: totalConversations },
      timestamp: new Date().toISOString(),
    };
  }
}

module.exports = MonitoringService;
```

---

## 14. Testing

```javascript
// tests/unit/openai.test.js
const { OpenAI } = require('openai');
const openaiService = require('../../src/services/openai');

// Mock OpenAI
jest.mock('openai');

describe('OpenAIService', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('chat', () => {
    it('should return response from OpenAI', async () => {
      const mockResponse = {
        choices: [{
          message: { content: 'สวัสดี!' },
          finish_reason: 'stop',
        }],
        usage: {
          prompt_tokens: 10,
          completion_tokens: 5,
          total_tokens: 15,
        },
      };

      OpenAI.prototype.chat = {
        completions: {
          create: jest.fn().mockResolvedValue(mockResponse),
        },
      };

      const messages = [{ role: 'user', content: 'สวัสดี' }];
      const result = await openaiService.chat(messages);

      expect(result.content).toBe('สวัสดี!');
      expect(result.usage.total_tokens).toBe(15);
    });

    it('should handle rate limit error', async () => {
      OpenAI.prototype.chat = {
        completions: {
          create: jest.fn().mockRejectedValue(
            Object.assign(new Error('Rate limit exceeded'), { status: 429 })
          ),
        },
      };

      const messages = [{ role: 'user', content: 'test' }];
      await expect(openaiService.chat(messages)).rejects.toThrow(
        'ขณะนี้มีผู้ใช้งานมาก'
      );
    });
  });

  describe('buildSystemPrompt', () => {
    it('should build concise prompt for concise preference', () => {
      const user = { preferences: { responseStyle: 'concise' } };
      const prompt = openaiService.buildSystemPrompt(user);
      expect(prompt).toContain('กระชับ');
    });

    it('should use custom prompt if provided', () => {
      const customPrompt = 'Custom system prompt';
      const prompt = openaiService.buildSystemPrompt(null, customPrompt);
      expect(prompt).toBe(customPrompt);
    });
  });
});
```

---

## 15. Docker และ Deployment

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app

# ติดตั้ง dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy source
COPY src/ ./src/
COPY .env.example .env.example

# สร้าง log directory
RUN mkdir -p logs

# ตั้งค่า User ที่ไม่ใช่ root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001 && \
    chown -R nodejs:nodejs /app

USER nodejs

EXPOSE 3000

HEALTHCHECK --interval=30s --timeout=10s --start-period=5s \
  CMD curl -f http://localhost:3000/health || exit 1

CMD ["node", "src/index.js"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  app:
    build: .
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    env_file:
      - .env
    depends_on:
      - mongodb
      - redis
    restart: unless-stopped
    volumes:
      - ./logs:/app/logs

  mongodb:
    image: mongo:7
    ports:
      - "27017:27017"
    volumes:
      - mongodb_data:/data/db
    environment:
      MONGO_INITDB_DATABASE: line-openai-bot
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - app
    restart: unless-stopped

volumes:
  mongodb_data:
  redis_data:
```

---

## 16. Sequence Diagram การทำงาน

```
ผู้ใช้          LINE Server    Node.js Server    Redis    MongoDB    OpenAI
  │                  │                │             │         │          │
  │──ส่งข้อความ──►  │                │             │         │          │
  │                  │──Webhook POST─►│             │         │          │
  │                  │                │──check rate─►│         │          │
  │                  │                │◄─ok──────── │         │          │
  │                  │                │──get session►│         │          │
  │                  │                │◄─session_id─ │         │          │
  │                  │                │──get history►│         │          │
  │                  │                │◄─messages──  │         │          │
  │                  │                │──load user───────────► │          │
  │                  │                │◄─user data───────────  │          │
  │                  │                │──────────────────────────────────►│
  │                  │                │              │         │  API Call │
  │                  │                │◄─────────────────────────────── AI│
  │                  │                │──save msgs───────────► │          │
  │                  │                │──record cost►│         │          │
  │                  │──reply message─►              │         │          │
  │◄─ข้อความตอบ── │                │             │         │          │
  │                  │                │             │         │          │
```

---

## 17. กลยุทธ์ประหยัดต้นทุน

### 17.1 การเลือก Model อัจฉริยะ

```javascript
// src/utils/modelSelector.js
class ModelSelector {
  selectModel(message, context = {}) {
    const complexity = this.analyzeComplexity(message);

    // ข้อความซับซ้อนหรือต้องการความแม่นยำสูง
    if (complexity.score > 0.7 || context.requiresExpertise) {
      return { model: 'gpt-4o', reason: 'high complexity' };
    }

    // ข้อความทั่วไป
    if (complexity.score > 0.3) {
      return { model: 'gpt-4o-mini', reason: 'medium complexity' };
    }

    // ข้อความง่าย
    return { model: 'gpt-4o-mini', reason: 'low complexity' };
  }

  analyzeComplexity(message) {
    const factors = {
      length: message.length > 200 ? 0.3 : 0,
      hasCode: /```|function|class|def |import /.test(message) ? 0.4 : 0,
      hasNumbers: /\d+[%$]|\d+\.\d+/.test(message) ? 0.1 : 0,
      hasTechnical: /API|HTTP|SQL|JSON|algorithm/i.test(message) ? 0.2 : 0,
    };

    const score = Object.values(factors).reduce((a, b) => a + b, 0);
    return { score: Math.min(score, 1), factors };
  }
}

module.exports = new ModelSelector();
```

### 17.2 Response Caching

```javascript
// src/utils/responseCache.js
class ResponseCache {
  constructor(redis) {
    this.redis = redis;
    this.TTL = 3600; // 1 ชั่วโมง
  }

  generateKey(message, systemPrompt) {
    const crypto = require('crypto');
    const content = `${systemPrompt}:${message}`;
    return `cache:response:${crypto.createHash('md5').update(content).digest('hex')}`;
  }

  async get(message, systemPrompt) {
    const key = this.generateKey(message, systemPrompt);
    const cached = await this.redis.get(key);
    return cached ? JSON.parse(cached) : null;
  }

  async set(message, systemPrompt, response) {
    const key = this.generateKey(message, systemPrompt);
    await this.redis.setex(key, this.TTL, JSON.stringify(response));
  }

  // ตรวจสอบว่าควร cache หรือไม่
  shouldCache(message) {
    // ไม่ cache ข้อความที่มีบริบทเฉพาะตัว
    const personalPatterns = [
      /ฉัน|ผม|หนู|เรา/,
      /วันนี้|ตอนนี้|เมื่อกี้/,
      /ของฉัน|ของผม/,
    ];

    return !personalPatterns.some((p) => p.test(message));
  }
}

module.exports = ResponseCache;
```

---

## สรุปบทที่ 61

ในบทนี้เราได้เรียนรู้:

1. **OpenAI API Setup** - การตั้งค่าและการยืนยันตัวตน
2. **Conversation Management** - การจัดการประวัติสนทนาด้วย Redis และ MongoDB
3. **Context Window** - การควบคุมขนาด Context เพื่อประสิทธิภาพและต้นทุน
4. **Rate Limiting** - การป้องกัน Abuse ด้วย Sliding Window Algorithm
5. **Cost Management** - การติดตามและควบคุมต้นทุน
6. **Function Calling** - การให้ AI เรียกใช้ฟังก์ชันจริง
7. **Streaming** - การส่งคำตอบแบบ Real-time
8. **Full Node.js & Python** - Implementation ครบถ้วนทั้งสองภาษา

### คำแนะนำ Production

- ใช้ **gpt-4o-mini** สำหรับข้อความทั่วไปเพื่อประหยัดต้นทุน
- ตั้ง **Daily Budget Limit** เพื่อควบคุมค่าใช้จ่าย
- ใช้ **Response Caching** สำหรับคำถามซ้ำ
- Monitor **Token Usage** อย่างสม่ำเสมอ
- เก็บ **Conversation History** ไม่เกิน 20 ข้อความ

---

*ต่อไป: Part 62 - NLP Processing สำหรับ LINE Bot*
