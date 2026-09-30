# Part 53: LINE Pay

## สารบัญ

1. [LINE Pay Overview](#line-pay-overview)
2. [LINE Pay Sandbox Setup](#line-pay-sandbox-setup)
3. [Payment Flow](#payment-flow)
4. [Request Payment API](#request-payment-api)
5. [Payment Confirmation API](#payment-confirmation-api)
6. [Capture API](#capture-api)
7. [Void/Refund API](#voidrefund-api)
8. [Check Payment Status](#check-payment-status)
9. [Recurring Payment](#recurring-payment)
10. [LINE Pay กับ LINE OA](#line-pay-กับ-line-oa)
11. [Thailand-specific Setup](#thailand-specific-setup)
12. [Node.js Integration Example](#nodejs-integration-example)
13. [Error Codes](#error-codes)
14. [Testing ใน Sandbox](#testing-ใน-sandbox)

---

## 1. LINE Pay Overview

LINE Pay คือระบบชำระเงินออนไลน์ของ LINE ที่รองรับการชำระเงินผ่านแอป LINE โดยตรง

### LINE Pay ใน Thailand

```
LINE Pay Thailand:
- ชำระผ่าน LINE App
- รองรับ PromptPay, บัตรเครดิต/เดบิต
- เชื่อมต่อกับ LINE OA ได้
- รองรับ Recurring Payment
- มี Sandbox สำหรับทดสอบ
```

### LINE Pay vs PromptPay

| ฟีเจอร์ | LINE Pay | PromptPay |
|---------|---------|---------|
| ใช้ผ่าน | LINE App | หลายแอป |
| User Experience | ใน LINE | ออกไปนอก LINE |
| Recurring | รองรับ | ไม่รองรับ |
| Integration | API | QR Code |
| รองรับ | ทุกประเทศ | ไทยเท่านั้น |

### API Endpoints

```
Sandbox:  https://sandbox-api-pay.line.me
Production: https://api-pay.line.me
```

---

## 2. LINE Pay Sandbox Setup

### ขั้นตอนการสร้าง Sandbox Account

1. ไปที่ LINE Pay Merchant Console: https://pay.line.me/portal/th/mypage/sandbox/list
2. สร้าง Sandbox Account ใหม่
3. รับข้อมูล:
   - **Channel ID**: รหัส Channel
   - **Channel Secret Key**: สำหรับ HMAC Signature

### Environment Variables

```env
# LINE Pay Configuration
LINE_PAY_CHANNEL_ID=1234567890
LINE_PAY_CHANNEL_SECRET=abcdef1234567890abcdef1234567890
LINE_PAY_SANDBOX=true
LINE_PAY_CONFIRM_URL=https://yoursite.com/line-pay/confirm
LINE_PAY_CANCEL_URL=https://yoursite.com/line-pay/cancel
```

### การสร้าง HMAC-SHA256 Signature

LINE Pay ต้องการ Signature ในทุก Request เพื่อความปลอดภัย

```javascript
const crypto = require('crypto');

function generateSignature(channelSecret, requestBody, nonce) {
  const text = channelSecret + requestBody + nonce;
  return crypto
    .createHmac('SHA256', channelSecret)
    .update(text)
    .digest('base64');
}

// สร้าง headers สำหรับ LINE Pay API
function createLinePayHeaders(channelId, channelSecret, requestBody, isV3 = true) {
  const nonce = crypto.randomBytes(8).toString('hex');
  const body = typeof requestBody === 'string' ? requestBody : JSON.stringify(requestBody);
  const signature = generateSignature(channelSecret, body, nonce);

  return {
    'Content-Type': 'application/json',
    'X-LINE-ChannelId': channelId,
    'X-LINE-Authorization-Nonce': nonce,
    'X-LINE-Authorization': signature
  };
}
```

---

## 3. Payment Flow

### Payment Flow แบบสมบูรณ์

```
ผู้ใช้                  เว็บไซต์/App              LINE Pay API
  │                          │                         │
  │  1. กดสั่งซื้อ            │                         │
  │─────────────────────────>│                         │
  │                          │  2. Request Payment     │
  │                          │────────────────────────>│
  │                          │                         │
  │                          │  3. paymentUrl          │
  │                          │<────────────────────────│
  │                          │                         │
  │  4. Redirect ไป LINE Pay │                         │
  │<─────────────────────────│                         │
  │                          │                         │
  │  5. ยืนยันการชำระเงิน     │                         │
  │─────────────────────────────────────────────────>  │
  │                          │                         │
  │  6. Redirect + transactionId                       │
  │<─────────────────────────│                         │
  │                          │  7. Confirm Payment     │
  │                          │────────────────────────>│
  │                          │                         │
  │                          │  8. Payment Success     │
  │                          │<────────────────────────│
  │  9. แสดงหน้า Success      │                         │
  │<─────────────────────────│                         │
```

### Payment States

```
INITIAL → AUTHORIZATION → AUTHORIZATION_VOIDED
                        ↓
                   CAPTURE → CAPTURE_FAILED
                        ↓
                   COMPLETE
```

---

## 4. Request Payment API

### Endpoint

```
POST /v3/payments/request
```

### Request Body

```javascript
const requestPayment = async (orderData) => {
  const payload = {
    amount: orderData.amount,
    currency: 'THB',
    orderId: orderData.orderId,
    packages: [
      {
        id: orderData.packageId || 'package-001',
        amount: orderData.amount,
        name: orderData.packageName || 'สินค้า',
        products: orderData.products.map(product => ({
          id: product.id,
          name: product.name,
          imageUrl: product.imageUrl,
          quantity: product.quantity,
          price: product.price
        }))
      }
    ],
    redirectUrls: {
      confirmUrl: `${process.env.LINE_PAY_CONFIRM_URL}?orderId=${orderData.orderId}`,
      cancelUrl: `${process.env.LINE_PAY_CANCEL_URL}?orderId=${orderData.orderId}`,
      confirmUrlType: 'SERVER' // หรือ 'CLIENT'
    },
    options: {
      payment: {
        capture: true // false = Authorization only
      },
      display: {
        locale: 'th',
        checkConfirmUrlBrowser: false
      }
    }
  };

  const baseUrl = process.env.LINE_PAY_SANDBOX === 'true'
    ? 'https://sandbox-api-pay.line.me'
    : 'https://api-pay.line.me';

  const endpoint = '/v3/payments/request';
  const headers = createLinePayHeaders(
    process.env.LINE_PAY_CHANNEL_ID,
    process.env.LINE_PAY_CHANNEL_SECRET,
    JSON.stringify(payload)
  );

  try {
    const response = await axios.post(`${baseUrl}${endpoint}`, payload, { headers });
    return response.data;
  } catch (error) {
    console.error('LINE Pay request error:', error.response?.data);
    throw error;
  }
};
```

### Response

```json
{
  "returnCode": "0000",
  "returnMessage": "Success.",
  "info": {
    "paymentUrl": {
      "web": "https://sandbox-web-pay.line.me/web/payment/wait?transactionReserveId=...",
      "app": "line://pay/payment/..."
    },
    "transactionId": 2021122300000000000,
    "paymentAccessToken": "..."
  }
}
```

### การ Redirect ผู้ใช้

```javascript
// Express route สำหรับสร้าง payment
app.post('/checkout', requireAuth, async (req, res) => {
  const { cartItems } = req.body;
  const user = req.user;

  try {
    // คำนวณยอดรวม
    const totalAmount = cartItems.reduce((sum, item) =>
      sum + (item.price * item.quantity), 0
    );

    // สร้าง Order ใน Database
    const order = await Order.create({
      userId: user._id,
      items: cartItems,
      totalAmount,
      status: 'pending',
      paymentMethod: 'line_pay'
    });

    // Request Payment
    const paymentData = await requestPayment({
      amount: totalAmount,
      orderId: order._id.toString(),
      packageName: 'คำสั่งซื้อ #' + order.orderNumber,
      products: cartItems.map(item => ({
        id: item.productId,
        name: item.productName,
        imageUrl: item.imageUrl,
        quantity: item.quantity,
        price: item.price
      }))
    });

    if (paymentData.returnCode !== '0000') {
      throw new Error(`LINE Pay error: ${paymentData.returnMessage}`);
    }

    // บันทึก transactionId
    await Order.findByIdAndUpdate(order._id, {
      linePayTransactionId: paymentData.info.transactionId,
      linePayToken: paymentData.info.paymentAccessToken
    });

    // ตรวจสอบว่า Mobile หรือ Web
    const isMobile = req.headers['user-agent'].includes('Line/');
    const paymentUrl = isMobile
      ? paymentData.info.paymentUrl.app
      : paymentData.info.paymentUrl.web;

    res.json({ paymentUrl, orderId: order._id });
  } catch (error) {
    console.error('Checkout error:', error);
    res.status(500).json({ error: 'เกิดข้อผิดพลาด กรุณาลองใหม่' });
  }
});
```

---

## 5. Payment Confirmation API

เรียกหลังจาก LINE Pay Redirect กลับมายัง `confirmUrl`

### Endpoint

```
POST /v3/payments/{transactionId}/confirm
```

```javascript
async function confirmPayment(transactionId, amount, currency = 'THB') {
  const payload = {
    amount,
    currency
  };

  const endpoint = `/v3/payments/${transactionId}/confirm`;
  const baseUrl = process.env.LINE_PAY_SANDBOX === 'true'
    ? 'https://sandbox-api-pay.line.me'
    : 'https://api-pay.line.me';

  const headers = createLinePayHeaders(
    process.env.LINE_PAY_CHANNEL_ID,
    process.env.LINE_PAY_CHANNEL_SECRET,
    JSON.stringify(payload)
  );

  try {
    const response = await axios.post(`${baseUrl}${endpoint}`, payload, { headers });
    return response.data;
  } catch (error) {
    console.error('LINE Pay confirm error:', error.response?.data);
    throw error;
  }
}

// Confirm URL Handler
app.get('/line-pay/confirm', async (req, res) => {
  const { transactionId, orderId } = req.query;

  try {
    const order = await Order.findById(orderId);

    if (!order) {
      return res.redirect('/orders?error=order_not_found');
    }

    // ตรวจสอบว่า transaction ถูกต้อง
    if (order.linePayTransactionId != transactionId) {
      return res.redirect(`/orders/${orderId}?error=invalid_transaction`);
    }

    // Confirm Payment
    const confirmResult = await confirmPayment(transactionId, order.totalAmount);

    if (confirmResult.returnCode === '0000') {
      // อัพเดต Order Status
      await Order.findByIdAndUpdate(orderId, {
        status: 'paid',
        paymentConfirmedAt: new Date(),
        linePayInfo: confirmResult.info
      });

      // ส่ง Confirmation Message ผ่าน LINE
      if (order.userId) {
        const user = await User.findById(order.userId);
        if (user?.lineUserId) {
          await sendOrderConfirmation(user.lineUserId, order);
        }
      }

      res.redirect(`/orders/${orderId}?success=payment_completed`);
    } else {
      throw new Error(`Payment failed: ${confirmResult.returnMessage}`);
    }
  } catch (error) {
    console.error('Confirm payment error:', error);
    await Order.findByIdAndUpdate(orderId, {
      status: 'payment_failed',
      paymentError: error.message
    });
    res.redirect(`/orders/${orderId}?error=payment_failed`);
  }
});
```

### Confirm Response

```json
{
  "returnCode": "0000",
  "returnMessage": "Success.",
  "info": {
    "transactionId": 2021122300000000000,
    "orderId": "order-001",
    "transactionDate": "2021-12-23T10:00:00Z",
    "transactionType": "PAYMENT",
    "payInfo": [
      {
        "method": "CREDIT_CARD",
        "amount": 500,
        "creditCardNickname": "VISA-0000",
        "creditCardBrand": "VISA",
        "maskedCreditCardNumber": "****-****-****-0000"
      }
    ],
    "packages": [
      {
        "id": "package-001",
        "name": "สินค้า",
        "amount": 500,
        "userFeeAmount": 0
      }
    ],
    "merchant": {
      "channelId": "1234567890",
      "serviceCode": "01",
      "brandName": "Your Shop",
      "pictureUrl": "https://..."
    }
  }
}
```

---

## 6. Capture API

ใช้เมื่อ `capture: false` ในขั้นตอน Request (Authorization only mode)

### Endpoint

```
POST /v3/payments/authorizations/{transactionId}/capture
```

```javascript
async function capturePayment(transactionId, amount, currency = 'THB') {
  const payload = {
    amount,
    currency
  };

  const endpoint = `/v3/payments/authorizations/${transactionId}/capture`;
  const baseUrl = process.env.LINE_PAY_SANDBOX === 'true'
    ? 'https://sandbox-api-pay.line.me'
    : 'https://api-pay.line.me';

  const headers = createLinePayHeaders(
    process.env.LINE_PAY_CHANNEL_ID,
    process.env.LINE_PAY_CHANNEL_SECRET,
    JSON.stringify(payload)
  );

  const response = await axios.post(`${baseUrl}${endpoint}`, payload, { headers });
  return response.data;
}

// ตัวอย่าง: Authorization + Capture แยกกัน
// เหมาะสำหรับระบบที่ต้อง verify stock ก่อน capture
async function authAndCapture(orderId) {
  const order = await Order.findById(orderId);

  // 1. ตรวจสอบ stock
  const isAvailable = await checkStock(order.items);
  if (!isAvailable) {
    // Void authorization ถ้า stock ไม่พอ
    await voidAuthorization(order.linePayTransactionId);
    await Order.findByIdAndUpdate(orderId, { status: 'out_of_stock' });
    return false;
  }

  // 2. Capture payment
  const captureResult = await capturePayment(
    order.linePayTransactionId,
    order.totalAmount
  );

  if (captureResult.returnCode === '0000') {
    await Order.findByIdAndUpdate(orderId, {
      status: 'paid',
      capturedAt: new Date()
    });
    return true;
  }

  return false;
}
```

---

## 7. Void/Refund API

### Void Authorization

```javascript
// Void ยกเลิก Authorization ก่อน Capture
async function voidAuthorization(transactionId) {
  const endpoint = `/v3/payments/authorizations/${transactionId}/void`;
  const baseUrl = process.env.LINE_PAY_SANDBOX === 'true'
    ? 'https://sandbox-api-pay.line.me'
    : 'https://api-pay.line.me';

  const headers = createLinePayHeaders(
    process.env.LINE_PAY_CHANNEL_ID,
    process.env.LINE_PAY_CHANNEL_SECRET,
    '' // GET request ไม่มี body
  );

  const response = await axios.post(`${baseUrl}${endpoint}`, {}, { headers });
  return response.data;
}
```

### Refund Payment

```javascript
// Refund หลังจาก Capture/Confirm แล้ว
async function refundPayment(transactionId, refundAmount = null) {
  const payload = {};
  if (refundAmount !== null) {
    payload.refundAmount = refundAmount; // Partial refund
  }
  // ถ้าไม่ระบุ = Full refund

  const endpoint = `/v3/payments/${transactionId}/refund`;
  const baseUrl = process.env.LINE_PAY_SANDBOX === 'true'
    ? 'https://sandbox-api-pay.line.me'
    : 'https://api-pay.line.me';

  const headers = createLinePayHeaders(
    process.env.LINE_PAY_CHANNEL_ID,
    process.env.LINE_PAY_CHANNEL_SECRET,
    JSON.stringify(payload)
  );

  const response = await axios.post(`${baseUrl}${endpoint}`, payload, { headers });
  return response.data;
}

// API Route สำหรับ Refund
app.post('/admin/refund/:orderId', requireAdmin, async (req, res) => {
  const { orderId } = req.params;
  const { refundAmount, reason } = req.body;

  try {
    const order = await Order.findById(orderId);

    if (!order || order.status !== 'paid') {
      return res.status(400).json({ error: 'ไม่สามารถ Refund ได้' });
    }

    if (refundAmount && refundAmount > order.totalAmount) {
      return res.status(400).json({ error: 'ยอด Refund เกิน' });
    }

    const refundResult = await refundPayment(
      order.linePayTransactionId,
      refundAmount || null
    );

    if (refundResult.returnCode === '0000') {
      const actualRefund = refundAmount || order.totalAmount;

      await Order.findByIdAndUpdate(orderId, {
        status: refundAmount ? 'partially_refunded' : 'refunded',
        refundAmount: actualRefund,
        refundReason: reason,
        refundedAt: new Date(),
        linePayRefundTransactionId: refundResult.info?.refundTransactionId
      });

      // แจ้งผู้ใช้ผ่าน LINE
      const user = await User.findById(order.userId);
      if (user?.lineUserId) {
        await notifyRefund(user.lineUserId, order, actualRefund);
      }

      res.json({ success: true, refundAmount: actualRefund });
    } else {
      throw new Error(`Refund failed: ${refundResult.returnMessage}`);
    }
  } catch (error) {
    console.error('Refund error:', error);
    res.status(500).json({ error: error.message });
  }
});
```

### Refund Response

```json
{
  "returnCode": "0000",
  "returnMessage": "Success.",
  "info": {
    "refundTransactionId": 2021122300000000001,
    "refundTransactionDate": "2021-12-23T11:00:00Z"
  }
}
```

---

## 8. Check Payment Status

### Get Payment Details

```javascript
// ดูข้อมูล Payment ด้วย Transaction ID
async function getPaymentDetails(transactionId) {
  const endpoint = `/v3/payments?transactionId=${transactionId}`;
  const baseUrl = process.env.LINE_PAY_SANDBOX === 'true'
    ? 'https://sandbox-api-pay.line.me'
    : 'https://api-pay.line.me';

  const headers = createLinePayHeaders(
    process.env.LINE_PAY_CHANNEL_ID,
    process.env.LINE_PAY_CHANNEL_SECRET,
    endpoint // สำหรับ GET request ใช้ path เป็น body
  );

  const response = await axios.get(`${baseUrl}${endpoint}`, { headers });
  return response.data;
}

// ดูข้อมูล Payment ด้วย Order ID
async function getPaymentByOrderId(orderId) {
  const endpoint = `/v3/payments?orderId=${orderId}`;
  const baseUrl = process.env.LINE_PAY_SANDBOX === 'true'
    ? 'https://sandbox-api-pay.line.me'
    : 'https://api-pay.line.me';

  const headers = createLinePayHeaders(
    process.env.LINE_PAY_CHANNEL_ID,
    process.env.LINE_PAY_CHANNEL_SECRET,
    endpoint
  );

  const response = await axios.get(`${baseUrl}${endpoint}`, { headers });
  return response.data;
}
```

### Payment Status Response

```json
{
  "returnCode": "0000",
  "returnMessage": "Success.",
  "info": [
    {
      "transactionId": 2021122300000000000,
      "transactionDate": "2021-12-23T10:00:00Z",
      "transactionType": "PAYMENT",
      "status": "CAPTURE",
      "authorizationExpireDate": "2021-12-30T10:00:00Z",
      "orderId": "order-001",
      "packages": [...],
      "payInfo": [...]
    }
  ]
}
```

### Transaction Status Values

| Status | ความหมาย |
|--------|----------|
| `AUTHORIZATION` | รอ Capture |
| `AUTHORIZATION_VOIDED` | Void แล้ว |
| `CAPTURE` | Capture สำเร็จ |
| `CAPTURE_FAILED` | Capture ล้มเหลว |
| `REFUNDED` | Refund แล้ว |
| `PARTIAL_REFUNDED` | Partial Refund แล้ว |
| `EXPIRED` | หมดอายุ |

---

## 9. Recurring Payment

LINE Pay รองรับ Recurring Payment สำหรับสมาชิกรายเดือน

### ขั้นตอน Recurring Payment

```
1. Request Recurring Payment (สร้าง regKey)
2. ผู้ใช้ยืนยัน + ได้ regKey
3. Confirm แรก
4. ใช้ regKey เพื่อ Charge ครั้งต่อไปโดยไม่ต้องให้ user confirm
```

### Step 1: Request Recurring Payment

```javascript
async function requestRecurringPayment(customerData, planData) {
  const payload = {
    amount: planData.amount,
    currency: 'THB',
    orderId: `subscription-${Date.now()}`,
    packages: [
      {
        id: 'subscription-package',
        amount: planData.amount,
        name: planData.planName,
        products: [
          {
            name: planData.planName,
            quantity: 1,
            price: planData.amount
          }
        ]
      }
    ],
    redirectUrls: {
      confirmUrl: `${process.env.BASE_URL}/line-pay/recurring/confirm`,
      cancelUrl: `${process.env.BASE_URL}/line-pay/cancel`
    },
    options: {
      extra: {
        subscriptionInfo: {
          frequency: planData.frequency, // MONTHLY, WEEKLY, DAILY
          chargeType: 'FIXED' // หรือ VARIABLE
        }
      }
    }
  };

  const endpoint = '/v3/payments/request';
  const baseUrl = process.env.LINE_PAY_SANDBOX === 'true'
    ? 'https://sandbox-api-pay.line.me'
    : 'https://api-pay.line.me';

  const headers = createLinePayHeaders(
    process.env.LINE_PAY_CHANNEL_ID,
    process.env.LINE_PAY_CHANNEL_SECRET,
    JSON.stringify(payload)
  );

  const response = await axios.post(`${baseUrl}${endpoint}`, payload, { headers });
  return response.data;
}
```

### Step 2: Confirm และรับ regKey

```javascript
async function confirmRecurringPayment(transactionId, amount) {
  const payload = {
    amount,
    currency: 'THB'
  };

  const endpoint = `/v3/payments/${transactionId}/confirm`;
  const baseUrl = process.env.LINE_PAY_SANDBOX === 'true'
    ? 'https://sandbox-api-pay.line.me'
    : 'https://api-pay.line.me';

  const headers = createLinePayHeaders(
    process.env.LINE_PAY_CHANNEL_ID,
    process.env.LINE_PAY_CHANNEL_SECRET,
    JSON.stringify(payload)
  );

  const response = await axios.post(`${baseUrl}${endpoint}`, payload, { headers });
  // response.data.info.regKey คือ key สำหรับ charge ครั้งต่อไป
  return response.data;
}

// Callback สำหรับ Recurring
app.get('/line-pay/recurring/confirm', async (req, res) => {
  const { transactionId, orderId } = req.query;

  try {
    const confirmResult = await confirmRecurringPayment(transactionId, /* amount */);

    if (confirmResult.returnCode === '0000') {
      const regKey = confirmResult.info.regKey;

      // บันทึก regKey ลงฐานข้อมูล
      await Subscription.findOneAndUpdate(
        { orderId },
        {
          regKey,
          status: 'active',
          confirmedAt: new Date(),
          nextChargeDate: getNextChargeDate('MONTHLY')
        },
        { upsert: true }
      );

      res.redirect('/subscription/success');
    }
  } catch (error) {
    res.redirect('/subscription/failed');
  }
});
```

### Step 3: Charge ด้วย regKey

```javascript
async function chargeWithRegKey(regKey, amount, orderId) {
  const payload = {
    productName: 'รายการสมาชิกรายเดือน',
    amount,
    currency: 'THB',
    orderId: orderId || `recurring-${Date.now()}`
  };

  const endpoint = `/v3/payments/preapprovedPay/${regKey}/payment`;
  const baseUrl = process.env.LINE_PAY_SANDBOX === 'true'
    ? 'https://sandbox-api-pay.line.me'
    : 'https://api-pay.line.me';

  const headers = createLinePayHeaders(
    process.env.LINE_PAY_CHANNEL_ID,
    process.env.LINE_PAY_CHANNEL_SECRET,
    JSON.stringify(payload)
  );

  const response = await axios.post(`${baseUrl}${endpoint}`, payload, { headers });
  return response.data;
}

// Cron Job สำหรับ Recurring Charge
const cron = require('node-cron');

cron.schedule('0 9 * * *', async () => {
  console.log('Running daily subscription charge...');

  const today = new Date();
  today.setHours(0, 0, 0, 0);
  const tomorrow = new Date(today);
  tomorrow.setDate(tomorrow.getDate() + 1);

  // หา subscriptions ที่ถึงกำหนด charge วันนี้
  const dueSubscriptions = await Subscription.find({
    status: 'active',
    nextChargeDate: { $gte: today, $lt: tomorrow }
  });

  console.log(`Found ${dueSubscriptions.length} subscriptions to charge`);

  for (const subscription of dueSubscriptions) {
    try {
      const orderId = `recurring-${subscription._id}-${Date.now()}`;
      const result = await chargeWithRegKey(
        subscription.regKey,
        subscription.amount,
        orderId
      );

      if (result.returnCode === '0000') {
        // อัพเดต subscription
        await Subscription.findByIdAndUpdate(subscription._id, {
          lastChargedAt: new Date(),
          nextChargeDate: getNextChargeDate(subscription.frequency),
          $push: {
            chargeHistory: {
              orderId,
              amount: subscription.amount,
              chargedAt: new Date(),
              transactionId: result.info?.transactionId
            }
          }
        });

        // แจ้งผู้ใช้
        const user = await User.findById(subscription.userId);
        if (user?.lineUserId) {
          await notifySubscriptionCharged(user.lineUserId, subscription);
        }
      } else {
        throw new Error(`Charge failed: ${result.returnMessage}`);
      }
    } catch (error) {
      console.error(`Failed to charge ${subscription._id}:`, error.message);

      // บันทึก error และ suspend subscription
      await Subscription.findByIdAndUpdate(subscription._id, {
        status: 'charge_failed',
        lastError: error.message,
        lastErrorAt: new Date()
      });
    }
  }
});

function getNextChargeDate(frequency) {
  const now = new Date();
  switch (frequency) {
    case 'MONTHLY': {
      const next = new Date(now);
      next.setMonth(next.getMonth() + 1);
      return next;
    }
    case 'WEEKLY': {
      const next = new Date(now);
      next.setDate(next.getDate() + 7);
      return next;
    }
    case 'DAILY': {
      const next = new Date(now);
      next.setDate(next.getDate() + 1);
      return next;
    }
    default:
      throw new Error(`Unknown frequency: ${frequency}`);
  }
}
```

### ยกเลิก Recurring (Expire regKey)

```javascript
async function expireRegKey(regKey) {
  const endpoint = `/v3/payments/preapprovedPay/${regKey}/expire`;
  const baseUrl = process.env.LINE_PAY_SANDBOX === 'true'
    ? 'https://sandbox-api-pay.line.me'
    : 'https://api-pay.line.me';

  const headers = createLinePayHeaders(
    process.env.LINE_PAY_CHANNEL_ID,
    process.env.LINE_PAY_CHANNEL_SECRET,
    endpoint
  );

  const response = await axios.post(`${baseUrl}${endpoint}`, {}, { headers });
  return response.data;
}
```

---

## 10. LINE Pay กับ LINE OA

### ส่ง Payment Link ผ่าน LINE Bot

```javascript
const { Client } = require('@line/bot-sdk');

const lineClient = new Client({
  channelAccessToken: process.env.CHANNEL_ACCESS_TOKEN
});

// ส่ง Payment Flex Message
async function sendPaymentMessage(userId, order, paymentUrl) {
  const message = {
    type: 'flex',
    altText: `คำสั่งซื้อ #${order.orderNumber} - ยอด ${order.totalAmount} บาท`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: 'ยืนยันการชำระเงิน',
            color: '#ffffff',
            size: 'lg',
            weight: 'bold'
          }
        ],
        backgroundColor: '#00B900'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: `คำสั่งซื้อ #${order.orderNumber}`,
            weight: 'bold',
            size: 'md'
          },
          {
            type: 'separator',
            margin: 'md'
          },
          ...order.items.map(item => ({
            type: 'box',
            layout: 'horizontal',
            margin: 'sm',
            contents: [
              { type: 'text', text: item.name, flex: 3, size: 'sm' },
              { type: 'text', text: `x${item.quantity}`, flex: 1, size: 'sm', align: 'center' },
              { type: 'text', text: `${item.price * item.quantity} ฿`, flex: 2, size: 'sm', align: 'end' }
            ]
          })),
          {
            type: 'separator',
            margin: 'md'
          },
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'md',
            contents: [
              { type: 'text', text: 'ยอดรวม', weight: 'bold' },
              { type: 'text', text: `${order.totalAmount} บาท`, weight: 'bold', align: 'end', color: '#00B900' }
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
              label: 'ชำระด้วย LINE Pay',
              uri: paymentUrl
            },
            style: 'primary',
            color: '#00B900'
          },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: 'ยกเลิกคำสั่งซื้อ',
              data: `action=cancel_order&orderId=${order._id}`
            },
            style: 'secondary',
            margin: 'sm'
          }
        ]
      }
    }
  };

  await lineClient.pushMessage(userId, message);
}

// ส่งข้อความยืนยันการชำระเงิน
async function sendOrderConfirmation(userId, order) {
  const message = {
    type: 'flex',
    altText: `ชำระเงินสำเร็จ - คำสั่งซื้อ #${order.orderNumber}`,
    contents: {
      type: 'bubble',
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'box',
            layout: 'horizontal',
            contents: [
              {
                type: 'text',
                text: '✅',
                size: 'xxl',
                flex: 0
              },
              {
                type: 'box',
                layout: 'vertical',
                flex: 1,
                margin: 'md',
                contents: [
                  {
                    type: 'text',
                    text: 'ชำระเงินสำเร็จ',
                    weight: 'bold',
                    size: 'lg'
                  },
                  {
                    type: 'text',
                    text: `คำสั่งซื้อ #${order.orderNumber}`,
                    color: '#666666'
                  }
                ]
              }
            ]
          },
          {
            type: 'separator',
            margin: 'lg'
          },
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'lg',
            contents: [
              { type: 'text', text: 'ยอดชำระ', color: '#666666' },
              { type: 'text', text: `฿${order.totalAmount}`, align: 'end', weight: 'bold' }
            ]
          },
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'sm',
            contents: [
              { type: 'text', text: 'วันที่', color: '#666666' },
              { type: 'text', text: new Date().toLocaleDateString('th-TH'), align: 'end' }
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
              type: 'postback',
              label: 'ดูคำสั่งซื้อ',
              data: `action=view_order&orderId=${order._id}`
            },
            style: 'primary'
          }
        ]
      }
    }
  };

  await lineClient.pushMessage(userId, message);
}
```

---

## 11. Thailand-specific LINE Pay Setup

### การตั้งค่าสำหรับไทย

```javascript
// config/line-pay.js
const linePayConfig = {
  // Thailand Merchant Settings
  channelId: process.env.LINE_PAY_CHANNEL_ID,
  channelSecret: process.env.LINE_PAY_CHANNEL_SECRET,
  isSandbox: process.env.NODE_ENV !== 'production',

  // Thailand Payment Methods
  supportedPaymentMethods: [
    'CREDIT_CARD',   // บัตรเครดิต
    'BALANCE',       // LINE Pay Balance
    'POINT',         // LINE Points
    'DISCOUNT'       // ส่วนลด
  ],

  // Thai Baht
  currency: 'THB',

  // Thailand merchant settings
  locale: 'th',

  // Webhook (สำหรับ Server-side confirmation)
  webhookUrl: process.env.LINE_PAY_WEBHOOK_URL
};

// Middleware สำหรับ validate LINE Pay Webhook
function validateLinePayWebhook(req, res, next) {
  // LINE Pay ส่ง HMAC ใน Header
  const signature = req.headers['x-line-authorization'];
  const nonce = req.headers['x-line-authorization-nonce'];
  const body = JSON.stringify(req.body);

  const expectedSig = crypto
    .createHmac('SHA256', linePayConfig.channelSecret)
    .update(linePayConfig.channelSecret + body + nonce)
    .digest('base64');

  if (signature !== expectedSig) {
    return res.status(401).json({ error: 'Invalid signature' });
  }

  next();
}
```

### PromptPay Integration (Alternative)

```javascript
// สร้าง PromptPay QR Code สำหรับ Fallback
const QRCode = require('qrcode');

async function generatePromptPayQR(amount, promptPayId) {
  // PromptPay QR Format
  const payload = generateEMVPayload({
    amount,
    promptPayId, // เบอร์โทรหรือ Tax ID
    ref: Date.now().toString()
  });

  const qrDataUrl = await QRCode.toDataURL(payload);
  return qrDataUrl;
}

// ส่ง PromptPay QR ผ่าน LINE
async function sendPromptPayQR(userId, amount, orderId) {
  const qrDataUrl = await generatePromptPayQR(amount, process.env.PROMPTPAY_ID);

  // แปลง Data URL เป็น Buffer
  const base64Data = qrDataUrl.replace(/^data:image\/png;base64,/, '');
  const imageBuffer = Buffer.from(base64Data, 'base64');

  // อัพโหลดไปยัง server ก่อน (LINE ต้องการ public URL)
  const imageUrl = await uploadImageToServer(imageBuffer);

  await lineClient.pushMessage(userId, {
    type: 'image',
    originalContentUrl: imageUrl,
    previewImageUrl: imageUrl
  });

  await lineClient.pushMessage(userId, {
    type: 'text',
    text: `QR Code สำหรับชำระเงิน ฿${amount}\nคำสั่งซื้อ #${orderId}\n\nกรุณาชำระภายใน 30 นาที`
  });
}
```

---

## 12. Node.js Integration Example

### LINE Pay Service Class

```javascript
// services/LinePayService.js
const axios = require('axios');
const crypto = require('crypto');

class LinePayService {
  constructor(config = {}) {
    this.channelId = config.channelId || process.env.LINE_PAY_CHANNEL_ID;
    this.channelSecret = config.channelSecret || process.env.LINE_PAY_CHANNEL_SECRET;
    this.isSandbox = config.sandbox ?? (process.env.LINE_PAY_SANDBOX === 'true');
    this.baseUrl = this.isSandbox
      ? 'https://sandbox-api-pay.line.me'
      : 'https://api-pay.line.me';
    this.confirmUrl = config.confirmUrl || process.env.LINE_PAY_CONFIRM_URL;
    this.cancelUrl = config.cancelUrl || process.env.LINE_PAY_CANCEL_URL;
  }

  // สร้าง Headers
  _createHeaders(endpoint, payload = {}) {
    const nonce = crypto.randomBytes(8).toString('hex');
    const body = typeof payload === 'string'
      ? payload
      : Object.keys(payload).length > 0
        ? JSON.stringify(payload)
        : endpoint;

    const signature = crypto
      .createHmac('SHA256', this.channelSecret)
      .update(this.channelSecret + body + nonce)
      .digest('base64');

    return {
      'Content-Type': 'application/json',
      'X-LINE-ChannelId': this.channelId,
      'X-LINE-Authorization-Nonce': nonce,
      'X-LINE-Authorization': signature
    };
  }

  async _get(endpoint) {
    const headers = this._createHeaders(endpoint);
    const response = await axios.get(`${this.baseUrl}${endpoint}`, { headers });
    return this._handleResponse(response.data);
  }

  async _post(endpoint, payload) {
    const headers = this._createHeaders(endpoint, payload);
    const response = await axios.post(`${this.baseUrl}${endpoint}`, payload, { headers });
    return this._handleResponse(response.data);
  }

  _handleResponse(data) {
    if (data.returnCode !== '0000') {
      const error = new Error(`LINE Pay Error: ${data.returnMessage}`);
      error.returnCode = data.returnCode;
      error.returnMessage = data.returnMessage;
      throw error;
    }
    return data.info || data;
  }

  // ─── Payment Methods ────────────────────────────────────────────

  async requestPayment({ amount, orderId, products, options = {} }) {
    const payload = {
      amount,
      currency: 'THB',
      orderId,
      packages: [
        {
          id: `pkg-${orderId}`,
          amount,
          name: options.packageName || 'คำสั่งซื้อ',
          products: products.map(p => ({
            id: p.id,
            name: p.name,
            imageUrl: p.imageUrl,
            quantity: p.quantity,
            price: p.price
          }))
        }
      ],
      redirectUrls: {
        confirmUrl: `${this.confirmUrl}?orderId=${orderId}`,
        cancelUrl: `${this.cancelUrl}?orderId=${orderId}`,
        confirmUrlType: options.confirmUrlType || 'SERVER'
      },
      options: {
        payment: { capture: options.autoCapture !== false },
        display: { locale: 'th' },
        ...options.linePayOptions
      }
    };

    return await this._post('/v3/payments/request', payload);
  }

  async confirmPayment(transactionId, amount) {
    return await this._post(
      `/v3/payments/${transactionId}/confirm`,
      { amount, currency: 'THB' }
    );
  }

  async getPaymentStatus(transactionId) {
    return await this._get(`/v3/payments?transactionId=${transactionId}`);
  }

  async getPaymentByOrderId(orderId) {
    return await this._get(`/v3/payments?orderId=${orderId}`);
  }

  async refund(transactionId, refundAmount = null) {
    const payload = {};
    if (refundAmount !== null) payload.refundAmount = refundAmount;
    return await this._post(`/v3/payments/${transactionId}/refund`, payload);
  }

  async captureAuthorization(transactionId, amount) {
    return await this._post(
      `/v3/payments/authorizations/${transactionId}/capture`,
      { amount, currency: 'THB' }
    );
  }

  async voidAuthorization(transactionId) {
    return await this._post(
      `/v3/payments/authorizations/${transactionId}/void`,
      {}
    );
  }

  // Recurring
  async chargeRecurring(regKey, amount, productName) {
    return await this._post(
      `/v3/payments/preapprovedPay/${regKey}/payment`,
      {
        productName,
        amount,
        currency: 'THB',
        orderId: `recurring-${Date.now()}`
      }
    );
  }

  async expireRegKey(regKey) {
    return await this._post(`/v3/payments/preapprovedPay/${regKey}/expire`, {});
  }
}

module.exports = LinePayService;
```

### Controller

```javascript
// controllers/paymentController.js
const LinePayService = require('../services/LinePayService');
const Order = require('../models/Order');

const linePay = new LinePayService();

exports.createPayment = async (req, res) => {
  const { items, deliveryFee = 0 } = req.body;
  const userId = req.session.userId;

  try {
    const subtotal = items.reduce((sum, i) => sum + i.price * i.quantity, 0);
    const total = subtotal + deliveryFee;

    // สร้าง Order
    const order = await Order.create({
      userId,
      items,
      subtotal,
      deliveryFee,
      totalAmount: total,
      status: 'pending',
      paymentMethod: 'line_pay'
    });

    // Request Payment
    const paymentInfo = await linePay.requestPayment({
      amount: total,
      orderId: order._id.toString(),
      products: items.map(i => ({
        id: i.productId,
        name: i.name,
        imageUrl: i.imageUrl,
        quantity: i.quantity,
        price: i.price
      }))
    });

    // บันทึก Transaction ID
    order.linePayTransactionId = paymentInfo.transactionId;
    await order.save();

    res.json({
      orderId: order._id,
      paymentUrl: paymentInfo.paymentUrl.web,
      mobilePaymentUrl: paymentInfo.paymentUrl.app
    });
  } catch (error) {
    console.error('Create payment error:', error);
    res.status(500).json({ error: 'ไม่สามารถสร้างการชำระเงินได้' });
  }
};

exports.confirmPayment = async (req, res) => {
  const { transactionId, orderId } = req.query;

  try {
    const order = await Order.findById(orderId);
    if (!order) return res.redirect('/orders?error=not_found');

    // Confirm
    const confirmInfo = await linePay.confirmPayment(
      transactionId,
      order.totalAmount
    );

    order.status = 'paid';
    order.paidAt = new Date();
    order.linePayConfirmInfo = confirmInfo;
    await order.save();

    res.redirect(`/orders/${orderId}?success=1`);
  } catch (error) {
    console.error('Confirm payment error:', error);
    res.redirect(`/orders/${orderId}?error=payment_failed`);
  }
};
```

---

## 13. Error Codes

### LINE Pay Error Codes

| Return Code | ความหมาย | วิธีแก้ไข |
|-------------|---------|----------|
| `0000` | สำเร็จ | - |
| `1101` | Buyer cancelled | แจ้งผู้ใช้ |
| `1102` | Buyer blocked by LINE Pay | แจ้งผู้ใช้ติดต่อ LINE Pay |
| `1104` | Merchant not found | ตรวจสอบ Channel ID |
| `1105` | Merchant status invalid | ติดต่อ LINE Pay |
| `1106` | Header info error | ตรวจสอบ Headers |
| `1110` | Not in scope | ตรวจสอบ Permission |
| `1124` | Amount mismatch | ตรวจสอบยอดเงิน |
| `1141` | Order not found | ตรวจสอบ Order ID |
| `1142` | Already ordered | Order ID ซ้ำ |
| `1153` | Order amount error | ตรวจสอบยอดเงิน |
| `1154` | Product not found | ตรวจสอบสินค้า |
| `1172` | Pre-approved payment expired | regKey หมดอายุ |
| `1180` | Already void | Void ซ้ำ |
| `1190` | Already captured | Capture ซ้ำ |
| `9000` | Internal error | ลองใหม่ภายหลัง |

```javascript
// Error Handler
function handleLinePayError(error) {
  const messages = {
    '1101': 'ผู้ใช้ยกเลิกการชำระเงิน',
    '1102': 'บัญชี LINE Pay ถูกระงับ',
    '1124': 'จำนวนเงินไม่ถูกต้อง',
    '1142': 'หมายเลขคำสั่งซื้อซ้ำกัน',
    '9000': 'เกิดข้อผิดพลาดจาก LINE Pay กรุณาลองใหม่'
  };

  const code = error.returnCode;
  const userMessage = messages[code] || `เกิดข้อผิดพลาด (${code})`;

  console.error(`LINE Pay Error [${code}]: ${error.returnMessage}`);
  return userMessage;
}
```

---

## 14. Testing ใน Sandbox

### Sandbox Test Cards

```
VISA:   4111 1111 1111 1111  CVV: 123  Exp: 12/26
Master: 5500 0000 0000 0004  CVV: 123  Exp: 12/26
JCB:    3530 1113 3330 0000  CVV: 123  Exp: 12/26
```

### Test Scenarios

```javascript
// Test Helper
async function testLinePayIntegration() {
  const linePay = new LinePayService({ sandbox: true });

  console.log('=== Testing LINE Pay Integration ===\n');

  // Test 1: Request Payment
  console.log('Test 1: Request Payment');
  let paymentInfo;
  try {
    paymentInfo = await linePay.requestPayment({
      amount: 100,
      orderId: `test-order-${Date.now()}`,
      products: [{ id: 'p001', name: 'Test Product', quantity: 1, price: 100 }]
    });
    console.log('✅ Request Payment: OK');
    console.log('   Payment URL:', paymentInfo.paymentUrl.web);
  } catch (error) {
    console.log('❌ Request Payment Failed:', error.message);
  }

  // Test 2: Get Payment Status
  if (paymentInfo?.transactionId) {
    console.log('\nTest 2: Get Payment Status');
    try {
      const status = await linePay.getPaymentStatus(paymentInfo.transactionId);
      console.log('✅ Get Status: OK - Status:', status[0]?.status);
    } catch (error) {
      console.log('❌ Get Status Failed:', error.message);
    }
  }

  console.log('\n=== Test Complete ===');
}

// ตรวจสอบ Sandbox Configuration
function validateSandboxConfig() {
  const required = [
    'LINE_PAY_CHANNEL_ID',
    'LINE_PAY_CHANNEL_SECRET',
    'LINE_PAY_CONFIRM_URL'
  ];

  const missing = required.filter(key => !process.env[key]);

  if (missing.length > 0) {
    console.error('Missing environment variables:', missing);
    return false;
  }

  if (process.env.LINE_PAY_SANDBOX !== 'true') {
    console.warn('⚠️ LINE_PAY_SANDBOX is not set to "true"');
  }

  console.log('✅ LINE Pay configuration valid');
  console.log('   Channel ID:', process.env.LINE_PAY_CHANNEL_ID);
  console.log('   Sandbox:', process.env.LINE_PAY_SANDBOX === 'true');
  return true;
}
```

---

## สรุป LINE Pay

| ขั้นตอน | API | หมายเหตุ |
|--------|-----|---------|
| สร้างคำขอ | `POST /v3/payments/request` | ได้ paymentUrl |
| ยืนยันการชำระ | `POST /v3/payments/{id}/confirm` | หลัง Redirect กลับ |
| ตรวจสอบสถานะ | `GET /v3/payments?transactionId={id}` | ทุกเวลา |
| Refund | `POST /v3/payments/{id}/refund` | หลัง Capture |
| Recurring Charge | `POST /v3/payments/preapprovedPay/{key}/payment` | ใช้ regKey |

---

*ส่วนถัดไป: [Part 54: Notification Channel](part-054-notification-channel.md)*
