# ส่วนที่ 5: แผนการส่งข้อความและราคา LINE Official Account

---

## สารบัญ

1. [ทำความเข้าใจระบบ Message Quota](#1-ทำความเข้าใจระบบ-message-quota)
2. [Free Plan รายละเอียดและข้อจำกัด](#2-free-plan-รายละเอียดและข้อจำกัด)
3. [แผนชำระเงิน (Paid Plans)](#3-แผนชำระเงิน-paid-plans)
4. [ตารางเปรียบเทียบทุกแผน](#4-ตารางเปรียบเทียบทุกแผน)
5. [วิธีคำนวณ Quota ที่ต้องการ](#5-วิธีคำนวณ-quota-ที่ต้องการ)
6. [ราคาต่อข้อความและการเปรียบเทียบ](#6-ราคาต่อข้อความและการเปรียบเทียบ)
7. [การอัปเกรดและดาวน์เกรดแผน](#7-การอัปเกรดและดาวน์เกรดแผน)
8. [ช่องทางการชำระเงิน](#8-ช่องทางการชำระเงิน)
9. [เคล็ดลับประหยัด Message Quota](#9-เคล็ดลับประหยัด-message-quota)
10. [กลยุทธ์วางแผน Budget สำหรับ LINE OA](#10-กลยุทธ์วางแผน-budget-สำหรับ-line-oa)
11. [ROI Calculator สำหรับ LINE OA](#11-roi-calculator-สำหรับ-line-oa)
12. [แบบฝึกหัดท้ายบท](#12-แบบฝึกหัดท้ายบท)

---

## 1. ทำความเข้าใจระบบ Message Quota

### 1.1 Message Quota คืออะไร?

Message Quota คือจำนวนข้อความสูงสุดที่คุณสามารถส่งได้ในแต่ละเดือน โดยนับเฉพาะข้อความที่ส่งจาก LINE OA ไปยัง Followers เท่านั้น

```
การนับ Message Quota:
════════════════════════════════════════════════════

✓ นับ (ใช้ Quota):
  • Broadcast Message (ส่งทุกคน)
  • Multicast Message (ส่งกลุ่ม)
  • Narrowcast Message (ส่งเฉพาะกลุ่ม)
  • Reply Message ผ่าน Messaging API (บางกรณี)
  • Push Message ผ่าน Messaging API

✗ ไม่นับ (ฟรีไม่จำกัด):
  • Reply Message ใน Chat (ตอบกลับลูกค้า)
  • Auto Reply เมื่อลูกค้าส่งมา
  • Greeting Message เมื่อ Add Friend
  • LINE VOOM Posts
  • Chat ปกติ (1-on-1)

════════════════════════════════════════════════════
```

### 1.2 วิธีนับ Quota แต่ละข้อความ

```
การนับ Quota:

1 Broadcast → ส่ง 1,000 คน = นับ 1,000 Quota

ตัวอย่าง:
Followers: 1,000 คน
Broadcast 1 ข้อความ = ใช้ 1,000 Quota

Followers: 1,000 คน
Broadcast 5 ข้อความ/เดือน = ใช้ 5,000 Quota

ดังนั้น Free Plan (500 Quota):
→ ถ้ามี Followers 500 คน = ส่งได้ 1 ข้อความ/เดือน
→ ถ้ามี Followers 100 คน = ส่งได้ 5 ข้อความ/เดือน
→ ถ้ามี Followers 50 คน = ส่งได้ 10 ข้อความ/เดือน
```

### 1.3 ฟีเจอร์ที่ฟรีไม่จำกัดในทุกแผน

> **💡 สำคัญ:** ฟีเจอร์เหล่านี้ฟรีทุกแผน ไม่ว่าจะใช้แผนไหนก็ตาม

| ฟีเจอร์ | ฟรีหรือไม่ | หมายเหตุ |
|--------|-----------|---------|
| Chat (ตอบกลับ) | ✓ ฟรีไม่จำกัด | Reply ใน Chat |
| Auto Reply | ✓ ฟรี | ตอบตาม Keyword |
| Greeting Message | ✓ ฟรี | เมื่อ Add Friend |
| Rich Menu | ✓ ฟรี | เมนูด้านล่าง Chat |
| LINE VOOM Post | ✓ ฟรีไม่จำกัด | โพสต์ Feed |
| Coupon | ✓ ฟรี (สร้าง) | นับ Quota เมื่อ Broadcast |
| Point Card | ✓ ฟรี | ระบบสะสมแต้ม |
| Analytics | ✓ ฟรี | ดูสถิติ |

---

## 2. Free Plan รายละเอียดและข้อจำกัด

### 2.1 Free Plan Overview

```
FREE PLAN สรุป:
════════════════════════════════════════

📊 Message Quota: 500 ข้อความ/เดือน
💰 ราคา: ฟรี
🔄 Rollover: ไม่มี (ไม่ Carry-over ไปเดือนถัดไป)
📅 Reset: ทุกวันที่ 1 ของเดือน

สิ่งที่ทำได้:
✓ สร้างและจัดการบัญชี LINE OA
✓ Chat กับลูกค้าไม่จำกัด
✓ Auto Reply
✓ Greeting Message
✓ Rich Menu (1 ชุด)
✓ LINE VOOM Post ไม่จำกัด
✓ Coupon และ Point Card
✓ Basic Analytics

ข้อจำกัด:
✗ Broadcast ได้จำกัด (500 Quota/เดือน)
✗ ไม่มี Narrowcast (Targeted Messaging)
✗ Rich Menu จำกัด Feature

════════════════════════════════════════
```

### 2.2 Free Plan เหมาะกับใคร?

```
Free Plan เหมาะกับ:

1. ธุรกิจเพิ่งเริ่มต้น
   → มี Follower น้อยกว่า 500 คน
   → ยังทดสอบ Platform

2. ธุรกิจที่เน้น Chat มากกว่า Broadcast
   → ร้านที่ลูกค้า Chat เข้ามาเอง
   → บริการที่ต้องคุยรายบุคคล

3. ธุรกิจที่ใช้ LINE VOOM เป็นหลัก
   → Content Creator
   → ร้านที่โพสต์รูปสินค้า

4. ทดสอบก่อนตัดสินใจซื้อ Plan
   → ทดลองระบบก่อน 1-3 เดือน

Free Plan ไม่เหมาะกับ:
✗ ธุรกิจที่มี Followers มากกว่า 500 คน
✗ ต้องการส่ง Campaign สม่ำเสมอ
✗ ต้องการ Targeted Messaging
```

### 2.3 การใช้ Free Plan อย่างคุ้มค่า

```
เคล็ดลับ Free Plan:

1. ใช้ LINE VOOM แทน Broadcast:
   → VOOM ฟรีไม่จำกัด แต่ Open Rate ต่ำกว่า
   → เหมาะสำหรับ Content ที่ไม่ด่วน

2. กระตุ้นให้ลูกค้า Chat เข้ามาเอง:
   → โพสต์ใน VOOM ให้ Reply/DM
   → ตอบกลับ Chat ฟรีไม่จำกัด

3. ประหยัด Quota สำหรับข้อความสำคัญ:
   → ใช้ 500 Quota กับ Promotion ใหญ่สุดของเดือน
   → อย่าส่ง Broadcast ทุกวัน

4. Segment Followers:
   → ถ้ามี 1,000 Followers, Broadcast แค่ 500 คน
   → เลือกกลุ่มที่ Active ที่สุด
```

---

## 3. แผนชำระเงิน (Paid Plans)

### 3.1 Light Plan

```
LIGHT PLAN
════════════════════════════════════════

💰 ราคา: 599 บาท/เดือน (ประมาณ)
📊 Message Quota: 5,000 ข้อความ/เดือน
➕ Additional Messages: 0.22 บาท/ข้อความ
📅 Contract: รายเดือน (ยกเลิกได้)

เพิ่มเติมจาก Free:
✓ Quota เพิ่มเป็น 5,000 ข้อความ
✓ Narrowcast Message (Targeted)
✓ Multiple Rich Menu
✓ Advanced Analytics

เหมาะกับ:
→ ธุรกิจ SME ที่เริ่มมี Followers
→ Followers 500-3,000 คน
→ ส่ง Broadcast 1-2 ครั้ง/สัปดาห์

════════════════════════════════════════
```

### 3.2 Standard Plan

```
STANDARD PLAN
════════════════════════════════════════

💰 ราคา: 1,500 บาท/เดือน (ประมาณ)
📊 Message Quota: 15,000 ข้อความ/เดือน
➕ Additional Messages: 0.19 บาท/ข้อความ
📅 Contract: รายเดือน (ยกเลิกได้)

เพิ่มเติมจาก Light:
✓ Quota เพิ่มเป็น 15,000 ข้อความ
✓ ราคา Additional Message ถูกลง
✓ Priority Support
✓ Advanced Audience Segmentation

เหมาะกับ:
→ ธุรกิจขนาดกลาง
→ Followers 3,000-10,000 คน
→ มี Campaign สม่ำเสมอ

════════════════════════════════════════
```

### 3.3 Premium Plan

```
PREMIUM PLAN
════════════════════════════════════════

💰 ราคา: ติดต่อ LINE (Enterprise Pricing)
📊 Message Quota: 25,000+ ข้อความ/เดือน
➕ Additional Messages: ราคาพิเศษตาม Volume
📅 Contract: รายปี (ราคาดีกว่า)

เพิ่มเติม:
✓ Custom Quota ตามต้องการ
✓ Dedicated Account Manager
✓ API Access เต็มรูปแบบ
✓ SLA Guarantee
✓ Custom Integration Support

เหมาะกับ:
→ แบรนด์ขนาดใหญ่
→ Followers 10,000+ คน
→ ต้องการ Custom Solution

════════════════════════════════════════
```

### 3.4 Additional Message Pricing

เมื่อใช้ Quota หมดแล้ว ยังส่งต่อได้โดยซื้อ Additional Message:

```
Additional Message Rate:
─────────────────────────────────────────
Free Plan:      ไม่สามารถซื้อเพิ่มได้
                (ต้องอัปเกรด Plan)

Light Plan:     0.22 บาท/ข้อความ
Standard Plan:  0.19 บาท/ข้อความ
Premium Plan:   ราคาต่อรองได้

ตัวอย่างการคำนวณ (Light Plan):
1,000 ข้อความเพิ่ม = 220 บาท
5,000 ข้อความเพิ่ม = 1,100 บาท
10,000 ข้อความเพิ่ม = 2,200 บาท
─────────────────────────────────────────
```

---

## 4. ตารางเปรียบเทียบทุกแผน

### 4.1 ตารางเปรียบเทียบหลัก

| คุณสมบัติ | Free | Light | Standard | Premium |
|----------|------|-------|----------|---------|
| **ราคา/เดือน** | ฟรี | ~599 ฿ | ~1,500 ฿ | ติดต่อ LINE |
| **Message Quota** | 500 | 5,000 | 15,000 | Custom |
| **Additional (/msg)** | ไม่ได้ | ~0.22 ฿ | ~0.19 ฿ | ต่อรอง |
| **Chat (Reply)** | ไม่จำกัด | ไม่จำกัด | ไม่จำกัด | ไม่จำกัด |
| **Auto Reply** | ✓ | ✓ | ✓ | ✓ |
| **Rich Menu** | 1 | หลาย | หลาย | หลาย |
| **Narrowcast** | ✗ | ✓ | ✓ | ✓ |
| **LINE VOOM** | ✓ | ✓ | ✓ | ✓ |
| **Coupon/Point** | ✓ | ✓ | ✓ | ✓ |
| **Analytics** | พื้นฐาน | ขยาย | เต็ม | เต็ม+Custom |
| **Priority Support** | ✗ | ✗ | ✓ | ✓ |
| **API Access** | จำกัด | ✓ | ✓ | ✓ |

### 4.2 เปรียบเทียบ Value per Baht

```
Value Analysis:
─────────────────────────────────────────────────────

Free Plan:
  500 msg ÷ 0 ฿ = ∞ (แต่มีข้อจำกัดมาก)
  
Light Plan:
  5,000 msg ÷ 599 ฿ = 8.3 msg/฿
  ราคาต่อข้อความ = 0.12 ฿/msg
  
Standard Plan:
  15,000 msg ÷ 1,500 ฿ = 10 msg/฿
  ราคาต่อข้อความ = 0.10 ฿/msg ← คุ้มกว่า
  
ข้อสรุป:
→ Standard Plan คุ้มค่ากว่า Light ถ้าใช้ครบ Quota
→ Light Plan ดีถ้าใช้แค่ 5,000-8,000 msg/เดือน

─────────────────────────────────────────────────────
```

### 4.3 การเลือกแผนตามจำนวน Followers

```
แนวทางเลือกแผน:
═══════════════════════════════════════════════════

Followers    ส่งต่อเดือน    แนะนำ
──────────────────────────────────────────────────
0-100        1-5 ครั้ง      Free Plan
100-500      1-3 ครั้ง      Free Plan
500-1,000    2-4 ครั้ง      Light Plan
1,000-3,000  2-4 ครั้ง      Light Plan
3,000-5,000  3-6 ครั้ง      Standard Plan
5,000-10,000 2-4 ครั้ง      Standard Plan
10,000+      ตามต้องการ     Premium/ติดต่อ LINE

สูตรคำนวณ:
แผนที่เหมาะ = Followers × ครั้งที่ส่ง/เดือน

ตัวอย่าง:
2,000 Followers × 4 ครั้ง = 8,000 msg → Light Plan (5,000 + Buy 3,000)
หรือ Standard Plan (15,000 ใช้ไป 8,000 เหลือ 7,000)

═══════════════════════════════════════════════════
```

---

## 5. วิธีคำนวณ Quota ที่ต้องการ

### 5.1 สูตรคำนวณพื้นฐาน

```
สูตร:
Monthly Quota ที่ต้องการ = Followers × Broadcasts per Month

ตัวแปร:
- Followers = จำนวนผู้ติดตามปัจจุบัน
- Broadcasts = ข้อความที่วางแผนส่งต่อเดือน

ตัวอย่าง:
Followers: 3,500 คน
ส่ง Broadcast: 4 ครั้ง/เดือน (ทุกสัปดาห์)
Quota ที่ต้องการ: 3,500 × 4 = 14,000 msg/เดือน
→ แนะนำ: Standard Plan (15,000 msg)
```

### 5.2 คิดรวม Growth Rate

ธุรกิจส่วนใหญ่มี Follower เพิ่มขึ้นต่อเนื่อง ควรคิดรวม Growth ด้วย:

```
สูตรคำนวณรวม Growth:

ปัจจุบัน: 2,000 Followers
Growth Rate: 10%/เดือน

เดือน 1: 2,000 × 4 = 8,000 msg
เดือน 2: 2,200 × 4 = 8,800 msg
เดือน 3: 2,420 × 4 = 9,680 msg
เดือน 6: 3,221 × 4 = 12,884 msg

สรุป: ควรเลือก Plan ที่รองรับ 6-12 เดือนข้างหน้า
→ เลือก Standard Plan ดีกว่า Light Plan
   เพราะ 6 เดือนต่อมาจะเกิน 5,000 msg
```

### 5.3 Quota Calculator เครื่องมือช่วยคำนวณ

```python
# Quota Calculator (Python Script)
# วิธีใช้: คำนวณ Quota และเลือก Plan ที่เหมาะสม

def calculate_quota(followers, broadcasts_per_month, growth_rate=0):
    """
    คำนวณ Quota ที่ต้องการ
    
    Parameters:
    - followers: จำนวน Followers ปัจจุบัน
    - broadcasts_per_month: จำนวน Broadcast ต่อเดือน
    - growth_rate: อัตราการเติบโต (เช่น 0.1 = 10%)
    """
    # Quota เดือนปัจจุบัน
    current_quota = followers * broadcasts_per_month
    
    # Quota หลัง 6 เดือน (รวม Growth)
    future_followers = followers * ((1 + growth_rate) ** 6)
    future_quota = future_followers * broadcasts_per_month
    
    # แนะนำ Plan
    plans = {
        "Free": 500,
        "Light": 5000,
        "Standard": 15000,
        "Premium": 25000
    }
    
    recommended = "Premium"
    for plan, quota in plans.items():
        if future_quota <= quota:
            recommended = plan
            break
    
    return {
        "current_monthly_quota": int(current_quota),
        "future_6month_quota": int(future_quota),
        "recommended_plan": recommended
    }

# ตัวอย่างการใช้งาน
result = calculate_quota(
    followers=3000,
    broadcasts_per_month=4,
    growth_rate=0.10  # 10% growth per month
)

print(f"Quota เดือนนี้: {result['current_monthly_quota']:,} msg")
print(f"Quota หลัง 6 เดือน: {result['future_6month_quota']:,} msg")
print(f"แนะนำ: {result['recommended_plan']} Plan")

# Output:
# Quota เดือนนี้: 12,000 msg
# Quota หลัง 6 เดือน: 21,282 msg
# แนะนำ: Premium Plan
```

### 5.4 Worksheet คำนวณ Quota ของคุณ

```
กรอกข้อมูลของคุณ:
════════════════════════════════════════

จำนวน Followers ปัจจุบัน:        _________ คน
ประมาณ Broadcast ต่อเดือน:       _________ ครั้ง
Growth Rate ที่คาดหวัง:           _________% 
จำนวนเดือนที่วางแผน (แนะนำ 6):   _________ เดือน

การคำนวณ:
Quota เดือนนี้:
= ________ × ________ = ________ msg

Quota หลัง __ เดือน:
= ________ × (1 + ________)^__ × ________ = ________ msg

แผนที่เหมาะสม:
[ ] Free (≤ 500 msg)
[ ] Light (≤ 5,000 msg)
[ ] Standard (≤ 15,000 msg)
[ ] Premium (> 15,000 msg)

════════════════════════════════════════
```

---

## 6. ราคาต่อข้อความและการเปรียบเทียบ

### 6.1 Cost per Message แต่ละแผน

```
Cost Analysis:
─────────────────────────────────────────────────────

Free Plan:
  Base quota: 500 msg @ 0 ฿ = 0.00 ฿/msg (Quota)
  Additional: ไม่สามารถซื้อ

Light Plan (599฿/เดือน):
  Base cost: 599 ÷ 5,000 = 0.12 ฿/msg
  If using all: 599 ÷ 5,000 = 0.12 ฿/msg
  Additional: 0.22 ฿/msg
  
Standard Plan (1,500฿/เดือน):
  Base cost: 1,500 ÷ 15,000 = 0.10 ฿/msg
  Additional: 0.19 ฿/msg
  
เปรียบเทียบกับช่องทางอื่น:
  SMS Marketing: ~0.35-0.80 ฿/msg
  Email Marketing: ~0.01-0.10 ฿/msg (ต้นทุนต่ำ แต่ Open Rate ต่ำมาก)
  Facebook Ads: ~0.50-5.00 ฿/Reach

LINE OA ถือว่า Cost Effective สูงมาก
เมื่อเทียบ Open Rate ~70% vs SMS ~90%
แต่ LINE มี Rich Content ที่ SMS ไม่มี

─────────────────────────────────────────────────────
```

### 6.2 Cost per Conversion

```
คำนวณ Cost per Conversion:

ตัวอย่าง Standard Plan:
─────────────────────────────────────────
ส่ง Broadcast: 10,000 ข้อความ/เดือน
ค่าใช้จ่าย: 1,500 ฿ (Standard)
Open Rate: 70% = 7,000 คนเปิดอ่าน
Click Rate: 20% = 1,400 คนคลิก
Conversion Rate: 5% = 70 คนซื้อ

Cost per Click = 1,500 ÷ 1,400 = 1.07 ฿/click
Cost per Conversion = 1,500 ÷ 70 = 21.4 ฿/sale

ถ้า Average Order Value = 500 ฿:
Revenue = 70 × 500 = 35,000 ฿
Cost = 1,500 ฿
ROI = (35,000 - 1,500) / 1,500 × 100 = 2,233%!

─────────────────────────────────────────
```

---

## 7. การอัปเกรดและดาวน์เกรดแผน

### 7.1 การอัปเกรด Plan

```
ขั้นตอนอัปเกรด:
1. LINE OA Manager → Settings → Messaging
2. Plan & Message Quota
3. คลิก "Upgrade Plan"
4. เลือก Plan ที่ต้องการ
5. ตรวจสอบรายละเอียด
6. ชำระเงิน
7. Plan Active ทันที

หมายเหตุอัปเกรด:
✓ Quota เพิ่มทันที
✓ Quota เดิมที่เหลือยังใช้ได้
✓ เริ่ม Charge วันที่อัปเกรด
```

### 7.2 การดาวน์เกรด Plan

```
ขั้นตอนดาวน์เกรด:
1. LINE OA Manager → Settings → Messaging
2. Plan & Message Quota
3. คลิก "Change Plan"
4. เลือก Plan ที่ต่ำกว่า
5. ยืนยันการเปลี่ยน

หมายเหตุดาวน์เกรด:
⚠️ Quota ลดลงทันที (รอบบิลหน้า)
⚠️ Quota ที่ใช้ไปแล้วไม่คืน
⚠️ ถ้าใช้ Quota เกิน Plan ใหม่ → อาจถูก Charge Additional
✓ Feature ยังคงอยู่จนถึงปลายรอบบิล
```

### 7.3 การยกเลิก Plan

```
ขั้นตอนยกเลิก (ลงมา Free):
1. LINE OA Manager → Settings → Messaging
2. Plan & Message Quota
3. คลิก "Cancel Plan"
4. ยืนยัน

หลังยกเลิก:
→ Quota ลดลงเป็น 500/เดือน รอบหน้า
→ Feature ขั้นสูงบางอย่างอาจถูกปิด
→ บัญชียังใช้งานได้ตามปกติ
→ ไม่มีค่าปรับการยกเลิก
```

---

## 8. ช่องทางการชำระเงิน

### 8.1 วิธีชำระเงิน LINE OA

```
ช่องทางชำระเงิน:
├── บัตรเครดิต/เดบิต
│   ├── Visa
│   ├── Mastercard
│   └── JCB
│
├── LINE Pay
│   └── จ่ายผ่าน LINE Pay Balance หรือบัตร
│
└── PromptPay / QR Code (บางกรณี)

หมายเหตุ:
→ ชำระเป็น บาท (THB) สำหรับบัญชีไทย
→ หัก Auto ทุกต้นเดือน
→ รับ Invoice/Tax Invoice อีเมล
```

### 8.2 การจัดการ Payment Settings

```
จัดการข้อมูลการชำระเงิน:
Settings → Plan & Payment → Payment Methods

ทำได้:
✓ เพิ่ม/เปลี่ยน บัตรเครดิต
✓ ดู Invoice ย้อนหลัง
✓ ขอ Tax Invoice
✓ ตั้งค่า Auto-renewal
```

### 8.3 Tax Invoice และการทำบัญชี

```
การขอ Tax Invoice:
Settings → Plan & Payment → Billing History
→ เลือก Invoice ที่ต้องการ
→ Download PDF

ข้อมูลใน Invoice:
→ ชื่อบริษัท LINE Corporation
→ จำนวนเงิน
→ ภาษี VAT 7%
→ วันที่
→ รายการบริการ

สำหรับบัญชีธุรกิจ:
→ สามารถใช้เป็นค่าใช้จ่ายทางภาษีได้
→ ปรึกษาผู้ทำบัญชีสำหรับรายละเอียด
```

---

## 9. เคล็ดลับประหยัด Message Quota

### 9.1 กลยุทธ์ลด Quota Usage

```
10 เคล็ดลับประหยัด Message Quota:
════════════════════════════════════════

1. ใช้ LINE VOOM แทน Broadcast สำหรับ Content ไม่ด่วน
   → VOOM ฟรีไม่จำกัด
   → ดี สำหรับ Regular Content

2. ส่ง Broadcast เฉพาะกลุ่มที่ Active
   → ใช้ Narrowcast ตัด Inactive Followers ออก
   → ประหยัดได้ 20-30%

3. กำจัด Followers ที่ Block มานาน
   → Followers ที่ Block ก็ยังนับ Quota ถ้า Send ไป
   → ทาง API สามารถ Filter ออกได้

4. Segment ก่อนส่ง
   → ส่งสินค้าผู้หญิงให้ผู้หญิงเท่านั้น
   → ประหยัดได้ครึ่งหนึ่ง

5. ผสานข้อมูลในข้อความเดียว
   → แทนส่ง 3 ข้อความ ส่ง 1 ข้อความที่มีเนื้อหาครบ
   → ประหยัด 66%

6. ใช้ Chat แทน Broadcast สำหรับลูกค้ารายบุคคล
   → Chat ไม่นับ Quota

7. สร้าง Audience จาก Chat History
   → ส่งเฉพาะคนที่เคย Engage
   → Higher ROI + ประหยัด Quota

8. กำหนดตาราง Broadcast ที่ชัดเจน
   → ส่งเดือนละ 4 ครั้ง (ทุกอาทิตย์)
   → ไม่ส่งแบบ Ad-hoc เยอะเกินไป

9. วัด A/B Test ก่อน Full Broadcast
   → ส่งให้ 20% ก่อน วัดผล แล้วค่อย Send ทั้งหมด
   → ลด Waste จาก Message ที่ผลไม่ดี

10. ตั้ง Budget Alert
    → แจ้งเตือนเมื่อใช้ไป 80% ของ Quota
    → ไม่เกิน Budget โดยไม่ตั้งใจ

════════════════════════════════════════
```

### 9.2 Segment Strategy เพื่อประหยัด Quota

```
ตัวอย่าง Segment Strategy:

ธุรกิจ: ร้านเสื้อผ้า
Followers: 5,000 คน
Plan: Light (5,000 Quota)
เป้าหมาย: ส่งได้ 4 ครั้ง/เดือน โดยไม่เกิน Quota

ปัญหา: 5,000 × 4 = 20,000 msg > 5,000 Quota

วิธีแก้ด้วย Segment:

Week 1: ส่งสินค้าผู้หญิงเฉพาะผู้หญิง (2,500 คน)
Week 2: ส่งสินค้าผู้ชายเฉพาะผู้ชาย (1,500 คน)
Week 3: ส่ง Flash Sale ผู้ที่เคย Click (1,000 คน)
Week 4: ส่ง Restock ผู้ที่เคย Chat (500 คน)

Total: 2,500+1,500+1,000+500 = 5,500 msg
→ เกินแค่ 500 msg = ซื้อเพิ่มแค่ 110 ฿ (Light)

ผล: ส่งได้ 4 ครั้ง ค่าใช้จ่ายรวม 599+110 = 709 ฿
แทน Standard 1,500 ฿ → ประหยัด 791 ฿/เดือน!
```

### 9.3 Rich Message vs Text Message

```
เคล็ดลับ: รวมเนื้อหาในข้อความ Rich เดียว

แทนส่ง 3 ข้อความแยก:
❌ Text: "สวัสดีครับ มีโปรโมชั่นใหม่!"
❌ Image: [รูปสินค้า]
❌ Text: "กด Link นี้เพื่อสั่งซื้อ: [URL]"
= ใช้ 3 × Followers Quota

ส่ง 1 Rich Message รวม:
✓ Rich Message: รูป + ข้อความ + ปุ่ม Link
= ใช้ 1 × Followers Quota (ประหยัด 67%!)

Flex Message (ขั้นสูง):
✓ ออกแบบเอง = Card Style Message
= ข้อมูลเยอะ ใน 1 Message
```

---

## 10. กลยุทธ์วางแผน Budget สำหรับ LINE OA

### 10.1 Budget Planning Framework

```
LINE OA Budget Planning:
════════════════════════════════════════

Step 1: กำหนด Business Goal
  → ต้องการลูกค้าใหม่ X คน/เดือน
  → ต้องการ Revenue Y บาท/เดือน
  → Retention Rate Z%

Step 2: คำนวณ Required Quota
  → ดูหัวข้อ 5.1-5.4

Step 3: เลือก Plan
  → ดูตารางเปรียบเทียบ

Step 4: วาง Campaign Calendar
  → กำหนดวัน/เวลา Broadcast ทุกเดือน
  → จัดทำ Content Calendar

Step 5: วัดผลและปรับ
  → วัด ROI ทุกเดือน
  → ปรับ Plan ตามความจำเป็น

════════════════════════════════════════
```

### 10.2 ตัวอย่าง Budget Planning ตามขนาดธุรกิจ

**Micro Business (1-2 คน):**
```
ธุรกิจ: ขายของ Handmade ใน LINE
Followers: 300 คน
Budget: < 500 ฿/เดือน
แผน: Free Plan

การใช้:
→ 500 Quota ส่งได้เดือนละ 1 ครั้ง (Broadcast ทั้ง 300 คน)
→ เสริมด้วย LINE VOOM ทุกสัปดาห์ (ฟรี)
→ ตอบ Chat ด้วยตัวเองทุกวัน
ค่าใช้จ่าย: 0 ฿/เดือน
```

**Small Business:**
```
ธุรกิจ: ร้านกาแฟ 1 สาขา
Followers: 2,000 คน
Budget: 1,000-2,000 ฿/เดือน
แผน: Light Plan

การใช้:
→ 5,000 Quota = ส่งได้ 2 ครั้ง/เดือน (2 × 2,000)
→ LINE VOOM 3-4 ครั้ง/สัปดาห์
→ Auto Reply สำหรับ FAQ
→ Narrowcast: โปรโมชั่นวันเกิด
ค่าใช้จ่าย: 599 ฿/เดือน
```

**Medium Business:**
```
ธุรกิจ: Fashion E-commerce
Followers: 8,000 คน
Budget: 3,000-5,000 ฿/เดือน
แผน: Standard Plan

การใช้:
→ 15,000 Quota = ส่งได้ ~2 ครั้ง/เดือน (2 × 8,000 - Extra Savings จาก Segment)
→ Narrowcast เฉพาะกลุ่ม (Male/Female, Region)
→ Campaign พิเศษ 11.11, 12.12
→ Loyalty Program via Point Card
ค่าใช้จ่าย: 1,500 ฿/เดือน (+ Additional ถ้าต้องการ)
```

### 10.3 Seasonal Budget Planning

```
ปฏิทิน Broadcast สำหรับปีหนึ่ง:
────────────────────────────────────────────────

Q1 (ม.ค.-มี.ค.):
  ม.ค.: ปีใหม่ Campaign (Budget เพิ่ม)
  ก.พ.: Valentine's Day Campaign (Segment: คู่รัก)
  มี.ค.: ปกติ

Q2 (เม.ย.-มิ.ย.):
  เม.ย.: สงกรานต์ Campaign (Budget เพิ่ม)
  พ.ค.: วันแม่ (ล่วงหน้า 2 สัปดาห์)
  มิ.ย.: Mid-year Sale

Q3 (ก.ค.-ก.ย.):
  ก.ค. - ก.ย.: ปกติ (Budget ประหยัด)
  ก.ย.: เริ่มเตรียม Q4

Q4 (ต.ค.-ธ.ค.):
  ต.ค.: Halloween (ถ้าเกี่ยวข้อง)
  พ.ย.: 11.11 MEGA SALE (Budget สูงสุด!)
  ธ.ค.: ปีใหม่ Campaign (Budget สูง)

กฎ Budget:
→ ปกติ: Standard Plan
→ Big Campaign: Standard + Additional Message
→ ปีใหม่/11.11: พิจารณา Upgrade ชั่วคราว

────────────────────────────────────────────────
```

---

## 11. ROI Calculator สำหรับ LINE OA

### 11.1 สูตรคำนวณ ROI

```
ROI Formula:
ROI = (Revenue - Cost) / Cost × 100%

LINE OA ROI Calculation:
─────────────────────────────────────────

Input:
Plan Cost:        _______ ฿/เดือน
Additional:       _______ ฿/เดือน
Total Cost:       _______ ฿/เดือน

Output:
Recipients:       _______ คน
Open Rate:        _______ %
Opens:            _______ คน
Click Rate:       _______ %
Clicks:           _______ คน
Conversion Rate:  _______ %
Conversions:      _______ คน
Avg Order Value:  _______ ฿
Total Revenue:    _______ ฿

ROI = (Total Revenue - Total Cost) / Total Cost × 100%
ROI = _________%

─────────────────────────────────────────
```

### 11.2 ROI Benchmark ตามอุตสาหกรรม

```
ROI เฉลี่ยของ LINE OA ตามอุตสาหกรรม:
════════════════════════════════════════

ร้านอาหาร:          ROI 300-800%
ร้านค้าออนไลน์:     ROI 500-2,000%
คลินิก/สุขภาพ:     ROI 200-500%
อสังหาริมทรัพย์:   ROI 1,000-5,000%
การศึกษา:          ROI 200-600%
บริการทั่วไป:       ROI 150-400%

หมายเหตุ:
→ ค่าเหล่านี้เป็นค่าประมาณ
→ ขึ้นกับ Content, Timing, และ Offer
→ ROI จะดีขึ้นเรื่อยๆ เมื่อ Optimize

════════════════════════════════════════
```

### 11.3 ตัวอย่าง ROI Calculation จริง

```
Case Study: ร้านเบเกอรี่ "หวานหอม"
─────────────────────────────────────────────────────

ข้อมูล:
Plan: Light (599 ฿/เดือน)
Followers: 1,500 คน
Broadcast: 4 ครั้ง/เดือน (ใช้ Quota 6,000)
Additional: 1,000 msg × 0.22 = 220 ฿
Total Cost: 599 + 220 = 819 ฿/เดือน

ผลการส่ง (เฉลี่ย/Broadcast):
Recipients: 1,500
Opened: 1,050 (70%)
Clicked: 262 (17.5%)
Orders: 39 (15% of Clicks)
Average Order: 450 ฿
Revenue per Broadcast: 39 × 450 = 17,550 ฿

รายเดือน:
Revenue: 17,550 × 4 = 70,200 ฿
Cost: 819 ฿
ROI: (70,200 - 819) / 819 × 100 = 8,469%!!!

หมายเหตุ: นี่คือ Revenue ที่เกิดจาก LINE OA โดยตรง
อาจไม่รวม Repeat Purchase จากลูกค้าที่รู้จักร้านมาก่อน

─────────────────────────────────────────────────────
```

---

## 12. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: เลือก Plan สำหรับธุรกิจ

**คำสั่ง:** กรอกข้อมูลธุรกิจของคุณและคำนวณ Plan ที่เหมาะสม

```
กรอกข้อมูล:
════════════════════════════════════════

ชื่อธุรกิจ: _______________________________
ประเภทธุรกิจ: _____________________________

จำนวน Followers ปัจจุบัน: _______ คน
จำนวน Broadcast ที่ต้องการ/เดือน: _______ ครั้ง
Growth Rate (ประมาณ): _______ %/เดือน
ระยะเวลาวางแผน: _______ เดือน

คำนวณ:
Quota เดือนนี้: _______ × _______ = _______ msg
Quota หลัง __ เดือน: _______ msg (รวม Growth)

Plan ที่แนะนำ: _______
ค่าใช้จ่าย/เดือน: _______ ฿

เหตุผลที่เลือก: ________________________________
________________________________________________

════════════════════════════════════════
```

### แบบฝึกหัดที่ 2: วาง Content Calendar

**คำสั่ง:** วาง Broadcast Calendar สำหรับเดือนหน้า

```
Broadcast Calendar - [เดือน ปี]:
════════════════════════════════════════════════════

สัปดาห์ 1 (วันที่ __-__):
  วัน/เวลาส่ง: ___________
  หัวข้อ: ___________
  กลุ่มเป้าหมาย: ___________
  Quota ที่ใช้: ___________

สัปดาห์ 2 (วันที่ __-__):
  วัน/เวลาส่ง: ___________
  หัวข้อ: ___________
  กลุ่มเป้าหมาย: ___________
  Quota ที่ใช้: ___________

สัปดาห์ 3 (วันที่ __-__):
  วัน/เวลาส่ง: ___________
  หัวข้อ: ___________
  กลุ่มเป้าหมาย: ___________
  Quota ที่ใช้: ___________

สัปดาห์ 4 (วันที่ __-__):
  วัน/เวลาส่ง: ___________
  หัวข้อ: ___________
  กลุ่มเป้าหมาย: ___________
  Quota ที่ใช้: ___________

Total Quota ที่จะใช้: ___________
Plan ที่ใช้: ___________ (Quota: _________)
ซื้อเพิ่ม (ถ้าต้องการ): ___________ msg × ___ ฿ = ___ ฿

════════════════════════════════════════════════════
```

### แบบฝึกหัดที่ 3: คำนวณ ROI

**คำสั่ง:** คำนวณ ROI ที่คาดหวังจาก LINE OA ของคุณ

```
ROI Calculation Worksheet:
════════════════════════════════════════

ค่าใช้จ่าย:
Plan Cost: _______ ฿/เดือน
Additional Message: _______ ฿/เดือน
ค่า Content (ถ้ามี): _______ ฿/เดือน
Total Cost: _______ ฿/เดือน

ประมาณผลลัพธ์:
Recipients/Broadcast: _______
Open Rate (ประมาณ): _______ %
Click Rate (ประมาณ): _______ %
Conversion Rate (ประมาณ): _______ %
Avg Order Value: _______ ฿
Broadcasts/เดือน: _______

คำนวณ Revenue:
= _______ × _______ × _______ × _______ × _______ × _______
= _______ ฿/เดือน

ROI = (_______ - _______) / _______ × 100 = _______ %

ถ้า ROI ต่ำกว่า 200%:
→ ปรับ: Open Rate, Conversion, ลด Cost, เพิ่ม Order Value

════════════════════════════════════════
```

### แบบฝึกหัดที่ 4: Segment Strategy

**คำสั่ง:** ออกแบบ Segment Strategy เพื่อประหยัด Quota

```
Segment Strategy Plan:
════════════════════════════════════════

Followers ทั้งหมด: _______ คน
Plan Quota: _______ msg/เดือน
เป้าหมาย: ส่งให้ทุกคนอย่างน้อย _______ ครั้ง/เดือน

Segment ที่จะสร้าง:
1. Segment: _____________ 
   ขนาด: _______ คน
   ส่งเนื้อหา: _____________

2. Segment: _____________
   ขนาด: _______ คน
   ส่งเนื้อหา: _____________

3. Segment: _____________
   ขนาด: _______ คน
   ส่งเนื้อหา: _____________

Total Quota ที่ใช้: _______ msg/เดือน
ประหยัดเทียบกับ Broadcast ทั้งหมด: _______ msg

════════════════════════════════════════
```

---

## สรุปส่วนที่ 5

ในส่วนนี้เราได้เรียนรู้:

✅ **Message Quota** - สิ่งที่นับและไม่นับ Quota

✅ **Free Plan** - 500 msg/เดือน เหมาะกับธุรกิจเริ่มต้น

✅ **Paid Plans** - Light, Standard, Premium รายละเอียดและราคา

✅ **ตารางเปรียบเทียบ** - เลือก Plan ที่คุ้มค่าที่สุด

✅ **การคำนวณ Quota** - สูตรและ Worksheet สำหรับธุรกิจของคุณ

✅ **เคล็ดลับประหยัด** - 10 วิธีลด Quota Usage

✅ **ROI Calculator** - วัดผลตอบแทนจาก LINE OA

---

## ส่วนถัดไป

ในส่วนที่ 6 เราจะเรียนรู้การส่งข้อความประเภทต่างๆ ในเชิงลึก:
- Text, Image, Video Messages
- Rich Message และ Image Map
- Flex Message (ออกแบบเอง)
- Card Type Messages
- Best Practices สำหรับแต่ละประเภท

---

## อ้างอิงและแหล่งข้อมูลเพิ่มเติม

```
แหล่งข้อมูลที่เป็นประโยชน์:

LINE Official:
→ ราคาและแผน: https://www.linebiz.com/th/service/line-official-account/plan/
→ FAQ: https://help.line.me/line-official-account/
→ Developer Docs: https://developers.line.biz/

เครื่องมือช่วยคำนวณ:
→ Google Sheets Template (สร้างเองตาม Worksheet ในบทเรียน)
→ ROI Calculator ออนไลน์ต่างๆ

Community:
→ LINE Developers Thailand (Facebook Group)
→ LINE Business Thailand (Official Page)
```

---

*อัพเดทล่าสุด: กันยายน 2025*  
*หลักสูตร: LINE OA Complete Course*  
*ส่วนที่ 5 จาก 13*

> **หมายเหตุ:** ราคาและรายละเอียดแผนอาจเปลี่ยนแปลงได้  
> ตรวจสอบราคาล่าสุดที่ www.linebiz.com/th เสมอ
