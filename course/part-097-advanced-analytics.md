# ส่วนที่ 97: Advanced Analytics และ Business Intelligence สำหรับ LINE OA

---

## สารบัญ

1. [Business Intelligence สำหรับ LINE OA](#1-business-intelligence-สำหรับ-line-oa)
2. [Analytics Data Model](#2-analytics-data-model)
3. [Apache Superset สำหรับ BI](#3-apache-superset-สำหรับ-bi)
4. [Google Looker Studio Integration](#4-google-looker-studio-integration)
5. [Custom Metrics Design](#5-custom-metrics-design)
6. [User Journey Analytics](#6-user-journey-analytics)
7. [Revenue Attribution](#7-revenue-attribution)
8. [Predictive Analytics](#8-predictive-analytics)
9. [Cohort Retention Analysis](#9-cohort-retention-analysis)
10. [Dashboard Templates สำหรับ LINE OA](#10-dashboard-templates-สำหรับ-line-oa)
11. [Executive Reporting Automation](#11-executive-reporting-automation)
12. [Complete BI Stack สำหรับ LINE OA](#12-complete-bi-stack-สำหรับ-line-oa)

---

## 1. Business Intelligence สำหรับ LINE OA

### 1.1 ทำไมต้องมี BI สำหรับ LINE OA

```
LINE OA Analytics Maturity Model:

Level 1: Basic Reporting
├── Follower count
├── Message read rate
└── Click rates

Level 2: Operational Analytics
├── User behavior tracking
├── Campaign performance
└── Response time metrics

Level 3: Strategic Analytics
├── Customer lifetime value
├── Cohort analysis
└── Attribution modeling

Level 4: Predictive Analytics
├── Churn prediction
├── Next best action
└── Revenue forecasting

Level 5: Prescriptive Analytics
├── Automated optimization
├── AI-driven personalization
└── Real-time decision engine
```

### 1.2 KPIs หลักสำหรับ LINE OA

| Category | KPI | เป้าหมาย | วิธีวัด |
|----------|-----|---------|---------|
| Growth | Follower Growth Rate | +10%/เดือน | (ใหม่-ออก)/ก่อนหน้า |
| Engagement | Open Rate | >50% | Opened/Sent |
| Engagement | Click Rate | >10% | Clicked/Opened |
| Retention | 30-day Retention | >60% | Active/Total |
| Revenue | ARPU | >฿500/เดือน | Revenue/Active Users |
| Service | Response Time | <2 นาที | Avg time to reply |
| Service | Resolution Rate | >85% | Resolved/Total tickets |

### 1.3 ระบบ Analytics Architecture

```
LINE OA Analytics Architecture:

┌─────────────────────────────────────────────────────────────┐
│                     Data Sources                            │
│  LINE Webhook  │  LIFF Events  │  Bot Logs  │  Sales Data  │
└────────────────┴───────────────┴────────────┴──────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   Data Pipeline   │
                    │  (Apache Kafka)   │
                    └─────────┬─────────┘
                              │
               ┌──────────────┼──────────────┐
               │              │              │
    ┌──────────▼───┐  ┌───────▼──────┐  ┌───▼──────────┐
    │  Data Lake   │  │   Real-time  │  │  Stream      │
    │  (S3/GCS)    │  │  (ClickHouse)│  │  Processing  │
    └──────────┬───┘  └───────┬──────┘  └───┬──────────┘
               │              │              │
               └──────────────┼──────────────┘
                              │
                    ┌─────────▼─────────┐
                    │   Data Warehouse   │
                    │  (BigQuery/Redshift)│
                    └─────────┬─────────┘
                              │
               ┌──────────────┼──────────────┐
               │              │              │
    ┌──────────▼───┐  ┌───────▼──────┐  ┌───▼──────────┐
    │   Superset   │  │ Looker Studio│  │  Custom API  │
    │   (BI Tool)  │  │  (Reports)   │  │  (Dashboard) │
    └──────────────┘  └──────────────┘  └──────────────┘
```

---

## 2. Analytics Data Model

### 2.1 Event Schema Design

```sql
-- Database Schema สำหรับ LINE OA Analytics

-- Users table
CREATE TABLE line_users (
    user_id VARCHAR(100) PRIMARY KEY,
    display_name VARCHAR(255),
    picture_url TEXT,
    language VARCHAR(10),
    followed_at TIMESTAMP,
    unfollowed_at TIMESTAMP,
    first_message_at TIMESTAMP,
    last_active_at TIMESTAMP,
    is_blocked BOOLEAN DEFAULT FALSE,
    tags JSONB DEFAULT '[]',
    custom_properties JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Events table (partitioned by date)
CREATE TABLE line_events (
    event_id UUID DEFAULT gen_random_uuid(),
    user_id VARCHAR(100) REFERENCES line_users(user_id),
    event_type VARCHAR(50) NOT NULL,
    event_source VARCHAR(20) NOT NULL, -- user, group, room
    message_type VARCHAR(20),
    message_content TEXT,
    metadata JSONB DEFAULT '{}',
    session_id VARCHAR(100),
    timestamp TIMESTAMP NOT NULL,
    date DATE GENERATED ALWAYS AS (timestamp::DATE) STORED
) PARTITION BY RANGE (date);

-- Create monthly partitions
CREATE TABLE line_events_2025_01 
    PARTITION OF line_events 
    FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

-- Messages sent by bot
CREATE TABLE bot_messages (
    message_id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id VARCHAR(100),
    message_type VARCHAR(20) NOT NULL,
    content JSONB,
    campaign_id VARCHAR(100),
    template_id VARCHAR(100),
    sent_at TIMESTAMP NOT NULL,
    delivered_at TIMESTAMP,
    opened_at TIMESTAMP,
    clicked_at TIMESTAMP,
    converted_at TIMESTAMP,
    conversion_value DECIMAL(10,2),
    is_delivered BOOLEAN DEFAULT FALSE,
    is_opened BOOLEAN DEFAULT FALSE,
    is_clicked BOOLEAN DEFAULT FALSE,
    is_converted BOOLEAN DEFAULT FALSE
);

-- Campaigns
CREATE TABLE campaigns (
    campaign_id VARCHAR(100) PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    type VARCHAR(50) NOT NULL, -- broadcast, segment, triggered
    status VARCHAR(20) DEFAULT 'draft',
    audience_size INTEGER,
    sent_count INTEGER DEFAULT 0,
    opened_count INTEGER DEFAULT 0,
    clicked_count INTEGER DEFAULT 0,
    converted_count INTEGER DEFAULT 0,
    revenue_generated DECIMAL(12,2) DEFAULT 0,
    start_at TIMESTAMP,
    end_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Sessions
CREATE TABLE user_sessions (
    session_id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    user_id VARCHAR(100) REFERENCES line_users(user_id),
    start_time TIMESTAMP NOT NULL,
    end_time TIMESTAMP,
    duration_seconds INTEGER,
    message_count INTEGER DEFAULT 0,
    intent_detected VARCHAR(100),
    outcome VARCHAR(50), -- resolved, escalated, abandoned
    satisfaction_score INTEGER
);

-- Orders (for e-commerce)
CREATE TABLE orders (
    order_id VARCHAR(100) PRIMARY KEY,
    user_id VARCHAR(100) REFERENCES line_users(user_id),
    campaign_id VARCHAR(100),
    status VARCHAR(50) NOT NULL,
    items JSONB NOT NULL,
    subtotal DECIMAL(10,2),
    discount DECIMAL(10,2) DEFAULT 0,
    tax DECIMAL(10,2) DEFAULT 0,
    total DECIMAL(10,2) NOT NULL,
    payment_method VARCHAR(50),
    source VARCHAR(50), -- direct_message, broadcast, liff, rich_menu
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Daily aggregations (materialized view)
CREATE MATERIALIZED VIEW daily_metrics AS
SELECT 
    DATE(timestamp) as date,
    COUNT(DISTINCT user_id) as active_users,
    COUNT(*) as total_events,
    COUNT(CASE WHEN event_type = 'follow' THEN 1 END) as new_followers,
    COUNT(CASE WHEN event_type = 'unfollow' THEN 1 END) as unfollowers,
    COUNT(CASE WHEN event_type = 'message' THEN 1 END) as messages_received
FROM line_events
GROUP BY DATE(timestamp)
WITH DATA;

CREATE INDEX ON daily_metrics (date);

-- Refresh schedule
CREATE OR REPLACE FUNCTION refresh_daily_metrics()
RETURNS void AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY daily_metrics;
END;
$$ LANGUAGE plpgsql;
```

### 2.2 Event Tracking Implementation

```javascript
// src/analytics/eventTracker.js
const { Pool } = require('pg');
const Redis = require('ioredis');
const config = require('../config');

class EventTracker {
  constructor() {
    this.db = new Pool({ connectionString: config.database.url });
    this.redis = new Redis(config.redis);
    this.buffer = [];
    this.bufferSize = 100;
    this.flushInterval = 5000; // 5 วินาที
    
    this.startFlushTimer();
  }

  // บันทึก event
  async track(userId, eventType, properties = {}) {
    const event = {
      userId,
      eventType,
      timestamp: new Date(),
      sessionId: await this.getOrCreateSession(userId),
      metadata: properties,
    };

    // เพิ่มใน buffer
    this.buffer.push(event);

    // Update real-time counters ใน Redis
    await this.updateRealtimeCounters(event);

    // Flush ถ้า buffer เต็ม
    if (this.buffer.length >= this.bufferSize) {
      await this.flush();
    }
  }

  async updateRealtimeCounters(event) {
    const pipeline = this.redis.pipeline();
    const today = new Date().toISOString().split('T')[0];
    
    // Daily counters
    pipeline.incr(`metrics:${today}:events:${event.eventType}`);
    pipeline.sadd(`metrics:${today}:active_users`, event.userId);
    pipeline.incr(`metrics:${today}:events:total`);
    
    // Expire after 7 days
    pipeline.expire(`metrics:${today}:events:${event.eventType}`, 604800);
    pipeline.expire(`metrics:${today}:active_users`, 604800);
    pipeline.expire(`metrics:${today}:events:total`, 604800);
    
    // Hourly counters
    const hour = new Date().getHours();
    pipeline.incr(`metrics:${today}:hour:${hour}:events`);
    pipeline.expire(`metrics:${today}:hour:${hour}:events`, 86400);
    
    await pipeline.exec();
  }

  async getOrCreateSession(userId) {
    const sessionKey = `session:active:${userId}`;
    let sessionId = await this.redis.get(sessionKey);
    
    if (!sessionId) {
      sessionId = `sess-${Date.now()}-${Math.random().toString(36).substr(2, 8)}`;
      await this.redis.set(sessionKey, sessionId, 'EX', 1800); // 30 นาที timeout
      
      // สร้าง session ใน DB
      await this.db.query(
        'INSERT INTO user_sessions (session_id, user_id, start_time) VALUES ($1, $2, NOW())',
        [sessionId, userId]
      );
    } else {
      // ต่ออายุ session
      await this.redis.expire(sessionKey, 1800);
      await this.db.query(
        'UPDATE user_sessions SET message_count = message_count + 1 WHERE session_id = $1',
        [sessionId]
      );
    }
    
    return sessionId;
  }

  startFlushTimer() {
    setInterval(() => {
      if (this.buffer.length > 0) {
        this.flush();
      }
    }, this.flushInterval);
  }

  async flush() {
    if (this.buffer.length === 0) return;
    
    const events = [...this.buffer];
    this.buffer = [];
    
    try {
      // Bulk insert
      const values = events.map((e, i) => 
        `($${i*5+1}, $${i*5+2}, $${i*5+3}, $${i*5+4}, $${i*5+5})`
      ).join(', ');
      
      const params = events.flatMap(e => [
        e.userId,
        e.eventType,
        'user',
        JSON.stringify(e.metadata),
        e.timestamp,
      ]);
      
      await this.db.query(
        `INSERT INTO line_events (user_id, event_type, event_source, metadata, timestamp) 
         VALUES ${values}`,
        params
      );
    } catch (error) {
      console.error('Error flushing events:', error);
      // นำ events กลับมาใส่ buffer
      this.buffer = [...events, ...this.buffer];
    }
  }

  // ดึง real-time metrics
  async getRealtimeMetrics(date = new Date().toISOString().split('T')[0]) {
    const pipeline = this.redis.pipeline();
    
    pipeline.get(`metrics:${date}:events:total`);
    pipeline.scard(`metrics:${date}:active_users`);
    pipeline.get(`metrics:${date}:events:follow`);
    pipeline.get(`metrics:${date}:events:message`);
    
    // Hourly breakdown
    for (let h = 0; h < 24; h++) {
      pipeline.get(`metrics:${date}:hour:${h}:events`);
    }
    
    const results = await pipeline.exec();
    
    const hourlyData = [];
    for (let h = 0; h < 24; h++) {
      hourlyData.push({
        hour: h,
        events: parseInt(results[4 + h][1] || 0),
      });
    }
    
    return {
      totalEvents: parseInt(results[0][1] || 0),
      activeUsers: parseInt(results[1][1] || 0),
      newFollowers: parseInt(results[2][1] || 0),
      messagesReceived: parseInt(results[3][1] || 0),
      hourlyBreakdown: hourlyData,
    };
  }
}

module.exports = new EventTracker();
```

### 2.3 User Property Tracking

```javascript
// src/analytics/userPropertyManager.js
class UserPropertyManager {
  constructor(db, redis) {
    this.db = db;
    this.redis = redis;
  }

  // บันทึก/อัพเดท user properties
  async setUserProperties(userId, properties) {
    const { rows } = await this.db.query(
      `INSERT INTO line_users (user_id, custom_properties)
       VALUES ($1, $2)
       ON CONFLICT (user_id) 
       DO UPDATE SET 
         custom_properties = line_users.custom_properties || $2,
         updated_at = NOW()
       RETURNING *`,
      [userId, JSON.stringify(properties)]
    );

    // Cache ใน Redis
    await this.redis.set(
      `user:properties:${userId}`,
      JSON.stringify(rows[0]),
      'EX',
      3600
    );

    return rows[0];
  }

  // ดึง user profile สมบูรณ์
  async getUserProfile(userId) {
    // Check cache first
    const cached = await this.redis.get(`user:properties:${userId}`);
    if (cached) return JSON.parse(cached);

    const { rows } = await this.db.query(
      `SELECT 
        u.*,
        COUNT(DISTINCT e.date) as active_days,
        COUNT(e.event_id) as total_events,
        MAX(e.timestamp) as last_activity,
        SUM(CASE WHEN o.status = 'completed' THEN o.total ELSE 0 END) as total_spent,
        COUNT(DISTINCT o.order_id) as total_orders
       FROM line_users u
       LEFT JOIN line_events e ON u.user_id = e.user_id
       LEFT JOIN orders o ON u.user_id = o.user_id
       WHERE u.user_id = $1
       GROUP BY u.user_id`,
      [userId]
    );

    if (rows.length === 0) return null;

    const profile = rows[0];
    await this.redis.set(
      `user:properties:${userId}`,
      JSON.stringify(profile),
      'EX',
      3600
    );

    return profile;
  }

  // คำนวณ user segment
  async calculateUserSegment(userId) {
    const profile = await this.getUserProfile(userId);
    if (!profile) return 'new';

    const daysSinceFollow = Math.floor(
      (Date.now() - new Date(profile.followed_at).getTime()) / 86400000
    );
    const daysSinceActive = Math.floor(
      (Date.now() - new Date(profile.last_activity).getTime()) / 86400000
    );

    // RFM-based segmentation
    const recency = daysSinceActive <= 7 ? 5 : 
                    daysSinceActive <= 14 ? 4 :
                    daysSinceActive <= 30 ? 3 :
                    daysSinceActive <= 60 ? 2 : 1;

    const frequency = profile.active_days >= 20 ? 5 :
                      profile.active_days >= 10 ? 4 :
                      profile.active_days >= 5 ? 3 :
                      profile.active_days >= 2 ? 2 : 1;

    const monetary = profile.total_spent >= 10000 ? 5 :
                     profile.total_spent >= 5000 ? 4 :
                     profile.total_spent >= 1000 ? 3 :
                     profile.total_spent >= 100 ? 2 : 1;

    const rfmScore = recency + frequency + monetary;

    if (rfmScore >= 13) return 'champions';
    if (rfmScore >= 10) return 'loyal_customers';
    if (rfmScore >= 7 && recency >= 4) return 'potential_loyalists';
    if (recency >= 4 && frequency <= 2) return 'new_customers';
    if (recency >= 3 && rfmScore >= 6) return 'promising';
    if (recency <= 2 && rfmScore >= 8) return 'at_risk';
    if (recency <= 2 && frequency >= 3) return 'cant_lose_them';
    if (recency <= 2 && rfmScore <= 4) return 'lost';
    return 'hibernating';
  }

  // อัพเดท segment ทุกวัน
  async updateAllSegments() {
    const { rows } = await this.db.query('SELECT user_id FROM line_users WHERE unfollowed_at IS NULL');
    
    const updates = await Promise.all(
      rows.map(async ({ user_id }) => {
        const segment = await this.calculateUserSegment(user_id);
        return { userId: user_id, segment };
      })
    );

    // Bulk update
    for (const update of updates) {
      await this.db.query(
        `UPDATE line_users 
         SET custom_properties = custom_properties || $2
         WHERE user_id = $1`,
        [update.userId, JSON.stringify({ segment: update.segment })]
      );
    }

    return updates.length;
  }
}

module.exports = UserPropertyManager;
```

---

## 3. Apache Superset สำหรับ BI

### 3.1 การติดตั้ง Superset

```yaml
# docker-compose.superset.yml
version: '3.8'

services:
  superset:
    image: apache/superset:latest
    environment:
      - SUPERSET_SECRET_KEY=${SUPERSET_SECRET_KEY}
      - DATABASE_URL=postgresql://superset:${DB_PASSWORD}@superset-db:5432/superset
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/1
      - SUPERSET_LOAD_EXAMPLES=no
    ports:
      - "8088:8088"
    depends_on:
      - superset-db
      - redis
    volumes:
      - ./superset_config.py:/app/superset/config.py
    command: >
      sh -c "
        superset db upgrade &&
        superset fab create-admin --username admin --password admin --firstname Admin --lastname User --email admin@example.com &&
        superset init &&
        superset run -h 0.0.0.0 -p 8088 --with-threads
      "

  superset-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: superset
      POSTGRES_USER: superset
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - superset-db-data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}

volumes:
  superset-db-data:
```

```python
# superset_config.py
import os
from datetime import timedelta

# Security
SECRET_KEY = os.getenv('SUPERSET_SECRET_KEY')
WTF_CSRF_ENABLED = True
SESSION_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True

# Database
SQLALCHEMY_DATABASE_URI = os.getenv('DATABASE_URL')

# Redis cache
CACHE_CONFIG = {
    'CACHE_TYPE': 'RedisCache',
    'CACHE_DEFAULT_TIMEOUT': 300,
    'CACHE_KEY_PREFIX': 'superset_',
    'CACHE_REDIS_URL': os.getenv('REDIS_URL'),
}

# Feature flags
FEATURE_FLAGS = {
    'ENABLE_TEMPLATE_PROCESSING': True,
    'DASHBOARD_NATIVE_FILTERS': True,
    'DASHBOARD_CROSS_FILTERS': True,
    'DRILL_TO_DETAIL': True,
    'EMBEDDABLE_CHARTS': True,
}

# Row limit
ROW_LIMIT = 100000

# Async queries
GLOBAL_ASYNC_QUERIES_TRANSPORT = 'polling'

# Allow embedding
TALISMAN_ENABLED = False  # Disable for embedding
```

### 3.2 Custom Charts สำหรับ LINE OA

```python
# superset_plugins/line_funnel_chart.py
# Custom chart สำหรับ LINE OA funnel

from superset.charts.post_processing import (
    apply_post_processing,
)

LINE_OA_CHARTS = {
    'line_funnel': {
        'name': 'LINE OA Funnel',
        'description': 'Funnel chart สำหรับ LINE OA user journey',
        'thumbnail': 'line_funnel_thumb.png',
    },
    'line_cohort': {
        'name': 'LINE Cohort Retention',
        'description': 'Cohort retention analysis',
        'thumbnail': 'line_cohort_thumb.png',
    },
    'line_user_flow': {
        'name': 'LINE User Flow',
        'description': 'Sankey diagram สำหรับ user flow',
        'thumbnail': 'line_flow_thumb.png',
    },
}
```

### 3.3 SQL Queries สำหรับ Superset Dashboards

```sql
-- Dashboard 1: Overview Metrics

-- Follower Growth Trend
SELECT 
    DATE_TRUNC('week', followed_at) as week,
    COUNT(*) as new_followers,
    SUM(COUNT(*)) OVER (ORDER BY DATE_TRUNC('week', followed_at)) as cumulative_followers
FROM line_users
WHERE followed_at >= NOW() - INTERVAL '6 months'
GROUP BY DATE_TRUNC('week', followed_at)
ORDER BY week;

-- Message Activity Heatmap
SELECT 
    EXTRACT(DOW FROM timestamp) as day_of_week,
    EXTRACT(HOUR FROM timestamp) as hour,
    COUNT(*) as message_count
FROM line_events
WHERE event_type = 'message'
  AND timestamp >= NOW() - INTERVAL '30 days'
GROUP BY day_of_week, hour
ORDER BY day_of_week, hour;

-- Top Users by Value
SELECT 
    u.user_id,
    u.display_name,
    u.followed_at,
    COUNT(DISTINCT e.date) as active_days,
    COALESCE(SUM(o.total), 0) as lifetime_value,
    COALESCE(COUNT(DISTINCT o.order_id), 0) as order_count,
    u.custom_properties->>'segment' as segment
FROM line_users u
LEFT JOIN line_events e ON u.user_id = e.user_id
LEFT JOIN orders o ON u.user_id = o.user_id AND o.status = 'completed'
WHERE u.unfollowed_at IS NULL
GROUP BY u.user_id, u.display_name, u.followed_at, u.custom_properties
ORDER BY lifetime_value DESC
LIMIT 100;

-- Campaign Performance
SELECT 
    c.campaign_id,
    c.name as campaign_name,
    c.type as campaign_type,
    c.sent_count,
    c.opened_count,
    c.clicked_count,
    c.converted_count,
    c.revenue_generated,
    CASE WHEN c.sent_count > 0 THEN ROUND(c.opened_count::NUMERIC/c.sent_count * 100, 2) ELSE 0 END as open_rate,
    CASE WHEN c.opened_count > 0 THEN ROUND(c.clicked_count::NUMERIC/c.opened_count * 100, 2) ELSE 0 END as ctr,
    CASE WHEN c.clicked_count > 0 THEN ROUND(c.converted_count::NUMERIC/c.clicked_count * 100, 2) ELSE 0 END as conversion_rate,
    CASE WHEN c.converted_count > 0 THEN ROUND(c.revenue_generated/c.converted_count, 2) ELSE 0 END as revenue_per_conversion
FROM campaigns c
WHERE c.start_at >= NOW() - INTERVAL '3 months'
ORDER BY c.revenue_generated DESC;

-- Response Time Analysis
SELECT 
    DATE_TRUNC('hour', timestamp) as hour,
    AVG(EXTRACT(EPOCH FROM (reply_time - message_time))/60) as avg_response_minutes,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (reply_time - message_time))/60) as median_response_minutes,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (reply_time - message_time))/60) as p95_response_minutes,
    COUNT(*) as total_conversations
FROM conversation_metrics
WHERE timestamp >= NOW() - INTERVAL '7 days'
GROUP BY DATE_TRUNC('hour', timestamp)
ORDER BY hour;
```

---

## 4. Google Looker Studio Integration

### 4.1 Data Connector Setup

```javascript
// src/analytics/lookerConnector.js
// Custom connector สำหรับ Looker Studio

const { BigQuery } = require('@google-cloud/bigquery');
const config = require('../config');

class LookerStudioConnector {
  constructor() {
    this.bigquery = new BigQuery({
      projectId: config.gcp.projectId,
      keyFilename: config.gcp.keyFile,
    });
    this.datasetId = 'line_oa_analytics';
  }

  // Sync data ไปยัง BigQuery
  async syncToBigQuery(data, tableName) {
    const dataset = this.bigquery.dataset(this.datasetId);
    const table = dataset.table(tableName);

    await table.insert(data, {
      skipInvalidRows: true,
      ignoreUnknownValues: true,
    });
  }

  // สร้าง scheduled sync
  async scheduledSync() {
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);
    const dateStr = yesterday.toISOString().split('T')[0];

    // Sync daily metrics
    const metricsQuery = `
      SELECT 
        '${dateStr}'::DATE as date,
        COUNT(DISTINCT user_id) as active_users,
        COUNT(CASE WHEN event_type = 'follow' THEN 1 END) as new_followers,
        COUNT(CASE WHEN event_type = 'unfollow' THEN 1 END) as unfollowers,
        COUNT(CASE WHEN event_type = 'message' THEN 1 END) as messages_received
      FROM line_events
      WHERE date = '${dateStr}'
    `;

    const { rows } = await db.query(metricsQuery);

    await this.syncToBigQuery(rows, 'daily_metrics');

    // Sync campaigns
    const campaignsQuery = `
      SELECT * FROM campaigns
      WHERE DATE(updated_at) = '${dateStr}'
    `;
    const { rows: campaigns } = await db.query(campaignsQuery);
    await this.syncToBigQuery(campaigns, 'campaigns');

    // Sync orders
    const ordersQuery = `
      SELECT * FROM orders
      WHERE DATE(created_at) = '${dateStr}'
    `;
    const { rows: orders } = await db.query(ordersQuery);
    await this.syncToBigQuery(orders, 'orders');

    console.log(`Synced data for ${dateStr} to BigQuery`);
  }

  // BigQuery schema definitions
  getTableSchemas() {
    return {
      daily_metrics: [
        { name: 'date', type: 'DATE' },
        { name: 'active_users', type: 'INTEGER' },
        { name: 'new_followers', type: 'INTEGER' },
        { name: 'unfollowers', type: 'INTEGER' },
        { name: 'messages_received', type: 'INTEGER' },
      ],
      campaigns: [
        { name: 'campaign_id', type: 'STRING' },
        { name: 'name', type: 'STRING' },
        { name: 'type', type: 'STRING' },
        { name: 'sent_count', type: 'INTEGER' },
        { name: 'opened_count', type: 'INTEGER' },
        { name: 'clicked_count', type: 'INTEGER' },
        { name: 'converted_count', type: 'INTEGER' },
        { name: 'revenue_generated', type: 'FLOAT' },
        { name: 'start_at', type: 'TIMESTAMP' },
      ],
      orders: [
        { name: 'order_id', type: 'STRING' },
        { name: 'user_id', type: 'STRING' },
        { name: 'campaign_id', type: 'STRING' },
        { name: 'status', type: 'STRING' },
        { name: 'total', type: 'FLOAT' },
        { name: 'source', type: 'STRING' },
        { name: 'created_at', type: 'TIMESTAMP' },
      ],
    };
  }
}

module.exports = LookerStudioConnector;
```

### 4.2 Looker Studio Templates

```javascript
// templates/lookerStudioConfig.js
// Configuration สำหรับ Looker Studio Reports

const LOOKER_REPORTS = {
  executiveDashboard: {
    name: 'LINE OA Executive Dashboard',
    pages: [
      {
        id: 'overview',
        name: 'ภาพรวม',
        charts: [
          {
            type: 'scorecard',
            metric: 'total_followers',
            comparison: 'previous_period',
            title: 'ผู้ติดตามทั้งหมด',
          },
          {
            type: 'scorecard',
            metric: 'monthly_revenue',
            format: 'currency_thb',
            title: 'รายได้เดือนนี้',
          },
          {
            type: 'time_series',
            dimensions: ['date'],
            metrics: ['active_users', 'new_followers'],
            title: 'Follower Growth',
          },
          {
            type: 'bar_chart',
            dimensions: ['campaign_name'],
            metrics: ['revenue_generated'],
            sort: 'desc',
            limit: 10,
            title: 'Top Campaigns by Revenue',
          },
        ],
      },
      {
        id: 'user-analysis',
        name: 'วิเคราะห์ผู้ใช้',
        charts: [
          {
            type: 'pie_chart',
            dimension: 'segment',
            metric: 'user_count',
            title: 'User Segments',
          },
          {
            type: 'table',
            dimensions: ['user_id', 'display_name', 'segment'],
            metrics: ['lifetime_value', 'order_count', 'active_days'],
            sort: 'lifetime_value:desc',
            title: 'Top Users',
          },
        ],
      },
    ],
  },
};

module.exports = LOOKER_REPORTS;
```

---

## 5. Custom Metrics Design

### 5.1 LINE OA Specific Metrics

```javascript
// src/analytics/lineMetrics.js
class LINEMetricsEngine {
  constructor(db, redis) {
    this.db = db;
    this.redis = redis;
  }

  // Message Engagement Score (MES)
  async calculateMES(userId, days = 30) {
    const { rows } = await this.db.query(`
      WITH user_activity AS (
        SELECT 
          COUNT(*) as message_count,
          COUNT(DISTINCT DATE(timestamp)) as active_days,
          MAX(timestamp) as last_activity
        FROM line_events
        WHERE user_id = $1 
          AND event_type = 'message'
          AND timestamp >= NOW() - INTERVAL '${days} days'
      ),
      response_data AS (
        SELECT 
          COUNT(*) as bot_messages,
          SUM(CASE WHEN is_opened THEN 1 ELSE 0 END) as opened_messages,
          SUM(CASE WHEN is_clicked THEN 1 ELSE 0 END) as clicked_messages
        FROM bot_messages
        WHERE user_id = $1
          AND sent_at >= NOW() - INTERVAL '${days} days'
      )
      SELECT 
        COALESCE(ua.message_count, 0) as sent_messages,
        COALESCE(ua.active_days, 0) as active_days,
        COALESCE(rd.bot_messages, 0) as received_messages,
        COALESCE(rd.opened_messages, 0) as opened_messages,
        COALESCE(rd.clicked_messages, 0) as clicked_messages,
        EXTRACT(DAYS FROM NOW() - COALESCE(ua.last_activity, NOW() - INTERVAL '999 days')) as days_since_active
      FROM user_activity ua, response_data rd
    `, [userId]);

    if (rows.length === 0) return 0;

    const data = rows[0];
    
    // Recency score (0-30)
    const recencyScore = Math.max(0, 30 - data.days_since_active);
    
    // Frequency score (0-30)
    const frequencyScore = Math.min(30, data.active_days * (30 / days));
    
    // Engagement score (0-40)
    const openRate = data.received_messages > 0 ? 
      data.opened_messages / data.received_messages : 0;
    const clickRate = data.opened_messages > 0 ? 
      data.clicked_messages / data.opened_messages : 0;
    const engagementScore = (openRate * 20) + (clickRate * 20);
    
    const mes = recencyScore + frequencyScore + engagementScore;
    
    return Math.round(mes);
  }

  // Customer Lifetime Value (CLV)
  async calculateCLV(userId) {
    const { rows } = await this.db.query(`
      SELECT 
        COALESCE(SUM(total), 0) as total_revenue,
        COUNT(DISTINCT order_id) as order_count,
        MIN(created_at) as first_order,
        MAX(created_at) as last_order,
        EXTRACT(DAYS FROM MAX(created_at) - MIN(created_at)) as customer_age_days
      FROM orders
      WHERE user_id = $1 AND status = 'completed'
    `, [userId]);

    if (rows.length === 0) return { clv: 0 };

    const data = rows[0];
    
    if (data.order_count === 0) return { clv: 0, totalRevenue: 0 };

    const avgOrderValue = data.total_revenue / data.order_count;
    const purchaseFrequency = data.customer_age_days > 0 
      ? data.order_count / (data.customer_age_days / 365) 
      : data.order_count;
    
    // Assumed customer lifespan: 3 years, margin: 30%
    const customerLifespan = 3;
    const margin = 0.3;
    
    const clv = avgOrderValue * purchaseFrequency * customerLifespan * margin;
    
    return {
      clv: Math.round(clv),
      totalRevenue: data.total_revenue,
      orderCount: data.order_count,
      avgOrderValue: Math.round(avgOrderValue),
      purchaseFrequency: Math.round(purchaseFrequency * 10) / 10,
    };
  }

  // Net Promoter Score (NPS) ผ่าน LINE
  async calculateNPS(startDate, endDate) {
    const { rows } = await this.db.query(`
      SELECT 
        score,
        COUNT(*) as count
      FROM nps_responses
      WHERE submitted_at BETWEEN $1 AND $2
      GROUP BY score
    `, [startDate, endDate]);

    const total = rows.reduce((sum, r) => sum + parseInt(r.count), 0);
    if (total === 0) return { nps: 0, total: 0 };

    const promoters = rows
      .filter(r => r.score >= 9)
      .reduce((sum, r) => sum + parseInt(r.count), 0);
    
    const detractors = rows
      .filter(r => r.score <= 6)
      .reduce((sum, r) => sum + parseInt(r.count), 0);

    const nps = Math.round(((promoters - detractors) / total) * 100);

    return {
      nps,
      total,
      promoters,
      detractors,
      passives: total - promoters - detractors,
      promoterRate: Math.round((promoters / total) * 100),
      detractorRate: Math.round((detractors / total) * 100),
    };
  }

  // Feature Usage Analytics
  async getFeatureUsage(days = 30) {
    const { rows } = await this.db.query(`
      SELECT 
        metadata->>'feature' as feature,
        COUNT(*) as usage_count,
        COUNT(DISTINCT user_id) as unique_users
      FROM line_events
      WHERE event_type = 'feature_used'
        AND timestamp >= NOW() - INTERVAL '${days} days'
      GROUP BY metadata->>'feature'
      ORDER BY usage_count DESC
    `);

    return rows;
  }
}

module.exports = LINEMetricsEngine;
```

---

## 6. User Journey Analytics

### 6.1 Journey Mapping System

```javascript
// src/analytics/journeyMapper.js
class UserJourneyMapper {
  constructor(db) {
    this.db = db;
  }

  // บันทึก journey touchpoint
  async recordTouchpoint(userId, touchpoint) {
    await this.db.query(`
      INSERT INTO user_journey (
        user_id, touchpoint, channel, action, metadata, timestamp
      ) VALUES ($1, $2, $3, $4, $5, NOW())
    `, [
      userId,
      touchpoint.name,
      touchpoint.channel || 'line',
      touchpoint.action,
      JSON.stringify(touchpoint.metadata || {}),
    ]);
  }

  // ดึง user journey
  async getUserJourney(userId, days = 30) {
    const { rows } = await this.db.query(`
      SELECT 
        touchpoint,
        channel,
        action,
        metadata,
        timestamp,
        LAG(touchpoint) OVER (PARTITION BY user_id ORDER BY timestamp) as prev_touchpoint,
        LEAD(touchpoint) OVER (PARTITION BY user_id ORDER BY timestamp) as next_touchpoint
      FROM user_journey
      WHERE user_id = $1
        AND timestamp >= NOW() - INTERVAL '${days} days'
      ORDER BY timestamp
    `, [userId]);

    return rows;
  }

  // วิเคราะห์ conversion paths
  async analyzeConversionPaths(days = 30) {
    const { rows } = await this.db.query(`
      WITH conversion_paths AS (
        SELECT 
          user_id,
          ARRAY_AGG(touchpoint ORDER BY timestamp) as path,
          MAX(CASE WHEN action = 'purchase' THEN 1 ELSE 0 END) as converted
        FROM user_journey
        WHERE timestamp >= NOW() - INTERVAL '${days} days'
        GROUP BY user_id
      )
      SELECT 
        ARRAY_TO_STRING(path, ' -> ') as journey_path,
        COUNT(*) as user_count,
        SUM(converted) as conversions,
        ROUND(SUM(converted)::NUMERIC / COUNT(*) * 100, 2) as conversion_rate
      FROM conversion_paths
      GROUP BY path
      HAVING COUNT(*) >= 5
      ORDER BY conversions DESC
      LIMIT 20
    `);

    return rows;
  }

  // Funnel Analysis
  async analyzeFunnel(funnelSteps, startDate, endDate) {
    let query = `
      WITH step_data AS (
        SELECT user_id, MIN(timestamp) as first_event
        FROM user_journey
        WHERE touchpoint = '${funnelSteps[0]}'
          AND timestamp BETWEEN '${startDate}' AND '${endDate}'
        GROUP BY user_id
      )
    `;

    funnelSteps.slice(1).forEach((step, index) => {
      query += `,
        step_${index + 1} AS (
          SELECT DISTINCT uj.user_id
          FROM step_data sd
          JOIN user_journey uj ON sd.user_id = uj.user_id
          WHERE uj.touchpoint = '${step}'
            AND uj.timestamp > sd.first_event
        )
      `;
    });

    query += `
      SELECT 
        '${funnelSteps[0]}' as step,
        COUNT(*) as users
      FROM step_data
    `;

    funnelSteps.slice(1).forEach((step, index) => {
      query += `
        UNION ALL
        SELECT '${step}' as step, COUNT(*) as users FROM step_${index + 1}
      `;
    });

    const { rows } = await this.db.query(query);

    // คำนวณ drop-off rates
    const result = rows.map((row, index) => {
      const previousCount = index > 0 ? rows[index - 1].users : row.users;
      return {
        step: row.step,
        users: row.users,
        dropOff: index > 0 ? previousCount - row.users : 0,
        dropOffRate: index > 0 ? 
          Math.round((1 - row.users / previousCount) * 100) : 0,
        conversionRate: Math.round(row.users / rows[0].users * 100),
      };
    });

    return result;
  }

  // Attribution Analysis
  async attributeConversions(startDate, endDate, model = 'last_touch') {
    let attributionQuery;
    
    switch (model) {
      case 'first_touch':
        attributionQuery = `
          SELECT 
            FIRST_VALUE(channel) OVER (PARTITION BY user_id ORDER BY timestamp) as channel,
            1.0 as attribution_weight
          FROM user_journey
          WHERE action = 'purchase'
        `;
        break;
      case 'linear':
        attributionQuery = `
          SELECT 
            channel,
            1.0 / COUNT(*) OVER (PARTITION BY user_id) as attribution_weight
          FROM user_journey
          WHERE action = 'purchase'
        `;
        break;
      case 'last_touch':
      default:
        attributionQuery = `
          SELECT 
            LAST_VALUE(channel) OVER (
              PARTITION BY user_id ORDER BY timestamp
              ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
            ) as channel,
            1.0 as attribution_weight
          FROM user_journey
          WHERE action = 'purchase'
        `;
    }

    const { rows } = await this.db.query(`
      WITH attributed AS (${attributionQuery})
      SELECT 
        channel,
        SUM(attribution_weight) as attributed_conversions,
        ROUND(SUM(attribution_weight) / SUM(SUM(attribution_weight)) OVER () * 100, 2) as percentage
      FROM attributed
      GROUP BY channel
      ORDER BY attributed_conversions DESC
    `);

    return rows;
  }
}

module.exports = UserJourneyMapper;
```

---

## 7. Revenue Attribution

### 7.1 Multi-touch Attribution Model

```javascript
// src/analytics/revenueAttribution.js
class RevenueAttributionEngine {
  constructor(db) {
    this.db = db;
  }

  // Position-based attribution (40% first, 40% last, 20% middle)
  async positionBasedAttribution(orderId) {
    const { rows } = await this.db.query(`
      SELECT 
        uj.channel,
        uj.touchpoint,
        uj.timestamp,
        o.total as order_value,
        ROW_NUMBER() OVER (PARTITION BY uj.user_id ORDER BY uj.timestamp) as position,
        COUNT(*) OVER (PARTITION BY uj.user_id) as total_touchpoints
      FROM orders o
      JOIN user_journey uj ON o.user_id = uj.user_id
      WHERE o.order_id = $1
        AND uj.timestamp <= o.created_at
      ORDER BY uj.timestamp
    `, [orderId]);

    if (rows.length === 0) return [];

    const totalTouchpoints = rows[0].total_touchpoints;
    const orderValue = rows[0].order_value;

    return rows.map(row => {
      let weight;
      if (totalTouchpoints === 1) {
        weight = 1.0;
      } else if (row.position === 1) {
        weight = 0.4;
      } else if (row.position === totalTouchpoints) {
        weight = 0.4;
      } else {
        weight = 0.2 / (totalTouchpoints - 2);
      }

      return {
        channel: row.channel,
        touchpoint: row.touchpoint,
        timestamp: row.timestamp,
        attributedRevenue: Math.round(orderValue * weight * 100) / 100,
        weight: Math.round(weight * 100) / 100,
      };
    });
  }

  // Campaign ROI Analysis
  async analyzeCampaignROI(campaignId) {
    const { rows } = await this.db.query(`
      SELECT 
        c.campaign_id,
        c.name,
        c.type,
        c.sent_count,
        c.revenue_generated,
        COALESCE(SUM(ce.cost), 0) as total_cost,
        CASE 
          WHEN COALESCE(SUM(ce.cost), 0) > 0 
          THEN ROUND((c.revenue_generated - COALESCE(SUM(ce.cost), 0)) / COALESCE(SUM(ce.cost), 0) * 100, 2)
          ELSE NULL
        END as roi_percentage
      FROM campaigns c
      LEFT JOIN campaign_expenses ce ON c.campaign_id = ce.campaign_id
      WHERE c.campaign_id = $1
      GROUP BY c.campaign_id, c.name, c.type, c.sent_count, c.revenue_generated
    `, [campaignId]);

    return rows[0];
  }

  // Channel Attribution Report
  async getChannelAttributionReport(startDate, endDate, model = 'position_based') {
    const { rows } = await this.db.query(`
      WITH order_touchpoints AS (
        SELECT 
          o.order_id,
          o.total,
          uj.channel,
          uj.timestamp as touch_time,
          o.created_at as conversion_time,
          ROW_NUMBER() OVER (PARTITION BY o.order_id ORDER BY uj.timestamp) as touch_position,
          COUNT(*) OVER (PARTITION BY o.order_id) as total_touches
        FROM orders o
        JOIN user_journey uj ON o.user_id = uj.user_id
        WHERE o.created_at BETWEEN '${startDate}' AND '${endDate}'
          AND o.status = 'completed'
          AND uj.timestamp <= o.created_at
          AND uj.timestamp >= o.created_at - INTERVAL '30 days'
      ),
      attributed AS (
        SELECT 
          channel,
          order_id,
          total,
          CASE 
            WHEN total_touches = 1 THEN total
            WHEN touch_position = 1 THEN total * 0.4
            WHEN touch_position = total_touches THEN total * 0.4
            ELSE total * 0.2 / GREATEST(total_touches - 2, 1)
          END as attributed_revenue
        FROM order_touchpoints
      )
      SELECT 
        channel,
        COUNT(DISTINCT order_id) as attributed_orders,
        SUM(attributed_revenue) as attributed_revenue,
        ROUND(SUM(attributed_revenue) / SUM(SUM(attributed_revenue)) OVER () * 100, 2) as revenue_share
      FROM attributed
      GROUP BY channel
      ORDER BY attributed_revenue DESC
    `);

    return rows;
  }
}

module.exports = RevenueAttributionEngine;
```

---

## 8. Predictive Analytics

### 8.1 Churn Prediction Model

```javascript
// src/analytics/churnPrediction.js
const axios = require('axios');

class ChurnPredictionService {
  constructor(db, redis) {
    this.db = db;
    this.redis = redis;
    this.modelEndpoint = process.env.ML_MODEL_ENDPOINT;
  }

  // ดึง features สำหรับ prediction
  async extractUserFeatures(userId) {
    const { rows } = await this.db.query(`
      SELECT 
        -- Recency
        EXTRACT(DAYS FROM NOW() - MAX(e.timestamp)) as days_since_last_active,
        
        -- Frequency (last 30 days)
        COUNT(CASE WHEN e.timestamp >= NOW() - INTERVAL '30 days' THEN 1 END) as messages_30d,
        COUNT(CASE WHEN e.timestamp >= NOW() - INTERVAL '7 days' THEN 1 END) as messages_7d,
        COUNT(DISTINCT DATE(e.timestamp)) FILTER (WHERE e.timestamp >= NOW() - INTERVAL '30 days') as active_days_30d,
        
        -- Monetary
        COALESCE(SUM(o.total) FILTER (WHERE o.created_at >= NOW() - INTERVAL '90 days'), 0) as spending_90d,
        COUNT(DISTINCT o.order_id) FILTER (WHERE o.created_at >= NOW() - INTERVAL '90 days') as orders_90d,
        
        -- Engagement
        COUNT(bm.message_id) FILTER (WHERE bm.sent_at >= NOW() - INTERVAL '30 days') as messages_received_30d,
        COUNT(bm.message_id) FILTER (WHERE bm.is_opened AND bm.sent_at >= NOW() - INTERVAL '30 days') as messages_opened_30d,
        COUNT(bm.message_id) FILTER (WHERE bm.is_clicked AND bm.sent_at >= NOW() - INTERVAL '30 days') as messages_clicked_30d,
        
        -- Account age
        EXTRACT(DAYS FROM NOW() - u.followed_at) as account_age_days,
        
        -- Current segment
        u.custom_properties->>'segment' as current_segment
        
      FROM line_users u
      LEFT JOIN line_events e ON u.user_id = e.user_id
      LEFT JOIN orders o ON u.user_id = o.user_id
      LEFT JOIN bot_messages bm ON u.user_id = bm.user_id
      WHERE u.user_id = $1
      GROUP BY u.user_id, u.followed_at, u.custom_properties
    `, [userId]);

    if (rows.length === 0) return null;

    const features = rows[0];
    
    // Normalize features
    return {
      days_since_last_active: Math.min(features.days_since_last_active || 0, 365),
      messages_30d: Math.min(features.messages_30d || 0, 500),
      messages_7d: Math.min(features.messages_7d || 0, 100),
      active_days_30d: features.active_days_30d || 0,
      spending_90d: Math.min(features.spending_90d || 0, 100000),
      orders_90d: Math.min(features.orders_90d || 0, 50),
      open_rate_30d: features.messages_received_30d > 0 
        ? features.messages_opened_30d / features.messages_received_30d 
        : 0,
      click_rate_30d: features.messages_opened_30d > 0 
        ? features.messages_clicked_30d / features.messages_opened_30d 
        : 0,
      account_age_days: features.account_age_days || 0,
    };
  }

  // ทำนาย churn probability
  async predictChurn(userId) {
    const features = await this.extractUserFeatures(userId);
    if (!features) return null;

    // Simple rule-based model (ก่อนใช้ ML model จริง)
    let churnScore = 0;

    if (features.days_since_last_active > 30) churnScore += 30;
    else if (features.days_since_last_active > 14) churnScore += 15;
    else if (features.days_since_last_active > 7) churnScore += 5;

    if (features.messages_30d === 0) churnScore += 25;
    else if (features.messages_30d < 3) churnScore += 10;

    if (features.open_rate_30d < 0.1) churnScore += 20;
    else if (features.open_rate_30d < 0.3) churnScore += 10;

    if (features.orders_90d === 0 && features.account_age_days > 30) churnScore += 15;

    if (features.spending_90d === 0 && features.account_age_days > 60) churnScore += 10;

    const churnProbability = Math.min(100, churnScore) / 100;
    
    let riskLevel;
    if (churnProbability >= 0.7) riskLevel = 'high';
    else if (churnProbability >= 0.4) riskLevel = 'medium';
    else riskLevel = 'low';

    return {
      userId,
      churnProbability,
      riskLevel,
      features,
      recommendations: this.getChurnPreventionActions(riskLevel, features),
    };
  }

  getChurnPreventionActions(riskLevel, features) {
    const actions = [];

    if (riskLevel === 'high') {
      actions.push({
        type: 'win_back_campaign',
        message: 'ส่ง win-back campaign ด้วย offer พิเศษ',
        priority: 'high',
      });
    }

    if (features.days_since_last_active > 14) {
      actions.push({
        type: 're_engagement_message',
        message: 'ส่งข้อความ re-engagement',
        priority: 'medium',
      });
    }

    if (features.open_rate_30d < 0.2) {
      actions.push({
        type: 'content_optimization',
        message: 'ปรับ content ให้ relevant มากขึ้น',
        priority: 'medium',
      });
    }

    if (features.orders_90d === 0) {
      actions.push({
        type: 'first_purchase_offer',
        message: 'ส่ง discount code สำหรับการซื้อครั้งแรก',
        priority: 'high',
      });
    }

    return actions;
  }

  // Batch prediction สำหรับทุก user
  async batchPredict(limit = 1000) {
    const { rows } = await this.db.query(`
      SELECT user_id FROM line_users
      WHERE unfollowed_at IS NULL
        AND followed_at < NOW() - INTERVAL '7 days'
      ORDER BY updated_at ASC
      LIMIT $1
    `, [limit]);

    const predictions = [];
    const highRiskUsers = [];

    for (const { user_id } of rows) {
      const prediction = await this.predictChurn(user_id);
      if (prediction) {
        predictions.push(prediction);
        
        if (prediction.riskLevel === 'high') {
          highRiskUsers.push(prediction);
        }

        // บันทึกผลลัพธ์
        await this.db.query(`
          UPDATE line_users 
          SET custom_properties = custom_properties || $2
          WHERE user_id = $1
        `, [user_id, JSON.stringify({
          churn_probability: prediction.churnProbability,
          churn_risk: prediction.riskLevel,
          churn_predicted_at: new Date().toISOString(),
        })]);
      }
    }

    return {
      total: predictions.length,
      highRisk: highRiskUsers.length,
      mediumRisk: predictions.filter(p => p.riskLevel === 'medium').length,
      lowRisk: predictions.filter(p => p.riskLevel === 'low').length,
      highRiskUsers,
    };
  }
}

module.exports = ChurnPredictionService;
```

### 8.2 Revenue Forecasting

```javascript
// src/analytics/revenueForecasting.js
class RevenueForecastingService {
  constructor(db) {
    this.db = db;
  }

  // Simple time series forecasting ด้วย linear regression
  async forecastRevenue(days = 30) {
    const { rows } = await this.db.query(`
      SELECT 
        DATE(created_at) as date,
        SUM(total) as daily_revenue,
        COUNT(*) as order_count
      FROM orders
      WHERE status = 'completed'
        AND created_at >= NOW() - INTERVAL '90 days'
      GROUP BY DATE(created_at)
      ORDER BY date
    `);

    if (rows.length < 14) {
      return { error: 'Not enough data for forecasting' };
    }

    // Linear regression
    const n = rows.length;
    const xValues = rows.map((_, i) => i);
    const yValues = rows.map(r => parseFloat(r.daily_revenue));

    const xMean = xValues.reduce((a, b) => a + b) / n;
    const yMean = yValues.reduce((a, b) => a + b) / n;

    const numerator = xValues.reduce((sum, x, i) => sum + (x - xMean) * (yValues[i] - yMean), 0);
    const denominator = xValues.reduce((sum, x) => sum + Math.pow(x - xMean, 2), 0);

    const slope = numerator / denominator;
    const intercept = yMean - slope * xMean;

    // Generate forecasts
    const forecasts = [];
    for (let i = 1; i <= days; i++) {
      const x = n + i - 1;
      const forecastDate = new Date();
      forecastDate.setDate(forecastDate.getDate() + i);
      
      const predicted = slope * x + intercept;
      
      // Add seasonality adjustment (7-day pattern)
      const dayOfWeek = forecastDate.getDay();
      const weekdayMultipliers = [0.85, 1.0, 1.05, 1.1, 1.1, 1.2, 0.9];
      const adjusted = predicted * weekdayMultipliers[dayOfWeek];
      
      forecasts.push({
        date: forecastDate.toISOString().split('T')[0],
        forecastRevenue: Math.max(0, Math.round(adjusted)),
        confidence: {
          lower: Math.max(0, Math.round(adjusted * 0.8)),
          upper: Math.round(adjusted * 1.2),
        },
      });
    }

    const totalForecast = forecasts.reduce((sum, f) => sum + f.forecastRevenue, 0);
    const avgDaily = Math.round(totalForecast / days);

    return {
      forecasts,
      summary: {
        forecastPeriodDays: days,
        totalForecastRevenue: totalForecast,
        avgDailyRevenue: avgDaily,
        trend: slope > 0 ? 'increasing' : slope < 0 ? 'decreasing' : 'stable',
        trendRate: Math.round(Math.abs(slope / yMean) * 100 * 100) / 100,
      },
      historicalAvg: Math.round(yMean),
    };
  }
}

module.exports = RevenueForecastingService;
```

---

## 9. Cohort Retention Analysis

### 9.1 Cohort Analysis Implementation

```javascript
// src/analytics/cohortAnalysis.js
class CohortAnalysisService {
  constructor(db) {
    this.db = db;
  }

  // สร้าง cohort retention matrix
  async buildRetentionMatrix(months = 6) {
    const { rows } = await this.db.query(`
      WITH cohort_dates AS (
        SELECT 
          user_id,
          DATE_TRUNC('month', followed_at) as cohort_month
        FROM line_users
        WHERE followed_at >= NOW() - INTERVAL '${months + 1} months'
      ),
      activity_months AS (
        SELECT 
          cd.user_id,
          cd.cohort_month,
          DATE_TRUNC('month', e.timestamp) as activity_month
        FROM cohort_dates cd
        JOIN line_events e ON cd.user_id = e.user_id
        WHERE e.timestamp >= NOW() - INTERVAL '${months + 1} months'
      ),
      cohort_retention AS (
        SELECT 
          cohort_month,
          activity_month,
          COUNT(DISTINCT user_id) as retained_users
        FROM activity_months
        GROUP BY cohort_month, activity_month
      ),
      cohort_sizes AS (
        SELECT 
          cohort_month,
          COUNT(DISTINCT user_id) as cohort_size
        FROM cohort_dates
        GROUP BY cohort_month
      )
      SELECT 
        cr.cohort_month,
        cs.cohort_size,
        cr.activity_month,
        cr.retained_users,
        ROUND(cr.retained_users::NUMERIC / cs.cohort_size * 100, 2) as retention_rate,
        EXTRACT(MONTH FROM cr.activity_month - cr.cohort_month) as months_since_join
      FROM cohort_retention cr
      JOIN cohort_sizes cs ON cr.cohort_month = cs.cohort_month
      ORDER BY cr.cohort_month, cr.activity_month
    `);

    // จัดรูปแบบเป็น matrix
    const cohorts = {};
    rows.forEach(row => {
      const cohortKey = row.cohort_month.toISOString().split('T')[0].substring(0, 7);
      if (!cohorts[cohortKey]) {
        cohorts[cohortKey] = {
          cohortMonth: cohortKey,
          cohortSize: row.cohort_size,
          retention: {},
        };
      }
      cohorts[cohortKey].retention[row.months_since_join] = {
        users: row.retained_users,
        rate: row.retention_rate,
      };
    });

    return Object.values(cohorts);
  }

  // คำนวณ Average Retention Curve
  async getAverageRetentionCurve(months = 6) {
    const matrix = await this.buildRetentionMatrix(months);
    
    const monthRetention = {};
    
    matrix.forEach(cohort => {
      Object.entries(cohort.retention).forEach(([month, data]) => {
        if (!monthRetention[month]) {
          monthRetention[month] = { rates: [], totalUsers: 0 };
        }
        monthRetention[month].rates.push(data.rate);
        monthRetention[month].totalUsers += data.users;
      });
    });

    const curve = Object.entries(monthRetention).map(([month, data]) => ({
      month: parseInt(month),
      avgRetentionRate: Math.round(
        data.rates.reduce((a, b) => a + b, 0) / data.rates.length * 100
      ) / 100,
      cohortCount: data.rates.length,
    })).sort((a, b) => a.month - b.month);

    return curve;
  }

  // Segment-based retention
  async getRetentionBySegment(segment, months = 3) {
    const { rows } = await this.db.query(`
      WITH segment_users AS (
        SELECT user_id
        FROM line_users
        WHERE custom_properties->>'segment' = $1
          AND followed_at >= NOW() - INTERVAL '${months} months'
      ),
      cohort_activity AS (
        SELECT 
          su.user_id,
          DATE_TRUNC('month', u.followed_at) as cohort_month,
          DATE_TRUNC('month', e.timestamp) as activity_month
        FROM segment_users su
        JOIN line_users u ON su.user_id = u.user_id
        JOIN line_events e ON su.user_id = e.user_id
        WHERE e.timestamp >= NOW() - INTERVAL '${months + 1} months'
      )
      SELECT 
        cohort_month,
        activity_month,
        COUNT(DISTINCT user_id) as active_users,
        EXTRACT(MONTH FROM activity_month - cohort_month) as months_since_join
      FROM cohort_activity
      GROUP BY cohort_month, activity_month
      ORDER BY cohort_month, activity_month
    `, [segment]);

    return rows;
  }

  // Day-level retention (สำหรับ mobile apps / LIFF)
  async getDayLevelRetention(days = 30) {
    const { rows } = await this.db.query(`
      WITH new_users AS (
        SELECT 
          user_id,
          DATE(followed_at) as join_date
        FROM line_users
        WHERE followed_at >= NOW() - INTERVAL '${days + 30} days'
      ),
      daily_activity AS (
        SELECT DISTINCT
          nu.user_id,
          nu.join_date,
          DATE(e.timestamp) as activity_date,
          DATE(e.timestamp) - nu.join_date as day_number
        FROM new_users nu
        JOIN line_events e ON nu.user_id = e.user_id
        WHERE e.timestamp >= nu.followed_at
      )
      SELECT 
        day_number,
        COUNT(DISTINCT user_id) as active_users,
        (SELECT COUNT(DISTINCT user_id) FROM new_users) as total_users,
        ROUND(COUNT(DISTINCT user_id)::NUMERIC / 
              (SELECT COUNT(DISTINCT user_id) FROM new_users) * 100, 2) as retention_rate
      FROM daily_activity
      WHERE day_number BETWEEN 0 AND $1
      GROUP BY day_number
      ORDER BY day_number
    `, [days]);

    return rows;
  }
}

module.exports = CohortAnalysisService;
```

---

## 10. Dashboard Templates สำหรับ LINE OA

### 10.1 Executive Summary Dashboard

```javascript
// src/analytics/dashboards/executiveDashboard.js
class ExecutiveDashboard {
  constructor(services) {
    this.metricsEngine = services.metricsEngine;
    this.cohortAnalysis = services.cohortAnalysis;
    this.forecasting = services.forecasting;
    this.attribution = services.attribution;
  }

  async generateExecutiveSummary(period = 'monthly') {
    const endDate = new Date();
    const startDate = new Date();
    
    if (period === 'monthly') {
      startDate.setMonth(startDate.getMonth() - 1);
    } else if (period === 'weekly') {
      startDate.setDate(startDate.getDate() - 7);
    } else if (period === 'quarterly') {
      startDate.setMonth(startDate.getMonth() - 3);
    }

    const [
      followersData,
      revenueData,
      engagementData,
      retentionData,
      forecastData,
      attributionData,
    ] = await Promise.all([
      this.getFollowerMetrics(startDate, endDate),
      this.getRevenueMetrics(startDate, endDate),
      this.getEngagementMetrics(startDate, endDate),
      this.cohortAnalysis.getAverageRetentionCurve(3),
      this.forecasting.forecastRevenue(30),
      this.attribution.getChannelAttributionReport(startDate, endDate),
    ]);

    return {
      period: {
        start: startDate.toISOString().split('T')[0],
        end: endDate.toISOString().split('T')[0],
        type: period,
      },
      kpis: {
        followers: followersData,
        revenue: revenueData,
        engagement: engagementData,
      },
      retention: {
        curve: retentionData,
        day1: retentionData.find(r => r.month === 0)?.avgRetentionRate || 0,
        day30: retentionData.find(r => r.month === 1)?.avgRetentionRate || 0,
        day90: retentionData.find(r => r.month === 3)?.avgRetentionRate || 0,
      },
      forecast: {
        next30Days: forecastData.summary?.totalForecastRevenue || 0,
        trend: forecastData.summary?.trend || 'stable',
      },
      attribution: attributionData,
      generatedAt: new Date().toISOString(),
    };
  }

  async getFollowerMetrics(startDate, endDate) {
    const { rows } = await this.db.query(`
      SELECT 
        COUNT(*) FILTER (WHERE followed_at < '${endDate.toISOString()}') as total_followers,
        COUNT(*) FILTER (WHERE followed_at BETWEEN '${startDate.toISOString()}' AND '${endDate.toISOString()}') as new_followers,
        COUNT(*) FILTER (WHERE unfollowed_at BETWEEN '${startDate.toISOString()}' AND '${endDate.toISOString()}') as unfollowers
      FROM line_users
    `);

    const data = rows[0];
    const netGrowth = data.new_followers - data.unfollowers;
    
    return {
      total: data.total_followers,
      new: data.new_followers,
      unfollowed: data.unfollowers,
      netGrowth,
      growthRate: data.total_followers > 0 
        ? Math.round(netGrowth / (data.total_followers - netGrowth) * 100 * 100) / 100
        : 0,
    };
  }

  async getRevenueMetrics(startDate, endDate) {
    const { rows } = await this.db.query(`
      SELECT 
        SUM(total) as total_revenue,
        COUNT(*) as total_orders,
        AVG(total) as avg_order_value,
        COUNT(DISTINCT user_id) as paying_customers
      FROM orders
      WHERE status = 'completed'
        AND created_at BETWEEN '${startDate.toISOString()}' AND '${endDate.toISOString()}'
    `);

    return rows[0];
  }

  async getEngagementMetrics(startDate, endDate) {
    const { rows } = await this.db.query(`
      SELECT 
        COUNT(*) as total_sent,
        SUM(CASE WHEN is_opened THEN 1 ELSE 0 END) as total_opened,
        SUM(CASE WHEN is_clicked THEN 1 ELSE 0 END) as total_clicked,
        ROUND(SUM(CASE WHEN is_opened THEN 1 ELSE 0 END)::NUMERIC / COUNT(*) * 100, 2) as open_rate,
        ROUND(SUM(CASE WHEN is_clicked THEN 1 ELSE 0 END)::NUMERIC / 
              NULLIF(SUM(CASE WHEN is_opened THEN 1 ELSE 0 END), 0) * 100, 2) as ctr
      FROM bot_messages
      WHERE sent_at BETWEEN '${startDate.toISOString()}' AND '${endDate.toISOString()}'
    `);

    return rows[0];
  }
}

module.exports = ExecutiveDashboard;
```

---

## 11. Executive Reporting Automation

### 11.1 Automated Report Generator

```javascript
// src/reporting/reportGenerator.js
const puppeteer = require('puppeteer');
const nodemailer = require('nodemailer');
const handlebars = require('handlebars');
const fs = require('fs');
const path = require('path');

class ExecutiveReportGenerator {
  constructor(dashboard, lineClient) {
    this.dashboard = dashboard;
    this.lineClient = lineClient;
    this.templatesDir = path.join(__dirname, '../templates/reports');
  }

  // สร้าง weekly report
  async generateWeeklyReport() {
    const summary = await this.dashboard.generateExecutiveSummary('weekly');
    
    // Render HTML report
    const html = await this.renderHTMLReport(summary, 'weekly');
    
    // Generate PDF
    const pdfBuffer = await this.htmlToPDF(html);
    
    return { summary, html, pdf: pdfBuffer };
  }

  // Render HTML template
  async renderHTMLReport(data, type) {
    const templatePath = path.join(this.templatesDir, `${type}-report.html`);
    const template = fs.readFileSync(templatePath, 'utf8');
    const compiled = handlebars.compile(template);
    
    return compiled({
      ...data,
      formatThaiDate: (date) => new Date(date).toLocaleDateString('th-TH', {
        year: 'numeric',
        month: 'long',
        day: 'numeric',
      }),
      formatCurrency: (amount) => `฿${parseFloat(amount || 0).toLocaleString('th-TH')}`,
      formatPercent: (rate) => `${parseFloat(rate || 0).toFixed(2)}%`,
    });
  }

  // Convert HTML to PDF
  async htmlToPDF(html) {
    const browser = await puppeteer.launch({
      headless: true,
      args: ['--no-sandbox', '--disable-setuid-sandbox'],
    });
    
    const page = await browser.newPage();
    await page.setContent(html, { waitUntil: 'networkidle0' });
    
    const pdf = await page.pdf({
      format: 'A4',
      margin: { top: '20mm', right: '20mm', bottom: '20mm', left: '20mm' },
      printBackground: true,
    });
    
    await browser.close();
    return pdf;
  }

  // ส่ง report ทาง email
  async sendEmailReport(recipients, subject, html, pdf) {
    const transporter = nodemailer.createTransport({
      host: process.env.SMTP_HOST,
      port: process.env.SMTP_PORT,
      secure: true,
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASSWORD,
      },
    });

    await transporter.sendMail({
      from: `"LINE OA Analytics" <${process.env.SMTP_USER}>`,
      to: recipients.join(', '),
      subject,
      html,
      attachments: [
        {
          filename: `line-oa-report-${new Date().toISOString().split('T')[0]}.pdf`,
          content: pdf,
          contentType: 'application/pdf',
        },
      ],
    });
  }

  // ส่ง summary ผ่าน LINE
  async sendLineSummary(userId, summary) {
    const kpis = summary.kpis;
    
    const message = {
      type: 'flex',
      altText: 'รายงานประจำสัปดาห์ LINE OA',
      contents: {
        type: 'bubble',
        header: {
          type: 'box',
          layout: 'vertical',
          backgroundColor: '#00B900',
          contents: [
            {
              type: 'text',
              text: '📊 รายงานประจำสัปดาห์',
              color: '#FFFFFF',
              weight: 'bold',
              size: 'lg',
            },
            {
              type: 'text',
              text: `${summary.period.start} - ${summary.period.end}`,
              color: '#CCFFCC',
              size: 'sm',
            },
          ],
        },
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'box',
              layout: 'horizontal',
              contents: [
                this.createMetricBox('ผู้ติดตาม', kpis.followers.total?.toLocaleString(), `+${kpis.followers.new}`),
                this.createMetricBox('รายได้', `฿${parseFloat(kpis.revenue.total_revenue || 0).toLocaleString()}`, `${kpis.revenue.total_orders} orders`),
              ],
            },
            {
              type: 'box',
              layout: 'horizontal',
              margin: 'md',
              contents: [
                this.createMetricBox('Open Rate', `${kpis.engagement.open_rate}%`, ''),
                this.createMetricBox('Retention', `${summary.retention.day30}%`, '30-day'),
              ],
            },
          ],
        },
      },
    };

    await this.lineClient.pushMessage(userId, message);
  }

  createMetricBox(label, value, subtext) {
    return {
      type: 'box',
      layout: 'vertical',
      flex: 1,
      borderWidth: '1px',
      borderColor: '#EEEEEE',
      cornerRadius: 'sm',
      paddingAll: 'sm',
      margin: 'xs',
      contents: [
        { type: 'text', text: label, size: 'xs', color: '#888888' },
        { type: 'text', text: value || '0', weight: 'bold', size: 'lg' },
        ...(subtext ? [{ type: 'text', text: subtext, size: 'xxs', color: '#00B900' }] : []),
      ],
    };
  }
}

module.exports = ExecutiveReportGenerator;
```

### 11.2 Scheduled Reports Setup

```javascript
// src/reporting/scheduler.js
const cron = require('node-cron');
const ExecutiveReportGenerator = require('./reportGenerator');

function setupReportScheduler(services) {
  const generator = new ExecutiveReportGenerator(services.dashboard, services.lineClient);

  // รายงานประจำวัน - ทุกวัน 8:00 น.
  cron.schedule('0 8 * * *', async () => {
    try {
      const { summary, html, pdf } = await generator.generateWeeklyReport();
      
      // ส่ง email
      await generator.sendEmailReport(
        process.env.DAILY_REPORT_RECIPIENTS?.split(',') || [],
        `LINE OA Daily Report - ${new Date().toLocaleDateString('th-TH')}`,
        html,
        pdf
      );

      console.log('Daily report sent successfully');
    } catch (error) {
      console.error('Error generating daily report:', error);
    }
  }, { timezone: 'Asia/Bangkok' });

  // รายงานประจำสัปดาห์ - ทุกวันจันทร์ 9:00 น.
  cron.schedule('0 9 * * 1', async () => {
    try {
      const { summary, html, pdf } = await generator.generateWeeklyReport();
      
      // ส่ง email ไปยัง executives
      await generator.sendEmailReport(
        process.env.WEEKLY_REPORT_RECIPIENTS?.split(',') || [],
        `LINE OA Weekly Report - สัปดาห์ที่ ${getWeekNumber(new Date())}`,
        html,
        pdf
      );

      // ส่ง LINE summary ไปยัง admin
      await generator.sendLineSummary(process.env.ADMIN_LINE_USER_ID, summary);

      console.log('Weekly report sent successfully');
    } catch (error) {
      console.error('Error generating weekly report:', error);
    }
  }, { timezone: 'Asia/Bangkok' });

  // รายงานประจำเดือน - วันที่ 1 ของทุกเดือน 9:00 น.
  cron.schedule('0 9 1 * *', async () => {
    try {
      const summary = await services.dashboard.generateExecutiveSummary('monthly');
      // Generate and send monthly report
      console.log('Monthly report generated');
    } catch (error) {
      console.error('Error generating monthly report:', error);
    }
  }, { timezone: 'Asia/Bangkok' });
}

function getWeekNumber(date) {
  const firstDayOfYear = new Date(date.getFullYear(), 0, 1);
  const pastDaysOfYear = (date - firstDayOfYear) / 86400000;
  return Math.ceil((pastDaysOfYear + firstDayOfYear.getDay() + 1) / 7);
}

module.exports = setupReportScheduler;
```

---

## 12. Complete BI Stack สำหรับ LINE OA

### 12.1 Docker Compose - Complete BI Stack

```yaml
# docker-compose.bi.yml
version: '3.8'

services:
  # Main app
  line-bot:
    build: .
    environment:
      - DATABASE_URL=postgresql://linebot:${DB_PASSWORD}@postgres:5432/linebot_db
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379/0
    depends_on:
      - postgres
      - redis

  # PostgreSQL - Primary Database
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: linebot_db
      POSTGRES_USER: linebot
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./sql/init.sql:/docker-entrypoint-initdb.d/init.sql

  # Redis
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data

  # Apache Superset
  superset:
    image: apache/superset:latest
    ports:
      - "8088:8088"
    environment:
      SUPERSET_SECRET_KEY: ${SUPERSET_SECRET_KEY}
      DATABASE_URL: postgresql://superset:${SUPERSET_DB_PASSWORD}@superset-db:5432/superset
    depends_on:
      - superset-db
      - redis

  superset-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: superset
      POSTGRES_USER: superset
      POSTGRES_PASSWORD: ${SUPERSET_DB_PASSWORD}
    volumes:
      - superset-db-data:/var/lib/postgresql/data

  # ClickHouse - Real-time Analytics
  clickhouse:
    image: clickhouse/clickhouse-server:latest
    ports:
      - "8123:8123"
      - "9000:9000"
    volumes:
      - clickhouse-data:/var/lib/clickhouse
      - ./clickhouse/config.xml:/etc/clickhouse-server/config.xml

  # Metabase - Alternative BI Tool
  metabase:
    image: metabase/metabase:latest
    ports:
      - "3001:3000"
    environment:
      MB_DB_TYPE: postgres
      MB_DB_HOST: metabase-db
      MB_DB_PORT: 5432
      MB_DB_DBNAME: metabase
      MB_DB_USER: metabase
      MB_DB_PASS: ${METABASE_DB_PASSWORD}
    depends_on:
      - metabase-db

  metabase-db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: metabase
      POSTGRES_USER: metabase
      POSTGRES_PASSWORD: ${METABASE_DB_PASSWORD}
    volumes:
      - metabase-db-data:/var/lib/postgresql/data

  # Grafana - Metrics Visualization
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3002:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./grafana/datasources:/etc/grafana/provisioning/datasources

  # Prometheus - Metrics Collection
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

volumes:
  postgres-data:
  redis-data:
  superset-db-data:
  clickhouse-data:
  metabase-db-data:
  grafana-data:
  prometheus-data:
```

### 12.2 Grafana Dashboard Configuration

```yaml
# grafana/dashboards/line-oa-overview.json
{
  "dashboard": {
    "title": "LINE OA Overview",
    "panels": [
      {
        "id": 1,
        "title": "Active Users (Real-time)",
        "type": "stat",
        "datasource": "Redis",
        "targets": [
          {
            "queryType": "command",
            "query": "SCARD metrics:today:active_users"
          }
        ]
      },
      {
        "id": 2,
        "title": "Messages Received Today",
        "type": "stat",
        "datasource": "Redis",
        "targets": [
          {
            "queryType": "command",
            "query": "GET metrics:today:events:message"
          }
        ]
      },
      {
        "id": 3,
        "title": "Follower Growth (30 days)",
        "type": "timeseries",
        "datasource": "PostgreSQL",
        "targets": [
          {
            "rawSql": "SELECT followed_at as time, COUNT(*) as value FROM line_users WHERE followed_at >= NOW() - INTERVAL '30 days' GROUP BY DATE(followed_at) ORDER BY 1"
          }
        ]
      }
    ]
  }
}
```

---

## สรุปบทที่ 97

ในบทนี้เราได้เรียนรู้ระบบ Analytics และ Business Intelligence สมบูรณ์สำหรับ LINE OA:

1. **BI Architecture** - Data pipeline จาก LINE ไปสู่ BI tools
2. **Data Model** - Schema ที่เหมาะสมสำหรับ LINE analytics
3. **Apache Superset** - BI dashboard สำหรับทีม
4. **Looker Studio** - Reports สำหรับ executives
5. **Custom Metrics** - MES, CLV, NPS สำหรับ LINE OA
6. **User Journey** - วิเคราะห์ path ของผู้ใช้
7. **Revenue Attribution** - Multi-touch attribution
8. **Predictive Analytics** - Churn prediction, revenue forecasting
9. **Cohort Analysis** - การวิเคราะห์ retention
10. **Automated Reporting** - Report อัตโนมัติผ่าน email และ LINE

### Checklist การ Implement

- [ ] ออกแบบ data schema
- [ ] ติดตั้ง event tracking
- [ ] Setup Apache Superset
- [ ] เชื่อมต่อ BigQuery สำหรับ Looker Studio
- [ ] Implement cohort analysis
- [ ] ตั้งค่า churn prediction
- [ ] สร้าง executive dashboard
- [ ] Setup automated reporting
- [ ] Deploy BI stack ด้วย Docker Compose
- [ ] ตั้งค่า scheduled reports
