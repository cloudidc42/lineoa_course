# ส่วนที่ 10: Follower Management

## บทนำ

Follower Management คือการจัดการและทำความเข้าใจผู้ติดตาม (Friends) ของ LINE Official Account อย่างเป็นระบบ ซึ่งเป็นรากฐานสำคัญของการทำ Marketing ที่มีประสิทธิภาพ เพราะการส่งข้อความที่ "ใช่" ให้กับ "คนที่ใช่" ในเวลาที่ "เหมาะสม" คือหัวใจของ LINE OA Marketing

ในบทนี้คุณจะเรียนรู้:
- ความเข้าใจเรื่อง Followers และ Friends
- การสร้าง Audience สำหรับ Narrowcast
- ประเภท Audience ต่างๆ
- กลยุทธ์การแบ่ง Segment
- การจัดการผู้ใช้ที่ Block
- Analytics เพื่อเข้าใจ Audience

---

## 10.1 ทำความเข้าใจ Followers/Friends

### 10.1.1 ความแตกต่างของคำศัพท์

| คำศัพท์ | ความหมาย |
|---------|---------|
| **Followers** | ผู้ที่เพิ่ม LINE OA เป็นเพื่อน และยังไม่ได้ Block |
| **Friends** | เหมือนกับ Followers (ใช้สลับกันได้) |
| **Blocked Users** | ผู้ที่เคยเป็น Friend แต่ Block LINE OA ออกไปแล้ว |
| **Unblocked** | ผู้ที่ Unblock กลับมา (นับเป็น Follower ใหม่) |
| **Reached Users** | ผู้ที่ได้รับข้อความ Broadcast อย่างน้อย 1 ครั้ง |
| **Targeted Reach** | ผู้ที่ได้รับ Narrowcast |

### 10.1.2 Friend Counter ทำงานอย่างไร

```
Friend Count = เพิ่มเพื่อนทั้งหมด - Block ทั้งหมด

ตัวอย่าง:
เพิ่มเพื่อน: 10,000 คน
Block: 2,000 คน
Friend Count: 8,000 คน

เมื่อมีคน Block เพิ่ม → Friend Count ลดลง
เมื่อมีคน Unblock → Friend Count เพิ่มขึ้น
```

### 10.1.3 ดู Friend Statistics

**ข้อมูลที่ดูได้ใน Dashboard:**

1. เข้า LINE OA Manager
2. ไปที่ **"Home"** หรือ **"Dashboard"**
3. ดูส่วน **"Friends"**

```
ข้อมูลที่จะเห็น:
📊 Total Friends: 8,234
📈 Added this month: +342
📉 Blocked: -58
🔁 Net Change: +284

กราฟ:
- Friends Growth (เพิ่มขึ้นตลอดเวลา)
- Block Rate (ควรต่ำกว่า 2%)
- Monthly New Friends
```

### 10.1.4 เส้นทางการเป็น Follower

```
วิธีที่คนเพิ่ม LINE OA เป็นเพื่อน:

1. QR Code (สแกนจากสื่อต่างๆ)
   ├── หน้าร้าน / บัตรนามบัตร
   ├── บรรจุภัณฑ์สินค้า
   └── สื่อออนไลน์

2. LINE URL (เปิดเป็นลิงก์)
   └── https://line.me/R/ti/p/@username

3. LINE ID (ค้นหาใน LINE App)
   └── @YourLineID

4. LINE Ads (ผ่านโฆษณา LINE)
   └── Message Ad, Banner Ad

5. Beacon / Geofencing
   └── แจ้งเตือนเมื่ออยู่ใกล้ร้าน

6. Social Media
   └── ลิงก์จาก Facebook, Instagram, Website
```

---

## 10.2 การสร้าง Audience

### 10.2.1 Audience คืออะไร

**Audience** คือกลุ่มผู้ใช้ที่คุณสร้างขึ้นเพื่อใช้ใน Narrowcast หรือ A/B Testing โดยสามารถกำหนดเงื่อนไขต่างๆ ได้ เช่น ผู้ที่คลิกลิงก์ในข้อความก่อนหน้า, ผู้ที่มี Chat Tag, หรือ Upload รายชื่อเอง

### 10.2.2 ขั้นตอนการสร้าง Audience

1. ไปที่ **"Audience"** ในเมนูหลัก
2. คลิก **"Create new audience"**
3. เลือกประเภท Audience
4. ตั้งชื่อและกำหนดเงื่อนไข
5. คลิก **"Create"**

> **ข้อควรรู้**: Audience ต้องมีสมาชิกอย่างน้อย 50 คนจึงจะใช้ Narrowcast ได้

---

## 10.3 ประเภท Audience

### 10.3.1 Upload Audience (อัปโหลดรายชื่อ)

**เหมาะสำหรับ:**
- มีรายชื่อลูกค้าเดิมจาก CRM
- นำเข้าสมาชิกจากระบบอื่น
- Retargeting จาก E-commerce platform

**รูปแบบไฟล์ที่รองรับ:**

```csv
# ไฟล์ CSV - Upload ด้วย LINE User ID
userId
Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxy
Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxz
```

**ขั้นตอน:**
1. สร้างไฟล์ CSV ที่มีคอลัมน์ `userId`
2. ใส่ LINE User ID ทีละบรรทัด
3. อัปโหลดไฟล์ (สูงสุด 1.5 ล้านบรรทัด)

**วิธีดึง User IDs:**

```javascript
// ดึง User ID จาก Webhook Events
const userIds = new Set();

app.post('/webhook', middleware(config), async (req, res) => {
  const events = req.body.events;
  
  events.forEach(event => {
    if (event.source.userId) {
      userIds.add(event.source.userId);
    }
  });
  
  res.status(200).json({ status: 'ok' });
});

// Export User IDs เป็น CSV
const { createObjectCsvWriter } = require('csv-writer');

async function exportUserIdsToCsv(filename = 'audience.csv') {
  const csvWriter = createObjectCsvWriter({
    path: filename,
    header: [{ id: 'userId', title: 'userId' }]
  });
  
  const records = Array.from(userIds).map(id => ({ userId: id }));
  await csvWriter.writeRecords(records);
  
  console.log(`Exported ${records.length} user IDs to ${filename}`);
}
```

### 10.3.2 Click Audience (ผู้ที่คลิก URL)

**เหมาะสำหรับ:**
- Retarget ผู้ที่คลิกลิงก์แต่ยังไม่ซื้อ
- ส่งข้อความ Follow-up ให้ผู้ที่สนใจ
- วัด Engagement ของ Campaign

**วิธีตั้งค่า:**
1. เมื่อสร้าง Broadcast/Narrowcast กำหนด Tracking ID ให้กับ URL
2. หลังส่งแล้ว ไปที่ **Audience** > **Create** > **Click audience**
3. เลือก Message ที่ต้องการดู Clickers
4. เลือก Tracking ID (URL) ที่ต้องการ
5. ตั้งชื่อ Audience และ Save

**ตัวอย่างการใช้ Click Audience:**

```
Campaign Flow:
Day 1: ส่ง Broadcast เรื่องสินค้าใหม่
       └── URL: https://example.com/new-product (tracking_id: new_product_oct)

Day 3: ตรวจสอบ Click Audience
       └── พบว่ามี 500 คนคลิกแต่ยังไม่ซื้อ

Day 4: ส่ง Narrowcast ให้ Click Audience
       └── ข้อความ: "คุณสนใจสินค้า X ใช่ไหม? รับส่วนลดพิเศษ 15% วันนี้!"
```

### 10.3.3 Impression Audience (ผู้ที่เห็นข้อความ)

**เหมาะสำหรับ:**
- ส่งข้อความซ้ำให้ผู้ที่เห็นแต่ไม่คลิก
- วัดว่าใครเปิดอ่านข้อความ

**การใช้งาน:**
- คล้ายกับ Click Audience แต่ใช้ Open Event แทน Click
- สร้างหลังจาก Broadcast ส่งแล้วอย่างน้อย 24 ชั่วโมง

### 10.3.4 Video View Audience (ผู้ที่ดูวิดีโอ)

**เหมาะสำหรับ:**
- Retarget ผู้ที่ดูวิดีโอบางส่วน
- แบ่งตาม Percentage ที่ดู (25%, 50%, 75%, 100%)

**ตัวอย่าง:**

```
Campaign วิดีโอ Tutorial:
Send: วิดีโอ How-to สินค้า

Audience Segments:
- ดูครบ 100%: น่าจะสนใจมาก → ส่ง Offer ตรงๆ
- ดู 50-75%: สนใจพอสมควร → ส่งข้อมูลเพิ่มเติม
- ดู <25%: อาจไม่สนใจ → ข้ามหรือส่ง Different Angle
```

### 10.3.5 Chat Audience (ผู้ที่ Chat ด้วย Tag)

**เหมาะสำหรับ:**
- ส่งข้อความตาม Segment จาก Chat Tags
- Follow-up ลูกค้าที่มี Tag เฉพาะ

**วิธีสร้าง:**
1. **Audience** > **Create new audience**
2. เลือก **"Chat tag audience"**
3. เลือก Tag เช่น "VIP" หรือ "รอชำระเงิน"
4. กำหนดเงื่อนไข: มี Tag นี้ / ไม่มี Tag นี้
5. บันทึก Audience

### 10.3.6 Add Friend Audience (ช่องทางที่เพิ่มเพื่อน)

ดึง Audience ตาม Source ที่คนเพิ่มเพื่อน:

```
Sources ที่ติดตามได้:
- From QR Code
- From LINE Ads
- From Search in LINE
- From Website (Social Plugin)
- Manually (ค้นหา LINE ID)

ประโยชน์:
รู้ว่า Channel ไหนที่ Effective ที่สุด
→ ลงทุนใน Channel นั้นมากขึ้น
```

---

## 10.4 กลยุทธ์การแบ่ง Segment

### 10.4.1 RFM Segmentation สำหรับ LINE OA

**RFM** = Recency (ความล่าสุด) + Frequency (ความถี่) + Monetary (มูลค่า)

```
R - Recency: ล่าสุดทำอะไรบน LINE OA เมื่อไหร่
  1 = มากกว่า 90 วัน
  2 = 31-90 วัน  
  3 = 8-30 วัน
  4 = 2-7 วัน
  5 = วันนี้หรือเมื่อวาน

F - Frequency: ซื้อกี่ครั้ง / Chat กี่ครั้ง
  1 = 1 ครั้ง
  2 = 2-3 ครั้ง
  3 = 4-6 ครั้ง
  4 = 7-12 ครั้ง
  5 = 13+ ครั้ง

M - Monetary: ใช้เงินไปเท่าไหร่
  1 = 0-500 บาท
  2 = 501-1,500 บาท
  3 = 1,501-5,000 บาท
  4 = 5,001-15,000 บาท
  5 = 15,001+ บาท
```

**การแบ่ง Segment ตาม RFM:**

```javascript
// rfm-segmentation.js
function calculateRFMSegment(customer) {
  const { recencyScore, frequencyScore, monetaryScore } = customer;
  const totalScore = recencyScore + frequencyScore + monetaryScore;
  
  // Champions: R=5, F=5, M=5
  if (recencyScore >= 4 && frequencyScore >= 4 && monetaryScore >= 4) {
    return 'champions';
  }
  
  // Loyal Customers: F=4-5
  if (frequencyScore >= 4) {
    return 'loyal_customers';
  }
  
  // Potential Loyalists: R=3-5, F=1-3, M=1-3
  if (recencyScore >= 3 && frequencyScore <= 3) {
    return 'potential_loyalists';
  }
  
  // At Risk: R=2-3, F=3-5, M=3-5
  if (recencyScore <= 3 && frequencyScore >= 3 && monetaryScore >= 3) {
    return 'at_risk';
  }
  
  // Can't Lose: R=1-2, F=4-5
  if (recencyScore <= 2 && frequencyScore >= 4) {
    return 'cant_lose';
  }
  
  // Lost: R=1-2, F=1-2
  if (recencyScore <= 2 && frequencyScore <= 2) {
    return 'lost';
  }
  
  return 'others';
}

// Strategy ต่อ Segment
const SEGMENT_STRATEGIES = {
  champions: {
    message: 'Early access โปรใหม่ + ขอบคุณที่เป็นลูกค้า',
    frequency: 'รายสัปดาห์',
    offer: 'ส่วนลดพิเศษ + Free Gift'
  },
  loyal_customers: {
    message: 'โปรโมชัน Member Exclusive',
    frequency: '2 สัปดาห์ครั้ง',
    offer: 'Loyalty Points + Tier Upgrade'
  },
  potential_loyalists: {
    message: 'แนะนำ Loyalty Program',
    frequency: 'รายสัปดาห์',
    offer: 'โปรซื้อครั้งที่ 2 รับส่วนลด'
  },
  at_risk: {
    message: 'Win-back Message - เราคิดถึงคุณ',
    frequency: '1-2 สัปดาห์',
    offer: 'ส่วนลด 20% ถ้ากลับมาซื้อภายใน 7 วัน'
  },
  cant_lose: {
    message: 'VIP Exclusive Offer',
    frequency: 'รายสัปดาห์',
    offer: 'Personal Shopper + Free Delivery'
  },
  lost: {
    message: 'Re-engagement Campaign',
    frequency: 'รายเดือน',
    offer: 'Heavy Discount เพื่อดึงกลับมา'
  }
};
```

### 10.4.2 Behavioral Segmentation

แบ่งตามพฤติกรรมบน LINE OA:

```
Segment 1: Power Users
เกณฑ์: เปิดอ่าน Broadcast >70%, คลิก >20%
กลยุทธ์: ส่งเนื้อหา Premium, Early Access

Segment 2: Passive Readers  
เกณฑ์: เปิดอ่าน Broadcast 30-70%, คลิก <10%
กลยุทธ์: ปรับปรุง CTA, ลอง Format ต่างๆ

Segment 3: Low Engagement
เกณฑ์: เปิดอ่าน <30%
กลยุทธ์: ส่งน้อยลง, เน้น High-value Content

Segment 4: Re-engagement Targets
เกณฑ์: ไม่เปิดอ่านนาน >60 วัน
กลยุทธ์: Win-back Campaign พร้อม Strong Offer
```

### 10.4.3 Lifecycle Segmentation

```
New Followers (Week 1-2):
→ Onboarding sequence
→ แนะนำ Products/Services
→ ให้ Welcome Offer

Engaged Followers (Month 1-3):
→ Regular Content
→ Product Education
→ Community Building

Loyal Followers (Month 3+):
→ Exclusive Benefits
→ Referral Programs
→ Co-creation

Dormant (No activity >90 days):
→ Re-engagement Campaign
→ Survey: ยังสนใจอยู่ไหม?
```

### 10.4.4 Implementation ระบบ Segmentation

```javascript
// segmentation-system.js
const db = require('./database');

class UserSegmentationSystem {
  constructor() {
    this.segments = new Map();
  }
  
  // คำนวณ Recency Score
  calculateRecencyScore(lastActivityDate) {
    const daysSinceActivity = (Date.now() - new Date(lastActivityDate)) / (1000 * 60 * 60 * 24);
    
    if (daysSinceActivity <= 1) return 5;
    if (daysSinceActivity <= 7) return 4;
    if (daysSinceActivity <= 30) return 3;
    if (daysSinceActivity <= 90) return 2;
    return 1;
  }
  
  // คำนวณ Frequency Score
  calculateFrequencyScore(totalPurchases) {
    if (totalPurchases >= 13) return 5;
    if (totalPurchases >= 7) return 4;
    if (totalPurchases >= 4) return 3;
    if (totalPurchases >= 2) return 2;
    return 1;
  }
  
  // คำนวณ Monetary Score
  calculateMonetaryScore(totalSpend) {
    if (totalSpend >= 15001) return 5;
    if (totalSpend >= 5001) return 4;
    if (totalSpend >= 1501) return 3;
    if (totalSpend >= 501) return 2;
    return 1;
  }
  
  // Update Segment ของผู้ใช้
  async updateUserSegment(userId) {
    const user = await db.users.findOne({ where: { lineUserId: userId } });
    if (!user) return null;
    
    const recency = this.calculateRecencyScore(user.lastActivityAt);
    const frequency = this.calculateFrequencyScore(user.totalOrders);
    const monetary = this.calculateMonetaryScore(user.totalSpend);
    
    const segment = calculateRFMSegment({
      recencyScore: recency,
      frequencyScore: frequency,
      monetaryScore: monetary
    });
    
    await db.users.update(
      { 
        rfmRecency: recency,
        rfmFrequency: frequency,
        rfmMonetary: monetary,
        segment: segment,
        segmentUpdatedAt: new Date()
      },
      { where: { lineUserId: userId } }
    );
    
    return segment;
  }
  
  // ดึงผู้ใช้ตาม Segment
  async getUsersBySegment(segmentName) {
    return await db.users.findAll({
      where: { segment: segmentName },
      attributes: ['lineUserId', 'displayName', 'segment']
    });
  }
  
  // Export Audience สำหรับ Narrowcast
  async exportAudienceForSegment(segmentName, filename) {
    const users = await this.getUsersBySegment(segmentName);
    const csvWriter = createObjectCsvWriter({
      path: filename,
      header: [{ id: 'userId', title: 'userId' }]
    });
    
    const records = users.map(u => ({ userId: u.lineUserId }));
    await csvWriter.writeRecords(records);
    
    return records.length;
  }
}

// ตั้ง Cron Job อัปเดต Segment ทุกคืน
const cron = require('node-cron');
const segSystem = new UserSegmentationSystem();

cron.schedule('0 2 * * *', async () => {
  console.log('เริ่ม Update User Segments...');
  
  const allUsers = await db.users.findAll({ attributes: ['lineUserId'] });
  
  for (const user of allUsers) {
    await segSystem.updateUserSegment(user.lineUserId);
  }
  
  console.log(`อัปเดต Segment เสร็จสิ้น: ${allUsers.length} users`);
});
```

---

## 10.5 การจัดการผู้ใช้ที่ Block

### 10.5.1 ทำไมคนถึง Block LINE OA

**สาเหตุหลัก:**

1. ข้อความมากเกินไป (Spam)
2. เนื้อหาไม่เกี่ยวข้องกับความสนใจ
3. ข้อความตอนดึกหรือเช้ามืด
4. ไม่ได้รับคุณค่าจากการเป็น Follower
5. ลืมว่าเพิ่มเพื่อนไว้
6. ต้องการล้างรายชื่อเพื่อน

**อัตรา Block ที่ยอมรับได้:**

```
Block Rate = Block ใหม่ / Broadcast ที่ส่ง × 100%

< 0.5%: ดีมาก
0.5-1%: ยอมรับได้
1-2%: ต้องปรับปรุง Content
> 2%: ปัญหาร้ายแรง ต้องแก้ไขทันที
```

### 10.5.2 การติดตาม Block Rate

```javascript
// track-block-events.js
// บันทึก Follow/Unfollow Events

handler.add(FollowEvent, async (event) => {
  const userId = event.source.userId;
  
  // บันทึก New Follower
  await db.followers.create({
    userId,
    status: 'active',
    followedAt: new Date(),
    source: await detectFollowSource(userId)
  });
  
  console.log(`New Follower: ${userId}`);
});

handler.add(UnfollowEvent, async (event) => {
  const userId = event.source.userId;
  
  // บันทึก Block Event
  await db.followers.update(
    { 
      status: 'blocked', 
      blockedAt: new Date() 
    },
    { where: { userId } }
  );
  
  // วิเคราะห์ว่า Block เกิดหลัง Broadcast ไหน
  const recentBroadcast = await getRecentBroadcast();
  if (recentBroadcast) {
    await db.broadcastMetrics.increment('blockCount', {
      where: { id: recentBroadcast.id }
    });
  }
  
  console.log(`User Blocked: ${userId}`);
});

// คำนวณ Block Rate รายวัน
async function calculateDailyBlockRate() {
  const yesterday = new Date();
  yesterday.setDate(yesterday.getDate() - 1);
  yesterday.setHours(0, 0, 0, 0);
  
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  
  const [newFollowers, newBlocks, broadcasts] = await Promise.all([
    db.followers.count({
      where: {
        followedAt: { gte: yesterday, lt: today },
        status: 'active'
      }
    }),
    db.followers.count({
      where: {
        blockedAt: { gte: yesterday, lt: today },
        status: 'blocked'
      }
    }),
    db.broadcasts.count({
      where: {
        sentAt: { gte: yesterday, lt: today }
      }
    })
  ]);
  
  const blockRate = broadcasts > 0 ? (newBlocks / broadcasts) * 100 : 0;
  
  return {
    date: yesterday.toLocaleDateString('th-TH'),
    newFollowers,
    newBlocks,
    blockRate: blockRate.toFixed(2) + '%'
  };
}
```

### 10.5.3 กลยุทธ์ลด Block Rate

**1. Content Value Check:**
```
ทุกข้อความที่จะส่ง ถามตัวเองก่อน:
✅ ข้อมูลนี้มีประโยชน์กับผู้ใช้ไหม?
✅ ผู้ใช้จะรู้สึกดีเมื่อได้รับหรือเปล่า?
✅ เราส่งบ่อยเกินไปไหม?
✅ ข้อมูลนี้เหมาะกับกลุ่มนี้จริงๆ ไหม?

❌ ถ้าตอบ "ไม่" แม้แต่ข้อเดียว → ปรับปรุงก่อนส่ง
```

**2. Frequency Optimization:**
```
แนวทาง:
- ไม่เกิน 4-6 ครั้งต่อเดือน (General)
- ไม่เกิน 2-3 ครั้งต่อสัปดาห์ (Active Promotion)
- ห้ามส่งหลายข้อความในวันเดียว (เว้นมีเหตุสำคัญ)

เครื่องมือช่วย:
- Content Calendar ล่วงหน้า 1 เดือน
- Review ก่อนส่งทุกครั้ง
```

**3. Segmented Messaging:**
```
แทนที่จะส่งให้ทุกคน:
ส่งเฉพาะคนที่น่าจะสนใจ

ตัวอย่าง:
✅ โปรโมชันเสื้อผ้าผู้หญิง → ส่งเฉพาะ Segment หญิง
✅ โปรโมชันสินค้า B2B → ส่งเฉพาะ Segment ธุรกิจ
✅ Flash Sale สินค้า High-end → ส่งเฉพาะ VIP / High Monetary
```

### 10.5.4 Win-back Campaign สำหรับ Blocked Users

> **ข้อสำคัญ**: ผู้ที่ Block แล้วจะไม่ได้รับข้อความจาก LINE OA อีก คุณ**ไม่สามารถ** ส่งข้อความไปหาพวกเขาได้โดยตรง

**กลยุทธ์ทางอ้อม:**

```
1. Retarget ผ่าน LINE Ads
   - สร้าง Custom Audience จาก User IDs ที่ Block
   - รัน Ad เพื่อกระตุ้นให้ Unblock และ Follow กลับ

2. Other Channels
   - Email Marketing (ถ้ามีอีเมล)
   - SMS (ถ้ามีเบอร์โทร)
   - Facebook/Instagram Retargeting

3. ป้องกันการ Block ล่วงหน้า
   - Preference Center ให้ผู้ใช้เลือกประเภท Content
   - Easy Opt-down (ลดความถี่แทน Block)
```

---

## 10.6 Analytics สำหรับ Audience Insights

### 10.6.1 Friend Demographics

LINE OA Manager แสดงข้อมูล Demographics (ในบัญชีที่มีข้อมูลเพียงพอ):

```
ข้อมูลที่ดูได้:
- เพศ (เพศชาย/เพศหญิง/ไม่ระบุ)
- อายุ (กลุ่มอายุ)
- ระบบปฏิบัติการ (iOS/Android)
- ภูมิภาค/จังหวัด

หน้าที่ดูข้อมูล:
LINE OA Manager → Analytics → Friends
```

### 10.6.2 Audience Analytics Dashboard

```javascript
// audience-analytics.js
// สร้าง Custom Analytics Dashboard

async function generateAudienceReport() {
  const [
    totalFollowers,
    newThisMonth,
    blockThisMonth,
    segmentDistribution,
    topLocations,
    deviceBreakdown
  ] = await Promise.all([
    // จำนวน Followers ทั้งหมด
    db.followers.count({ where: { status: 'active' } }),
    
    // ใหม่เดือนนี้
    db.followers.count({
      where: {
        status: 'active',
        followedAt: { gte: startOfMonth() }
      }
    }),
    
    // Block เดือนนี้
    db.followers.count({
      where: {
        status: 'blocked',
        blockedAt: { gte: startOfMonth() }
      }
    }),
    
    // การกระจาย Segment
    db.users.findAll({
      attributes: [
        'segment',
        [db.fn('COUNT', db.col('id')), 'count']
      ],
      group: ['segment'],
      order: [[db.fn('COUNT', db.col('id')), 'DESC']]
    }),
    
    // Top Locations (ถ้าเก็บข้อมูลไว้)
    db.users.findAll({
      attributes: [
        'province',
        [db.fn('COUNT', db.col('id')), 'count']
      ],
      group: ['province'],
      order: [[db.fn('COUNT', db.col('id')), 'DESC']],
      limit: 10
    }),
    
    // Device Breakdown
    db.users.findAll({
      attributes: [
        'device',
        [db.fn('COUNT', db.col('id')), 'count']
      ],
      group: ['device']
    })
  ]);
  
  return {
    summary: {
      total: totalFollowers,
      newThisMonth,
      blockThisMonth,
      growthRate: ((newThisMonth - blockThisMonth) / totalFollowers * 100).toFixed(2)
    },
    segments: segmentDistribution,
    topLocations,
    deviceBreakdown
  };
}

// Cohort Analysis
async function getCohortAnalysis() {
  const cohorts = [];
  
  // วิเคราะห์ทีละเดือนย้อนหลัง 6 เดือน
  for (let i = 0; i < 6; i++) {
    const monthStart = new Date();
    monthStart.setMonth(monthStart.getMonth() - i);
    monthStart.setDate(1);
    monthStart.setHours(0, 0, 0, 0);
    
    const monthEnd = new Date(monthStart);
    monthEnd.setMonth(monthEnd.getMonth() + 1);
    
    const cohortFollowers = await db.followers.findAll({
      where: {
        followedAt: { gte: monthStart, lt: monthEnd }
      },
      attributes: ['userId', 'followedAt', 'blockedAt', 'status']
    });
    
    const total = cohortFollowers.length;
    const retained = cohortFollowers.filter(f => f.status === 'active').length;
    
    cohorts.push({
      month: monthStart.toLocaleDateString('th-TH', { month: 'long', year: 'numeric' }),
      total,
      retained,
      retentionRate: total > 0 ? (retained / total * 100).toFixed(1) + '%' : 'N/A'
    });
  }
  
  return cohorts;
}
```

### 10.6.3 Growth Tracking

```javascript
// growth-tracker.js
async function trackFollowerGrowth() {
  const today = new Date();
  today.setHours(0, 0, 0, 0);
  
  const metrics = {
    date: today.toISOString().split('T')[0],
    totalFollowers: await getTotalFollowers(),
    newToday: await getNewFollowersToday(),
    blockedToday: await getBlockedToday(),
    netGrowth: null
  };
  
  metrics.netGrowth = metrics.newToday - metrics.blockedToday;
  
  // บันทึกเพื่อดูแนวโน้ม
  await db.growthMetrics.create(metrics);
  
  // แจ้งเตือนถ้า Block Rate สูง
  if (metrics.blockedToday > metrics.newToday) {
    await alertHighBlockRate(metrics);
  }
  
  return metrics;
}

// ดู Growth Trend 30 วัน
async function getGrowthTrend(days = 30) {
  const startDate = new Date();
  startDate.setDate(startDate.getDate() - days);
  
  const metrics = await db.growthMetrics.findAll({
    where: { date: { gte: startDate } },
    order: [['date', 'ASC']]
  });
  
  return metrics.map(m => ({
    date: m.date,
    followers: m.totalFollowers,
    growth: m.netGrowth
  }));
}
```

### 10.6.4 Broadcast Performance per Audience Segment

```javascript
// campaign-analytics.js
async function analyzeCampaignBySegment(broadcastId) {
  const results = {};
  const segments = ['champions', 'loyal_customers', 'at_risk', 'new'];
  
  for (const segment of segments) {
    const audienceIds = await getUsersBySegment(segment);
    
    const opens = await db.broadcastOpens.count({
      where: {
        broadcastId,
        userId: audienceIds
      }
    });
    
    const clicks = await db.broadcastClicks.count({
      where: {
        broadcastId,
        userId: audienceIds
      }
    });
    
    const conversions = await db.orders.count({
      where: {
        createdAt: { gte: getBroadcastSentAt(broadcastId) },
        userId: audienceIds,
        source: 'line_broadcast'
      }
    });
    
    results[segment] = {
      sent: audienceIds.length,
      opens,
      clicks,
      conversions,
      openRate: (opens / audienceIds.length * 100).toFixed(1) + '%',
      clickRate: (clicks / audienceIds.length * 100).toFixed(1) + '%',
      conversionRate: (conversions / audienceIds.length * 100).toFixed(1) + '%'
    };
  }
  
  return results;
}
```

---

## 10.7 Friend Growth Strategies

### 10.7.1 Online Channels

**1. Website Integration:**

```html
<!-- LINE Social Plugin บนเว็บไซต์ -->
<a href="https://line.me/R/ti/p/@your_line_id">
  <img src="https://scdn.line-apps.com/n/line_add_friends/btn/th.png" 
       alt="เพิ่มเพื่อน LINE" 
       height="36" 
       border="0"
  />
</a>

<!-- หรือใช้ LINE Badge Widget -->
<script src="https://scdn.line-apps.com/n/line_id/widget/ja/widget_static.js"></script>
```

**2. Social Media:**

```
Facebook:
- ลิงก์ LINE ใน Bio
- Post แชร์ QR Code
- Facebook Ads → LINE Add Friend

Instagram:
- Link in Bio ไปหน้า Landing Page ที่มี LINE QR
- Story ที่มี Swipe Up (ถ้ามี Followers >10K)

TikTok:
- QR Code ใน Video
- Link in Bio

LINE OpenChat → Invite เข้า LINE OA
```

**3. Email:**

```html
<!-- Signature ในอีเมล -->
<p>ติดตามข่าวสารเพิ่มเติมผ่าน LINE:</p>
<a href="https://line.me/R/ti/p/@your_line_id">
  <img src="line-button.png" width="120" />
</a>
```

### 10.7.2 Offline Channels

**QR Code บนสื่อต่างๆ:**

```
สื่อที่ควรมี QR Code:
1. นามบัตร - มุมล่างขวา
2. บรรจุภัณฑ์ - ด้านหลัง / ด้านข้าง
3. ป้ายหน้าร้าน - ระดับสายตา
4. ใบเสร็จ - ด้านล่าง
5. Brochure/Leaflet - หน้าหลัง
6. โปสเตอร์ - มุมที่เห็นชัด

QR Code Design:
- ขนาดขั้นต่ำ: 3x3 cm (บน print)
- มี Clear Zone รอบๆ อย่างน้อย 0.5 cm
- โลโก้ LINE อยู่กลาง QR (สำหรับ Branded QR)
```

### 10.7.3 Incentive Programs สำหรับเพิ่ม Followers

**Offer ที่ได้ผลดี:**

```
ประเภท Incentive:
1. ส่วนลดทันที (Immediate Discount)
   "เพิ่มเพื่อน LINE รับส่วนลด 15%"
   ✅ ผลลัพธ์เร็ว แต่คน Block ง่ายหลังได้ส่วนลด

2. Exclusive Content
   "สมาชิก LINE OA ดูราคา Wholesale"
   ✅ Retention ดีกว่า เพราะมีเหตุผลให้อยู่ต่อ

3. Lucky Draw
   "เพิ่มเพื่อนรับสิทธิ์ลุ้นรางวัล"
   ✅ Viral ได้ดี แต่คุณภาพ Follower อาจต่ำ

4. Point System
   "เพิ่มเพื่อนรับ 100 Points"
   ✅ ดึงดูดคนที่มีโอกาสกลายเป็นลูกค้าจริง
```

---

## 10.8 Privacy และ PDPA Compliance

### 10.8.1 ข้อกำหนด PDPA สำหรับ LINE OA

**พ.ร.บ. คุ้มครองข้อมูลส่วนบุคคล (PDPA)** มีผลกับ LINE OA ดังนี้:

```
ข้อมูลที่ถือว่าเป็น "ข้อมูลส่วนบุคคล":
- LINE User ID
- ชื่อ-นามสกุล
- รูปโปรไฟล์
- ข้อความที่สนทนา
- พฤติกรรมการคลิก
- ที่อยู่จัดส่ง
- ข้อมูลการชำระเงิน
```

**ความยินยอมที่ต้องได้รับ:**

```
1. การรับข้อความ Marketing:
   ต้องแจ้งให้ผู้ใช้ทราบและได้รับความยินยอม
   "ฉันยินยอมรับข่าวสาร โปรโมชัน จาก [บริษัท]"

2. การเก็บข้อมูลการสนทนา:
   แจ้งในนโยบาย Privacy Policy
   บอก URL ใน Greeting Message

3. การนำข้อมูลไปใช้ประโยชน์:
   ระบุวัตถุประสงค์ให้ชัดเจน
```

### 10.8.2 Template Privacy Notice สำหรับ LINE OA

```
นโยบายความเป็นส่วนตัวของ [ชื่อบริษัท]

เมื่อคุณเพิ่ม LINE Official Account ของเราเป็นเพื่อน คุณยินยอมให้เราเก็บข้อมูล:
1. LINE User ID เพื่อการสื่อสาร
2. ประวัติการสนทนาเพื่อปรับปรุงบริการ
3. พฤติกรรมการใช้งานเพื่อนำเสนอเนื้อหาที่เหมาะสม

สิทธิของคุณ:
✅ ขอดูข้อมูลที่เราเก็บไว้
✅ ขอแก้ไขข้อมูล
✅ ขอลบข้อมูล (Block LINE OA หรือติดต่อ privacy@example.com)
✅ ถอนความยินยอมได้ทุกเวลา

ติดต่อเจ้าหน้าที่คุ้มครองข้อมูล:
📧 privacy@example.com
📞 02-XXX-XXXX
```

### 10.8.3 Data Deletion Request Handler

```javascript
// data-deletion.js
async function handleDataDeletionRequest(userId) {
  console.log(`Data deletion request for user: ${userId}`);
  
  // ลบข้อมูลทั้งหมดของผู้ใช้
  await Promise.all([
    db.users.destroy({ where: { lineUserId: userId } }),
    db.messages.destroy({ where: { userId } }),
    db.orders.update(
      { userId: 'DELETED' },
      { where: { userId } }
    ),
    db.audienceMembers.destroy({ where: { userId } }),
    db.followers.destroy({ where: { userId } })
  ]);
  
  // แจ้ง Admin
  await notifyAdmin(`Data deletion completed for user: ${userId}`);
  
  // ส่งยืนยันให้ผู้ใช้ (ถ้ายังไม่ Block)
  try {
    await client.pushMessage(userId, {
      type: 'text',
      text: 'เราได้ดำเนินการลบข้อมูลของคุณเรียบร้อยแล้วครับ 🙏\nหากมีคำถามเพิ่มเติม ติดต่อ privacy@example.com ครับ'
    });
  } catch (error) {
    // ผู้ใช้อาจ Block แล้ว ไม่สามารถส่งได้
    console.log('Cannot send confirmation, user may have blocked');
  }
}
```

---

## 10.9 Advanced Audience Tactics

### 10.9.1 Lookalike Audience

```
Lookalike Audience = ผู้คนที่คล้ายกับลูกค้าดีที่สุดของคุณ

วิธีสร้าง (ผ่าน LINE Ads):
1. Export User IDs ของลูกค้า Top 20% (ตาม RFM)
2. สร้าง Custom Audience ใน LINE Ads Manager
3. สร้าง Lookalike Audience ขนาด 1-10% ของ Source
4. ใช้ Lookalike Audience รัน LINE Ads
5. ผู้ที่คลิก Ad → Add Friend → ได้ Follower คุณภาพสูง
```

### 10.9.2 Sequential Messaging

```
ส่งข้อความเป็นลำดับตาม Customer Journey:

Day 0 (เพิ่งเพิ่มเพื่อน): Greeting + Welcome Offer
Day 3 (ยังไม่ซื้อ): Product Showcase
Day 7 (ยังไม่ซื้อ): Social Proof + Reviews
Day 14 (ยังไม่ซื้อ): Limited Time Offer
Day 30 (ซื้อแล้ว): Follow-up + Upsell
Day 60 (ซื้อแล้ว): Re-purchase Reminder
```

```javascript
// sequential-messaging.js
async function triggerSequence(userId, sequenceName) {
  const sequences = {
    new_follower: [
      { delay: 0, messageKey: 'welcome' },
      { delay: 3 * 24 * 60, messageKey: 'product_showcase' }, // 3 วัน
      { delay: 7 * 24 * 60, messageKey: 'social_proof' },     // 7 วัน
      { delay: 14 * 24 * 60, messageKey: 'limited_offer' }    // 14 วัน
    ],
    post_purchase: [
      { delay: 0, messageKey: 'thank_you' },
      { delay: 3 * 24 * 60, messageKey: 'how_to_use' },
      { delay: 30 * 24 * 60, messageKey: 'reorder_reminder' }
    ]
  };
  
  const sequence = sequences[sequenceName];
  if (!sequence) return;
  
  for (const step of sequence) {
    // Schedule การส่งข้อความ
    await scheduleMessage({
      userId,
      messageKey: step.messageKey,
      sendAt: new Date(Date.now() + step.delay * 60 * 1000)
    });
  }
}
```

---

## 10.10 แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: สร้าง Audience หลายประเภท (พื้นฐาน)
**เวลา: 60 นาที**

1. สร้าง Click Audience จาก Broadcast ที่ส่งไปแล้ว
2. สร้าง Upload Audience จากไฟล์ CSV (ใช้ User ID ทดสอบ)
3. สร้าง Chat Tag Audience จาก Tag ที่มีอยู่
4. ทดสอบ Narrowcast ไปยัง Audience แต่ละกลุ่ม
5. เปรียบเทียบ Open Rate ระหว่างกลุ่ม

### แบบฝึกหัดที่ 2: RFM Segmentation System (ระดับกลาง)
**เวลา: 3-4 ชั่วโมง**

สร้างระบบ RFM Segmentation:
1. ออกแบบ Database Schema สำหรับเก็บข้อมูลผู้ใช้ (users, orders, activities)
2. เขียนฟังก์ชัน `calculateRFMScore(userId)` ที่ดึงข้อมูลจาก DB
3. เขียน Cron Job อัปเดต Segment ทุกคืน
4. สร้าง API endpoint `GET /audience/segment/:segmentName` ที่ Return User IDs

### แบบฝึกหัดที่ 3: Follower Analytics Dashboard (ระดับสูง)
**เวลา: 5-6 ชั่วโมง**

สร้าง Web Dashboard (ใช้ Chart.js หรือ Recharts) ที่แสดง:
1. **Follower Growth** - กราฟเส้นแสดงจำนวน Followers 30 วันย้อนหลัง
2. **Block Rate Trend** - กราฟแท่งแสดง Block Rate รายสัปดาห์
3. **Segment Distribution** - Pie Chart แสดงสัดส่วน RFM Segments
4. **Broadcast Performance** - ตารางแสดง Open Rate, Click Rate ต่อ Segment
5. **Top Locations** - Bar Chart แสดง Top 10 จังหวัด

ข้อมูลทั้งหมดดึงจาก Express.js API ที่สร้างในแบบฝึกหัดก่อนหน้า

---

## 10.11 Final Project: LINE OA Analytics System

### โปรเจกต์ปิดท้าย: ระบบวิเคราะห์ LINE OA แบบสมบูรณ์

ประกอบด้วย:

```
System Architecture:

LINE OA ──── Webhook ────► Express.js API ──► PostgreSQL DB
                                │
                                ├── User Service (บันทึกผู้ใช้, RFM)
                                ├── Message Service (บันทึก Chat History)
                                ├── Broadcast Service (Track Performance)
                                ├── Segment Service (อัปเดต Segments)
                                └── Analytics Service (รายงาน)
                                         │
                            React Dashboard ◄─────────────┘
                            (Charts, Tables, Filters)
```

**Deliverables:**

1. **Webhook Server** - รับ Events จาก LINE และบันทึกลง DB
2. **User Management** - ระบบ Track Followers + RFM Scoring
3. **Broadcast Analytics** - วัด Open/Click Rate แบบ Real-time
4. **Dashboard** - แสดงข้อมูลแบบ Visual
5. **Export Tool** - ส่งออก Audience CSV สำหรับ Narrowcast

---

## สรุปบทที่ 10 และ Course ทั้งหมด

### สรุปบทที่ 10

ในบทนี้คุณได้เรียนรู้:

- ✅ ทำความเข้าใจ Followers, Friends และ Blocked Users
- ✅ การสร้าง Audience ทุกประเภท: Upload, Click, Impression, Video, Chat Tag
- ✅ กลยุทธ์ RFM Segmentation สำหรับ LINE Marketing
- ✅ Behavioral และ Lifecycle Segmentation
- ✅ การจัดการและป้องกัน Block Rate สูง
- ✅ Analytics Tools สำหรับเข้าใจ Audience
- ✅ Privacy และ PDPA Compliance

### สรุป Course LINE OA Development (Parts 1-10)

| Part | หัวข้อ | Key Takeaway |
|------|--------|-------------|
| 1 | Introduction | LINE OA Ecosystem |
| 2 | Account Setup | สร้างและตั้งค่า Account |
| 3 | LINE API Basics | Webhook, API Fundamentals |
| 4 | Messaging API | ส่ง/รับข้อความทุกประเภท |
| 5 | LIFF Development | Mini Apps บน LINE |
| 6 | **Broadcast** | ส่งข้อความหมู่แบบมืออาชีพ |
| 7 | **Auto-Reply** | ระบบตอบกลับอัตโนมัติ |
| 8 | **Rich Menu** | Navigation UI ที่ทรงพลัง |
| 9 | **Chat Features** | จัดการ Support อย่างมีระบบ |
| 10 | **Follower Management** | Segmentation และ Analytics |

### Next Steps

หลังจาก Complete Course นี้แล้ว แนะนำให้ศึกษาต่อใน:

1. **LINE Ads** - การทำโฆษณาเพื่อเพิ่ม Followers
2. **LINE Pay** - การรับชำระเงินผ่าน LINE
3. **LINE Points** - ระบบ Loyalty Points
4. **LINE Beacon** - Location-based Marketing
5. **Advanced LIFF** - สร้าง Mini App ซับซ้อน
6. **LINE Bot AI** - เชื่อมต่อ NLP และ Machine Learning

---

*อัปเดตล่าสุด: ตุลาคม 2024 | LINE Official Account Manager v2024*
*สร้างโดย LINE OA Developer Community Thailand*
