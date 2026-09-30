# Part 78: Monitoring & Alerting สำหรับ LINE Bot Production

## บทนำ

เมื่อ LINE Bot ของคุณ deploy ขึ้น production แล้ว การ monitoring เป็นสิ่งสำคัญอย่างยิ่ง คุณต้องรู้ทันทีเมื่อ:
- Bot ไม่ตอบสนอง
- Error rate สูงขึ้นผิดปกติ
- API quota ใกล้หมด
- Response time ช้าเกินไป
- มีการโจมตีหรือ abuse

```
Monitoring Architecture:
┌─────────────────────────────────────────────────────────────────┐
│                     LINE Bot Production                          │
│                                                                  │
│  ┌──────────┐    ┌──────────────┐    ┌──────────────────────┐  │
│  │ LINE Bot │───>│  Prometheus  │───>│  Grafana Dashboard   │  │
│  │ (Metrics)│    │  (Scraping)  │    │  (Visualization)     │  │
│  └──────────┘    └──────────────┘    └──────────────────────┘  │
│       │                │                         │               │
│       │          ┌─────┴──────┐          ┌──────┴──────┐       │
│       │          │AlertManager│          │   PagerDuty │       │
│       │          └─────┬──────┘          └─────────────┘       │
│       │                │                                         │
│  ┌────┴─────┐    ┌─────┴──────┐                                │
│  │  Pino    │    │  LINE OA   │                                │
│  │  Logger  │    │  Alert Bot │                                │
│  └────┬─────┘    └────────────┘                                │
│       │                                                          │
│  ┌────┴─────┐                                                   │
│  │ELK Stack │                                                   │
│  │(Elastic) │                                                   │
│  └──────────┘                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 1. Prometheus + Grafana Setup

### 1.1 Installation ด้วย Docker Compose

```yaml
# docker-compose.monitoring.yml
version: '3.8'

services:
  # LINE Bot Application
  linebot:
    image: your-linebot:latest
    ports:
      - "3000:3000"
    environment:
      - NODE_ENV=production
    networks:
      - monitoring

  # Prometheus
  prometheus:
    image: prom/prometheus:v2.48.0
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./monitoring/alerts.yml:/etc/prometheus/alerts.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    networks:
      - monitoring

  # Grafana
  grafana:
    image: grafana/grafana:10.2.0
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    networks:
      - monitoring
    depends_on:
      - prometheus

  # AlertManager
  alertmanager:
    image: prom/alertmanager:v0.26.0
    ports:
      - "9093:9093"
    volumes:
      - ./monitoring/alertmanager.yml:/etc/alertmanager/alertmanager.yml
    networks:
      - monitoring

  # Node Exporter (system metrics)
  node-exporter:
    image: prom/node-exporter:v1.7.0
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
    networks:
      - monitoring

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:
```

### 1.2 Prometheus Configuration

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    environment: 'production'
    service: 'linebot'

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - alertmanager:9093

rule_files:
  - "alerts.yml"

scrape_configs:
  # LINE Bot metrics
  - job_name: 'linebot'
    static_configs:
      - targets: ['linebot:3000']
    metrics_path: '/metrics'
    scrape_interval: 10s
    scrape_timeout: 5s

  # Node Exporter
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
    scrape_interval: 15s

  # Redis
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']

  # MySQL
  - job_name: 'mysql'
    static_configs:
      - targets: ['mysql-exporter:9104']
```

---

## 2. Custom Metrics สำหรับ LINE Bot

### 2.1 Prometheus Metrics Implementation

```javascript
// monitoring/metrics.js
const client = require('prom-client');

// สร้าง Registry
const register = new client.Registry();

// Default metrics (CPU, Memory, etc.)
client.collectDefaultMetrics({ register });

// ============================
// Custom Metrics สำหรับ LINE Bot
// ============================

// 1. Messages per minute
const messagesTotal = new client.Counter({
  name: 'linebot_messages_total',
  help: 'Total number of messages received',
  labelNames: ['event_type', 'source_type', 'status'],
  registers: [register]
});

// 2. Webhook Processing Time
const webhookDuration = new client.Histogram({
  name: 'linebot_webhook_duration_seconds',
  help: 'Time spent processing webhook events',
  labelNames: ['event_type', 'status'],
  buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5],
  registers: [register]
});

// 3. Error Rate
const errorsTotal = new client.Counter({
  name: 'linebot_errors_total',
  help: 'Total number of errors',
  labelNames: ['error_type', 'event_type'],
  registers: [register]
});

// 4. API Calls
const lineApiCallsTotal = new client.Counter({
  name: 'linebot_api_calls_total',
  help: 'Total LINE API calls made',
  labelNames: ['endpoint', 'method', 'status_code'],
  registers: [register]
});

// 5. API Response Time
const lineApiDuration = new client.Histogram({
  name: 'linebot_api_duration_seconds',
  help: 'LINE API call duration',
  labelNames: ['endpoint'],
  buckets: [0.1, 0.3, 0.5, 1, 2, 5, 10],
  registers: [register]
});

// 6. Active Users (Gauge)
const activeUsers = new client.Gauge({
  name: 'linebot_active_users',
  help: 'Number of users who interacted in last 5 minutes',
  registers: [register]
});

// 7. Message Queue Size
const messageQueueSize = new client.Gauge({
  name: 'linebot_message_queue_size',
  help: 'Number of messages waiting to be processed',
  registers: [register]
});

// 8. API Quota Usage
const apiQuotaUsage = new client.Gauge({
  name: 'linebot_api_quota_used',
  help: 'LINE API quota used (messages sent this month)',
  registers: [register]
});

// 9. Webhook Events by Type
const webhookEventsTotal = new client.Counter({
  name: 'linebot_webhook_events_total',
  help: 'Total webhook events by type',
  labelNames: ['event_type', 'source_type'],
  registers: [register]
});

// 10. Database Query Duration
const dbQueryDuration = new client.Histogram({
  name: 'linebot_db_query_duration_seconds',
  help: 'Database query duration',
  labelNames: ['operation', 'table'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1],
  registers: [register]
});

// 11. Redis Operations
const redisOpsTotal = new client.Counter({
  name: 'linebot_redis_ops_total',
  help: 'Redis operations count',
  labelNames: ['operation', 'status'],
  registers: [register]
});

// 12. Concurrent Webhooks
const concurrentWebhooks = new client.Gauge({
  name: 'linebot_concurrent_webhooks',
  help: 'Number of webhooks currently being processed',
  registers: [register]
});

module.exports = {
  register,
  metrics: {
    messagesTotal,
    webhookDuration,
    errorsTotal,
    lineApiCallsTotal,
    lineApiDuration,
    activeUsers,
    messageQueueSize,
    apiQuotaUsage,
    webhookEventsTotal,
    dbQueryDuration,
    redisOpsTotal,
    concurrentWebhooks
  }
};
```

### 2.2 Metrics Middleware

```javascript
// middleware/metricsMiddleware.js
const { metrics } = require('../monitoring/metrics');

/**
 * Webhook Metrics Middleware
 */
function webhookMetricsMiddleware(req, res, next) {
  const startTime = Date.now();
  
  // เพิ่ม concurrent webhook counter
  metrics.concurrentWebhooks.inc();
  
  // Override res.json เพื่อ capture response
  const originalJson = res.json.bind(res);
  res.json = (body) => {
    const duration = (Date.now() - startTime) / 1000;
    const status = res.statusCode < 400 ? 'success' : 'error';
    
    // บันทึก duration
    metrics.webhookDuration.observe(
      { event_type: 'webhook', status },
      duration
    );
    
    // ลด concurrent counter
    metrics.concurrentWebhooks.dec();
    
    return originalJson(body);
  };
  
  next();
}

/**
 * LINE API Call Tracker
 */
class LineApiTracker {
  /**
   * Wrap LINE Client method ด้วย metrics
   */
  static wrapClient(lineClient) {
    const originalReplyMessage = lineClient.replyMessage.bind(lineClient);
    const originalPushMessage = lineClient.pushMessage.bind(lineClient);
    
    lineClient.replyMessage = async (...args) => {
      const timer = metrics.lineApiDuration.startTimer({ endpoint: 'replyMessage' });
      try {
        const result = await originalReplyMessage(...args);
        metrics.lineApiCallsTotal.inc({ 
          endpoint: 'replyMessage', 
          method: 'POST', 
          status_code: '200' 
        });
        return result;
      } catch (err) {
        metrics.lineApiCallsTotal.inc({ 
          endpoint: 'replyMessage', 
          method: 'POST', 
          status_code: err.statusCode || '500' 
        });
        metrics.errorsTotal.inc({ 
          error_type: 'api_call_failed', 
          event_type: 'reply' 
        });
        throw err;
      } finally {
        timer();
      }
    };
    
    lineClient.pushMessage = async (...args) => {
      const timer = metrics.lineApiDuration.startTimer({ endpoint: 'pushMessage' });
      try {
        const result = await originalPushMessage(...args);
        metrics.lineApiCallsTotal.inc({ 
          endpoint: 'pushMessage', 
          method: 'POST', 
          status_code: '200' 
        });
        return result;
      } catch (err) {
        metrics.lineApiCallsTotal.inc({ 
          endpoint: 'pushMessage', 
          method: 'POST', 
          status_code: err.statusCode || '500' 
        });
        throw err;
      } finally {
        timer();
      }
    };
    
    return lineClient;
  }
}

module.exports = { webhookMetricsMiddleware, LineApiTracker };
```

### 2.3 Metrics Endpoint

```javascript
// routes/metrics.js
const express = require('express');
const router = express.Router();
const { register } = require('../monitoring/metrics');

// Prometheus scrape endpoint
// ควร protect ด้วย IP whitelist หรือ basic auth
router.get('/metrics', async (req, res) => {
  // ตรวจสอบว่ามาจาก Prometheus server
  const allowedIPs = (process.env.PROMETHEUS_IPS || '').split(',');
  const clientIP = req.ip;
  
  if (process.env.NODE_ENV === 'production' && !allowedIPs.includes(clientIP)) {
    return res.status(403).send('Forbidden');
  }
  
  try {
    res.set('Content-Type', register.contentType);
    const metrics = await register.metrics();
    res.end(metrics);
  } catch (err) {
    res.status(500).end(err.message);
  }
});

// Health check endpoint
router.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: process.env.npm_package_version
  });
});

// Readiness probe
router.get('/ready', async (req, res) => {
  try {
    // ตรวจสอบ dependencies
    await checkDatabase();
    await checkRedis();
    
    res.json({ status: 'ready' });
  } catch (err) {
    res.status(503).json({ 
      status: 'not_ready',
      reason: err.message
    });
  }
});

module.exports = router;
```

---

## 3. Logging ด้วย Winston และ Pino

### 3.1 Winston Logger Setup

```javascript
// logging/logger.js
const winston = require('winston');
const { format } = winston;

// Custom format สำหรับ LINE Bot logs
const lineBotFormat = format.combine(
  format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss.SSS' }),
  format.errors({ stack: true }),
  format.metadata({
    fillExcept: ['message', 'level', 'timestamp', 'label']
  }),
  format.json()
);

// สร้าง logger
const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: lineBotFormat,
  defaultMeta: {
    service: 'linebot',
    environment: process.env.NODE_ENV,
    version: process.env.APP_VERSION
  },
  transports: [
    // Console (development)
    new winston.transports.Console({
      format: process.env.NODE_ENV === 'development'
        ? format.combine(format.colorize(), format.simple())
        : lineBotFormat
    }),
    
    // File - all logs
    new winston.transports.File({
      filename: '/var/log/linebot/combined.log',
      maxsize: 100 * 1024 * 1024, // 100MB
      maxFiles: 30,
      tailable: true
    }),
    
    // File - errors only
    new winston.transports.File({
      filename: '/var/log/linebot/error.log',
      level: 'error',
      maxsize: 50 * 1024 * 1024,
      maxFiles: 90,
      tailable: true
    })
  ],
  
  exceptionHandlers: [
    new winston.transports.File({ 
      filename: '/var/log/linebot/exceptions.log' 
    })
  ],
  
  rejectionHandlers: [
    new winston.transports.File({ 
      filename: '/var/log/linebot/rejections.log' 
    })
  ]
});

// Helper methods สำหรับ LINE Bot specific logging
class LineBotLogger {
  constructor(logger) {
    this.logger = logger;
  }
  
  // บันทึก webhook event
  webhookReceived(event) {
    this.logger.info('Webhook event received', {
      category: 'webhook',
      eventType: event.type,
      sourceType: event.source?.type,
      userId: event.source?.userId,
      timestamp: event.timestamp
    });
  }
  
  // บันทึก message ที่ส่งออก
  messageSent(type, target, messageType, success) {
    this.logger.info('Message sent', {
      category: 'message',
      type, // 'reply' or 'push'
      target,
      messageType,
      success
    });
  }
  
  // บันทึก API call
  apiCall(endpoint, duration, statusCode) {
    this.logger.info('LINE API call', {
      category: 'api',
      endpoint,
      duration,
      statusCode
    });
  }
  
  // บันทึก error
  error(message, context = {}) {
    this.logger.error(message, {
      category: 'error',
      ...context
    });
  }
  
  // บันทึก security event
  security(event, details = {}) {
    this.logger.warn('Security event', {
      category: 'security',
      event,
      ...details
    });
  }
  
  // บันทึก business event
  business(event, data = {}) {
    this.logger.info('Business event', {
      category: 'business',
      event,
      ...data
    });
  }
}

module.exports = new LineBotLogger(logger);
```

### 3.2 Pino Logger (สำหรับ Performance)

```javascript
// logging/pinoLogger.js
const pino = require('pino');
const pinoHttp = require('pino-http');

// Pino logger (เร็วกว่า Winston มาก)
const pinoLogger = pino({
  level: process.env.LOG_LEVEL || 'info',
  
  // Redact sensitive fields
  redact: {
    paths: [
      'req.headers.authorization',
      'req.headers["x-line-signature"]',
      '*.password',
      '*.credit_card',
      '*.phone_number'
    ],
    censor: '[REDACTED]'
  },
  
  // Custom serializers
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      ip: req.remoteAddress,
      userAgent: req.headers['user-agent']
    }),
    res: (res) => ({
      statusCode: res.statusCode
    }),
    err: pino.stdSerializers.err
  },
  
  // Transport (production: JSON, development: pretty print)
  transport: process.env.NODE_ENV === 'development'
    ? {
        target: 'pino-pretty',
        options: {
          colorize: true,
          translateTime: 'SYS:standard',
          ignore: 'pid,hostname'
        }
      }
    : undefined,
  
  // Metadata
  base: {
    pid: process.pid,
    service: 'linebot',
    env: process.env.NODE_ENV
  }
});

// HTTP request logging middleware
const httpLogger = pinoHttp({
  logger: pinoLogger,
  
  // Custom log level สำหรับ request/response
  customLogLevel: (req, res, err) => {
    if (res.statusCode >= 500) return 'error';
    if (res.statusCode >= 400) return 'warn';
    return 'info';
  },
  
  // ซ่อน health check requests
  autoLogging: {
    ignore: (req) => req.url === '/health' || req.url === '/ready'
  }
});

module.exports = { pinoLogger, httpLogger };
```

---

## 4. Log Aggregation ด้วย ELK Stack

### 4.1 Filebeat Configuration

```yaml
# filebeat/filebeat.yml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /var/log/linebot/*.log
    json.keys_under_root: true
    json.add_error_key: true
    
    # เพิ่ม metadata
    fields:
      service: linebot
      environment: production
    fields_under_root: true
    
    # Multiline สำหรับ stack traces
    multiline.pattern: '^\s+'
    multiline.negate: false
    multiline.match: after

processors:
  - add_host_metadata: ~
  - add_cloud_metadata: ~
  
  # ลบ fields ที่ไม่ต้องการ
  - drop_fields:
      fields: ["log.file.path", "agent.type"]

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "linebot-logs-%{+yyyy.MM.dd}"
  
  # Template settings
  template.name: "linebot"
  template.pattern: "linebot-*"

setup.kibana:
  host: "kibana:5601"
```

### 4.2 Logstash Pipeline

```
# logstash/pipeline/linebot.conf
input {
  beats {
    port => 5044
  }
}

filter {
  # Parse JSON logs
  if [message] =~ /^\{/ {
    json {
      source => "message"
    }
  }
  
  # Extract LINE User ID สำหรับ search
  if [userId] {
    mutate {
      add_field => { "user_id_masked" => "%{userId}" }
    }
    # mask user ID (เก็บ 8 chars แรก)
    mutate {
      gsub => ["user_id_masked", "(.{8}).+", "\1****"]
    }
  }
  
  # กำหนด log level color
  if [level] == "error" {
    mutate { add_tag => ["alert"] }
  }
  
  # เพิ่ม geo data ถ้ามี IP
  if [ip_address] {
    geoip {
      source => "ip_address"
      target => "geoip"
    }
  }
  
  # ลบ fields ที่ sensitive
  mutate {
    remove_field => ["password", "token", "secret"]
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "linebot-logs-%{+YYYY.MM.dd}"
  }
  
  # ส่ง error logs ไปยัง alerting
  if "alert" in [tags] {
    http {
      url => "http://alertmanager:9093/api/v2/alerts"
      http_method => "post"
      format => "json"
      content_type => "application/json"
    }
  }
}
```

---

## 5. Alerting Rules

### 5.1 Prometheus Alert Rules

```yaml
# monitoring/alerts.yml
groups:
  - name: linebot_alerts
    interval: 30s
    rules:
      # Bot ไม่ตอบสนอง
      - alert: LineBotDown
        expr: up{job="linebot"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "LINE Bot is down"
          description: "LINE Bot has been unreachable for more than 1 minute"
          runbook: "https://wiki.your-company.com/runbooks/linebot-down"

      # Error rate สูง
      - alert: HighErrorRate
        expr: |
          rate(linebot_errors_total[5m]) / 
          rate(linebot_messages_total[5m]) > 0.1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 10%)"

      # Response time ช้า
      - alert: SlowWebhookProcessing
        expr: |
          histogram_quantile(0.95, 
            rate(linebot_webhook_duration_seconds_bucket[5m])
          ) > 3
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Webhook processing is slow"
          description: "P95 processing time is {{ $value }}s (threshold: 3s)"

      # API Quota ใกล้หมด
      - alert: APIQuotaWarning
        expr: linebot_api_quota_used > 800
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "LINE API quota warning"
          description: "API quota used: {{ $value }} (limit: 1000 per month)"

      - alert: APIQuotaCritical
        expr: linebot_api_quota_used > 950
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "LINE API quota critical"
          description: "API quota nearly exhausted: {{ $value }}/1000"

      # Memory usage สูง
      - alert: HighMemoryUsage
        expr: |
          (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / 
          node_memory_MemTotal_bytes > 0.85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is {{ $value | humanizePercentage }}"

      # Database connection pool ใกล้หมด
      - alert: DatabaseConnectionPoolExhausted
        expr: linebot_db_pool_available == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Database connection pool exhausted"

      # Concurrent webhooks สูง
      - alert: HighConcurrentWebhooks
        expr: linebot_concurrent_webhooks > 100
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High concurrent webhook load"
          description: "{{ $value }} webhooks being processed simultaneously"
```

### 5.2 AlertManager Configuration

```yaml
# monitoring/alertmanager.yml
global:
  resolve_timeout: 5m
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@your-company.com'
  smtp_auth_username: 'alerts@your-company.com'
  smtp_auth_password: '${SMTP_PASSWORD}'

templates:
  - '/etc/alertmanager/templates/*.tmpl'

route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  
  routes:
    # Critical alerts - แจ้งทันที
    - match:
        severity: critical
      receiver: 'pagerduty'
      continue: true
    
    # Warning alerts - แจ้งทุก 1 ชั่วโมง
    - match:
        severity: warning
      receiver: 'slack'
      group_interval: 1h
    
    # Business hours only
    - match_re:
        alertname: 'NonCritical.*'
      receiver: 'email'
      active_time_intervals:
        - business_hours

time_intervals:
  - name: business_hours
    time_intervals:
      - times:
          - start_time: '08:00'
            end_time: '20:00'
        weekdays: ['monday:friday']
        location: 'Asia/Bangkok'

receivers:
  - name: 'default'
    email_configs:
      - to: 'team@your-company.com'
        html: '{{ template "email.html" . }}'

  - name: 'pagerduty'
    pagerduty_configs:
      - routing_key: '${PAGERDUTY_ROUTING_KEY}'
        description: '{{ template "pagerduty.description" . }}'

  - name: 'slack'
    slack_configs:
      - api_url: '${SLACK_WEBHOOK_URL}'
        channel: '#linebot-alerts'
        title: '{{ template "slack.title" . }}'
        text: '{{ template "slack.text" . }}'
        send_resolved: true
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'

  - name: 'email'
    email_configs:
      - to: 'devops@your-company.com'
        subject: '[LINE Bot Alert] {{ .GroupLabels.alertname }}'
```

---

## 6. LINE OA เป็น Alerting Channel

### 6.1 ส่ง Alert ผ่าน LINE Bot (Meta!)

```javascript
// services/lineAlertService.js

class LineAlertService {
  constructor(lineClient) {
    this.lineClient = lineClient;
    this.adminGroupId = process.env.ADMIN_GROUP_ID;
    this.alertBotUserId = process.env.ALERT_BOT_USER_ID;
  }
  
  /**
   * ส่ง alert ไปยัง Admin Group
   */
  async sendAlert(alert) {
    const { name, severity, description, value, labels } = alert;
    
    const severityEmoji = {
      critical: '🚨',
      warning: '⚠️',
      info: 'ℹ️'
    }[severity] || '🔔';
    
    const color = {
      critical: '#FF0000',
      warning: '#FF8C00',
      info: '#0000FF'
    }[severity] || '#888888';
    
    const flexMessage = {
      type: 'flex',
      altText: `${severityEmoji} ${name}`,
      contents: {
        type: 'bubble',
        styles: {
          header: { backgroundColor: color }
        },
        header: {
          type: 'box',
          layout: 'horizontal',
          contents: [
            {
              type: 'text',
              text: `${severityEmoji} ${severity.toUpperCase()}`,
              color: '#FFFFFF',
              weight: 'bold'
            },
            {
              type: 'text',
              text: new Date().toLocaleString('th-TH', {
                timeZone: 'Asia/Bangkok'
              }),
              color: '#FFFFFF',
              size: 'sm',
              align: 'end'
            }
          ]
        },
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: name,
              weight: 'bold',
              size: 'lg',
              wrap: true
            },
            {
              type: 'text',
              text: description,
              color: '#555555',
              size: 'sm',
              wrap: true,
              margin: 'sm'
            },
            {
              type: 'separator',
              margin: 'md'
            },
            {
              type: 'box',
              layout: 'vertical',
              margin: 'md',
              contents: Object.entries(labels || {}).map(([k, v]) => ({
                type: 'box',
                layout: 'horizontal',
                contents: [
                  {
                    type: 'text',
                    text: k,
                    size: 'xs',
                    color: '#888888',
                    flex: 2
                  },
                  {
                    type: 'text',
                    text: String(v),
                    size: 'xs',
                    flex: 3,
                    wrap: true
                  }
                ]
              }))
            }
          ]
        },
        footer: {
          type: 'box',
          layout: 'horizontal',
          contents: [
            {
              type: 'button',
              action: {
                type: 'uri',
                label: 'ดู Grafana',
                uri: `${process.env.GRAFANA_URL}/d/linebot`
              },
              style: 'link'
            },
            {
              type: 'button',
              action: {
                type: 'uri',
                label: 'Runbook',
                uri: `https://wiki.your-company.com/runbooks/${name.toLowerCase()}`
              },
              style: 'link'
            }
          ]
        }
      }
    };
    
    await this.lineClient.pushMessage(this.adminGroupId, flexMessage);
  }
  
  /**
   * Webhook endpoint สำหรับรับ alerts จาก AlertManager
   */
  static createAlertWebhook(lineAlertService) {
    const router = require('express').Router();
    
    router.post('/alerts', async (req, res) => {
      const { alerts } = req.body;
      
      for (const alert of alerts) {
        if (alert.status === 'firing') {
          await lineAlertService.sendAlert({
            name: alert.labels.alertname,
            severity: alert.labels.severity,
            description: alert.annotations.description,
            labels: alert.labels,
            value: alert.annotations.value
          });
        }
      }
      
      res.json({ status: 'ok' });
    });
    
    return router;
  }
}

module.exports = LineAlertService;
```

---

## 7. Dashboard Templates

### 7.1 Grafana Dashboard JSON

```json
{
  "title": "LINE Bot Production Dashboard",
  "uid": "linebot-prod",
  "timezone": "Asia/Bangkok",
  "panels": [
    {
      "title": "Messages per Minute",
      "type": "stat",
      "gridPos": { "x": 0, "y": 0, "w": 4, "h": 4 },
      "targets": [{
        "expr": "rate(linebot_messages_total[1m]) * 60",
        "legendFormat": "Messages/min"
      }],
      "options": {
        "colorMode": "background",
        "graphMode": "area",
        "reduceOptions": { "calcs": ["lastNotNull"] }
      }
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "gridPos": { "x": 4, "y": 0, "w": 4, "h": 4 },
      "targets": [{
        "expr": "rate(linebot_errors_total[5m]) / rate(linebot_messages_total[5m]) * 100",
        "legendFormat": "Error %"
      }],
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 1 },
          { "color": "red", "value": 10 }
        ]
      }
    },
    {
      "title": "Webhook Processing Time (P95)",
      "type": "stat",
      "gridPos": { "x": 8, "y": 0, "w": 4, "h": 4 },
      "targets": [{
        "expr": "histogram_quantile(0.95, rate(linebot_webhook_duration_seconds_bucket[5m]))",
        "legendFormat": "P95 Duration"
      }],
      "unit": "s"
    },
    {
      "title": "API Quota Usage",
      "type": "gauge",
      "gridPos": { "x": 12, "y": 0, "w": 4, "h": 4 },
      "targets": [{
        "expr": "linebot_api_quota_used",
        "legendFormat": "Quota Used"
      }],
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"] },
        "minValue": 0,
        "maxValue": 1000
      },
      "thresholds": {
        "mode": "absolute",
        "steps": [
          { "color": "green", "value": null },
          { "color": "yellow", "value": 800 },
          { "color": "red", "value": 950 }
        ]
      }
    }
  ]
}
```

---

## 8. SLA Monitoring

### 8.1 SLA Definition และ Tracking

```javascript
// services/slaMonitor.js

class SLAMonitor {
  constructor(metricsClient) {
    this.metrics = metricsClient;
    this.slaDefinitions = {
      availability: {
        target: 99.9,  // 99.9% uptime = 8.76 hours downtime per year
        measurement: 'percentage'
      },
      responseTime: {
        target: 2000,  // P95 < 2 seconds
        measurement: 'milliseconds',
        percentile: 95
      },
      errorRate: {
        target: 1,     // < 1% error rate
        measurement: 'percentage'
      }
    };
  }
  
  /**
   * คำนวณ SLA สำหรับช่วงเวลา
   */
  async calculateSLA(startDate, endDate) {
    const results = {};
    
    // Availability
    const uptimeQuery = await this.metrics.query({
      query: `avg_over_time(up{job="linebot"}[${this.getRange(startDate, endDate)}])`,
      time: endDate.toISOString()
    });
    
    results.availability = {
      value: (uptimeQuery.data?.result[0]?.value[1] || 0) * 100,
      target: this.slaDefinitions.availability.target,
      met: (uptimeQuery.data?.result[0]?.value[1] || 0) * 100 >= 99.9
    };
    
    return results;
  }
  
  /**
   * สร้าง SLA Report รายเดือน
   */
  async generateMonthlyReport(year, month) {
    const startDate = new Date(year, month - 1, 1);
    const endDate = new Date(year, month, 0, 23, 59, 59);
    
    const sla = await this.calculateSLA(startDate, endDate);
    
    const report = {
      period: `${year}-${String(month).padStart(2, '0')}`,
      generatedAt: new Date().toISOString(),
      sla,
      summary: {
        allMetMet: Object.values(sla).every(m => m.met),
        overallScore: this.calculateOverallScore(sla)
      }
    };
    
    return report;
  }
  
  calculateOverallScore(sla) {
    const scores = Object.values(sla).map(m => m.met ? 1 : 0);
    return (scores.reduce((a, b) => a + b, 0) / scores.length) * 100;
  }
  
  getRange(startDate, endDate) {
    const diffMs = endDate - startDate;
    const diffDays = Math.ceil(diffMs / (1000 * 60 * 60 * 24));
    return `${diffDays}d`;
  }
}

module.exports = SLAMonitor;
```

---

## 9. Complete Monitoring Setup

### 9.1 Integration ใน Application

```javascript
// app.js (complete monitoring setup)
const express = require('express');
const { register, metrics } = require('./monitoring/metrics');
const { webhookMetricsMiddleware, LineApiTracker } = require('./middleware/metricsMiddleware');
const { pinoLogger, httpLogger } = require('./logging/pinoLogger');
const LineAlertService = require('./services/lineAlertService');

const app = express();

// Logging
app.use(httpLogger);

// Metrics
app.use(webhookMetricsMiddleware);

// Setup LINE client with metrics
const lineClient = LineApiTracker.wrapClient(rawLineClient);

// Alert service
const alertService = new LineAlertService(lineClient);
app.use('/api/alerts', LineAlertService.createAlertWebhook(alertService));

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

// Webhook với metrics
app.post('/webhook', async (req, res) => {
  const events = req.body.events || [];
  
  for (const event of events) {
    // Track event
    metrics.webhookEventsTotal.inc({
      event_type: event.type,
      source_type: event.source?.type || 'unknown'
    });
    metrics.messagesTotal.inc({
      event_type: event.type,
      source_type: event.source?.type || 'unknown',
      status: 'received'
    });
    
    // Process event
    const timer = metrics.webhookDuration.startTimer({
      event_type: event.type
    });
    
    try {
      await handleEvent(event);
      timer({ status: 'success' });
      metrics.messagesTotal.inc({
        event_type: event.type,
        source_type: event.source?.type || 'unknown',
        status: 'processed'
      });
    } catch (err) {
      timer({ status: 'error' });
      metrics.errorsTotal.inc({
        error_type: err.constructor.name,
        event_type: event.type
      });
      pinoLogger.error({ err, event: event.type }, 'Error processing webhook event');
    }
  }
  
  res.status(200).json({ status: 'ok' });
});

// Periodic tasks
setInterval(async () => {
  // อัปเดต active users count
  const activeUsersCount = await redis.scard('active_users:last5min');
  metrics.activeUsers.set(activeUsersCount);
  
  // อัปเดต queue size
  const queueSize = await redis.llen('message_queue');
  metrics.messageQueueSize.set(queueSize);
  
  // Reset active users set
  await redis.del('active_users:last5min');
}, 5 * 60 * 1000);

module.exports = app;
```

---

## 10. PagerDuty Integration

### 10.1 PagerDuty AlertManager Config

```yaml
# monitoring/alertmanager-pagerduty.yml
receivers:
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: '${PAGERDUTY_INTEGRATION_KEY}'
        description: '{{ template "pagerduty.default.description" . }}'
        severity: '{{ if eq .CommonLabels.severity "critical" }}critical{{ else }}warning{{ end }}'
        details:
          firing: '{{ .Alerts.Firing | len }}'
          resolved: '{{ .Alerts.Resolved | len }}'
          alertname: '{{ .CommonLabels.alertname }}'
          service: '{{ .CommonLabels.service }}'
          environment: '{{ .CommonLabels.environment }}'
        links:
          - href: '{{ (index .Alerts 0).GeneratorURL }}'
            text: 'View in Prometheus'
          - href: 'https://grafana.your-company.com/d/linebot-prod'
            text: 'View in Grafana'
        client: 'LINE Bot AlertManager'
        client_url: 'https://alertmanager.your-company.com'
```

### 10.2 OpsGenie Integration

```yaml
# monitoring/alertmanager-opsgenie.yml
receivers:
  - name: 'opsgenie-critical'
    opsgenie_configs:
      - api_key: '${OPSGENIE_API_KEY}'
        api_url: 'https://api.opsgenie.com/'
        message: '{{ .CommonLabels.alertname }}'
        description: '{{ .CommonAnnotations.description }}'
        priority: '{{ if eq .CommonLabels.severity "critical" }}P1{{ else if eq .CommonLabels.severity "warning" }}P2{{ else }}P3{{ end }}'
        tags: ['linebot', 'production', '{{ .CommonLabels.severity }}']
        details:
          environment: 'production'
          service: 'linebot'
          runbook: '{{ .CommonAnnotations.runbook }}'
        responders:
          - name: 'linebot-oncall'
            type: 'team'
```

---

## 11. Dashboard Templates ขั้นสูง

### 11.1 Grafana Provisioning

```yaml
# monitoring/grafana/datasources/prometheus.yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    version: 1
    editable: false
    jsonData:
      timeInterval: "15s"
      httpMethod: POST
      
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    version: 1
    editable: false
```

```yaml
# monitoring/grafana/dashboards/linebot-dashboard.yaml
apiVersion: 1

providers:
  - name: 'LINE Bot Dashboards'
    orgId: 1
    folder: 'LINE Bot'
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    allowUiUpdates: true
    options:
      path: /etc/grafana/provisioning/dashboards
```

### 11.2 Key Dashboard Panels

```javascript
// monitoring/grafana/dashboard-config.js
// Configuration สำหรับ Grafana Dashboard panels

const DASHBOARD_PANELS = [
  // Row 1: Overview
  {
    title: "Bot Status",
    type: "stat",
    query: "up{job='linebot'}",
    thresholds: [
      { color: "red", value: 0 },
      { color: "green", value: 1 }
    ],
    mappings: [
      { value: "0", text: "DOWN" },
      { value: "1", text: "UP" }
    ]
  },
  {
    title: "Messages Today",
    type: "stat",
    query: "increase(linebot_messages_total[24h])",
    colorMode: "value"
  },
  {
    title: "Error Rate (5m)",
    type: "gauge",
    query: "rate(linebot_errors_total[5m]) / rate(linebot_messages_total[5m]) * 100",
    min: 0,
    max: 100,
    unit: "percent",
    thresholds: [
      { color: "green", value: 0 },
      { color: "yellow", value: 1 },
      { color: "red", value: 10 }
    ]
  },
  
  // Row 2: Throughput
  {
    title: "Messages per Minute",
    type: "timeseries",
    query: "rate(linebot_messages_total[1m]) * 60",
    legend: "{{event_type}}"
  },
  {
    title: "Webhook Processing Time (P95)",
    type: "timeseries",
    query: "histogram_quantile(0.95, rate(linebot_webhook_duration_seconds_bucket[5m]))",
    unit: "s"
  },
  
  // Row 3: System Health
  {
    title: "CPU Usage",
    type: "timeseries",
    query: "rate(process_cpu_seconds_total{job='linebot'}[5m]) * 100",
    unit: "percent"
  },
  {
    title: "Memory Usage",
    type: "timeseries",
    query: "process_resident_memory_bytes{job='linebot'} / 1024 / 1024",
    unit: "megabytes"
  },
  
  // Row 4: LINE API
  {
    title: "LINE API Calls",
    type: "timeseries",
    query: "rate(linebot_api_calls_total[5m])",
    legend: "{{endpoint}} {{status_code}}"
  },
  {
    title: "API Quota Progress",
    type: "bargauge",
    query: "linebot_api_quota_used",
    min: 0,
    max: 1000,
    thresholds: [
      { color: "green", value: 0 },
      { color: "yellow", value: 800 },
      { color: "red", value: 950 }
    ]
  }
];

module.exports = DASHBOARD_PANELS;
```

---

## 12. Uptime Monitoring

### 12.1 External Uptime Monitoring

```javascript
// scripts/uptimeMonitor.js
// สำหรับรัน uptime check จาก external เพื่อ monitor end-to-end

const https = require('https');

const ENDPOINTS = [
  {
    url: 'https://bot.your-company.com/health',
    name: 'Health Check',
    expectedStatus: 200,
    timeout: 5000
  },
  {
    url: 'https://bot.your-company.com/ready',
    name: 'Readiness Check',
    expectedStatus: 200,
    timeout: 5000
  }
];

async function checkEndpoint(endpoint) {
  const startTime = Date.now();
  
  return new Promise((resolve) => {
    const req = https.get(endpoint.url, { timeout: endpoint.timeout }, (res) => {
      const duration = Date.now() - startTime;
      
      resolve({
        name: endpoint.name,
        url: endpoint.url,
        status: res.statusCode,
        duration,
        success: res.statusCode === endpoint.expectedStatus
      });
    });
    
    req.on('error', (err) => {
      resolve({
        name: endpoint.name,
        url: endpoint.url,
        status: 0,
        duration: Date.now() - startTime,
        success: false,
        error: err.message
      });
    });
    
    req.on('timeout', () => {
      req.destroy();
      resolve({
        name: endpoint.name,
        url: endpoint.url,
        status: 0,
        duration: endpoint.timeout,
        success: false,
        error: 'timeout'
      });
    });
  });
}

async function runUptimeCheck() {
  const results = await Promise.all(
    ENDPOINTS.map(checkEndpoint)
  );
  
  const allHealthy = results.every(r => r.success);
  
  if (!allHealthy) {
    const failedEndpoints = results.filter(r => !r.success);
    console.error('UPTIME CHECK FAILED:', JSON.stringify(failedEndpoints));
    
    // ส่ง alert
    process.exit(1);
  } else {
    console.log('All endpoints healthy:', results.map(r => 
      `${r.name}: ${r.duration}ms`
    ).join(', '));
  }
}

runUptimeCheck();
```

### 12.2 Synthetic Monitoring

```javascript
// tests/synthetic/webhookSynthetic.test.js
// ทดสอบ webhook แบบ end-to-end จาก external

const axios = require('axios');
const crypto = require('crypto');

describe('Synthetic Webhook Tests', () => {
  const WEBHOOK_URL = process.env.WEBHOOK_URL || 'https://bot.your-company.com/webhook';
  const CHANNEL_SECRET = process.env.LINE_CHANNEL_SECRET;
  
  function createSignature(body) {
    return crypto
      .createHmac('SHA256', CHANNEL_SECRET)
      .update(body)
      .digest('base64');
  }
  
  test('webhook responds within 3 seconds', async () => {
    const body = JSON.stringify({ events: [] });
    const signature = createSignature(body);
    
    const startTime = Date.now();
    
    const response = await axios.post(WEBHOOK_URL, body, {
      headers: {
        'Content-Type': 'application/json',
        'X-Line-Signature': signature
      },
      timeout: 5000
    });
    
    const duration = Date.now() - startTime;
    
    expect(response.status).toBe(200);
    expect(duration).toBeLessThan(3000);
  }, 10000);
  
  test('webhook rejects invalid signature', async () => {
    try {
      await axios.post(WEBHOOK_URL, JSON.stringify({ events: [] }), {
        headers: {
          'Content-Type': 'application/json',
          'X-Line-Signature': 'invalidsignature'
        }
      });
      fail('Should have thrown 401');
    } catch (err) {
      expect(err.response.status).toBe(401);
    }
  });
});
```

---

## สรุปบทที่ 78

### Monitoring Checklist

```
Production Monitoring Checklist:
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  ✅ Prometheus scraping metrics ทุก 15 วินาที                 │
│  ✅ Grafana dashboard แสดง key metrics                       │
│  ✅ AlertManager ส่ง alerts เมื่อ threshold ถูกข้าม          │
│  ✅ PagerDuty/OpsGenie สำหรับ on-call escalation            │
│  ✅ LINE OA รับ critical alerts โดยตรง                       │
│  ✅ Logs ส่งไป ELK Stack                                    │
│  ✅ Log retention ตามนโยบาย                                  │
│  ✅ SLA monitoring รายเดือน                                  │
│  ✅ Health check endpoints พร้อมใช้                          │
│  ✅ Uptime monitoring (external)                             │
│  ✅ Synthetic tests รัน end-to-end ทุก 5 นาที               │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

### Quick Reference Commands

```bash
# ดู Prometheus targets
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'

# Query metrics ด้วย PromQL
curl 'http://localhost:9090/api/v1/query?query=rate(linebot_messages_total[5m])' | jq '.data.result'

# ดู active alerts
curl http://localhost:9090/api/v1/alerts | jq '.data.alerts[] | {name: .labels.alertname, state: .state}'

# ทดสอบ AlertManager routing
curl -X POST http://localhost:9093/api/v2/alerts \
  -H "Content-Type: application/json" \
  -d '[{
    "labels": {"alertname": "TestAlert", "severity": "warning"},
    "annotations": {"description": "Test alert from curl"}
  }]'
```

**ต่อไป**: [Part 79 - CI/CD Pipeline](./part-079-cicd-pipeline.md)
