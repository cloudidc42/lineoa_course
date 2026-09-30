# ส่วนที่ 6: การส่ง Broadcast Messages

## บทนำ

Broadcast Messages คือหัวใจสำคัญของ LINE Official Account ที่ช่วยให้คุณสามารถส่งข้อความไปหาผู้ติดตามทั้งหมดหรือกลุ่มเป้าหมายที่เฉพาะเจาะจงได้ในครั้งเดียว ไม่ว่าจะเป็นโปรโมชัน ข่าวสาร หรือเนื้อหาที่มีประโยชน์

ในบทนี้คุณจะเรียนรู้:
- หลักการทำงานของ Broadcast
- วิธีสร้างและส่ง Broadcast ครั้งแรก
- ประเภทข้อความที่รองรับ
- การตั้งเวลาส่ง
- ตัวเลือกการกำหนดกลุ่มเป้าหมาย
- การวิเคราะห์ผลลัพธ์
- แนวปฏิบัติที่ดีที่สุดเพื่ออัตราการเปิดอ่านสูง

---

## 6.1 Broadcast คืออะไร?

### ความหมายและหลักการทำงาน

**Broadcast** คือการส่งข้อความจาก LINE Official Account ไปยังผู้ติดตาม (Followers) พร้อมกันในครั้งเดียว เปรียบได้กับการส่งอีเมลหาสมาชิกทั้งหมดในรายชื่อผู้รับข่าวสาร แต่มีข้อได้เปรียบที่สำคัญกว่ามาก เพราะ LINE มีอัตราการเปิดอ่าน (Open Rate) สูงกว่าอีเมลถึง 5-10 เท่า

```
ผู้ดูแล LINE OA
        |
        | ส่ง Broadcast
        v
  LINE Platform
        |
        | กระจายข้อความ
        v
  ┌─────────────────────────────────────────┐
  │  ผู้ติดตาม 1  ผู้ติดตาม 2  ผู้ติดตาม 3 ... │
  └─────────────────────────────────────────┘
```

### ความแตกต่างระหว่าง Broadcast และการส่งข้อความแบบอื่น

| ประเภทการส่ง | กลุ่มเป้าหมาย | ค่าใช้จ่าย | ข้อจำกัด |
|--------------|--------------|------------|----------|
| **Broadcast** | ผู้ติดตามทั้งหมด หรือกลุ่มที่กำหนด | คิดตามจำนวนข้อความที่ส่ง | ต้องใช้ Message Credits |
| **Narrowcast** | กลุ่มเป้าหมายเฉพาะตาม Audience | คิดตามจำนวน | ต้องการ Audience |
| **1-on-1 Chat** | ผู้ใช้คนเดียว | ฟรี (Reply) หรือคิดค่า Push | มีขีดจำกัดตาม Plan |
| **Auto Reply** | ผู้ที่ส่งข้อความมา | ไม่คิดค่า | ตอบสนองตามเงื่อนไข |

### ทำไมต้อง Broadcast?

Broadcast เป็นเครื่องมือที่ทรงพลังด้วยเหตุผลต่อไปนี้:

1. **อัตราการเปิดอ่านสูง**: เฉลี่ย 50-70% เทียบกับอีเมลที่ประมาณ 20%
2. **การแจ้งเตือนทันที**: ผู้ใช้ LINE มักเปิดแอปบ่อยครั้งตลอดวัน
3. **รองรับมีเดียหลายประเภท**: ข้อความ รูปภาพ วิดีโอ สติกเกอร์ และอื่นๆ
4. **วัดผลได้**: ดูได้ว่าใครเปิดอ่าน คลิก หรือบล็อก
5. **กำหนดเวลาได้**: วางแผนล่วงหน้าและตั้งเวลาส่งอัตโนมัติ

> **สถิติที่น่าสนใจ**: จากการศึกษาของ LINE ประเทศไทย พบว่า Broadcast ที่ส่งในช่วงเวลา 20:00-21:00 น. มีอัตราการเปิดอ่านสูงกว่าช่วงเวลาอื่นถึง 35%

---

## 6.2 การสร้าง Broadcast ครั้งแรก

### ขั้นตอนการเข้าถึง Broadcast Manager

1. เปิดเบราว์เซอร์และไปที่ [manager.line.biz](https://manager.line.biz)
2. ล็อกอินด้วยบัญชี LINE ของคุณ
3. เลือก Official Account ที่ต้องการจัดการ
4. ในเมนูซ้ายมือ คลิกที่ **"Messages"** หรือ **"ข้อความ"**
5. คลิกที่ **"Create new message"** หรือ **"สร้างข้อความใหม่"**

> **หมายเหตุ**: หน้าจอจะแสดงปุ่ม "Create new message" สีเขียวอยู่ด้านบนขวาของหน้า Message List โดยปกติจะมีไอคอนดินสอหรือบวก (+) อยู่ด้วย

### การสร้าง Broadcast ขั้นพื้นฐาน

**ขั้นตอนที่ 1: กำหนดชื่อข้อความ**
- ในช่อง "Message name" ใส่ชื่อที่ช่วยให้คุณจำได้ เช่น "โปรโมชัน_กันยายน_2024"
- ชื่อนี้จะไม่แสดงให้ผู้ใช้เห็น มีไว้เพื่อการจัดการภายในเท่านั้น

**ขั้นตอนที่ 2: เลือกประเภทข้อความ**
- คลิกที่ประเภทข้อความที่ต้องการ: Text, Image, Video, หรืออื่นๆ
- สามารถเพิ่มข้อความได้สูงสุด 5 บอลลูน (Balloon) ในการส่งครั้งเดียว

**ขั้นตอนที่ 3: สร้างเนื้อหา**
- ใส่เนื้อหาข้อความตามประเภทที่เลือก
- สามารถ Preview ได้ในตัวอย่างโทรศัพท์ทางด้านขวา

**ขั้นตอนที่ 4: ตั้งค่าการส่ง**
- เลือก "Send now" หรือ "Schedule"
- เลือกกลุ่มเป้าหมาย (All followers หรือ Audience)

**ขั้นตอนที่ 5: ยืนยันและส่ง**
- ตรวจสอบรายละเอียดทั้งหมด
- คลิก "Send" หรือ "Confirm"

### ตัวอย่าง Broadcast แรกสุด

สมมติว่าเราต้องการส่งข้อความต้อนรับผู้ติดตามใหม่ในเดือนนี้:

```
ชื่อข้อความ: Welcome_Message_Q4_2024

เนื้อหา:
สวัสดีครับ/ค่ะ สมาชิกทุกท่าน! 🎉

ขอบคุณที่เพิ่ม [ชื่อบัญชี] เป็นเพื่อนนะครับ/ค่ะ

เดือนนี้เรามีโปรโมชันพิเศษสำหรับสมาชิกใหม่:
✅ ส่วนลด 15% สำหรับการสั่งครั้งแรก
✅ ฟรีค่าจัดส่งทุกออเดอร์
✅ สิทธิ์เข้าร่วม Member Club

กดปุ่มด้านล่างเพื่อดูโปรโมชันทั้งหมด 👇
```

---

## 6.3 ประเภทข้อความสำหรับ Broadcast

### 6.3.1 Text Message (ข้อความธรรมดา)

ข้อความที่ง่ายและตรงไปตรงมาที่สุด รองรับข้อความได้สูงสุด 5,000 ตัวอักษร

**คุณสมบัติพิเศษ:**
- รองรับ Emoji ทั้งหมด
- สามารถใส่ URL ที่คลิกได้
- รองรับ Line Break
- สามารถใส่ตัวแปรชื่อผู้ใช้ได้ (ใน Premium Plan)

**ตัวอย่างการใช้งาน:**

```
🔥 Flash Sale วันนี้วันเดียว!

สินค้า Top Hit ลดราคาสูงสุด 50%
⏰ หมดเขต 23:59 น. วันนี้เท่านั้น!

🛒 ช้อปเลย: https://example.com/sale

📞 สอบถามเพิ่มเติม: 02-XXX-XXXX
```

> **เคล็ดลับ**: ใช้ Emoji เพื่อดึงความสนใจ แต่อย่าใช้มากเกินไป เพราะอาจทำให้ข้อความดูไม่เป็นมืออาชีพ

### 6.3.2 Image Message (รูปภาพ)

ส่งรูปภาพเดี่ยวพร้อมลิงก์ที่คลิกได้

**ข้อกำหนดรูปภาพ:**

| ประเภท | ขนาดแนะนำ | ขนาดสูงสุด | รูปแบบไฟล์ |
|--------|-----------|-----------|-----------|
| Square | 1024 x 1024 px | 10 MB | JPG, PNG |
| Rectangle (Wide) | 1024 x 512 px | 10 MB | JPG, PNG |
| Rectangle (Tall) | 512 x 1024 px | 10 MB | JPG, PNG |

**ตัวอย่างโครงสร้าง:**

```json
{
  "type": "image",
  "originalContentUrl": "https://example.com/images/banner.jpg",
  "previewImageUrl": "https://example.com/images/banner_preview.jpg"
}
```

### 6.3.3 Video Message (วิดีโอ)

ส่งวิดีโอพร้อม Preview Image

**ข้อกำหนดวิดีโอ:**
- รูปแบบ: MP4
- ขนาดสูงสุด: 200 MB
- ความยาวแนะนำ: ไม่เกิน 60 วินาที
- Preview Image: JPG หรือ PNG

**ตัวอย่าง:**

```json
{
  "type": "video",
  "originalContentUrl": "https://example.com/videos/promo.mp4",
  "previewImageUrl": "https://example.com/images/promo_thumbnail.jpg",
  "trackingId": "video_promo_oct2024"
}
```

### 6.3.4 Audio Message (เสียง)

เหมาะสำหรับพอดแคสต์หรือข้อความเสียงสั้นๆ

**ข้อกำหนด:**
- รูปแบบ: M4A
- ขนาดสูงสุด: 200 MB
- ต้องระบุระยะเวลา (milliseconds)

### 6.3.5 Sticker Message (สติกเกอร์)

ส่งสติกเกอร์ LINE เพื่อเพิ่มความสนุกและความน่ารัก

**วิธีหา Package ID และ Sticker ID:**
1. ไปที่ LINE Store (store.line.me)
2. เลือกสติกเกอร์ที่ต้องการ
3. ดู URL: `https://store.line.me/stickershop/product/[PACKAGE_ID]`
4. Sticker ID ดูได้จาก LINE Sticker Preview

```json
{
  "type": "sticker",
  "packageId": "1",
  "stickerId": "1"
}
```

> **ข้อควรระวัง**: ใช้ได้เฉพาะสติกเกอร์ที่ LINE อนุญาตสำหรับ Messaging API เท่านั้น ไม่ใช่ทุกสติกเกอร์ที่จะส่งผ่าน Broadcast ได้

### 6.3.6 Flex Message (ข้อความยืดหยุ่น)

Flex Message คือรูปแบบข้อความที่ทรงพลังและยืดหยุ่นที่สุด สามารถสร้าง Layout แบบ Card ที่สวยงามได้

**โครงสร้างพื้นฐาน:**

```json
{
  "type": "flex",
  "altText": "ข้อความสำรองสำหรับ Notification",
  "contents": {
    "type": "bubble",
    "hero": {
      "type": "image",
      "url": "https://example.com/image.jpg",
      "size": "full",
      "aspectRatio": "20:13",
      "aspectMode": "cover"
    },
    "body": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        {
          "type": "text",
          "text": "หัวข้อหลัก",
          "weight": "bold",
          "size": "xl"
        },
        {
          "type": "text",
          "text": "รายละเอียด",
          "size": "sm",
          "color": "#999999",
          "wrap": true
        }
      ]
    },
    "footer": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        {
          "type": "button",
          "style": "primary",
          "action": {
            "type": "uri",
            "label": "ดูเพิ่มเติม",
            "uri": "https://example.com"
          }
        }
      ]
    }
  }
}
```

**ตัวอย่าง Flex Message สำหรับโปรโมชัน:**

```json
{
  "type": "flex",
  "altText": "โปรโมชันพิเศษประจำเดือน",
  "contents": {
    "type": "bubble",
    "header": {
      "type": "box",
      "layout": "vertical",
      "backgroundColor": "#FF6B35",
      "contents": [
        {
          "type": "text",
          "text": "🔥 FLASH SALE",
          "color": "#FFFFFF",
          "weight": "bold",
          "size": "xxl",
          "align": "center"
        }
      ]
    },
    "body": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        {
          "type": "text",
          "text": "ลดสูงสุด 70%",
          "size": "xl",
          "weight": "bold",
          "align": "center",
          "margin": "md"
        },
        {
          "type": "separator",
          "margin": "md"
        },
        {
          "type": "box",
          "layout": "horizontal",
          "margin": "md",
          "contents": [
            {
              "type": "text",
              "text": "วันที่:",
              "size": "sm",
              "color": "#555555",
              "flex": 0
            },
            {
              "type": "text",
              "text": "15-17 ตุลาคม 2024",
              "size": "sm",
              "color": "#111111",
              "align": "end"
            }
          ]
        }
      ]
    },
    "footer": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        {
          "type": "button",
          "style": "primary",
          "color": "#FF6B35",
          "action": {
            "type": "uri",
            "label": "ช้อปเลย!",
            "uri": "https://example.com/sale"
          }
        }
      ]
    }
  }
}
```

### 6.3.7 Rich Message (ข้อความ Rich)

Rich Message คือรูปภาพที่มี Action Areas สามารถแบ่งพื้นที่รูปภาพเป็นส่วนๆ ที่คลิกได้

**ขนาดที่แนะนำ:**
- 1040 x 1040 px (Square)
- 2080 x 1040 px (Rectangle)

**ตัวอย่างการตั้งค่า:**
1. อัปโหลดรูปภาพ
2. กำหนด Tap Areas (พื้นที่ที่คลิกได้)
3. ตั้ง Action สำหรับแต่ละพื้นที่ (URL, Text, หรือ Postback)

### 6.3.8 Card-type Message (Image Carousel)

ส่งหลายรูปในรูปแบบ Carousel ที่เลื่อนซ้าย-ขวาได้

```json
{
  "type": "template",
  "altText": "Image Carousel",
  "template": {
    "type": "image_carousel",
    "columns": [
      {
        "imageUrl": "https://example.com/image1.jpg",
        "action": {
          "type": "uri",
          "label": "action",
          "uri": "https://example.com/1"
        }
      },
      {
        "imageUrl": "https://example.com/image2.jpg",
        "action": {
          "type": "uri",
          "label": "action",
          "uri": "https://example.com/2"
        }
      }
    ]
  }
}
```

---

## 6.4 การตั้งเวลาส่ง Broadcast

### ประเภทการตั้งเวลา

**1. Send Now (ส่งทันที)**
- ส่งข้อความทันทีหลังจากกด Confirm
- เหมาะสำหรับข้อมูลเร่งด่วนหรือ Flash Sale

**2. Schedule (กำหนดเวลา)**
- ตั้งวันและเวลาล่วงหน้า
- สามารถแก้ไขหรือยกเลิกได้ก่อนถึงเวลา
- เหมาะสำหรับการวางแผนคอนเทนต์

### ขั้นตอนการตั้งเวลา

1. เมื่อสร้างข้อความเสร็จแล้ว เลื่อนลงมาหาส่วน "Delivery settings"
2. เลือก **"Schedule"**
3. คลิกที่ช่อง Date/Time เพื่อเลือกวันที่และเวลา
4. ตรวจสอบ Timezone ให้ถูกต้อง (ควรเป็น Asia/Bangkok สำหรับไทย)
5. คลิก **"Confirm"**

> **หมายเหตุเกี่ยวกับ Timezone**: LINE OA Manager ใช้ Timezone ตามการตั้งค่าบัญชี ตรวจสอบให้แน่ใจว่าตั้งค่าเป็น UTC+7 (Bangkok) เพื่อป้องกันการส่งผิดเวลา

### การจัดการ Scheduled Messages

- **ดู Schedule ทั้งหมด**: ไปที่ Messages > Scheduled
- **แก้ไข**: คลิกที่ข้อความที่ Schedule ไว้ > Edit
- **ยกเลิก**: คลิกที่ข้อความ > Delete หรือ Cancel
- **ระยะเวลาขั้นต่ำ**: ต้องตั้งเวลาล่วงหน้าอย่างน้อย 10 นาที

### ตารางเวลาที่แนะนำสำหรับแต่ละประเภทธุรกิจ

| ประเภทธุรกิจ | วันที่แนะนำ | เวลาที่แนะนำ | เหตุผล |
|-------------|------------|-------------|--------|
| ร้านอาหาร | อังคาร-พฤหัส | 11:00, 17:00 | ก่อนมื้อกลางวัน/เย็น |
| แฟชัน/เสื้อผ้า | พฤหัส-ศุกร์ | 20:00-21:00 | ช้อปปิ้งก่อนวันหยุด |
| สุขภาพ/ความงาม | อาทิตย์ | 08:00-09:00 | เตรียมตัวสัปดาห์ใหม่ |
| การศึกษา | จันทร์ | 09:00-10:00 | เริ่มต้นสัปดาห์ |
| อสังหาริมทรัพย์ | เสาร์-อาทิตย์ | 10:00-11:00 | วันหยุด |

---

## 6.5 ตัวเลือกการกำหนดกลุ่มเป้าหมาย

### 6.5.1 All Followers (ผู้ติดตามทั้งหมด)

ส่งไปหาผู้ติดตามทุกคนที่ยังไม่ได้บล็อก

**ข้อดี:**
- ง่ายที่สุด ไม่ต้องตั้งค่าเพิ่มเติม
- เข้าถึงผู้ใช้ได้มากที่สุด

**ข้อเสีย:**
- อาจมี Message Cost สูงกว่า
- อาจไม่เกี่ยวข้องกับผู้ใช้บางกลุ่ม ทำให้ Unblock Rate สูงขึ้น

### 6.5.2 Targeted/Narrowcast (กำหนดกลุ่มเฉพาะ)

ส่งไปยัง Audience ที่กำหนดไว้ล่วงหน้า

**ประเภท Audience ที่รองรับ:**

```
1. Demographic Targeting
   - เพศ (ชาย/หญิง)
   - อายุ (หลายช่วง)
   - ระบบปฏิบัติการ (iOS/Android)
   - ภูมิภาค

2. Behavioral Targeting  
   - Audience จาก Chat tag
   - ผู้ที่คลิก URL ใน Message เดิม
   - ผู้ที่ดูวิดีโอครบ
   - ผู้ที่เพิ่มเพื่อนจาก Channel ใดช่องใด

3. Custom Audience
   - Upload จากไฟล์ (UID list)
   - ผู้ที่มีคุณสมบัติตาม Webhook data
```

### 6.5.3 การสร้าง Audience สำหรับ Narrowcast

**สร้าง Audience จากไฟล์ CSV:**

1. ไปที่ **Audience** ในเมนูหลัก
2. คลิก **"Create new audience"**
3. เลือก **"Upload audience"**
4. เตรียมไฟล์ CSV:

```csv
userId
Uxxxxxxxxxxx1
Uxxxxxxxxxxx2
Uxxxxxxxxxxx3
```

5. อัปโหลดไฟล์ (รองรับสูงสุด 1,500,000 บรรทัด)
6. ตั้งชื่อ Audience และบันทึก

**สร้าง Audience จาก Chat Tag:**

```
1. ตั้ง Chat Tag ให้ลูกค้า (ดูรายละเอียดใน Part 9)
2. ไปที่ Audience > Create new audience
3. เลือก "Chat tag audience"
4. เลือก Tag ที่ต้องการ
5. กำหนดเงื่อนไข (มี Tag นี้ / ไม่มี Tag นี้)
```

> **เคล็ดลับ**: Narrowcast ต้องการ Audience ที่มีสมาชิกอย่างน้อย 50 คนขึ้นไป มิฉะนั้นระบบจะไม่อนุญาตให้ส่ง

### 6.5.4 Attribute Targeting

กำหนดเป้าหมายตามคุณสมบัติของผู้ใช้ (เฉพาะบัญชีที่มี Premium features):

```
ตัวกรองที่ใช้ได้:
- เพศ: ชาย / หญิง / ไม่ระบุ
- อายุ: <18, 18-24, 25-34, 35-44, 45-54, 55+
- OS: iOS / Android
- ภูมิภาค: ตามจังหวัดในไทย
- เพิ่มเพื่อน: ช่วงเวลาที่เพิ่ม
```

**ตัวอย่างการตั้งค่า Narrowcast สำหรับโปรโมชัน:**

```
เป้าหมาย: ผู้หญิง อายุ 25-34 ปี ในกรุงเทพฯ
ที่เคยคลิกลิงก์ในข้อความก่อนหน้า

Audience ที่ตั้งค่า:
- เพศ: หญิง
- อายุ: 25-34
- ภูมิภาค: กรุงเทพมหานคร
- Retargeting: เคยคลิก URL ใน Message ID: MSG001
```

---

## 6.6 Analytics สำหรับ Broadcast

### 6.6.1 เมตริกหลักที่ต้องติดตาม

| เมตริก | ความหมาย | เป้าหมายที่ดี |
|--------|----------|--------------|
| **Delivered** | จำนวนที่ส่งสำเร็จ | >95% |
| **Opened** | จำนวนที่เปิดอ่าน | >50% |
| **Clicked** | จำนวนที่คลิกลิงก์ | >5% |
| **Blocked** | จำนวนที่ Unblock | <2% |
| **Open Rate** | % ที่เปิดอ่าน | >40% |
| **Click Rate** | % ที่คลิก | >3% |

### 6.6.2 วิธีดู Analytics

1. ไปที่ **Messages** > เลือกข้อความที่ต้องการ
2. คลิก **"Statistics"** หรือ **"สถิติ"**
3. ดาวน์โหลดรายงานเป็น CSV ได้โดยคลิก **"Export"**

**ข้อมูลที่จะเห็น:**
- กราฟแสดง Open Rate ตามเวลา
- จำนวน Delivered, Opened, Clicked แบบ Real-time
- เปรียบเทียบกับ Broadcast ก่อนหน้า
- Blocked count ในช่วง 7 วันหลังส่ง

### 6.6.3 การใช้ UTM Parameters สำหรับการติดตามขั้นสูง

เพิ่ม UTM Parameters ใน URL เพื่อติดตามใน Google Analytics:

```
URL พื้นฐาน:
https://example.com/product

URL พร้อม UTM:
https://example.com/product?utm_source=line_oa&utm_medium=broadcast&utm_campaign=oct_sale_2024&utm_content=button_shop_now
```

**ตัวอย่าง UTM ที่แนะนำ:**

| Parameter | ค่าแนะนำ | ตัวอย่าง |
|-----------|---------|---------|
| utm_source | line_oa | line_oa |
| utm_medium | broadcast | broadcast |
| utm_campaign | ชื่อแคมเปญ | oct_sale_2024 |
| utm_content | ตำแหน่ง Button | hero_banner, cta_button |
| utm_term | ประเภท Audience | all, female_25_34 |

### 6.6.4 การสร้าง Dashboard Analytics ด้วย Google Looker Studio

เชื่อมข้อมูล LINE OA กับ Google Analytics เพื่อสร้าง Dashboard:

```
ขั้นตอน:
1. ตั้งค่า Google Analytics 4 บนเว็บไซต์
2. ใส่ UTM ใน URL ทุกลิงก์ที่ส่งผ่าน LINE OA
3. สร้าง Custom Dimension ใน GA4:
   - line_message_id
   - broadcast_campaign
   - audience_segment
4. เชื่อมต่อ GA4 กับ Looker Studio
5. สร้าง Report ที่แสดง:
   - Revenue จาก LINE OA
   - Sessions ที่มาจาก LINE
   - Conversion Rate
```

---

## 6.7 แนวปฏิบัติที่ดีที่สุดเพื่ออัตราการเปิดอ่านสูง

### 6.7.1 หลักการเขียนข้อความ

**DO (ควรทำ):**
1. เขียนประโยคแรกให้ดึงดูดใจ - ผู้ใช้เห็นแค่บรรทัดแรกใน Notification
2. มีความชัดเจน - บอกว่าต้องการให้ทำอะไร (Call to Action)
3. สร้าง Urgency - กำหนดเวลาหมดเขต
4. Personalization - ใช้ชื่อผู้ใช้ถ้าเป็นไปได้
5. ข้อความกระชับ - ไม่เกิน 160 ตัวอักษรสำหรับบรรทัดแรก

**DON'T (ไม่ควรทำ):**
1. ส่งบ่อยเกินไป - ไม่เกิน 4-5 ครั้งต่อเดือน
2. ส่งข้อความที่ไม่มีคุณค่า - แจ้งข่าวสาระน้อย
3. ใช้ภาษาเชิงขาย aggressive เกินไป
4. ส่งตอนดึก - หลัง 22:00 น.
5. ไม่มี Clear CTA - ผู้ใช้ไม่รู้ต้องทำอะไร

### 6.7.2 โครงสร้างข้อความที่มีประสิทธิภาพ

```
โครงสร้าง AIDA สำหรับ LINE OA:

A (Attention) - ดึงความสนใจใน 1-2 บรรทัดแรก
  "🔥 Flash Sale 24 ชั่วโมง! ลดกระหน่ำ 70%"

I (Interest) - สร้างความสนใจ
  "สินค้า Bestseller ราคาพิเศษ เฉพาะ LINE OA เท่านั้น"

D (Desire) - กระตุ้นความต้องการ
  "📦 ยังเหลืออีก 20 ชิ้นสุดท้าย!"

A (Action) - เรียกให้ดำเนินการ
  "กดช้อปเดี๋ยวนี้ 👇 [ลิงก์]"
```

### 6.7.3 การทดสอบ A/B สำหรับ Broadcast

**สิ่งที่ควรทดสอบ:**

1. **เวลาส่ง**: ทดสอบ 20:00 vs 12:00 vs 08:00
2. **ประเภทข้อความ**: Text vs Image vs Flex Message
3. **Call to Action**: "ช้อปเลย" vs "ดูสินค้า" vs "รับส่วนลด"
4. **ความยาวข้อความ**: สั้น vs ยาว
5. **ใช้ Emoji หรือไม่**: มี Emoji vs ไม่มี

**วิธีทำ A/B Test:**
1. แบ่ง Audience เป็น 2 กลุ่มเท่าๆ กัน
2. ส่งข้อความเวอร์ชัน A ให้กลุ่ม 1
3. ส่งข้อความเวอร์ชัน B ให้กลุ่ม 2
4. รอ 24-48 ชั่วโมง
5. เปรียบเทียบ Open Rate และ Click Rate
6. ใช้เวอร์ชันที่ดีกว่าสำหรับการส่งครั้งต่อไป

### 6.7.4 Content Calendar สำหรับ LINE OA

```
ตัวอย่าง Content Calendar รายเดือน:

สัปดาห์ที่ 1 (จันทร์): ข่าวสาร/Tips ที่มีประโยชน์
สัปดาห์ที่ 2 (พุธ): โปรโมชันกลางเดือน  
สัปดาห์ที่ 3 (ศุกร์): Flash Sale สั้นๆ
สัปดาห์ที่ 4 (อาทิตย์): แชร์ Content น่าสนใจ

ช่วงพิเศษ:
- ต้นเดือน: เงินเดือนออก = ส่งโปรโมชัน
- วันหยุดยาว: ส่งล่วงหน้า 3 วัน
- วันสำคัญ (วาเลนไทน์ สงกรานต์ ฯลฯ): เตรียมล่วงหน้า 2 สัปดาห์
```

### 6.7.5 กฎ 80/20 สำหรับ Content

```
80% Content มีคุณค่า:
  - Tips & Tricks ที่เป็นประโยชน์
  - ข้อมูลสินค้า/บริการที่เป็นประโยชน์
  - ข่าวสารที่เกี่ยวข้อง
  - Entertainment content

20% Promotional:
  - โปรโมชัน ส่วนลด
  - Flash Sale
  - New Arrival
```

### 6.7.6 การจัดการ Compliance

> **ข้อควรระวัง (IMPORTANT)**: ตรวจสอบว่าการส่ง Broadcast ของคุณเป็นไปตามกฎระเบียบต่อไปนี้:
> - PDPA (พ.ร.บ. คุ้มครองข้อมูลส่วนบุคคล): ต้องได้รับความยินยอมก่อนส่งข้อความการตลาด
> - LINE Terms of Service: ห้ามส่งสแปม ห้ามโฆษณาสินค้าผิดกฎหมาย
> - ต้องมีช่องทางให้ Unsubscribe ได้ง่าย

---

## 6.8 การส่ง Broadcast ผ่าน API

สำหรับนักพัฒนาที่ต้องการส่ง Broadcast ผ่านโค้ด:

### ตั้งค่า Environment

```bash
# .env
LINE_CHANNEL_ACCESS_TOKEN=your_channel_access_token
```

### ตัวอย่างโค้ด Node.js

```javascript
const axios = require('axios');

const channelAccessToken = process.env.LINE_CHANNEL_ACCESS_TOKEN;

async function sendBroadcast(message) {
  try {
    const response = await axios.post(
      'https://api.line.me/v2/bot/message/broadcast',
      {
        messages: [
          {
            type: 'text',
            text: message
          }
        ]
      },
      {
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${channelAccessToken}`
        }
      }
    );
    
    console.log('Broadcast ส่งสำเร็จ:', response.data);
    return response.data;
  } catch (error) {
    console.error('เกิดข้อผิดพลาด:', error.response?.data || error.message);
    throw error;
  }
}

// ตัวอย่างการใช้งาน
sendBroadcast('สวัสดีครับ! นี่คือข้อความ Broadcast ทดสอบ 🎉');
```

### ตัวอย่างโค้ด Python

```python
import requests
import os
from datetime import datetime

CHANNEL_ACCESS_TOKEN = os.environ.get('LINE_CHANNEL_ACCESS_TOKEN')
BASE_URL = 'https://api.line.me/v2/bot'

def send_broadcast(messages: list) -> dict:
    """
    ส่ง Broadcast ไปยังผู้ติดตามทั้งหมด
    
    Args:
        messages: รายการข้อความ (สูงสุด 5 รายการ)
    
    Returns:
        Response data จาก LINE API
    """
    url = f'{BASE_URL}/message/broadcast'
    
    headers = {
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {CHANNEL_ACCESS_TOKEN}'
    }
    
    payload = {
        'messages': messages
    }
    
    response = requests.post(url, json=payload, headers=headers)
    response.raise_for_status()
    
    return response.json()


def send_text_broadcast(text: str) -> dict:
    """ส่ง Text Broadcast"""
    messages = [{'type': 'text', 'text': text}]
    return send_broadcast(messages)


def send_flex_broadcast(flex_content: dict, alt_text: str) -> dict:
    """ส่ง Flex Message Broadcast"""
    messages = [
        {
            'type': 'flex',
            'altText': alt_text,
            'contents': flex_content
        }
    ]
    return send_broadcast(messages)


# ตัวอย่างการใช้งาน
if __name__ == '__main__':
    # ส่ง Text broadcast
    result = send_text_broadcast(
        f"สวัสดีครับ! อัปเดตล่าสุด: {datetime.now().strftime('%d/%m/%Y')}"
    )
    print(f"ส่งสำเร็จ: {result}")
```

### Narrowcast ผ่าน API

```javascript
async function sendNarrowcast(messages, audienceId) {
  const response = await axios.post(
    'https://api.line.me/v2/bot/message/narrowcast',
    {
      messages: messages,
      recipient: {
        type: 'audience',
        audienceGroupId: audienceId
      },
      notificationDisabled: false
    },
    {
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${channelAccessToken}`
      }
    }
  );
  
  return response.data;
}
```

### ตรวจสอบ Quota การส่ง

```javascript
async function checkMessageQuota() {
  const response = await axios.get(
    'https://api.line.me/v2/bot/message/quota',
    {
      headers: {
        'Authorization': `Bearer ${channelAccessToken}`
      }
    }
  );
  
  const { type, value } = response.data;
  console.log(`Quota ประเภท: ${type}`);
  console.log(`จำนวน Free: ${value}`);
  
  // ดู Quota ที่ใช้ไปแล้วในเดือนนี้
  const consumptionResponse = await axios.get(
    'https://api.line.me/v2/bot/message/quota/consumption',
    {
      headers: {
        'Authorization': `Bearer ${channelAccessToken}`
      }
    }
  );
  
  console.log(`ใช้ไปแล้ว: ${consumptionResponse.data.totalUsage} ข้อความ`);
}
```

---

## 6.9 การแก้ไขปัญหาที่พบบ่อย

### ปัญหา: Broadcast ไม่ส่ง

**สาเหตุที่เป็นไปได้:**

```
1. Quota หมด
   - แก้ไข: ตรวจสอบ Message Quota ในหน้า Settings
   - แก้ไข: Upgrade Plan หากต้องการส่งเพิ่ม

2. Channel Access Token ไม่ถูกต้อง (กรณีใช้ API)
   - แก้ไข: ตรวจสอบ Token ใน LINE Developers Console
   - แก้ไข: สร้าง Token ใหม่หากหมดอายุ

3. Audience มีสมาชิกน้อยกว่า 50 คน
   - แก้ไข: ใช้ All Followers แทน หรือเพิ่มสมาชิก Audience

4. ข้อความขัดต่อ LINE Policy
   - แก้ไข: ตรวจสอบ Content Guidelines
```

### ปัญหา: Open Rate ต่ำมาก

```
สาเหตุและแนวทางแก้ไข:

1. เวลาส่งไม่เหมาะสม
   → ทดสอบเวลาที่ต่างกัน

2. ข้อความบรรทัดแรกไม่น่าสนใจ
   → ปรับปรุงหัวเรื่อง (เหมือน Email Subject Line)

3. ส่งบ่อยเกินไป ผู้ใช้เบื่อ
   → ลดความถี่และเพิ่มคุณค่าของ Content

4. กลุ่มเป้าหมายไม่ตรง
   → ใช้ Narrowcast เพื่อกำหนดกลุ่มที่เหมาะสมกว่า
```

---

## 6.10 แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: สร้าง Broadcast แรก (พื้นฐาน)
**เวลา: 30 นาที**

ทำตามขั้นตอน:
1. เข้า LINE OA Manager
2. สร้าง Text Broadcast สำหรับโปรโมชันสมมติ
3. ใส่ Emoji อย่างน้อย 3 ตัว
4. เพิ่มลิงก์ (ใช้ URL ของเว็บไซต์ใดก็ได้)
5. ส่งเป็น Test Message ไปหาตัวเอง (ไม่ต้องส่ง Broadcast จริง)

### แบบฝึกหัดที่ 2: สร้าง Flex Message (ระดับกลาง)
**เวลา: 60 นาที**

สร้าง Flex Message สำหรับโปรโมชัน Flash Sale โดยมี:
- Header สีแดงพร้อมหัวข้อ
- รูปภาพสินค้า (ใช้รูปจาก Lorem Picsum หรือ Unsplash)
- ราคาเดิมและราคาที่ลด
- วันหมดเขต
- ปุ่ม "ช้อปเลย" ที่ Link ไปยัง URL

**ทดสอบผ่าน Flex Message Simulator:** [https://developers.line.biz/flex-simulator/](https://developers.line.biz/flex-simulator/)

### แบบฝึกหัดที่ 3: API Broadcast (ระดับสูง)
**เวลา: 90 นาที**

1. สร้างโปรเจกต์ Node.js ใหม่
2. ติดตั้ง `axios` และ `dotenv`
3. เขียนฟังก์ชัน `scheduleBroadcast(message, delayMinutes)` ที่ส่งข้อความหลังจากเวลาที่กำหนด
4. เพิ่มการตรวจสอบ Quota ก่อนส่งทุกครั้ง
5. บันทึก Log การส่งแต่ละครั้งลงในไฟล์ `broadcast-log.json`

### แบบฝึกหัดที่ 4: วิเคราะห์ผลลัพธ์ (ทุกระดับ)
**เวลา: 45 นาที**

หลังจากส่ง Broadcast จริง 2-3 ครั้ง:
1. บันทึกข้อมูล Open Rate, Click Rate ในตาราง Excel
2. วิเคราะห์ว่าเวลาใดที่ Open Rate สูงที่สุด
3. เขียน Report 1 หน้า สรุปสิ่งที่เรียนรู้และแผนปรับปรุง

---

## สรุปบทที่ 6

ในบทนี้คุณได้เรียนรู้:

- ✅ หลักการทำงานของ Broadcast และความแตกต่างจากการส่งข้อความรูปแบบอื่น
- ✅ ขั้นตอนการสร้าง Broadcast ทีละขั้น
- ✅ ประเภทข้อความทั้งหมดที่รองรับ รวมถึง Flex Message ขั้นสูง
- ✅ การตั้งเวลาส่งและ Content Calendar
- ✅ การกำหนดกลุ่มเป้าหมายด้วย Narrowcast และ Audience
- ✅ การวัดผลและวิเคราะห์ด้วย Analytics
- ✅ แนวปฏิบัติที่ดีที่สุดเพื่อ Open Rate สูง
- ✅ การส่ง Broadcast ผ่าน API

**บทถัดไป**: Part 7 จะครอบคลุม Auto-Reply Setup ที่ช่วยให้ระบบตอบสนองอัตโนมัติต่อข้อความของผู้ใช้ได้อย่างชาญฉลาด

---

*อัปเดตล่าสุด: ตุลาคม 2024 | LINE Official Account Manager v2024*
