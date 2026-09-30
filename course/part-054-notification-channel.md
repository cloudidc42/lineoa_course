# Part 54: Notification Channel

## สารบัญ

1. [Notification Channel Overview](#notification-channel-overview)
2. [Notification Channel vs Messaging API](#notification-channel-vs-messaging-api)
3. [LINE Notify (หมายเหตุ deprecated)](#line-notify-deprecated)
4. [Notification Channel API](#notification-channel-api)
5. [ส่ง Notification โดยไม่ต้องมี Reply Token](#ส่ง-notification-โดยไม่ต้องมี-reply-token)
6. [User Consent สำหรับ Notification](#user-consent-สำหรับ-notification)
7. [Manage Notification Channel](#manage-notification-channel)
8. [Linked OA](#linked-oa)
9. [Notification Types](#notification-types)
10. [ส่ง Rich Content Notifications](#ส่ง-rich-content-notifications)
11. [Integration กับ Order Management](#integration-กับ-order-management)
12. [Best Practices](#best-practices)

---

## 1. Notification Channel Overview

Notification Channel คือฟีเจอร์ที่ช่วยให้แอปพลิเคชันส่งข้อความแจ้งเตือนไปยังผู้ใช้ LINE ได้ โดยผ่านกระบวนการขอ Consent จากผู้ใช้

### ข้อดีของ Notification Channel

```
┌─────────────────────────────────────────────────────────────┐
│                 Notification Channel Benefits                │
├─────────────────────────────────────────────────────────────┤
│ ✓ ผู้ใช้เลือกรับการแจ้งเตือนเอง (User Consent)             │
│ ✓ ส่งได้โดยไม่ต้องรอ Reply Token                           │
│ ✓ แยก Notification ออกจาก Chat ปกติ                         │
│ ✓ ผู้ใช้ระบุความต้องการรับแจ้งเตือน                          │
│ ✓ ไม่นับใน Monthly Message Limit (Messaging API)            │
└─────────────────────────────────────────────────────────────┘
```

### Use Cases

- **E-commerce**: แจ้งสถานะคำสั่งซื้อ, การจัดส่ง
- **Banking**: แจ้งธุรกรรม, ยอดเงิน
- **Healthcare**: แจ้งนัดหมาย, ผลตรวจ
- **Travel**: แจ้งเตือนเที่ยวบิน, การจอง
- **IoT**: แจ้งเตือนจากอุปกรณ์

---

## 2. Notification Channel vs Messaging API

### ความแตกต่างหลัก

| ฟีเจอร์ | Notification Channel | Messaging API |
|---------|---------------------|---------------|
| **วัตถุประสงค์** | ส่ง Notification แบบ One-way | Two-way Chat |
| **User Consent** | ต้องขอ Consent | ผู้ใช้ Follow OA |
| **Reply Token** | ไม่ต้องใช้ | ต้องใช้สำหรับ Reply |
| **Rate Limit** | สูงกว่า | ขึ้นกับ Plan |
| **Message Quota** | ไม่นับ Monthly Quota | นับ Monthly Quota |
| **Message Types** | Text + ประเภทจำกัด | ทุกประเภท |
| **Bot Interaction** | ไม่มี | มีเต็มรูปแบบ |
| **Cost** | แยกจาก Messaging | ขึ้นกับ Plan |

### เมื่อไรควรใช้อะไร

```
ใช้ Notification Channel เมื่อ:
- ต้องการส่ง Notification ทางเดียว
- ผู้ใช้ไม่จำเป็นต้องตอบกลับ
- ต้องการส่งจำนวนมาก
- ต้องการแยก UX ออกจาก Bot

ใช้ Messaging API เมื่อ:
- ต้องการ Interactive Chat
- ผู้ใช้ต้องตอบกลับ
- มี Rich Menu
- ต้องการ Flex Message เต็มรูปแบบ
```

---

## 3. LINE Notify (Deprecated)

### หมายเหตุสำคัญ

```
⚠️ LINE Notify จะหยุดให้บริการในวันที่ 1 เมษายน 2025
   และปิดระบบสมบูรณ์วันที่ 31 มีนาคม 2025

Migration Path:
LINE Notify → Notification Channel API
```

### การ Migrate จาก LINE Notify

```javascript
// เดิม: LINE Notify
const notifyMessage = async (token, message) => {
  await axios.post(
    'https://notify-api.line.me/api/notify',
    new URLSearchParams({ message }),
    { headers: { 'Authorization': `Bearer ${token}` } }
  );
};

// ใหม่: Notification Channel API
// (ดูตัวอย่างในหัวข้อถัดไป)
```

---

## 4. Notification Channel API

### Channel Setup

Notification Channel เป็นส่วนหนึ่งของ LINE Login Channel ที่เปิดใช้ฟีเจอร์เพิ่มเติม

### Endpoints

```
Base: https://api.line.me

ส่ง Notification:
POST /message/v3/notificationsent/push

ดู Notification Profile:
GET /message/v3/notificationsent/notification/users/{userId}

ยกเลิก Notification:
DELETE /message/v3/notificationsent/notification/users/{userId}

ดู Consent Status:
GET /message/v3/notificationsent/notification/users/{userId}/status
```

### Authentication

```javascript
// Notification Channel ใช้ Channel Access Token แบบเดียวกับ Messaging API
const headers = {
  'Authorization': `Bearer ${process.env.CHANNEL_ACCESS_TOKEN}`,
  'Content-Type': 'application/json'
};
```

---

## 5. ส่ง Notification โดยไม่ต้องมี Reply Token

### ส่ง Text Notification

```javascript
const axios = require('axios');
require('dotenv').config();

const LINE_API = 'https://api.line.me';
const CHANNEL_TOKEN = process.env.CHANNEL_ACCESS_TOKEN;

async function sendNotification(userId, messages) {
  const payload = {
    recipient: {
      type: 'user',
      id: userId
    },
    notification: {
      messages: Array.isArray(messages) ? messages : [messages]
    }
  };

  try {
    const response = await axios.post(
      `${LINE_API}/message/v3/notificationsent/push`,
      payload,
      {
        headers: {
          'Authorization': `Bearer ${CHANNEL_TOKEN}`,
          'Content-Type': 'application/json'
        }
      }
    );
    return response.data;
  } catch (error) {
    console.error('Notification error:', error.response?.data);
    throw error;
  }
}

// ตัวอย่างการใช้งาน
async function sendOrderUpdateNotification(userId, order) {
  await sendNotification(userId, {
    type: 'text',
    text: `📦 อัพเดตคำสั่งซื้อ #${order.orderNumber}\n\nสถานะ: ${getStatusText(order.status)}\n\n${order.trackingNumber ? `Tracking: ${order.trackingNumber}` : ''}`
  });
}

function getStatusText(status) {
  const statusMap = {
    'pending': 'รอดำเนินการ',
    'confirmed': 'ยืนยันแล้ว',
    'processing': 'กำลังเตรียมสินค้า',
    'shipped': 'จัดส่งแล้ว',
    'delivered': 'ส่งถึงแล้ว',
    'cancelled': 'ยกเลิกแล้ว'
  };
  return statusMap[status] || status;
}
```

### ส่งหลาย Messages พร้อมกัน

```javascript
async function sendRichNotification(userId, orderData) {
  const messages = [
    // Message 1: Text summary
    {
      type: 'text',
      text: `✅ คำสั่งซื้อ #${orderData.orderNumber} ได้รับการยืนยันแล้ว`
    },
    // Message 2: Flex Message
    {
      type: 'flex',
      altText: `คำสั่งซื้อ #${orderData.orderNumber}`,
      contents: buildOrderFlexMessage(orderData)
    }
  ];

  await sendNotification(userId, messages);
}

function buildOrderFlexMessage(order) {
  return {
    type: 'bubble',
    body: {
      type: 'box',
      layout: 'vertical',
      contents: [
        {
          type: 'text',
          text: `คำสั่งซื้อ #${order.orderNumber}`,
          weight: 'bold',
          size: 'xl'
        },
        {
          type: 'box',
          layout: 'horizontal',
          margin: 'md',
          contents: [
            {
              type: 'text',
              text: 'สถานะ',
              color: '#555555',
              size: 'sm',
              flex: 1
            },
            {
              type: 'text',
              text: getStatusText(order.status),
              color: '#111111',
              size: 'sm',
              flex: 2
            }
          ]
        },
        {
          type: 'box',
          layout: 'horizontal',
          margin: 'sm',
          contents: [
            {
              type: 'text',
              text: 'ยอดรวม',
              color: '#555555',
              size: 'sm',
              flex: 1
            },
            {
              type: 'text',
              text: `฿${order.totalAmount.toLocaleString()}`,
              color: '#111111',
              size: 'sm',
              flex: 2,
              weight: 'bold'
            }
          ]
        }
      ]
    },
    footer: {
      type: 'box',
      layout: 'vertical',
      contents: [
        {
          type: 'button',
          action: {
            type: 'uri',
            label: 'ดูรายละเอียด',
            uri: `https://yourshop.com/orders/${order._id}`
          },
          style: 'primary'
        }
      ]
    }
  };
}
```

---

## 6. User Consent สำหรับ Notification

ก่อนที่จะส่ง Notification ได้ ผู้ใช้ต้องให้ Consent ก่อน

### Consent Flow

```
ผู้ใช้                    เว็บ/แอป                  LINE Platform
  │                          │                              │
  │  1. Login/Register       │                              │
  │─────────────────────────>│                              │
  │                          │  2. Generate Consent URL     │
  │                          │─────────────────────────────>│
  │                          │  3. Consent URL              │
  │                          │<─────────────────────────────│
  │  4. Redirect ไป Consent  │                              │
  │<─────────────────────────│                              │
  │  5. ยอมรับ Consent       │                              │
  │─────────────────────────────────────────────────────>   │
  │  6. Redirect + code      │                              │
  │─────────────────────────>│                              │
  │                          │  7. บันทึก Consent           │
  │                          │                              │
```

### สร้าง Consent URL

```javascript
const crypto = require('crypto');

function generateNotificationConsentUrl(userId, options = {}) {
  const {
    channelId = process.env.LINE_LOGIN_CHANNEL_ID,
    callbackUrl = process.env.NOTIFICATION_CONSENT_CALLBACK,
    scopes = ['openid', 'profile'],
    botPrompt = 'aggressive' // เชิญ Follow OA
  } = options;

  const state = crypto.randomBytes(16).toString('hex');
  const nonce = crypto.randomBytes(16).toString('hex');

  const params = new URLSearchParams({
    response_type: 'code',
    client_id: channelId,
    redirect_uri: callbackUrl,
    scope: [...scopes, 'message.write'].join(' '),
    state,
    nonce,
    bot_prompt: botPrompt
  });

  return {
    url: `https://access.line.me/oauth2/v2.1/authorize?${params}`,
    state,
    nonce,
    userId
  };
}

// Route สำหรับขอ Consent
app.get('/notifications/consent', requireAuth, (req, res) => {
  const consentData = generateNotificationConsentUrl(req.user.lineUserId);

  req.session.notificationConsent = {
    state: consentData.state,
    nonce: consentData.nonce,
    userId: req.user._id.toString()
  };

  res.redirect(consentData.url);
});

// Callback หลังได้รับ Consent
app.get('/notifications/consent/callback', async (req, res) => {
  const { code, state } = req.query;
  const consentSession = req.session.notificationConsent;

  if (!consentSession || consentSession.state !== state) {
    return res.redirect('/settings?error=invalid_consent');
  }

  try {
    // แลก code เอา token
    const tokenData = await exchangeCode(code);

    // บันทึกว่าผู้ใช้ให้ Consent แล้ว
    await User.findByIdAndUpdate(consentSession.userId, {
      notificationConsent: true,
      notificationConsentAt: new Date(),
      notificationAccessToken: tokenData.access_token
    });

    delete req.session.notificationConsent;
    res.redirect('/settings?success=notifications_enabled');
  } catch (error) {
    console.error('Consent callback error:', error);
    res.redirect('/settings?error=consent_failed');
  }
});
```

### ตรวจสอบ Consent Status

```javascript
async function checkConsentStatus(userId) {
  try {
    const response = await axios.get(
      `${LINE_API}/message/v3/notificationsent/notification/users/${userId}/status`,
      {
        headers: {
          'Authorization': `Bearer ${CHANNEL_TOKEN}`
        }
      }
    );
    return response.data;
  } catch (error) {
    if (error.response?.status === 404) {
      return { active: false };
    }
    throw error;
  }
}

// Middleware ตรวจสอบก่อนส่ง
async function requireNotificationConsent(userId) {
  const user = await User.findOne({ lineUserId: userId });

  if (!user?.notificationConsent) {
    throw new Error('User has not consented to notifications');
  }

  const status = await checkConsentStatus(userId);
  if (!status.active) {
    // อัพเดต Database
    await User.updateOne(
      { lineUserId: userId },
      { notificationConsent: false }
    );
    throw new Error('Notification consent has been revoked');
  }

  return true;
}
```

---

## 7. Manage Notification Channel

### ดูข้อมูล Notification ของ User

```javascript
async function getNotificationProfile(userId) {
  try {
    const response = await axios.get(
      `${LINE_API}/message/v3/notificationsent/notification/users/${userId}`,
      {
        headers: { 'Authorization': `Bearer ${CHANNEL_TOKEN}` }
      }
    );
    return response.data;
  } catch (error) {
    if (error.response?.status === 404) return null;
    throw error;
  }
}

// Response Example:
/*
{
  "userId": "U4af4980629...",
  "notificationEnabled": true,
  "notificationSettings": {
    "sound": true,
    "vibration": true
  }
}
*/
```

### ยกเลิก Notification สำหรับ User

```javascript
async function cancelNotificationConsent(userId) {
  try {
    await axios.delete(
      `${LINE_API}/message/v3/notificationsent/notification/users/${userId}`,
      {
        headers: { 'Authorization': `Bearer ${CHANNEL_TOKEN}` }
      }
    );

    // อัพเดต Database
    await User.updateOne(
      { lineUserId: userId },
      {
        notificationConsent: false,
        notificationCancelledAt: new Date()
      }
    );

    return true;
  } catch (error) {
    console.error('Cancel notification error:', error.response?.data);
    throw error;
  }
}

// Route สำหรับ Unsubscribe จากฝั่งผู้ใช้
app.post('/settings/notifications/disable', requireAuth, async (req, res) => {
  try {
    const user = req.user;
    if (user.lineUserId) {
      await cancelNotificationConsent(user.lineUserId);
    }
    res.json({ success: true, message: 'ปิดการแจ้งเตือนแล้ว' });
  } catch (error) {
    res.status(500).json({ error: 'เกิดข้อผิดพลาด' });
  }
});
```

---

## 8. Linked OA

Notification Channel สามารถเชื่อมกับ LINE OA เพื่อรวม UX

### การเชื่อม Notification Channel กับ OA

```javascript
// ตรวจสอบว่า User Follow OA หรือยัง
async function checkUserFollowsOA(userId) {
  try {
    const response = await axios.get(
      `https://api.line.me/v2/bot/profile/${userId}`,
      {
        headers: {
          'Authorization': `Bearer ${process.env.MESSAGING_API_TOKEN}`
        }
      }
    );
    return true; // User follows OA
  } catch (error) {
    if (error.response?.status === 404) return false; // User doesn't follow
    throw error;
  }
}

// ส่ง Notification พร้อม Link ไปยัง OA
async function sendNotificationWithOALink(userId, message) {
  const isFollowing = await checkUserFollowsOA(userId);

  const messages = [
    {
      type: 'text',
      text: message
    }
  ];

  if (!isFollowing) {
    // เพิ่ม Button ให้ Follow OA
    messages.push({
      type: 'flex',
      altText: 'ติดตาม LINE OA ของเรา',
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: 'ต้องการรับการแจ้งเตือนเพิ่มเติม?',
              wrap: true
            },
            {
              type: 'text',
              text: 'ติดตาม LINE OA ของเราเพื่อรับข้อมูลสินค้าและโปรโมชัน',
              color: '#666666',
              size: 'sm',
              wrap: true,
              margin: 'sm'
            }
          ]
        },
        footer: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'button',
              action: {
                type: 'uri',
                label: 'ติดตาม LINE OA',
                uri: `https://line.me/R/ti/p/@${process.env.OA_BASIC_ID}`
              },
              style: 'primary',
              color: '#00B900'
            }
          ]
        }
      }
    });
  }

  await sendNotification(userId, messages);
}
```

---

## 9. Notification Types

### Text Notification

```javascript
const textNotification = {
  type: 'text',
  text: 'ข้อความแจ้งเตือนธรรมดา'
};
```

### Image Notification

```javascript
const imageNotification = {
  type: 'image',
  originalContentUrl: 'https://example.com/image.jpg',
  previewImageUrl: 'https://example.com/image-preview.jpg'
};
```

### Flex Message Notification

```javascript
const flexNotification = {
  type: 'flex',
  altText: 'Alt text สำหรับ devices ที่ไม่รองรับ',
  contents: {
    type: 'bubble',
    // ... Flex Message structure
  }
};
```

### Template Notification

```javascript
// Confirm Template
const confirmNotification = {
  type: 'template',
  altText: 'ยืนยันการจอง',
  template: {
    type: 'confirm',
    text: 'คุณต้องการยืนยันการจองหรือไม่?',
    actions: [
      { type: 'postback', label: 'ยืนยัน', data: 'confirm=yes' },
      { type: 'postback', label: 'ยกเลิก', data: 'confirm=no' }
    ]
  }
};

// Buttons Template
const buttonsNotification = {
  type: 'template',
  altText: 'ดูรายละเอียด',
  template: {
    type: 'buttons',
    thumbnailImageUrl: 'https://example.com/product.jpg',
    title: 'สินค้ามาแล้ว!',
    text: 'สินค้าที่คุณสั่งได้จัดส่งแล้ว',
    actions: [
      {
        type: 'uri',
        label: 'ติดตามพัสดุ',
        uri: 'https://tracking.example.com'
      },
      {
        type: 'postback',
        label: 'ดูคำสั่งซื้อ',
        data: 'action=view_order'
      }
    ]
  }
};
```

---

## 10. ส่ง Rich Content Notifications

### Notification Service Class

```javascript
// services/NotificationService.js
const axios = require('axios');

class NotificationService {
  constructor(channelAccessToken) {
    this.token = channelAccessToken || process.env.CHANNEL_ACCESS_TOKEN;
    this.apiUrl = 'https://api.line.me';
  }

  async send(userId, messages) {
    if (!Array.isArray(messages)) messages = [messages];

    const payload = {
      recipient: { type: 'user', id: userId },
      notification: { messages }
    };

    try {
      const response = await axios.post(
        `${this.apiUrl}/message/v3/notificationsent/push`,
        payload,
        {
          headers: {
            'Authorization': `Bearer ${this.token}`,
            'Content-Type': 'application/json'
          }
        }
      );
      return { success: true, data: response.data };
    } catch (error) {
      const errData = error.response?.data;
      throw new Error(`Notification failed: ${errData?.message || error.message}`);
    }
  }

  // ─── Notification Templates ────────────────────────────────────────────

  async sendOrderConfirmation(userId, order) {
    return await this.send(userId, [
      {
        type: 'flex',
        altText: `ยืนยันคำสั่งซื้อ #${order.orderNumber}`,
        contents: this.buildOrderConfirmationFlex(order)
      }
    ]);
  }

  async sendShippingUpdate(userId, order) {
    return await this.send(userId, [
      {
        type: 'text',
        text: `🚚 คำสั่งซื้อ #${order.orderNumber} จัดส่งแล้ว!\n\n` +
              `Tracking: ${order.trackingNumber}\n` +
              `บริษัทขนส่ง: ${order.courier}\n\n` +
              `ติดตามพัสดุ: ${order.trackingUrl}`
      }
    ]);
  }

  async sendDeliveryConfirmation(userId, order) {
    return await this.send(userId, [
      {
        type: 'text',
        text: `✅ คำสั่งซื้อ #${order.orderNumber} ส่งถึงแล้ว!\n\nขอบคุณที่ใช้บริการ`
      },
      {
        type: 'template',
        altText: 'รีวิวสินค้า',
        template: {
          type: 'buttons',
          text: 'พอใจกับสินค้าไหม? รีวิวได้เลย!',
          actions: [
            {
              type: 'uri',
              label: 'เขียนรีวิว',
              uri: `https://yourshop.com/review/${order._id}`
            },
            {
              type: 'postback',
              label: 'ดีมาก!',
              data: `action=quick_review&rating=5&orderId=${order._id}`
            }
          ]
        }
      }
    ]);
  }

  async sendPaymentReminder(userId, order) {
    const expiresIn = Math.floor((order.paymentExpiresAt - Date.now()) / 60000);

    return await this.send(userId, [
      {
        type: 'flex',
        altText: `แจ้งเตือน: ชำระเงินภายใน ${expiresIn} นาที`,
        contents: {
          type: 'bubble',
          header: {
            type: 'box',
            layout: 'vertical',
            contents: [
              {
                type: 'text',
                text: '⏰ แจ้งเตือนการชำระเงิน',
                color: '#ffffff',
                weight: 'bold'
              }
            ],
            backgroundColor: '#FF6B35'
          },
          body: {
            type: 'box',
            layout: 'vertical',
            contents: [
              {
                type: 'text',
                text: `คำสั่งซื้อ #${order.orderNumber}`,
                weight: 'bold'
              },
              {
                type: 'text',
                text: `กรุณาชำระเงินภายใน ${expiresIn} นาที`,
                color: '#FF6B35',
                margin: 'sm'
              },
              {
                type: 'text',
                text: `ยอดชำระ: ฿${order.totalAmount.toLocaleString()}`,
                size: 'xl',
                weight: 'bold',
                margin: 'md'
              }
            ]
          },
          footer: {
            type: 'box',
            layout: 'vertical',
            contents: [
              {
                type: 'button',
                action: {
                  type: 'uri',
                  label: 'ชำระเงินตอนนี้',
                  uri: `https://yourshop.com/pay/${order._id}`
                },
                style: 'primary',
                color: '#FF6B35'
              }
            ]
          }
        }
      }
    ]);
  }

  async sendPointEarned(userId, points, totalPoints) {
    return await this.send(userId, [
      {
        type: 'flex',
        altText: `ได้รับ ${points} คะแนน`,
        contents: {
          type: 'bubble',
          hero: {
            type: 'box',
            layout: 'vertical',
            backgroundColor: '#FFD700',
            paddingAll: 'lg',
            contents: [
              {
                type: 'text',
                text: '⭐',
                size: 'xxl',
                align: 'center'
              },
              {
                type: 'text',
                text: `+${points} คะแนน`,
                size: 'xxl',
                weight: 'bold',
                align: 'center',
                color: '#333333'
              }
            ]
          },
          body: {
            type: 'box',
            layout: 'vertical',
            contents: [
              {
                type: 'text',
                text: 'คะแนนสะสมของคุณ',
                align: 'center'
              },
              {
                type: 'text',
                text: `${totalPoints.toLocaleString()} คะแนน`,
                size: 'xl',
                weight: 'bold',
                align: 'center',
                margin: 'sm'
              }
            ]
          }
        }
      }
    ]);
  }

  buildOrderConfirmationFlex(order) {
    return {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        backgroundColor: '#27AE60',
        contents: [
          { type: 'text', text: '✅ ยืนยันคำสั่งซื้อแล้ว', color: '#ffffff', weight: 'bold' }
        ]
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          { type: 'text', text: `#${order.orderNumber}`, weight: 'bold', size: 'lg' },
          { type: 'separator', margin: 'md' },
          ...order.items.map(item => ({
            type: 'box',
            layout: 'horizontal',
            margin: 'sm',
            contents: [
              { type: 'text', text: item.name, flex: 3, size: 'sm', wrap: true },
              { type: 'text', text: `x${item.quantity}`, flex: 1, size: 'sm', align: 'center' },
              { type: 'text', text: `฿${(item.price * item.quantity).toLocaleString()}`, flex: 2, size: 'sm', align: 'end' }
            ]
          })),
          { type: 'separator', margin: 'md' },
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'md',
            contents: [
              { type: 'text', text: 'ยอดรวม', weight: 'bold' },
              { type: 'text', text: `฿${order.totalAmount.toLocaleString()}`, weight: 'bold', align: 'end', color: '#27AE60' }
            ]
          }
        ]
      },
      footer: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'button',
            action: {
              type: 'uri',
              label: 'ดูรายละเอียด',
              uri: `https://yourshop.com/orders/${order._id}`
            },
            style: 'primary'
          }
        ]
      }
    };
  }
}

module.exports = NotificationService;
```

---

## 11. Integration กับ Order Management

### Order Status Notification System

```javascript
// services/OrderNotificationService.js
const NotificationService = require('./NotificationService');
const User = require('../models/User');

const notifier = new NotificationService();

// Map สถานะ → Notification Function
const statusNotifications = {
  'confirmed': async (userId, order) => {
    await notifier.sendOrderConfirmation(userId, order);
  },
  'processing': async (userId, order) => {
    await notifier.send(userId, {
      type: 'text',
      text: `🔧 คำสั่งซื้อ #${order.orderNumber} กำลังเตรียมสินค้า\nคาดว่าจะจัดส่งภายใน ${order.estimatedDelivery} วัน`
    });
  },
  'shipped': async (userId, order) => {
    await notifier.sendShippingUpdate(userId, order);
  },
  'out_for_delivery': async (userId, order) => {
    await notifier.send(userId, {
      type: 'text',
      text: `🚚 กำลังจัดส่งไปยังที่อยู่ของคุณ!\nคำสั่งซื้อ #${order.orderNumber}`
    });
  },
  'delivered': async (userId, order) => {
    await notifier.sendDeliveryConfirmation(userId, order);
  },
  'cancelled': async (userId, order) => {
    await notifier.send(userId, {
      type: 'text',
      text: `❌ คำสั่งซื้อ #${order.orderNumber} ถูกยกเลิก\n\nเหตุผล: ${order.cancelReason || 'ไม่ระบุ'}\n\nยอดเงินจะคืนภายใน 3-5 วันทำการ`
    });
  }
};

// Trigger notification เมื่อสถานะเปลี่ยน
async function notifyOrderStatusChange(order, newStatus) {
  try {
    // ดึงข้อมูล User
    const user = await User.findById(order.userId);

    if (!user?.lineUserId || !user?.notificationConsent) {
      console.log(`User ${order.userId} has no LINE or no consent`);
      return false;
    }

    // ส่ง Notification
    const notifyFn = statusNotifications[newStatus];
    if (notifyFn) {
      await notifyFn(user.lineUserId, order);
      console.log(`Sent ${newStatus} notification to ${user.lineUserId}`);
      return true;
    }

    return false;
  } catch (error) {
    console.error('Notification error:', error);
    return false;
  }
}

// Mongoose middleware (Hook)
// ใช้กับ Order model
const mongoose = require('mongoose');

const orderSchema = new mongoose.Schema({
  orderNumber: String,
  userId: mongoose.Schema.Types.ObjectId,
  items: Array,
  totalAmount: Number,
  status: String,
  trackingNumber: String,
  courier: String,
  cancelReason: String
});

// Post-save hook สำหรับ notification
orderSchema.post('save', async function(doc) {
  if (this.isModified('status')) {
    await notifyOrderStatusChange(doc, doc.status);
  }
});

// หรือใช้ findOneAndUpdate hook
orderSchema.post('findOneAndUpdate', async function(doc) {
  if (doc) {
    const update = this.getUpdate();
    const newStatus = update?.$set?.status;
    if (newStatus) {
      await notifyOrderStatusChange(doc, newStatus);
    }
  }
});
```

### Bulk Notification

```javascript
// ส่ง Notification ให้หลาย User พร้อมกัน
async function sendBulkNotification(userIds, messageBuilder, options = {}) {
  const { batchSize = 50, delay = 500 } = options;
  const results = { success: 0, failed: 0, errors: [] };

  for (let i = 0; i < userIds.length; i += batchSize) {
    const batch = userIds.slice(i, i + batchSize);

    const promises = batch.map(async userId => {
      try {
        const messages = await messageBuilder(userId);
        await notifier.send(userId, messages);
        results.success++;
      } catch (error) {
        results.failed++;
        results.errors.push({ userId, error: error.message });
      }
    });

    await Promise.all(promises);

    if (i + batchSize < userIds.length && delay > 0) {
      await new Promise(r => setTimeout(r, delay));
    }

    console.log(`Progress: ${Math.min(i + batchSize, userIds.length)}/${userIds.length}`);
  }

  return results;
}

// ตัวอย่าง: ส่ง Promotion ให้ Members ทั้งหมด
async function sendPromotionNotification(promotion) {
  // ดึง Members ที่ให้ Consent แล้ว
  const users = await User.find({
    notificationConsent: true,
    memberStatus: { $in: ['member', 'vip'] }
  }).select('lineUserId memberStatus');

  const userIds = users.map(u => u.lineUserId).filter(Boolean);
  console.log(`Sending promotion to ${userIds.length} users`);

  const result = await sendBulkNotification(
    userIds,
    async (userId) => {
      const user = users.find(u => u.lineUserId === userId);
      return [
        {
          type: 'flex',
          altText: promotion.title,
          contents: buildPromotionFlex(promotion, user)
        }
      ];
    },
    { batchSize: 100, delay: 1000 }
  );

  console.log(`Done: ${result.success} sent, ${result.failed} failed`);
  return result;
}

function buildPromotionFlex(promotion, user) {
  const discount = user?.memberStatus === 'vip' ? promotion.vipDiscount : promotion.discount;

  return {
    type: 'bubble',
    hero: {
      type: 'image',
      url: promotion.imageUrl,
      size: 'full',
      aspectRatio: '20:13',
      aspectMode: 'cover'
    },
    body: {
      type: 'box',
      layout: 'vertical',
      contents: [
        { type: 'text', text: promotion.title, weight: 'bold', size: 'xl' },
        { type: 'text', text: `ลด ${discount}%`, size: 'lg', color: '#E74C3C', margin: 'sm' },
        { type: 'text', text: promotion.description, wrap: true, color: '#666666', margin: 'sm' },
        { type: 'text', text: `หมดเขต: ${new Date(promotion.expiresAt).toLocaleDateString('th-TH')}`, size: 'sm', color: '#999999', margin: 'md' }
      ]
    },
    footer: {
      type: 'box',
      layout: 'vertical',
      contents: [
        {
          type: 'button',
          action: {
            type: 'uri',
            label: 'ใช้โปรโมชัน',
            uri: `https://yourshop.com/promo/${promotion.code}`
          },
          style: 'primary',
          color: '#E74C3C'
        }
      ]
    }
  };
}
```

### Scheduled Notification

```javascript
// Cron job สำหรับ Notification ตามเวลา
const cron = require('node-cron');

// แจ้งเตือนชำระเงินทุก 30 นาที
cron.schedule('*/30 * * * *', async () => {
  console.log('Checking pending payment orders...');

  const pendingOrders = await Order.find({
    status: 'pending',
    paymentMethod: 'line_pay',
    createdAt: {
      $gte: new Date(Date.now() - 2 * 60 * 60 * 1000), // สร้างในช่วง 2 ชั่วโมงที่ผ่านมา
      $lt: new Date(Date.now() - 30 * 60 * 1000) // แต่นานกว่า 30 นาทีที่แล้ว
    },
    paymentReminderSent: { $ne: true }
  }).populate('userId');

  for (const order of pendingOrders) {
    if (order.userId?.lineUserId && order.userId?.notificationConsent) {
      try {
        await notifier.sendPaymentReminder(order.userId.lineUserId, order);
        await Order.findByIdAndUpdate(order._id, { paymentReminderSent: true });
      } catch (error) {
        console.error(`Failed to send reminder for order ${order._id}:`, error);
      }
    }
  }
});

// แจ้งเตือนจัดส่งทุกเช้า 9 โมง
cron.schedule('0 9 * * *', async () => {
  console.log('Sending daily shipping updates...');

  const shippedOrders = await Order.find({
    status: 'shipped',
    shippedAt: {
      $lte: new Date(Date.now() - 24 * 60 * 60 * 1000)
    },
    deliveryUpdateSent: { $ne: true }
  }).populate('userId');

  for (const order of shippedOrders) {
    if (order.userId?.lineUserId && order.userId?.notificationConsent) {
      try {
        await notifier.send(order.userId.lineUserId, {
          type: 'text',
          text: `📦 คำสั่งซื้อ #${order.orderNumber}\nกำลังจัดส่ง Tracking: ${order.trackingNumber}\nตรวจสอบ: ${order.trackingUrl}`
        });
        await Order.findByIdAndUpdate(order._id, { deliveryUpdateSent: true });
      } catch (error) {
        console.error(`Error:`, error);
      }
    }
  }
});
```

---

## 12. Best Practices

### 1. Retry Logic

```javascript
async function sendNotificationWithRetry(userId, messages, maxRetries = 3) {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await notifier.send(userId, messages);
    } catch (error) {
      if (attempt === maxRetries) throw error;

      // ตรวจสอบประเภทข้อผิดพลาด
      const status = error.response?.status;
      if (status === 429) {
        // Rate Limited - รอแล้วลองใหม่
        const retryAfter = parseInt(error.response.headers['retry-after'] || '60') * 1000;
        console.log(`Rate limited. Waiting ${retryAfter}ms...`);
        await new Promise(r => setTimeout(r, retryAfter));
      } else if (status >= 500) {
        // Server Error - รอแล้วลองใหม่
        const delay = Math.pow(2, attempt) * 1000;
        await new Promise(r => setTimeout(r, delay));
      } else {
        throw error; // Client Error ไม่ต้อง retry
      }
    }
  }
}
```

### 2. Queue System

```javascript
// ใช้ Bull Queue สำหรับส่ง Notification
const Queue = require('bull');

const notificationQueue = new Queue('notifications', {
  redis: process.env.REDIS_URL
});

// Producer
async function queueNotification(userId, messages, options = {}) {
  await notificationQueue.add(
    { userId, messages },
    {
      attempts: 3,
      backoff: { type: 'exponential', delay: 2000 },
      removeOnComplete: true,
      ...options
    }
  );
}

// Consumer
notificationQueue.process(async (job) => {
  const { userId, messages } = job.data;
  return await notifier.send(userId, messages);
});

notificationQueue.on('failed', (job, error) => {
  console.error(`Notification failed for ${job.data.userId}:`, error.message);
});
```

### 3. Notification Log

```javascript
// บันทึกการส่ง Notification
const notificationLogSchema = new mongoose.Schema({
  userId: String,
  lineUserId: String,
  type: String,
  messagePreview: String,
  status: { type: String, enum: ['sent', 'failed', 'queued'] },
  error: String,
  sentAt: { type: Date, default: Date.now },
  metadata: mongoose.Schema.Types.Mixed
});

// Wrapper ที่บันทึก log
async function sendAndLog(userId, lineUserId, messages, metadata = {}) {
  let status = 'queued';
  let error = null;

  try {
    await notifier.send(lineUserId, messages);
    status = 'sent';
  } catch (err) {
    status = 'failed';
    error = err.message;
    throw err;
  } finally {
    await NotificationLog.create({
      userId,
      lineUserId,
      type: metadata.type || 'general',
      messagePreview: Array.isArray(messages)
        ? messages[0]?.text?.substring(0, 50)
        : messages?.text?.substring(0, 50),
      status,
      error,
      metadata
    });
  }
}
```

---

## สรุป Notification Channel

```
Notification Channel Flow:

1. User ให้ Consent
   └─> ได้ notificationConsent = true

2. Backend ส่ง Notification
   └─> POST /message/v3/notificationsent/push
       └─> ถึง User ใน LINE ทันที

3. Notification Types:
   - Text: ข้อความธรรมดา
   - Image: รูปภาพ  
   - Flex: UI ที่กำหนดเอง
   - Template: Confirm, Buttons

4. Best Practices:
   - ขอ Consent ก่อนเสมอ
   - ส่งเฉพาะข้อมูลที่เกี่ยวข้อง
   - ไม่ส่งบ่อยเกินไป
   - ใช้ Queue สำหรับ Bulk Send
```

| ประเภท | ความถี่แนะนำ | ตัวอย่าง |
|--------|------------|---------|
| Transaction | ทุกครั้ง | ยืนยัน Order, ชำระเงิน |
| Shipping | 2-3 ครั้ง | จัดส่ง, ออกจาก DC, ส่งถึง |
| Promotion | สูงสุด 2 ครั้ง/สัปดาห์ | โปรโมชัน, Flash Sale |
| Reminder | 1-2 ครั้ง | ชำระเงิน, นัดหมาย |

---

*ส่วนถัดไป: [Part 55: Audience API](part-055-audience-api.md)*
