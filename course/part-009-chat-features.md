# ส่วนที่ 9: Chat Features

## บทนำ

Chat Features ใน LINE Official Account คือชุดเครื่องมือที่ช่วยให้ทีม Support และพนักงานสามารถจัดการการสนทนากับลูกค้าได้อย่างมีประสิทธิภาพ ตั้งแต่การ Tag ลูกค้า การเขียนบันทึก การมอบหมายงาน ไปจนถึงการบริหารทีม

ในบทนี้คุณจะเรียนรู้:
- ฟีเจอร์ 1-on-1 Chat ทั้งหมด
- Chat Tags สำหรับการจัดหมวดหมู่
- Chat Notes สำหรับบันทึกข้อมูล
- การมอบหมายและโอน Chat
- การดู Chat History
- การ Mark as Resolved
- การตั้งค่าทีมและ Operators

---

## 9.1 ฟีเจอร์ 1-on-1 Chat

### 9.1.1 ภาพรวมหน้าจอ Chat

เมื่อเข้าหน้า **"Chat"** ใน LINE OA Manager คุณจะเห็น:

```
┌─────────────────────────────────────────────────────┐
│  LINE OA Manager - Chat                              │
├────────────────┬────────────────────────────────────┤
│                │                                    │
│  Chat List     │    ส่วน Conversation               │
│  ────────────  │   ──────────────────────────────   │
│  🔴 New (5)    │   ชื่อ: สมชาย ใจดี                │
│  🟡 Active (3) │   สถานะ: กำลังรอ                  │
│  ✅ Done (12)  │                                    │
│                │   [ข้อความ 1]  10:30               │
│  ── Filter ──  │   [ข้อความ 2]  10:31               │
│  All Tags      │                                    │
│  My Chats      │   ──── Notes & Tags Panel ────     │
│  Unassigned    │   🏷️ Tags: VIP, ซื้อแล้ว           │
│                │   📝 Notes: ชอบสินค้า A            │
│                │   👤 Assigned: คุณสมศรี             │
└────────────────┴────────────────────────────────────┘
```

### 9.1.2 Response Modes

LINE OA รองรับ 3 โหมดการตอบกลับ:

| โหมด | ลักษณะ | เหมาะสำหรับ |
|------|--------|------------|
| **Chat** | ตอบโดยทีมงานคนเดียว | ธุรกิจขนาดเล็ก |
| **Bot** | ตอบอัตโนมัติทั้งหมด | ระบบ Self-service |
| **Bot and Chat** | ผสมผสาน Bot + คน | ธุรกิจส่วนใหญ่ |

**การเปลี่ยน Response Mode:**
1. ไปที่ **"Chat settings"** > **"Response"**
2. เลือกโหมดที่ต้องการ
3. บันทึกการตั้งค่า

> **แนะนำ**: ใช้ "Bot and Chat" เพื่อให้ Bot ตอบคำถามทั่วไป และทีมงานรับเฉพาะกรณีที่ซับซ้อน

### 9.1.3 การส่งข้อความจาก Chat Manager

**ประเภทข้อความที่ส่งได้:**
- ข้อความธรรมดา
- รูปภาพ
- ไฟล์
- สติกเกอร์
- เสียง
- ตำแหน่งที่ตั้ง

**Shortcuts ที่มีประโยชน์:**
```
Enter: ขึ้นบรรทัดใหม่
Ctrl + Enter / Cmd + Enter: ส่งข้อความ
Ctrl + K / Cmd + K: ค้นหา Chat
```

### 9.1.4 เคล็ดลับการใช้ Chat อย่างมีประสิทธิภาพ

**Template Responses:**

สร้าง Template ข้อความที่ใช้บ่อยเพื่อประหยัดเวลา:

```
Template 1: ทักทาย
"สวัสดีครับคุณ [ชื่อ]! ยินดีให้บริการนะครับ 
มีอะไรให้ช่วยเหลือไหมครับ?"

Template 2: ขอข้อมูลเพิ่มเติม
"ขอบคุณที่ติดต่อมาครับ กรุณาระบุ:
1. ชื่อ-นามสกุล
2. เบอร์โทรศัพท์
3. รายละเอียดที่ต้องการ"

Template 3: ปิด Chat
"ขอบคุณที่ติดต่อ [ชื่อร้าน] นะครับ!
หากมีอะไรสงสัยเพิ่มเติม สามารถทักมาได้เสมอครับ 🙏"
```

---

## 9.2 Chat Tags

### 9.2.1 Tags คืออะไรและทำไมต้องใช้

**Chat Tags** คือป้ายกำกับที่คุณสามารถติดให้กับการสนทนาและผู้ใช้แต่ละราย เพื่อจัดหมวดหมู่และค้นหาได้ง่าย

**ประโยชน์ของ Tags:**
1. จัดกลุ่มลูกค้าตามคุณสมบัติ
2. Filter การสนทนาตาม Tag
3. ส่ง Narrowcast ไปยัง Audience ที่มี Tag เฉพาะ
4. ดู Analytics แยกตาม Segment
5. มอบหมายงานตาม Tag

### 9.2.2 การสร้างและจัดการ Tags

**สร้าง Tag ใหม่:**
1. ใน Chat Conversation คลิกที่ไอคอน Tag (🏷️)
2. คลิก **"Create tag"** หรือ **"+ สร้าง Tag ใหม่"**
3. ใส่ชื่อ Tag (เช่น "VIP", "ลูกค้าใหม่", "รอชำระเงิน")
4. เลือกสี (เพื่อให้มองเห็นง่าย)
5. กด Save

**การจัดการ Tags จากการตั้งค่า:**
1. ไปที่ **"Settings"** > **"Tags"**
2. ดูรายการ Tags ทั้งหมด
3. สร้าง แก้ไข หรือลบ Tags

### 9.2.3 ระบบ Tag ที่แนะนำ

**สำหรับร้านค้าออนไลน์:**

```
Status Tags (สถานะ):
🔴 รอตอบ         - ยังไม่ได้ตอบกลับ
🟡 รอลูกค้า       - รอข้อมูลจากลูกค้า
🟢 รอจัดส่ง       - ออเดอร์กำลังจัดส่ง
⚫ เสร็จสิ้น       - ปิดเคสแล้ว

Customer Tags (ประเภทลูกค้า):
⭐ VIP           - ลูกค้า VIP
🆕 ใหม่           - ลูกค้าใหม่
🔄 ประจำ          - ลูกค้าประจำ
💎 พรีเมียม        - สมาชิก Premium

Product Tags (สินค้า):
👗 เสื้อผ้า         - ถามเรื่องเสื้อผ้า
👜 กระเป๋า         - ถามเรื่องกระเป๋า
👠 รองเท้า         - ถามเรื่องรองเท้า

Issue Tags (ปัญหา):
🔄 คืนสินค้า        - ต้องการคืนสินค้า
📦 พัสดุหาย         - พัสดุไม่ถึง
💳 ชำระเงินมีปัญหา   - ปัญหาการชำระเงิน
```

**สำหรับบริการ B2B:**

```
Business Stage:
🎯 Lead              - ลูกค้าใหม่ที่สนใจ
💬 Negotiating        - กำลังเจรจา
✅ Closed             - ปิดการขายแล้ว
❌ Lost                - ไม่สำเร็จ

Size:
🏢 Enterprise         - บริษัทใหญ่
🏬 SME                - ธุรกิจกลาง
🏪 Small               - ธุรกิจเล็ก
```

### 9.2.4 การใช้ Tags สำหรับ Analytics

หลังจาก Tag ลูกค้าแล้ว สามารถใช้ประโยชน์ได้หลายอย่าง:

**1. Filter Chat ตาม Tag:**
- คลิก "Filter" ในหน้า Chat List
- เลือก Tag ที่ต้องการ
- เห็น Chat ทั้งหมดที่มี Tag นั้น

**2. สร้าง Audience จาก Tag (สำหรับ Broadcast):**
- ไปที่ **Audience** > **Create new audience**
- เลือก **"Chat tag audience"**
- เลือก Tag ที่ต้องการ
- ใช้ Audience นี้ใน Narrowcast

**3. ดู Analytics แยกตาม Tag:**

```
ตัวอย่างการวิเคราะห์:
- Tag "ลูกค้าใหม่" มีกี่ราย → วัด Growth
- Tag "รอชำระเงิน" ค้างอยู่กี่ราย → Follow-up
- Tag "VIP" มี Average Order Value เท่าไหร่ → Upsell
```

### 9.2.5 การจัดการ Tags ผ่าน API

```javascript
// ดู Chat Tags ของผู้ใช้
async function getUserChatTags(userId) {
  const response = await axios.get(
    `https://api.line.me/v2/bot/profile/${userId}`,
    { headers }
  );
  
  // Tags จัดการผ่าน LINE OA Manager เท่านั้น
  // API ไม่รองรับการอ่าน/เขียน Tags โดยตรง
  // แต่สามารถใช้ Webhook ร่วมกับ Custom Database ได้
  return response.data;
}

// ระบบ Tag ด้วย Custom Database
const userTagsDB = new Map(); // ในโปรเจกต์จริงใช้ Database

function addTagToUser(userId, tag) {
  const currentTags = userTagsDB.get(userId) || [];
  if (!currentTags.includes(tag)) {
    currentTags.push(tag);
    userTagsDB.set(userId, currentTags);
  }
  return currentTags;
}

function removeTagFromUser(userId, tag) {
  const currentTags = userTagsDB.get(userId) || [];
  const newTags = currentTags.filter(t => t !== tag);
  userTagsDB.set(userId, newTags);
  return newTags;
}

function getUsersByTag(tag) {
  const users = [];
  for (const [userId, tags] of userTagsDB.entries()) {
    if (tags.includes(tag)) {
      users.push(userId);
    }
  }
  return users;
}
```

---

## 9.3 Chat Notes

### 9.3.1 ทำไมต้องใช้ Chat Notes

**Chat Notes** คือบันทึกภายในที่ผู้ดูแล LINE OA เขียนเพื่อเก็บข้อมูลสำคัญของลูกค้าหรือเคส Notes จะไม่แสดงให้ลูกค้าเห็น

**กรณีที่ควรใช้:**
- บันทึกข้อตกลงพิเศษที่ทำกับลูกค้า
- จดความชอบ/ไม่ชอบของลูกค้า
- บันทึกประวัติปัญหาที่เคยเกิด
- Handover Notes เมื่อโอน Chat ให้เพื่อนร่วมงาน
- บันทึก Follow-up action ที่ต้องทำ

### 9.3.2 การเพิ่ม Chat Note

**ขั้นตอน:**
1. เปิด Conversation กับลูกค้า
2. หา Panel ด้านขวา (ส่วน Notes)
3. คลิก **"+ Add note"** หรือ **"+ เพิ่มบันทึก"**
4. พิมพ์บันทึก
5. คลิก **"Save"**

> **หมายเหตุ**: บางเวอร์ชันของ LINE OA Manager จะแสดงปุ่ม Notes เป็นไอคอนดินสอ (✏️) ที่ด้านบนของแผง Customer Info ทางด้านขวา

### 9.3.3 Format ของ Notes ที่ดี

```
Template สำหรับ Customer Notes:

📋 CUSTOMER PROFILE
ชื่อ: สมชาย ใจดี
โทร: 08X-XXX-XXXX
อีเมล: somchai@example.com

💡 ข้อมูลสำคัญ:
- สนใจสินค้า Category A
- ต้องการสีเข้ม ไม่ชอบสีพาสเทล
- Budget: 1,000-3,000 บาท
- มักซื้อช่วงสิ้นเดือน

🔄 ประวัติการซื้อ:
- 01/09: ซื้อสินค้า A ราคา 1,200 บาท (พอใจมาก)
- 15/09: ซื้อสินค้า B ราคา 800 บาท (คืนของ - ไซส์ไม่พอ)

⚠️ ข้อควรระวัง:
- เคยมีปัญหาเรื่องจัดส่งช้า ต้องติดตามพิเศษ
- ชอบให้ส่วนลด ทักถามก่อนทุกครั้ง

📅 Follow-up:
- ต้องส่งโปรโมชันเดือนหน้า
- รอยืนยันออเดอร์พรุ่งนี้
```

### 9.3.4 การใช้ Notes สำหรับ Team Handover

เมื่อต้องโอน Chat ให้เพื่อนร่วมงาน ควรเขียน Handover Note:

```
🔄 HANDOVER NOTE - [วันที่] [เวลา]
จาก: [ชื่อคุณ]
ถึง: [ชื่อผู้รับ]

สถานการณ์:
ลูกค้าติดต่อเรื่อง [ปัญหา/คำถาม]

สิ่งที่ได้ทำไปแล้ว:
✅ ตอบคำถามเรื่อง [X]
✅ ส่งรูป [Y] ไปแล้ว
⏳ รอลูกค้ายืนยัน [Z]

ต้องทำต่อ:
→ ติดตามเรื่อง [A]
→ ส่ง Quote ภายใน [เวลา]

ข้อมูลสำคัญ:
- ออเดอร์ล่าสุด: [หมายเลข]
- VIP Status: [ใช่/ไม่ใช่]
```

---

## 9.4 การมอบหมาย Chat ให้ Operators

### 9.4.1 Chat Assignment คืออะไร

เมื่อมีทีม Support หลายคน การ Assign Chat ให้ Operator คนใดคนหนึ่งรับผิดชอบช่วยป้องกันการสับสน "ใครดูแล Chat นี้?"

### 9.4.2 วิธี Assign Chat

**Manual Assignment:**
1. เปิด Conversation
2. คลิก **"Assign to"** หรือ **"มอบหมายให้"** ใน Panel ขวา
3. เลือก Operator จากรายชื่อ
4. คลิก **"Assign"**

**ผลที่เกิดขึ้น:**
- Operator ที่ได้รับ Assignment จะเห็น Chat ใน "My Chats"
- Operator อื่นยังเห็นได้แต่รู้ว่ามีคนรับผิดชอบแล้ว
- มีไอคอนแสดงชื่อผู้ที่รับผิดชอบ

### 9.4.3 Auto-Assignment Rules

สร้างกฎการ Assign อัตโนมัติ:

```javascript
// auto-assignment.js
// กำหนดกฎการ Assign Chat อัตโนมัติ

const OPERATORS = {
  'sales1': { 
    id: 'op_001', 
    name: 'คุณสมศรี',
    skills: ['sales', 'vip'],
    maxChats: 10 
  },
  'sales2': { 
    id: 'op_002', 
    name: 'คุณสมชาย',
    skills: ['sales', 'general'],
    maxChats: 8 
  },
  'support1': { 
    id: 'op_003', 
    name: 'คุณสมหญิง',
    skills: ['support', 'technical'],
    maxChats: 12 
  }
};

const ASSIGNMENT_RULES = [
  {
    condition: (chat) => chat.tags.includes('VIP'),
    assignTo: 'sales1',
    priority: 1,
    description: 'VIP ลูกค้า → คุณสมศรี'
  },
  {
    condition: (chat) => chat.tags.includes('technical'),
    assignTo: 'support1', 
    priority: 2,
    description: 'ปัญหาเทคนิค → คุณสมหญิง'
  },
  {
    condition: (chat) => chat.isNewCustomer,
    assignTo: 'sales2',
    priority: 3,
    description: 'ลูกค้าใหม่ → คุณสมชาย'
  }
];

async function autoAssignChat(chatId, chatData) {
  // เรียงตาม Priority
  const sortedRules = ASSIGNMENT_RULES.sort((a, b) => a.priority - b.priority);
  
  for (const rule of sortedRules) {
    if (rule.condition(chatData)) {
      const operator = OPERATORS[rule.assignTo];
      
      // ตรวจสอบว่า Operator ว่างพอไหม
      const currentLoad = await getOperatorChatCount(operator.id);
      if (currentLoad < operator.maxChats) {
        await assignChatToOperator(chatId, operator.id);
        console.log(`Auto-assigned: ${rule.description}`);
        return operator;
      }
    }
  }
  
  // Fallback: Load Balancing
  return await assignToLeastBusyOperator(chatId);
}

async function assignToLeastBusyOperator(chatId) {
  let minLoad = Infinity;
  let targetOperator = null;
  
  for (const [key, operator] of Object.entries(OPERATORS)) {
    const load = await getOperatorChatCount(operator.id);
    if (load < minLoad && load < operator.maxChats) {
      minLoad = load;
      targetOperator = operator;
    }
  }
  
  if (targetOperator) {
    await assignChatToOperator(chatId, targetOperator.id);
    return targetOperator;
  }
  
  // ถ้าทุกคนเต็ม ส่งไปหา Queue
  await addToWaitingQueue(chatId);
  return null;
}
```

### 9.4.4 Notification เมื่อได้รับ Assignment

Operators จะได้รับ Notification เมื่อมี Chat ถูก Assign มาให้:
- ใน LINE OA Manager: แถบสีแดงที่ชื่อ Chat
- Notification ใน App (ถ้าติดตั้ง LINE Official Account App)

---

## 9.5 Chat History

### 9.5.1 การดู Chat History

**ดูประวัติการสนทนา:**
1. ค้นหาชื่อลูกค้าหรือข้อความในช่อง Search
2. คลิกที่ Conversation ที่ต้องการ
3. Scroll ขึ้นเพื่อดูข้อความเก่า
4. กด **"Load more"** เพื่อโหลดข้อความเพิ่มเติม

**ค้นหาข้อความเฉพาะ:**
- ใช้ Search ในหน้า Chat ค้นหาคำสำคัญ
- กรอง Chat ตาม Date Range
- กรองตาม Operator, Tag, Status

### 9.5.2 การ Export Chat History

```
วิธี Export:
1. เปิด Conversation ที่ต้องการ
2. คลิกไอคอน "More" (...)
3. เลือก "Export chat"
4. เลือกรูปแบบ (CSV, Text)
5. ดาวน์โหลดไฟล์

รูปแบบข้อมูลที่ Export:
- วันที่เวลา
- ผู้ส่ง (ลูกค้า/Operator)
- เนื้อหาข้อความ
- ประเภทข้อความ
```

### 9.5.3 การดึง Chat History ผ่าน API

```javascript
// ดึงประวัติข้อความ
async function getChatHistory(userId, startDate, endDate) {
  // LINE API ไม่รองรับการดึง Chat History โดยตรง
  // ต้องบันทึกเองผ่าน Webhook
  
  // แนะนำ: บันทึกทุก Message Event ลง Database
  const messages = await db.messages.findAll({
    where: {
      userId: userId,
      timestamp: {
        gte: startDate,
        lte: endDate
      }
    },
    order: [['timestamp', 'ASC']]
  });
  
  return messages;
}

// บันทึก Message ทุกครั้ง
async function saveMessageToHistory(event) {
  const { type, source, message, timestamp } = event;
  
  await db.messages.create({
    userId: source.userId,
    direction: type === 'message' ? 'incoming' : 'outgoing',
    messageType: message?.type || 'text',
    content: message?.text || JSON.stringify(message),
    timestamp: new Date(timestamp),
    operatorId: event.operatorId || null
  });
}

// Webhook ที่บันทึก History
app.post('/webhook', middleware(config), async (req, res) => {
  const events = req.body.events;
  
  await Promise.all(events.map(async (event) => {
    // บันทึกทุก Message Event
    if (event.type === 'message') {
      await saveMessageToHistory(event);
    }
    
    // Handle ต่อไป
    await handleEvent(event);
  }));
  
  res.status(200).json({ status: 'ok' });
});
```

### 9.5.4 Chat History Best Practices

```
นโยบายการเก็บ Chat History:

1. ระยะเวลาเก็บข้อมูล:
   - ตามที่ LINE เก็บ: ประมาณ 7 วัน (Chat Manager)
   - Server-side: เก็บตาม Policy ของบริษัท (แนะนำ 1-3 ปี)

2. PDPA Compliance:
   - แจ้งผู้ใช้ว่าเก็บ Chat History
   - อนุญาตให้ขอลบข้อมูลได้
   - เข้ารหัสข้อมูลใน Storage

3. Access Control:
   - จำกัดว่าใครสามารถดู History ได้
   - Log การเข้าถึง History ทุกครั้ง
```

---

## 9.6 การ Mark as Resolved

### 9.6.1 Workflow การจัดการ Chat

```
Chat Lifecycle:

ลูกค้าส่งข้อความ
        ↓
Status: New (🔴)
        ↓
Operator รับ + ตอบกลับ
        ↓
Status: In Progress (🟡)
        ↓
ปัญหาได้รับการแก้ไข
        ↓
Operator กด "Resolved"
        ↓
Status: Resolved (✅)
        ↓
ถ้าลูกค้าส่งข้อความมาอีก → กลับสู่ New
```

### 9.6.2 การ Mark as Resolved

**ขั้นตอน:**
1. เปิด Conversation ที่ต้องการปิด
2. คลิกปุ่ม **"Resolve"** หรือ **"เสร็จสิ้น"** ที่ด้านบนของ Conversation
3. ระบบจะย้าย Chat ไป "Resolved" tab

**เมื่อไหรควร Resolve:**
- ปัญหาของลูกค้าได้รับการแก้ไขแล้ว
- ออเดอร์ได้รับการยืนยันและจัดส่งแล้ว
- ลูกค้าบอกว่า "ขอบคุณ" หรือ "โอเค"
- ไม่มีการติดต่อกลับภายใน 24-48 ชั่วโมง

### 9.6.3 Auto-resolve ด้วย Bot

```javascript
// auto-resolve.js
// ระบบ Auto-resolve เมื่อไม่มีการตอบกลับ

const RESOLVE_TIMEOUT = 48 * 60 * 60 * 1000; // 48 ชั่วโมง

// ตรวจสอบ Chats ที่ควร Auto-resolve
async function checkAndAutoResolve() {
  const now = Date.now();
  const cutoffTime = new Date(now - RESOLVE_TIMEOUT);
  
  // ดึง Chat ที่ไม่ได้ Active นานกว่า 48 ชั่วโมง
  const inactiveChats = await db.chats.findAll({
    where: {
      status: 'in_progress',
      lastMessageTime: { lt: cutoffTime },
      resolvedAt: null
    }
  });
  
  for (const chat of inactiveChats) {
    // ส่งข้อความแจ้งก่อน Resolve
    await client.pushMessage(chat.userId, {
      type: 'text',
      text: 'เคสของคุณได้รับการปิดโดยอัตโนมัติแล้วนะครับ\nหากยังมีคำถาม สามารถส่งข้อความมาได้เลยครับ 🙏'
    });
    
    // Mark as Resolved
    await db.chats.update(
      { 
        status: 'resolved', 
        resolvedAt: new Date(),
        resolvedBy: 'auto_system'
      },
      { where: { id: chat.id } }
    );
    
    console.log(`Auto-resolved chat for user: ${chat.userId}`);
  }
}

// รัน Auto-resolve ทุก 1 ชั่วโมง
setInterval(checkAndAutoResolve, 60 * 60 * 1000);

// ส่ง Satisfaction Survey หลัง Resolve
async function sendSatisfactionSurvey(userId) {
  await client.pushMessage(userId, {
    type: 'template',
    altText: 'แบบสอบถามความพึงพอใจ',
    template: {
      type: 'buttons',
      text: 'เราให้บริการได้ดีแค่ไหนครับ? 😊',
      actions: [
        {
          type: 'message',
          label: '⭐⭐⭐⭐⭐ ดีมาก',
          text: 'ให้คะแนน 5 ดาว'
        },
        {
          type: 'message',
          label: '⭐⭐⭐⭐ ดี',
          text: 'ให้คะแนน 4 ดาว'
        },
        {
          type: 'message',
          label: '⭐⭐⭐ ปานกลาง',
          text: 'ให้คะแนน 3 ดาว'
        },
        {
          type: 'message',
          label: '⭐⭐ ต้องปรับปรุง',
          text: 'ให้คะแนน 2 ดาว'
        }
      ]
    }
  });
}
```

---

## 9.7 Multiple Operator Setup

### 9.7.1 บทบาทใน LINE OA

| บทบาท | สิทธิ์ | จำกัด |
|-------|------|------|
| **Admin** | ทุกอย่าง รวมถึงตั้งค่าและ Billing | ไม่จำกัด (ขึ้นกับ Plan) |
| **Operator** | Chat, ส่งข้อความ, Tags, Notes | ไม่สามารถเปลี่ยนการตั้งค่าหลัก |

### 9.7.2 การเพิ่ม Operator

**ขั้นตอน:**
1. ไปที่ **"Settings"** > **"Operator management"** หรือ **"การจัดการผู้ดำเนินการ"**
2. คลิก **"Add operator"** หรือ **"เพิ่มผู้ดำเนินการ"**
3. ใส่ LINE Account ของ Operator (ค้นหาโดย ID หรือ QR Code)
4. เลือกบทบาท
5. ส่งคำเชิญ

> **หมายเหตุ**: Operator ต้องยอมรับคำเชิญก่อนจึงจะเข้าระบบได้ LINE จะส่ง Notification ไปหา Operator ทาง LINE

### 9.7.3 การจัดการ Operator ผ่าน API

```javascript
// การดู Operator ทั้งหมดและโหลดงาน

async function getOperatorWorkload() {
  // Mock data - ในระบบจริงดึงจาก Database
  const operators = await db.operators.findAll({
    where: { status: 'active' }
  });
  
  const workload = await Promise.all(
    operators.map(async (op) => {
      const activeChatCount = await db.chats.count({
        where: {
          assignedTo: op.id,
          status: 'in_progress'
        }
      });
      
      const resolvedToday = await db.chats.count({
        where: {
          assignedTo: op.id,
          status: 'resolved',
          resolvedAt: {
            gte: new Date().setHours(0, 0, 0, 0)
          }
        }
      });
      
      return {
        ...op.toJSON(),
        activeChats: activeChatCount,
        resolvedToday: resolvedToday,
        efficiency: `${resolvedToday} resolved today`
      };
    })
  );
  
  return workload;
}

// รายงานประสิทธิภาพ Operator
async function generateOperatorReport(startDate, endDate) {
  const report = await db.query(`
    SELECT 
      o.name as operator_name,
      COUNT(c.id) as total_chats,
      AVG(TIMESTAMPDIFF(MINUTE, c.created_at, c.resolved_at)) as avg_resolution_minutes,
      SUM(CASE WHEN cr.rating >= 4 THEN 1 ELSE 0 END) as high_ratings,
      AVG(cr.rating) as avg_rating
    FROM operators o
    LEFT JOIN chats c ON o.id = c.assigned_to
    LEFT JOIN chat_ratings cr ON c.id = cr.chat_id
    WHERE c.resolved_at BETWEEN :startDate AND :endDate
    GROUP BY o.id
    ORDER BY avg_rating DESC
  `, {
    replacements: { startDate, endDate }
  });
  
  return report;
}
```

### 9.7.4 ระบบ SLA (Service Level Agreement) สำหรับทีม

```javascript
// sla-monitor.js
// ติดตาม Response Time ตาม SLA

const SLA_CONFIG = {
  firstResponse: {
    target: 5, // นาที
    alert: 10   // แจ้งเตือนเมื่อเกิน
  },
  resolution: {
    target: 60,  // นาที
    alert: 90    // แจ้งเตือนเมื่อเกิน
  }
};

async function monitorSLA() {
  const now = new Date();
  
  // ตรวจ Chats ที่ยังไม่ได้รับการตอบกลับ
  const pendingChats = await db.chats.findAll({
    where: {
      status: 'new',
      firstResponseAt: null
    }
  });
  
  for (const chat of pendingChats) {
    const waitingMinutes = (now - new Date(chat.createdAt)) / 60000;
    
    if (waitingMinutes > SLA_CONFIG.firstResponse.alert) {
      await alertSLABreach({
        type: 'first_response',
        userId: chat.userId,
        waitingMinutes: Math.round(waitingMinutes),
        assignedTo: chat.assignedTo
      });
    }
  }
}

async function alertSLABreach(data) {
  // แจ้ง Supervisor ทาง LINE
  const supervisorId = process.env.SUPERVISOR_LINE_ID;
  
  await client.pushMessage(supervisorId, {
    type: 'text',
    text: `⚠️ SLA Alert!\n\nChat ยังไม่ได้รับการตอบกลับ\nรอแล้ว: ${data.waitingMinutes} นาที\nUser: ${data.userId}\nAssigned: ${data.assignedTo || 'ไม่มีคนรับ'}`
  });
}

// รัน Monitor ทุก 2 นาที
setInterval(monitorSLA, 2 * 60 * 1000);
```

---

## 9.8 Chat Labels และการจัดระเบียบ

### 9.8.1 การใช้ Labels และ Filters อย่างมีประสิทธิภาพ

```
สร้าง Custom Views สำหรับทีม:

View 1: "รอตอบ" - Filter: Status = New, No Assignment
View 2: "งานของฉัน" - Filter: Assigned = [ฉัน], Status = In Progress
View 3: "VIP ลูกค้า" - Filter: Tag = VIP, Status != Resolved
View 4: "ปัญหาด่วน" - Filter: Tag = ด่วน OR Status = Escalated
View 5: "ติดตาม" - Filter: Tag = ติดตาม
```

### 9.8.2 Chat Notification Settings

ตั้งค่า Notification เพื่อไม่พลาดข้อความสำคัญ:

```
การตั้งค่า Notification ที่แนะนำ:

Admin:
- New chat from VIP: ทันที
- SLA Breach: ทันที
- Daily report: 18:00 น.

Operator:
- New assigned chat: ทันที
- Customer reply to my chat: ทันที
- Morning reminder: 09:00 น.
```

---

## 9.9 Advanced Chat Features

### 9.9.1 Chat Bot Integration

```javascript
// hybrid-chat.js
// ระบบผสม Bot + Human

const ConversationState = {
  BOT: 'bot',
  HUMAN: 'human',
  WAITING: 'waiting'
};

const chatStates = new Map();

async function handleIncomingMessage(event) {
  const userId = event.source.userId;
  const message = event.message.text || '';
  
  const state = chatStates.get(userId) || ConversationState.BOT;
  
  switch (state) {
    case ConversationState.BOT:
      await handleBotMode(event, userId, message);
      break;
      
    case ConversationState.HUMAN:
      // ส่งต่อให้ Operator ดูใน Chat Manager
      // ไม่ต้อง Auto-reply
      await notifyOperatorNewMessage(userId, message);
      break;
      
    case ConversationState.WAITING:
      await handleWaitingMode(event, userId);
      break;
  }
}

async function handleBotMode(event, userId, message) {
  // ถ้าพิมพ์คำขอพนักงาน
  const humanKeywords = ['พนักงาน', 'เจ้าหน้าที่', 'คน', 'talk to human'];
  const wantsHuman = humanKeywords.some(kw => message.toLowerCase().includes(kw));
  
  if (wantsHuman) {
    chatStates.set(userId, ConversationState.WAITING);
    
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: 'รับทราบครับ กำลังประสานงานเจ้าหน้าที่ให้นะครับ\n⏰ รอสักครู่ (โดยทั่วไปไม่เกิน 5 นาที)'
    });
    
    // แจ้ง Operator ที่ว่าง
    await assignToAvailableOperator(userId);
    return;
  }
  
  // Bot ตอบปกติ
  await handleKeywordReply(event, message);
}

async function assignToAvailableOperator(userId) {
  const availableOp = await findAvailableOperator();
  
  if (availableOp) {
    chatStates.set(userId, ConversationState.HUMAN);
    
    // แจ้ง Operator
    await notifyOperator(availableOp.id, userId);
    
    // แจ้งลูกค้า
    await client.pushMessage(userId, {
      type: 'text',
      text: `เชื่อมต่อกับ ${availableOp.name} แล้วครับ\nสามารถพูดคุยได้เลยนะครับ 😊`
    });
  } else {
    // ไม่มี Operator ว่าง
    await client.pushMessage(userId, {
      type: 'text',
      text: 'ขออภัยนะครับ ขณะนี้เจ้าหน้าที่ทุกท่านยุ่งอยู่\nกรุณารอสักครู่ หรือพิมพ์คำถามไว้ได้เลยครับ 🙏'
    });
    
    // เพิ่มใน Queue
    await addToHumanQueue(userId);
  }
}
```

### 9.9.2 Customer Satisfaction (CSAT) System

```javascript
// csat.js
async function sendCSATSurvey(userId, chatId) {
  await client.pushMessage(userId, {
    type: 'flex',
    altText: 'แบบสอบถามความพึงพอใจ',
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        backgroundColor: '#00B900',
        contents: [{
          type: 'text',
          text: '💬 ความคิดเห็นของคุณ',
          color: '#FFFFFF',
          weight: 'bold',
          size: 'lg',
          align: 'center'
        }]
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: 'เราให้บริการได้ดีแค่ไหนสำหรับการสนทนาครั้งนี้?',
            wrap: true,
            margin: 'sm'
          }
        ]
      },
      footer: {
        type: 'box',
        layout: 'horizontal',
        contents: [1, 2, 3, 4, 5].map(rating => ({
          type: 'button',
          style: rating >= 4 ? 'primary' : 'secondary',
          action: {
            type: 'postback',
            label: '⭐'.repeat(rating),
            data: `action=csat_rating&rating=${rating}&chatId=${chatId}`
          }
        }))
      }
    }
  });
}

// จัดการ CSAT Response
async function handleCSATResponse(event) {
  const data = new URLSearchParams(event.postback.data);
  
  if (data.get('action') === 'csat_rating') {
    const rating = parseInt(data.get('rating'));
    const chatId = data.get('chatId');
    
    // บันทึกคะแนน
    await db.chatRatings.create({
      chatId,
      userId: event.source.userId,
      rating,
      timestamp: new Date()
    });
    
    // ตอบกลับตามคะแนน
    if (rating >= 4) {
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: 'ขอบคุณมากครับ! 🙏 ดีใจที่ได้ให้บริการนะครับ\n\nถ้ามีอะไรให้ช่วยอีก สามารถทักมาได้เสมอครับ 😊'
      });
    } else {
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: 'ขอบคุณสำหรับ Feedback นะครับ 🙏\n\nเราจะปรับปรุงการบริการให้ดียิ่งขึ้นครับ\nหากมีเรื่องที่ต้องการแจ้งเพิ่มเติม พิมพ์มาได้เลยนะครับ'
      });
      
      // แจ้ง Supervisor เมื่อได้คะแนนต่ำ
      await alertLowCSAT(chatId, rating, event.source.userId);
    }
  }
}
```

---

## 9.10 แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: ระบบ Chat Tags (พื้นฐาน)
**เวลา: 45 นาที**

1. สร้าง Tag System สำหรับธุรกิจสมมติ อย่างน้อย 10 Tags
2. จัดหมวดหมู่ Tags เป็น 3 กลุ่ม (Status, Customer Type, Issue Type)
3. ทดสอบ Tag ใน Conversation จริง
4. สร้าง Audience จาก Tag สำหรับ Broadcast

### แบบฝึกหัดที่ 2: Chat Notes Template (ระดับกลาง)
**เวลา: 30 นาที**

1. สร้าง Template สำหรับ Customer Profile Note
2. สร้าง Template สำหรับ Handover Note
3. สร้าง Template สำหรับ Issue Resolution Note
4. Document วิธีใช้ Template ไว้ใน Team Wiki (สร้าง Google Doc)

### แบบฝึกหัดที่ 3: Hybrid Bot-Human System (ระดับสูง)
**เวลา: 4-5 ชั่วโมง**

สร้าง Express.js Server ที่มี:
1. Bot ตอบ FAQ อัตโนมัติ
2. ผู้ใช้สามารถพิมพ์ "พนักงาน" เพื่อโอนไปคุยกับคนจริง
3. ระบบ Queue เมื่อ Operator ยุ่ง
4. CSAT Survey หลัง Resolve ทุกครั้ง
5. Dashboard แสดง:
   - จำนวน Chat ที่ Bot จัดการ
   - จำนวน Chat ที่โอนให้คน
   - Average CSAT Score

---

## สรุปบทที่ 9

ในบทนี้คุณได้เรียนรู้:

- ✅ ฟีเจอร์ 1-on-1 Chat ทั้งหมดและวิธีใช้
- ✅ Chat Tags สำหรับจัดหมวดหมู่และสร้าง Audience
- ✅ Chat Notes สำหรับเก็บข้อมูลสำคัญของลูกค้า
- ✅ การมอบหมาย Chat และ Auto-assignment Rules
- ✅ Chat History และการ Export
- ✅ Workflow การ Resolve และ Auto-resolve
- ✅ การตั้งค่า Multiple Operators และบทบาท
- ✅ ระบบ Hybrid Bot-Human
- ✅ CSAT Survey สำหรับวัดคุณภาพบริการ

**บทถัดไป**: Part 10 จะครอบคลุม Follower Management การจัดการผู้ติดตามและการสร้าง Audience สำหรับการตลาดที่ตรงเป้าหมาย

---

*อัปเดตล่าสุด: ตุลาคม 2024 | LINE Official Account Manager v2024*
