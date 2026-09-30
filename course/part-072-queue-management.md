# Part 72: Queue Management สำหรับ LINE Bot

## บทนำ

Queue Management คือหัวใจสำคัญของระบบ LINE Bot ที่มี traffic สูง การใช้ Queue ช่วยให้ระบบรองรับ load spike ได้ โดยไม่ทำให้ webhook response ช้า และสามารถ process messages แบบ async ได้อย่างมีประสิทธิภาพ

---

## 1. ทำไมต้องใช้ Queue

### 1.1 ปัญหาที่แก้ได้ด้วย Queue

```
Without Queue:
─────────────
LINE → Webhook → Process → Reply
                    │
                ใช้เวลา 2-10 วิ
                → Timeout!

With Queue:
──────────
LINE → Webhook → [Queue] → Worker → Process → Reply
         │
      ตอบ 200 OK
      ทันที (< 100ms)
```

### 1.2 Use Cases สำหรับ LINE Bot

| Job Type | Priority | Processing Time | Queue |
|----------|----------|-----------------|-------|
| Webhook events | High | < 500ms | webhook-events |
| Reply messages | High | < 1s | message-sending |
| Broadcast | Normal | 1-5 min | broadcast |
| Report generation | Low | 1-30 min | reports |
| Analytics processing | Low | 5-60 min | analytics |
| Scheduled messages | Scheduled | Configurable | scheduler |

---

## 2. BullMQ - Queue สำหรับ Node.js

### 2.1 การติดตั้ง BullMQ

```bash
npm install bullmq ioredis
npm install bull-board  # สำหรับ UI monitoring
```

### 2.2 Queue Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    BullMQ Architecture                   │
│                                                         │
│  Producers                    Workers                   │
│  ┌──────────┐                ┌──────────────────────┐  │
│  │ Webhook  │──→ [Queue 1] ──│ Message Handler x3   │  │
│  │ Receiver │                └──────────────────────┘  │
│  └──────────┘                                          │
│                               ┌──────────────────────┐  │
│  ┌──────────┐                │ Notification         │  │
│  │  Admin   │──→ [Queue 2] ──│ Worker x2            │  │
│  │  API     │                └──────────────────────┘  │
│  └──────────┘                                          │
│                               ┌──────────────────────┐  │
│  ┌──────────┐                │ Report Generator     │  │
│  │Scheduler │──→ [Queue 3] ──│ Worker x1            │  │
│  └──────────┘                └──────────────────────┘  │
│                                                         │
│              Redis (Storage)                            │
│         ┌───────────────────────────┐                   │
│         │  Waiting | Active | Done  │                   │
│         │  Failed  | Delayed        │                   │
│         └───────────────────────────┘                   │
└─────────────────────────────────────────────────────────┘
```

### 2.3 Queue Setup

```javascript
// queues/setup.js
const { Queue, Worker, QueueEvents, FlowProducer } = require('bullmq');
const IORedis = require('ioredis');

// Redis connection
const redisConnection = new IORedis({
  host: process.env.REDIS_HOST || 'localhost',
  port: process.env.REDIS_PORT || 6379,
  password: process.env.REDIS_PASSWORD,
  maxRetriesPerRequest: null, // สำคัญ! BullMQ ต้องการค่านี้
  enableReadyCheck: false,
  lazyConnect: true,
});

// Queue definitions
const QUEUE_NAMES = {
  WEBHOOK_EVENTS: 'webhook:events',
  MESSAGE_SENDING: 'message:sending',
  BROADCAST: 'broadcast:messages',
  NOTIFICATIONS: 'notifications:push',
  REPORTS: 'reports:generation',
  ANALYTICS: 'analytics:processing',
  SCHEDULED: 'scheduled:messages',
};

// Default queue options
const DEFAULT_JOB_OPTIONS = {
  attempts: 3,
  backoff: {
    type: 'exponential',
    delay: 1000,
  },
  removeOnComplete: {
    count: 1000,
    age: 24 * 3600, // 24 hours
  },
  removeOnFail: {
    count: 5000,
    age: 7 * 24 * 3600, // 7 days
  },
};

// สร้าง queues
const queues = {};

Object.entries(QUEUE_NAMES).forEach(([key, name]) => {
  queues[key] = new Queue(name, {
    connection: redisConnection,
    defaultJobOptions: DEFAULT_JOB_OPTIONS,
  });
});

// Queue Events สำหรับ monitoring
const queueEvents = {};
Object.entries(QUEUE_NAMES).forEach(([key, name]) => {
  queueEvents[key] = new QueueEvents(name, {
    connection: redisConnection,
  });
});

module.exports = {
  queues,
  queueEvents,
  QUEUE_NAMES,
  redisConnection,
};
```

### 2.4 Webhook Event Queue

```javascript
// queues/webhookQueue.js
const { Queue, Worker, QueueScheduler } = require('bullmq');
const { redisConnection, QUEUE_NAMES } = require('./setup');
const logger = require('../utils/logger');

// Webhook Event Queue Producer
class WebhookQueueProducer {
  constructor() {
    this.queue = new Queue(QUEUE_NAMES.WEBHOOK_EVENTS, {
      connection: redisConnection,
      defaultJobOptions: {
        attempts: 5,
        backoff: {
          type: 'exponential',
          delay: 500,
        },
        priority: 10, // สูง = สำคัญกว่า
      }
    });
  }

  async addEvent(event, destination) {
    const jobData = {
      event,
      destination,
      receivedAt: Date.now(),
    };

    // กำหนด priority ตาม event type
    const priority = this.getEventPriority(event.type);

    const job = await this.queue.add(
      `webhook:${event.type}`,
      jobData,
      {
        priority,
        jobId: `${event.source?.userId}-${event.timestamp}`, // Deduplication
      }
    );

    logger.debug(`Added webhook event to queue`, {
      jobId: job.id,
      eventType: event.type,
      priority
    });

    return job.id;
  }

  getEventPriority(eventType) {
    const priorities = {
      'message': 1,      // สูงสุด
      'postback': 2,
      'follow': 3,
      'unfollow': 5,
      'join': 5,
      'leave': 5,
      'memberJoined': 7,
      'memberLeft': 7,
    };
    return priorities[eventType] || 10;
  }

  async getQueueStats() {
    const [waiting, active, failed, delayed, completed] = await Promise.all([
      this.queue.getWaitingCount(),
      this.queue.getActiveCount(),
      this.queue.getFailedCount(),
      this.queue.getDelayedCount(),
      this.queue.getCompletedCount(),
    ]);

    return { waiting, active, failed, delayed, completed };
  }
}

// Webhook Event Worker
class WebhookWorker {
  constructor(concurrency = 10) {
    this.worker = new Worker(
      QUEUE_NAMES.WEBHOOK_EVENTS,
      this.processJob.bind(this),
      {
        connection: redisConnection,
        concurrency,
        limiter: {
          max: 100,         // สูงสุด 100 jobs
          duration: 1000,   // ต่อ 1 วินาที
        }
      }
    );

    this.setupListeners();
  }

  async processJob(job) {
    const { event, destination, receivedAt } = job.data;
    const processingDelay = Date.now() - receivedAt;

    logger.info(`Processing webhook event`, {
      jobId: job.id,
      eventType: event.type,
      userId: event.source?.userId,
      delay: `${processingDelay}ms`
    });

    // Update progress
    await job.updateProgress(10);

    try {
      await this.routeEvent(event, destination);
      await job.updateProgress(100);

      return {
        processed: true,
        processingTime: Date.now() - receivedAt,
        eventType: event.type
      };

    } catch (error) {
      logger.error(`Failed to process event ${event.type}:`, error);
      throw error;
    }
  }

  async routeEvent(event, destination) {
    const { handleMessage } = require('../handlers/messageHandler');
    const { handlePostback } = require('../handlers/postbackHandler');
    const { handleFollow } = require('../handlers/followHandler');

    const handlers = {
      message: handleMessage,
      postback: handlePostback,
      follow: handleFollow,
    };

    const handler = handlers[event.type];
    if (handler) {
      await handler(event, destination);
    }
  }

  setupListeners() {
    this.worker.on('completed', (job, result) => {
      logger.debug(`Job ${job.id} completed`, result);
    });

    this.worker.on('failed', (job, err) => {
      logger.error(`Job ${job?.id} failed:`, {
        error: err.message,
        attempts: job?.attemptsMade,
        data: job?.data
      });
    });

    this.worker.on('stalled', (jobId) => {
      logger.warn(`Job ${jobId} stalled - worker crashed?`);
    });

    this.worker.on('error', (err) => {
      logger.error('Worker error:', err);
    });
  }

  async close() {
    await this.worker.close();
  }
}

module.exports = { WebhookQueueProducer, WebhookWorker };
```

### 2.5 Message Sending Queue

```javascript
// queues/messageSendingQueue.js
const { Queue, Worker } = require('bullmq');
const { redisConnection, QUEUE_NAMES } = require('./setup');
const { LineMessagingClient } = require('../services/lineClient');
const logger = require('../utils/logger');

const lineClient = new LineMessagingClient();

class MessageSendingQueue {
  constructor() {
    this.queue = new Queue(QUEUE_NAMES.MESSAGE_SENDING, {
      connection: redisConnection,
      defaultJobOptions: {
        attempts: 3,
        backoff: {
          type: 'exponential',
          delay: 1000,
        },
      }
    });
  }

  // Reply message (ใช้ reply token)
  async addReply(replyToken, messages, userId) {
    return this.queue.add('reply', {
      type: 'reply',
      replyToken,
      messages: Array.isArray(messages) ? messages : [messages],
      userId,
      addedAt: Date.now()
    }, {
      priority: 1,          // สูงสุด
      delay: 0,             // ส่งทันที
      attempts: 1,          // reply token ใช้ได้ครั้งเดียว
    });
  }

  // Push message (ส่งหา user โดยตรง)
  async addPush(userId, messages, options = {}) {
    return this.queue.add('push', {
      type: 'push',
      userId,
      messages: Array.isArray(messages) ? messages : [messages],
      addedAt: Date.now()
    }, {
      priority: options.priority || 5,
      delay: options.delay || 0,
      attempts: 3,
    });
  }

  // Multicast (ส่งหลาย users)
  async addMulticast(userIds, messages, options = {}) {
    // แบ่ง userIds เป็น chunks ละ 500
    const chunks = [];
    for (let i = 0; i < userIds.length; i += 500) {
      chunks.push(userIds.slice(i, i + 500));
    }

    const jobs = await Promise.all(
      chunks.map((chunk, index) =>
        this.queue.add('multicast', {
          type: 'multicast',
          userIds: chunk,
          messages: Array.isArray(messages) ? messages : [messages],
          chunkIndex: index,
          totalChunks: chunks.length,
          addedAt: Date.now()
        }, {
          priority: options.priority || 8,
          delay: index * 100, // Stagger แต่ละ chunk
        })
      )
    );

    return jobs;
  }
}

// Message Sending Worker
class MessageSendingWorker {
  constructor() {
    this.worker = new Worker(
      QUEUE_NAMES.MESSAGE_SENDING,
      this.processJob.bind(this),
      {
        connection: redisConnection,
        concurrency: 20,  // ส่งได้พร้อมกัน 20 messages
        limiter: {
          max: 1000,         // LINE API limit: 1000 req/s
          duration: 1000,
        }
      }
    );

    this.setupListeners();
  }

  async processJob(job) {
    const { type, ...data } = job.data;

    switch (type) {
      case 'reply':
        return this.processReply(job, data);
      case 'push':
        return this.processPush(job, data);
      case 'multicast':
        return this.processMulticast(job, data);
      default:
        throw new Error(`Unknown message type: ${type}`);
    }
  }

  async processReply(job, { replyToken, messages, userId }) {
    const delay = Date.now() - job.data.addedAt;
    
    if (delay > 55000) { // reply token หมดอายุหลัง 60 วิ
      logger.warn(`Reply token may be expired for user ${userId} (delay: ${delay}ms)`);
    }

    await lineClient.replyMessage(replyToken, messages);
    
    return { sent: messages.length, userId, delay };
  }

  async processPush(job, { userId, messages }) {
    await lineClient.pushMessage(userId, messages);
    return { sent: messages.length, userId };
  }

  async processMulticast(job, { userIds, messages, chunkIndex, totalChunks }) {
    await lineClient.multicast(userIds, messages);
    
    logger.info(`Multicast chunk ${chunkIndex + 1}/${totalChunks} sent`, {
      recipientCount: userIds.length
    });

    return { sent: messages.length, recipientCount: userIds.length };
  }

  setupListeners() {
    this.worker.on('failed', async (job, err) => {
      if (job?.data.type === 'reply') {
        // Log แต่ไม่ retry reply (token หมดอายุ)
        logger.warn(`Reply failed (token may be expired):`, {
          userId: job.data.userId,
          error: err.message
        });
      }
    });
  }
}

module.exports = { MessageSendingQueue, MessageSendingWorker };
```

---

## 3. Broadcast Queue

### 3.1 Large-Scale Broadcast

```javascript
// queues/broadcastQueue.js
const { Queue, Worker, FlowProducer } = require('bullmq');
const { redisConnection, QUEUE_NAMES } = require('./setup');
const { MessageSendingQueue } = require('./messageSendingQueue');
const logger = require('../utils/logger');

const messageSendingQueue = new MessageSendingQueue();

class BroadcastQueue {
  constructor() {
    this.queue = new Queue(QUEUE_NAMES.BROADCAST, {
      connection: redisConnection,
      defaultJobOptions: {
        attempts: 2,
        backoff: { type: 'fixed', delay: 60000 },
        removeOnComplete: { age: 7 * 24 * 3600 },
      }
    });
  }

  // เพิ่ม broadcast job
  async addBroadcast(params) {
    const {
      messages,
      userIds,          // ถ้าไม่กำหนด = ส่งทุกคน
      segments,         // กำหนด segment tags
      scheduledAt,      // ถ้าไม่กำหนด = ส่งทันที
      batchSize = 500,
      name = 'Broadcast',
    } = params;

    const delay = scheduledAt 
      ? Math.max(0, new Date(scheduledAt) - Date.now())
      : 0;

    const job = await this.queue.add('broadcast', {
      messages,
      userIds,
      segments,
      batchSize,
      name,
      scheduledAt,
      totalSent: 0,
      status: 'pending',
      createdAt: Date.now()
    }, {
      delay,
      priority: scheduledAt ? 5 : 10,
    });

    logger.info(`Broadcast queued: ${job.id}`, {
      name,
      recipientCount: userIds?.length || 'all',
      scheduledAt: scheduledAt || 'immediate'
    });

    return { jobId: job.id };
  }
}

// Broadcast Worker
class BroadcastWorker {
  constructor() {
    this.worker = new Worker(
      QUEUE_NAMES.BROADCAST,
      this.processJob.bind(this),
      {
        connection: redisConnection,
        concurrency: 1, // Broadcast ทีละ 1 (ไม่ให้ overlap)
      }
    );
  }

  async processJob(job) {
    const { messages, userIds, segments, batchSize, name } = job.data;

    logger.info(`Starting broadcast: ${name}`, { jobId: job.id });

    // ดึง userIds ถ้าไม่ได้กำหนด
    let targetUserIds = userIds;
    
    if (!targetUserIds) {
      targetUserIds = await this.getUserIds(segments);
    }

    const total = targetUserIds.length;
    let sent = 0;
    
    await job.updateProgress({ 
      status: 'sending', 
      total,
      sent: 0 
    });

    // แบ่งส่งเป็น batches
    for (let i = 0; i < targetUserIds.length; i += batchSize) {
      const batch = targetUserIds.slice(i, i + batchSize);
      
      try {
        await messageSendingQueue.addMulticast(batch, messages, {
          priority: 5
        });
        
        sent += batch.length;
        
        await job.updateProgress({ 
          status: 'sending',
          total,
          sent,
          percentage: Math.floor((sent / total) * 100)
        });

        // Rate limiting: หยุดพักระหว่าง batches
        if (i + batchSize < targetUserIds.length) {
          await this.sleep(100);
        }

      } catch (error) {
        logger.error(`Batch send failed at offset ${i}:`, error);
        throw error;
      }
    }

    await job.updateProgress({ status: 'completed', total, sent });

    logger.info(`Broadcast completed: ${name}`, {
      jobId: job.id,
      totalSent: sent
    });

    return { success: true, totalSent: sent, total };
  }

  async getUserIds(segments) {
    // ดึง user IDs จาก database ตาม segments
    const { User } = require('../models');
    const where = { isFollowing: true };
    
    if (segments && segments.length > 0) {
      where.tags = { [Op.overlap]: segments };
    }

    const users = await User.findAll({
      where,
      attributes: ['lineUserId'],
      raw: true
    });

    return users.map(u => u.lineUserId);
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }
}

module.exports = { BroadcastQueue, BroadcastWorker };
```

---

## 4. Report Generation Queue

### 4.1 Long-Running Jobs

```javascript
// queues/reportQueue.js
const { Queue, Worker } = require('bullmq');
const { redisConnection, QUEUE_NAMES } = require('./setup');
const ExcelJS = require('exceljs');
const { S3Client, PutObjectCommand } = require('@aws-sdk/client-s3');
const logger = require('../utils/logger');

const s3 = new S3Client({ region: process.env.AWS_REGION });

class ReportQueue {
  constructor() {
    this.queue = new Queue(QUEUE_NAMES.REPORTS, {
      connection: redisConnection,
      defaultJobOptions: {
        attempts: 2,
        removeOnComplete: { age: 24 * 3600 },
        removeOnFail: { age: 7 * 24 * 3600 },
      }
    });
  }

  async addReport(params) {
    const {
      reportType,
      dateRange,
      filters,
      requestedBy,
      notifyUserId, // LINE user ID ที่จะส่งแจ้งเตือน
    } = params;

    const job = await this.queue.add('generate-report', {
      reportType,
      dateRange,
      filters,
      requestedBy,
      notifyUserId,
      requestedAt: Date.now()
    }, {
      priority: 1,
    });

    return { jobId: job.id };
  }
}

class ReportWorker {
  constructor() {
    this.worker = new Worker(
      QUEUE_NAMES.REPORTS,
      this.processJob.bind(this),
      {
        connection: redisConnection,
        concurrency: 2,  // สร้าง report ได้พร้อมกัน 2 อัน
        limiter: {
          max: 10,
          duration: 60000, // 10 reports per minute
        }
      }
    );
  }

  async processJob(job) {
    const { reportType, dateRange, filters, notifyUserId } = job.data;
    
    logger.info(`Generating report: ${reportType}`, {
      jobId: job.id,
      dateRange
    });

    try {
      await job.updateProgress({ status: 'querying', percentage: 10 });

      // ดึงข้อมูล
      const data = await this.fetchReportData(reportType, dateRange, filters);
      
      await job.updateProgress({ status: 'processing', percentage: 40 });

      // สร้าง Excel file
      const buffer = await this.generateExcel(reportType, data);
      
      await job.updateProgress({ status: 'uploading', percentage: 70 });

      // Upload ไป S3
      const filename = `reports/${reportType}-${Date.now()}.xlsx`;
      await s3.send(new PutObjectCommand({
        Bucket: process.env.S3_BUCKET,
        Key: filename,
        Body: buffer,
        ContentType: 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        ContentDisposition: `attachment; filename="${reportType}-report.xlsx"`,
        Expires: new Date(Date.now() + 7 * 24 * 3600 * 1000), // 7 days
      }));

      const downloadUrl = `https://${process.env.S3_BUCKET}.s3.amazonaws.com/${filename}`;
      
      await job.updateProgress({ status: 'notifying', percentage: 90 });

      // แจ้ง user ผ่าน LINE
      if (notifyUserId) {
        await this.notifyUser(notifyUserId, reportType, downloadUrl);
      }

      await job.updateProgress({ status: 'completed', percentage: 100 });

      return { 
        success: true, 
        downloadUrl,
        rowCount: data.length 
      };

    } catch (error) {
      logger.error(`Report generation failed:`, error);
      
      if (notifyUserId) {
        await this.notifyUserError(notifyUserId, reportType, error.message);
      }
      
      throw error;
    }
  }

  async fetchReportData(reportType, dateRange, filters) {
    const { startDate, endDate } = dateRange;
    
    switch (reportType) {
      case 'users':
        return this.fetchUsersReport(startDate, endDate, filters);
      case 'messages':
        return this.fetchMessagesReport(startDate, endDate, filters);
      case 'engagement':
        return this.fetchEngagementReport(startDate, endDate, filters);
      default:
        throw new Error(`Unknown report type: ${reportType}`);
    }
  }

  async generateExcel(reportType, data) {
    const workbook = new ExcelJS.Workbook();
    const worksheet = workbook.addWorksheet('Report');

    if (data.length === 0) {
      worksheet.addRow(['No data found']);
      return workbook.xlsx.writeBuffer();
    }

    // Headers
    const headers = Object.keys(data[0]);
    const headerRow = worksheet.addRow(headers);
    headerRow.font = { bold: true };
    headerRow.fill = {
      type: 'pattern',
      pattern: 'solid',
      fgColor: { argb: 'FF00B900' }
    };
    headerRow.font = { bold: true, color: { argb: 'FFFFFFFF' } };

    // Data rows
    data.forEach(row => {
      worksheet.addRow(Object.values(row));
    });

    // Auto-fit columns
    worksheet.columns.forEach(column => {
      let maxLength = 0;
      column.eachCell({ includeEmpty: true }, (cell) => {
        const cellLength = cell.value ? cell.value.toString().length : 10;
        if (cellLength > maxLength) maxLength = cellLength;
      });
      column.width = Math.min(maxLength + 2, 50);
    });

    return workbook.xlsx.writeBuffer();
  }

  async notifyUser(lineUserId, reportType, downloadUrl) {
    const { MessageSendingQueue } = require('./messageSendingQueue');
    const queue = new MessageSendingQueue();

    await queue.addPush(lineUserId, [{
      type: 'flex',
      altText: `รายงาน ${reportType} พร้อมแล้ว`,
      contents: {
        type: 'bubble',
        header: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'text',
            text: '📊 รายงานพร้อมแล้ว!',
            weight: 'bold',
            color: '#ffffff'
          }],
          backgroundColor: '#00B900'
        },
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: `รายงาน: ${reportType}`,
              wrap: true
            },
            {
              type: 'text',
              text: 'คลิกปุ่มด้านล่างเพื่อดาวน์โหลด',
              size: 'sm',
              color: '#666666'
            }
          ]
        },
        footer: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'button',
            action: {
              type: 'uri',
              label: '⬇️ ดาวน์โหลดรายงาน',
              uri: downloadUrl
            },
            style: 'primary',
            color: '#00B900'
          }]
        }
      }
    }]);
  }
}

module.exports = { ReportQueue, ReportWorker };
```

---

## 5. Scheduled Tasks Queue

### 5.1 Cron-based Scheduled Messages

```javascript
// queues/schedulerQueue.js
const { Queue, Worker } = require('bullmq');
const { QueueScheduler } = require('bullmq');
const cron = require('node-cron');
const { redisConnection, QUEUE_NAMES } = require('./setup');
const logger = require('../utils/logger');

class SchedulerQueue {
  constructor() {
    this.queue = new Queue(QUEUE_NAMES.SCHEDULED, {
      connection: redisConnection,
    });

    // QueueScheduler ต้องรัน 1 instance เท่านั้น
    this.scheduler = new QueueScheduler(QUEUE_NAMES.SCHEDULED, {
      connection: redisConnection,
    });
  }

  // เพิ่ม scheduled message
  async scheduleMessage(params) {
    const {
      name,
      cronExpression,  // หรือ
      scheduledAt,     // เลือกอย่างใดอย่างหนึ่ง
      messages,
      recipients,      // { type: 'all' | 'segment' | 'specific', value: [...] }
    } = params;

    if (cronExpression) {
      // Recurring schedule
      await this.queue.add(
        name,
        { messages, recipients, scheduledAt: 'recurring' },
        {
          repeat: {
            cron: cronExpression,
            tz: 'Asia/Bangkok',
          },
          removeOnComplete: { age: 24 * 3600 },
        }
      );
      
      logger.info(`Scheduled recurring message: ${name}`, { cronExpression });
    } else if (scheduledAt) {
      // One-time schedule
      const delay = new Date(scheduledAt) - Date.now();
      
      if (delay < 0) {
        throw new Error('scheduledAt must be in the future');
      }

      await this.queue.add(
        name,
        { messages, recipients, scheduledAt },
        {
          delay,
          removeOnComplete: { age: 7 * 24 * 3600 },
        }
      );

      logger.info(`Scheduled one-time message: ${name}`, { scheduledAt });
    }
  }

  // ลบ scheduled message
  async cancelSchedule(name) {
    const repeatableJobs = await this.queue.getRepeatableJobs();
    const job = repeatableJobs.find(j => j.name === name);
    
    if (job) {
      await this.queue.removeRepeatableByKey(job.key);
      logger.info(`Cancelled recurring schedule: ${name}`);
      return true;
    }
    
    return false;
  }

  // ดู scheduled messages ทั้งหมด
  async listSchedules() {
    const repeatableJobs = await this.queue.getRepeatableJobs();
    return repeatableJobs.map(job => ({
      name: job.name,
      cron: job.cron,
      tz: job.tz,
      next: job.next,
      endDate: job.endDate,
    }));
  }
}

// ตัวอย่าง scheduled jobs
async function setupDefaultSchedules(schedulerQueue) {
  // ส่ง daily promotion ทุกวันเวลา 9:00 น.
  await schedulerQueue.scheduleMessage({
    name: 'daily-promotion',
    cronExpression: '0 9 * * *',
    messages: [{
      type: 'flex',
      altText: 'โปรโมชั่นประจำวัน',
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'text',
            text: '🌟 โปรโมชั่นประจำวันนี้!',
            weight: 'bold',
            size: 'xl'
          }]
        }
      }
    }],
    recipients: { type: 'segment', value: ['vip', 'active'] }
  });

  // สรุปรายสัปดาห์ทุกวันจันทร์เวลา 8:00 น.
  await schedulerQueue.scheduleMessage({
    name: 'weekly-summary',
    cronExpression: '0 8 * * 1',
    messages: [{
      type: 'text',
      text: 'สรุปรายการสัปดาห์ที่ผ่านมา 📊\nคลิก "เมนู" เพื่อดูรายละเอียด'
    }],
    recipients: { type: 'segment', value: ['subscribed'] }
  });
}

module.exports = { SchedulerQueue, setupDefaultSchedules };
```

---

## 6. Priority Queue

### 6.1 Priority Configuration

```javascript
// queues/priorityQueue.js
const { Queue, Worker } = require('bullmq');
const { redisConnection } = require('./setup');

// Priority levels
const PRIORITY = {
  CRITICAL: 1,    // Reply messages (time-sensitive)
  HIGH: 3,        // Push to VIP users
  NORMAL: 5,      // Regular push messages
  LOW: 7,         // Notifications
  BACKGROUND: 10  // Analytics, reports
};

class PriorityMessageQueue {
  constructor() {
    this.queue = new Queue('priority:messages', {
      connection: redisConnection,
    });
  }

  async addCritical(data) {
    return this.queue.add('critical', data, { priority: PRIORITY.CRITICAL });
  }

  async addHigh(data) {
    return this.queue.add('high', data, { priority: PRIORITY.HIGH });
  }

  async addNormal(data) {
    return this.queue.add('normal', data, { priority: PRIORITY.NORMAL });
  }

  async addLow(data) {
    return this.queue.add('low', data, { priority: PRIORITY.LOW });
  }

  async addBackground(data) {
    return this.queue.add('background', data, { priority: PRIORITY.BACKGROUND });
  }
}

module.exports = { PriorityMessageQueue, PRIORITY };
```

---

## 7. Dead Letter Queue

### 7.1 DLQ Handler

```javascript
// queues/deadLetterQueue.js
const { Queue, Worker } = require('bullmq');
const { redisConnection } = require('./setup');
const logger = require('../utils/logger');

const DLQ_NAME = 'dead-letter-queue';

class DeadLetterQueue {
  constructor() {
    this.queue = new Queue(DLQ_NAME, {
      connection: redisConnection,
    });

    // Worker สำหรับ process DLQ jobs
    this.worker = new Worker(
      DLQ_NAME,
      this.processDLQJob.bind(this),
      {
        connection: redisConnection,
        concurrency: 1,
      }
    );
  }

  async processDLQJob(job) {
    const { originalJob, error, source } = job.data;
    
    logger.error(`Processing DLQ job from ${source}:`, {
      jobId: originalJob.id,
      jobName: originalJob.name,
      failedReason: error,
      attemptsMade: originalJob.attemptsMade,
      data: originalJob.data
    });

    // บันทึกลง database เพื่อ investigate ภายหลัง
    await this.saveToDatabase(originalJob, error, source);

    // แจ้งเตือน admin
    await this.alertAdmin(originalJob, error, source);
  }

  async saveToDatabase(job, error, source) {
    try {
      const { FailedJob } = require('../models');
      await FailedJob.create({
        jobId: job.id,
        jobName: job.name,
        queue: source,
        payload: job.data,
        error: error,
        failedAt: new Date()
      });
    } catch (dbError) {
      logger.error('Failed to save DLQ job to database:', dbError);
    }
  }

  async alertAdmin(job, error, source) {
    // ส่งแจ้งเตือนผ่าน Slack หรือ LINE
    const adminWebhook = process.env.ADMIN_ALERT_WEBHOOK;
    if (!adminWebhook) return;

    try {
      const axios = require('axios');
      await axios.post(adminWebhook, {
        text: `⚠️ Job failed after max retries\n*Queue:* ${source}\n*Job:* ${job.name}\n*Error:* ${error}`,
      });
    } catch (alertError) {
      logger.error('Failed to send admin alert:', alertError);
    }
  }

  // ลอง retry jobs ใน DLQ
  async retryAllDLQ() {
    const jobs = await this.queue.getJobs(['waiting', 'failed']);
    
    for (const job of jobs) {
      try {
        await job.retry();
        logger.info(`Retrying DLQ job: ${job.id}`);
      } catch (error) {
        logger.error(`Failed to retry DLQ job ${job.id}:`, error);
      }
    }

    return jobs.length;
  }
}

// Middleware สำหรับ capture failed jobs
function setupDLQCapture(sourceQueue, dlq) {
  const queueEvents = sourceQueue.on('failed', async (job, err) => {
    if (job.attemptsMade >= job.opts.attempts) {
      // Job ถึง max retries แล้ว ส่งไป DLQ
      await dlq.queue.add('failed-job', {
        originalJob: {
          id: job.id,
          name: job.name,
          data: job.data,
          attemptsMade: job.attemptsMade,
          opts: job.opts
        },
        error: err.message,
        source: sourceQueue.name,
        failedAt: Date.now()
      });
    }
  });
}

module.exports = { DeadLetterQueue, setupDLQCapture };
```

---

## 8. Queue Monitoring Dashboard

### 8.1 Bull Board Setup

```javascript
// monitoring/queueDashboard.js
const express = require('express');
const { createBullBoard } = require('@bull-board/api');
const { BullMQAdapter } = require('@bull-board/api/bullMQAdapter');
const { ExpressAdapter } = require('@bull-board/express');
const { Queue } = require('bullmq');
const { redisConnection, QUEUE_NAMES } = require('../queues/setup');

const app = express();

// สร้าง queue adapters
const queues = Object.values(QUEUE_NAMES).map(name => 
  new BullMQAdapter(new Queue(name, { connection: redisConnection }))
);

const serverAdapter = new ExpressAdapter();
serverAdapter.setBasePath('/admin/queues');

createBullBoard({
  queues,
  serverAdapter,
  options: {
    uiConfig: {
      boardTitle: 'LINE Bot Queue Monitor',
      favIcon: {
        default: 'https://line.me/favicon.ico',
      },
    }
  }
});

app.use('/admin/queues', serverAdapter.getRouter());

// Custom metrics endpoint
app.get('/admin/metrics', async (req, res) => {
  const metrics = await Promise.all(
    Object.entries(QUEUE_NAMES).map(async ([key, name]) => {
      const queue = new Queue(name, { connection: redisConnection });
      const [waiting, active, completed, failed, delayed] = await Promise.all([
        queue.getWaitingCount(),
        queue.getActiveCount(),
        queue.getCompletedCount(),
        queue.getFailedCount(),
        queue.getDelayedCount(),
      ]);
      
      return {
        name,
        waiting,
        active,
        completed,
        failed,
        delayed,
        total: waiting + active + delayed
      };
    })
  );

  res.json({ queues: metrics, timestamp: new Date().toISOString() });
});

module.exports = app;
```

### 8.2 Custom Monitoring Metrics

```javascript
// monitoring/queueMetrics.js
const { Queue } = require('bullmq');
const { redisConnection, QUEUE_NAMES } = require('../queues/setup');
const client = require('prom-client');
const logger = require('../utils/logger');

// Prometheus metrics สำหรับ queues
const queueWaiting = new client.Gauge({
  name: 'bullmq_queue_waiting',
  help: 'Number of jobs waiting in queue',
  labelNames: ['queue']
});

const queueActive = new client.Gauge({
  name: 'bullmq_queue_active',
  help: 'Number of jobs currently being processed',
  labelNames: ['queue']
});

const queueFailed = new client.Gauge({
  name: 'bullmq_queue_failed',
  help: 'Number of failed jobs',
  labelNames: ['queue']
});

const queueDelayed = new client.Gauge({
  name: 'bullmq_queue_delayed',
  help: 'Number of delayed jobs',
  labelNames: ['queue']
});

const jobDuration = new client.Histogram({
  name: 'bullmq_job_duration_seconds',
  help: 'Duration of completed jobs',
  labelNames: ['queue', 'job_name'],
  buckets: [0.1, 0.5, 1, 2, 5, 10, 30, 60]
});

// Collect metrics ทุก 15 วินาที
async function collectQueueMetrics() {
  for (const [key, queueName] of Object.entries(QUEUE_NAMES)) {
    try {
      const queue = new Queue(queueName, { connection: redisConnection });
      
      const [waiting, active, failed, delayed] = await Promise.all([
        queue.getWaitingCount(),
        queue.getActiveCount(),
        queue.getFailedCount(),
        queue.getDelayedCount(),
      ]);

      queueWaiting.set({ queue: queueName }, waiting);
      queueActive.set({ queue: queueName }, active);
      queueFailed.set({ queue: queueName }, failed);
      queueDelayed.set({ queue: queueName }, delayed);

    } catch (error) {
      logger.error(`Error collecting metrics for ${queueName}:`, error);
    }
  }
}

// Start collecting
const METRICS_INTERVAL = 15000;
setInterval(collectQueueMetrics, METRICS_INTERVAL);
collectQueueMetrics(); // Initial collection

module.exports = {
  collectQueueMetrics,
  queueWaiting,
  queueActive,
  queueFailed,
  queueDelayed,
  jobDuration
};
```

---

## 9. RabbitMQ Integration

### 9.1 RabbitMQ Consumer Pattern

```javascript
// rabbitmq/consumer.js
const amqp = require('amqplib');
const logger = require('../utils/logger');

class RabbitMQConsumer {
  constructor(config) {
    this.config = {
      url: process.env.RABBITMQ_URL || 'amqp://localhost:5672',
      prefetch: 10,
      reconnectDelay: 5000,
      maxRetries: 3,
      ...config
    };
    
    this.connection = null;
    this.channel = null;
    this.isConnecting = false;
  }

  async connect() {
    if (this.isConnecting) return;
    this.isConnecting = true;

    try {
      this.connection = await amqp.connect(this.config.url);
      this.channel = await this.connection.createChannel();
      
      await this.channel.prefetch(this.config.prefetch);

      this.connection.on('error', (err) => {
        logger.error('RabbitMQ connection error:', err);
        this.reconnect();
      });

      this.connection.on('close', () => {
        logger.warn('RabbitMQ connection closed, reconnecting...');
        this.reconnect();
      });

      this.isConnecting = false;
      logger.info('Connected to RabbitMQ');
      
    } catch (error) {
      this.isConnecting = false;
      logger.error('Failed to connect to RabbitMQ:', error);
      await this.reconnect();
    }
  }

  async reconnect() {
    await new Promise(resolve => setTimeout(resolve, this.config.reconnectDelay));
    await this.connect();
  }

  async consume(queueName, handler, options = {}) {
    if (!this.channel) {
      await this.connect();
    }

    await this.channel.assertQueue(queueName, {
      durable: true,
      arguments: {
        'x-dead-letter-exchange': `${queueName}.dlx`,
        'x-dead-letter-routing-key': 'dead.letter',
        'x-message-ttl': 3600000, // 1 hour TTL
        ...options.queueArguments
      }
    });

    // Dead letter queue
    await this.channel.assertExchange(`${queueName}.dlx`, 'direct', { durable: true });
    await this.channel.assertQueue(`${queueName}.dead-letter`, { durable: true });
    await this.channel.bindQueue(`${queueName}.dead-letter`, `${queueName}.dlx`, 'dead.letter');

    this.channel.consume(queueName, async (msg) => {
      if (!msg) return;

      const startTime = Date.now();
      let retryCount = 0;

      try {
        const content = JSON.parse(msg.content.toString());
        retryCount = msg.properties.headers?.['x-retry-count'] || 0;

        await handler(content, msg);
        
        this.channel.ack(msg);
        
        logger.debug(`Message processed in ${Date.now() - startTime}ms`, {
          queue: queueName,
          retryCount
        });

      } catch (error) {
        logger.error(`Error processing message from ${queueName}:`, error);

        if (retryCount < this.config.maxRetries) {
          // Retry with delay
          const delay = Math.pow(2, retryCount) * 1000;
          
          setTimeout(() => {
            this.channel.publish('', queueName, msg.content, {
              persistent: true,
              headers: {
                ...msg.properties.headers,
                'x-retry-count': retryCount + 1,
                'x-original-error': error.message
              }
            });
          }, delay);

          this.channel.ack(msg);
        } else {
          // Send to DLQ
          this.channel.nack(msg, false, false);
          logger.warn(`Message sent to DLQ after ${retryCount} retries`);
        }
      }
    }, { noAck: false });

    logger.info(`Consuming from queue: ${queueName}`);
  }
}

module.exports = { RabbitMQConsumer };
```

---

## 10. Worker Scaling

### 10.1 Dynamic Worker Scaling

```javascript
// workers/dynamicScaler.js
const { Worker } = require('bullmq');
const { redisConnection } = require('../queues/setup');
const logger = require('../utils/logger');

class DynamicWorkerScaler {
  constructor(config) {
    this.config = {
      queueName: config.queueName,
      workerFactory: config.workerFactory,
      minWorkers: config.minWorkers || 1,
      maxWorkers: config.maxWorkers || 10,
      scaleUpThreshold: config.scaleUpThreshold || 100,    // jobs waiting
      scaleDownThreshold: config.scaleDownThreshold || 10, // jobs waiting
      scaleUpCooldown: config.scaleUpCooldown || 60000,    // 1 minute
      scaleDownCooldown: config.scaleDownCooldown || 300000, // 5 minutes
      checkInterval: config.checkInterval || 30000,        // 30 seconds
    };

    this.workers = [];
    this.lastScaleUp = 0;
    this.lastScaleDown = 0;
    this.monitoring = null;
  }

  async start() {
    // เริ่มต้นด้วย minWorkers
    for (let i = 0; i < this.config.minWorkers; i++) {
      await this.addWorker();
    }

    // เริ่ม monitoring loop
    this.monitoring = setInterval(
      () => this.checkAndScale(),
      this.config.checkInterval
    );

    logger.info(`DynamicWorkerScaler started`, {
      queue: this.config.queueName,
      workers: this.workers.length
    });
  }

  async addWorker() {
    const worker = this.config.workerFactory();
    this.workers.push(worker);
    logger.info(`Added worker #${this.workers.length} for ${this.config.queueName}`);
  }

  async removeWorker() {
    if (this.workers.length <= this.config.minWorkers) return;
    
    const worker = this.workers.pop();
    await worker.close();
    logger.info(`Removed worker, remaining: ${this.workers.length}`);
  }

  async checkAndScale() {
    try {
      const { Queue } = require('bullmq');
      const queue = new Queue(this.config.queueName, { connection: redisConnection });
      
      const [waiting, active] = await Promise.all([
        queue.getWaitingCount(),
        queue.getActiveCount(),
      ]);

      const now = Date.now();
      
      logger.debug(`Queue status: waiting=${waiting}, active=${active}, workers=${this.workers.length}`);

      // Scale up
      if (waiting > this.config.scaleUpThreshold && 
          this.workers.length < this.config.maxWorkers &&
          now - this.lastScaleUp > this.config.scaleUpCooldown) {
        
        const workersToAdd = Math.min(
          Math.ceil(waiting / this.config.scaleUpThreshold),
          this.config.maxWorkers - this.workers.length
        );
        
        for (let i = 0; i < workersToAdd; i++) {
          await this.addWorker();
        }
        
        this.lastScaleUp = now;
        logger.info(`Scaled up to ${this.workers.length} workers`);
      }
      
      // Scale down
      else if (waiting < this.config.scaleDownThreshold && 
               active === 0 &&
               this.workers.length > this.config.minWorkers &&
               now - this.lastScaleDown > this.config.scaleDownCooldown) {
        
        await this.removeWorker();
        this.lastScaleDown = now;
        logger.info(`Scaled down to ${this.workers.length} workers`);
      }

    } catch (error) {
      logger.error('Error in checkAndScale:', error);
    }
  }

  async stop() {
    if (this.monitoring) {
      clearInterval(this.monitoring);
    }

    await Promise.all(this.workers.map(w => w.close()));
    logger.info(`All workers stopped for ${this.config.queueName}`);
  }
}

// ตัวอย่างการใช้งาน
const webhookScaler = new DynamicWorkerScaler({
  queueName: 'webhook:events',
  workerFactory: () => new Worker(
    'webhook:events',
    require('../handlers/webhookEventHandler'),
    { connection: redisConnection, concurrency: 5 }
  ),
  minWorkers: 2,
  maxWorkers: 10,
  scaleUpThreshold: 500,
  scaleDownThreshold: 50,
});

module.exports = { DynamicWorkerScaler, webhookScaler };
```

---

## 11. Complete Docker Compose สำหรับ Queue System

```yaml
# docker-compose.queue.yml
version: '3.9'

services:
  redis-queue:
    image: redis:7.2-alpine
    restart: unless-stopped
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 2gb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
    ports:
      - "6379:6379"
    volumes:
      - redis_queue_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "--no-auth-warning", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - queue-network

  redis-exporter:
    image: oliver006/redis_exporter:latest
    environment:
      REDIS_ADDR: redis://redis-queue:6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    ports:
      - "9121:9121"
    depends_on:
      - redis-queue
    networks:
      - queue-network

  webhook-worker:
    build:
      context: .
      dockerfile: workers/Dockerfile
    restart: unless-stopped
    environment:
      WORKER_TYPE: webhook
      REDIS_HOST: redis-queue
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      LINE_CHANNEL_ACCESS_TOKEN: ${LINE_CHANNEL_ACCESS_TOKEN}
      WORKER_CONCURRENCY: 20
      NODE_ENV: production
    depends_on:
      redis-queue:
        condition: service_healthy
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '1'
          memory: 512M
    networks:
      - queue-network

  notification-worker:
    build:
      context: .
      dockerfile: workers/Dockerfile
    restart: unless-stopped
    environment:
      WORKER_TYPE: notification
      REDIS_HOST: redis-queue
      WORKER_CONCURRENCY: 50
      LINE_CHANNEL_ACCESS_TOKEN: ${LINE_CHANNEL_ACCESS_TOKEN}
    deploy:
      replicas: 2
    networks:
      - queue-network

  report-worker:
    build:
      context: .
      dockerfile: workers/Dockerfile
    restart: unless-stopped
    environment:
      WORKER_TYPE: report
      REDIS_HOST: redis-queue
      WORKER_CONCURRENCY: 2
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
      S3_BUCKET: ${S3_BUCKET}
    deploy:
      replicas: 1
      resources:
        limits:
          cpus: '2'
          memory: 1G
    networks:
      - queue-network

  queue-dashboard:
    build:
      context: .
      dockerfile: monitoring/Dockerfile
    restart: unless-stopped
    environment:
      REDIS_HOST: redis-queue
      REDIS_PASSWORD: ${REDIS_PASSWORD}
      DASHBOARD_PORT: 3100
      ADMIN_USERNAME: admin
      ADMIN_PASSWORD: ${DASHBOARD_PASSWORD}
    ports:
      - "3100:3100"
    networks:
      - queue-network

volumes:
  redis_queue_data:

networks:
  queue-network:
    driver: bridge
```

---

## 12. Celery สำหรับ Python Workers

### 12.1 Celery Setup

```python
# celery_app.py
from celery import Celery
from kombu import Exchange, Queue
import os

# Celery configuration
app = Celery('linebot_workers')

app.conf.update(
    broker_url=os.getenv('REDIS_URL', 'redis://localhost:6379/0'),
    result_backend=os.getenv('REDIS_URL', 'redis://localhost:6379/0'),
    
    # Task serialization
    task_serializer='json',
    accept_content=['json'],
    result_serializer='json',
    timezone='Asia/Bangkok',
    enable_utc=True,
    
    # Queue configuration
    task_queues=[
        Queue('webhook_events', Exchange('webhook'), routing_key='webhook.#',
              queue_arguments={'x-max-priority': 10}),
        Queue('messages', Exchange('messages'), routing_key='message.#'),
        Queue('reports', Exchange('reports'), routing_key='report.#'),
        Queue('analytics', Exchange('analytics'), routing_key='analytics.#'),
    ],
    
    task_default_queue='messages',
    task_default_exchange='messages',
    task_default_routing_key='message.default',
    
    # Worker configuration
    worker_prefetch_multiplier=4,
    task_acks_late=True,
    worker_max_tasks_per_child=1000,  # Restart worker ทุก 1000 tasks
    
    # Retry configuration
    task_max_retries=3,
    task_default_retry_delay=60,
    
    # Result expiration
    result_expires=3600,
)

# Route tasks to queues
app.conf.task_routes = {
    'tasks.webhook.*': {'queue': 'webhook_events'},
    'tasks.message.*': {'queue': 'messages'},
    'tasks.report.*': {'queue': 'reports'},
    'tasks.analytics.*': {'queue': 'analytics'},
}
```

```python
# tasks/message_tasks.py
from celery_app import app
import requests
import os
import logging

logger = logging.getLogger(__name__)

LINE_API_URL = 'https://api.line.me/v2/bot/message'
ACCESS_TOKEN = os.getenv('LINE_CHANNEL_ACCESS_TOKEN')

HEADERS = {
    'Authorization': f'Bearer {ACCESS_TOKEN}',
    'Content-Type': 'application/json'
}

@app.task(
    name='tasks.message.send_reply',
    bind=True,
    max_retries=1,  # Reply token ใช้ได้ครั้งเดียว
    priority=10,
    queue='messages'
)
def send_reply_message(self, reply_token: str, messages: list, user_id: str = None):
    """ส่ง reply message ไปยัง LINE"""
    try:
        response = requests.post(
            f'{LINE_API_URL}/reply',
            json={
                'replyToken': reply_token,
                'messages': messages
            },
            headers=HEADERS,
            timeout=5
        )
        
        response.raise_for_status()
        logger.info(f'Reply sent successfully to {user_id}')
        return {'success': True, 'user_id': user_id}
        
    except requests.HTTPError as e:
        if e.response.status_code == 400:
            # Reply token invalid/expired
            logger.warning(f'Reply token expired for {user_id}')
            return {'success': False, 'reason': 'token_expired'}
        raise self.retry(exc=e, countdown=5)

@app.task(
    name='tasks.message.send_push',
    bind=True,
    max_retries=3,
    priority=5,
    queue='messages'
)
def send_push_message(self, user_id: str, messages: list):
    """ส่ง push message ไปยัง user"""
    try:
        response = requests.post(
            f'{LINE_API_URL}/push',
            json={
                'to': user_id,
                'messages': messages if isinstance(messages, list) else [messages]
            },
            headers=HEADERS,
            timeout=10
        )
        
        response.raise_for_status()
        return {'success': True, 'user_id': user_id}
        
    except requests.HTTPError as e:
        if e.response.status_code == 429:
            # Rate limited - retry with longer delay
            raise self.retry(exc=e, countdown=60)
        raise self.retry(exc=e, countdown=10)

@app.task(
    name='tasks.message.send_multicast',
    bind=True,
    max_retries=3,
    priority=3,
    queue='messages'
)
def send_multicast_message(self, user_ids: list, messages: list, chunk_index: int = 0):
    """ส่ง multicast message ไปยังหลาย users"""
    # แบ่งเป็น chunks ละ 500
    CHUNK_SIZE = 500
    
    for i in range(0, len(user_ids), CHUNK_SIZE):
        chunk = user_ids[i:i + CHUNK_SIZE]
        
        try:
            response = requests.post(
                f'{LINE_API_URL}/multicast',
                json={
                    'to': chunk,
                    'messages': messages
                },
                headers=HEADERS,
                timeout=30
            )
            
            response.raise_for_status()
            logger.info(f'Multicast sent to {len(chunk)} users (chunk {i//CHUNK_SIZE + 1})')
            
        except requests.HTTPError as e:
            logger.error(f'Multicast chunk {i//CHUNK_SIZE + 1} failed: {e}')
            raise self.retry(exc=e, countdown=30)

@app.task(
    name='tasks.report.generate',
    bind=True,
    max_retries=2,
    priority=1,
    queue='reports',
    time_limit=1800,  # 30 minutes max
    soft_time_limit=1500  # Soft limit 25 minutes
)
def generate_report(self, report_type: str, date_range: dict, 
                    filters: dict = None, notify_user_id: str = None):
    """สร้าง report"""
    from celery.exceptions import SoftTimeLimitExceeded
    
    try:
        logger.info(f'Generating {report_type} report')
        
        # Fetch data
        self.update_state(state='PROGRESS', meta={'status': 'fetching_data', 'progress': 10})
        data = fetch_report_data(report_type, date_range, filters)
        
        # Generate Excel
        self.update_state(state='PROGRESS', meta={'status': 'generating_excel', 'progress': 50})
        file_path = generate_excel(report_type, data)
        
        # Upload to S3
        self.update_state(state='PROGRESS', meta={'status': 'uploading', 'progress': 80})
        download_url = upload_to_s3(file_path, report_type)
        
        # Notify user
        if notify_user_id:
            send_push_message.delay(notify_user_id, [{
                'type': 'text',
                'text': f'📊 รายงาน {report_type} พร้อมแล้ว!\nดาวน์โหลด: {download_url}'
            }])
        
        return {'success': True, 'download_url': download_url, 'rows': len(data)}
        
    except SoftTimeLimitExceeded:
        logger.error('Report generation timed out')
        if notify_user_id:
            send_push_message.delay(notify_user_id, [{
                'type': 'text',
                'text': '⚠️ การสร้างรายงานใช้เวลานานเกินไป กรุณาลองใหม่'
            }])
        raise
```

---

## 13. สรุป Queue Management Best Practices

### 13.1 Checklist

```
Queue Design:
  ✅ แยก queues ตาม priority และ use case
  ✅ ตั้ง TTL สำหรับ jobs (ป้องกัน queue ใหญ่เกิน)
  ✅ Dead Letter Queue สำหรับ failed jobs
  ✅ Retry with exponential backoff

Worker Design:
  ✅ Concurrency ที่เหมาะสม (ไม่มากเกินไป)
  ✅ Graceful shutdown
  ✅ Health checks
  ✅ Memory limit (worker_max_tasks_per_child)

Monitoring:
  ✅ Queue depth alerts
  ✅ Failed job notifications
  ✅ Processing time metrics
  ✅ Bull Board dashboard

Production:
  ✅ Redis persistence (AOF)
  ✅ Redis replication
  ✅ Worker auto-scaling
  ✅ DLQ review process
```

### 13.2 Performance Benchmarks

```
BullMQ Throughput:
  - Simple jobs: 10,000-50,000 jobs/second
  - Complex jobs: 1,000-5,000 jobs/second
  - With concurrency=20: ~500 LINE messages/second

Redis Requirements:
  - Small (< 10,000 jobs/day): 256MB RAM
  - Medium (< 1M jobs/day): 1-2GB RAM
  - Large (> 1M jobs/day): 4GB+ RAM + Cluster

Worker Scaling:
  - 1 worker = ~100-500 messages/second (depending on job type)
  - Use DynamicWorkerScaler for auto-scaling
  - Monitor queue depth to determine scaling
```

บทถัดไปจะพูดถึง Multi-Bot Architecture สำหรับระบบที่มีหลาย LINE OA
