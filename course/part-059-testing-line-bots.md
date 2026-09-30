# Part 59: Testing LINE Bots - การทดสอบบอทอย่างครบวงจร

## บทนำ

การทดสอบ LINE Bot อย่างครอบคลุมเป็นสิ่งสำคัญมากในการพัฒนาระบบที่เชื่อถือได้ บทนี้ครอบคลุมทุกระดับของการทดสอบตั้งแต่ Unit Tests, Integration Tests, End-to-End Tests ไปจนถึง Load Testing

---

## 1. โครงสร้างการทดสอบ

```
tests/
├── unit/
│   ├── handlers/
│   │   ├── message-handler.test.js
│   │   ├── follow-handler.test.js
│   │   └── postback-handler.test.js
│   ├── services/
│   │   ├── reply-service.test.js
│   │   └── user-service.test.js
│   └── utils/
│       ├── message-validator.test.js
│       └── flex-builder.test.js
├── integration/
│   ├── webhook.test.js
│   ├── api-client.test.js
│   └── database.test.js
├── e2e/
│   └── bot-flows.test.js
├── fixtures/
│   ├── events/
│   │   ├── text-message.json
│   │   ├── follow-event.json
│   │   ├── postback-event.json
│   │   └── image-message.json
│   └── responses/
│       └── api-responses.json
├── mocks/
│   ├── line-client.mock.js
│   └── redis.mock.js
└── setup/
    ├── jest.setup.js
    └── test-db.js
```

---

## 2. Jest Setup สำหรับ LINE Bot

### 2.1 การติดตั้ง

```bash
npm install --save-dev jest @types/jest supertest nock jest-extended
```

### 2.2 jest.config.js

```javascript
// jest.config.js
module.exports = {
  testEnvironment: 'node',
  testMatch: ['**/__tests__/**/*.js', '**/*.test.js'],
  collectCoverageFrom: [
    'src/**/*.js',
    '!src/index.js',
    '!**/node_modules/**'
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80
    }
  },
  setupFilesAfterFramework: ['./tests/setup/jest.setup.js'],
  testTimeout: 10000,
  clearMocks: true,
  restoreMocks: true
};
```

### 2.3 jest.setup.js

```javascript
// tests/setup/jest.setup.js
require('dotenv').config({ path: '.env.test' });

// ตั้งค่า Environment Variables สำหรับการทดสอบ
process.env.LINE_CHANNEL_ACCESS_TOKEN = 'test-access-token';
process.env.LINE_CHANNEL_SECRET = 'test-channel-secret';
process.env.NODE_ENV = 'test';

// Mock console.error เพื่อป้องกัน noise ใน tests
const originalConsoleError = console.error;
beforeAll(() => {
  console.error = jest.fn();
});

afterAll(() => {
  console.error = originalConsoleError;
});

// Extend Jest matchers
expect.extend({
  toBeValidLineMessage(received) {
    const validTypes = ['text', 'image', 'video', 'audio', 'location',
                        'sticker', 'flex', 'template'];
    const pass = received && validTypes.includes(received.type);
    return {
      pass,
      message: () => pass
        ? `ข้อความ "${received.type}" ควรไม่ใช่ LINE Message ที่ถูกต้อง`
        : `ข้อความ "${received?.type}" ไม่ใช่ LINE Message ที่ถูกต้อง (ควรเป็น ${validTypes.join(', ')})`
    };
  }
});
```

---

## 3. Test Fixtures and Factories

### 3.1 Event Fixtures

```javascript
// tests/fixtures/events/text-message.json
{
  "destination": "Uc8166f3e4f74429573a529316aef37f",
  "events": [
    {
      "type": "message",
      "message": {
        "type": "text",
        "id": "14353798921116",
        "text": "สวัสดี"
      },
      "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
      "deliveryContext": {
        "isRedelivery": false
      },
      "timestamp": 1625665242211,
      "source": {
        "type": "user",
        "userId": "U80696558e1aa831fa6c011bd6cdb8b"
      },
      "replyToken": "757913772c4a4b12bb54eb59ce2bf43f",
      "mode": "active"
    }
  ]
}
```

```javascript
// tests/fixtures/events/follow-event.json
{
  "destination": "Uc8166f3e4f74429573a529316aef37f",
  "events": [
    {
      "type": "follow",
      "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZS",
      "deliveryContext": {
        "isRedelivery": false
      },
      "timestamp": 1625665242211,
      "source": {
        "type": "user",
        "userId": "U80696558e1aa831fa6c011bd6cdb8b"
      },
      "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA",
      "mode": "active"
    }
  ]
}
```

### 3.2 Event Factory

```javascript
// tests/factories/event-factory.js
const { v4: uuidv4 } = require('uuid');

class EventFactory {
  static createTextMessageEvent(options = {}) {
    return {
      type: 'message',
      message: {
        type: 'text',
        id: options.messageId || String(Date.now()),
        text: options.text || 'สวัสดีครับ'
      },
      webhookEventId: options.eventId || uuidv4().replace(/-/g, '').toUpperCase().slice(0, 26),
      deliveryContext: {
        isRedelivery: options.isRedelivery || false
      },
      timestamp: options.timestamp || Date.now(),
      source: {
        type: options.sourceType || 'user',
        userId: options.userId || `U${uuidv4().replace(/-/g, '').slice(0, 32)}`
      },
      replyToken: options.replyToken || uuidv4().replace(/-/g, ''),
      mode: options.mode || 'active'
    };
  }

  static createFollowEvent(options = {}) {
    return {
      type: 'follow',
      webhookEventId: uuidv4().replace(/-/g, '').toUpperCase().slice(0, 26),
      deliveryContext: { isRedelivery: false },
      timestamp: Date.now(),
      source: {
        type: 'user',
        userId: options.userId || `U${uuidv4().replace(/-/g, '').slice(0, 32)}`
      },
      replyToken: uuidv4().replace(/-/g, ''),
      mode: 'active'
    };
  }

  static createPostbackEvent(options = {}) {
    return {
      type: 'postback',
      webhookEventId: uuidv4().replace(/-/g, '').toUpperCase().slice(0, 26),
      deliveryContext: { isRedelivery: false },
      timestamp: Date.now(),
      source: {
        type: 'user',
        userId: options.userId || `U${uuidv4().replace(/-/g, '').slice(0, 32)}`
      },
      replyToken: uuidv4().replace(/-/g, ''),
      postback: {
        data: options.data || 'action=buy&productId=123',
        params: options.params || {}
      },
      mode: 'active'
    };
  }

  static createImageMessageEvent(options = {}) {
    return {
      type: 'message',
      message: {
        type: 'image',
        id: options.messageId || String(Date.now()),
        contentProvider: {
          type: 'line'
        }
      },
      webhookEventId: uuidv4().replace(/-/g, '').toUpperCase().slice(0, 26),
      deliveryContext: { isRedelivery: false },
      timestamp: Date.now(),
      source: {
        type: 'user',
        userId: options.userId || `U${uuidv4().replace(/-/g, '').slice(0, 32)}`
      },
      replyToken: uuidv4().replace(/-/g, ''),
      mode: 'active'
    };
  }

  static createStickerMessageEvent(options = {}) {
    return {
      type: 'message',
      message: {
        type: 'sticker',
        id: String(Date.now()),
        packageId: options.packageId || '1',
        stickerId: options.stickerId || '1',
        stickerResourceType: 'STATIC'
      },
      webhookEventId: uuidv4().replace(/-/g, '').toUpperCase().slice(0, 26),
      deliveryContext: { isRedelivery: false },
      timestamp: Date.now(),
      source: {
        type: 'user',
        userId: options.userId || `U${uuidv4().replace(/-/g, '').slice(0, 32)}`
      },
      replyToken: uuidv4().replace(/-/g, ''),
      mode: 'active'
    };
  }

  static createWebhookPayload(events, destination = 'Uc8166f3e4f74429573a529316aef37f') {
    return { destination, events };
  }
}

module.exports = EventFactory;
```

---

## 4. Mock LINE API Client

### 4.1 LINE Client Mock

```javascript
// tests/mocks/line-client.mock.js

class MockLineClient {
  constructor() {
    this.sentMessages = [];
    this.pushedMessages = [];
    this.broadcastMessages = [];
    this.multicastMessages = [];
    this.profiles = new Map();

    // Mock methods
    this.replyMessage = jest.fn(async (replyToken, messages) => {
      const messageArray = Array.isArray(messages) ? messages : [messages];
      this.sentMessages.push({ replyToken, messages: messageArray });
      return { requestId: 'test-request-id' };
    });

    this.pushMessage = jest.fn(async (userId, messages) => {
      const messageArray = Array.isArray(messages) ? messages : [messages];
      this.pushedMessages.push({ userId, messages: messageArray });
      return { requestId: 'test-request-id' };
    });

    this.broadcastMessage = jest.fn(async (messages) => {
      const messageArray = Array.isArray(messages) ? messages : [messages];
      this.broadcastMessages.push({ messages: messageArray });
      return { requestId: 'test-request-id' };
    });

    this.multicastMessage = jest.fn(async (userIds, messages) => {
      const messageArray = Array.isArray(messages) ? messages : [messages];
      this.multicastMessages.push({ userIds, messages: messageArray });
      return { requestId: 'test-request-id' };
    });

    this.getProfile = jest.fn(async (userId) => {
      if (this.profiles.has(userId)) {
        return this.profiles.get(userId);
      }
      return {
        userId,
        displayName: 'Test User',
        pictureUrl: 'https://example.com/pic.jpg',
        statusMessage: 'Test status'
      };
    });

    this.getBotInfo = jest.fn(async () => ({
      userId: 'Uc8166f3e4f74429573a529316aef37f',
      displayName: 'Test Bot',
      pictureUrl: 'https://example.com/bot.jpg',
      chatMode: 'bot',
      markAsReadMode: 'auto'
    }));
  }

  // Helper methods สำหรับ assertions
  getLastSentMessage() {
    return this.sentMessages[this.sentMessages.length - 1];
  }

  getAllSentMessages() {
    return this.sentMessages;
  }

  clearAll() {
    this.sentMessages = [];
    this.pushedMessages = [];
    this.broadcastMessages = [];
    this.multicastMessages = [];
  }

  addMockProfile(userId, profile) {
    this.profiles.set(userId, { userId, ...profile });
  }
}

module.exports = MockLineClient;
```

---

## 5. Unit Testing Bot Handlers

### 5.1 Text Message Handler Test

```javascript
// tests/unit/handlers/message-handler.test.js
const EventFactory = require('../../factories/event-factory');
const MockLineClient = require('../../mocks/line-client.mock');
const MessageHandler = require('../../../src/handlers/message-handler');

describe('MessageHandler', () => {
  let handler;
  let mockClient;

  beforeEach(() => {
    mockClient = new MockLineClient();
    handler = new MessageHandler(mockClient);
  });

  afterEach(() => {
    mockClient.clearAll();
  });

  describe('handleTextMessage', () => {
    test('ควรตอบสวัสดีเมื่อรับข้อความ "สวัสดี"', async () => {
      const event = EventFactory.createTextMessageEvent({ text: 'สวัสดี' });

      await handler.handle(event);

      const lastMessage = mockClient.getLastSentMessage();
      expect(lastMessage).toBeDefined();
      expect(lastMessage.replyToken).toBe(event.replyToken);
      expect(lastMessage.messages[0].type).toBe('text');
      expect(lastMessage.messages[0].text).toContain('สวัสดี');
    });

    test('ควรตอบข้อความ Help เมื่อพิมพ์ "ช่วยเหลือ"', async () => {
      const event = EventFactory.createTextMessageEvent({ text: 'ช่วยเหลือ' });

      await handler.handle(event);

      const lastMessage = mockClient.getLastSentMessage();
      expect(lastMessage.messages[0].text).toContain('ช่วยเหลือ');
    });

    test('ควรตอบข้อความ Default เมื่อไม่รู้จัก', async () => {
      const event = EventFactory.createTextMessageEvent({
        text: 'ข้อความที่ไม่รู้จัก xyz123'
      });

      await handler.handle(event);

      expect(mockClient.replyMessage).toHaveBeenCalledTimes(1);
      const lastMessage = mockClient.getLastSentMessage();
      expect(lastMessage.messages[0].type).toBe('text');
    });

    test('ควร Reply ด้วย Flex Message สำหรับ Menu', async () => {
      const event = EventFactory.createTextMessageEvent({ text: 'เมนู' });

      await handler.handle(event);

      const lastMessage = mockClient.getLastSentMessage();
      expect(lastMessage.messages[0].type).toBe('flex');
    });

    test('ควรไม่ตอบเมื่อ source เป็น bot', async () => {
      const event = EventFactory.createTextMessageEvent({ text: 'สวัสดี' });
      event.source.type = 'bot';

      await handler.handle(event);

      expect(mockClient.replyMessage).not.toHaveBeenCalled();
    });
  });

  describe('handleImageMessage', () => {
    test('ควรตอบว่าได้รับรูปภาพ', async () => {
      const event = EventFactory.createImageMessageEvent();

      await handler.handle(event);

      const lastMessage = mockClient.getLastSentMessage();
      expect(lastMessage.messages[0].text).toContain('รูปภาพ');
    });
  });

  describe('handleStickerMessage', () => {
    test('ควรตอบด้วย Sticker', async () => {
      const event = EventFactory.createStickerMessageEvent({
        packageId: '1',
        stickerId: '1'
      });

      await handler.handle(event);

      const lastMessage = mockClient.getLastSentMessage();
      expect(lastMessage.messages[0].type).toBe('sticker');
    });
  });
});
```

### 5.2 Follow Handler Test

```javascript
// tests/unit/handlers/follow-handler.test.js
const EventFactory = require('../../factories/event-factory');
const MockLineClient = require('../../mocks/line-client.mock');
const FollowHandler = require('../../../src/handlers/follow-handler');
const UserService = require('../../../src/services/user-service');

jest.mock('../../../src/services/user-service');

describe('FollowHandler', () => {
  let handler;
  let mockClient;

  beforeEach(() => {
    mockClient = new MockLineClient();
    handler = new FollowHandler(mockClient);
    UserService.createUser = jest.fn().mockResolvedValue({
      id: 1,
      lineUserId: 'U123',
      displayName: 'Test User'
    });
  });

  test('ควรส่งข้อความยินดีต้อนรับเมื่อมีผู้ติดตามใหม่', async () => {
    const event = EventFactory.createFollowEvent();
    mockClient.addMockProfile(event.source.userId, {
      displayName: 'สมชาย ใจดี'
    });

    await handler.handle(event);

    expect(mockClient.replyMessage).toHaveBeenCalledTimes(1);
    const lastMessage = mockClient.getLastSentMessage();
    expect(lastMessage.messages[0].type).toBe('text');
    expect(lastMessage.messages[0].text).toContain('ยินดีต้อนรับ');
  });

  test('ควรสร้าง User ในระบบเมื่อมีผู้ติดตามใหม่', async () => {
    const event = EventFactory.createFollowEvent();

    await handler.handle(event);

    expect(UserService.createUser).toHaveBeenCalledWith(
      expect.objectContaining({ lineUserId: event.source.userId })
    );
  });

  test('ควรส่ง Flex Welcome Message', async () => {
    const event = EventFactory.createFollowEvent();

    await handler.handle(event);

    const messages = mockClient.getLastSentMessage().messages;
    const flexMessage = messages.find(m => m.type === 'flex');
    expect(flexMessage).toBeDefined();
  });
});
```

### 5.3 Postback Handler Test

```javascript
// tests/unit/handlers/postback-handler.test.js
const EventFactory = require('../../factories/event-factory');
const MockLineClient = require('../../mocks/line-client.mock');
const PostbackHandler = require('../../../src/handlers/postback-handler');

describe('PostbackHandler', () => {
  let handler;
  let mockClient;

  beforeEach(() => {
    mockClient = new MockLineClient();
    handler = new PostbackHandler(mockClient);
  });

  test('ควรจัดการ action=buy', async () => {
    const event = EventFactory.createPostbackEvent({
      data: 'action=buy&productId=PROD001'
    });

    await handler.handle(event);

    expect(mockClient.replyMessage).toHaveBeenCalledTimes(1);
    const lastMessage = mockClient.getLastSentMessage();
    expect(lastMessage.messages[0].text).toContain('PROD001');
  });

  test('ควรจัดการ action=menu', async () => {
    const event = EventFactory.createPostbackEvent({ data: 'action=menu' });

    await handler.handle(event);

    const lastMessage = mockClient.getLastSentMessage();
    expect(lastMessage.messages[0].type).toBe('flex');
  });

  test('ควรตอบ error เมื่อ action ไม่รู้จัก', async () => {
    const event = EventFactory.createPostbackEvent({
      data: 'action=unknown_action'
    });

    await handler.handle(event);

    expect(mockClient.replyMessage).toHaveBeenCalledTimes(1);
  });

  test('ควรแปลง URLSearchParams จาก postback data', async () => {
    const event = EventFactory.createPostbackEvent({
      data: 'action=buy&productId=123&quantity=2&color=red'
    });

    const params = new URLSearchParams(event.postback.data);
    expect(params.get('action')).toBe('buy');
    expect(params.get('productId')).toBe('123');
    expect(params.get('quantity')).toBe('2');
  });
});
```

---

## 6. Integration Testing

### 6.1 Webhook Integration Test

```javascript
// tests/integration/webhook.test.js
const request = require('supertest');
const crypto = require('crypto');
const app = require('../../src/app');
const EventFactory = require('../factories/event-factory');

const CHANNEL_SECRET = 'test-channel-secret';

function generateSignature(body) {
  return crypto
    .createHmac('sha256', CHANNEL_SECRET)
    .update(body)
    .digest('base64');
}

describe('Webhook Integration Tests', () => {
  describe('POST /webhook', () => {
    test('ควรตอบ 200 เมื่อ Signature ถูกต้อง', async () => {
      const payload = EventFactory.createWebhookPayload([
        EventFactory.createTextMessageEvent()
      ]);
      const body = JSON.stringify(payload);
      const signature = generateSignature(body);

      const res = await request(app)
        .post('/webhook')
        .set('x-line-signature', signature)
        .set('content-type', 'application/json')
        .send(body);

      expect(res.status).toBe(200);
    });

    test('ควรตอบ 401 เมื่อไม่มี Signature', async () => {
      const res = await request(app)
        .post('/webhook')
        .send({ events: [] });

      expect(res.status).toBe(401);
    });

    test('ควรตอบ 401 เมื่อ Signature ไม่ถูกต้อง', async () => {
      const payload = EventFactory.createWebhookPayload([]);
      const res = await request(app)
        .post('/webhook')
        .set('x-line-signature', 'invalid-sig')
        .send(payload);

      expect(res.status).toBe(401);
    });

    test('ควรรับ Webhook หลาย Events พร้อมกัน', async () => {
      const events = [
        EventFactory.createTextMessageEvent({ text: 'test1' }),
        EventFactory.createTextMessageEvent({ text: 'test2' }),
        EventFactory.createFollowEvent()
      ];
      const payload = EventFactory.createWebhookPayload(events);
      const body = JSON.stringify(payload);
      const signature = generateSignature(body);

      const res = await request(app)
        .post('/webhook')
        .set('x-line-signature', signature)
        .set('content-type', 'application/json')
        .send(body);

      expect(res.status).toBe(200);
    });

    test('ควรรองรับ Empty Events', async () => {
      const payload = EventFactory.createWebhookPayload([]);
      const body = JSON.stringify(payload);
      const signature = generateSignature(body);

      const res = await request(app)
        .post('/webhook')
        .set('x-line-signature', signature)
        .set('content-type', 'application/json')
        .send(body);

      expect(res.status).toBe(200);
    });
  });
});
```

---

## 7. Testing Flex Messages

### 7.1 Flex Message Builder Test

```javascript
// tests/unit/utils/flex-builder.test.js
const FlexMessageBuilder = require('../../../src/utils/flex-builder');

describe('FlexMessageBuilder', () => {
  describe('buildProductCard', () => {
    const product = {
      id: 'PROD001',
      name: 'สินค้าทดสอบ',
      price: 299,
      image: 'https://example.com/product.jpg',
      description: 'คำอธิบายสินค้า'
    };

    test('ควรสร้าง Flex Message ที่ถูกต้อง', () => {
      const flex = FlexMessageBuilder.buildProductCard(product);

      expect(flex.type).toBe('flex');
      expect(flex.altText).toBeDefined();
      expect(flex.contents).toBeDefined();
      expect(flex.contents.type).toBe('bubble');
    });

    test('ควรมีชื่อสินค้าใน Flex Message', () => {
      const flex = FlexMessageBuilder.buildProductCard(product);
      const flexStr = JSON.stringify(flex);

      expect(flexStr).toContain(product.name);
    });

    test('ควรมีราคาสินค้าใน Flex Message', () => {
      const flex = FlexMessageBuilder.buildProductCard(product);
      const flexStr = JSON.stringify(flex);

      expect(flexStr).toContain('299');
    });

    test('ควรมีปุ่ม Buy ที่มี Postback', () => {
      const flex = FlexMessageBuilder.buildProductCard(product);
      const flexStr = JSON.stringify(flex);

      expect(flexStr).toContain('action=buy');
      expect(flexStr).toContain(product.id);
    });

    test('ควรมี Image URL ถูกต้อง', () => {
      const flex = FlexMessageBuilder.buildProductCard(product);
      const flexStr = JSON.stringify(flex);

      expect(flexStr).toContain(product.image);
    });
  });

  describe('buildCarousel', () => {
    test('ควรสร้าง Carousel ได้สูงสุด 12 items', () => {
      const items = Array.from({ length: 15 }, (_, i) => ({
        id: `PROD${i}`,
        name: `สินค้า ${i}`,
        price: 100 + i,
        image: `https://example.com/product${i}.jpg`
      }));

      const carousel = FlexMessageBuilder.buildCarousel(items, 'สินค้าทั้งหมด');
      expect(carousel.contents.contents.length).toBeLessThanOrEqual(12);
    });

    test('ควรสร้าง Carousel type ถูกต้อง', () => {
      const items = [{ id: '1', name: 'Test', price: 100, image: 'https://test.com/img.jpg' }];
      const carousel = FlexMessageBuilder.buildCarousel(items, 'Test');

      expect(carousel.type).toBe('flex');
      expect(carousel.contents.type).toBe('carousel');
    });
  });

  describe('validateFlexMessage', () => {
    test('ควรผ่านการ Validate Flex Message ที่ถูกต้อง', () => {
      const validFlex = {
        type: 'flex',
        altText: 'Test',
        contents: {
          type: 'bubble',
          body: {
            type: 'box',
            layout: 'vertical',
            contents: []
          }
        }
      };

      expect(() => FlexMessageBuilder.validate(validFlex)).not.toThrow();
    });

    test('ควร Throw error เมื่อ altText หายไป', () => {
      const invalidFlex = {
        type: 'flex',
        contents: { type: 'bubble' }
      };

      expect(() => FlexMessageBuilder.validate(invalidFlex))
        .toThrow('altText เป็นสิ่งจำเป็น');
    });
  });
});
```

---

## 8. Python Pytest สำหรับ LINE Bot

### 8.1 Conftest.py

```python
# tests/conftest.py
import pytest
import json
import os
from unittest.mock import AsyncMock, MagicMock, patch
from linebot import LineBotApi, WebhookParser
from linebot.models import TextMessage, TextSendMessage


# ตั้งค่า Environment Variables สำหรับ Tests
@pytest.fixture(autouse=True)
def set_env_vars():
    os.environ['LINE_CHANNEL_ACCESS_TOKEN'] = 'test-access-token'
    os.environ['LINE_CHANNEL_SECRET'] = 'test-channel-secret'
    yield
    # Cleanup
    for key in ['LINE_CHANNEL_ACCESS_TOKEN', 'LINE_CHANNEL_SECRET']:
        os.environ.pop(key, None)


@pytest.fixture
def mock_line_bot_api():
    """Mock สำหรับ LineBotApi"""
    with patch('linebot.LineBotApi') as mock:
        api = MagicMock(spec=LineBotApi)
        api.reply_message = AsyncMock(return_value=None)
        api.push_message = AsyncMock(return_value=None)
        api.get_profile = AsyncMock(return_value=MagicMock(
            user_id='U123',
            display_name='Test User',
            picture_url='https://example.com/pic.jpg'
        ))
        mock.return_value = api
        yield api


@pytest.fixture
def text_message_event():
    """Fixture สำหรับ Text Message Event"""
    return {
        "type": "message",
        "message": {
            "type": "text",
            "id": "14353798921116",
            "text": "สวัสดี"
        },
        "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZR",
        "timestamp": 1625665242211,
        "source": {
            "type": "user",
            "userId": "U80696558e1aa831fa6c011bd6cdb8b"
        },
        "replyToken": "757913772c4a4b12bb54eb59ce2bf43f",
        "mode": "active"
    }


@pytest.fixture
def follow_event():
    """Fixture สำหรับ Follow Event"""
    return {
        "type": "follow",
        "webhookEventId": "01FZ74A0TDDPYRVKNK77XKC3ZS",
        "timestamp": 1625665242211,
        "source": {
            "type": "user",
            "userId": "U80696558e1aa831fa6c011bd6cdb8b"
        },
        "replyToken": "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA",
        "mode": "active"
    }
```

### 8.2 Python Bot Handler Tests

```python
# tests/test_message_handler.py
import pytest
from unittest.mock import AsyncMock, MagicMock, call
from linebot.models import TextSendMessage, StickerSendMessage


class TestMessageHandler:
    """ทดสอบ Message Handler"""

    @pytest.mark.asyncio
    async def test_reply_greeting_to_hello(self, mock_line_bot_api, text_message_event):
        """ควรตอบสวัสดีเมื่อรับข้อความ 'สวัสดี'"""
        from src.handlers.message_handler import MessageHandler

        handler = MessageHandler(mock_line_bot_api)
        text_message_event['message']['text'] = 'สวัสดี'

        await handler.handle(text_message_event)

        mock_line_bot_api.reply_message.assert_called_once()
        call_args = mock_line_bot_api.reply_message.call_args
        reply_token = call_args[0][0]
        message = call_args[0][1]

        assert reply_token == text_message_event['replyToken']
        assert isinstance(message, TextSendMessage)
        assert 'สวัสดี' in message.text

    @pytest.mark.asyncio
    async def test_no_reply_for_empty_text(self, mock_line_bot_api, text_message_event):
        """ไม่ควรตอบเมื่อข้อความว่าง"""
        from src.handlers.message_handler import MessageHandler

        handler = MessageHandler(mock_line_bot_api)
        text_message_event['message']['text'] = ''

        await handler.handle(text_message_event)

        mock_line_bot_api.reply_message.assert_not_called()

    @pytest.mark.asyncio
    async def test_handle_sticker_message(self, mock_line_bot_api):
        """ควรตอบด้วย Sticker เมื่อรับ Sticker"""
        from src.handlers.message_handler import MessageHandler

        handler = MessageHandler(mock_line_bot_api)
        sticker_event = {
            "type": "message",
            "message": {
                "type": "sticker",
                "id": "12345",
                "packageId": "1",
                "stickerId": "2"
            },
            "replyToken": "test-reply-token",
            "source": {"type": "user", "userId": "U123"},
            "timestamp": 1625665242211
        }

        await handler.handle(sticker_event)

        mock_line_bot_api.reply_message.assert_called_once()


class TestFollowHandler:
    """ทดสอบ Follow Handler"""

    @pytest.mark.asyncio
    async def test_sends_welcome_on_follow(self, mock_line_bot_api, follow_event):
        """ควรส่งข้อความยินดีต้อนรับเมื่อมีผู้ติดตามใหม่"""
        from src.handlers.follow_handler import FollowHandler

        handler = FollowHandler(mock_line_bot_api)
        await handler.handle(follow_event)

        mock_line_bot_api.reply_message.assert_called_once()
        message = mock_line_bot_api.reply_message.call_args[0][1]
        assert 'ยินดีต้อนรับ' in message.text

    @pytest.mark.asyncio
    async def test_fetches_user_profile_on_follow(self, mock_line_bot_api, follow_event):
        """ควรดึงข้อมูล Profile ของผู้ใช้เมื่อติดตาม"""
        from src.handlers.follow_handler import FollowHandler

        handler = FollowHandler(mock_line_bot_api)
        await handler.handle(follow_event)

        mock_line_bot_api.get_profile.assert_called_once_with(
            follow_event['source']['userId']
        )
```

---

## 9. Testing Message Flows (ทดสอบ Flow การสนทนา)

### 9.1 Conversation Flow Tests

```javascript
// tests/e2e/bot-flows.test.js
const MockLineClient = require('../mocks/line-client.mock');
const EventFactory = require('../factories/event-factory');
const BotApp = require('../../src/bot');

describe('Bot Conversation Flows', () => {
  let bot;
  let mockClient;
  const userId = 'U80696558e1aa831fa6c011bd6cdb8b';

  beforeEach(() => {
    mockClient = new MockLineClient();
    bot = new BotApp(mockClient);
  });

  describe('Order Flow', () => {
    test('สามารถสั่งซื้อสินค้าได้ครบ Flow', async () => {
      // Step 1: เริ่มต้นด้วยข้อความ "สั่งซื้อ"
      const step1 = EventFactory.createTextMessageEvent({
        userId,
        text: 'สั่งซื้อ'
      });
      await bot.processEvent(step1);

      let lastMsg = mockClient.getLastSentMessage();
      expect(lastMsg.messages[0].type).toBe('flex'); // แสดงเมนูสินค้า
      mockClient.clearAll();

      // Step 2: เลือกสินค้าด้วย Postback
      const step2 = EventFactory.createPostbackEvent({
        userId,
        data: 'action=addToCart&productId=PROD001'
      });
      await bot.processEvent(step2);

      lastMsg = mockClient.getLastSentMessage();
      expect(lastMsg.messages[0].text).toContain('เพิ่มสินค้าแล้ว');
      mockClient.clearAll();

      // Step 3: ยืนยันการสั่งซื้อ
      const step3 = EventFactory.createPostbackEvent({
        userId,
        data: 'action=checkout'
      });
      await bot.processEvent(step3);

      lastMsg = mockClient.getLastSentMessage();
      expect(lastMsg.messages[0].text).toContain('สั่งซื้อสำเร็จ');
    });
  });

  describe('FAQ Flow', () => {
    const faqs = [
      { question: 'เวลาทำการ', expectedKeyword: 'จันทร์' },
      { question: 'ที่ตั้ง', expectedKeyword: 'แผนที่' },
      { question: 'ราคา', expectedKeyword: 'บาท' }
    ];

    faqs.forEach(({ question, expectedKeyword }) => {
      test(`ควรตอบคำถาม "${question}"`, async () => {
        const event = EventFactory.createTextMessageEvent({ userId, text: question });
        await bot.processEvent(event);

        const lastMsg = mockClient.getLastSentMessage();
        const reply = JSON.stringify(lastMsg.messages);
        expect(reply).toContain(expectedKeyword);
      });
    });
  });
});
```

---

## 10. Load Testing (การทดสอบภายใต้โหลดสูง)

### 10.1 Artillery Config

```yaml
# tests/load/artillery.yml
config:
  target: 'http://localhost:3000'
  phases:
    - duration: 60
      arrivalRate: 10
      name: "Warm up"
    - duration: 120
      arrivalRate: 50
      name: "Ramp up"
    - duration: 300
      arrivalRate: 100
      name: "Sustained load"
  plugins:
    metrics-by-endpoint: {}

scenarios:
  - name: "Text Message Webhook"
    weight: 70
    flow:
      - function: "generateWebhookPayload"
      - post:
          url: "/webhook"
          headers:
            "x-line-signature": "{{ signature }}"
            "Content-Type": "application/json"
          body: "{{ payload }}"
          expect:
            - statusCode: 200

  - name: "Follow Event"
    weight: 15
    flow:
      - function: "generateFollowPayload"
      - post:
          url: "/webhook"
          headers:
            "x-line-signature": "{{ signature }}"
            "Content-Type": "application/json"
          body: "{{ payload }}"

  - name: "Health Check"
    weight: 15
    flow:
      - get:
          url: "/health"
          expect:
            - statusCode: 200
```

### 10.2 Load Test Helper

```javascript
// tests/load/helpers.js
const crypto = require('crypto');

function generateWebhookPayload(context, events, done) {
  const payload = JSON.stringify({
    destination: 'Uc8166f3e4f74429573a529316aef37f',
    events: [{
      type: 'message',
      message: { type: 'text', id: String(Date.now()), text: 'สวัสดี' },
      webhookEventId: `TEST${Date.now()}`,
      deliveryContext: { isRedelivery: false },
      timestamp: Date.now(),
      source: { type: 'user', userId: `U${Math.random().toString(36).slice(2, 34)}` },
      replyToken: crypto.randomBytes(16).toString('hex'),
      mode: 'active'
    }]
  });

  const signature = crypto
    .createHmac('sha256', process.env.LINE_CHANNEL_SECRET || 'test-secret')
    .update(payload)
    .digest('base64');

  context.vars.payload = payload;
  context.vars.signature = signature;
  done();
}

module.exports = { generateWebhookPayload };
```

---

## 11. CI/CD Integration

### 11.1 GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: LINE Bot Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Create test environment
        run: |
          cat > .env.test << EOF
          LINE_CHANNEL_ACCESS_TOKEN=test-access-token
          LINE_CHANNEL_SECRET=test-channel-secret
          REDIS_URL=redis://localhost:6379
          NODE_ENV=test
          EOF

      - name: Run unit tests
        run: npm run test:unit -- --coverage

      - name: Run integration tests
        run: npm run test:integration

      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info

      - name: Check coverage threshold
        run: npm run test:coverage-check
```

### 11.2 Package.json Scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:unit": "jest tests/unit",
    "test:integration": "jest tests/integration",
    "test:e2e": "jest tests/e2e",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage",
    "test:coverage-check": "jest --coverage --coverageThreshold='{\"global\":{\"lines\":80}}'",
    "test:load": "artillery run tests/load/artillery.yml"
  }
}
```

---

## 12. Automated Regression Testing

### 12.1 Regression Test Suite

```javascript
// tests/regression/regression.test.js
const request = require('supertest');
const crypto = require('crypto');
const app = require('../../src/app');
const MockLineClient = require('../mocks/line-client.mock');

const CHANNEL_SECRET = 'test-channel-secret';

const regressionCases = [
  {
    name: 'ข้อความสวัสดี',
    input: { text: 'สวัสดี' },
    expectReply: true,
    expectMessageType: 'text'
  },
  {
    name: 'ข้อความ Hello',
    input: { text: 'hello' },
    expectReply: true,
    expectMessageType: 'text'
  },
  {
    name: 'ข้อความว่าง',
    input: { text: '' },
    expectReply: false
  },
  {
    name: 'ข้อความยาวมาก',
    input: { text: 'ก'.repeat(1000) },
    expectReply: true
  },
  {
    name: 'อักขระพิเศษ',
    input: { text: '<script>alert("xss")</script>' },
    expectReply: true
  },
  {
    name: 'Emoji',
    input: { text: '😀🎉👍' },
    expectReply: true
  }
];

describe('Regression Tests', () => {
  regressionCases.forEach(testCase => {
    test(`[Regression] ${testCase.name}`, async () => {
      const mockClient = new MockLineClient();

      const event = {
        type: 'message',
        message: { type: 'text', id: '123', text: testCase.input.text },
        webhookEventId: 'TEST001',
        deliveryContext: { isRedelivery: false },
        timestamp: Date.now(),
        source: { type: 'user', userId: 'U123' },
        replyToken: 'test-reply-token',
        mode: 'active'
      };

      const payload = JSON.stringify({
        destination: 'Uc123',
        events: [event]
      });

      const signature = crypto
        .createHmac('sha256', CHANNEL_SECRET)
        .update(payload)
        .digest('base64');

      const res = await request(app)
        .post('/webhook')
        .set('x-line-signature', signature)
        .set('content-type', 'application/json')
        .send(payload);

      expect(res.status).toBe(200);
    });
  });
});
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Jest Setup** - การตั้งค่า Test Framework สำหรับ LINE Bot
2. **Test Fixtures** - สร้างข้อมูลทดสอบที่สม่ำเสมอ
3. **Event Factory** - สร้าง LINE Events สำหรับทดสอบ
4. **Mock LINE Client** - Mock API สำหรับ Unit Testing
5. **Unit Tests** - ทดสอบ Handler แต่ละตัว
6. **Integration Tests** - ทดสอบ Webhook Endpoint
7. **Flex Message Testing** - ทดสอบ Flex Messages
8. **Python Pytest** - ทดสอบ LINE Bot ด้วย Python
9. **Flow Testing** - ทดสอบ Conversation Flows
10. **Load Testing** - ทดสอบภายใต้โหลดสูง
11. **CI/CD Integration** - รวมการทดสอบเข้ากับ Pipeline

ขั้นตอนต่อไป: ใน Part 60 เราจะเรียนรู้เรื่อง LINE Things สำหรับ IoT
