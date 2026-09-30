# ส่วนที่ 3: LINE Official Account Manager Dashboard Overview

---

## สารบัญ

1. [ภาพรวม LINE OA Manager](#1-ภาพรวม-line-oa-manager)
2. [Navigation Menu หลัก](#2-navigation-menu-หลัก)
3. [Home Dashboard](#3-home-dashboard)
4. [Messaging Menu](#4-messaging-menu)
5. [Chat Menu](#5-chat-menu)
6. [Timeline (LINE VOOM)](#6-timeline-line-voom)
7. [Analysis (Analytics)](#7-analysis-analytics)
8. [Settings Menu](#8-settings-menu)
9. [การจัดการ Team Members](#9-การจัดการ-team-members)
10. [Mobile App vs Desktop](#10-mobile-app-vs-desktop)
11. [แบบฝึกหัดท้ายบท](#11-แบบฝึกหัดท้ายบท)

---

## 1. ภาพรวม LINE OA Manager

### 1.1 LINE OA Manager คืออะไร?

LINE OA Manager (manager.line.biz) คือ Control Panel หลักของ LINE Official Account ที่ให้คุณจัดการทุกอย่างของบัญชีผ่านหน้าเดียว ทั้ง Desktop Browser และ Mobile App

```
LINE OA Manager - ภาพรวมระบบ
═══════════════════════════════════════════════════════════

┌─── Navigation ───┬─────────────────────────────────────┐
│                  │                                     │
│  🏠 Home         │           Main Content Area         │
│  💬 Chat         │                                     │
│  📢 Messaging    │   ← เนื้อหาหลักแสดงที่นี่            │
│  📝 Timeline     │                                     │
│  📊 Analysis     │                                     │
│  ⚙️ Settings     │                                     │
│                  │                                     │
│  [Account Info]  │                                     │
│  [@accountid]    │                                     │
└──────────────────┴─────────────────────────────────────┘

URL: https://manager.line.biz/account/{accountId}/

═══════════════════════════════════════════════════════════
```

### 1.2 วิธีเข้าถึง LINE OA Manager

**Browser (Desktop/Mobile):**
```
URL: https://manager.line.biz
Login: LINE Account หรือ Email/Password
```

**LINE OA App (iOS/Android):**
```
ดาวน์โหลด: "LINE Official Account" จาก App Store/Google Play
Login: LINE Account
```

**จาก LINE App (Mobile):**
```
LINE App → โปรไฟล์ → Official Account Manager
```

### 1.3 ภาษาของ Interface

```
เปลี่ยนภาษา:
Settings → General → Language

ภาษาที่รองรับ:
- ไทย (Thai)
- English
- 日本語 (Japanese)
- 한국어 (Korean)
- 繁體中文 (Traditional Chinese)
```

---

## 2. Navigation Menu หลัก

### 2.1 โครงสร้าง Navigation

```
LINE OA Manager Navigation Structure:
════════════════════════════════════════════════════════

🏠 HOME
   └── Dashboard หน้าหลัก, สถิติสรุป, Quick Actions

💬 CHAT
   ├── All Chats
   ├── Assigned to me
   ├── Unread
   └── Resolved

📢 MESSAGING
   ├── Create New Message
   ├── Broadcast
   ├── Multicast
   ├── Narrowcast
   ├── Auto Reply
   ├── Greeting Message
   ├── AI Reply
   └── Message Templates

📝 TIMELINE (LINE VOOM)
   ├── Create Post
   ├── My Posts
   └── VOOM Statistics

📊 ANALYSIS
   ├── Overview
   ├── Followers
   ├── Message Delivery
   ├── Chat
   └── VOOM Statistics

🛒 MY SHOP (ถ้าเปิดใช้)
   ├── Products
   ├── Orders
   └── Promotions

⚙️ SETTINGS
   ├── Account
   ├── Messaging
   ├── Chat
   ├── Rich Menu
   ├── Loyalty Card (Point Card)
   ├── Coupon
   ├── LINE Pay
   └── Account Rights

════════════════════════════════════════════════════════
```

---

## 3. Home Dashboard

### 3.1 Overview ของหน้า Home

หน้า Home แสดงข้อมูลสรุปและสถิติที่สำคัญทั้งหมดในที่เดียว:

```
Home Dashboard Layout:
┌─────────────────────────────────────────────────┐
│  Account Name          [Switch Account ▼]       │
├─────────────────────────────────────────────────┤
│  📢 Quick Actions:                              │
│  [Create Message] [View Chat] [Post VOOM]       │
├─────────────────────────────────────────────────┤
│  📊 Statistics Overview (7 วันล่าสุด):           │
│  ┌──────────┬──────────┬──────────┬──────────┐  │
│  │ Followers│ New      │ Blocked  │ Messages │  │
│  │ 1,234    │ +45      │ -3       │ Sent:200 │  │
│  └──────────┴──────────┴──────────┴──────────┘  │
├─────────────────────────────────────────────────┤
│  🔔 Notifications & Alerts                      │
│  • Unread chat: 5 messages                      │
│  • Scheduled broadcast: 1 tomorrow 10:00        │
├─────────────────────────────────────────────────┤
│  📈 Recent Activity Graph                       │
│  [กราฟแสดง Follower Trend 30 วัน]               │
└─────────────────────────────────────────────────┘
```

### 3.2 Metrics บน Home Dashboard

**Follower Statistics:**

| Metric | ความหมาย | วิธีปรับปรุง |
|--------|---------|------------|
| Total Followers | จำนวน Friends ทั้งหมด | เพิ่ม QR Code บนสื่อต่างๆ |
| New Followers | เพื่อนใหม่ในช่วงเวลาที่เลือก | รัน Campaign ดึงดูด |
| Blocked | คนที่ Block บัญชี | ปรับปรุงคุณภาพเนื้อหา |
| Unblocked | คนที่ Unblock กลับมา | - |

**Message Statistics:**

| Metric | ความหมาย |
|--------|---------|
| Sent | ข้อความที่ส่งออกไป |
| Delivered | ข้อความที่ถูกส่งถึง |
| Opened | ข้อความที่ถูกเปิดอ่าน |
| Clicked | Link ที่ถูกคลิก |

### 3.3 Quick Actions บน Dashboard

```
Quick Actions ที่เข้าถึงได้ทันที:

[📢 Create Message]  → ไปหน้าสร้างข้อความใหม่
[💬 View Chats]      → ไปหน้า Chat
[📝 Post to VOOM]    → สร้างโพสต์ใหม่
[🎫 Create Coupon]   → สร้าง Coupon ใหม่
[📊 View Analysis]   → ไปหน้าวิเคราะห์
```

---

## 4. Messaging Menu

### 4.1 Create New Message

**ประเภทของข้อความที่สร้างได้:**

```
สร้างข้อความใหม่:
├── Text Message (ข้อความธรรมดา)
│   └── สูงสุด 5,000 ตัวอักษร
│
├── Image (รูปภาพ)
│   └── JPG/PNG สูงสุด 10 MB
│
├── Video (วิดีโอ)
│   └── MP4 สูงสุด 200 MB, 1 นาที
│
├── Sticker (สติ๊กเกอร์)
│   └── เลือกจาก LINE Sticker Store
│
├── Coupon (คูปอง)
│   └── ต้องสร้าง Coupon ก่อน
│
├── Rich Message (ข้อความ Rich)
│   └── รูป + ข้อความ + ลิงก์ + ปุ่ม
│
├── Image Map (แผนที่รูปภาพ)
│   └── รูป 1 ภาพ ลิงก์ได้หลายจุด
│
├── Card Type Message
│   └── Product Card, Person Card, Location Card
│
└── Flex Message (ขั้นสูง)
    └── Design เองผ่าน JSON
```

### 4.2 Broadcast

Broadcast คือการส่งข้อความหาผู้ติดตามทุกคน (หรือ Segment ที่เลือก)

```
ขั้นตอนสร้าง Broadcast:

1. Messaging → Create New Message
2. เลือกประเภทข้อความ
3. สร้างเนื้อหา
4. เลือกผู้รับ:
   a. All followers (ทุกคน)
   b. Attributes (กรอง Gender, Age, OS, Region)
5. เลือกเวลาส่ง:
   a. Send now (ส่งทันที)
   b. Schedule (กำหนดเวลา)
6. ตรวจสอบ Preview
7. คลิก Send/Schedule
```

**ข้อควรระวังใน Broadcast:**

> **⚠️ การส่ง Broadcast จะนับ Message Quota!**
> - Free Plan: 500 ข้อความ/เดือน
> - Light Plan: 5,000 ข้อความ/เดือน
> - 1 ข้อความ Broadcast = จำนวน Recipients × 1

### 4.3 Narrowcast (Targeted Messaging)

Narrowcast ช่วยให้ส่งข้อความไปหาเฉพาะกลุ่มที่ต้องการ:

```
Narrowcast Filter Options:
├── Demographics
│   ├── Age: 15-24, 25-34, 35-44, 45-54, 55+
│   ├── Gender: Male, Female
│   └── Region: กรุงเทพ, ภาคเหนือ, ภาคใต้, ฯลฯ
│
├── Operating System
│   ├── iOS
│   └── Android
│
└── Audience (Segment ที่สร้างเอง)
    ├── Chat Audience (คนที่เคย Chat)
    ├── Clicked Audience (คนที่เคยคลิก)
    └── Custom Audience (Upload User List)
```

**ตัวอย่างการใช้ Narrowcast:**

```python
# ตัวอย่าง: ส่ง Promotion เฉพาะผู้หญิงอายุ 25-35 ในกรุงเทพ
# ทำผ่าน Narrowcast UI หรือ API

{
  "recipient": {
    "type": "filter",
    "demographic": {
      "age": {"gte": 25, "lte": 35},
      "gender": "female",
      "region": ["TH-10"]  // กรุงเทพมหานคร
    }
  },
  "messages": [
    {
      "type": "text",
      "text": "สาวๆ กรุงเทพ! โปรโมชั่นพิเศษเฉพาะคุณ 🎁"
    }
  ]
}
```

### 4.4 Auto Reply

Auto Reply คือการตอบกลับอัตโนมัติเมื่อลูกค้าส่งข้อความมา:

```
ประเภท Auto Reply:
├── Keyword Reply (ตาม Keyword)
│   └── ลูกค้าส่ง "ราคา" → ตอบ "ดูราคาได้ที่..."
│
├── Default Reply (ตอบเมื่อไม่ตรง Keyword)
│   └── "ขอบคุณที่ติดต่อ เราจะตอบภายใน 1 ชั่วโมง"
│
└── Business Hours Reply
    └── นอกเวลาทำการ → "ปิดทำการ จะตอบวันพรุ่งนี้"
```

**การตั้งค่า Auto Reply:**

```
Settings → Messaging → Auto Reply
→ Toggle ON
→ + Add keyword
→ กรอก Keyword (หลาย Keyword ใช้ "," คั่น)
→ สร้างข้อความตอบกลับ
→ Save
```

### 4.5 AI Reply (Intelligent Auto Reply)

```
AI Reply ทำงานอย่างไร:

ลูกค้า: "สินค้ายังมีไหม?"
   │
AI วิเคราะห์ความหมาย
   │
ตอบกลับโดยอัตโนมัติ: "สินค้ายังมีในสต็อกครับ กดที่เมนู 'ดูสินค้า' เพื่อดูรายละเอียดเพิ่มเติม"

สิ่งที่ AI Reply ทำได้:
✓ ตอบคำถามทั่วไป
✓ แนะนำ URL/Link
✓ ส่ง Rich Message
✓ Escalate ไป Human Agent เมื่อจำเป็น

สิ่งที่ AI Reply ทำไม่ได้:
✗ ประมวลผลข้อมูล Real-time (เช่น ตรวจสต็อก)
✗ ทำธุรกรรม
✗ เข้าถึงข้อมูลส่วนตัวของลูกค้า
```

---

## 5. Chat Menu

### 5.1 ภาพรวม Chat Interface

```
Chat Menu Layout:
┌──────────────────┬────────────────────────────────┐
│  Chat List       │  Chat Window                   │
│                  │                                │
│  [All Chats ▼]   │  👤 ชื่อลูกค้า / User ID       │
│                  │  ─────────────────────────────│
│  🟢 สมชาย        │  ลูกค้า: สวัสดี อยากถามเรื่อง  │
│     "สวัสดี..."  │         การส่งของครับ           │
│     10:30 AM     │                               │
│                  │  Admin: สวัสดีครับ ยินดีช่วย   │
│  🟡 มาลี         │        ครับ ต้องการทราบ         │
│     "ราคาเท่า..."│        Track Number ไหมครับ?   │
│     Yesterday    │                               │
│                  │  ─────────────────────────────│
│  🔴 นิติ         │  [พิมพ์ข้อความ...] [📎] [Send] │
│     "ยัง..."     │                                │
│     2 days ago   └────────────────────────────────┘
└──────────────────┘
```

### 5.2 Chat Tabs และ Filter

```
Chat Filters:
├── All (ทั้งหมด)
├── Unread (ยังไม่อ่าน)
├── Assigned to me (ที่มอบหมายให้ฉัน)
├── Unassigned (ยังไม่ได้มอบหมาย)
├── Resolved (แก้ไขแล้ว)
└── Spam (สแปม)
```

### 5.3 Chat Features

**ฟีเจอร์ใน Chat:**

| ฟีเจอร์ | คำอธิบาย |
|--------|---------|
| Tags | ติด Label บน Chat เช่น "VIP", "Complaint" |
| Notes | จดบันทึก Internal Note (ลูกค้าไม่เห็น) |
| Assign | มอบหมาย Chat ให้ Admin อื่น |
| Resolve | ปิด Chat ที่แก้ไขเรียบร้อยแล้ว |
| Follow-up | ตั้ง Reminder ติดตาม Chat |
| Templates | ใช้ Message Template ส่งได้เร็ว |
| Customer Info | ดูข้อมูลลูกค้าจาก CRM Integration |

### 5.4 Chat Tags

```
ตัวอย่าง Chat Tags ที่มีประโยชน์:

🔴 Urgent          - เร่งด่วน ต้องตอบทันที
⭐ VIP             - ลูกค้า VIP
💰 High Value      - ออเดอร์ใหญ่
🚫 Complaint       - ลูกค้าร้องเรียน
🤝 Follow-up       - ต้องติดตาม
✅ Resolved        - แก้ไขแล้ว
🔄 Pending Payment - รอชำระเงิน
📦 Shipping Issue  - ปัญหาการจัดส่ง
```

### 5.5 การตั้งค่า Chat Mode

```
Chat Mode Options:
├── AI Chat Mode
│   └── AI ตอบอัตโนมัติ พร้อม Escalate
│
├── Manual Chat Mode  
│   └── Admin ตอบเองทั้งหมด
│
└── Smart Chat Mode (ผสม)
    └── AI ตอบก่อน ถ้าไม่สามารถตอบ → ส่งต่อ Admin
```

---

## 6. Timeline (LINE VOOM)

### 6.1 LINE VOOM คืออะไร?

LINE VOOM เป็น Social Feed บน LINE คล้ายกับ Facebook Feed ที่ผู้ติดตาม LINE OA จะเห็นโพสต์ของคุณ

```
ความแตกต่าง VOOM vs Broadcast:
─────────────────────────────────────────
VOOM Post:
✓ ฟรี ไม่นับ Message Quota
✓ ผู้ติดตามเห็นใน Feed
✓ รองรับ Video ยาวได้
✓ มีระบบ Like, Comment, Share
✗ Algorithm อาจไม่แสดงทุกคน
✗ Open Rate ต่ำกว่า Broadcast

Broadcast Message:
✓ ส่งตรงถึง Chat ของผู้ติดตาม
✓ Push Notification ทำให้เปิดอ่านมากกว่า
✗ นับ Message Quota (มีค่าใช้จ่าย)
✗ ไม่มี Like/Comment
─────────────────────────────────────────
```

### 6.2 ประเภทโพสต์ใน VOOM

```
ประเภทโพสต์:
├── Text Post
│   └── ข้อความเดี่ยว สูงสุด 10,000 ตัวอักษร
│
├── Photo Post
│   └── รูปเดี่ยวหรือหลายรูป (สูงสุด 20 รูป)
│
├── Video Post
│   └── MP4, สูงสุด 1 GB, ไม่จำกัดความยาว
│
├── Photo+Video Mixed
│   └── ผสมรูปและวิดีโอ
│
└── Live Stream
    └── VOOM Live สำหรับ Verified Accounts
```

### 6.3 การตั้งค่าโพสต์

```
Post Settings:
├── Visibility: Public / Followers Only
├── Allow Comments: Yes/No
├── Allow Sharing: Yes/No
├── Pin to Top: Yes/No
└── Schedule: Post Now / Future Date
```

---

## 7. Analysis (Analytics)

### 7.1 ภาพรวม Analysis Dashboard

```
Analysis Menu:
├── Overview
│   └── Summary ของทุก Metric
│
├── Followers
│   ├── Total Followers Trend
│   ├── New Followers
│   ├── Blocked Followers
│   └── Demographics (Age, Gender, Region)
│
├── Message Delivery
│   ├── Broadcast Statistics
│   ├── Delivery Rate
│   ├── Open Rate
│   └── Click-through Rate
│
├── Chat
│   ├── Total Chats
│   ├── Average Response Time
│   └── Chat Resolution Rate
│
└── VOOM
    ├── Post Reach
    ├── Impressions
    ├── Likes / Comments / Shares
    └── Video Views
```

### 7.2 Followers Analytics

**กราฟ Follower Growth:**

```
Follower Growth Chart (ตัวอย่าง):
════════════════════════════════════════

1,500 ┤                          ████
1,400 ┤                     █████
1,300 ┤               ██████
1,200 ┤         ██████
1,100 ┤    █████
1,000 ┤████
      └────────────────────────────────
      Jan  Feb  Mar  Apr  May  Jun

สิ่งที่ต้องสังเกต:
→ วันไหน Followers เพิ่มมาก? (มี Campaign?)
→ วันไหน Block เยอะ? (ส่ง Content ไม่เหมาะ?)
→ Trend โดยรวม ขึ้น/ลง/คงที่?
════════════════════════════════════════
```

**Demographics ของ Followers:**

| Metric | ข้อมูลที่ได้ | ประโยชน์ |
|--------|-----------|--------|
| Gender | ชาย/หญิง/ไม่ระบุ | ปรับ Content ให้เหมาะ |
| Age Range | 15-24, 25-34, etc. | เลือก Tone of Voice |
| OS | iOS/Android | ออกแบบ UX ให้เหมาะ |
| Region | จังหวัด/ภูมิภาค | โปรโมชั่นตามพื้นที่ |

### 7.3 Message Delivery Analytics

```
Message Performance Report:
─────────────────────────────────────────────

Broadcast: "Flash Sale 11.11"
Sent: 2025-11-11 10:00 AM

Recipients:    1,234
Delivered:     1,198  (97.1%)
Opened:          852  (69.1%)   ← Open Rate
Clicked:         234  (18.9%)   ← CTR

Top Clicked:
1. [ดูสินค้า Sale]    156 clicks (66.7%)
2. [ดู Coupon]         52 clicks (22.2%)
3. [ติดต่อเรา]          26 clicks (11.1%)

─────────────────────────────────────────────
```

### 7.4 การ Export ข้อมูล Analytics

```
Export Options:
Analysis → [เลือก Metric] → Export

รูปแบบ:
└── CSV (Excel compatible)

ข้อมูลที่ Export ได้:
├── Follower data รายวัน
├── Message delivery รายข้อความ
├── Click data รายลิงก์
└── Chat statistics
```

> **💡 เคล็ดลับ:** Export ข้อมูลเป็น CSV แล้วนำเข้า Google Sheets หรือ Excel เพื่อสร้าง Dashboard ที่ต้องการได้

---

## 8. Settings Menu

### 8.1 Account Settings

```
Account Settings:
├── Account Info
│   ├── Display Name (เปลี่ยนชื่อ)
│   ├── Account Type
│   ├── Account Status
│   ├── LINE ID (Premium ID)
│   └── QR Code Download
│
├── Profile
│   ├── Profile Image
│   ├── Cover Photo
│   ├── Bio/Description
│   └── Phone/Website/Location
│
└── Verification
    ├── Verification Status
    └── Apply for Verification
```

### 8.2 Messaging Settings

```
Messaging Settings:
├── Chat
│   ├── Enable/Disable Chat
│   ├── Assign Mode (Manual/Auto)
│   └── Business Hours
│
├── Response
│   ├── Auto Reply Toggle
│   ├── Greeting Message
│   └── AI Reply Settings
│
└── Message Quota
    ├── Current Plan
    ├── Usage This Month
    └── Upgrade Plan
```

### 8.3 Rich Menu Settings

```
Rich Menu:
├── Create New Rich Menu
├── Manage Existing Menus
├── Set Default Menu
└── Schedule Menus

Rich Menu คือเมนูที่ปรากฏด้านล่าง Chat Window
เปรียบเสมือน "เมนูนำทาง" ของ LINE OA
```

### 8.4 Loyalty Card (Point Card) Settings

```
Point Card Settings:
├── Create Point Card
│   ├── Card Design
│   ├── Points per Visit
│   ├── Reward Details
│   └── Expiry Settings
│
└── Manage Existing Cards
    ├── View Issued Cards
    ├── Point Transactions
    └── Redemption History
```

### 8.5 Coupon Settings

```
Coupon Settings:
├── Create Coupon
│   ├── Coupon Type: Discount/Gift/Free
│   ├── Discount Amount/Percentage
│   ├── Validity Period
│   ├── Usage Limit
│   └── Redemption Conditions
│
└── Manage Coupons
    ├── Active Coupons
    ├── Expired Coupons
    └── Redemption Statistics
```

---

## 9. การจัดการ Team Members

### 9.1 Account Rights Overview

```
Team Management:
Settings → Account Rights

ข้อมูลที่เห็น:
├── Owner (เจ้าของบัญชี)
│   └── มีสิทธิ์ทุกอย่าง
│
├── Admins (ผู้ดูแล)
│   └── รายชื่อและสิทธิ์
│
└── Invited (รอยืนยัน)
    └── คนที่รับเชิญแต่ยังไม่ยืนยัน
```

### 9.2 ระดับสิทธิ์

| สิทธิ์ | จัดการบัญชี | ส่งข้อความ | ดู Chat | ดู Analytics | Billing |
|--------|-----------|----------|--------|------------|---------|
| Owner | ✓ | ✓ | ✓ | ✓ | ✓ |
| Admin | ✓ | ✓ | ✓ | ✓ | ✗ |
| Editor | ✗ | ✓ | ✓ | ✓ | ✗ |
| Chat Moderator | ✗ | ✗ | ✓ | ✗ | ✗ |
| Viewer | ✗ | ✗ | ✗ | ✓ | ✗ |

### 9.3 การเพิ่มและลบ Team Member

**เพิ่ม Member:**
```
1. Settings → Account Rights
2. คลิก "Add member"
3. กรอก LINE ID ของบุคคล
4. เลือกระดับสิทธิ์
5. คลิก "Invite"
6. บุคคลนั้นจะได้รับการแจ้งเตือนใน LINE
7. รอการยืนยัน
```

**ลบ Member:**
```
1. Settings → Account Rights
2. คลิก "..." หน้าชื่อบุคคล
3. เลือก "Remove"
4. ยืนยันการลบ
```

### 9.4 การมอบหมาย Chat

```
Assign Chat:
Chat Window → Assign To → เลือก Admin

Auto-Assign Settings:
Settings → Chat → Assign Mode
├── Manual: Admin เลือกเองว่าจะรับ Chat ไหน
├── Round-Robin: กระจาย Chat ให้ Admin แต่ละคนเท่าๆ กัน
└── By Keyword: ตาม Keyword ส่งให้ Admin ที่เชี่ยวชาญ
```

---

## 10. Mobile App vs Desktop

### 10.1 ความแตกต่างระหว่าง Mobile App และ Desktop

| ฟีเจอร์ | Mobile App | Desktop Browser |
|--------|-----------|----------------|
| ดู Dashboard | ✓ | ✓ |
| ส่ง Broadcast | ✓ | ✓ |
| จัดการ Chat | ✓ (ดีกว่า) | ✓ |
| สร้าง Rich Menu | จำกัด | ✓ (เต็มรูปแบบ) |
| Flex Message | ✗ | ✓ |
| Analytics | ✓ (พื้นฐาน) | ✓ (เต็ม) |
| แจ้งเตือน Push | ✓ | ✓ (Browser) |
| Offline Draft | ✓ | ✓ |

### 10.2 แนะนำการใช้งาน

```
แนะนำให้ใช้ Mobile App สำหรับ:
✓ ตอบ Chat ระหว่างวัน
✓ ดู Statistics รายวัน
✓ ส่ง Quick Message
✓ รับ Push Notification

แนะนำให้ใช้ Desktop Browser สำหรับ:
✓ สร้าง Rich Menu
✓ ออกแบบ Flex Message
✓ Export Analytics
✓ ตั้งค่าระบบ
✓ สร้าง Campaign ที่ซับซ้อน
```

---

## 11. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: สำรวจ Dashboard

**คำสั่ง:** เข้า LINE OA Manager ของคุณและตอบคำถามต่อไปนี้:

```
คำถาม Dashboard Exploration:

1. จำนวน Follower ปัจจุบัน: _______
2. Follower ใหม่ใน 7 วันที่ผ่านมา: _______
3. จำนวน Unread Chat: _______
4. Message Quota ที่ใช้ไปเดือนนี้: _______
5. Plan ปัจจุบัน: _______

Screenshot ส่วนสำคัญ:
[ ] Home Dashboard
[ ] Analytics Overview
[ ] Settings Page
```

### แบบฝึกหัดที่ 2: ตั้งค่า Auto Reply

**คำสั่ง:** สร้าง Auto Reply อย่างน้อย 5 รายการสำหรับธุรกิจของคุณ

```
Template Auto Reply:

| Keyword     | Reply Message              |
|-------------|----------------------------|
| สวัสดี      | สวัสดีครับ! มีอะไรให้ช่วย? |
| ราคา        | ดูราคาได้ที่: [URL]        |
| เปิด        | เปิด จ-ส 9:00-18:00 น.     |
| ที่อยู่     | [ที่อยู่ร้าน]              |
| ออเดอร์     | กรุณาส่ง Order ID มาครับ   |

เพิ่มเองอีก 2-3 รายการตามธุรกิจของคุณ
```

### แบบฝึกหัดที่ 3: วิเคราะห์ Analytics

**คำสั่ง:** ไปที่ Analysis Menu และตอบคำถาม:

```
Analytics Report:

ช่วงเวลา: 30 วันที่ผ่านมา

1. วันไหน Follower เพิ่มมากที่สุด? ______
   เหตุผลที่คาดว่าเป็น: ______

2. Follower Demographics:
   - เพศชาย: ___%
   - เพศหญิง: ___%
   - อายุมากที่สุด: ___
   - จังหวัดที่มาก: ___

3. Message ที่ส่งได้ผลดีที่สุด (Open Rate สูง): ______

4. สิ่งที่จะปรับปรุงจากข้อมูล: ______
```

### แบบฝึกหัดที่ 4: Team Member Setup

**คำสั่ง:** (สำหรับผู้ที่มีทีม) เพิ่ม Team Member อย่างน้อย 1 คน

```
ขั้นตอน:
1. ไปที่ Settings → Account Rights
2. เพิ่ม Admin หรือ Editor
3. กำหนดสิทธิ์ตามหน้าที่
4. ส่ง Invitation

บันทึก:
- ชื่อที่เพิ่ม: ______
- สิทธิ์ที่ให้: ______
- เหตุผล: ______
```

---

## สรุปส่วนที่ 3

ในส่วนนี้เราได้เรียนรู้:

✅ **ภาพรวม Dashboard** - โครงสร้างและการนำทางใน Manager

✅ **Messaging Menu** - Broadcast, Narrowcast, Auto Reply, AI Reply

✅ **Chat Management** - จัดการ Chat, Tags, การมอบหมาย

✅ **LINE VOOM** - โพสต์ Content ฟรีไม่นับ Quota

✅ **Analytics** - วิเคราะห์ Follower, Message, Chat Statistics

✅ **Settings** - การตั้งค่าทุกด้านของบัญชี

✅ **Team Management** - ระดับสิทธิ์และการจัดการทีม

---

## ส่วนถัดไป

ในส่วนที่ 4 เราจะเรียนรู้การตั้งค่าโปรไฟล์ให้สมบูรณ์:
- ขนาดและรูปแบบของ Profile Image
- Cover Photo ที่ดึงดูด
- การเขียน Bio และ Description
- Contact Information
- Status Message

---

*อัพเดทล่าสุด: กันยายน 2025*  
*หลักสูตร: LINE OA Complete Course*  
*ส่วนที่ 3 จาก 13*
