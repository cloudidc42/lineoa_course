# Part 65: Enterprise Customer Service Bot

## บทนำ

Customer Service Bot ระดับ Enterprise คือระบบที่รวม AI, Human Agent และ Business Logic เข้าไว้ด้วยกัน เพื่อให้บริการลูกค้าได้อย่างมีประสิทธิภาพ บทนี้จะออกแบบและสร้างระบบ Customer Service Bot ที่สมบูรณ์ พร้อม Knowledge Base, Escalation, SLA Tracking และ Performance Metrics

## เนื้อหาที่จะเรียนรู้

1. Architecture Design
2. Knowledge Base Management
3. FAQ Bot
4. Escalation to Human Agent
5. Hybrid AI + Human Model
6. Chat Routing System
7. SLA Tracking
8. Performance Metrics
9. Agent Handoff Protocol
10. Customer Satisfaction Tracking
11. Complete System Implementation

---

## สถาปัตยกรรมระบบ

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Enterprise CS Bot Architecture                    │
│                                                                      │
│  LINE Users                                                          │
│  ┌────┐ ┌────┐ ┌────┐                                               │
│  │U1  │ │U2  │ │U3  │                                               │
│  └────┘ └────┘ └────┘                                               │
│     │       │       │                                                │
│     └───────┴───────┘                                               │
│             │                                                        │
│             ▼                                                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    API Gateway + Load Balancer               │   │
│  └──────────────────────────────────────────────────────────────┘   │
│             │                                                        │
│             ▼                                                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    LINE Webhook Handler                      │   │
│  └──────────────────────────────────────────────────────────────┘   │
│             │                                                        │
│             ▼                                                        │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Chat Router                               │   │
│  │                                                              │   │
│  │  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │   │
│  │  │  AI Bot      │    │  FAQ Engine  │    │  Human Agent │   │   │
│  │  │  (OpenAI)    │    │              │    │  (Live Chat) │   │   │
│  │  └──────────────┘    └──────────────┘    └──────────────┘   │   │
│  └──────────────────────────────────────────────────────────────┘   │
│             │                                                        │
│             ▼                                                        │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │                     Data Layer                                 │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────┐ │ │
│  │  │ Redis    │  │ MongoDB  │  │ Postgres │  │ Elasticsearch│ │ │
│  │  │ Sessions │  │ History  │  │ Tickets  │  │ Knowledge    │ │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────────┘ │ │
│  └────────────────────────────────────────────────────────────────┘ │
│             │                                                        │
│             ▼                                                        │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │              Analytics & Monitoring                            │ │
│  │  ┌──────────────┐  ┌───────────────┐  ┌──────────────────┐   │ │
│  │  │  SLA Tracker │  │  CSAT Tracker │  │  Performance     │   │ │
│  │  │              │  │               │  │  Dashboard       │   │ │
│  │  └──────────────┘  └───────────────┘  └──────────────────┘   │ │
│  └────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────┘
         │                    │                    │
         ▼                    ▼                    ▼
  ┌─────────────┐    ┌─────────────┐    ┌────────────────────┐
  │ Human Agent │    │  Supervisor │    │  Admin Dashboard   │
  │  Dashboard  │    │  Dashboard  │    │  (Analytics)       │
  └─────────────┘    └─────────────┘    └────────────────────┘
```

---

## 1. ติดตั้งและตั้งค่า

```bash
npm init -y

# Core
npm install @line/bot-sdk openai express dotenv

# Database
npm install mongoose redis ioredis pg sequelize
npm install @elastic/elasticsearch

# Utilities
npm install uuid winston socket.io dayjs
npm install node-schedule nodemailer

# API
npm install helmet cors morgan express-validator
npm install multer axios

# Real-time
npm install socket.io

# Testing
npm install -D jest supertest nodemon
```

```env
# .env
LINE_CHANNEL_ACCESS_TOKEN=your_token
LINE_CHANNEL_SECRET=your_secret

# OpenAI
OPENAI_API_KEY=your_openai_key
OPENAI_MODEL=gpt-4o

# Database
MONGODB_URI=mongodb://localhost:27017/cs-bot
REDIS_URL=redis://localhost:6379
POSTGRES_URL=postgresql://user:password@localhost:5432/cs-bot

# Elasticsearch (Knowledge Base)
ELASTICSEARCH_URL=http://localhost:9200
ELASTICSEARCH_INDEX=knowledge-base

# SLA Settings
SLA_FIRST_RESPONSE_MINUTES=5
SLA_RESOLUTION_HOURS=24
SLA_URGENT_MINUTES=2

# Agent Settings
MAX_CONCURRENT_CHATS_PER_AGENT=5
AGENT_IDLE_TIMEOUT_MINUTES=10

# Email Notifications
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=support@example.com
SMTP_PASS=your_app_password
NOTIFICATION_EMAIL=manager@example.com

PORT=3000
WEBSOCKET_PORT=3001
```

---

## 2. Database Models

### 2.1 Ticket Model

```javascript
// src/models/ticket.js
const mongoose = require('mongoose');

const messageSchema = new mongoose.Schema({
  sender: {
    type: String,
    enum: ['customer', 'bot', 'agent', 'system'],
    required: true,
  },
  senderName: String,
  content: String,
  contentType: {
    type: String,
    enum: ['text', 'image', 'file', 'system_note'],
    default: 'text',
  },
  attachments: [{
    url: String,
    type: String,
    filename: String,
  }],
  timestamp: { type: Date, default: Date.now },
  isRead: { type: Boolean, default: false },
  metadata: mongoose.Schema.Types.Mixed,
});

const slaSchema = new mongoose.Schema({
  firstResponseDue: Date,
  resolutionDue: Date,
  firstResponseAt: Date,
  resolvedAt: Date,
  firstResponseBreached: { type: Boolean, default: false },
  resolutionBreached: { type: Boolean, default: false },
  pausedAt: Date,
  totalPausedTime: { type: Number, default: 0 }, // milliseconds
});

const ticketSchema = new mongoose.Schema({
  ticketId: {
    type: String,
    unique: true,
    required: true,
  },
  lineUserId: {
    type: String,
    required: true,
    index: true,
  },
  customerName: String,
  customerPhone: String,
  customerEmail: String,

  status: {
    type: String,
    enum: ['new', 'open', 'pending', 'waiting_customer', 'escalated', 'resolved', 'closed'],
    default: 'new',
    index: true,
  },

  priority: {
    type: String,
    enum: ['low', 'medium', 'high', 'urgent'],
    default: 'medium',
    index: true,
  },

  category: {
    type: String,
    enum: ['general', 'billing', 'technical', 'shipping', 'product', 'other'],
    default: 'general',
    index: true,
  },

  tags: [String],

  subject: String,
  description: String,

  messages: [messageSchema],

  assignedTo: {
    type: mongoose.Schema.Types.ObjectId,
    ref: 'Agent',
    index: true,
  },
  assignedAt: Date,

  createdBy: {
    type: String, // 'bot' or 'customer'
    default: 'customer',
  },

  sla: slaSchema,

  escalationHistory: [{
    from: String,
    to: String,
    reason: String,
    timestamp: { type: Date, default: Date.now },
  }],

  satisfaction: {
    score: Number, // 1-5
    comment: String,
    ratedAt: Date,
  },

  resolutionNote: String,
  internalNotes: [{
    content: String,
    agentId: mongoose.Schema.Types.ObjectId,
    timestamp: { type: Date, default: Date.now },
  }],

  firstContactResolution: { type: Boolean, default: false },
  reopenCount: { type: Number, default: 0 },

  metadata: mongoose.Schema.Types.Mixed,

  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now },
  closedAt: Date,
});

ticketSchema.pre('save', function (next) {
  this.updatedAt = Date.now();
  next();
});

ticketSchema.index({ lineUserId: 1, status: 1 });
ticketSchema.index({ assignedTo: 1, status: 1 });
ticketSchema.index({ createdAt: -1 });
ticketSchema.index({ 'sla.firstResponseDue': 1 });
ticketSchema.index({ 'sla.resolutionDue': 1 });

module.exports = mongoose.model('Ticket', ticketSchema);
```

### 2.2 Agent Model

```javascript
// src/models/agent.js
const mongoose = require('mongoose');
const bcrypt = require('bcryptjs');

const agentSchema = new mongoose.Schema({
  username: { type: String, unique: true, required: true },
  email: { type: String, unique: true, required: true },
  password: { type: String, required: true },
  displayName: { type: String, required: true },
  avatar: String,
  lineUserId: String, // LINE ID ของ Agent (ถ้ามี)

  role: {
    type: String,
    enum: ['agent', 'senior_agent', 'supervisor', 'admin'],
    default: 'agent',
  },

  department: String,
  skills: [String],
  languages: [{ type: String, default: 'th' }],

  status: {
    type: String,
    enum: ['online', 'busy', 'away', 'offline'],
    default: 'offline',
    index: true,
  },

  availability: {
    maxConcurrentChats: { type: Number, default: 5 },
    currentChats: { type: Number, default: 0 },
    isAvailable: { type: Boolean, default: false },
  },

  stats: {
    totalTicketsHandled: { type: Number, default: 0 },
    totalMessagessSent: { type: Number, default: 0 },
    averageResponseTimeMs: { type: Number, default: 0 },
    averageSatisfactionScore: { type: Number, default: 0 },
    firstContactResolutionRate: { type: Number, default: 0 },
  },

  settings: {
    notifications: {
      newTicket: { type: Boolean, default: true },
      slaWarning: { type: Boolean, default: true },
      escalation: { type: Boolean, default: true },
    },
    autoAccept: { type: Boolean, default: false },
  },

  lastActiveAt: Date,
  createdAt: { type: Date, default: Date.now },
});

agentSchema.pre('save', async function (next) {
  if (this.isModified('password')) {
    this.password = await bcrypt.hash(this.password, 12);
  }
  next();
});

agentSchema.methods.comparePassword = async function (password) {
  return bcrypt.compare(password, this.password);
};

agentSchema.methods.isAvailableForNewChat = function () {
  return (
    this.status === 'online' &&
    this.availability.currentChats < this.availability.maxConcurrentChats
  );
};

module.exports = mongoose.model('Agent', agentSchema);
```

### 2.3 Knowledge Base Model

```javascript
// src/models/knowledgeBase.js
const mongoose = require('mongoose');

const knowledgeBaseSchema = new mongoose.Schema({
  title: { type: String, required: true, index: true },
  content: { type: String, required: true },
  summary: String,
  category: {
    type: String,
    index: true,
  },
  tags: [String],
  keywords: [String],

  // สำหรับ FAQ
  isFaq: { type: Boolean, default: false },
  faqQuestion: String,
  faqAnswer: String,

  views: { type: Number, default: 0 },
  helpfulVotes: { type: Number, default: 0 },
  unhelpfulVotes: { type: Number, default: 0 },

  isPublished: { type: Boolean, default: true, index: true },
  version: { type: Number, default: 1 },

  createdBy: { type: mongoose.Schema.Types.ObjectId, ref: 'Agent' },
  updatedBy: { type: mongoose.Schema.Types.ObjectId, ref: 'Agent' },

  createdAt: { type: Date, default: Date.now },
  updatedAt: { type: Date, default: Date.now },
});

knowledgeBaseSchema.index({ title: 'text', content: 'text', tags: 'text' });

module.exports = mongoose.model('KnowledgeBase', knowledgeBaseSchema);
```

---

## 3. Knowledge Base Service

```javascript
// src/services/knowledgeBase.js
const KnowledgeBase = require('../models/knowledgeBase');
const { Client: ElasticsearchClient } = require('@elastic/elasticsearch');
const openai = require('openai');
const logger = require('../utils/logger');

class KnowledgeBaseService {
  constructor() {
    this.es = new ElasticsearchClient({ node: process.env.ELASTICSEARCH_URL });
    this.openaiClient = new openai({ apiKey: process.env.OPENAI_API_KEY });
    this.indexName = process.env.ELASTICSEARCH_INDEX || 'knowledge-base';
  }

  async initialize() {
    await this.createIndex();
  }

  // สร้าง Elasticsearch Index
  async createIndex() {
    const exists = await this.es.indices.exists({ index: this.indexName });

    if (!exists) {
      await this.es.indices.create({
        index: this.indexName,
        body: {
          settings: {
            analysis: {
              analyzer: {
                thai_analyzer: {
                  tokenizer: 'standard',
                  filter: ['lowercase', 'stop'],
                },
              },
            },
          },
          mappings: {
            properties: {
              title: { type: 'text', analyzer: 'thai_analyzer' },
              content: { type: 'text', analyzer: 'thai_analyzer' },
              summary: { type: 'text' },
              category: { type: 'keyword' },
              tags: { type: 'keyword' },
              keywords: { type: 'keyword' },
              isFaq: { type: 'boolean' },
              faqQuestion: { type: 'text', analyzer: 'thai_analyzer' },
              faqAnswer: { type: 'text' },
              embedding: { type: 'dense_vector', dims: 1536 }, // OpenAI embedding
              isPublished: { type: 'boolean' },
              createdAt: { type: 'date' },
            },
          },
        },
      });
      logger.info('Elasticsearch index created', { index: this.indexName });
    }
  }

  // เพิ่มหรืออัพเดทบทความ
  async upsertArticle(articleData) {
    const article = await KnowledgeBase.findOneAndUpdate(
      { _id: articleData._id },
      articleData,
      { upsert: true, new: true }
    );

    // สร้าง Embedding สำหรับ Semantic Search
    const embedding = await this.createEmbedding(
      `${article.title} ${article.content}`
    );

    // Index ใน Elasticsearch
    await this.es.index({
      index: this.indexName,
      id: article._id.toString(),
      body: {
        title: article.title,
        content: article.content,
        summary: article.summary,
        category: article.category,
        tags: article.tags,
        keywords: article.keywords,
        isFaq: article.isFaq,
        faqQuestion: article.faqQuestion,
        faqAnswer: article.faqAnswer,
        embedding,
        isPublished: article.isPublished,
        createdAt: article.createdAt,
      },
    });

    return article;
  }

  // ค้นหาด้วย Full-text Search
  async search(query, options = {}) {
    const { size = 5, category = null, minScore = 0.5 } = options;

    const must = [
      {
        multi_match: {
          query,
          fields: ['title^3', 'faqQuestion^3', 'content', 'summary', 'tags'],
          type: 'best_fields',
          fuzziness: 'AUTO',
        },
      },
      { term: { isPublished: true } },
    ];

    if (category) {
      must.push({ term: { category } });
    }

    const response = await this.es.search({
      index: this.indexName,
      body: {
        query: { bool: { must } },
        size,
        min_score: minScore,
        highlight: {
          fields: {
            content: { number_of_fragments: 2 },
            faqQuestion: {},
          },
        },
      },
    });

    return response.hits.hits.map((hit) => ({
      id: hit._id,
      score: hit._score,
      title: hit._source.title,
      content: hit._source.content,
      summary: hit._source.summary,
      faqQuestion: hit._source.faqQuestion,
      faqAnswer: hit._source.faqAnswer,
      category: hit._source.category,
      highlight: hit.highlight,
    }));
  }

  // Semantic Search ด้วย Embedding
  async semanticSearch(query, options = {}) {
    const { size = 5 } = options;

    const queryEmbedding = await this.createEmbedding(query);

    const response = await this.es.search({
      index: this.indexName,
      body: {
        query: {
          script_score: {
            query: { term: { isPublished: true } },
            script: {
              source: "cosineSimilarity(params.query_vector, 'embedding') + 1.0",
              params: { query_vector: queryEmbedding },
            },
          },
        },
        size,
      },
    });

    return response.hits.hits.map((hit) => ({
      id: hit._id,
      score: hit._score - 1, // Convert back to cosine similarity
      title: hit._source.title,
      content: hit._source.content,
      faqQuestion: hit._source.faqQuestion,
      faqAnswer: hit._source.faqAnswer,
    }));
  }

  // Hybrid Search (Text + Semantic)
  async hybridSearch(query, options = {}) {
    const [textResults, semanticResults] = await Promise.all([
      this.search(query, { ...options, size: 10 }),
      this.semanticSearch(query, { ...options, size: 10 }),
    ]);

    // รวมผลลัพธ์และ normalize scores
    const combined = new Map();

    textResults.forEach((r, i) => {
      combined.set(r.id, {
        ...r,
        textRank: i,
        textScore: r.score,
        combinedScore: r.score * 0.4,
      });
    });

    semanticResults.forEach((r, i) => {
      if (combined.has(r.id)) {
        const existing = combined.get(r.id);
        existing.semanticRank = i;
        existing.semanticScore = r.score;
        existing.combinedScore += r.score * 0.6;
      } else {
        combined.set(r.id, {
          ...r,
          semanticRank: i,
          semanticScore: r.score,
          combinedScore: r.score * 0.6,
        });
      }
    });

    return Array.from(combined.values())
      .sort((a, b) => b.combinedScore - a.combinedScore)
      .slice(0, options.size || 5);
  }

  // สร้าง Embedding
  async createEmbedding(text) {
    const response = await this.openaiClient.embeddings.create({
      model: 'text-embedding-3-small',
      input: text.substring(0, 8000),
    });
    return response.data[0].embedding;
  }

  // ค้นหา FAQ
  async findFAQ(query) {
    const results = await this.hybridSearch(query, {
      size: 3,
      minScore: 0.6,
    });

    return results.filter((r) => r.faqAnswer);
  }

  // สร้างคำตอบจาก Knowledge Base
  async generateAnswer(query, userId) {
    const relevantDocs = await this.hybridSearch(query, { size: 5 });

    if (relevantDocs.length === 0) {
      return null;
    }

    // สร้าง Context จากเอกสาร
    const context = relevantDocs
      .slice(0, 3)
      .map((doc, i) => `[${i + 1}] ${doc.title}\n${doc.content}`)
      .join('\n\n---\n\n');

    const prompt = [
      {
        role: 'system',
        content: `คุณคือ Customer Service AI สำหรับ [ชื่อบริษัท]
        
ตอบคำถามโดยใช้ข้อมูลจาก Knowledge Base ที่ให้มา
ถ้าหาคำตอบไม่ได้ให้แจ้งว่าจะส่งต่อให้เจ้าหน้าที่
ตอบภาษาไทย กระชับ ชัดเจน และเป็นมิตร`,
      },
      {
        role: 'user',
        content: `Context:\n${context}\n\nคำถาม: ${query}`,
      },
    ];

    const response = await this.openaiClient.chat.completions.create({
      model: process.env.OPENAI_MODEL,
      messages: prompt,
      max_tokens: 500,
    });

    // บันทึก View Count
    for (const doc of relevantDocs.slice(0, 3)) {
      await KnowledgeBase.updateOne({ _id: doc.id }, { $inc: { views: 1 } });
    }

    return {
      answer: response.choices[0].message.content,
      sources: relevantDocs.slice(0, 3).map((d) => d.title),
      confidence: relevantDocs[0]?.combinedScore || 0,
    };
  }
}

module.exports = new KnowledgeBaseService();
```

---

## 4. Chat Routing Service

```javascript
// src/services/chatRouter.js
const Agent = require('../models/agent');
const Ticket = require('../models/ticket');
const Redis = require('ioredis');
const logger = require('../utils/logger');

class ChatRouter {
  constructor() {
    this.redis = new Redis(process.env.REDIS_URL);
    this.routingStrategies = {
      round_robin: this.roundRobinRoute.bind(this),
      least_busy: this.leastBusyRoute.bind(this),
      skill_based: this.skillBasedRoute.bind(this),
      priority_based: this.priorityBasedRoute.bind(this),
    };
  }

  // เลือก Agent สำหรับ Ticket
  async assignAgent(ticket) {
    const strategy = await this.redis.get('routing:strategy') || 'least_busy';
    const routeFn = this.routingStrategies[strategy];

    if (!routeFn) {
      throw new Error(`Unknown routing strategy: ${strategy}`);
    }

    const agent = await routeFn(ticket);

    if (agent) {
      await this.assignTicketToAgent(ticket, agent);
      return agent;
    }

    // ไม่มี Agent ว่าง - ใส่ Queue
    await this.addToQueue(ticket);
    return null;
  }

  // Round Robin Strategy
  async roundRobinRoute(ticket) {
    const agents = await Agent.find({
      status: 'online',
      'availability.isAvailable': true,
      'availability.currentChats': {
        $lt: process.env.MAX_CONCURRENT_CHATS_PER_AGENT || 5,
      },
    }).sort({ assignedAt: 1 }); // เรียงตาม Assignment เก่าสุด

    return agents[0] || null;
  }

  // Least Busy Strategy
  async leastBusyRoute(ticket) {
    const agents = await Agent.find({
      status: 'online',
      'availability.isAvailable': true,
    }).sort({ 'availability.currentChats': 1 });

    return agents.find(
      (a) => a.availability.currentChats < a.availability.maxConcurrentChats
    ) || null;
  }

  // Skill-based Strategy
  async skillBasedRoute(ticket) {
    // หา Agent ที่มี Skills ตรงกับ Category
    const requiredSkills = this.getRequiredSkills(ticket.category);

    const agents = await Agent.find({
      status: 'online',
      'availability.isAvailable': true,
      skills: { $in: requiredSkills },
    }).sort({ 'availability.currentChats': 1 });

    if (agents.length > 0) {
      return agents.find(
        (a) => a.availability.currentChats < a.availability.maxConcurrentChats
      ) || null;
    }

    // ถ้าไม่มี Skill-matched agent ใช้ Least Busy
    return this.leastBusyRoute(ticket);
  }

  // Priority-based Strategy
  async priorityBasedRoute(ticket) {
    const priorityQuery = ticket.priority === 'urgent'
      ? { role: { $in: ['senior_agent', 'supervisor'] } }
      : {};

    const agents = await Agent.find({
      status: 'online',
      'availability.isAvailable': true,
      ...priorityQuery,
    }).sort({ 'availability.currentChats': 1 });

    return agents.find(
      (a) => a.availability.currentChats < a.availability.maxConcurrentChats
    ) || null;
  }

  // กำหนด Skills ตาม Category
  getRequiredSkills(category) {
    const skillMap = {
      technical: ['technical_support', 'it_support'],
      billing: ['billing', 'finance'],
      shipping: ['logistics', 'shipping'],
      product: ['product_knowledge'],
      general: ['customer_service'],
    };
    return skillMap[category] || ['customer_service'];
  }

  // Assign Ticket ให้ Agent
  async assignTicketToAgent(ticket, agent) {
    // อัพเดท Ticket
    await Ticket.updateOne(
      { _id: ticket._id },
      {
        $set: {
          assignedTo: agent._id,
          assignedAt: new Date(),
          status: 'open',
        },
        $push: {
          messages: {
            sender: 'system',
            content: `มอบหมายให้ ${agent.displayName}`,
            contentType: 'system_note',
          },
        },
      }
    );

    // อัพเดท Agent
    await Agent.updateOne(
      { _id: agent._id },
      { $inc: { 'availability.currentChats': 1 } }
    );

    logger.info('Ticket assigned', {
      ticketId: ticket.ticketId,
      agentId: agent._id,
      agentName: agent.displayName,
    });

    return true;
  }

  // เพิ่มลง Queue
  async addToQueue(ticket) {
    const queueKey = `queue:${ticket.priority}`;
    const score = this.getPriorityScore(ticket.priority, ticket.createdAt);

    await this.redis.zadd(queueKey, score, ticket._id.toString());
    logger.info('Ticket added to queue', {
      ticketId: ticket.ticketId,
      priority: ticket.priority,
    });
  }

  // คำนวณ Priority Score สำหรับ Queue
  getPriorityScore(priority, createdAt) {
    const priorityMultiplier = {
      urgent: 1000000,
      high: 100000,
      medium: 10000,
      low: 1000,
    };

    const baseScore = priorityMultiplier[priority] || 1000;
    const timeScore = Date.now() - new Date(createdAt).getTime();
    return baseScore - timeScore / 1000; // Higher = สำคัญกว่า
  }

  // ดึง Ticket จาก Queue
  async dequeueTicket() {
    const priorities = ['urgent', 'high', 'medium', 'low'];

    for (const priority of priorities) {
      const queueKey = `queue:${priority}`;
      const ticketIds = await this.redis.zrevrange(queueKey, 0, 0);

      if (ticketIds.length > 0) {
        await this.redis.zrem(queueKey, ticketIds[0]);
        return ticketIds[0];
      }
    }

    return null;
  }

  // Re-route เมื่อ Agent ออกไป
  async rerouteOnAgentOffline(agentId) {
    const openTickets = await Ticket.find({
      assignedTo: agentId,
      status: { $in: ['open', 'pending'] },
    });

    for (const ticket of openTickets) {
      const newAgent = await this.leastBusyRoute(ticket);
      if (newAgent) {
        await this.assignTicketToAgent(ticket, newAgent);
      } else {
        await this.addToQueue(ticket);
        await Ticket.updateOne(
          { _id: ticket._id },
          { $set: { assignedTo: null, status: 'new' } }
        );
      }
    }

    logger.info('Tickets rerouted', {
      agentId,
      count: openTickets.length,
    });
  }

  // สถานะ Queue
  async getQueueStatus() {
    const priorities = ['urgent', 'high', 'medium', 'low'];
    const status = {};

    for (const priority of priorities) {
      status[priority] = await this.redis.zcard(`queue:${priority}`);
    }

    return {
      queues: status,
      total: Object.values(status).reduce((a, b) => a + b, 0),
    };
  }
}

module.exports = new ChatRouter();
```

---

## 5. SLA Tracking Service

```javascript
// src/services/slaTracker.js
const Ticket = require('../models/ticket');
const notificationService = require('./notificationService');
const schedule = require('node-schedule');
const logger = require('../utils/logger');
const dayjs = require('dayjs');

class SLATracker {
  constructor() {
    this.firstResponseMinutes = parseInt(process.env.SLA_FIRST_RESPONSE_MINUTES) || 5;
    this.resolutionHours = parseInt(process.env.SLA_RESOLUTION_HOURS) || 24;
    this.urgentMinutes = parseInt(process.env.SLA_URGENT_MINUTES) || 2;
  }

  // คำนวณ SLA Due dates
  calculateSLADates(ticket) {
    const now = dayjs();

    const firstResponseMinutes = ticket.priority === 'urgent'
      ? this.urgentMinutes
      : this.firstResponseMinutes;

    const resolutionHours = {
      urgent: 2,
      high: 8,
      medium: this.resolutionHours,
      low: 48,
    }[ticket.priority] || this.resolutionHours;

    return {
      firstResponseDue: now.add(firstResponseMinutes, 'minute').toDate(),
      resolutionDue: now.add(resolutionHours, 'hour').toDate(),
    };
  }

  // เริ่ม SLA สำหรับ Ticket ใหม่
  async startSLA(ticket) {
    const slaDates = this.calculateSLADates(ticket);

    await Ticket.updateOne(
      { _id: ticket._id },
      {
        $set: {
          'sla.firstResponseDue': slaDates.firstResponseDue,
          'sla.resolutionDue': slaDates.resolutionDue,
        },
      }
    );

    logger.info('SLA started', {
      ticketId: ticket.ticketId,
      firstResponseDue: slaDates.firstResponseDue,
      resolutionDue: slaDates.resolutionDue,
    });

    return slaDates;
  }

  // บันทึก First Response
  async recordFirstResponse(ticketId) {
    const ticket = await Ticket.findOne({ ticketId });
    if (!ticket || ticket.sla?.firstResponseAt) return;

    const now = new Date();
    const isBreached = now > ticket.sla.firstResponseDue;

    await Ticket.updateOne(
      { ticketId },
      {
        $set: {
          'sla.firstResponseAt': now,
          'sla.firstResponseBreached': isBreached,
        },
      }
    );

    if (isBreached) {
      logger.warn('SLA first response breached', {
        ticketId,
        due: ticket.sla.firstResponseDue,
        respondedAt: now,
        delayMs: now - ticket.sla.firstResponseDue,
      });
    }

    return { isBreached, respondedAt: now };
  }

  // บันทึก Resolution
  async recordResolution(ticketId) {
    const ticket = await Ticket.findOne({ ticketId });
    if (!ticket) return;

    const now = new Date();
    const isBreached = now > ticket.sla.resolutionDue;

    await Ticket.updateOne(
      { ticketId },
      {
        $set: {
          'sla.resolvedAt': now,
          'sla.resolutionBreached': isBreached,
          status: 'resolved',
          closedAt: now,
        },
      }
    );

    if (isBreached) {
      logger.warn('SLA resolution breached', {
        ticketId,
        due: ticket.sla.resolutionDue,
        resolvedAt: now,
      });
    }

    return { isBreached, resolvedAt: now };
  }

  // หยุดนับ SLA (เช่น รอลูกค้า)
  async pauseSLA(ticketId) {
    const ticket = await Ticket.findOne({ ticketId });
    if (!ticket || ticket.sla?.pausedAt) return;

    await Ticket.updateOne(
      { ticketId },
      { $set: { 'sla.pausedAt': new Date() } }
    );
  }

  // เริ่มนับต่อ SLA
  async resumeSLA(ticketId) {
    const ticket = await Ticket.findOne({ ticketId });
    if (!ticket || !ticket.sla?.pausedAt) return;

    const pausedAt = ticket.sla.pausedAt;
    const pausedDuration = Date.now() - pausedAt.getTime();

    // ขยาย Due dates
    const newFirstResponseDue = new Date(
      ticket.sla.firstResponseDue.getTime() + pausedDuration
    );
    const newResolutionDue = new Date(
      ticket.sla.resolutionDue.getTime() + pausedDuration
    );

    await Ticket.updateOne(
      { ticketId },
      {
        $set: {
          'sla.firstResponseDue': newFirstResponseDue,
          'sla.resolutionDue': newResolutionDue,
          'sla.pausedAt': null,
        },
        $inc: {
          'sla.totalPausedTime': pausedDuration,
        },
      }
    );
  }

  // ตรวจสอบ SLA Breaches (เรียกทุก 1 นาที)
  startBreachChecker() {
    schedule.scheduleJob('* * * * *', async () => {
      await this.checkAndAlertBreaches();
    });

    logger.info('SLA breach checker started');
  }

  async checkAndAlertBreaches() {
    const now = new Date();
    const warningThreshold = new Date(now.getTime() + 5 * 60 * 1000); // 5 นาทีล่วงหน้า

    // ค้นหา Tickets ที่ใกล้จะ Breach
    const nearBreachTickets = await Ticket.find({
      status: { $in: ['new', 'open', 'pending'] },
      $or: [
        {
          'sla.firstResponseAt': { $exists: false },
          'sla.firstResponseDue': { $lt: warningThreshold, $gt: now },
        },
        {
          'sla.resolvedAt': { $exists: false },
          'sla.resolutionDue': { $lt: warningThreshold, $gt: now },
        },
      ],
    }).populate('assignedTo');

    for (const ticket of nearBreachTickets) {
      await notificationService.sendSLAWarning(ticket);
    }

    // ค้นหา Tickets ที่ Breach แล้ว
    const breachedTickets = await Ticket.find({
      status: { $in: ['new', 'open', 'pending'] },
      'sla.firstResponseBreached': { $ne: true },
      'sla.firstResponseAt': { $exists: false },
      'sla.firstResponseDue': { $lt: now },
    });

    for (const ticket of breachedTickets) {
      await Ticket.updateOne(
        { _id: ticket._id },
        { $set: { 'sla.firstResponseBreached': true } }
      );
      await notificationService.sendSLABreach(ticket);
    }
  }

  // รายงาน SLA
  async getSLAReport(startDate, endDate) {
    const tickets = await Ticket.find({
      createdAt: { $gte: startDate, $lte: endDate },
      status: { $in: ['resolved', 'closed'] },
    });

    const totalTickets = tickets.length;
    const firstResponseMet = tickets.filter(
      (t) => !t.sla?.firstResponseBreached
    ).length;
    const resolutionMet = tickets.filter(
      (t) => !t.sla?.resolutionBreached
    ).length;

    const avgFirstResponseMs = tickets
      .filter((t) => t.sla?.firstResponseAt)
      .reduce((sum, t) => {
        return sum + (t.sla.firstResponseAt - t.createdAt);
      }, 0) / (totalTickets || 1);

    const avgResolutionMs = tickets
      .filter((t) => t.sla?.resolvedAt)
      .reduce((sum, t) => {
        return sum + (t.sla.resolvedAt - t.createdAt);
      }, 0) / (totalTickets || 1);

    return {
      period: { startDate, endDate },
      totalTickets,
      slaMetrics: {
        firstResponse: {
          met: firstResponseMet,
          breached: totalTickets - firstResponseMet,
          rate: totalTickets ? (firstResponseMet / totalTickets * 100).toFixed(1) : 0,
          avgTimeMinutes: (avgFirstResponseMs / 60000).toFixed(1),
        },
        resolution: {
          met: resolutionMet,
          breached: totalTickets - resolutionMet,
          rate: totalTickets ? (resolutionMet / totalTickets * 100).toFixed(1) : 0,
          avgTimeHours: (avgResolutionMs / 3600000).toFixed(1),
        },
      },
    };
  }
}

module.exports = new SLATracker();
```

---

## 6. Agent Handoff Protocol

```javascript
// src/services/agentHandoff.js
const Ticket = require('../models/ticket');
const Agent = require('../models/agent');
const chatRouter = require('./chatRouter');
const notificationService = require('./notificationService');
const slaTracker = require('./slaTracker');
const logger = require('../utils/logger');

class AgentHandoffService {
  // ส่งต่อจาก Bot ไปยัง Agent
  async handoffToAgent(ticket, reason, metadata = {}) {
    logger.info('Initiating handoff to agent', {
      ticketId: ticket.ticketId,
      reason,
    });

    // เพิ่มระดับความสำคัญถ้าจำเป็น
    const newPriority = this.determinePriority(ticket, reason);

    await Ticket.updateOne(
      { _id: ticket._id },
      {
        $set: {
          status: 'escalated',
          priority: newPriority,
        },
        $push: {
          escalationHistory: {
            from: 'bot',
            to: 'human',
            reason,
            timestamp: new Date(),
          },
          messages: {
            sender: 'system',
            content: `ส่งต่อให้เจ้าหน้าที่: ${reason}`,
            contentType: 'system_note',
          },
        },
      }
    );

    // หา Agent
    const updatedTicket = await Ticket.findById(ticket._id);
    const agent = await chatRouter.assignAgent(updatedTicket);

    if (agent) {
      // แจ้ง Agent
      await notificationService.notifyAgent(agent, updatedTicket, {
        type: 'new_ticket',
        escalationReason: reason,
      });

      return {
        success: true,
        agent,
        message: `กำลังเชื่อมต่อกับ ${agent.displayName} กรุณารอสักครู่...`,
      };
    } else {
      return {
        success: false,
        agent: null,
        message: 'ขณะนี้เจ้าหน้าที่ทุกคนกำลังให้บริการ กรุณารอสักครู่ ท่านอยู่ในคิวที่...',
        queuePosition: await this.getQueuePosition(updatedTicket),
      };
    }
  }

  // เงื่อนไขที่ต้องส่งต่อ Agent
  shouldHandoff(analysisResult, ticket) {
    const reasons = [];

    // Confidence ต่ำ
    if (analysisResult.confidence < 0.5) {
      reasons.push('AI confidence ต่ำ');
    }

    // เจอ Negative Sentiment รุนแรง
    if (analysisResult.sentiment?.score < -0.7) {
      reasons.push('ลูกค้าไม่พอใจอย่างมาก');
    }

    // ขอคุยกับคน
    if (analysisResult.intent === 'request_human_agent') {
      reasons.push('ลูกค้าต้องการคุยกับเจ้าหน้าที่');
    }

    // คำพิเศษที่ต้องส่งต่อ
    const escalationKeywords = [
      'ฟ้อง', 'กฎหมาย', 'ผู้จัดการ', 'หัวหน้า',
      'คืนเงิน', 'เสียหาย', 'ร้องเรียน', 'สื่อ'
    ];
    if (escalationKeywords.some((kw) => analysisResult.input?.original?.includes(kw))) {
      reasons.push('คำสำคัญที่ต้องการ Escalation');
    }

    // Ticket มีการ Fallback บ่อย
    const fallbackCount = ticket.messages?.filter(
      (m) => m.metadata?.isFallback
    ).length;
    if (fallbackCount >= 3) {
      reasons.push('AI ตอบไม่ได้หลายครั้ง');
    }

    return {
      shouldHandoff: reasons.length > 0,
      reasons,
    };
  }

  // กำหนด Priority
  determinePriority(ticket, reason) {
    if (reason.includes('ลูกค้าไม่พอใจ') || reason.includes('ฟ้อง')) {
      return 'urgent';
    }
    if (ticket.priority === 'low' && reason.includes('AI')) {
      return 'medium';
    }
    return ticket.priority;
  }

  // ดู Queue Position
  async getQueuePosition(ticket) {
    const queueStatus = await chatRouter.getQueueStatus();
    const priorities = ['urgent', 'high', 'medium', 'low'];
    let position = 1;

    for (const p of priorities) {
      if (p === ticket.priority) {
        position += queueStatus.queues[p] || 0;
        break;
      }
      if (['urgent', 'high'].includes(p)) {
        position += queueStatus.queues[p] || 0;
      }
    }

    return position;
  }

  // ส่งต่อระหว่าง Agents
  async transferBetweenAgents(ticketId, fromAgentId, toAgentId, reason) {
    const ticket = await Ticket.findOne({ ticketId });
    const toAgent = await Agent.findById(toAgentId);

    if (!ticket || !toAgent) {
      throw new Error('Ticket or Agent not found');
    }

    if (!toAgent.isAvailableForNewChat()) {
      throw new Error('Target agent is not available');
    }

    // ลด current chats ของ Agent เดิม
    await Agent.updateOne(
      { _id: fromAgentId },
      { $inc: { 'availability.currentChats': -1 } }
    );

    // Assign ให้ Agent ใหม่
    await chatRouter.assignTicketToAgent(ticket, toAgent);

    // บันทึกประวัติ
    await Ticket.updateOne(
      { ticketId },
      {
        $push: {
          escalationHistory: {
            from: fromAgentId,
            to: toAgentId,
            reason,
            timestamp: new Date(),
          },
          messages: {
            sender: 'system',
            content: `โอนสายไปให้ ${toAgent.displayName}: ${reason}`,
            contentType: 'system_note',
          },
        },
      }
    );

    logger.info('Ticket transferred', {
      ticketId,
      from: fromAgentId,
      to: toAgentId,
      reason,
    });

    return { success: true, newAgent: toAgent };
  }
}

module.exports = new AgentHandoffService();
```

---

## 7. Customer Satisfaction Tracking

```javascript
// src/services/csatTracker.js
const Ticket = require('../models/ticket');
const Agent = require('../models/agent');
const logger = require('../utils/logger');

class CSATTracker {
  // ส่ง Survey หลังปิด Ticket
  async sendSurvey(ticket, lineClient) {
    const surveyMessage = {
      type: 'template',
      altText: 'ให้คะแนนการบริการ',
      template: {
        type: 'buttons',
        text: `ขอบคุณที่ติดต่อเรา!\n\nคุณพอใจกับการบริการครั้งนี้แค่ไหน?\n(Ticket: ${ticket.ticketId})`,
        actions: [
          {
            type: 'postback',
            label: '⭐⭐⭐⭐⭐ ดีมาก',
            data: `action=csat&ticketId=${ticket.ticketId}&score=5`,
          },
          {
            type: 'postback',
            label: '⭐⭐⭐⭐ ดี',
            data: `action=csat&ticketId=${ticket.ticketId}&score=4`,
          },
          {
            type: 'postback',
            label: '⭐⭐⭐ พอใช้',
            data: `action=csat&ticketId=${ticket.ticketId}&score=3`,
          },
          {
            type: 'postback',
            label: '⭐⭐ ไม่ค่อยดี',
            data: `action=csat&ticketId=${ticket.ticketId}&score=2`,
          },
        ],
      },
    };

    await lineClient.pushMessage({
      to: ticket.lineUserId,
      messages: [surveyMessage],
    });
  }

  // บันทึก CSAT
  async recordScore(ticketId, score, comment = null) {
    const ticket = await Ticket.findOneAndUpdate(
      { ticketId },
      {
        $set: {
          'satisfaction.score': score,
          'satisfaction.comment': comment,
          'satisfaction.ratedAt': new Date(),
        },
      },
      { new: true }
    );

    if (!ticket) {
      throw new Error('Ticket not found');
    }

    // อัพเดทสถิติ Agent
    if (ticket.assignedTo) {
      await this.updateAgentStats(ticket.assignedTo, score);
    }

    logger.info('CSAT recorded', { ticketId, score, comment });

    return ticket;
  }

  // อัพเดทสถิติ Agent
  async updateAgentStats(agentId, score) {
    const agent = await Agent.findById(agentId);
    if (!agent) return;

    const currentAvg = agent.stats.averageSatisfactionScore;
    const totalRated = agent.stats.totalTicketsHandled;

    const newAvg = totalRated > 0
      ? ((currentAvg * totalRated) + score) / (totalRated + 1)
      : score;

    await Agent.updateOne(
      { _id: agentId },
      {
        $set: { 'stats.averageSatisfactionScore': newAvg },
        $inc: { 'stats.totalTicketsHandled': 1 },
      }
    );
  }

  // รายงาน CSAT
  async getCSATReport(startDate, endDate) {
    const tickets = await Ticket.find({
      closedAt: { $gte: startDate, $lte: endDate },
      'satisfaction.score': { $exists: true },
    });

    const totalRated = tickets.length;
    const scores = tickets.map((t) => t.satisfaction.score);
    const avgScore = totalRated
      ? scores.reduce((a, b) => a + b, 0) / totalRated
      : 0;

    const distribution = { 1: 0, 2: 0, 3: 0, 4: 0, 5: 0 };
    scores.forEach((s) => { distribution[s] = (distribution[s] || 0) + 1; });

    const promoters = scores.filter((s) => s >= 4).length;
    const detractors = scores.filter((s) => s <= 2).length;
    const nps = totalRated
      ? ((promoters - detractors) / totalRated) * 100
      : 0;

    return {
      period: { startDate, endDate },
      totalRated,
      averageScore: avgScore.toFixed(2),
      distribution,
      nps: nps.toFixed(1),
      satisfactionRate: totalRated
        ? ((promoters / totalRated) * 100).toFixed(1)
        : 0,
    };
  }
}

module.exports = new CSATTracker();
```

---

## 8. Main Bot Service

```javascript
// src/services/mainBotService.js
const line = require('@line/bot-sdk');
const knowledgeBase = require('./knowledgeBase');
const agentHandoff = require('./agentHandoff');
const slaTracker = require('./slaTracker');
const csatTracker = require('./csatTracker');
const chatRouter = require('./chatRouter');
const Ticket = require('../models/ticket');
const logger = require('../utils/logger');
const { v4: uuidv4 } = require('uuid');

class MainBotService {
  constructor() {
    this.lineClient = new line.messagingApi.MessagingApiClient({
      channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
    });
  }

  // จัดการ Event หลัก
  async handleEvent(event) {
    const userId = event.source.userId;

    switch (event.type) {
      case 'message':
        return this.handleMessage(event, userId);
      case 'follow':
        return this.handleFollow(event, userId);
      case 'postback':
        return this.handlePostback(event, userId);
      default:
        return null;
    }
  }

  // จัดการข้อความ
  async handleMessage(event, userId) {
    const text = event.message.text?.trim();
    if (!text) return;

    // ดึงหรือสร้าง Active Ticket
    const ticket = await this.getOrCreateTicket(userId);

    // ถ้า Ticket มี Agent ส่งต่อให้ Agent
    if (ticket.assignedTo && ['open', 'pending'].includes(ticket.status)) {
      await this.forwardToAgent(ticket, text, userId);
      return;
    }

    // แสดง Loading
    await this.lineClient.showLoadingAnimation({ chatId: userId, loadingSeconds: 10 });

    // บันทึกข้อความลูกค้า
    await Ticket.updateOne(
      { _id: ticket._id },
      {
        $push: {
          messages: {
            sender: 'customer',
            content: text,
          },
        },
      }
    );

    // บันทึก First Response SLA
    if (ticket.sla && !ticket.sla.firstResponseAt) {
      // นี่คือข้อความแรกของลูกค้า
    }

    try {
      // ค้นหาใน Knowledge Base
      const kbResult = await knowledgeBase.generateAnswer(text, userId);

      if (kbResult && kbResult.confidence > 0.7) {
        // ตอบด้วย Knowledge Base
        await this.replyWithKB(event.replyToken, kbResult, ticket);

        // บันทึก Bot Response
        await Ticket.updateOne(
          { _id: ticket._id },
          {
            $push: {
              messages: {
                sender: 'bot',
                content: kbResult.answer,
                metadata: {
                  sources: kbResult.sources,
                  confidence: kbResult.confidence,
                },
              },
            },
          }
        );

        // บันทึก First Response
        await slaTracker.recordFirstResponse(ticket.ticketId);
      } else {
        // Knowledge Base ตอบไม่ได้ - Escalate
        const handoffResult = await agentHandoff.handoffToAgent(
          ticket,
          'AI ไม่สามารถตอบคำถามได้',
          { query: text }
        );

        const message = handoffResult.success
          ? handoffResult.message
          : `${handoffResult.message} (คิวที่ ${handoffResult.queuePosition})`;

        await this.lineClient.replyMessage({
          replyToken: event.replyToken,
          messages: [{ type: 'text', text: message }],
        });
      }
    } catch (error) {
      logger.error('Message handling error', { error: error.message, userId });
      await this.lineClient.replyMessage({
        replyToken: event.replyToken,
        messages: [{ type: 'text', text: 'ขออภัยครับ เกิดข้อผิดพลาด กำลังติดต่อเจ้าหน้าที่' }],
      });

      // Emergency Escalation
      await agentHandoff.handoffToAgent(ticket, 'System Error');
    }
  }

  // ตอบด้วย Knowledge Base
  async replyWithKB(replyToken, kbResult, ticket) {
    const messages = [{ type: 'text', text: kbResult.answer }];

    // เพิ่ม Quick Replies
    const lastMsg = messages[messages.length - 1];
    lastMsg.quickReply = {
      items: [
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '👍 มีประโยชน์',
            data: `action=helpful&ticketId=${ticket.ticketId}&helpful=true`,
          },
        },
        {
          type: 'action',
          action: {
            type: 'postback',
            label: '👎 ไม่มีประโยชน์',
            data: `action=helpful&ticketId=${ticket.ticketId}&helpful=false`,
          },
        },
        {
          type: 'action',
          action: {
            type: 'message',
            label: '👤 คุยกับเจ้าหน้าที่',
            text: 'ต้องการคุยกับเจ้าหน้าที่',
          },
        },
      ],
    };

    await this.lineClient.replyMessage({ replyToken, messages });
  }

  // ส่งต่อข้อความไปยัง Agent
  async forwardToAgent(ticket, text, userId) {
    // ส่ง Real-time ไปยัง Agent Dashboard
    const io = require('../socket').getIO();
    io.to(`agent:${ticket.assignedTo}`).emit('customer_message', {
      ticketId: ticket.ticketId,
      userId,
      text,
      timestamp: new Date(),
    });
  }

  // จัดการ Postback
  async handlePostback(event, userId) {
    const params = new URLSearchParams(event.postback.data);
    const action = params.get('action');
    const ticketId = params.get('ticketId');

    switch (action) {
      case 'csat': {
        const score = parseInt(params.get('score'));
        await csatTracker.recordScore(ticketId, score);

        let message = '';
        if (score >= 4) {
          message = '🙏 ขอบคุณสำหรับคะแนนที่ดีครับ! ยินดีให้บริการเสมอ';
        } else if (score === 3) {
          message = '🙏 ขอบคุณครับ เราจะปรับปรุงบริการให้ดียิ่งขึ้น';
        } else {
          message = '🙏 ขอโทษสำหรับประสบการณ์ที่ไม่ดีครับ เราจะนำ feedback ไปปรับปรุง';
        }

        await this.lineClient.replyMessage({
          replyToken: event.replyToken,
          messages: [{ type: 'text', text: message }],
        });
        break;
      }

      case 'helpful': {
        const isHelpful = params.get('helpful') === 'true';
        await KnowledgeBase.updateOne(
          { ticketId },
          isHelpful
            ? { $inc: { helpfulVotes: 1 } }
            : { $inc: { unhelpfulVotes: 1 } }
        );

        if (!isHelpful) {
          // ถ้าไม่มีประโยชน์ ส่งต่อ Agent
          const ticket = await Ticket.findOne({ ticketId });
          if (ticket) {
            const result = await agentHandoff.handoffToAgent(
              ticket, 'คำตอบ AI ไม่ตรงกับที่ต้องการ'
            );
            await this.lineClient.replyMessage({
              replyToken: event.replyToken,
              messages: [{ type: 'text', text: result.message }],
            });
          }
        } else {
          await this.lineClient.replyMessage({
            replyToken: event.replyToken,
            messages: [{ type: 'text', text: '😊 ดีใจที่ช่วยได้ครับ มีอะไรให้ช่วยอีกไหม?' }],
          });
        }
        break;
      }

      default:
        logger.debug('Unhandled postback action', { action });
    }
  }

  // Follow Event
  async handleFollow(event, userId) {
    const profile = await this.lineClient.getProfile(userId);

    const welcomeMessage = {
      type: 'flex',
      altText: 'ยินดีต้อนรับสู่ Customer Service',
      contents: {
        type: 'bubble',
        hero: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'text',
            text: '🎉 ยินดีต้อนรับ!',
            size: '2xl',
            weight: 'bold',
            align: 'center',
            color: '#ffffff',
          }],
          backgroundColor: '#4A90D9',
          paddingAll: '20px',
        },
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: `สวัสดี ${profile.displayName}!`,
              weight: 'bold',
              size: 'lg',
            },
            {
              type: 'text',
              text: 'เราพร้อมช่วยเหลือคุณตลอด 24/7',
              color: '#888',
              margin: 'sm',
            },
            {
              type: 'separator',
              margin: 'lg',
            },
            {
              type: 'text',
              text: 'สิ่งที่เราช่วยได้:',
              weight: 'bold',
              margin: 'lg',
            },
            {
              type: 'text',
              text: '• ตอบคำถามทั่วไป\n• ติดตามสถานะออเดอร์\n• แก้ปัญหาทางเทคนิค\n• ข้อมูลสินค้าและบริการ',
              wrap: true,
              margin: 'sm',
            },
          ],
        },
        footer: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'button',
            action: {
              type: 'message',
              label: 'เริ่มต้นใช้งาน',
              text: 'ต้องการความช่วยเหลือ',
            },
            style: 'primary',
            color: '#4A90D9',
          }],
        },
      },
    };

    await this.lineClient.replyMessage({
      replyToken: event.replyToken,
      messages: [welcomeMessage],
    });
  }

  // สร้างหรือดึง Active Ticket
  async getOrCreateTicket(userId) {
    // ดึง Ticket ที่ Active อยู่
    const activeTicket = await Ticket.findOne({
      lineUserId: userId,
      status: { $in: ['new', 'open', 'pending', 'escalated', 'waiting_customer'] },
    });

    if (activeTicket) return activeTicket;

    // สร้าง Ticket ใหม่
    const ticketId = `TKT-${Date.now()}`;
    const ticket = new Ticket({
      ticketId,
      lineUserId: userId,
      status: 'new',
      priority: 'medium',
      category: 'general',
    });

    await ticket.save();

    // เริ่ม SLA
    await slaTracker.startSLA(ticket);

    logger.info('New ticket created', { ticketId, userId });
    return ticket;
  }
}

module.exports = new MainBotService();
```

---

## 9. Performance Metrics Dashboard

```javascript
// src/services/metricsService.js
const Ticket = require('../models/ticket');
const Agent = require('../models/agent');
const dayjs = require('dayjs');

class MetricsService {
  // Dashboard ภาพรวม
  async getDashboardMetrics(period = 'today') {
    const { startDate, endDate } = this.getPeriodDates(period);

    const [ticketMetrics, agentMetrics, channelMetrics] = await Promise.all([
      this.getTicketMetrics(startDate, endDate),
      this.getAgentMetrics(startDate, endDate),
      this.getChannelMetrics(startDate, endDate),
    ]);

    return {
      period,
      generatedAt: new Date(),
      tickets: ticketMetrics,
      agents: agentMetrics,
      channels: channelMetrics,
    };
  }

  // Ticket Metrics
  async getTicketMetrics(startDate, endDate) {
    const tickets = await Ticket.find({
      createdAt: { $gte: startDate, $lte: endDate },
    });

    const byStatus = this.groupBy(tickets, 'status');
    const byPriority = this.groupBy(tickets, 'priority');
    const byCategory = this.groupBy(tickets, 'category');

    const resolved = tickets.filter((t) => t.status === 'resolved');
    const withSatisfaction = tickets.filter((t) => t.satisfaction?.score);
    const avgSatisfaction = withSatisfaction.length
      ? withSatisfaction.reduce((sum, t) => sum + t.satisfaction.score, 0) / withSatisfaction.length
      : 0;

    const fcr = tickets.length
      ? tickets.filter((t) => t.firstContactResolution).length / tickets.length
      : 0;

    return {
      total: tickets.length,
      byStatus,
      byPriority,
      byCategory,
      resolved: resolved.length,
      resolutionRate: tickets.length
        ? ((resolved.length / tickets.length) * 100).toFixed(1)
        : 0,
      avgSatisfaction: avgSatisfaction.toFixed(2),
      firstContactResolutionRate: (fcr * 100).toFixed(1),
      averageMessages: tickets.length
        ? (tickets.reduce((sum, t) => sum + (t.messages?.length || 0), 0) / tickets.length).toFixed(1)
        : 0,
    };
  }

  // Agent Metrics
  async getAgentMetrics(startDate, endDate) {
    const agents = await Agent.find();
    const agentStats = [];

    for (const agent of agents) {
      const handledTickets = await Ticket.countDocuments({
        assignedTo: agent._id,
        createdAt: { $gte: startDate, $lte: endDate },
      });

      const resolvedTickets = await Ticket.countDocuments({
        assignedTo: agent._id,
        status: { $in: ['resolved', 'closed'] },
        closedAt: { $gte: startDate, $lte: endDate },
      });

      agentStats.push({
        agentId: agent._id,
        name: agent.displayName,
        status: agent.status,
        currentChats: agent.availability.currentChats,
        handled: handledTickets,
        resolved: resolvedTickets,
        avgSatisfaction: agent.stats.averageSatisfactionScore?.toFixed(2) || '-',
        avgResponseTime: agent.stats.averageResponseTimeMs
          ? `${(agent.stats.averageResponseTimeMs / 1000).toFixed(0)}s`
          : '-',
      });
    }

    return {
      total: agents.length,
      online: agents.filter((a) => a.status === 'online').length,
      agentStats: agentStats.sort((a, b) => b.handled - a.handled),
    };
  }

  // Channel Metrics
  async getChannelMetrics(startDate, endDate) {
    const botHandled = await Ticket.countDocuments({
      createdAt: { $gte: startDate, $lte: endDate },
      createdBy: 'bot',
      assignedTo: { $exists: false },
    });

    const humanHandled = await Ticket.countDocuments({
      createdAt: { $gte: startDate, $lte: endDate },
      assignedTo: { $exists: true },
    });

    const total = botHandled + humanHandled;

    return {
      bot: {
        count: botHandled,
        percentage: total ? ((botHandled / total) * 100).toFixed(1) : 0,
      },
      human: {
        count: humanHandled,
        percentage: total ? ((humanHandled / total) * 100).toFixed(1) : 0,
      },
      total,
    };
  }

  // Helper: Group By Field
  groupBy(items, field) {
    const groups = {};
    for (const item of items) {
      const key = item[field] || 'unknown';
      groups[key] = (groups[key] || 0) + 1;
    }
    return groups;
  }

  // Helper: Period Dates
  getPeriodDates(period) {
    const now = dayjs();
    switch (period) {
      case 'today':
        return { startDate: now.startOf('day').toDate(), endDate: now.endOf('day').toDate() };
      case 'week':
        return { startDate: now.startOf('week').toDate(), endDate: now.endOf('week').toDate() };
      case 'month':
        return { startDate: now.startOf('month').toDate(), endDate: now.endOf('month').toDate() };
      default:
        return { startDate: now.startOf('day').toDate(), endDate: now.endOf('day').toDate() };
    }
  }
}

module.exports = new MetricsService();
```

---

## 10. Main Application

```javascript
// src/index.js
const express = require('express');
const line = require('@line/bot-sdk');
const mongoose = require('mongoose');
const http = require('http');
const socketIO = require('socket.io');
const helmet = require('helmet');
const cors = require('cors');
const morgan = require('morgan');

const mainBotService = require('./services/mainBotService');
const slaTracker = require('./services/slaTracker');
const knowledgeBase = require('./services/knowledgeBase');
const metricsService = require('./services/metricsService');
const agentRouter = require('./routes/agent');
const adminRouter = require('./routes/admin');
const logger = require('./utils/logger');

const app = express();
const server = http.createServer(app);
const io = socketIO(server, { cors: { origin: '*' } });

// Store IO instance
require('./socket').setIO(io);

// Middleware
app.use(helmet());
app.use(cors());
app.use(morgan('combined', { stream: { write: (m) => logger.info(m.trim()) } }));

// LINE Webhook
app.post('/webhook', line.middleware({
  channelSecret: process.env.LINE_CHANNEL_SECRET,
}), async (req, res) => {
  try {
    await Promise.all(req.body.events.map((e) => mainBotService.handleEvent(e)));
    res.sendStatus(200);
  } catch (error) {
    logger.error('Webhook error', { error: error.message });
    res.sendStatus(500);
  }
});

// JSON Parsing (ต้องอยู่หลัง LINE webhook)
app.use(express.json());

// API Routes
app.use('/api/agent', agentRouter);
app.use('/api/admin', adminRouter);

// Metrics endpoint
app.get('/api/metrics', async (req, res) => {
  const metrics = await metricsService.getDashboardMetrics(req.query.period);
  res.json(metrics);
});

// Health Check
app.get('/health', (req, res) => {
  res.json({ status: 'healthy', timestamp: new Date().toISOString() });
});

// Socket.IO สำหรับ Agent Dashboard
io.on('connection', (socket) => {
  logger.info('Agent connected', { socketId: socket.id });

  socket.on('agent:join', (agentId) => {
    socket.join(`agent:${agentId}`);
    logger.info('Agent joined room', { agentId });
  });

  socket.on('agent:message', async (data) => {
    const { ticketId, message } = data;
    // ส่งข้อความจาก Agent ไปยังลูกค้า
    await mainBotService.lineClient.pushMessage({
      to: data.lineUserId,
      messages: [{ type: 'text', text: message }],
    });
  });

  socket.on('disconnect', () => {
    logger.info('Agent disconnected', { socketId: socket.id });
  });
});

// Start
async function start() {
  try {
    await mongoose.connect(process.env.MONGODB_URI);
    logger.info('Connected to MongoDB');

    await knowledgeBase.initialize();
    logger.info('Knowledge base initialized');

    slaTracker.startBreachChecker();
    logger.info('SLA breach checker started');

    const port = process.env.PORT || 3000;
    server.listen(port, () => {
      logger.info(`Server started on port ${port}`);
    });
  } catch (error) {
    logger.error('Startup error', { error: error.message });
    process.exit(1);
  }
}

process.on('SIGTERM', async () => {
  await mongoose.disconnect();
  process.exit(0);
});

start();

module.exports = { app, io };
```

---

## 11. Notification Service

```javascript
// src/services/notificationService.js
const nodemailer = require('nodemailer');
const logger = require('../utils/logger');

class NotificationService {
  constructor() {
    this.transporter = nodemailer.createTransport({
      host: process.env.SMTP_HOST,
      port: parseInt(process.env.SMTP_PORT),
      secure: false,
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASS,
      },
    });
  }

  async sendSLAWarning(ticket) {
    const agent = ticket.assignedTo;
    if (!agent?.email) return;

    await this.sendEmail({
      to: agent.email,
      subject: `⚠️ SLA Warning: Ticket ${ticket.ticketId}`,
      text: `SLA กำลังจะถึงกำหนด!\n\nTicket: ${ticket.ticketId}\nลูกค้า: ${ticket.customerName || ticket.lineUserId}\nCategory: ${ticket.category}\nPriority: ${ticket.priority}`,
    });
  }

  async sendSLABreach(ticket) {
    await this.sendEmail({
      to: process.env.NOTIFICATION_EMAIL,
      subject: `🚨 SLA Breach: Ticket ${ticket.ticketId}`,
      text: `SLA ถูก Breach!\n\nTicket: ${ticket.ticketId}\nCategory: ${ticket.category}\nPriority: ${ticket.priority}\nCreated: ${ticket.createdAt}`,
    });
  }

  async notifyAgent(agent, ticket, metadata = {}) {
    if (!agent.settings?.notifications?.newTicket) return;
    if (!agent.email) return;

    await this.sendEmail({
      to: agent.email,
      subject: `📩 Ticket ใหม่: ${ticket.ticketId}`,
      text: `มี Ticket ใหม่สำหรับคุณ\n\nTicket: ${ticket.ticketId}\nCategory: ${ticket.category}\nPriority: ${ticket.priority}${metadata.escalationReason ? `\nเหตุผล: ${metadata.escalationReason}` : ''}`,
    });
  }

  async sendEmail({ to, subject, text, html }) {
    try {
      await this.transporter.sendMail({
        from: `"CS Bot" <${process.env.SMTP_USER}>`,
        to,
        subject,
        text,
        html,
      });
      logger.info('Email sent', { to, subject });
    } catch (error) {
      logger.error('Email send error', { error: error.message, to });
    }
  }
}

module.exports = new NotificationService();
```

---

## 12. Sequence Diagram: Complete Flow

```
ลูกค้า       LINE Bot     KB Service    AI Engine   Agent System   Human Agent
   │              │              │             │            │              │
   │─ข้อความ────►│              │             │            │              │
   │              │─ค้นหา─────►│             │            │              │
   │              │◄─ผลลัพธ────│             │            │              │
   │              │              │             │            │              │
   │         [confidence >= 0.7?]              │            │              │
   │              │─ใช่─────────────────────►│            │              │
   │              │              │    สร้างคำตอบ│            │              │
   │              │◄─────────────────────────│            │              │
   │◄─คำตอบ──────│              │             │            │              │
   │              │              │             │            │              │
   │         [confidence < 0.7]               │            │              │
   │              │──────Escalate────────────────────────►│              │
   │              │              │             │    หา Agent│              │
   │              │              │             │◄───────────│              │
   │◄─"กำลังโอน"─│              │             │            │──แจ้ง Agent►│
   │              │              │             │            │              │
   │              │              │             │            │◄─ Accept────│
   │─ข้อความ────►│              │             │            │              │
   │              │──Forward────────────────────────────────────────────►│
   │◄─ตอบ─────────────────────────────────────────────────────────────── │
   │              │              │             │            │              │
   │         [Close Ticket]      │             │            │              │
   │              │──Send Survey►│             │            │              │
   │◄─ Survey────│              │             │            │              │
   │─คะแนน──────►│              │             │            │              │
   │              │──บันทึก CSAT─────────────────────────►│              │
```

---

## สรุปบทที่ 65

ในบทนี้เราได้สร้างระบบ Enterprise Customer Service Bot ที่สมบูรณ์:

1. **Architecture** - ออกแบบ Scalable Architecture รองรับ Production
2. **Knowledge Base** - Elasticsearch + OpenAI Embedding สำหรับ Semantic Search
3. **Ticket System** - MongoDB ติดตาม Tickets พร้อม Status และ Priority
4. **Chat Routing** - Round Robin, Least Busy, Skill-based, Priority-based
5. **SLA Tracking** - ติดตาม SLA Breach พร้อม Alert อัตโนมัติ
6. **Agent Handoff** - Protocol ส่งต่อจาก Bot ไปยัง Human Agent
7. **CSAT Tracking** - วัด Customer Satisfaction หลังปิด Ticket
8. **Real-time** - Socket.IO สำหรับ Agent Dashboard
9. **Notifications** - Email Alerts สำหรับ SLA และ Escalations
10. **Performance Metrics** - Dashboard ครบถ้วน

### Best Practices

- ใช้ **Hybrid Search** (Text + Semantic) สำหรับ Knowledge Base
- กำหนด **SLA ตาม Priority** ให้ชัดเจน
- มี **Fallback หลายชั้น** (Bot → KB → AI → Human)
- เก็บ **Conversation History** ครบถ้วนสำหรับ Agent
- ติดตาม **CSAT ทุก Ticket** เพื่อปรับปรุงบริการ
- Monitor **SLA อย่างต่อเนื่อง** ด้วย Real-time Alerts

---

*จบ Part 65 - Enterprise Customer Service Bot*

*ซีรีส์ LINE OA Course Parts 61-65 ครอบคลุม AI/ML Integration ทั้งหมดแล้ว*
