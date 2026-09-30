# ตอนที่ 70: Analytics Dashboard สำหรับ LINE Bots

## บทนำ

Analytics Dashboard ช่วยให้คุณติดตามประสิทธิภาพของ LINE Bot ได้แบบ Real-time เรียนรู้พฤติกรรมผู้ใช้ วิเคราะห์ Conversion Rate และวัด ROI ได้อย่างแม่นยำ

## สถาปัตยกรรม Dashboard

```
┌──────────────────────────────────────────────────────────────────┐
│                   Analytics Dashboard System                      │
│                                                                    │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────────────┐  │
│  │  LINE OA    │───▶│  Event      │───▶│   Analytics DB       │  │
│  │  Webhook    │    │  Collector  │    │   (MySQL/ClickHouse)  │  │
│  └─────────────┘    └─────────────┘    └──────────────────────┘  │
│                                                 │                  │
│  ┌─────────────┐                               ▼                  │
│  │  Real-time  │◀──────── Socket.io ──── ┌──────────────────┐   │
│  │  Dashboard  │                          │  Express.js API  │   │
│  │  (Browser)  │◀─── REST API ───────────│  + Chart.js      │   │
│  └─────────────┘                          └──────────────────┘   │
│                                                                    │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  Dashboard Views                                          │    │
│  │  • Message Volume  • User Growth  • Conversion Funnel    │    │
│  │  • Revenue         • Bot Performance  • Automated Reports│    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

## 1. Database Schema สำหรับ Analytics

```sql
-- สร้าง Analytics Database
CREATE DATABASE line_analytics;
USE line_analytics;

-- ตาราง events หลัก
CREATE TABLE events (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    event_id VARCHAR(100) UNIQUE,
    event_type VARCHAR(50) NOT NULL,
    line_user_id VARCHAR(100),
    session_id VARCHAR(100),
    
    -- ข้อมูล Event
    properties JSON,
    
    -- เวลา
    occurred_at TIMESTAMP(3) DEFAULT CURRENT_TIMESTAMP(3),
    
    -- สำหรับ Partitioning
    event_date DATE GENERATED ALWAYS AS (DATE(occurred_at)) STORED,
    
    INDEX idx_event_type (event_type),
    INDEX idx_line_user_id (line_user_id),
    INDEX idx_occurred_at (occurred_at),
    INDEX idx_event_date (event_date)
) PARTITION BY RANGE (TO_DAYS(event_date)) (
    PARTITION p_2024_01 VALUES LESS THAN (TO_DAYS('2024-02-01')),
    PARTITION p_2024_02 VALUES LESS THAN (TO_DAYS('2024-03-01')),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- ตาราง message_stats (Aggregated)
CREATE TABLE message_stats (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    stat_date DATE NOT NULL,
    stat_hour TINYINT,
    
    -- จำนวนข้อความ
    total_messages BIGINT DEFAULT 0,
    inbound_messages BIGINT DEFAULT 0,
    outbound_messages BIGINT DEFAULT 0,
    broadcast_messages BIGINT DEFAULT 0,
    
    -- ประเภทข้อความ
    text_messages BIGINT DEFAULT 0,
    image_messages BIGINT DEFAULT 0,
    video_messages BIGINT DEFAULT 0,
    audio_messages BIGINT DEFAULT 0,
    file_messages BIGINT DEFAULT 0,
    location_messages BIGINT DEFAULT 0,
    sticker_messages BIGINT DEFAULT 0,
    
    -- Bot responses
    bot_responses BIGINT DEFAULT 0,
    bot_response_time_avg DECIMAL(10, 2) DEFAULT 0,
    bot_response_time_p95 DECIMAL(10, 2) DEFAULT 0,
    
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY unique_date_hour (stat_date, stat_hour)
);

-- ตาราง user_stats
CREATE TABLE user_stats (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    stat_date DATE NOT NULL UNIQUE,
    
    -- จำนวนผู้ใช้
    total_users BIGINT DEFAULT 0,
    new_users BIGINT DEFAULT 0,
    returning_users BIGINT DEFAULT 0,
    active_users_dau BIGINT DEFAULT 0,
    active_users_wau BIGINT DEFAULT 0,
    active_users_mau BIGINT DEFAULT 0,
    
    -- Followers
    total_followers BIGINT DEFAULT 0,
    new_followers BIGINT DEFAULT 0,
    unfollowers BIGINT DEFAULT 0,
    net_follower_change INT DEFAULT 0,
    
    -- Engagement
    avg_session_duration DECIMAL(10, 2) DEFAULT 0,
    avg_messages_per_session DECIMAL(10, 2) DEFAULT 0,
    
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ตาราง conversion_events
CREATE TABLE conversion_events (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    line_user_id VARCHAR(100) NOT NULL,
    funnel_stage VARCHAR(50) NOT NULL,
    
    -- ข้อมูล Conversion
    product_id VARCHAR(100),
    order_id VARCHAR(100),
    revenue DECIMAL(15, 2),
    currency VARCHAR(3) DEFAULT 'THB',
    
    -- Attribution
    source VARCHAR(100),
    campaign_id VARCHAR(100),
    
    occurred_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_funnel_stage (funnel_stage),
    INDEX idx_occurred_at (occurred_at),
    INDEX idx_campaign_id (campaign_id)
);

-- ตาราง revenue_stats
CREATE TABLE revenue_stats (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    stat_date DATE NOT NULL UNIQUE,
    
    -- รายได้
    gross_revenue DECIMAL(15, 2) DEFAULT 0,
    net_revenue DECIMAL(15, 2) DEFAULT 0,
    refunds DECIMAL(15, 2) DEFAULT 0,
    
    -- Orders
    total_orders INT DEFAULT 0,
    avg_order_value DECIMAL(10, 2) DEFAULT 0,
    
    -- Conversions
    conversion_rate DECIMAL(5, 4) DEFAULT 0,
    cost_per_acquisition DECIMAL(10, 2) DEFAULT 0,
    
    -- Campaign Revenue
    campaign_revenue JSON,
    
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ตาราง bot_performance
CREATE TABLE bot_performance (
    id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    stat_date DATE NOT NULL,
    stat_hour TINYINT,
    
    -- Intent Recognition
    total_intents BIGINT DEFAULT 0,
    recognized_intents BIGINT DEFAULT 0,
    fallback_count BIGINT DEFAULT 0,
    intent_accuracy DECIMAL(5, 4) DEFAULT 0,
    
    -- Response
    avg_response_time_ms DECIMAL(10, 2) DEFAULT 0,
    p95_response_time_ms DECIMAL(10, 2) DEFAULT 0,
    p99_response_time_ms DECIMAL(10, 2) DEFAULT 0,
    
    -- Errors
    error_count BIGINT DEFAULT 0,
    error_rate DECIMAL(5, 4) DEFAULT 0,
    
    -- User Satisfaction
    positive_feedback INT DEFAULT 0,
    negative_feedback INT DEFAULT 0,
    satisfaction_score DECIMAL(3, 2) DEFAULT 0,
    
    UNIQUE KEY unique_date_hour (stat_date, stat_hour)
);
```

## 2. Event Collector Service

```javascript
// src/analytics/event-collector.js
const db = require('../database/mysql');
const { v4: uuidv4 } = require('uuid');
const redis = require('../cache/redis');

class EventCollector {
    constructor() {
        this.batchSize = 100;
        this.batchInterval = 5000; // 5 วินาที
        this.eventBuffer = [];
        
        // เริ่ม Batch Processing
        setInterval(() => this.flushBuffer(), this.batchInterval);
    }

    // บันทึก Event
    async track(eventType, lineUserId, properties = {}) {
        const event = {
            event_id: uuidv4(),
            event_type: eventType,
            line_user_id: lineUserId,
            session_id: await this.getSessionId(lineUserId),
            properties: JSON.stringify(properties),
            occurred_at: new Date().toISOString()
        };
        
        // เพิ่มใน Buffer
        this.eventBuffer.push(event);
        
        // อัพเดท Real-time Counters ใน Redis
        await this.updateRealtimeCounters(eventType, lineUserId);
        
        // Flush ถ้า Buffer เต็ม
        if (this.eventBuffer.length >= this.batchSize) {
            await this.flushBuffer();
        }
    }

    // Flush Buffer ไป Database
    async flushBuffer() {
        if (this.eventBuffer.length === 0) return;
        
        const events = [...this.eventBuffer];
        this.eventBuffer = [];
        
        try {
            const values = events.map(e => [
                e.event_id,
                e.event_type,
                e.line_user_id,
                e.session_id,
                e.properties,
                e.occurred_at
            ]);
            
            await db.query(
                'INSERT IGNORE INTO events (event_id, event_type, line_user_id, session_id, properties, occurred_at) VALUES ?',
                [values]
            );
        } catch (error) {
            console.error('Error flushing event buffer:', error);
            // คืน Events กลับไป Buffer
            this.eventBuffer.unshift(...events);
        }
    }

    // ดึง Session ID
    async getSessionId(lineUserId) {
        const sessionKey = `session:${lineUserId}`;
        let sessionId = await redis.get(sessionKey);
        
        if (!sessionId) {
            sessionId = uuidv4();
            await redis.setex(sessionKey, 1800, sessionId); // 30 นาที
        }
        
        // Extend session
        await redis.expire(sessionKey, 1800);
        
        return sessionId;
    }

    // อัพเดท Real-time Counters
    async updateRealtimeCounters(eventType, lineUserId) {
        const now = new Date();
        const dateKey = now.toISOString().split('T')[0];
        const hourKey = now.getHours();
        
        const pipeline = redis.pipeline();
        
        // Counter รายวัน
        pipeline.hincrby(`stats:${dateKey}`, eventType, 1);
        pipeline.hincrby(`stats:${dateKey}`, 'total_events', 1);
        
        // Counter รายชั่วโมง
        pipeline.hincrby(`stats:${dateKey}:${hourKey}`, eventType, 1);
        
        // Active Users
        pipeline.sadd(`active_users:${dateKey}`, lineUserId);
        pipeline.sadd(`active_users:${dateKey}:${hourKey}`, lineUserId);
        
        // Set TTL
        pipeline.expire(`stats:${dateKey}`, 86400 * 7);
        pipeline.expire(`active_users:${dateKey}`, 86400 * 7);
        
        await pipeline.exec();
    }

    // ดึง Real-time Stats
    async getRealtimeStats() {
        const now = new Date();
        const dateKey = now.toISOString().split('T')[0];
        const hourKey = now.getHours();
        
        const [dayStats, hourStats, activeUsersDay, activeUsersHour] = await Promise.all([
            redis.hgetall(`stats:${dateKey}`),
            redis.hgetall(`stats:${dateKey}:${hourKey}`),
            redis.scard(`active_users:${dateKey}`),
            redis.scard(`active_users:${dateKey}:${hourKey}`)
        ]);
        
        return {
            today: {
                ...dayStats,
                active_users: activeUsersDay
            },
            thisHour: {
                ...hourStats,
                active_users: activeUsersHour
            }
        };
    }
}

module.exports = new EventCollector();
```

## 3. Express.js Dashboard Server

```javascript
// src/dashboard/server.js
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');
const path = require('path');
const db = require('../database/mysql');
const eventCollector = require('../analytics/event-collector');
const redis = require('../cache/redis');

const app = express();
const server = http.createServer(app);
const io = socketIo(server, {
    cors: {
        origin: process.env.DASHBOARD_ORIGIN || '*',
        methods: ['GET', 'POST']
    }
});

app.use(express.json());
app.use(express.static(path.join(__dirname, 'public')));

// ============================================
// REST API Endpoints
// ============================================

// Real-time Stats
app.get('/api/stats/realtime', async (req, res) => {
    try {
        const stats = await eventCollector.getRealtimeStats();
        res.json(stats);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Message Volume Dashboard
app.get('/api/stats/messages', async (req, res) => {
    try {
        const { period = '7d', interval = 'day' } = req.query;
        
        const periodDays = { '1d': 1, '7d': 7, '30d': 30, '90d': 90 }[period] || 7;
        
        let groupBy;
        let dateFormat;
        switch (interval) {
            case 'hour':
                groupBy = 'DATE_FORMAT(stat_date, "%Y-%m-%d %H:00:00")';
                dateFormat = '%Y-%m-%d %H:00:00';
                break;
            case 'week':
                groupBy = 'YEARWEEK(stat_date, 1)';
                dateFormat = '%Y-%u';
                break;
            default: // day
                groupBy = 'stat_date';
                dateFormat = '%Y-%m-%d';
        }
        
        const data = await db.query(`
            SELECT 
                DATE_FORMAT(stat_date, '${dateFormat}') as period,
                SUM(total_messages) as total,
                SUM(inbound_messages) as inbound,
                SUM(outbound_messages) as outbound,
                SUM(broadcast_messages) as broadcast,
                AVG(bot_response_time_avg) as avg_response_time
            FROM message_stats
            WHERE stat_date >= DATE_SUB(CURDATE(), INTERVAL ? DAY)
            GROUP BY ${groupBy}
            ORDER BY stat_date ASC
        `, [periodDays]);
        
        res.json(data);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// User Growth Dashboard
app.get('/api/stats/users', async (req, res) => {
    try {
        const { period = '30d' } = req.query;
        const periodDays = { '7d': 7, '30d': 30, '90d': 90, '1y': 365 }[period] || 30;
        
        const data = await db.query(`
            SELECT 
                stat_date,
                total_followers,
                new_followers,
                unfollowers,
                net_follower_change,
                active_users_dau,
                active_users_wau,
                active_users_mau,
                avg_session_duration,
                avg_messages_per_session
            FROM user_stats
            WHERE stat_date >= DATE_SUB(CURDATE(), INTERVAL ? DAY)
            ORDER BY stat_date ASC
        `, [periodDays]);
        
        // คำนวณ Growth Rate
        const processedData = data.map((row, index) => {
            const prevRow = data[index - 1];
            const growthRate = prevRow 
                ? ((row.total_followers - prevRow.total_followers) / prevRow.total_followers * 100).toFixed(2)
                : 0;
            return { ...row, growth_rate: parseFloat(growthRate) };
        });
        
        res.json(processedData);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Conversion Funnel
app.get('/api/stats/funnel', async (req, res) => {
    try {
        const { period = '30d', campaign_id } = req.query;
        const periodDays = { '7d': 7, '30d': 30, '90d': 90 }[period] || 30;
        
        const stages = [
            'impression',
            'message_sent',
            'engaged',
            'product_viewed',
            'added_to_cart',
            'checkout_started',
            'purchased'
        ];
        
        let query = `
            SELECT funnel_stage, COUNT(DISTINCT line_user_id) as users
            FROM conversion_events
            WHERE occurred_at >= DATE_SUB(NOW(), INTERVAL ? DAY)
        `;
        const params = [periodDays];
        
        if (campaign_id) {
            query += ' AND campaign_id = ?';
            params.push(campaign_id);
        }
        
        query += ' GROUP BY funnel_stage';
        
        const data = await db.query(query, params);
        
        const funnelData = stages.map((stage, index) => {
            const stageData = data.find(d => d.funnel_stage === stage);
            const users = stageData?.users || 0;
            const prevStage = index > 0 ? stages[index - 1] : null;
            const prevData = prevStage ? data.find(d => d.funnel_stage === prevStage) : null;
            const dropoffRate = prevData?.users 
                ? ((prevData.users - users) / prevData.users * 100).toFixed(1)
                : 0;
            
            return {
                stage,
                users,
                dropoff_rate: parseFloat(dropoffRate),
                conversion_rate: index === 0 ? 100 : 
                    data[0]?.users ? ((users / data[0].users) * 100).toFixed(2) : 0
            };
        });
        
        res.json(funnelData);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Revenue Analytics
app.get('/api/stats/revenue', async (req, res) => {
    try {
        const { period = '30d' } = req.query;
        const periodDays = { '7d': 7, '30d': 30, '90d': 90, '1y': 365 }[period] || 30;
        
        const [dailyRevenue, summary] = await Promise.all([
            db.query(`
                SELECT 
                    stat_date,
                    gross_revenue,
                    net_revenue,
                    total_orders,
                    avg_order_value,
                    conversion_rate
                FROM revenue_stats
                WHERE stat_date >= DATE_SUB(CURDATE(), INTERVAL ? DAY)
                ORDER BY stat_date ASC
            `, [periodDays]),
            
            db.query(`
                SELECT 
                    SUM(gross_revenue) as total_gross_revenue,
                    SUM(net_revenue) as total_net_revenue,
                    SUM(total_orders) as total_orders,
                    AVG(avg_order_value) as avg_order_value,
                    AVG(conversion_rate) as avg_conversion_rate,
                    SUM(gross_revenue) / NULLIF(SUM(total_orders), 0) as overall_aov
                FROM revenue_stats
                WHERE stat_date >= DATE_SUB(CURDATE(), INTERVAL ? DAY)
            `, [periodDays])
        ]);
        
        // คำนวณ Revenue Growth
        const currentPeriod = await db.query(`
            SELECT SUM(gross_revenue) as revenue
            FROM revenue_stats
            WHERE stat_date >= DATE_SUB(CURDATE(), INTERVAL ? DAY)
        `, [periodDays]);
        
        const prevPeriod = await db.query(`
            SELECT SUM(gross_revenue) as revenue
            FROM revenue_stats
            WHERE stat_date >= DATE_SUB(CURDATE(), INTERVAL ? DAY)
            AND stat_date < DATE_SUB(CURDATE(), INTERVAL ? DAY)
        `, [periodDays * 2, periodDays]);
        
        const revenueGrowth = prevPeriod[0]?.revenue 
            ? ((currentPeriod[0].revenue - prevPeriod[0].revenue) / prevPeriod[0].revenue * 100).toFixed(1)
            : null;
        
        res.json({
            daily: dailyRevenue,
            summary: {
                ...summary[0],
                revenue_growth: revenueGrowth
            }
        });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// Bot Performance Metrics
app.get('/api/stats/bot-performance', async (req, res) => {
    try {
        const { period = '7d' } = req.query;
        const periodDays = { '1d': 1, '7d': 7, '30d': 30 }[period] || 7;
        
        const data = await db.query(`
            SELECT 
                stat_date,
                SUM(total_intents) as total_intents,
                SUM(recognized_intents) as recognized_intents,
                SUM(fallback_count) as fallback_count,
                AVG(intent_accuracy) as intent_accuracy,
                AVG(avg_response_time_ms) as avg_response_time_ms,
                AVG(p95_response_time_ms) as p95_response_time_ms,
                SUM(error_count) as error_count,
                AVG(error_rate) as error_rate,
                SUM(positive_feedback) as positive_feedback,
                SUM(negative_feedback) as negative_feedback,
                AVG(satisfaction_score) as satisfaction_score
            FROM bot_performance
            WHERE stat_date >= DATE_SUB(CURDATE(), INTERVAL ? DAY)
            GROUP BY stat_date
            ORDER BY stat_date ASC
        `, [periodDays]);
        
        res.json(data);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// ============================================
// Socket.io Real-time Updates
// ============================================

io.on('connection', (socket) => {
    console.log('Dashboard client connected:', socket.id);
    
    // ส่งข้อมูล Initial
    socket.emit('connected', { message: 'Connected to analytics stream' });
    
    // Subscribe ไป Real-time Stats
    socket.on('subscribe', (channels) => {
        channels.forEach(channel => socket.join(channel));
    });
    
    socket.on('disconnect', () => {
        console.log('Dashboard client disconnected:', socket.id);
    });
});

// Real-time Stats Broadcaster (ทุก 5 วินาที)
setInterval(async () => {
    try {
        const stats = await eventCollector.getRealtimeStats();
        io.to('realtime').emit('stats_update', stats);
    } catch (error) {
        console.error('Error broadcasting stats:', error);
    }
}, 5000);

// Real-time Message Counter Broadcaster (ทุก 1 วินาที)
setInterval(async () => {
    try {
        const now = new Date();
        const dateKey = now.toISOString().split('T')[0];
        
        const [msgCount, activeUsers] = await Promise.all([
            redis.hget(`stats:${dateKey}`, 'message'),
            redis.scard(`active_users:${dateKey}`)
        ]);
        
        io.to('messages').emit('message_count', {
            count: parseInt(msgCount) || 0,
            activeUsers: activeUsers || 0,
            timestamp: now.toISOString()
        });
    } catch (error) {
        // ไม่ต้อง log error นี้เพราะจะ spam
    }
}, 1000);

module.exports = { app, server, io };
```

## 4. Dashboard Frontend (HTML + Chart.js)

```html
<!-- src/dashboard/public/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>LINE Bot Analytics Dashboard</title>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
    <script src="https://cdn.socket.io/4.7.2/socket.io.min.js"></script>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Segoe UI', sans-serif; background: #0f172a; color: #e2e8f0; }
        
        .header {
            background: #1e293b;
            padding: 16px 24px;
            display: flex;
            align-items: center;
            gap: 12px;
            border-bottom: 1px solid #334155;
        }
        
        .header h1 { font-size: 1.2rem; color: #06b6d4; }
        
        .status-dot {
            width: 8px; height: 8px;
            border-radius: 50%;
            background: #22c55e;
            animation: pulse 2s infinite;
        }
        
        @keyframes pulse {
            0%, 100% { opacity: 1; }
            50% { opacity: 0.3; }
        }
        
        .container { padding: 24px; max-width: 1600px; margin: 0 auto; }
        
        .kpi-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 16px;
            margin-bottom: 24px;
        }
        
        .kpi-card {
            background: #1e293b;
            border-radius: 12px;
            padding: 20px;
            border: 1px solid #334155;
        }
        
        .kpi-card .label { 
            font-size: 0.75rem; 
            color: #94a3b8; 
            text-transform: uppercase; 
            letter-spacing: 0.05em;
        }
        
        .kpi-card .value { 
            font-size: 2rem; 
            font-weight: 700; 
            color: #f1f5f9; 
            margin: 8px 0;
        }
        
        .kpi-card .change { font-size: 0.8rem; }
        .change.positive { color: #22c55e; }
        .change.negative { color: #ef4444; }
        
        .charts-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(500px, 1fr));
            gap: 16px;
            margin-bottom: 24px;
        }
        
        .chart-card {
            background: #1e293b;
            border-radius: 12px;
            padding: 20px;
            border: 1px solid #334155;
        }
        
        .chart-card h3 { 
            font-size: 0.9rem; 
            color: #94a3b8; 
            margin-bottom: 16px;
        }
        
        .chart-wrapper { position: relative; height: 250px; }
        
        .period-selector {
            display: flex;
            gap: 8px;
            margin-bottom: 16px;
        }
        
        .period-btn {
            padding: 4px 12px;
            border-radius: 6px;
            border: 1px solid #334155;
            background: transparent;
            color: #94a3b8;
            cursor: pointer;
            font-size: 0.8rem;
        }
        
        .period-btn.active {
            background: #06b6d4;
            color: white;
            border-color: #06b6d4;
        }
        
        .funnel-container { 
            display: flex; 
            flex-direction: column; 
            gap: 8px;
        }
        
        .funnel-stage {
            display: flex;
            align-items: center;
            gap: 12px;
        }
        
        .funnel-bar-container {
            flex: 1;
            background: #334155;
            border-radius: 4px;
            height: 32px;
            position: relative;
            overflow: hidden;
        }
        
        .funnel-bar {
            height: 100%;
            background: linear-gradient(90deg, #06b6d4, #3b82f6);
            border-radius: 4px;
            transition: width 0.5s ease;
        }
        
        .funnel-label { 
            width: 140px; 
            font-size: 0.8rem; 
            color: #94a3b8;
        }
        
        .funnel-count { 
            width: 80px; 
            text-align: right; 
            font-size: 0.9rem;
        }
        
        .realtime-indicator {
            display: flex;
            align-items: center;
            gap: 8px;
            color: #22c55e;
            font-size: 0.8rem;
        }
    </style>
</head>
<body>
    <div class="header">
        <div class="status-dot" id="connectionStatus"></div>
        <h1>LINE Bot Analytics Dashboard</h1>
        <span class="realtime-indicator">
            <span id="realtimeStatus">● Live</span>
        </span>
    </div>
    
    <div class="container">
        <!-- KPI Cards -->
        <div class="kpi-grid" id="kpiGrid">
            <div class="kpi-card">
                <div class="label">ข้อความวันนี้</div>
                <div class="value" id="kpi-messages">-</div>
                <div class="change" id="kpi-messages-change">-</div>
            </div>
            <div class="kpi-card">
                <div class="label">ผู้ใช้ Active</div>
                <div class="value" id="kpi-active-users">-</div>
                <div class="change" id="kpi-active-users-change">-</div>
            </div>
            <div class="kpi-card">
                <div class="label">Followers ทั้งหมด</div>
                <div class="value" id="kpi-followers">-</div>
                <div class="change" id="kpi-followers-change">-</div>
            </div>
            <div class="kpi-card">
                <div class="label">รายได้วันนี้</div>
                <div class="value" id="kpi-revenue">-</div>
                <div class="change" id="kpi-revenue-change">-</div>
            </div>
            <div class="kpi-card">
                <div class="label">Conversion Rate</div>
                <div class="value" id="kpi-conversion">-</div>
                <div class="change" id="kpi-conversion-change">-</div>
            </div>
            <div class="kpi-card">
                <div class="label">Bot Accuracy</div>
                <div class="value" id="kpi-accuracy">-</div>
                <div class="change" id="kpi-accuracy-change">-</div>
            </div>
        </div>
        
        <!-- Charts -->
        <div class="charts-grid">
            <!-- Message Volume Chart -->
            <div class="chart-card">
                <h3>📨 ปริมาณข้อความ</h3>
                <div class="period-selector">
                    <button class="period-btn active" onclick="loadMessageChart('7d', this)">7 วัน</button>
                    <button class="period-btn" onclick="loadMessageChart('30d', this)">30 วัน</button>
                    <button class="period-btn" onclick="loadMessageChart('90d', this)">90 วัน</button>
                </div>
                <div class="chart-wrapper">
                    <canvas id="messageChart"></canvas>
                </div>
            </div>
            
            <!-- User Growth Chart -->
            <div class="chart-card">
                <h3>👥 การเติบโตของผู้ใช้</h3>
                <div class="period-selector">
                    <button class="period-btn active" onclick="loadUserChart('30d', this)">30 วัน</button>
                    <button class="period-btn" onclick="loadUserChart('90d', this)">90 วัน</button>
                    <button class="period-btn" onclick="loadUserChart('1y', this)">1 ปี</button>
                </div>
                <div class="chart-wrapper">
                    <canvas id="userChart"></canvas>
                </div>
            </div>
            
            <!-- Revenue Chart -->
            <div class="chart-card">
                <h3>💰 รายได้</h3>
                <div class="period-selector">
                    <button class="period-btn active" onclick="loadRevenueChart('30d', this)">30 วัน</button>
                    <button class="period-btn" onclick="loadRevenueChart('90d', this)">90 วัน</button>
                </div>
                <div class="chart-wrapper">
                    <canvas id="revenueChart"></canvas>
                </div>
            </div>
            
            <!-- Bot Performance Chart -->
            <div class="chart-card">
                <h3>🤖 Bot Performance</h3>
                <div class="period-selector">
                    <button class="period-btn active" onclick="loadBotChart('7d', this)">7 วัน</button>
                    <button class="period-btn" onclick="loadBotChart('30d', this)">30 วัน</button>
                </div>
                <div class="chart-wrapper">
                    <canvas id="botChart"></canvas>
                </div>
            </div>
        </div>
        
        <!-- Conversion Funnel -->
        <div class="chart-card" style="margin-bottom: 24px;">
            <h3>🔄 Conversion Funnel</h3>
            <div id="funnelChart" class="funnel-container"></div>
        </div>
    </div>
    
    <script>
        // ============================================
        // Chart.js Configuration
        // ============================================
        const chartDefaults = {
            responsive: true,
            maintainAspectRatio: false,
            plugins: {
                legend: {
                    labels: { color: '#94a3b8', font: { size: 11 } }
                }
            },
            scales: {
                x: {
                    ticks: { color: '#94a3b8', font: { size: 10 } },
                    grid: { color: '#1e293b' }
                },
                y: {
                    ticks: { color: '#94a3b8', font: { size: 10 } },
                    grid: { color: '#334155' }
                }
            }
        };
        
        let charts = {};
        
        // ============================================
        // Load Message Volume Chart
        // ============================================
        async function loadMessageChart(period = '7d', btn = null) {
            updatePeriodButtons(btn);
            
            const data = await fetchAPI(`/api/stats/messages?period=${period}`);
            
            const labels = data.map(d => d.period);
            const datasets = [
                {
                    label: 'ขาเข้า',
                    data: data.map(d => d.inbound),
                    borderColor: '#06b6d4',
                    backgroundColor: 'rgba(6, 182, 212, 0.1)',
                    fill: true,
                    tension: 0.3
                },
                {
                    label: 'ขาออก',
                    data: data.map(d => d.outbound),
                    borderColor: '#3b82f6',
                    backgroundColor: 'rgba(59, 130, 246, 0.1)',
                    fill: true,
                    tension: 0.3
                },
                {
                    label: 'Broadcast',
                    data: data.map(d => d.broadcast),
                    borderColor: '#a855f7',
                    backgroundColor: 'rgba(168, 85, 247, 0.1)',
                    fill: true,
                    tension: 0.3
                }
            ];
            
            renderLineChart('messageChart', labels, datasets);
        }
        
        // ============================================
        // Load User Growth Chart
        // ============================================
        async function loadUserChart(period = '30d', btn = null) {
            updatePeriodButtons(btn);
            
            const data = await fetchAPI(`/api/stats/users?period=${period}`);
            
            const labels = data.map(d => d.stat_date);
            const datasets = [
                {
                    label: 'Followers ทั้งหมด',
                    data: data.map(d => d.total_followers),
                    borderColor: '#22c55e',
                    backgroundColor: 'rgba(34, 197, 94, 0.1)',
                    fill: true,
                    tension: 0.3,
                    yAxisID: 'y'
                },
                {
                    label: 'DAU',
                    data: data.map(d => d.active_users_dau),
                    borderColor: '#f59e0b',
                    backgroundColor: 'rgba(245, 158, 11, 0.1)',
                    fill: true,
                    tension: 0.3,
                    yAxisID: 'y1'
                }
            ];
            
            renderDualAxisChart('userChart', labels, datasets);
        }
        
        // ============================================
        // Load Revenue Chart
        // ============================================
        async function loadRevenueChart(period = '30d', btn = null) {
            updatePeriodButtons(btn);
            
            const { daily } = await fetchAPI(`/api/stats/revenue?period=${period}`);
            
            const labels = daily.map(d => d.stat_date);
            const datasets = [
                {
                    label: 'รายได้ (บาท)',
                    data: daily.map(d => d.gross_revenue),
                    borderColor: '#f59e0b',
                    backgroundColor: 'rgba(245, 158, 11, 0.15)',
                    fill: true,
                    tension: 0.3,
                    yAxisID: 'y'
                },
                {
                    label: 'จำนวน Orders',
                    type: 'bar',
                    data: daily.map(d => d.total_orders),
                    backgroundColor: 'rgba(59, 130, 246, 0.4)',
                    yAxisID: 'y1'
                }
            ];
            
            renderDualAxisChart('revenueChart', labels, datasets);
        }
        
        // ============================================
        // Load Bot Performance Chart
        // ============================================
        async function loadBotChart(period = '7d', btn = null) {
            updatePeriodButtons(btn);
            
            const data = await fetchAPI(`/api/stats/bot-performance?period=${period}`);
            
            const labels = data.map(d => d.stat_date);
            const datasets = [
                {
                    label: 'Response Time (ms)',
                    data: data.map(d => d.avg_response_time_ms),
                    borderColor: '#06b6d4',
                    fill: false,
                    tension: 0.3,
                    yAxisID: 'y'
                },
                {
                    label: 'Intent Accuracy (%)',
                    data: data.map(d => (d.intent_accuracy * 100).toFixed(1)),
                    borderColor: '#22c55e',
                    fill: false,
                    tension: 0.3,
                    yAxisID: 'y1'
                },
                {
                    label: 'Error Rate (%)',
                    data: data.map(d => (d.error_rate * 100).toFixed(2)),
                    borderColor: '#ef4444',
                    fill: false,
                    tension: 0.3,
                    yAxisID: 'y1'
                }
            ];
            
            renderDualAxisChart('botChart', labels, datasets);
        }
        
        // ============================================
        // Load Conversion Funnel
        // ============================================
        async function loadFunnelChart() {
            const data = await fetchAPI('/api/stats/funnel?period=30d');
            
            const maxUsers = data[0]?.users || 1;
            const container = document.getElementById('funnelChart');
            
            container.innerHTML = data.map(stage => `
                <div class="funnel-stage">
                    <div class="funnel-label">${formatStageName(stage.stage)}</div>
                    <div class="funnel-bar-container">
                        <div class="funnel-bar" style="width: ${(stage.users / maxUsers * 100)}%"></div>
                    </div>
                    <div class="funnel-count">${formatNumber(stage.users)}</div>
                    <div class="funnel-count" style="color: ${stage.dropoff_rate > 50 ? '#ef4444' : '#94a3b8'}">
                        ${stage.dropoff_rate > 0 ? `-${stage.dropoff_rate}%` : ''}
                    </div>
                </div>
            `).join('');
        }
        
        // ============================================
        // Chart Helpers
        // ============================================
        function renderLineChart(canvasId, labels, datasets) {
            if (charts[canvasId]) charts[canvasId].destroy();
            
            const ctx = document.getElementById(canvasId).getContext('2d');
            charts[canvasId] = new Chart(ctx, {
                type: 'line',
                data: { labels, datasets },
                options: { ...chartDefaults }
            });
        }
        
        function renderDualAxisChart(canvasId, labels, datasets) {
            if (charts[canvasId]) charts[canvasId].destroy();
            
            const ctx = document.getElementById(canvasId).getContext('2d');
            charts[canvasId] = new Chart(ctx, {
                data: { labels, datasets },
                options: {
                    ...chartDefaults,
                    scales: {
                        ...chartDefaults.scales,
                        y: { ...chartDefaults.scales.y, position: 'left' },
                        y1: { ...chartDefaults.scales.y, position: 'right', grid: { drawOnChartArea: false } }
                    }
                }
            });
        }
        
        function updatePeriodButtons(activeBtn) {
            if (!activeBtn) return;
            const siblings = activeBtn.parentElement.querySelectorAll('.period-btn');
            siblings.forEach(b => b.classList.remove('active'));
            activeBtn.classList.add('active');
        }
        
        async function fetchAPI(url) {
            const response = await fetch(url);
            return response.json();
        }
        
        function formatNumber(n) {
            if (n >= 1000000) return (n / 1000000).toFixed(1) + 'M';
            if (n >= 1000) return (n / 1000).toFixed(1) + 'K';
            return n?.toLocaleString() || '0';
        }
        
        function formatCurrency(n) {
            return '฿' + (n || 0).toLocaleString('th-TH', { minimumFractionDigits: 0 });
        }
        
        function formatStageName(stage) {
            const names = {
                impression: 'เห็นโฆษณา',
                message_sent: 'ส่งข้อความ',
                engaged: 'มีส่วนร่วม',
                product_viewed: 'ดูสินค้า',
                added_to_cart: 'เพิ่มลงตะกร้า',
                checkout_started: 'เริ่ม Checkout',
                purchased: 'ซื้อสำเร็จ'
            };
            return names[stage] || stage;
        }
        
        // ============================================
        // Socket.io Real-time Updates
        // ============================================
        const socket = io();
        
        socket.on('connect', () => {
            document.getElementById('realtimeStatus').textContent = '● Live';
            document.getElementById('connectionStatus').style.background = '#22c55e';
            socket.emit('subscribe', ['realtime', 'messages']);
        });
        
        socket.on('disconnect', () => {
            document.getElementById('realtimeStatus').textContent = '● Offline';
            document.getElementById('connectionStatus').style.background = '#ef4444';
        });
        
        socket.on('stats_update', (stats) => {
            // อัพเดท KPI Cards
            const today = stats.today;
            
            document.getElementById('kpi-messages').textContent = 
                formatNumber(parseInt(today.message) || 0);
            document.getElementById('kpi-active-users').textContent = 
                formatNumber(today.active_users);
        });
        
        socket.on('message_count', (data) => {
            // อัพเดท Live Counter
            document.getElementById('kpi-messages').textContent = 
                formatNumber(data.count);
        });
        
        // ============================================
        // Load KPI Summary
        // ============================================
        async function loadKPISummary() {
            try {
                const [userStats, revenueStats, botStats] = await Promise.all([
                    fetchAPI('/api/stats/users?period=1d'),
                    fetchAPI('/api/stats/revenue?period=7d'),
                    fetchAPI('/api/stats/bot-performance?period=1d')
                ]);
                
                // Followers
                if (userStats.length > 0) {
                    const latest = userStats[userStats.length - 1];
                    document.getElementById('kpi-followers').textContent = 
                        formatNumber(latest.total_followers);
                    
                    const change = latest.net_follower_change;
                    const el = document.getElementById('kpi-followers-change');
                    el.textContent = `${change >= 0 ? '+' : ''}${change} วันนี้`;
                    el.className = `change ${change >= 0 ? 'positive' : 'negative'}`;
                }
                
                // Revenue
                if (revenueStats.summary) {
                    document.getElementById('kpi-revenue').textContent = 
                        formatCurrency(revenueStats.summary.total_gross_revenue);
                    
                    const growth = revenueStats.summary.revenue_growth;
                    const el = document.getElementById('kpi-revenue-change');
                    if (growth !== null) {
                        el.textContent = `${growth >= 0 ? '+' : ''}${growth}% vs ช่วงก่อน`;
                        el.className = `change ${growth >= 0 ? 'positive' : 'negative'}`;
                    }
                    
                    document.getElementById('kpi-conversion').textContent = 
                        (revenueStats.summary.avg_conversion_rate * 100).toFixed(1) + '%';
                }
                
                // Bot Accuracy
                if (botStats.length > 0) {
                    const latest = botStats[botStats.length - 1];
                    document.getElementById('kpi-accuracy').textContent = 
                        (latest.intent_accuracy * 100).toFixed(1) + '%';
                }
            } catch (error) {
                console.error('Error loading KPI:', error);
            }
        }
        
        // ============================================
        // Initialize Dashboard
        // ============================================
        async function initDashboard() {
            await Promise.all([
                loadKPISummary(),
                loadMessageChart('7d'),
                loadUserChart('30d'),
                loadRevenueChart('30d'),
                loadBotChart('7d'),
                loadFunnelChart()
            ]);
        }
        
        initDashboard();
        
        // Auto-refresh ทุก 5 นาที
        setInterval(loadKPISummary, 300000);
    </script>
</body>
</html>
```

## 5. Analytics Aggregation Job

```javascript
// src/analytics/aggregation-job.js
const cron = require('node-cron');
const db = require('../database/mysql');
const redis = require('../cache/redis');
const lineClient = require('../line/client');

class AnalyticsAggregationJob {
    startJobs() {
        // รัน Hourly Aggregation
        cron.schedule('5 * * * *', () => this.runHourlyAggregation());
        
        // รัน Daily Aggregation เที่ยงคืน
        cron.schedule('0 0 * * *', () => this.runDailyAggregation());
        
        // ส่ง Weekly Report วันจันทร์ 9 โมง
        cron.schedule('0 9 * * 1', () => this.sendWeeklyReport());
        
        console.log('Analytics aggregation jobs started');
    }

    // Hourly Aggregation
    async runHourlyAggregation() {
        const now = new Date();
        const date = now.toISOString().split('T')[0];
        const hour = now.getHours() - 1; // ชั่วโมงที่ผ่านมา
        
        try {
            // นับจำนวน Messages
            await db.query(`
                INSERT INTO message_stats (stat_date, stat_hour, total_messages, inbound_messages, outbound_messages)
                SELECT 
                    DATE(occurred_at) as stat_date,
                    HOUR(occurred_at) as stat_hour,
                    COUNT(*) as total,
                    SUM(CASE WHEN JSON_EXTRACT(properties, '$.direction') = 'inbound' THEN 1 ELSE 0 END) as inbound,
                    SUM(CASE WHEN JSON_EXTRACT(properties, '$.direction') = 'outbound' THEN 1 ELSE 0 END) as outbound
                FROM events
                WHERE 
                    event_type = 'message'
                    AND DATE(occurred_at) = ?
                    AND HOUR(occurred_at) = ?
                GROUP BY DATE(occurred_at), HOUR(occurred_at)
                ON DUPLICATE KEY UPDATE
                    total_messages = VALUES(total_messages),
                    inbound_messages = VALUES(inbound_messages),
                    outbound_messages = VALUES(outbound_messages)
            `, [date, hour]);
            
            // Bot Performance
            await db.query(`
                INSERT INTO bot_performance (stat_date, stat_hour, total_intents, recognized_intents, fallback_count, avg_response_time_ms)
                SELECT 
                    DATE(occurred_at),
                    HOUR(occurred_at),
                    COUNT(*) as total_intents,
                    SUM(CASE WHEN JSON_EXTRACT(properties, '$.recognized') = 1 THEN 1 ELSE 0 END) as recognized,
                    SUM(CASE WHEN JSON_EXTRACT(properties, '$.is_fallback') = 1 THEN 1 ELSE 0 END) as fallbacks,
                    AVG(CAST(JSON_EXTRACT(properties, '$.response_time_ms') AS DECIMAL(10,2))) as avg_response_time
                FROM events
                WHERE 
                    event_type = 'bot_response'
                    AND DATE(occurred_at) = ?
                    AND HOUR(occurred_at) = ?
                GROUP BY DATE(occurred_at), HOUR(occurred_at)
                ON DUPLICATE KEY UPDATE
                    total_intents = VALUES(total_intents),
                    recognized_intents = VALUES(recognized_intents),
                    fallback_count = VALUES(fallback_count),
                    avg_response_time_ms = VALUES(avg_response_time_ms),
                    intent_accuracy = VALUES(recognized_intents) / NULLIF(VALUES(total_intents), 0)
            `, [date, hour]);
            
        } catch (error) {
            console.error('Hourly aggregation error:', error);
        }
    }

    // Daily Aggregation
    async runDailyAggregation() {
        const yesterday = new Date();
        yesterday.setDate(yesterday.getDate() - 1);
        const date = yesterday.toISOString().split('T')[0];
        
        try {
            // User Stats
            const userStats = await db.query(`
                SELECT 
                    COUNT(DISTINCT line_user_id) as active_users,
                    COUNT(DISTINCT CASE WHEN DATE(first_seen_at) = ? THEN line_user_id END) as new_users
                FROM events
                WHERE DATE(occurred_at) = ?
            `, [date, date]);
            
            const followerStats = await db.query(`
                SELECT 
                    COUNT(CASE WHEN event_type = 'follow' THEN 1 END) as new_followers,
                    COUNT(CASE WHEN event_type = 'unfollow' THEN 1 END) as unfollowers
                FROM events
                WHERE DATE(occurred_at) = ?
            `, [date]);
            
            await db.query(`
                INSERT INTO user_stats (
                    stat_date, active_users_dau, new_users,
                    new_followers, unfollowers, net_follower_change
                ) VALUES (?, ?, ?, ?, ?, ?)
                ON DUPLICATE KEY UPDATE
                    active_users_dau = VALUES(active_users_dau),
                    new_users = VALUES(new_users),
                    new_followers = VALUES(new_followers),
                    unfollowers = VALUES(unfollowers),
                    net_follower_change = VALUES(net_follower_change)
            `, [
                date,
                userStats[0].active_users,
                userStats[0].new_users,
                followerStats[0].new_followers,
                followerStats[0].unfollowers,
                followerStats[0].new_followers - followerStats[0].unfollowers
            ]);
            
            // Revenue Stats
            await db.query(`
                INSERT INTO revenue_stats (stat_date, gross_revenue, total_orders, avg_order_value, conversion_rate)
                SELECT 
                    DATE(occurred_at) as stat_date,
                    SUM(CAST(JSON_EXTRACT(properties, '$.revenue') AS DECIMAL(15,2))) as gross_revenue,
                    COUNT(*) as total_orders,
                    AVG(CAST(JSON_EXTRACT(properties, '$.revenue') AS DECIMAL(15,2))) as avg_order_value,
                    COUNT(*) / NULLIF((SELECT COUNT(DISTINCT line_user_id) FROM events WHERE event_type = 'session_start' AND DATE(occurred_at) = ?), 0) as conversion_rate
                FROM events
                WHERE event_type = 'purchase' AND DATE(occurred_at) = ?
                GROUP BY DATE(occurred_at)
                ON DUPLICATE KEY UPDATE
                    gross_revenue = VALUES(gross_revenue),
                    total_orders = VALUES(total_orders),
                    avg_order_value = VALUES(avg_order_value),
                    conversion_rate = VALUES(conversion_rate)
            `, [date, date]);
            
            console.log(`Daily aggregation completed for ${date}`);
        } catch (error) {
            console.error('Daily aggregation error:', error);
        }
    }

    // ส่ง Weekly Report ผ่าน LINE Broadcast
    async sendWeeklyReport() {
        try {
            const weekAgo = new Date();
            weekAgo.setDate(weekAgo.getDate() - 7);
            const weekAgoDate = weekAgo.toISOString().split('T')[0];
            
            // ดึงข้อมูลสรุปสัปดาห์
            const [msgStats, userStats, revenueStats] = await Promise.all([
                db.query(`
                    SELECT SUM(total_messages) as total, SUM(inbound_messages) as inbound
                    FROM message_stats WHERE stat_date >= ?
                `, [weekAgoDate]),
                
                db.query(`
                    SELECT SUM(new_followers) as new_followers, AVG(active_users_dau) as avg_dau
                    FROM user_stats WHERE stat_date >= ?
                `, [weekAgoDate]),
                
                db.query(`
                    SELECT SUM(gross_revenue) as revenue, SUM(total_orders) as orders
                    FROM revenue_stats WHERE stat_date >= ?
                `, [weekAgoDate])
            ]);
            
            const msg = userStats[0];
            const rev = revenueStats[0];
            const msgData = msgStats[0];
            
            // สร้าง Report Message
            const reportText = `📊 รายงานประจำสัปดาห์\n\n` +
                `📨 ข้อความทั้งหมด: ${parseInt(msgData.total || 0).toLocaleString()}\n` +
                `👥 ผู้ติดตามใหม่: ${parseInt(msg.new_followers || 0).toLocaleString()}\n` +
                `📈 DAU เฉลี่ย: ${parseInt(msg.avg_dau || 0).toLocaleString()}\n` +
                `💰 รายได้รวม: ฿${parseInt(rev.revenue || 0).toLocaleString()}\n` +
                `🛒 จำนวน Orders: ${parseInt(rev.orders || 0).toLocaleString()}\n\n` +
                `ดูรายงานเพิ่มเติม: ${process.env.DASHBOARD_URL}`;
            
            // ส่งให้ Admin ผ่าน LINE
            const adminIds = process.env.ADMIN_LINE_USER_IDS?.split(',') || [];
            
            for (const adminId of adminIds) {
                await lineClient.pushMessage(adminId.trim(), {
                    type: 'text',
                    text: reportText
                });
            }
            
            console.log('Weekly report sent');
        } catch (error) {
            console.error('Error sending weekly report:', error);
        }
    }
}

module.exports = new AnalyticsAggregationJob();
```

## 6. Export ไป Google Sheets

```javascript
// src/analytics/google-sheets-export.js
const { google } = require('googleapis');
const db = require('../database/mysql');

class GoogleSheetsExporter {
    constructor() {
        this.auth = new google.auth.GoogleAuth({
            keyFile: process.env.GOOGLE_SERVICE_ACCOUNT_KEY_FILE,
            scopes: ['https://www.googleapis.com/auth/spreadsheets']
        });
        
        this.spreadsheetId = process.env.GOOGLE_SPREADSHEET_ID;
    }

    async exportDailyStats(date = null) {
        const targetDate = date || new Date().toISOString().split('T')[0];
        
        try {
            const authClient = await this.auth.getClient();
            const sheets = google.sheets({ version: 'v4', auth: authClient });
            
            // ดึงข้อมูลจาก DB
            const [msgStats, userStats, revenueStats] = await Promise.all([
                db.query('SELECT * FROM message_stats WHERE stat_date = ?', [targetDate]),
                db.query('SELECT * FROM user_stats WHERE stat_date = ?', [targetDate]),
                db.query('SELECT * FROM revenue_stats WHERE stat_date = ?', [targetDate])
            ]);
            
            // สร้าง Row ข้อมูล
            const row = [
                targetDate,
                msgStats[0]?.total_messages || 0,
                msgStats[0]?.inbound_messages || 0,
                msgStats[0]?.outbound_messages || 0,
                userStats[0]?.total_followers || 0,
                userStats[0]?.new_followers || 0,
                userStats[0]?.active_users_dau || 0,
                revenueStats[0]?.gross_revenue || 0,
                revenueStats[0]?.total_orders || 0,
                revenueStats[0]?.conversion_rate || 0
            ];
            
            // Append ไปยัง Google Sheet
            await sheets.spreadsheets.values.append({
                spreadsheetId: this.spreadsheetId,
                range: 'Daily Stats!A:J',
                valueInputOption: 'USER_ENTERED',
                resource: { values: [row] }
            });
            
            console.log(`Exported data for ${targetDate} to Google Sheets`);
        } catch (error) {
            console.error('Google Sheets export error:', error);
            throw error;
        }
    }

    // สร้าง Spreadsheet ใหม่
    async createDashboardSpreadsheet() {
        const authClient = await this.auth.getClient();
        const sheets = google.sheets({ version: 'v4', auth: authClient });
        
        const spreadsheet = await sheets.spreadsheets.create({
            resource: {
                properties: { title: 'LINE Bot Analytics Dashboard' },
                sheets: [
                    { properties: { title: 'Daily Stats' } },
                    { properties: { title: 'User Growth' } },
                    { properties: { title: 'Revenue' } },
                    { properties: { title: 'Bot Performance' } }
                ]
            }
        });
        
        // เพิ่ม Headers
        await sheets.spreadsheets.values.update({
            spreadsheetId: spreadsheet.data.spreadsheetId,
            range: 'Daily Stats!A1:J1',
            valueInputOption: 'USER_ENTERED',
            resource: {
                values: [[
                    'วันที่', 'ข้อความทั้งหมด', 'ขาเข้า', 'ขาออก',
                    'Followers ทั้งหมด', 'Followers ใหม่', 'DAU',
                    'รายได้', 'Orders', 'Conversion Rate'
                ]]
            }
        });
        
        return spreadsheet.data.spreadsheetId;
    }
}

module.exports = new GoogleSheetsExporter();
```

## 7. Package.json

```json
{
    "name": "line-analytics-dashboard",
    "version": "1.0.0",
    "scripts": {
        "start": "node src/index.js",
        "dev": "nodemon src/index.js",
        "aggregate": "node src/analytics/run-aggregation.js"
    },
    "dependencies": {
        "@line/bot-sdk": "^9.0.0",
        "chart.js": "^4.4.0",
        "express": "^4.18.2",
        "googleapis": "^128.0.0",
        "ioredis": "^5.3.2",
        "mysql2": "^3.6.0",
        "node-cron": "^3.0.3",
        "socket.io": "^4.7.2",
        "uuid": "^9.0.0"
    }
}
```

## สรุป

ในบทนี้เราสร้าง Analytics Dashboard ที่ประกอบด้วย:

1. **Database Schema** - โครงสร้างข้อมูลสำหรับ Analytics
2. **Event Collector** - บันทึก Events แบบ Batch + Real-time Redis
3. **Express.js API** - REST API สำหรับ Dashboard
4. **Socket.io** - Real-time Updates ผ่าน WebSocket
5. **Chart.js Dashboard** - UI Dashboard พร้อม Charts
6. **Aggregation Jobs** - Cron Jobs สำหรับรวมข้อมูล
7. **Weekly Reports** - รายงานอัตโนมัติผ่าน LINE
8. **Google Sheets Export** - Export ข้อมูลไป Google Sheets
