# Part 70: Analytics Dashboard สำหรับ LINE Bot

## บทนำ

ในบทนี้เราจะสร้างระบบ Analytics ครบวงจรสำหรับ LINE Bot ครอบคลุมการเก็บ metrics, time-series data, dashboard แบบ real-time ด้วย Socket.io, automated reports และ A/B testing

## สิ่งที่จะได้เรียนรู้

- Data Collection Architecture
- Metrics ที่ควรติดตาม
- Time-series Database
- Real-time Dashboard ด้วย Express + Chart.js
- Socket.io สำหรับ live data
- Automated Reports ผ่าน LINE Broadcast
- A/B Test Analysis
- User Retention Analysis
- Revenue per User

---

## 1. Database Schema (Analytics)

```sql
CREATE DATABASE IF NOT EXISTS lineoa_analytics CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE lineoa_analytics;

-- Events raw data
CREATE TABLE events (
  id BIGINT PRIMARY KEY AUTO_INCREMENT,
  session_id VARCHAR(100),
  user_id VARCHAR(100) NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  event_name VARCHAR(200),
  properties JSON,
  source VARCHAR(50) DEFAULT 'line',
  platform VARCHAR(50),
  created_at TIMESTAMP(3) DEFAULT CURRENT_TIMESTAMP(3),
  INDEX idx_user_id (user_id),
  INDEX idx_event_type (event_type),
  INDEX idx_created_at (created_at),
  INDEX idx_source (source)
) PARTITION BY RANGE (UNIX_TIMESTAMP(created_at)) (
  PARTITION p2024 VALUES LESS THAN (UNIX_TIMESTAMP('2025-01-01')),
  PARTITION p2025 VALUES LESS THAN (UNIX_TIMESTAMP('2026-01-01')),
  PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Daily Metrics Aggregation
CREATE TABLE daily_metrics (
  id INT PRIMARY KEY AUTO_INCREMENT,
  date DATE NOT NULL,
  metric_name VARCHAR(100) NOT NULL,
  dimension VARCHAR(100),
  dimension_value VARCHAR(200),
  value DECIMAL(20,4) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  UNIQUE KEY unique_metric (date, metric_name, dimension, dimension_value),
  INDEX idx_date (date),
  INDEX idx_metric_name (metric_name)
);

-- User Sessions
CREATE TABLE sessions (
  id VARCHAR(100) PRIMARY KEY,
  user_id VARCHAR(100) NOT NULL,
  started_at TIMESTAMP NOT NULL,
  ended_at TIMESTAMP,
  duration_seconds INT,
  event_count INT DEFAULT 0,
  entry_event VARCHAR(100),
  exit_event VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_user_id (user_id),
  INDEX idx_started_at (started_at)
);

-- User Profiles (denormalized for analytics)
CREATE TABLE user_profiles (
  user_id VARCHAR(100) PRIMARY KEY,
  first_seen_at TIMESTAMP,
  last_seen_at TIMESTAMP,
  total_sessions INT DEFAULT 0,
  total_messages INT DEFAULT 0,
  total_purchases INT DEFAULT 0,
  total_revenue DECIMAL(15,2) DEFAULT 0,
  lifecycle_stage VARCHAR(50) DEFAULT 'new',
  cohort_month VARCHAR(7),
  properties JSON,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_first_seen (first_seen_at),
  INDEX idx_cohort (cohort_month),
  INDEX idx_lifecycle (lifecycle_stage)
);

-- Funnels Definition
CREATE TABLE funnels (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  steps JSON NOT NULL,
  description TEXT,
  is_active TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- A/B Test Variants
CREATE TABLE ab_tests (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  description TEXT,
  status ENUM('draft','running','paused','completed') DEFAULT 'draft',
  start_date TIMESTAMP,
  end_date TIMESTAMP,
  metric_goal VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE ab_variants (
  id INT PRIMARY KEY AUTO_INCREMENT,
  test_id INT NOT NULL,
  name VARCHAR(100) NOT NULL,
  description TEXT,
  weight INT DEFAULT 50,
  content JSON,
  FOREIGN KEY (test_id) REFERENCES ab_tests(id) ON DELETE CASCADE
);

CREATE TABLE ab_assignments (
  user_id VARCHAR(100) NOT NULL,
  test_id INT NOT NULL,
  variant_id INT NOT NULL,
  assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (user_id, test_id),
  FOREIGN KEY (test_id) REFERENCES ab_tests(id)
);

CREATE TABLE ab_conversions (
  id INT PRIMARY KEY AUTO_INCREMENT,
  user_id VARCHAR(100) NOT NULL,
  test_id INT NOT NULL,
  variant_id INT NOT NULL,
  converted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  value DECIMAL(15,2),
  FOREIGN KEY (test_id) REFERENCES ab_tests(id)
);

-- Report Schedules
CREATE TABLE report_schedules (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  type ENUM('daily','weekly','monthly') NOT NULL,
  recipients JSON,
  report_config JSON,
  last_sent_at TIMESTAMP,
  next_send_at TIMESTAMP,
  is_active TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Broadcast Analytics
CREATE TABLE broadcasts (
  id INT PRIMARY KEY AUTO_INCREMENT,
  message_id VARCHAR(200),
  title VARCHAR(300),
  sent_count INT DEFAULT 0,
  delivered_count INT DEFAULT 0,
  opened_count INT DEFAULT 0,
  clicked_count INT DEFAULT 0,
  revenue_generated DECIMAL(15,2) DEFAULT 0,
  sent_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

## 2. Analytics Event Tracker

```javascript
// src/services/analyticsService.js
const db = require('../config/database');
const redis = require('../config/redis');

class AnalyticsService {
  // Track event
  async track(userId, eventType, eventName, properties = {}) {
    const sessionId = await this.getOrCreateSession(userId, eventType);

    try {
      await db.query(
        `INSERT INTO events (session_id, user_id, event_type, event_name, properties)
         VALUES (?, ?, ?, ?, ?)`,
        [sessionId, userId, eventType, eventName, JSON.stringify(properties)]
      );

      // อัปเดต session event count
      await db.query(
        'UPDATE sessions SET event_count = event_count + 1, exit_event = ? WHERE id = ?',
        [eventType, sessionId]
      );

      // อัปเดต user profile
      await this.updateUserProfile(userId, eventType, properties);

      // Buffer สำหรับ real-time aggregation
      await this.bufferMetric(userId, eventType, properties);
    } catch (error) {
      console.error('Analytics track error:', error.message);
    }
  }

  // Session management
  async getOrCreateSession(userId, eventType) {
    const sessionKey = `session:${userId}`;
    let sessionId = await redis.get(sessionKey);

    if (!sessionId) {
      sessionId = `${userId}_${Date.now()}`;
      await redis.setex(sessionKey, 1800, sessionId); // 30 นาที

      await db.query(
        `INSERT INTO sessions (id, user_id, started_at, entry_event)
         VALUES (?, ?, NOW(), ?)
         ON DUPLICATE KEY UPDATE started_at = started_at`,
        [sessionId, userId, eventType]
      );
    } else {
      // ต่ออายุ session
      await redis.expire(sessionKey, 1800);
    }

    return sessionId;
  }

  // อัปเดต user profile
  async updateUserProfile(userId, eventType, properties = {}) {
    const now = new Date().toISOString().slice(0, 10);
    const cohortMonth = now.slice(0, 7);

    await db.query(
      `INSERT INTO user_profiles (user_id, first_seen_at, last_seen_at, cohort_month)
       VALUES (?, NOW(), NOW(), ?)
       ON DUPLICATE KEY UPDATE last_seen_at = NOW()`,
      [userId, cohortMonth]
    );

    if (eventType === 'message') {
      await db.query(
        'UPDATE user_profiles SET total_messages = total_messages + 1 WHERE user_id = ?',
        [userId]
      );
    }

    if (eventType === 'purchase') {
      const amount = parseFloat(properties.amount || 0);
      await db.query(
        `UPDATE user_profiles 
         SET total_purchases = total_purchases + 1, total_revenue = total_revenue + ?
         WHERE user_id = ?`,
        [amount, userId]
      );
    }
  }

  // Buffer metric สำหรับ real-time
  async bufferMetric(userId, eventType, properties) {
    const key = `metrics:buffer:${new Date().toISOString().slice(0, 13)}`; // per hour
    const field = eventType;
    await redis.hincrby(key, field, 1);
    await redis.expire(key, 7200); // 2 ชั่วโมง
  }

  // ดึง metrics รวม
  async getMetrics(startDate, endDate) {
    const [rows] = await db.query(
      `SELECT
        DATE(created_at) as date,
        COUNT(DISTINCT user_id) as dau,
        COUNT(*) as events,
        SUM(event_type = 'message') as messages,
        SUM(event_type = 'purchase') as purchases,
        SUM(event_type = 'follow') as new_follows,
        SUM(event_type = 'unfollow') as unfollows
       FROM events
       WHERE DATE(created_at) BETWEEN ? AND ?
       GROUP BY DATE(created_at)
       ORDER BY date`,
      [startDate, endDate]
    );

    return rows;
  }

  // Message Volume
  async getMessageVolume(days = 30) {
    const [rows] = await db.query(
      `SELECT
        DATE(created_at) as date,
        HOUR(created_at) as hour,
        COUNT(*) as count
       FROM events
       WHERE event_type = 'message'
         AND created_at >= DATE_SUB(NOW(), INTERVAL ? DAY)
       GROUP BY DATE(created_at), HOUR(created_at)
       ORDER BY date, hour`,
      [days]
    );
    return rows;
  }

  // Response Time Analysis
  async getResponseTimeStats(days = 7) {
    // จำลอง response time จาก event pairs
    const [stats] = await db.query(
      `SELECT
        AVG(response_time) as avg_ms,
        MIN(response_time) as min_ms,
        MAX(response_time) as max_ms,
        PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY response_time) as p50_ms,
        PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY response_time) as p95_ms
       FROM (
         SELECT
           TIMESTAMPDIFF(MICROSECOND,
             LAG(created_at) OVER (PARTITION BY user_id ORDER BY created_at),
             created_at) / 1000 as response_time
         FROM events
         WHERE event_type IN ('message', 'reply')
           AND created_at >= DATE_SUB(NOW(), INTERVAL ? DAY)
       ) t
       WHERE response_time BETWEEN 100 AND 30000`,
      [days]
    );

    return stats[0] || { avg_ms: 0, min_ms: 0, max_ms: 0, p50_ms: 0, p95_ms: 0 };
  }

  // Retention Analysis (Day N retention)
  async getRetentionData(cohortMonth) {
    const [retention] = await db.query(
      `SELECT
        u.cohort_month,
        DATEDIFF(DATE(e.created_at), u.first_seen_at) as days_since_first,
        COUNT(DISTINCT e.user_id) as retained_users,
        cohort_size.total as cohort_size,
        ROUND(COUNT(DISTINCT e.user_id) / cohort_size.total * 100, 1) as retention_rate
       FROM user_profiles u
       JOIN events e ON u.user_id = e.user_id
       JOIN (
         SELECT cohort_month, COUNT(*) as total
         FROM user_profiles
         GROUP BY cohort_month
       ) cohort_size ON u.cohort_month = cohort_size.cohort_month
       WHERE u.cohort_month = ?
         AND DATEDIFF(DATE(e.created_at), u.first_seen_at) IN (0, 1, 3, 7, 14, 30)
       GROUP BY days_since_first
       ORDER BY days_since_first`,
      [cohortMonth]
    );

    return retention;
  }

  // Conversion Rate
  async getConversionRates(days = 30) {
    const [[totals]] = await db.query(
      `SELECT
        COUNT(DISTINCT user_id) as total_users,
        COUNT(DISTINCT CASE WHEN event_type = 'view_product' THEN user_id END) as viewed_product,
        COUNT(DISTINCT CASE WHEN event_type = 'add_to_cart' THEN user_id END) as added_to_cart,
        COUNT(DISTINCT CASE WHEN event_type = 'checkout_start' THEN user_id END) as started_checkout,
        COUNT(DISTINCT CASE WHEN event_type = 'purchase' THEN user_id END) as purchased
       FROM events
       WHERE created_at >= DATE_SUB(NOW(), INTERVAL ? DAY)`,
      [days]
    );

    const { total_users, viewed_product, added_to_cart, started_checkout, purchased } = totals;

    return {
      totalUsers: total_users,
      funnelSteps: [
        { step: 'Active Users', users: total_users, rate: 100 },
        { step: 'Viewed Product', users: viewed_product, rate: total_users > 0 ? (viewed_product / total_users * 100).toFixed(1) : 0 },
        { step: 'Added to Cart', users: added_to_cart, rate: viewed_product > 0 ? (added_to_cart / viewed_product * 100).toFixed(1) : 0 },
        { step: 'Started Checkout', users: started_checkout, rate: added_to_cart > 0 ? (started_checkout / added_to_cart * 100).toFixed(1) : 0 },
        { step: 'Purchased', users: purchased, rate: started_checkout > 0 ? (purchased / started_checkout * 100).toFixed(1) : 0 }
      ]
    };
  }

  // Revenue Per User
  async getRevenueMetrics(days = 30) {
    const [[revenue]] = await db.query(
      `SELECT
        COUNT(DISTINCT e.user_id) as paying_users,
        SUM(CAST(JSON_EXTRACT(e.properties, '$.amount') AS DECIMAL(15,2))) as total_revenue,
        AVG(CAST(JSON_EXTRACT(e.properties, '$.amount') AS DECIMAL(15,2))) as avg_order_value,
        SUM(CAST(JSON_EXTRACT(e.properties, '$.amount') AS DECIMAL(15,2))) / 
          COUNT(DISTINCT e.user_id) as revenue_per_user
       FROM events e
       WHERE e.event_type = 'purchase'
         AND e.created_at >= DATE_SUB(NOW(), INTERVAL ? DAY)`,
      [days]
    );

    const [[allUsers]] = await db.query(
      `SELECT COUNT(DISTINCT user_id) as total
       FROM events
       WHERE created_at >= DATE_SUB(NOW(), INTERVAL ? DAY)`,
      [days]
    );

    return {
      ...revenue,
      total_users: allUsers.total,
      arpu: allUsers.total > 0 ? (parseFloat(revenue.total_revenue || 0) / allUsers.total).toFixed(2) : 0
    };
  }

  // Real-time metrics จาก Redis buffer
  async getRealTimeMetrics() {
    const now = new Date();
    const keys = [];

    // ดึง 6 ชั่วโมงล่าสุด
    for (let i = 0; i < 6; i++) {
      const d = new Date(now.getTime() - i * 3600000);
      keys.push(`metrics:buffer:${d.toISOString().slice(0, 13)}`);
    }

    const results = [];
    for (const key of keys) {
      const data = await redis.hgetall(key);
      if (data) {
        const hour = key.split(':')[2];
        results.unshift({ hour, ...data });
      }
    }

    return results;
  }

  // Aggregate daily metrics (สำหรับ cron job)
  async aggregateDailyMetrics(date) {
    const [events] = await db.query(
      `SELECT event_type, COUNT(*) as count, COUNT(DISTINCT user_id) as unique_users
       FROM events WHERE DATE(created_at) = ?
       GROUP BY event_type`,
      [date]
    );

    for (const row of events) {
      await db.query(
        `INSERT INTO daily_metrics (date, metric_name, dimension, dimension_value, value)
         VALUES (?, 'event_count', 'event_type', ?, ?)
         ON DUPLICATE KEY UPDATE value = ?`,
        [date, row.event_type, row.count, row.count]
      );

      await db.query(
        `INSERT INTO daily_metrics (date, metric_name, dimension, dimension_value, value)
         VALUES (?, 'unique_users', 'event_type', ?, ?)
         ON DUPLICATE KEY UPDATE value = ?`,
        [date, row.event_type, row.unique_users, row.unique_users]
      );
    }

    // DAU
    const [[dau]] = await db.query(
      'SELECT COUNT(DISTINCT user_id) as count FROM events WHERE DATE(created_at) = ?',
      [date]
    );

    await db.query(
      `INSERT INTO daily_metrics (date, metric_name, dimension, dimension_value, value)
       VALUES (?, 'dau', 'all', 'all', ?)
       ON DUPLICATE KEY UPDATE value = ?`,
      [date, dau.count, dau.count]
    );

    console.log(`Aggregated metrics for ${date}`);
  }
}

module.exports = new AnalyticsService();
```

## 3. A/B Testing Service

```javascript
// src/services/abTestService.js
const db = require('../config/database');
const redis = require('../config/redis');

class ABTestService {
  // กำหนด variant ให้ user
  async assignVariant(userId, testId) {
    // ตรวจสอบว่า assign ไปแล้ว
    const cacheKey = `ab:${testId}:${userId}`;
    const cached = await redis.get(cacheKey);
    if (cached) return parseInt(cached);

    const [[existing]] = await db.query(
      'SELECT variant_id FROM ab_assignments WHERE user_id = ? AND test_id = ?',
      [userId, testId]
    );

    if (existing) {
      await redis.setex(cacheKey, 3600, String(existing.variant_id));
      return existing.variant_id;
    }

    // ดึง variants และ weight
    const [variants] = await db.query(
      'SELECT * FROM ab_variants WHERE test_id = ?',
      [testId]
    );

    if (!variants.length) return null;

    // Weighted random assignment
    const totalWeight = variants.reduce((sum, v) => sum + v.weight, 0);
    let random = Math.random() * totalWeight;
    let selectedVariant = variants[0];

    for (const variant of variants) {
      random -= variant.weight;
      if (random <= 0) {
        selectedVariant = variant;
        break;
      }
    }

    await db.query(
      'INSERT INTO ab_assignments (user_id, test_id, variant_id) VALUES (?, ?, ?)',
      [userId, testId, selectedVariant.id]
    );

    await redis.setex(cacheKey, 3600, String(selectedVariant.id));
    return selectedVariant.id;
  }

  // บันทึก conversion
  async trackConversion(userId, testId, value = null) {
    const [[assignment]] = await db.query(
      'SELECT * FROM ab_assignments WHERE user_id = ? AND test_id = ?',
      [userId, testId]
    );

    if (!assignment) return;

    // ตรวจสอบว่า convert ไปแล้ว
    const [[existing]] = await db.query(
      'SELECT id FROM ab_conversions WHERE user_id = ? AND test_id = ?',
      [userId, testId]
    );

    if (!existing) {
      await db.query(
        'INSERT INTO ab_conversions (user_id, test_id, variant_id, value) VALUES (?, ?, ?, ?)',
        [userId, testId, assignment.variant_id, value]
      );
    }
  }

  // วิเคราะห์ผลการทดสอบ
  async analyzeTest(testId) {
    const [[test]] = await db.query('SELECT * FROM ab_tests WHERE id = ?', [testId]);
    if (!test) return null;

    const [variants] = await db.query('SELECT * FROM ab_variants WHERE test_id = ?', [testId]);

    const results = await Promise.all(variants.map(async variant => {
      const [[stats]] = await db.query(
        `SELECT
          COUNT(DISTINCT aa.user_id) as assigned,
          COUNT(DISTINCT ac.user_id) as converted,
          COALESCE(SUM(ac.value), 0) as total_value,
          COALESCE(AVG(ac.value), 0) as avg_value
         FROM ab_assignments aa
         LEFT JOIN ab_conversions ac ON aa.user_id = ac.user_id AND aa.test_id = ac.test_id
         WHERE aa.test_id = ? AND aa.variant_id = ?`,
        [testId, variant.id]
      );

      const conversionRate = stats.assigned > 0
        ? (stats.converted / stats.assigned * 100).toFixed(2)
        : 0;

      return {
        variant,
        assigned: stats.assigned,
        converted: stats.converted,
        conversionRate: parseFloat(conversionRate),
        totalValue: parseFloat(stats.total_value),
        avgValue: parseFloat(stats.avg_value)
      };
    }));

    // Statistical significance (Chi-square test approximation)
    const significance = this.calculateSignificance(results);

    return { test, variants: results, significance };
  }

  calculateSignificance(results) {
    if (results.length < 2) return null;

    const [control, ...treatments] = results;
    const totalAssigned = results.reduce((sum, r) => sum + r.assigned, 0);

    return treatments.map(treatment => {
      const diff = treatment.conversionRate - control.conversionRate;
      const relativeDiff = control.conversionRate > 0
        ? (diff / control.conversionRate * 100).toFixed(1)
        : 0;

      // Simplified p-value calculation
      const p1 = control.converted / control.assigned;
      const p2 = treatment.converted / treatment.assigned;
      const pooledP = (control.converted + treatment.converted) / (control.assigned + treatment.assigned);
      const se = Math.sqrt(pooledP * (1 - pooledP) * (1/control.assigned + 1/treatment.assigned));
      const zScore = se > 0 ? Math.abs(p2 - p1) / se : 0;

      return {
        variant: treatment.variant.name,
        absoluteDiff: diff.toFixed(2),
        relativeDiff: `${relativeDiff}%`,
        zScore: zScore.toFixed(2),
        isSignificant: zScore > 1.96 // 95% confidence
      };
    });
  }

  // ดึง content สำหรับ variant
  async getVariantContent(testId, variantId) {
    const [[variant]] = await db.query(
      'SELECT * FROM ab_variants WHERE test_id = ? AND id = ?',
      [testId, variantId]
    );
    return variant ? JSON.parse(variant.content || '{}') : null;
  }
}

module.exports = new ABTestService();
```

## 4. Report Service

```javascript
// src/services/reportService.js
const db = require('../config/database');
const analyticsService = require('./analyticsService');
const line = require('@line/bot-sdk');

const lineClient = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

class ReportService {
  // สร้าง Daily Report
  async generateDailyReport(date = null) {
    const reportDate = date || new Date().toISOString().split('T')[0];
    const yesterday = new Date(new Date(reportDate).getTime() - 86400000).toISOString().split('T')[0];

    const [metrics] = await db.query(
      `SELECT metric_name, dimension_value, value
       FROM daily_metrics
       WHERE date = ?`,
      [reportDate]
    );

    const [prevMetrics] = await db.query(
      `SELECT metric_name, dimension_value, value
       FROM daily_metrics
       WHERE date = ?`,
      [yesterday]
    );

    const getMetric = (name, source, dimVal = 'all') => {
      const m = metrics.find(m => m.metric_name === name && m.dimension_value === dimVal);
      return m ? parseFloat(m.value) : 0;
    };

    const getPrevMetric = (name, dimVal = 'all') => {
      const m = prevMetrics.find(m => m.metric_name === name && m.dimension_value === dimVal);
      return m ? parseFloat(m.value) : 0;
    };

    const dau = getMetric('dau', 'all', 'all');
    const prevDau = getPrevMetric('dau', 'all');
    const dauChange = prevDau > 0 ? ((dau - prevDau) / prevDau * 100).toFixed(1) : 0;

    const messages = getMetric('event_count', null, 'message');
    const purchases = getMetric('event_count', null, 'purchase');
    const follows = getMetric('event_count', null, 'follow');

    // Revenue for the day
    const [[revenueData]] = await db.query(
      `SELECT COALESCE(SUM(CAST(JSON_EXTRACT(properties, '$.amount') AS DECIMAL(15,2))), 0) as total
       FROM events
       WHERE DATE(created_at) = ? AND event_type = 'purchase'`,
      [reportDate]
    );

    return {
      date: reportDate,
      dau,
      dauChange,
      messages,
      purchases,
      follows,
      revenue: parseFloat(revenueData.total || 0)
    };
  }

  // สร้าง Weekly Report
  async generateWeeklyReport() {
    const endDate = new Date().toISOString().split('T')[0];
    const startDate = new Date(Date.now() - 7 * 86400000).toISOString().split('T')[0];

    const metrics = await analyticsService.getMetrics(startDate, endDate);
    const conversionData = await analyticsService.getConversionRates(7);
    const revenueData = await analyticsService.getRevenueMetrics(7);

    const totalDau = metrics.reduce((sum, m) => sum + parseInt(m.dau), 0);
    const avgDau = metrics.length > 0 ? (totalDau / metrics.length).toFixed(0) : 0;
    const totalMessages = metrics.reduce((sum, m) => sum + parseInt(m.messages || 0), 0);
    const totalPurchases = metrics.reduce((sum, m) => sum + parseInt(m.purchases || 0), 0);

    return {
      period: `${startDate} - ${endDate}`,
      avgDau,
      totalMessages,
      totalPurchases,
      totalRevenue: parseFloat(revenueData.total_revenue || 0),
      arpu: revenueData.arpu,
      conversionRate: conversionData.funnelSteps[conversionData.funnelSteps.length - 1]?.rate || 0,
      dailyBreakdown: metrics
    };
  }

  // ส่ง Daily Report ผ่าน LINE Broadcast
  async sendDailyReportToLine(lineGroupIds = []) {
    const report = await this.generateDailyReport();

    const changeIcon = parseFloat(report.dauChange) >= 0 ? '📈' : '📉';

    const message = {
      type: 'flex',
      altText: `📊 รายงานประจำวัน ${report.date}`,
      contents: {
        type: 'bubble',
        header: {
          type: 'box',
          layout: 'vertical',
          contents: [
            { type: 'text', text: '📊 รายงานประจำวัน', weight: 'bold', color: '#FFFFFF', size: 'lg' },
            { type: 'text', text: report.date, color: '#FFFFFF', size: 'sm', opacity: 0.8 }
          ],
          backgroundColor: '#2C3E50'
        },
        body: {
          type: 'box',
          layout: 'vertical',
          spacing: 'md',
          contents: [
            createStatRow('👥 DAU', `${report.dau.toLocaleString()} คน`, `${changeIcon} ${report.dauChange}%`),
            createStatRow('💬 ข้อความ', `${report.messages.toLocaleString()} ครั้ง`),
            createStatRow('🛒 คำสั่งซื้อ', `${report.purchases.toLocaleString()} ออเดอร์`),
            createStatRow('💰 รายได้', `฿${report.revenue.toLocaleString('th-TH', { minimumFractionDigits: 2 })}`),
            createStatRow('➕ Follows', `${report.follows.toLocaleString()} คน`)
          ]
        },
        footer: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'button',
            action: {
              type: 'uri',
              label: '📈 ดู Dashboard',
              uri: process.env.DASHBOARD_URL || '#'
            },
            style: 'primary',
            color: '#2C3E50'
          }]
        }
      }
    };

    // ส่งให้ผู้รับทุกคน
    for (const groupId of lineGroupIds) {
      try {
        await lineClient.pushMessage(groupId, message);
      } catch (err) {
        console.error(`Report send error to ${groupId}:`, err.message);
      }
    }

    return report;
  }

  // ส่ง Weekly Report ผ่าน LINE
  async sendWeeklyReport(lineGroupIds = []) {
    const report = await this.generateWeeklyReport();

    const message = {
      type: 'flex',
      altText: `📊 รายงานประจำสัปดาห์`,
      contents: {
        type: 'bubble',
        size: 'mega',
        header: {
          type: 'box', layout: 'vertical',
          contents: [
            { type: 'text', text: '📊 รายงานประจำสัปดาห์', weight: 'bold', color: '#FFFFFF', size: 'lg' },
            { type: 'text', text: report.period, color: '#FFFFFF', size: 'xs' }
          ],
          backgroundColor: '#27AE60'
        },
        body: {
          type: 'box', layout: 'vertical', spacing: 'md',
          contents: [
            createStatRow('👥 Avg DAU', `${parseInt(report.avgDau).toLocaleString()} คน/วัน`),
            createStatRow('💬 ข้อความรวม', `${report.totalMessages.toLocaleString()} ครั้ง`),
            createStatRow('🛒 คำสั่งซื้อรวม', `${report.totalPurchases.toLocaleString()} ออเดอร์`),
            createStatRow('💰 รายได้รวม', `฿${report.totalRevenue.toLocaleString('th-TH', { minimumFractionDigits: 2 })}`),
            createStatRow('💎 ARPU', `฿${report.arpu}`),
            createStatRow('🎯 Conversion Rate', `${report.conversionRate}%`)
          ]
        }
      }
    };

    for (const groupId of lineGroupIds) {
      try {
        await lineClient.pushMessage(groupId, message);
      } catch (err) {
        console.error(`Weekly report error:`, err.message);
      }
    }

    return report;
  }
}

function createStatRow(label, value, subValue = null) {
  const contents = [
    { type: 'text', text: label, size: 'sm', color: '#666666', flex: 3 },
    {
      type: 'box', layout: 'vertical', flex: 3,
      contents: [
        { type: 'text', text: value, size: 'sm', weight: 'bold', align: 'end' },
        subValue ? { type: 'text', text: subValue, size: 'xs', color: '#999999', align: 'end' } : null
      ].filter(Boolean)
    }
  ];

  return { type: 'box', layout: 'horizontal', contents };
}

module.exports = new ReportService();
```

## 5. Dashboard Server (Express + Socket.io)

```javascript
// src/dashboard/server.js
const express = require('express');
const http = require('http');
const socketIo = require('socket.io');
const path = require('path');
const analyticsService = require('../services/analyticsService');
const abTestService = require('../services/abTestService');
const reportService = require('../services/reportService');
const db = require('../config/database');

const app = express();
const server = http.createServer(app);
const io = socketIo(server, {
  cors: { origin: '*' }
});

app.use(express.static(path.join(__dirname, 'public')));
app.use(express.json());

// Auth middleware
function dashboardAuth(req, res, next) {
  const token = req.headers['x-dashboard-token'] || req.query.token;
  if (token !== process.env.DASHBOARD_TOKEN) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
}

app.use('/api', dashboardAuth);

// API Endpoints

// Overview stats
app.get('/api/overview', async (req, res) => {
  const { days = 30 } = req.query;

  const endDate = new Date().toISOString().split('T')[0];
  const startDate = new Date(Date.now() - parseInt(days) * 86400000).toISOString().split('T')[0];

  const [dailyMetrics] = await Promise.all([
    analyticsService.getMetrics(startDate, endDate)
  ]);

  const conversionRates = await analyticsService.getConversionRates(parseInt(days));
  const revenueMetrics = await analyticsService.getRevenueMetrics(parseInt(days));

  const totalDAU = dailyMetrics.reduce((sum, d) => sum + parseInt(d.dau || 0), 0);
  const avgDAU = dailyMetrics.length > 0 ? Math.round(totalDAU / dailyMetrics.length) : 0;

  res.json({
    avgDAU,
    totalMessages: dailyMetrics.reduce((sum, d) => sum + parseInt(d.messages || 0), 0),
    totalPurchases: dailyMetrics.reduce((sum, d) => sum + parseInt(d.purchases || 0), 0),
    totalRevenue: parseFloat(revenueMetrics.total_revenue || 0),
    arpu: revenueMetrics.arpu,
    conversionRates,
    dailyMetrics
  });
});

// Message Volume
app.get('/api/message-volume', async (req, res) => {
  const data = await analyticsService.getMessageVolume(parseInt(req.query.days || 7));
  res.json({ data });
});

// Retention
app.get('/api/retention', async (req, res) => {
  const cohortMonth = req.query.cohort || new Date().toISOString().slice(0, 7);
  const data = await analyticsService.getRetentionData(cohortMonth);
  res.json({ data });
});

// A/B Tests
app.get('/api/ab-tests', async (req, res) => {
  const [tests] = await db.query('SELECT * FROM ab_tests ORDER BY created_at DESC LIMIT 20');
  res.json({ tests });
});

app.get('/api/ab-tests/:id/results', async (req, res) => {
  const results = await abTestService.analyzeTest(parseInt(req.params.id));
  res.json(results);
});

// Real-time metrics
app.get('/api/realtime', async (req, res) => {
  const data = await analyticsService.getRealTimeMetrics();
  res.json({ data });
});

// Reports
app.get('/api/reports/daily', async (req, res) => {
  const report = await reportService.generateDailyReport(req.query.date);
  res.json(report);
});

app.get('/api/reports/weekly', async (req, res) => {
  const report = await reportService.generateWeeklyReport();
  res.json(report);
});

// Top users
app.get('/api/top-users', async (req, res) => {
  const [users] = await db.query(
    `SELECT user_id, total_purchases, total_revenue, total_messages, lifecycle_stage
     FROM user_profiles
     ORDER BY total_revenue DESC LIMIT 20`
  );
  res.json({ users });
});

// Broadcast analytics
app.get('/api/broadcasts', async (req, res) => {
  const [broadcasts] = await db.query(
    'SELECT * FROM broadcasts ORDER BY sent_at DESC LIMIT 20'
  );
  res.json({ broadcasts });
});

// Real-time Socket.io
io.on('connection', (socket) => {
  console.log('Dashboard connected:', socket.id);

  // ส่ง real-time metrics ทุก 30 วินาที
  const interval = setInterval(async () => {
    try {
      const realtime = await analyticsService.getRealTimeMetrics();

      // Active users ในชั่วโมงนี้
      const [[activeUsers]] = await db.query(
        `SELECT COUNT(DISTINCT user_id) as count
         FROM events WHERE created_at >= DATE_SUB(NOW(), INTERVAL 5 MINUTE)`
      );

      socket.emit('realtime_update', {
        activeUsers: activeUsers.count,
        realtimeMetrics: realtime,
        timestamp: new Date().toISOString()
      });
    } catch (err) {
      console.error('Socket emit error:', err.message);
    }
  }, 30000);

  socket.on('disconnect', () => {
    clearInterval(interval);
    console.log('Dashboard disconnected:', socket.id);
  });
});

module.exports = { app, server, io };
```

## 6. Dashboard Frontend (HTML + Chart.js)

```html
<!-- src/dashboard/public/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>LINE Bot Analytics Dashboard</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
  <script src="https://cdn.socket.io/4.6.0/socket.io.min.js"></script>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Segoe UI', sans-serif; background: #0F172A; color: #E2E8F0; }

    .header {
      background: linear-gradient(135deg, #06C755, #00A651);
      padding: 20px 24px;
      display: flex;
      justify-content: space-between;
      align-items: center;
    }
    .header h1 { color: white; font-size: 1.5rem; }
    .live-badge {
      background: #EF4444;
      color: white;
      padding: 4px 12px;
      border-radius: 20px;
      font-size: 0.75rem;
      animation: pulse 2s infinite;
    }
    @keyframes pulse { 0%, 100% { opacity: 1; } 50% { opacity: 0.5; } }

    .container { padding: 24px; max-width: 1400px; margin: 0 auto; }

    .stats-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
      gap: 16px;
      margin-bottom: 24px;
    }

    .stat-card {
      background: #1E293B;
      border: 1px solid #334155;
      border-radius: 12px;
      padding: 20px;
    }
    .stat-card .label { font-size: 0.8rem; color: #94A3B8; margin-bottom: 8px; }
    .stat-card .value { font-size: 1.8rem; font-weight: bold; color: #F1F5F9; }
    .stat-card .change { font-size: 0.8rem; margin-top: 4px; }
    .change.positive { color: #22C55E; }
    .change.negative { color: #EF4444; }

    .charts-grid {
      display: grid;
      grid-template-columns: 2fr 1fr;
      gap: 24px;
      margin-bottom: 24px;
    }

    .chart-card {
      background: #1E293B;
      border: 1px solid #334155;
      border-radius: 12px;
      padding: 20px;
    }
    .chart-card h3 { font-size: 1rem; margin-bottom: 16px; color: #CBD5E1; }

    canvas { max-height: 300px; }

    .funnel-section, .retention-section, .ab-section {
      background: #1E293B;
      border: 1px solid #334155;
      border-radius: 12px;
      padding: 20px;
      margin-bottom: 24px;
    }

    .funnel-step {
      display: flex;
      align-items: center;
      margin-bottom: 12px;
      gap: 12px;
    }
    .funnel-bar-wrap { flex: 1; background: #334155; border-radius: 4px; height: 28px; overflow: hidden; }
    .funnel-bar { height: 100%; background: linear-gradient(90deg, #06C755, #00A651); border-radius: 4px; transition: width 0.8s; }
    .funnel-label { width: 140px; font-size: 0.85rem; color: #CBD5E1; }
    .funnel-count { width: 80px; text-align: right; font-size: 0.85rem; font-weight: bold; }
    .funnel-rate { width: 60px; text-align: right; font-size: 0.8rem; color: #94A3B8; }

    .realtime-counter {
      display: inline-block;
      background: #1E293B;
      border: 1px solid #06C755;
      border-radius: 8px;
      padding: 8px 16px;
      margin-bottom: 16px;
    }
    .realtime-counter .number { font-size: 1.5rem; font-weight: bold; color: #06C755; }
    .realtime-counter .desc { font-size: 0.75rem; color: #94A3B8; }

    @media (max-width: 768px) {
      .charts-grid { grid-template-columns: 1fr; }
      .stats-grid { grid-template-columns: repeat(2, 1fr); }
    }
  </style>
</head>
<body>
  <div class="header">
    <h1>📊 LINE Bot Analytics</h1>
    <div style="display:flex;align-items:center;gap:16px">
      <div class="realtime-counter">
        <div class="number" id="active-users">0</div>
        <div class="desc">ใช้งานตอนนี้</div>
      </div>
      <span class="live-badge">● LIVE</span>
    </div>
  </div>

  <div class="container">
    <!-- Stat Cards -->
    <div class="stats-grid">
      <div class="stat-card">
        <div class="label">👥 DAU วันนี้</div>
        <div class="value" id="stat-dau">-</div>
        <div class="change" id="stat-dau-change"></div>
      </div>
      <div class="stat-card">
        <div class="label">💬 ข้อความ</div>
        <div class="value" id="stat-messages">-</div>
      </div>
      <div class="stat-card">
        <div class="label">🛒 คำสั่งซื้อ</div>
        <div class="value" id="stat-purchases">-</div>
      </div>
      <div class="stat-card">
        <div class="label">💰 รายได้วันนี้</div>
        <div class="value" id="stat-revenue">-</div>
      </div>
      <div class="stat-card">
        <div class="label">💎 ARPU</div>
        <div class="value" id="stat-arpu">-</div>
      </div>
    </div>

    <!-- Charts -->
    <div class="charts-grid">
      <div class="chart-card">
        <h3>📈 DAU (30 วันล่าสุด)</h3>
        <canvas id="chart-dau"></canvas>
      </div>
      <div class="chart-card">
        <h3>💬 ปริมาณข้อความ (24 ชั่วโมง)</h3>
        <canvas id="chart-messages"></canvas>
      </div>
    </div>

    <!-- Conversion Funnel -->
    <div class="funnel-section">
      <h3 style="margin-bottom:16px;">🎯 Conversion Funnel (30 วันล่าสุด)</h3>
      <div id="funnel-steps"></div>
    </div>

    <!-- Revenue Chart -->
    <div class="chart-card" style="margin-bottom:24px">
      <h3>💰 รายได้รายวัน</h3>
      <canvas id="chart-revenue"></canvas>
    </div>

    <!-- Retention Heatmap -->
    <div class="retention-section">
      <h3 style="margin-bottom:16px;">🔄 User Retention</h3>
      <div id="retention-data"></div>
    </div>

    <!-- A/B Test Results -->
    <div class="ab-section">
      <h3 style="margin-bottom:16px;">🧪 A/B Tests</h3>
      <div id="ab-tests"></div>
    </div>
  </div>

  <script>
    const TOKEN = new URLSearchParams(location.search).get('token');
    const API = (path) => fetch(`/api${path}`, { headers: { 'x-dashboard-token': TOKEN } }).then(r => r.json());

    const chartColors = {
      green: 'rgba(6,199,85,0.8)',
      blue: 'rgba(59,130,246,0.8)',
      purple: 'rgba(168,85,247,0.8)',
      orange: 'rgba(249,115,22,0.8)',
      greenLight: 'rgba(6,199,85,0.2)'
    };

    // Load Overview
    async function loadOverview() {
      const data = await API('/overview?days=30');

      document.getElementById('stat-dau').textContent = data.avgDAU.toLocaleString();
      document.getElementById('stat-messages').textContent = data.totalMessages.toLocaleString();
      document.getElementById('stat-purchases').textContent = data.totalPurchases.toLocaleString();
      document.getElementById('stat-revenue').textContent = `฿${parseFloat(data.totalRevenue).toLocaleString('th-TH', {minimumFractionDigits:2})}`;
      document.getElementById('stat-arpu').textContent = `฿${data.arpu}`;

      // DAU Chart
      const labels = data.dailyMetrics.map(d => d.date.slice(5));
      const dauData = data.dailyMetrics.map(d => parseInt(d.dau));

      new Chart(document.getElementById('chart-dau'), {
        type: 'line',
        data: {
          labels,
          datasets: [{
            label: 'DAU',
            data: dauData,
            borderColor: chartColors.green,
            backgroundColor: chartColors.greenLight,
            fill: true,
            tension: 0.4
          }]
        },
        options: {
          responsive: true,
          plugins: { legend: { display: false } },
          scales: {
            x: { ticks: { color: '#94A3B8' }, grid: { color: '#334155' } },
            y: { ticks: { color: '#94A3B8' }, grid: { color: '#334155' } }
          }
        }
      });

      // Revenue Chart
      new Chart(document.getElementById('chart-revenue'), {
        type: 'bar',
        data: {
          labels,
          datasets: [{
            label: 'Revenue (฿)',
            data: data.dailyMetrics.map(d => parseInt(d.purchases) * 500),
            backgroundColor: chartColors.purple
          }]
        },
        options: {
          responsive: true,
          scales: {
            x: { ticks: { color: '#94A3B8' }, grid: { color: '#334155' } },
            y: { ticks: { color: '#94A3B8' }, grid: { color: '#334155' } }
          }
        }
      });

      // Conversion Funnel
      if (data.conversionRates?.funnelSteps) {
        const maxUsers = data.conversionRates.funnelSteps[0]?.users || 1;
        const funnelHTML = data.conversionRates.funnelSteps.map(step => `
          <div class="funnel-step">
            <div class="funnel-label">${step.step}</div>
            <div class="funnel-bar-wrap">
              <div class="funnel-bar" style="width:${step.users/maxUsers*100}%"></div>
            </div>
            <div class="funnel-count">${parseInt(step.users).toLocaleString()}</div>
            <div class="funnel-rate">${step.rate}%</div>
          </div>
        `).join('');
        document.getElementById('funnel-steps').innerHTML = funnelHTML;
      }
    }

    // Load Message Volume
    async function loadMessageVolume() {
      const { data } = await API('/message-volume?days=1');
      const labels = data.map(d => `${d.hour}:00`);
      const values = data.map(d => parseInt(d.count));

      new Chart(document.getElementById('chart-messages'), {
        type: 'bar',
        data: {
          labels,
          datasets: [{
            label: 'ข้อความ',
            data: values,
            backgroundColor: chartColors.blue
          }]
        },
        options: {
          responsive: true,
          plugins: { legend: { display: false } },
          scales: {
            x: { ticks: { color: '#94A3B8' }, grid: { color: '#334155' } },
            y: { ticks: { color: '#94A3B8' }, grid: { color: '#334155' } }
          }
        }
      });
    }

    // Load Retention
    async function loadRetention() {
      const cohortMonth = new Date().toISOString().slice(0, 7);
      const { data } = await API(`/retention?cohort=${cohortMonth}`);

      if (!data || data.length === 0) {
        document.getElementById('retention-data').innerHTML = '<p style="color:#94A3B8">ยังไม่มีข้อมูล Retention</p>';
        return;
      }

      const retentionHTML = data.map(row => `
        <div style="display:flex;align-items:center;gap:12px;margin-bottom:8px">
          <span style="width:80px;font-size:0.85rem;color:#94A3B8">Day ${row.days_since_first}</span>
          <div style="flex:1;background:#334155;border-radius:4px;height:24px;overflow:hidden">
            <div style="width:${row.retention_rate}%;height:100%;background:linear-gradient(90deg,#3B82F6,#06C755)"></div>
          </div>
          <span style="width:80px;text-align:right;font-size:0.85rem">${row.retention_rate}% (${row.retained_users})</span>
        </div>
      `).join('');

      document.getElementById('retention-data').innerHTML = retentionHTML;
    }

    // Load A/B Tests
    async function loadABTests() {
      const { tests } = await API('/ab-tests');

      if (!tests || tests.length === 0) {
        document.getElementById('ab-tests').innerHTML = '<p style="color:#94A3B8">ยังไม่มี A/B Tests</p>';
        return;
      }

      const abHTML = tests.map(test => `
        <div style="border:1px solid #334155;border-radius:8px;padding:12px;margin-bottom:12px">
          <div style="display:flex;justify-content:space-between;align-items:center">
            <strong>${test.name}</strong>
            <span style="padding:2px 8px;border-radius:12px;font-size:0.75rem;background:${test.status === 'running' ? '#22C55E33' : '#64748B33'};color:${test.status === 'running' ? '#22C55E' : '#94A3B8'}">${test.status}</span>
          </div>
          <p style="color:#94A3B8;font-size:0.8rem;margin-top:4px">${test.description || ''}</p>
          <a href="#" onclick="loadABTestResults(${test.id})" style="color:#3B82F6;font-size:0.8rem">ดูผลลัพธ์</a>
        </div>
      `).join('');

      document.getElementById('ab-tests').innerHTML = abHTML;
    }

    async function loadABTestResults(testId) {
      const data = await API(`/ab-tests/${testId}/results`);
      console.log('AB Test Results:', data);
      // แสดง modal หรือ alert
      alert(JSON.stringify(data?.variants?.map(v => ({
        name: v.variant.name,
        assigned: v.assigned,
        converted: v.converted,
        rate: v.conversionRate + '%'
      })), null, 2));
    }

    // Socket.io Real-time
    const socket = io('/', {
      auth: { token: TOKEN }
    });

    socket.on('realtime_update', (data) => {
      document.getElementById('active-users').textContent = data.activeUsers;
    });

    // Init
    (async () => {
      await Promise.all([
        loadOverview(),
        loadMessageVolume(),
        loadRetention(),
        loadABTests()
      ]);
    })();

    // Refresh ทุก 5 นาที
    setInterval(loadOverview, 300000);
  </script>
</body>
</html>
```

## 7. Cron Jobs

```javascript
// cron/analyticsJobs.js
const cron = require('node-cron');
const analyticsService = require('../src/services/analyticsService');
const reportService = require('../src/services/reportService');
const db = require('../src/config/database');

// Aggregate daily metrics ทุกคืน 00:05
cron.schedule('5 0 * * *', async () => {
  try {
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1);
    const dateStr = yesterday.toISOString().split('T')[0];

    await analyticsService.aggregateDailyMetrics(dateStr);
    console.log(`Aggregated metrics for ${dateStr}`);
  } catch (err) {
    console.error('Aggregation error:', err);
  }
}, { timezone: 'Asia/Bangkok' });

// Daily Report เวลา 08:00
cron.schedule('0 8 * * *', async () => {
  try {
    const [schedules] = await db.query(
      "SELECT * FROM report_schedules WHERE type = 'daily' AND is_active = 1"
    );

    for (const schedule of schedules) {
      const recipients = JSON.parse(schedule.recipients || '[]');
      await reportService.sendDailyReportToLine(recipients);
      await db.query(
        'UPDATE report_schedules SET last_sent_at = NOW() WHERE id = ?',
        [schedule.id]
      );
    }

    console.log('Daily reports sent');
  } catch (err) {
    console.error('Daily report error:', err);
  }
}, { timezone: 'Asia/Bangkok' });

// Weekly Report ทุกวันจันทร์ 08:30
cron.schedule('30 8 * * 1', async () => {
  try {
    const [schedules] = await db.query(
      "SELECT * FROM report_schedules WHERE type = 'weekly' AND is_active = 1"
    );

    for (const schedule of schedules) {
      const recipients = JSON.parse(schedule.recipients || '[]');
      await reportService.sendWeeklyReport(recipients);
    }

    console.log('Weekly reports sent');
  } catch (err) {
    console.error('Weekly report error:', err);
  }
}, { timezone: 'Asia/Bangkok' });

// Cleanup old events (เก็บไว้แค่ 90 วัน สำหรับ raw events)
cron.schedule('0 2 * * 0', async () => {
  try {
    const [result] = await db.query(
      'DELETE FROM events WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY)'
    );
    console.log(`Cleaned up ${result.affectedRows} old events`);
  } catch (err) {
    console.error('Cleanup error:', err);
  }
}, { timezone: 'Asia/Bangkok' });

// Mark overdue tasks (ทุกชั่วโมง)
cron.schedule('0 * * * *', async () => {
  await db.query(
    "UPDATE tasks SET status = 'overdue' WHERE status = 'pending' AND due_date < NOW()"
  );
});

console.log('Analytics cron jobs started');
```

## 8. index.js หลัก

```javascript
// index.js
require('dotenv').config();
const { server } = require('./src/dashboard/server');
const analyticsService = require('./src/services/analyticsService');
const line = require('@line/bot-sdk');
const express = require('express');

require('./cron/analyticsJobs');

const lineConfig = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};

const lineClient = new line.Client(lineConfig);
const app = require('./src/dashboard/server').app;

// LINE Webhook
app.post('/webhook', line.middleware(lineConfig), async (req, res) => {
  res.json({ ok: true });

  for (const event of req.body.events) {
    try {
      const userId = event.source.userId;

      if (event.type === 'follow') {
        await analyticsService.track(userId, 'follow', 'Follow LINE OA');
      } else if (event.type === 'unfollow') {
        await analyticsService.track(userId, 'unfollow', 'Unfollow LINE OA');
      } else if (event.type === 'message') {
        await analyticsService.track(userId, 'message', `Message: ${event.message.type}`, {
          message_type: event.message.type
        });
      } else if (event.type === 'postback') {
        const params = new URLSearchParams(event.postback.data);
        const action = params.get('action');
        await analyticsService.track(userId, 'postback', action, {
          postback_data: event.postback.data
        });
      }
    } catch (err) {
      console.error('Analytics event error:', err.message);
    }
  }
});

const PORT = process.env.PORT || 3000;
server.listen(PORT, () => {
  console.log(`Analytics Dashboard: http://localhost:${PORT}?token=${process.env.DASHBOARD_TOKEN}`);
});
```

## 9. package.json

```json
{
  "name": "lineoa-analytics",
  "version": "1.0.0",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js"
  },
  "dependencies": {
    "@line/bot-sdk": "^8.0.0",
    "dotenv": "^16.0.0",
    "express": "^4.18.0",
    "ioredis": "^5.3.0",
    "mysql2": "^3.6.0",
    "node-cron": "^3.0.0",
    "socket.io": "^4.6.0"
  }
}
```

## 10. .env ตัวอย่าง

```env
# LINE
LINE_CHANNEL_ACCESS_TOKEN=your_token
LINE_CHANNEL_SECRET=your_secret

# Database
DB_HOST=localhost
DB_USER=root
DB_PASSWORD=password
DB_NAME=lineoa_analytics

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379

# Dashboard
DASHBOARD_TOKEN=your_dashboard_secret_token
DASHBOARD_URL=https://your-domain.com/dashboard

# Admin
ADMIN_TOKEN=your_admin_token

PORT=3000
```

## 11. การทดสอบ

```javascript
// tests/analytics.test.js
const analyticsService = require('../src/services/analyticsService');
const abTestService = require('../src/services/abTestService');
const db = require('../src/config/database');

describe('Analytics Service', () => {
  const testUserId = 'U_test_analytics_001';

  test('track event', async () => {
    await analyticsService.track(testUserId, 'test_event', 'Test', { key: 'value' });

    const [[event]] = await db.query(
      'SELECT * FROM events WHERE user_id = ? AND event_type = ? ORDER BY id DESC LIMIT 1',
      [testUserId, 'test_event']
    );

    expect(event).toBeTruthy();
    expect(event.user_id).toBe(testUserId);
  });

  test('user profile upsert', async () => {
    await analyticsService.updateUserProfile(testUserId, 'follow');

    const [[profile]] = await db.query(
      'SELECT * FROM user_profiles WHERE user_id = ?',
      [testUserId]
    );

    expect(profile).toBeTruthy();
    expect(profile.user_id).toBe(testUserId);
  });

  afterAll(async () => {
    await db.query('DELETE FROM events WHERE user_id = ?', [testUserId]);
    await db.query('DELETE FROM user_profiles WHERE user_id = ?', [testUserId]);
    await db.end();
  });
});

describe('A/B Test Service', () => {
  test('assign variant', async () => {
    // สร้าง test และ variants ก่อน
    const [testResult] = await db.query(
      "INSERT INTO ab_tests (name, status) VALUES ('Test Variant', 'running')"
    );
    const testId = testResult.insertId;

    await db.query(
      'INSERT INTO ab_variants (test_id, name, weight) VALUES (?, ?, 50), (?, ?, 50)',
      [testId, 'Control', testId, 'Treatment']
    );

    const variantId = await abTestService.assignVariant('test_user_ab', testId);
    expect(variantId).toBeTruthy();

    // ตรวจสอบ consistent assignment
    const variantId2 = await abTestService.assignVariant('test_user_ab', testId);
    expect(variantId).toBe(variantId2);

    // Cleanup
    await db.query('DELETE FROM ab_tests WHERE id = ?', [testId]);
  });
});
```

## สรุป

ระบบ Analytics Dashboard สำหรับ LINE Bot ที่สร้างในบทนี้ครอบคลุม:

1. **Event Tracking** - บันทึก events ทุกการกระทำของผู้ใช้
2. **Session Management** - ติดตาม sessions ด้วย Redis
3. **Daily Aggregation** - รวมข้อมูลรายวันด้วย Cron Job
4. **Real-time Dashboard** - Dashboard พร้อม Socket.io
5. **Chart.js Visualizations** - กราฟ DAU, Revenue, Funnel
6. **Conversion Funnel** - วิเคราะห์ funnel แต่ละขั้น
7. **User Retention** - Cohort retention analysis
8. **A/B Testing** - สร้างและวิเคราะห์ A/B tests
9. **Automated Reports** - Daily/Weekly reports ผ่าน LINE
10. **Revenue Metrics** - ARPU, revenue tracking
