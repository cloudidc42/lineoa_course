# ส่วนที่ 7: การตั้งค่า Auto-Reply

## บทนำ

Auto-Reply หรือการตอบกลับอัตโนมัติ คือหนึ่งในเครื่องมือที่สำคัญที่สุดสำหรับ LINE Official Account เพราะช่วยให้ธุรกิจของคุณสามารถ "ตอบ" ลูกค้าได้ตลอด 24 ชั่วโมง 7 วันต่อสัปดาห์ แม้ในช่วงที่ไม่มีพนักงานออนไลน์

ในบทนี้คุณจะเรียนรู้:
- ประเภทของ Auto-Reply ทั้งหมด
- การตั้งค่า Keyword-based Auto-Reply
- การตั้งค่า Time-based Auto-Reply
- การสร้าง Greeting Message
- การตั้งค่า Away Message
- Pattern และตัวอย่างที่ใช้งานได้จริง

---

## 7.1 ประเภทของ Auto-Reply

LINE Official Account รองรับ Auto-Reply หลายประเภท ซึ่งแต่ละประเภทมีจุดประสงค์และวิธีทำงานที่แตกต่างกัน

### ภาพรวมประเภท Auto-Reply

```
Auto-Reply ใน LINE OA
│
├── Greeting Message (ข้อความต้อนรับ)
│   └── ส่งเมื่อมีคนเพิ่มเพื่อนครั้งแรก
│
├── Keyword Auto-Reply (ตอบตาม Keyword)
│   ├── Match แบบเต็ม (Exact match)
│   ├── Match บางส่วน (Partial match)  
│   └── Match หลาย Keyword
│
├── Time-based Auto-Reply (ตอบตามเวลา)
│   ├── Away Message (นอกเวลาทำการ)
│   └── Holiday Message (วันหยุด)
│
└── Default Reply (ตอบกรณีไม่ Match)
    └── ข้อความ Fallback
```

### ตารางเปรียบเทียบประเภท Auto-Reply

| ประเภท | Trigger | เหมาะสำหรับ | ความซับซ้อน |
|--------|---------|------------|------------|
| Greeting | เพิ่มเพื่อน | ต้อนรับใหม่ | ต่ำ |
| Keyword | ข้อความที่มี Keyword | FAQ, Menu | ปานกลาง |
| Time-based | ช่วงเวลา | นอกเวลาทำการ | ต่ำ |
| Default | ไม่ Match อะไร | Fallback | ต่ำ |

---

## 7.2 Keyword-based Auto-Reply

### หลักการทำงาน

เมื่อผู้ใช้ส่งข้อความที่มีคำสำคัญ (Keyword) ที่กำหนดไว้ ระบบจะตอบกลับด้วยข้อความที่ตั้งไว้โดยอัตโนมัติ

```
ผู้ใช้พิมพ์: "ราคา"
        ↓
ระบบตรวจสอบ Keyword
        ↓
พบ Match กับ Keyword "ราคา"
        ↓
ส่ง Auto-Reply ข้อมูลราคา
```

### 7.2.1 การตั้งค่าผ่าน LINE OA Manager

**ขั้นตอน:**
1. เข้า [manager.line.biz](https://manager.line.biz)
2. เลือก Official Account ของคุณ
3. ไปที่เมนู **"Auto-reply messages"** หรือ **"ข้อความตอบกลับอัตโนมัติ"**
4. คลิก **"Create"** หรือ **"สร้าง"**
5. ในส่วน "Keywords" ใส่คำสำคัญที่ต้องการ
6. สร้างข้อความตอบกลับ
7. คลิก **"Save"**

### 7.2.2 โหมด Matching

**1. Contains (มีคำนั้น)**
- ตอบกลับเมื่อข้อความมีคำนั้นอยู่ ไม่ว่าจะอยู่ตรงไหน
- ตัวอย่าง: Keyword = "ราคา" → จะ Match กับ "ราคาสินค้าเท่าไหร่" หรือ "ขอดูราคา"

**2. Equals (เท่ากับ)**
- ตอบกลับเฉพาะเมื่อข้อความตรงกันทั้งหมด
- ตัวอย่าง: Keyword = "ราคา" → Match เฉพาะ "ราคา" เท่านั้น ไม่ Match "ขอดูราคา"

> **เคล็ดลับ**: ใช้โหมด "Contains" สำหรับ FAQ ทั่วไป และ "Equals" สำหรับ Menu Commands เช่น "เมนู" หรือ "เริ่มต้น"

### 7.2.3 ตัวอย่าง Keyword Sets สำหรับธุรกิจต่างๆ

**ร้านค้าออนไลน์:**

```
Keyword Group 1: ราคา/Price
Keywords: ราคา, price, เท่าไหร่, cost, ราคาเท่าไร, how much
Reply:
"สวัสดีครับ! สอบถามราคาสินค้าได้ที่นี่เลยครับ 😊
📋 ดูรายการสินค้าและราคาทั้งหมด:
https://example.com/products

หรือแจ้งชื่อสินค้าที่สนใจมาได้เลยครับ เดี๋ยวรีบตอบให้นะครับ 💬"

---

Keyword Group 2: สั่งซื้อ/Order
Keywords: สั่ง, order, จอง, สั่งซื้อ, ซื้อ, buy
Reply:
"ขอบคุณที่สนใจสินค้าของเรานะครับ! 🛒
วิธีสั่งซื้อมี 2 ช่องทาง:
1️⃣ สั่งผ่านเว็บ: https://example.com/order
2️⃣ แจ้งรายการที่นี่ แล้วทีมงานจะติดต่อกลับ

ต้องการความช่วยเหลือเพิ่มเติมพิมพ์ 'ช่วยเหลือ' ได้เลยครับ"

---

Keyword Group 3: จัดส่ง/Delivery
Keywords: จัดส่ง, shipping, ส่ง, delivery, ส่งของ, กี่วัน
Reply:
"📦 ข้อมูลการจัดส่ง:
• ส่งทุกวัน จันทร์-เสาร์ (ยกเว้นวันหยุดนักขัตฤกษ์)
• ตัดรอบ 14:00 น. ส่งวันเดียวกัน
• ระยะเวลา: 1-3 วันทำการ
• Kerry / J&T / Flash Express

📍 ติดตามพัสดุ: https://example.com/tracking"
```

**ร้านอาหาร:**

```
Keyword Group 1: โต๊ะ/จอง
Keywords: จอง, reserve, โต๊ะ, table, จองโต๊ะ
Reply:
"🍽️ จองโต๊ะที่ [ชื่อร้าน]
📞 โทร: 02-XXX-XXXX
🌐 หรือจองออนไลน์: https://example.com/reserve

กรุณาแจ้ง:
• วันที่และเวลา
• จำนวนคน
• ชื่อผู้จอง"

---

Keyword Group 2: เมนู/Menu
Keywords: เมนู, menu, อาหาร, รายการ
Reply:
"🍜 ดูเมนูทั้งหมดของเราได้ที่:
https://example.com/menu

หรือดู Highlight เมนูแนะนำ:
⭐ [ชื่อเมนู 1] - xxx บาท
⭐ [ชื่อเมนู 2] - xxx บาท
⭐ [ชื่อเมนู 3] - xxx บาท"
```

**คลินิก/โรงพยาบาล:**

```
Keyword Group 1: นัด/Appointment
Keywords: นัด, appointment, จองคิว, นัดหมาย
Reply:
"📅 นัดหมายพบแพทย์
วิธีนัดหมาย:
1. โทร 02-XXX-XXXX (08:00-18:00 น.)
2. LINE Official (วันนี้ทีมงานจะติดต่อกลับ)
3. ผ่านแอป: https://example.com/app

กรุณาเตรียม: บัตรประชาชน, บัตรทอง/ประกัน"

---

Keyword Group 2: เวลาเปิด/Open hours
Keywords: เปิด, ปิด, เวลา, hours, กี่โมง, เวลาเปิด
Reply:
"🏥 เวลาทำการ [ชื่อคลินิก]
จันทร์-ศุกร์: 08:00-20:00 น.
เสาร์: 08:00-17:00 น.
อาทิตย์: ปิดทำการ (ยกเว้นฉุกเฉิน โทร 02-XXX-XXXX)"
```

### 7.2.4 Multi-keyword Matching

สามารถตั้งให้ข้อความเดียวตอบกลับได้จากหลาย Keyword:

```
Keywords (ตั้งทีละบรรทัด):
สวัสดี
hello
hi
หวัดดี
ดีครับ
ดีค่ะ

Reply:
"สวัสดีครับ! ยินดีต้อนรับสู่ [ชื่อร้าน] 👋

พิมพ์ได้เลยครับ:
📌 เมนู - ดูสินค้าทั้งหมด
💰 ราคา - ดูราคาสินค้า
📦 สั่ง - วิธีสั่งซื้อ
🚚 จัดส่ง - ข้อมูลการส่ง
☎️ ติดต่อ - ช่องทางติดต่อ"
```

### 7.2.5 การตั้งค่า Auto-Reply ผ่าน API

สำหรับนักพัฒนาที่ต้องการควบคุม Auto-Reply แบบ Programmatic:

**หมายเหตุ**: Auto-Reply ผ่าน API ต้องใช้ Webhook และการจัดการ Event ฝั่ง Server

```javascript
// webhook.js - จัดการ Message Event
const express = require('express');
const { Client, middleware } = require('@line/bot-sdk');

const config = {
  channelSecret: process.env.LINE_CHANNEL_SECRET,
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
};

const client = new Client(config);
const app = express();

// Keyword responses mapping
const keywordReplies = {
  'ราคา': {
    type: 'text',
    text: '📋 ดูราคาสินค้าทั้งหมดได้ที่: https://example.com/price'
  },
  'สั่ง': {
    type: 'text', 
    text: '🛒 วิธีสั่งซื้อ:\n1. เว็บไซต์: https://example.com\n2. แจ้งที่นี่ได้เลยครับ'
  },
  'จัดส่ง': {
    type: 'text',
    text: '📦 จัดส่งใน 1-3 วันทำการ\nตรวจสอบพัสดุ: https://example.com/track'
  }
};

// Handle text messages with keyword matching
async function handleTextMessage(event) {
  const userMessage = event.message.text.toLowerCase();
  const replyToken = event.replyToken;
  
  // Check keywords (partial match)
  for (const [keyword, reply] of Object.entries(keywordReplies)) {
    if (userMessage.includes(keyword.toLowerCase())) {
      await client.replyMessage(replyToken, reply);
      return;
    }
  }
  
  // Default reply if no keyword matched
  await client.replyMessage(replyToken, {
    type: 'text',
    text: 'ขอบคุณสำหรับข้อความนะครับ 😊 ทีมงานจะติดต่อกลับเร็วๆ นี้\n\nหรือพิมพ์ "เมนู" เพื่อดูตัวเลือกทั้งหมดครับ'
  });
}

app.post('/webhook', middleware(config), async (req, res) => {
  try {
    const events = req.body.events;
    
    await Promise.all(
      events.map(async (event) => {
        if (event.type === 'message' && event.message.type === 'text') {
          await handleTextMessage(event);
        }
      })
    );
    
    res.status(200).json({ status: 'ok' });
  } catch (error) {
    console.error('Webhook error:', error);
    res.status(500).json({ error: 'Internal Server Error' });
  }
});
```

```python
# Python ด้วย Flask
from flask import Flask, request, abort
from linebot import LineBotApi, WebhookHandler
from linebot.exceptions import InvalidSignatureError
from linebot.models import MessageEvent, TextMessage, TextSendMessage
import os

app = Flask(__name__)

line_bot_api = LineBotApi(os.environ['LINE_CHANNEL_ACCESS_TOKEN'])
handler = WebhookHandler(os.environ['LINE_CHANNEL_SECRET'])

# Keyword responses
KEYWORD_REPLIES = {
    'ราคา': 'ดูราคาสินค้าทั้งหมด: https://example.com/price',
    'เมนู': '📋 เมนูสินค้า:\n1. สินค้า A - 100 บาท\n2. สินค้า B - 200 บาท',
    'สั่ง': '🛒 วิธีสั่งซื้อ: https://example.com/order',
    'ติดต่อ': '📞 โทร: 02-XXX-XXXX\n📧 Email: info@example.com',
}

DEFAULT_REPLY = (
    'ขอบคุณสำหรับข้อความนะครับ 🙏\n'
    'ทีมงานจะติดต่อกลับเร็วๆ นี้\n\n'
    'หรือพิมพ์:\n'
    '📌 เมนู\n'
    '💰 ราคา\n'
    '🛒 สั่ง\n'
    '📞 ติดต่อ'
)


@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('X-Line-Signature', '')
    body = request.get_data(as_text=True)
    
    try:
        handler.handle(body, signature)
    except InvalidSignatureError:
        abort(400)
    
    return 'OK'


@handler.add(MessageEvent, message=TextMessage)
def handle_message(event):
    user_message = event.message.text.lower()
    
    # ตรวจสอบ Keyword
    for keyword, reply in KEYWORD_REPLIES.items():
        if keyword.lower() in user_message:
            line_bot_api.reply_message(
                event.reply_token,
                TextSendMessage(text=reply)
            )
            return
    
    # Default reply
    line_bot_api.reply_message(
        event.reply_token,
        TextSendMessage(text=DEFAULT_REPLY)
    )


if __name__ == '__main__':
    app.run(debug=True, port=8080)
```

---

## 7.3 Time-based Auto-Reply

### 7.3.1 หลักการทำงาน

Time-based Auto-Reply จะทำงานตามช่วงเวลาที่กำหนด ตัวอย่างเช่น ส่งข้อความตอบกลับอัตโนมัติเมื่อผู้ใช้ส่งข้อความในช่วงนอกเวลาทำการ

### 7.3.2 การตั้งค่าใน LINE OA Manager

**ขั้นตอนการตั้งค่า:**

1. ไปที่ **"Chat settings"** > **"Auto-reply messages"**
2. คลิก **"Response Mode"**
3. เลือก **"Bot"** หรือ **"Bot and Chat"**
4. ตั้งค่า Operating hours:
   - เลือก **"Set operating hours"**
   - กำหนดวันทำการ (เลือกวัน)
   - กำหนดเวลาเริ่มต้น-สิ้นสุด
5. ตั้งค่า Away message สำหรับนอกเวลาทำการ

### 7.3.3 ตัวอย่าง Away Messages

**สำหรับร้านค้าออนไลน์:**

```
ขอบคุณที่ติดต่อ [ชื่อร้าน] นะครับ 🙏

ขณะนี้อยู่นอกเวลาทำการ
⏰ เวลาทำการ: จ-ศ 9:00-18:00 น.

ทีมงานจะตอบกลับทันทีในวันทำการถัดไปครับ

สามารถดูข้อมูลเพิ่มเติมได้ที่:
🌐 https://example.com/faq
```

**สำหรับคลินิก:**

```
สวัสดีครับ นี่คือระบบตอบกลับอัตโนมัติของ [ชื่อคลินิก]

⏰ เวลาทำการ:
• จันทร์-ศุกร์: 8:00-18:00 น.
• เสาร์: 8:00-16:00 น.

🚨 กรณีฉุกเฉิน: โทร 1669

📅 นัดหมายวันทำการถัดไป: 
พิมพ์ "นัด" หรือโทร 02-XXX-XXXX
```

### 7.3.4 การตั้งค่า Time-based Reply ผ่าน Code

```javascript
// ฟังก์ชันตรวจสอบเวลาทำการ
function isBusinessHours() {
  const now = new Date();
  // แปลงเป็น Bangkok timezone (UTC+7)
  const bangkokTime = new Date(now.toLocaleString('en-US', { timeZone: 'Asia/Bangkok' }));
  
  const dayOfWeek = bangkokTime.getDay(); // 0=Sunday, 6=Saturday
  const hour = bangkokTime.getHours();
  const minute = bangkokTime.getMinutes();
  
  // วันทำการ: จันทร์(1) - ศุกร์(5)
  const isWeekday = dayOfWeek >= 1 && dayOfWeek <= 5;
  
  // เวลาทำการ: 9:00 - 18:00
  const currentMinutes = hour * 60 + minute;
  const startMinutes = 9 * 60;  // 9:00
  const endMinutes = 18 * 60;   // 18:00
  
  const isWorkingHour = currentMinutes >= startMinutes && currentMinutes < endMinutes;
  
  return isWeekday && isWorkingHour;
}

// ใช้ใน Message Handler
async function handleMessage(event) {
  const replyToken = event.replyToken;
  const userMessage = event.message.text;
  
  if (!isBusinessHours()) {
    // ส่ง Away Message
    await client.replyMessage(replyToken, {
      type: 'text',
      text: `ขอบคุณที่ติดต่อนะครับ 🙏

ขณะนี้อยู่นอกเวลาทำการ
⏰ เวลาทำการ: จ-ศ 9:00-18:00 น.

ทีมงานจะตอบกลับทันทีในวันทำการถัดไปครับ`
    });
    return;
  }
  
  // ตอบกลับปกติในเวลาทำการ
  await handleNormalReply(event);
}


// ตรวจสอบวันหยุดนักขัตฤกษ์ไทย
const THAI_HOLIDAYS_2024 = [
  '2024-01-01', // วันขึ้นปีใหม่
  '2024-02-26', // มาฆบูชา
  '2024-04-06', // จักรี
  '2024-04-13', // สงกรานต์
  '2024-04-14', // สงกรานต์
  '2024-04-15', // สงกรานต์
  // ... เพิ่มวันหยุดที่เหลือ
];

function isHoliday() {
  const now = new Date();
  const bangkokDate = now.toLocaleDateString('en-CA', { timeZone: 'Asia/Bangkok' });
  return THAI_HOLIDAYS_2024.includes(bangkokDate);
}
```

---

## 7.4 Greeting Message (ข้อความต้อนรับ)

### 7.4.1 หน้าที่ของ Greeting Message

Greeting Message คือข้อความที่ส่งโดยอัตโนมัติทันทีที่มีผู้ใช้เพิ่ม LINE OA ของคุณเป็นเพื่อนครั้งแรก นี่คือโอกาสทองในการสร้างความประทับใจแรก

**เป้าหมายของ Greeting Message:**
- ยืนยันว่าเพิ่มเพื่อนสำเร็จ
- แนะนำบัญชีและบริการ
- บอกวิธีใช้งาน
- สร้าง Engagement ตั้งแต่แรก
- เสนอ Incentive สำหรับผู้ใหม่

### 7.4.2 การตั้งค่า Greeting Message

1. ไปที่ **"Chat settings"** ใน LINE OA Manager
2. หา **"Greeting message"**
3. Toggle เปิดใช้งาน
4. สร้างข้อความต้อนรับ

### 7.4.3 Template Greeting Messages

**Template 1: ง่ายและเป็นมิตร (ร้านค้า)**

```
สวัสดีครับ! ขอบคุณที่เพิ่ม [ชื่อร้าน] เป็นเพื่อนนะครับ 🎉

เราเป็นร้าน [ประเภทธุรกิจ] ที่ [จุดเด่น]

🎁 ของขวัญต้อนรับสมาชิกใหม่:
รับโค้ดส่วนลด 10% สำหรับการสั่งซื้อครั้งแรก
👉 โค้ด: WELCOME10

พิมพ์ได้เลยครับ:
📋 เมนู - ดูสินค้าทั้งหมด
💰 ราคา - สอบถามราคา
🛒 สั่ง - วิธีสั่งซื้อ
```

**Template 2: Professional (บริการ B2B)**

```
ยินดีต้อนรับสู่ [ชื่อบริษัท] Official Account

เราให้บริการ [ประเภทบริการ] สำหรับ [กลุ่มลูกค้า]

📌 สิ่งที่คุณสามารถทำได้ที่นี่:
• สอบถามข้อมูลบริการ
• นัดหมายพบผู้เชี่ยวชาญ
• ติดตามสถานะงาน
• รับข่าวสารและอัปเดต

💬 เราพร้อมให้บริการ จ-ศ 8:00-17:00 น.
📞 หรือโทร: 02-XXX-XXXX
```

**Template 3: Interactive (พร้อม Quick Reply)**

```
สวัสดีครับ! 👋 ยินดีต้อนรับสู่ [ชื่อร้าน]

คุณสนใจเรื่องอะไรเป็นพิเศษครับ? 😊
(กดปุ่มด้านล่างได้เลย)
```

*พร้อม Quick Reply Buttons:*
- "ดูสินค้า"
- "โปรโมชัน"
- "วิธีสั่งซื้อ"
- "ติดต่อเรา"

### 7.4.4 Greeting Message พร้อม Quick Reply ผ่าน API

```javascript
// ส่ง Greeting Message พร้อม Quick Reply เมื่อมี Follow Event
async function handleFollowEvent(event) {
  const userId = event.source.userId;
  const replyToken = event.replyToken;
  
  // ดึงข้อมูลโปรไฟล์ผู้ใช้
  let userName = 'คุณ';
  try {
    const profile = await client.getProfile(userId);
    userName = profile.displayName;
  } catch (error) {
    console.error('ไม่สามารถดึงโปรไฟล์ได้:', error);
  }
  
  // ส่ง Greeting พร้อม Quick Reply
  await client.replyMessage(replyToken, {
    type: 'text',
    text: `สวัสดีครับคุณ${userName}! 🎉\n\nยินดีต้อนรับสู่ [ชื่อร้าน] นะครับ\n\nต้องการอะไรครับ?`,
    quickReply: {
      items: [
        {
          type: 'action',
          action: {
            type: 'message',
            label: '🛍️ ดูสินค้า',
            text: 'เมนู'
          }
        },
        {
          type: 'action',
          action: {
            type: 'message',
            label: '🏷️ โปรโมชัน',
            text: 'โปรโมชัน'
          }
        },
        {
          type: 'action',
          action: {
            type: 'uri',
            label: '🌐 เว็บไซต์',
            uri: 'https://example.com'
          }
        },
        {
          type: 'action',
          action: {
            type: 'message',
            label: '📞 ติดต่อ',
            text: 'ติดต่อ'
          }
        }
      ]
    }
  });
  
  // บันทึกผู้ใช้ใหม่ลงฐานข้อมูล
  await saveNewUser(userId, userName);
}
```

---

## 7.5 Away Messages (ข้อความขณะไม่อยู่)

### 7.5.1 ประเภท Away Scenarios

**1. นอกเวลาทำการ (After Hours)**
- เวลา: หลัง 18:00 น. หรือช่วงเช้าก่อนเปิด
- ควรบอก: เวลาทำการและเวลาที่จะตอบกลับ

**2. วันหยุด (Holidays)**
- วันหยุดนักขัตฤกษ์
- วันหยุดสงกรานต์ วันหยุดพิเศษ
- ควรบอก: วันกลับมาทำการ

**3. กำลังยุ่ง (Busy)**
- ช่วงที่มีงานมาก
- ควรบอก: จะตอบภายในกี่ชั่วโมง

### 7.5.2 ตัวอย่าง Away Messages แบบละเอียด

**Away Message สำหรับวันหยุดสงกรานต์:**

```
สวัสดีปีใหม่ไทย! 🎊💦

ทีมงาน [ชื่อร้าน] หยุดสงกรานต์
13-15 เมษายน 2567

กลับมาให้บริการวันที่ 16 เมษายน 2567

ในระหว่างนี้:
• ดูสินค้าได้ที่: https://example.com
• คำถามทั่วไป: https://example.com/faq

ขอบคุณที่รออย่างอดทนนะครับ 🙏
```

**Away Message สำหรับเหตุฉุกเฉิน:**

```
ขออภัยในความไม่สะดวกนะครับ 🙏

ขณะนี้ระบบกำลังอยู่ระหว่างการปรับปรุง
[วัน/เวลา - วัน/เวลา]

สำหรับเรื่องเร่งด่วน:
📞 โทร: 02-XXX-XXXX
📧 Email: support@example.com

จะกลับมาให้บริการโดยเร็วที่สุดครับ
```

### 7.5.3 Smart Away Message System

```javascript
// ระบบ Away Message อัจฉริยะที่ปรับตามสถานการณ์
class SmartAwayMessageSystem {
  constructor(lineClient) {
    this.client = lineClient;
    this.businessConfig = {
      weekdayHours: { start: 9, end: 18 }, // 9:00-18:00
      weekendHours: { start: 10, end: 16 }, // 10:00-16:00
      workDays: [1, 2, 3, 4, 5, 6], // จ-ส (ไม่รวมอาทิตย์)
      holidays: [
        '2024-01-01',
        '2024-04-13',
        '2024-04-14', 
        '2024-04-15',
      ]
    };
  }

  getBangkokTime() {
    const now = new Date();
    return new Date(now.toLocaleString('en-US', { timeZone: 'Asia/Bangkok' }));
  }

  getStatus() {
    const time = this.getBangkokTime();
    const day = time.getDay();
    const hour = time.getHours();
    const dateStr = time.toLocaleDateString('en-CA', { timeZone: 'Asia/Bangkok' });

    if (this.businessConfig.holidays.includes(dateStr)) {
      return 'holiday';
    }

    if (!this.businessConfig.workDays.includes(day)) {
      return 'weekend';
    }

    const hours = day === 6 ? this.businessConfig.weekendHours : this.businessConfig.weekdayHours;
    if (hour >= hours.start && hour < hours.end) {
      return 'open';
    }

    return 'closed';
  }

  getNextOpenTime() {
    const time = this.getBangkokTime();
    const day = time.getDay();
    const hour = time.getHours();

    if (hour < 9 && this.businessConfig.workDays.includes(day)) {
      return 'เวลา 09:00 น. วันนี้';
    }

    return 'เวลา 09:00 น. วันทำการถัดไป';
  }

  getAwayMessage(status) {
    const messages = {
      closed: `ขอบคุณที่ติดต่อ [ชื่อร้าน] นะครับ 🙏

⏰ ขณะนี้อยู่นอกเวลาทำการ
จะตอบกลับ${this.getNextOpenTime()}

เวลาทำการ: จ-ศ 9:00-18:00 | ส 10:00-16:00
🌐 FAQ: https://example.com/faq`,

      weekend: `ขอบคุณที่ติดต่อนะครับ! 😊

📅 วันนี้เป็นวันอาทิตย์ ทีมงานหยุดพัก
จะตอบกลับ${this.getNextOpenTime()}

ช้อปสินค้าออนไลน์ได้ตลอด 24 ชั่วโมง:
🛒 https://example.com`,

      holiday: `สวัสดีครับ!

🎊 วันนี้เป็นวันหยุดนักขัตฤกษ์
ทีมงานจะกลับมา${this.getNextOpenTime()}

ขอให้มีความสุขในวันหยุดนะครับ 🙏`
    };

    return messages[status] || messages.closed;
  }

  async handleMessage(event) {
    const status = this.getStatus();
    
    if (status === 'open') {
      return null; // ให้ระบบหลักจัดการ
    }

    await this.client.replyMessage(event.replyToken, {
      type: 'text',
      text: this.getAwayMessage(status)
    });
    
    return true; // บอกว่าจัดการแล้ว
  }
}
```

---

## 7.6 Pattern และตัวอย่างที่ใช้งานได้จริง

### 7.6.1 Pattern: FAQ Bot อัตโนมัติ

```javascript
// faq-bot.js
const FAQ_DATA = {
  'ราคา|price|เท่าไหร่|cost': {
    type: 'flex',
    altText: 'ข้อมูลราคา',
    contents: {
      type: 'bubble',
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: '💰 ราคาสินค้า',
            weight: 'bold',
            size: 'xl'
          },
          {
            type: 'text',
            text: 'สินค้า A: 299 บาท\nสินค้า B: 599 บาท\nสินค้า C: 899 บาท',
            wrap: true,
            margin: 'md'
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
            label: 'ดูราคาทั้งหมด',
            uri: 'https://example.com/price'
          },
          style: 'primary'
        }]
      }
    }
  },
  
  'สั่ง|order|ซื้อ|buy': {
    type: 'text',
    text: '🛒 วิธีสั่งซื้อ:\n\n1️⃣ เลือกสินค้าในเว็บ\n2️⃣ ใส่ตะกร้า\n3️⃣ กรอกที่อยู่\n4️⃣ เลือกการชำระเงิน\n5️⃣ รอรับสินค้า 1-3 วัน\n\n🌐 https://example.com/order'
  }
};

function findFAQMatch(userMessage) {
  const lowerMessage = userMessage.toLowerCase();
  
  for (const [pattern, response] of Object.entries(FAQ_DATA)) {
    const keywords = pattern.split('|');
    if (keywords.some(kw => lowerMessage.includes(kw))) {
      return response;
    }
  }
  
  return null;
}
```

### 7.6.2 Pattern: Multi-step Conversation

```javascript
// multi-step-conversation.js
// สำหรับการสนทนาหลายขั้นตอน เช่น การสั่งซื้อ

const userSessions = new Map(); // เก็บ state ของแต่ละ user

async function handleOrderFlow(event) {
  const userId = event.source.userId;
  const userMessage = event.message.text;
  const replyToken = event.replyToken;
  
  let session = userSessions.get(userId) || { step: 'idle' };
  
  switch (session.step) {
    case 'idle':
      if (userMessage.includes('สั่ง') || userMessage.includes('order')) {
        userSessions.set(userId, { step: 'ask_product' });
        await client.replyMessage(replyToken, {
          type: 'text',
          text: 'ยินดีช่วยด้านการสั่งซื้อครับ! 😊\n\nต้องการสั่งสินค้าอะไรครับ?\nหรือพิมพ์ "รายการ" เพื่อดูสินค้าทั้งหมด'
        });
      }
      break;
      
    case 'ask_product':
      userSessions.set(userId, { 
        step: 'ask_quantity',
        product: userMessage 
      });
      await client.replyMessage(replyToken, {
        type: 'text',
        text: `รับทราบครับ "${userMessage}"\n\nต้องการจำนวนเท่าไหร่ครับ?`,
        quickReply: {
          items: [1, 2, 3, 5].map(n => ({
            type: 'action',
            action: {
              type: 'message',
              label: `${n} ชิ้น`,
              text: `${n}`
            }
          }))
        }
      });
      break;
      
    case 'ask_quantity':
      const quantity = parseInt(userMessage);
      if (!isNaN(quantity)) {
        userSessions.set(userId, {
          ...session,
          step: 'confirm',
          quantity: quantity
        });
        
        const updatedSession = userSessions.get(userId);
        await client.replyMessage(replyToken, {
          type: 'text',
          text: `สรุปการสั่งซื้อ:\n📦 สินค้า: ${updatedSession.product}\n🔢 จำนวน: ${quantity} ชิ้น\n\nยืนยันการสั่งซื้อใช่ไหมครับ?`,
          quickReply: {
            items: [
              {
                type: 'action',
                action: { type: 'message', label: '✅ ยืนยัน', text: 'ยืนยัน' }
              },
              {
                type: 'action',
                action: { type: 'message', label: '❌ ยกเลิก', text: 'ยกเลิก' }
              }
            ]
          }
        });
      }
      break;
      
    case 'confirm':
      if (userMessage === 'ยืนยัน') {
        // บันทึกออเดอร์
        const orderId = `ORD-${Date.now()}`;
        userSessions.delete(userId);
        
        await client.replyMessage(replyToken, {
          type: 'text',
          text: `✅ สั่งซื้อสำเร็จแล้วครับ!\n\n🔖 หมายเลขออเดอร์: ${orderId}\nทีมงานจะติดต่อยืนยันและแจ้งรายละเอียดการชำระเงินนะครับ`
        });
      } else {
        userSessions.delete(userId);
        await client.replyMessage(replyToken, {
          type: 'text',
          text: 'ยกเลิกการสั่งซื้อแล้วครับ\nต้องการความช่วยเหลืออะไรเพิ่มเติมไหมครับ?'
        });
      }
      break;
  }
}
```

### 7.6.3 Pattern: Rich Menu + Auto-Reply Integration

```
การทำงานร่วมกัน:

Rich Menu (หน้าจอหลัก)
├── ปุ่ม "สินค้า" → ส่ง Text "เมนู" → Auto-Reply แสดงรายการ
├── ปุ่ม "โปรโมชัน" → URL ไปหน้าโปรโมชัน
├── ปุ่ม "ติดตามออเดอร์" → ส่ง Text "ติดตาม" → Auto-Reply ขอเลขออเดอร์
├── ปุ่ม "ติดต่อ" → ส่ง Text "ติดต่อ" → Auto-Reply แสดงข้อมูลติดต่อ
└── ปุ่ม "FAQ" → URL ไปหน้า FAQ
```

### 7.6.4 Anti-spam & Rate Limiting

```javascript
// ป้องกัน spam และ Auto-Reply Loop
const userMessageCount = new Map();
const RATE_LIMIT = 5; // ข้อความต่อนาที
const RATE_WINDOW = 60 * 1000; // 1 นาที

function checkRateLimit(userId) {
  const now = Date.now();
  const userHistory = userMessageCount.get(userId) || [];
  
  // ลบ entries เก่ากว่า 1 นาที
  const recentMessages = userHistory.filter(time => now - time < RATE_WINDOW);
  
  if (recentMessages.length >= RATE_LIMIT) {
    return false; // Rate limit exceeded
  }
  
  recentMessages.push(now);
  userMessageCount.set(userId, recentMessages);
  return true;
}

async function handleMessageWithRateLimit(event) {
  const userId = event.source.userId;
  
  if (!checkRateLimit(userId)) {
    // ไม่ตอบกลับเพื่อป้องกัน loop
    console.log(`Rate limit exceeded for user: ${userId}`);
    return;
  }
  
  // ดำเนินการตอบกลับปกติ
  await handleMessage(event);
}
```

---

## 7.7 Advanced Auto-Reply: NLP Integration

### ตัวอย่างการเชื่อมต่อกับ NLP (Natural Language Processing)

```javascript
// nlp-auto-reply.js
// ใช้ร่วมกับ Dialogflow หรือ OpenAI

const { Configuration, OpenAIApi } = require('openai');

const configuration = new Configuration({
  apiKey: process.env.OPENAI_API_KEY,
});
const openai = new OpenAIApi(configuration);

// System prompt สำหรับ AI
const SYSTEM_PROMPT = `คุณเป็นผู้ช่วยของร้านค้า [ชื่อร้าน]
ข้อมูลร้าน:
- ประเภท: ร้านขายเสื้อผ้าแฟชัน
- เวลาทำการ: จ-ศ 9:00-18:00 น.
- ราคา: 299-2999 บาท
- จัดส่ง: Kerry, J&T, Flash
- ชำระเงิน: โอนเงิน, COD, บัตรเครดิต

กรุณา:
1. ตอบเป็นภาษาไทยที่เป็นกันเอง
2. ตอบสั้น กระชับ ไม่เกิน 3 ย่อหน้า
3. แนะนำให้โทรถามเมื่อเรื่องซับซ้อน`;

async function getAIReply(userMessage, userId) {
  try {
    const response = await openai.createChatCompletion({
      model: 'gpt-3.5-turbo',
      messages: [
        { role: 'system', content: SYSTEM_PROMPT },
        { role: 'user', content: userMessage }
      ],
      max_tokens: 500,
      temperature: 0.7
    });
    
    return response.data.choices[0].message.content;
  } catch (error) {
    console.error('AI error:', error);
    return 'ขออภัยนะครับ ขณะนี้ระบบมีปัญหา กรุณาโทร 02-XXX-XXXX';
  }
}
```

---

## 7.8 การทดสอบ Auto-Reply

### ขั้นตอนการทดสอบ

**1. Test via LINE OA Manager:**
- ใช้ "Test" mode ในหน้า Auto-reply
- ส่งข้อความทดสอบและดูผลลัพธ์

**2. Test via Postman (สำหรับ API):**

```json
POST https://api.line.me/v2/bot/message/reply
Content-Type: application/json
Authorization: Bearer {ACCESS_TOKEN}

{
  "replyToken": "test_reply_token",
  "messages": [
    {
      "type": "text",
      "text": "ทดสอบข้อความ"
    }
  ]
}
```

**3. Automated Testing:**

```javascript
// auto-reply-test.js
const testCases = [
  { input: 'ราคา', expectedKeyword: 'ราคา', description: 'ทดสอบ keyword ราคา' },
  { input: 'ขอดูราคาหน่อยครับ', expectedKeyword: 'ราคา', description: 'ทดสอบ partial match' },
  { input: 'PRICE', expectedKeyword: 'ราคา', description: 'ทดสอบ case insensitive' },
  { input: 'ข้อความสุ่ม xyz', expectedKeyword: null, description: 'ทดสอบ no match (default)' }
];

function testKeywordMatching(userMessage, keywordReplies) {
  const lowerMessage = userMessage.toLowerCase();
  
  for (const [pattern, response] of Object.entries(keywordReplies)) {
    const keywords = pattern.split('|');
    if (keywords.some(kw => lowerMessage.includes(kw.toLowerCase()))) {
      return { matched: true, pattern, response };
    }
  }
  
  return { matched: false };
}

// รัน test cases
testCases.forEach(testCase => {
  const result = testKeywordMatching(testCase.input, KEYWORD_REPLIES);
  const passed = testCase.expectedKeyword 
    ? result.matched && result.pattern.includes(testCase.expectedKeyword)
    : !result.matched;
  
  console.log(`${passed ? '✅' : '❌'} ${testCase.description}`);
  if (!passed) {
    console.log(`   Input: "${testCase.input}"`);
    console.log(`   Expected: ${testCase.expectedKeyword || 'no match'}`);
    console.log(`   Got: ${result.matched ? result.pattern : 'no match'}`);
  }
});
```

---

## 7.9 แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Keyword Auto-Reply พื้นฐาน
**เวลา: 45 นาที**

ตั้งค่า Keyword Auto-Reply อย่างน้อย 5 กลุ่ม สำหรับธุรกิจสมมติ:
1. "สวัสดี/hello/hi" → ข้อความต้อนรับ
2. "ราคา/price" → ข้อมูลราคา
3. "เวลา/เปิด/ปิด" → เวลาทำการ
4. "ที่อยู่/location/map" → ที่อยู่พร้อมลิงก์ Google Maps
5. "ติดต่อ/contact" → ข้อมูลการติดต่อ

### แบบฝึกหัดที่ 2: Greeting Message ที่สมบูรณ์
**เวลา: 30 นาที**

สร้าง Greeting Message สำหรับบัญชีสมมติ ที่มี:
- ข้อความต้อนรับส่วนตัว
- แนะนำบริการหลัก 3 อย่าง
- Incentive สำหรับผู้ใหม่
- วิธีใช้งาน Quick Commands

### แบบฝึกหัดที่ 3: Auto-Reply System ด้วย Webhook
**เวลา: 120 นาที**

สร้าง Express.js Webhook Server ที่:
1. ตอบกลับตาม Keyword ด้วย JSON data จาก `faq.json`
2. มี Time-based checking (ตอบต่างกันในและนอกเวลาทำการ)
3. Track จำนวนข้อความแต่ละ keyword ที่ถูกถาม
4. Export report เป็น JSON เมื่อเรียก `/api/keyword-stats`

**โครงสร้างไฟล์:**
```
project/
├── server.js          # Main server
├── handlers/
│   ├── message.js     # Message handler
│   ├── follow.js      # Follow event handler
│   └── faq.js         # FAQ logic
├── data/
│   └── faq.json       # FAQ data
└── utils/
    ├── time.js        # Time utilities
    └── rate-limiter.js
```

---

## สรุปบทที่ 7

ในบทนี้คุณได้เรียนรู้:

- ✅ ประเภทของ Auto-Reply ทั้งหมดใน LINE OA
- ✅ การตั้งค่า Keyword Auto-Reply ทั้งจาก Manager และ API
- ✅ การสร้าง Time-based Auto-Reply และ Away Message
- ✅ การออกแบบ Greeting Message ที่มีประสิทธิภาพ
- ✅ Pattern สำคัญ เช่น Multi-step Conversation
- ✅ การป้องกัน Spam และ Rate Limiting
- ✅ การทดสอบ Auto-Reply ให้ถูกต้อง

**บทถัดไป**: Part 8 จะครอบคลุม Rich Menu ซึ่งเป็นเครื่องมือ UI ที่ทรงพลังที่สุดของ LINE OA

---

*อัปเดตล่าสุด: ตุลาคม 2024 | LINE Official Account Manager v2024*
