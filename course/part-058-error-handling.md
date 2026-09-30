# Part 58: Error Handling - การจัดการข้อผิดพลาดแบบครบวงจร

## บทนำ

การจัดการข้อผิดพลาดที่ดีเป็นสิ่งสำคัญมากสำหรับ LINE Bot ที่ทำงานใน Production ข้อผิดพลาดที่ไม่ได้รับการจัดการอาจทำให้บอทหยุดทำงาน ผู้ใช้ไม่ได้รับการตอบสนอง หรือแม้แต่ข้อมูลรั่วไหล

---

## 1. LINE API Error Codes (รหัสข้อผิดพลาด)

### 1.1 HTTP Status Codes

```javascript
// constants/error-codes.js

const LINE_HTTP_STATUS = {
  200: { message: 'สำเร็จ', action: 'ไม่ต้องทำอะไร' },
  400: { message: 'Bad Request - Request ไม่ถูกต้อง', action: 'ตรวจสอบ Request Body' },
  401: { message: 'Unauthorized - Token ไม่ถูกต้อง', action: 'ต่ออายุ Token' },
  403: { message: 'Forbidden - ไม่มีสิทธิ์', action: 'ตรวจสอบ Permissions' },
  404: { message: 'Not Found - ไม่พบทรัพยากร', action: 'ตรวจสอบ URL' },
  409: { message: 'Conflict - ข้อมูลขัดแย้ง', action: 'ตรวจสอบ State' },
  410: { message: 'Gone - Reply Token หมดอายุ', action: 'ไม่ต้อง Retry' },
  422: { message: 'Unprocessable Entity - ข้อมูลไม่ถูกต้อง', action: 'ตรวจสอบ Data Format' },
  429: { message: 'Too Many Requests - เกิน Rate Limit', action: 'รอแล้ว Retry' },
  500: { message: 'Internal Server Error - LINE Server มีปัญหา', action: 'Retry ภายหลัง' },
  503: { message: 'Service Unavailable - LINE Service ไม่พร้อม', action: 'Retry ภายหลัง' }
};

const LINE_ERROR_CODES = {
  // Authentication errors
  401: {
    1001: 'Access Token ไม่ถูกต้องหรือหมดอายุ',
    1002: 'Channel ถูก Deactivated',
    1003: 'Channel Secret ไม่ถูกต้อง'
  },
  // Request errors
  400: {
    // Message errors
    1300: 'ข้อความเกินความยาวที่กำหนด',
    1301: 'จำนวนข้อความใน Multicast เกินกำหนด',
    1302: 'Reply Token ไม่ถูกต้อง',
    1303: 'Reply Token ใช้ไปแล้ว',
    // Sticker errors
    1400: 'Sticker ไม่ถูกต้อง',
    1401: 'Package ID ไม่ถูกต้อง',
    1402: 'Sticker ID ไม่ถูกต้อง',
    // Image errors
    1500: 'รูปภาพมีขนาดใหญ่เกินไป',
    1501: 'รูปแบบไฟล์ไม่รองรับ',
    // Rich Menu errors
    1600: 'Rich Menu ไม่พบ',
    1601: 'Rich Menu มีอยู่แล้ว',
    1602: 'จำนวน Rich Menu เกินกำหนด'
  },
  // Rate limit errors
  429: {
    2001: 'เกิน Rate Limit รายนาที',
    2002: 'เกิน Rate Limit รายชั่วโมง',
    2003: 'เกิน Rate Limit รายวัน'
  }
};

const RETRYABLE_STATUS_CODES = new Set([408, 429, 500, 502, 503, 504]);
const NON_RETRYABLE_STATUS_CODES = new Set([400, 401, 403, 404, 410, 422]);

module.exports = {
  LINE_HTTP_STATUS,
  LINE_ERROR_CODES,
  RETRYABLE_STATUS_CODES,
  NON_RETRYABLE_STATUS_CODES
};
```

---

## 2. Custom Error Classes

### 2.1 สร้าง Error Hierarchy

```javascript
// errors/line-errors.js

class LineError extends Error {
  constructor(message, statusCode, errorCode, details = {}) {
    super(message);
    this.name = 'LineError';
    this.statusCode = statusCode;
    this.errorCode = errorCode;
    this.details = details;
    this.timestamp = new Date().toISOString();

    if (Error.captureStackTrace) {
      Error.captureStackTrace(this, this.constructor);
    }
  }

  toJSON() {
    return {
      name: this.name,
      message: this.message,
      statusCode: this.statusCode,
      errorCode: this.errorCode,
      details: this.details,
      timestamp: this.timestamp
    };
  }
}

class AuthenticationError extends LineError {
  constructor(message, errorCode, details) {
    super(message || 'Authentication ล้มเหลว', 401, errorCode, details);
    this.name = 'AuthenticationError';
    this.isRetryable = false;
  }
}

class RateLimitError extends LineError {
  constructor(retryAfter, details) {
    super('เกิน Rate Limit', 429, 2001, details);
    this.name = 'RateLimitError';
    this.retryAfter = retryAfter;
    this.isRetryable = true;
  }
}

class TokenExpiredError extends AuthenticationError {
  constructor() {
    super('Channel Access Token หมดอายุ', 1001, {});
    this.name = 'TokenExpiredError';
  }
}

class InvalidSignatureError extends LineError {
  constructor() {
    super('Webhook Signature ไม่ถูกต้อง', 401, 1003, {});
    this.name = 'InvalidSignatureError';
    this.isRetryable = false;
  }
}

class MessageTooLongError extends LineError {
  constructor(length, maxLength) {
    super(`ข้อความยาว ${length} ตัวอักษร เกินขีดจำกัด ${maxLength}`, 400, 1300, { length, maxLength });
    this.name = 'MessageTooLongError';
    this.isRetryable = false;
  }
}

class StickerNotFoundError extends LineError {
  constructor(packageId, stickerId) {
    super(`ไม่พบ Sticker: packageId=${packageId}, stickerId=${stickerId}`, 400, 1400, {
      packageId,
      stickerId
    });
    this.name = 'StickerNotFoundError';
    this.isRetryable = false;
  }
}

class ReplyTokenExpiredError extends LineError {
  constructor() {
    super('Reply Token หมดอายุหรือใช้ไปแล้ว', 400, 1302, {});
    this.name = 'ReplyTokenExpiredError';
    this.isRetryable = false;
  }
}

class NetworkError extends LineError {
  constructor(message, originalError) {
    super(message || 'Network Error', 0, 0, { originalError: originalError?.message });
    this.name = 'NetworkError';
    this.isRetryable = true;
    this.originalError = originalError;
  }
}

class WebhookProcessingError extends Error {
  constructor(message, eventId, eventType, originalError) {
    super(message);
    this.name = 'WebhookProcessingError';
    this.eventId = eventId;
    this.eventType = eventType;
    this.originalError = originalError;
    this.timestamp = new Date().toISOString();
  }
}

module.exports = {
  LineError,
  AuthenticationError,
  RateLimitError,
  TokenExpiredError,
  InvalidSignatureError,
  MessageTooLongError,
  StickerNotFoundError,
  ReplyTokenExpiredError,
  NetworkError,
  WebhookProcessingError
};
```

---

## 3. Error Handler Middleware (Node.js)

### 3.1 Global Error Handler

```javascript
// middleware/error-handler.js
const {
  LineError,
  AuthenticationError,
  RateLimitError,
  InvalidSignatureError
} = require('../errors/line-errors');
const logger = require('../utils/logger');
const alertService = require('../services/alert-service');

/**
 * Global Error Handler Middleware
 */
function errorHandler(err, req, res, next) {
  // บันทึก Log
  logger.error({
    error: err.name || 'Error',
    message: err.message,
    statusCode: err.statusCode,
    path: req.path,
    method: req.method,
    stack: err.stack,
    timestamp: new Date().toISOString()
  });

  // จัดการ Error แต่ละประเภท
  if (err instanceof InvalidSignatureError) {
    return res.status(401).json({
      error: 'Invalid Signature',
      message: 'Webhook signature verification failed'
    });
  }

  if (err instanceof RateLimitError) {
    const retryAfter = err.retryAfter || 60;
    return res.status(429)
      .set('Retry-After', retryAfter)
      .json({
        error: 'Rate Limit Exceeded',
        message: err.message,
        retryAfter
      });
  }

  if (err instanceof AuthenticationError) {
    // ส่งการแจ้งเตือนทีม
    alertService.sendAlert({
      level: 'critical',
      title: 'Authentication Error',
      message: err.message
    }).catch(console.error);

    return res.status(401).json({
      error: 'Authentication Failed',
      message: err.message
    });
  }

  if (err instanceof LineError) {
    return res.status(err.statusCode || 500).json({
      error: err.name,
      message: err.message,
      code: err.errorCode
    });
  }

  // Unexpected errors
  if (process.env.NODE_ENV === 'production') {
    alertService.sendAlert({
      level: 'error',
      title: 'Unexpected Error',
      message: err.message,
      stack: err.stack
    }).catch(console.error);

    return res.status(500).json({
      error: 'Internal Server Error',
      message: 'เกิดข้อผิดพลาดภายใน กรุณาลองใหม่ภายหลัง'
    });
  }

  // Development
  return res.status(500).json({
    error: err.name || 'Error',
    message: err.message,
    stack: err.stack
  });
}

/**
 * Async Error Wrapper
 */
function asyncHandler(fn) {
  return (req, res, next) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
}

module.exports = { errorHandler, asyncHandler };
```

### 3.2 LINE API Error Parser

```javascript
// utils/error-parser.js
const {
  LineError,
  AuthenticationError,
  RateLimitError,
  TokenExpiredError,
  MessageTooLongError,
  StickerNotFoundError,
  ReplyTokenExpiredError,
  NetworkError
} = require('../errors/line-errors');
const { RETRYABLE_STATUS_CODES } = require('../constants/error-codes');

class LineApiErrorParser {
  /**
   * แปลง Axios Error เป็น Custom Error
   */
  static parse(error) {
    // Network Error
    if (!error.response) {
      if (error.code === 'ECONNABORTED') {
        return new NetworkError('Request Timeout', error);
      }
      if (error.code === 'ECONNREFUSED') {
        return new NetworkError('Connection Refused', error);
      }
      return new NetworkError(error.message, error);
    }

    const { status, data } = error.response;
    const errorCode = data?.error?.code;

    switch (status) {
      case 400:
        return this.handle400Error(errorCode, data);
      case 401:
        return this.handle401Error(errorCode);
      case 429:
        return this.handle429Error(error.response);
      default:
        return new LineError(
          data?.message || `HTTP ${status} Error`,
          status,
          errorCode,
          data
        );
    }
  }

  static handle400Error(errorCode, data) {
    const message = data?.message || 'Bad Request';

    if (message.includes('Too long') || errorCode === 1300) {
      const match = message.match(/(\d+).*max (\d+)/);
      return new MessageTooLongError(
        match ? parseInt(match[1]) : 0,
        match ? parseInt(match[2]) : 5000
      );
    }

    if (message.includes('sticker') || errorCode === 1400) {
      return new StickerNotFoundError(
        data?.details?.packageId,
        data?.details?.stickerId
      );
    }

    if (message.includes('reply token') || errorCode === 1302 || errorCode === 1303) {
      return new ReplyTokenExpiredError();
    }

    return new LineError(message, 400, errorCode, data);
  }

  static handle401Error(errorCode) {
    if (errorCode === 1001) {
      return new TokenExpiredError();
    }
    return new AuthenticationError('Unauthorized', errorCode);
  }

  static handle429Error(response) {
    const retryAfter = parseInt(response.headers['retry-after'] || '60');
    return new RateLimitError(retryAfter, { retryAfter });
  }

  /**
   * ตรวจสอบว่า Error สามารถ Retry ได้หรือไม่
   */
  static isRetryable(error) {
    if (error.isRetryable !== undefined) return error.isRetryable;
    if (error instanceof LineError) {
      return RETRYABLE_STATUS_CODES.has(error.statusCode);
    }
    return false;
  }
}

module.exports = LineApiErrorParser;
```

---

## 4. Rate Limiting Handler

### 4.1 Rate Limit Manager

```javascript
// services/rate-limit-manager.js
const redis = require('../config/redis');
const { RateLimitError } = require('../errors/line-errors');

class RateLimitManager {
  constructor() {
    this.limits = {
      replyMessage: { perSecond: 50, perMinute: null, perDay: null },
      pushMessage: { perSecond: null, perMinute: null, perDay: 200 },
      broadcastMessage: { perSecond: null, perMinute: null, perDay: 200 },
      multicastMessage: { perSecond: null, perMinute: null, perDay: 200 }
    };
  }

  /**
   * ตรวจสอบ Rate Limit ก่อนส่ง Request
   */
  async checkLimit(operation) {
    const limit = this.limits[operation];
    if (!limit) return true;

    const now = Date.now();

    // ตรวจสอบ Per Second
    if (limit.perSecond) {
      const key = `ratelimit:${operation}:second:${Math.floor(now / 1000)}`;
      const count = await redis.incr(key);
      await redis.expire(key, 2);

      if (count > limit.perSecond) {
        throw new RateLimitError(1, { operation, type: 'per_second', count });
      }
    }

    // ตรวจสอบ Per Day
    if (limit.perDay) {
      const date = new Date().toISOString().slice(0, 10);
      const key = `ratelimit:${operation}:day:${date}`;
      const count = await redis.incr(key);
      await redis.expire(key, 86400);

      if (count > limit.perDay) {
        const secondsUntilMidnight = this.getSecondsUntilMidnight();
        throw new RateLimitError(secondsUntilMidnight, {
          operation,
          type: 'per_day',
          count,
          limit: limit.perDay
        });
      }
    }

    return true;
  }

  getSecondsUntilMidnight() {
    const now = new Date();
    const midnight = new Date(now);
    midnight.setHours(24, 0, 0, 0);
    return Math.floor((midnight - now) / 1000);
  }

  /**
   * จัดการ 429 Response จาก LINE API
   */
  async handleRateLimitResponse(error, retryFn) {
    if (error instanceof RateLimitError) {
      const waitMs = (error.retryAfter || 60) * 1000;
      console.log(`Rate limit ถูก hit รอ ${waitMs}ms ก่อน Retry`);
      await new Promise(resolve => setTimeout(resolve, waitMs));
      return retryFn();
    }
    throw error;
  }
}

module.exports = new RateLimitManager();
```

---

## 5. Token Management (จัดการ Token)

### 5.1 Token Manager

```javascript
// services/token-manager.js
const axios = require('axios');
const redis = require('../config/redis');

class TokenManager {
  constructor() {
    this.tokenKey = 'line:channel_access_token';
    this.tokenTTL = 2592000; // 30 วัน
  }

  /**
   * ดึง Token ปัจจุบัน (จาก Cache หรือสร้างใหม่)
   */
  async getToken() {
    // ลองจาก Cache ก่อน
    let token = await redis.get(this.tokenKey);
    if (token) return token;

    // สร้าง Token ใหม่
    return this.issueNewToken();
  }

  /**
   * สร้าง Short-lived Token ใหม่
   */
  async issueNewToken() {
    try {
      const response = await axios.post(
        'https://api.line.me/v2/oauth/accessToken',
        new URLSearchParams({
          grant_type: 'client_credentials',
          client_id: process.env.LINE_CHANNEL_ID,
          client_secret: process.env.LINE_CHANNEL_SECRET
        }),
        { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
      );

      const { access_token, expires_in } = response.data;
      const ttl = Math.floor(expires_in * 0.9); // ใช้ 90% ของ expire time

      // Cache Token
      await redis.setex(this.tokenKey, ttl, access_token);

      console.log(`Token ใหม่ออกแล้ว จะหมดอายุใน ${ttl}s`);
      return access_token;
    } catch (error) {
      throw new Error(`ไม่สามารถออก Token ใหม่: ${error.message}`);
    }
  }

  /**
   * Invalidate Token
   */
  async revokeToken() {
    const token = await redis.get(this.tokenKey);
    if (!token) return;

    try {
      await axios.post('https://api.line.me/v2/oauth/revoke', {
        access_token: token
      });
      await redis.del(this.tokenKey);
      console.log('Token ถูก Revoke แล้ว');
    } catch (error) {
      console.error(`ไม่สามารถ Revoke Token: ${error.message}`);
    }
  }

  /**
   * Handle Token Expired Error
   */
  async refreshAndRetry(apiCall) {
    try {
      return await apiCall();
    } catch (error) {
      if (error.name === 'TokenExpiredError' || error.statusCode === 401) {
        console.log('Token หมดอายุ กำลังต่ออายุ...');
        await redis.del(this.tokenKey);
        const newToken = await this.issueNewToken();
        // Retry ด้วย Token ใหม่
        return await apiCall(newToken);
      }
      throw error;
    }
  }
}

module.exports = new TokenManager();
```

---

## 6. Error Logging Best Practices

### 6.1 Structured Logger

```javascript
// utils/logger.js
const winston = require('winston');
const { combine, timestamp, printf, colorize, errors } = winston.format;

const logFormat = printf(({ level, message, timestamp, stack, ...metadata }) => {
  let log = `${timestamp} [${level.toUpperCase()}]: ${message}`;

  if (Object.keys(metadata).length > 0) {
    log += `\n  ${JSON.stringify(metadata, null, 2)}`;
  }

  if (stack) {
    log += `\n  Stack: ${stack}`;
  }

  return log;
});

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: combine(
    errors({ stack: true }),
    timestamp({ format: 'YYYY-MM-DD HH:mm:ss.SSS' }),
    logFormat
  ),
  transports: [
    // Console
    new winston.transports.Console({
      format: combine(
        colorize(),
        timestamp({ format: 'HH:mm:ss' }),
        logFormat
      )
    }),
    // Error file
    new winston.transports.File({
      filename: 'logs/error.log',
      level: 'error',
      maxsize: 10 * 1024 * 1024, // 10MB
      maxFiles: 5
    }),
    // Combined log
    new winston.transports.File({
      filename: 'logs/combined.log',
      maxsize: 50 * 1024 * 1024, // 50MB
      maxFiles: 10
    })
  ]
});

// สร้าง Child Logger สำหรับแต่ละ Module
logger.forModule = (moduleName) => {
  return logger.child({ module: moduleName });
};

module.exports = logger;
```

### 6.2 Error Context Logger

```javascript
// utils/error-logger.js
const logger = require('./logger');

class ErrorLogger {
  constructor(moduleName) {
    this.logger = logger.forModule(moduleName);
  }

  logLineApiError(error, context = {}) {
    const logData = {
      errorType: error.name,
      statusCode: error.statusCode,
      errorCode: error.errorCode,
      isRetryable: error.isRetryable,
      ...context
    };

    if (error.statusCode >= 500) {
      this.logger.error(error.message, logData);
    } else if (error.statusCode === 429) {
      this.logger.warn(error.message, logData);
    } else {
      this.logger.error(error.message, logData);
    }
  }

  logWebhookError(error, eventId, eventType) {
    this.logger.error('Webhook Processing Error', {
      eventId,
      eventType,
      error: error.message,
      stack: error.stack
    });
  }

  logCritical(message, details) {
    this.logger.error(`[CRITICAL] ${message}`, details);
  }
}

module.exports = ErrorLogger;
```

---

## 7. Alert Service (ระบบแจ้งเตือน)

### 7.1 Alert Manager

```javascript
// services/alert-service.js
const axios = require('axios');
const logger = require('../utils/logger');

class AlertService {
  constructor() {
    this.slackWebhookUrl = process.env.SLACK_WEBHOOK_URL;
    this.lineNotifyToken = process.env.LINE_NOTIFY_TOKEN;
    this.alertThrottleMs = 300000; // 5 นาที
    this.recentAlerts = new Map();
  }

  /**
   * ส่งการแจ้งเตือน
   */
  async sendAlert({ level, title, message, details = {} }) {
    const alertKey = `${level}:${title}`;

    // Throttle: ไม่ส่งซ้ำภายใน 5 นาที
    const lastSent = this.recentAlerts.get(alertKey);
    if (lastSent && Date.now() - lastSent < this.alertThrottleMs) {
      return;
    }

    this.recentAlerts.set(alertKey, Date.now());

    const promises = [];

    if (this.slackWebhookUrl) {
      promises.push(this.sendSlackAlert({ level, title, message, details }));
    }

    if (this.lineNotifyToken) {
      promises.push(this.sendLineNotify({ level, title, message }));
    }

    await Promise.allSettled(promises);
  }

  async sendSlackAlert({ level, title, message, details }) {
    const colors = {
      critical: '#FF0000',
      error: '#FF6B6B',
      warning: '#FFA500',
      info: '#0000FF'
    };

    const payload = {
      attachments: [{
        color: colors[level] || colors.error,
        title: `[${level.toUpperCase()}] ${title}`,
        text: message,
        fields: Object.entries(details).map(([key, value]) => ({
          title: key,
          value: typeof value === 'object' ? JSON.stringify(value) : String(value),
          short: true
        })),
        footer: 'LINE Bot Alert System',
        ts: Math.floor(Date.now() / 1000)
      }]
    };

    try {
      await axios.post(this.slackWebhookUrl, payload);
    } catch (error) {
      logger.error('ส่ง Slack Alert ล้มเหลว:', error.message);
    }
  }

  async sendLineNotify({ level, title, message }) {
    const emoji = {
      critical: '🚨',
      error: '❌',
      warning: '⚠️',
      info: 'ℹ️'
    };

    const text = `\n${emoji[level] || '❌'} [${level.toUpperCase()}]\n${title}\n${message}`;

    try {
      await axios.post('https://notify-api.line.me/api/notify',
        new URLSearchParams({ message: text }),
        {
          headers: {
            'Authorization': `Bearer ${this.lineNotifyToken}`,
            'Content-Type': 'application/x-www-form-urlencoded'
          }
        }
      );
    } catch (error) {
      logger.error('ส่ง LINE Notify Alert ล้มเหลว:', error.message);
    }
  }
}

module.exports = new AlertService();
```

---

## 8. Graceful Degradation (การล้มเหลวอย่างสง่างาม)

### 8.1 Fallback Responses

```javascript
// services/fallback-service.js
const line = require('@line/bot-sdk');

const lineClient = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

class FallbackService {
  /**
   * ส่งข้อความ Fallback เมื่อ Service ปกติล้มเหลว
   */
  async sendFallbackMessage(replyToken, originalError) {
    const fallbackMessages = {
      default: 'ขออภัยครับ ระบบขัดข้องชั่วคราว กรุณาลองใหม่ภายหลัง',
      overload: 'ขณะนี้ระบบมีผู้ใช้งานจำนวนมาก กรุณารอสักครู่',
      maintenance: 'ระบบอยู่ระหว่างการบำรุงรักษา จะกลับมาให้บริการเร็วๆ นี้',
      database: 'ไม่สามารถเข้าถึงข้อมูลได้ในขณะนี้'
    };

    let message = fallbackMessages.default;

    if (originalError?.message?.includes('database')) {
      message = fallbackMessages.database;
    } else if (originalError?.statusCode === 503) {
      message = fallbackMessages.maintenance;
    }

    try {
      await lineClient.replyMessage(replyToken, {
        type: 'text',
        text: message
      });
    } catch (fallbackError) {
      // ถ้า Reply ก็ล้มเหลวด้วย ให้ Log เท่านั้น
      console.error('Fallback message ล้มเหลวด้วย:', fallbackError.message);
    }
  }

  /**
   * ตรวจสอบ Health ของ Dependencies
   */
  async checkDependencies() {
    const results = {};

    // ตรวจสอบ Database
    try {
      const db = require('../database/db');
      db.prepare('SELECT 1').get();
      results.database = { status: 'healthy' };
    } catch (error) {
      results.database = { status: 'unhealthy', error: error.message };
    }

    // ตรวจสอบ Redis
    try {
      const redis = require('../config/redis');
      await redis.ping();
      results.redis = { status: 'healthy' };
    } catch (error) {
      results.redis = { status: 'unhealthy', error: error.message };
    }

    // ตรวจสอบ LINE API
    try {
      const response = await lineClient.getBotInfo();
      results.lineApi = { status: 'healthy', botName: response.displayName };
    } catch (error) {
      results.lineApi = { status: 'unhealthy', error: error.message };
    }

    return results;
  }
}

module.exports = new FallbackService();
```

---

## 9. Python Error Handler

### 9.1 Python Error Classes

```python
# errors/line_errors.py
from datetime import datetime
from typing import Optional, Dict, Any


class LineError(Exception):
    """Base class สำหรับ LINE API Errors"""

    def __init__(
        self,
        message: str,
        status_code: int = 500,
        error_code: Optional[int] = None,
        details: Optional[Dict[str, Any]] = None
    ):
        super().__init__(message)
        self.status_code = status_code
        self.error_code = error_code
        self.details = details or {}
        self.timestamp = datetime.utcnow().isoformat()

    def to_dict(self) -> Dict[str, Any]:
        return {
            "name": self.__class__.__name__,
            "message": str(self),
            "status_code": self.status_code,
            "error_code": self.error_code,
            "details": self.details,
            "timestamp": self.timestamp
        }


class AuthenticationError(LineError):
    def __init__(self, message: str = "Authentication ล้มเหลว", error_code: int = None):
        super().__init__(message, 401, error_code)
        self.is_retryable = False


class RateLimitError(LineError):
    def __init__(self, retry_after: int = 60, details: Dict = None):
        super().__init__("เกิน Rate Limit", 429, 2001, details)
        self.retry_after = retry_after
        self.is_retryable = True


class TokenExpiredError(AuthenticationError):
    def __init__(self):
        super().__init__("Channel Access Token หมดอายุ", 1001)


class MessageTooLongError(LineError):
    def __init__(self, length: int, max_length: int = 5000):
        super().__init__(
            f"ข้อความยาว {length} ตัวอักษร เกินขีดจำกัด {max_length}",
            400,
            1300,
            {"length": length, "max_length": max_length}
        )
        self.is_retryable = False


class ReplyTokenExpiredError(LineError):
    def __init__(self):
        super().__init__("Reply Token หมดอายุหรือใช้ไปแล้ว", 400, 1302)
        self.is_retryable = False
```

### 9.2 Python Error Handler

```python
# middleware/error_handler.py
import logging
from functools import wraps
from typing import Callable, Any

from linebot.exceptions import LineBotApiError, InvalidSignatureError
from errors.line_errors import (
    LineError, RateLimitError, TokenExpiredError,
    MessageTooLongError, ReplyTokenExpiredError
)

logger = logging.getLogger(__name__)


def parse_line_api_error(error: LineBotApiError) -> LineError:
    """แปลง LineBotApiError เป็น Custom Error"""
    status_code = error.status_code
    message = str(error.message)

    if status_code == 401:
        return TokenExpiredError()
    elif status_code == 400:
        if "Too long" in message or "message" in message.lower():
            return MessageTooLongError(0, 5000)
        elif "reply token" in message.lower():
            return ReplyTokenExpiredError()
    elif status_code == 429:
        retry_after = int(error.headers.get("Retry-After", 60))
        return RateLimitError(retry_after)

    return LineError(message, status_code)


def handle_line_error(func: Callable) -> Callable:
    """Decorator สำหรับจัดการ LINE API Errors"""

    @wraps(func)
    async def wrapper(*args, **kwargs) -> Any:
        try:
            return await func(*args, **kwargs)
        except LineBotApiError as e:
            custom_error = parse_line_api_error(e)
            logger.error(
                f"LINE API Error: {custom_error}",
                extra={
                    "status_code": custom_error.status_code,
                    "error_code": custom_error.error_code,
                    "function": func.__name__
                }
            )
            raise custom_error
        except InvalidSignatureError:
            logger.warning("Webhook signature ไม่ถูกต้อง")
            raise
        except Exception as e:
            logger.error(
                f"Unexpected error in {func.__name__}: {str(e)}",
                exc_info=True
            )
            raise

    return wrapper


class RateLimitHandler:
    """จัดการ Rate Limit แบบ Automatic"""

    def __init__(self, max_retries: int = 3):
        self.max_retries = max_retries

    async def execute_with_retry(self, func: Callable, *args, **kwargs) -> Any:
        import asyncio

        for attempt in range(1, self.max_retries + 1):
            try:
                return await func(*args, **kwargs)
            except RateLimitError as e:
                if attempt == self.max_retries:
                    raise
                wait_time = e.retry_after
                logger.warning(f"Rate limit hit รอ {wait_time}s (attempt {attempt})")
                await asyncio.sleep(wait_time)
            except LineError as e:
                if not getattr(e, 'is_retryable', False):
                    raise
                if attempt == self.max_retries:
                    raise
                delay = min(2 ** attempt, 30)
                await asyncio.sleep(delay)
```

### 9.3 Python Flask Error Handler

```python
# app.py
import logging
from flask import Flask, jsonify, request
from linebot import WebhookParser
from linebot.exceptions import InvalidSignatureError, LineBotApiError

from errors.line_errors import LineError, RateLimitError
from middleware.error_handler import parse_line_api_error

app = Flask(__name__)
logger = logging.getLogger(__name__)

parser = WebhookParser(channel_secret=app.config['LINE_CHANNEL_SECRET'])


@app.errorhandler(InvalidSignatureError)
def handle_invalid_signature(e):
    logger.warning("Invalid webhook signature")
    return jsonify({"error": "Invalid signature"}), 401


@app.errorhandler(LineBotApiError)
def handle_line_api_error(e):
    custom_error = parse_line_api_error(e)
    logger.error(f"LINE API Error: {custom_error.to_dict()}")

    if isinstance(custom_error, RateLimitError):
        return jsonify(custom_error.to_dict()), 429, {
            "Retry-After": str(custom_error.retry_after)
        }

    return jsonify(custom_error.to_dict()), custom_error.status_code


@app.errorhandler(LineError)
def handle_line_error(e):
    logger.error(f"LINE Error: {e.to_dict()}")
    return jsonify(e.to_dict()), e.status_code


@app.errorhandler(Exception)
def handle_unexpected_error(e):
    logger.exception(f"Unexpected error: {str(e)}")
    return jsonify({
        "error": "Internal Server Error",
        "message": "เกิดข้อผิดพลาดภายใน"
    }), 500


@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('X-Line-Signature', '')
    body = request.get_data(as_text=True)

    try:
        events = parser.parse(body, signature)
    except InvalidSignatureError:
        raise

    for event in events:
        handle_event(event)

    return jsonify({"status": "ok"})


def handle_event(event):
    # Process event
    pass
```

---

## 10. Complete Error Handling System

### 10.1 Error Monitoring Dashboard

```javascript
// routes/error-dashboard.js
const express = require('express');
const router = express.Router();
const redis = require('../config/redis');

// บันทึก Error Statistics
async function recordError(errorType, details) {
  const date = new Date().toISOString().slice(0, 10);
  const pipe = redis.pipeline();

  pipe.incr(`errors:total:${date}`);
  pipe.incr(`errors:${errorType}:${date}`);
  pipe.lpush('errors:recent', JSON.stringify({
    type: errorType,
    ...details,
    timestamp: new Date().toISOString()
  }));
  pipe.ltrim('errors:recent', 0, 99); // เก็บ 100 errors ล่าสุด

  await pipe.exec();
}

router.get('/errors/stats', async (req, res) => {
  const { days = 7 } = req.query;
  const stats = {};

  for (let i = 0; i < parseInt(days); i++) {
    const date = new Date();
    date.setDate(date.getDate() - i);
    const dateStr = date.toISOString().slice(0, 10);

    const total = await redis.get(`errors:total:${dateStr}`);
    stats[dateStr] = { total: parseInt(total) || 0 };
  }

  const recentErrors = await redis.lrange('errors:recent', 0, 9);
  res.json({
    stats,
    recentErrors: recentErrors.map(e => JSON.parse(e))
  });
});

module.exports = { router, recordError };
```

---

## 11. Testing Error Handling

### 11.1 Node.js Error Tests

```javascript
// tests/error-handling.test.js
const LineApiErrorParser = require('../utils/error-parser');
const {
  MessageTooLongError,
  RateLimitError,
  TokenExpiredError,
  ReplyTokenExpiredError
} = require('../errors/line-errors');

describe('LineApiErrorParser', () => {
  test('ควรแปลง 400 Too Long เป็น MessageTooLongError', () => {
    const mockError = {
      response: {
        status: 400,
        data: { message: 'The text is Too long. max: 5000, actual: 5001' }
      }
    };

    const result = LineApiErrorParser.parse(mockError);
    expect(result).toBeInstanceOf(MessageTooLongError);
    expect(result.statusCode).toBe(400);
    expect(result.isRetryable).toBe(false);
  });

  test('ควรแปลง 401 เป็น TokenExpiredError', () => {
    const mockError = {
      response: {
        status: 401,
        data: { error: { code: 1001 } }
      }
    };

    const result = LineApiErrorParser.parse(mockError);
    expect(result).toBeInstanceOf(TokenExpiredError);
    expect(result.isRetryable).toBe(false);
  });

  test('ควรแปลง 429 เป็น RateLimitError', () => {
    const mockError = {
      response: {
        status: 429,
        headers: { 'retry-after': '30' },
        data: { message: 'Too Many Requests' }
      }
    };

    const result = LineApiErrorParser.parse(mockError);
    expect(result).toBeInstanceOf(RateLimitError);
    expect(result.retryAfter).toBe(30);
    expect(result.isRetryable).toBe(true);
  });

  test('ควรแปลง Network Error', () => {
    const mockError = {
      code: 'ECONNABORTED',
      message: 'timeout of 5000ms exceeded'
    };

    const result = LineApiErrorParser.parse(mockError);
    expect(result.name).toBe('NetworkError');
    expect(result.isRetryable).toBe(true);
  });

  test('isRetryable ควรคืนค่า false สำหรับ Non-retryable errors', () => {
    const error = new MessageTooLongError(5001);
    expect(LineApiErrorParser.isRetryable(error)).toBe(false);
  });

  test('isRetryable ควรคืนค่า true สำหรับ Rate Limit errors', () => {
    const error = new RateLimitError(60);
    expect(LineApiErrorParser.isRetryable(error)).toBe(true);
  });
});
```

### 11.2 Python Tests

```python
# tests/test_error_handler.py
import pytest
import asyncio
from unittest.mock import AsyncMock, patch, MagicMock
from linebot.exceptions import LineBotApiError

from errors.line_errors import (
    TokenExpiredError, RateLimitError,
    MessageTooLongError, ReplyTokenExpiredError
)
from middleware.error_handler import parse_line_api_error, handle_line_error


class TestParseLineApiError:
    def test_parse_401_returns_token_expired(self):
        """ควรแปลง 401 เป็น TokenExpiredError"""
        mock_error = MagicMock(spec=LineBotApiError)
        mock_error.status_code = 401
        mock_error.message = "Unauthorized"

        result = parse_line_api_error(mock_error)
        assert isinstance(result, TokenExpiredError)
        assert result.is_retryable == False

    def test_parse_429_returns_rate_limit(self):
        """ควรแปลง 429 เป็น RateLimitError"""
        mock_error = MagicMock(spec=LineBotApiError)
        mock_error.status_code = 429
        mock_error.message = "Too Many Requests"
        mock_error.headers = {"Retry-After": "30"}

        result = parse_line_api_error(mock_error)
        assert isinstance(result, RateLimitError)
        assert result.retry_after == 30
        assert result.is_retryable == True

    def test_parse_400_too_long(self):
        """ควรแปลง 400 Too Long เป็น MessageTooLongError"""
        mock_error = MagicMock(spec=LineBotApiError)
        mock_error.status_code = 400
        mock_error.message = "The text message is Too long"

        result = parse_line_api_error(mock_error)
        assert isinstance(result, MessageTooLongError)


class TestRateLimitHandler:
    @pytest.mark.asyncio
    async def test_retries_on_rate_limit(self):
        """ควร Retry หลังจาก Rate Limit"""
        from middleware.error_handler import RateLimitHandler

        call_count = 0

        async def flaky_func():
            nonlocal call_count
            call_count += 1
            if call_count < 3:
                raise RateLimitError(retry_after=0.01)
            return "success"

        handler = RateLimitHandler(max_retries=3)
        result = await handler.execute_with_retry(flaky_func)
        assert result == "success"
        assert call_count == 3
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Error Codes** - รหัสข้อผิดพลาดของ LINE API ทั้งหมด
2. **Custom Errors** - สร้าง Error Classes ที่มีความหมาย
3. **Error Parser** - แปลง API Errors เป็น Custom Errors
4. **Rate Limit Handling** - จัดการการ Retry อย่างถูกต้อง
5. **Token Management** - จัดการ Token ที่หมดอายุ
6. **Structured Logging** - บันทึก Log อย่างเป็นระบบ
7. **Alert System** - แจ้งเตือนทีมเมื่อเกิดปัญหา
8. **Graceful Degradation** - ล้มเหลวอย่างสง่างาม
9. **Python Error Handling** - การจัดการ Error ใน Python
10. **Testing** - ทดสอบ Error Handling อย่างครอบคลุม

ขั้นตอนต่อไป: ใน Part 59 เราจะเรียนรู้เรื่อง Testing LINE Bots อย่างครบวงจร
