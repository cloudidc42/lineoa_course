# Part 13: Coupons & Rewards - ระบบคูปองและสะสมแต้ม

## บทนำ

ระบบ Coupons และ Rewards ใน LINE Official Account เป็นเครื่องมือทรงพลังในการสร้างความภักดีของลูกค้า (Customer Loyalty) เพิ่มยอดขาย และกระตุ้นให้ลูกค้ากลับมาซื้อซ้ำ บทนี้จะครอบคลุมการสร้าง Digital Coupon ประเภทต่างๆ ระบบ Reward Card (สะสมแสตมป์) และกลยุทธ์การใช้งานที่มีประสิทธิภาพ

---

## 13.1 ภาพรวมระบบ Coupons ใน LINE OA

### ประเภทของ Coupon ใน LINE OA

| ประเภท | คำอธิบาย | การใช้งาน |
|--------|----------|-----------|
| Discount Coupon | ส่วนลดราคา (% หรือจำนวนเงิน) | โปรโมชั่นทั่วไป |
| Gift Coupon | ของแถม/ของสมนาคุณ | Upselling, Member Benefit |
| Special Coupon | สิทธิพิเศษ เช่น Priority Service | Premium Member |
| Stamp Card | สะสมแสตมป์แลกรางวัล | Loyalty Program |

### ข้อดีของ Digital Coupon เทียบกับ Physical Coupon

| คุณสมบัติ | Digital Coupon | Physical Coupon |
|-----------|---------------|-----------------|
| ต้นทุน | ต่ำมาก | สูง (พิมพ์, แจก) |
| การกระจาย | ทันที, ทุกที่ | จำกัดช่องทาง |
| การติดตาม | แม่นยำ 100% | ยาก |
| วันหมดอายุ | ปรับได้ | แก้ไขยาก |
| การป้องกันการทุจริต | ระบบตรวจสอบ | ยาก |
| ประสบการณ์ผู้ใช้ | สะดวก ไม่ต้องพกกระดาษ | ต้องพกตลอด |

---

## 13.2 การสร้าง Digital Coupon

### 13.2.1 Discount Coupon

#### วิธีสร้างใน LINE OA Manager

```
ขั้นตอนที่ 1: เข้า LINE OA Manager
ขั้นตอนที่ 2: คลิก "Messages" > "Coupon"
ขั้นตอนที่ 3: คลิก "+ Create Coupon"
ขั้นตอนที่ 4: กรอกรายละเอียด:

  ประเภทคูปอง:
  ○ Discount (ส่วนลด)
  ○ Gift (ของแถม)
  ○ Special (พิเศษ)

  ชื่อคูปอง: [ชื่อที่ชัดเจน เช่น "ลด 50 บาท"]
  คำอธิบาย: [รายละเอียดเงื่อนไข]
  วันหมดอายุ: [วัน/เดือน/ปี]
  จำนวนจำกัด: [จำนวนหรือไม่จำกัด]
  
ขั้นตอนที่ 5: อัปโหลดรูปคูปอง
ขั้นตอนที่ 6: กำหนดเงื่อนไข
ขั้นตอนที่ 7: บันทึก
```

#### รูปแบบส่วนลด

| รูปแบบ | ตัวอย่าง | เหมาะกับ |
|--------|---------|---------|
| % ส่วนลด | ลด 20% | สินค้าราคาสูง |
| จำนวนเงิน | ลด ฿50 | กระตุ้นการซื้อ |
| ซื้อ 1 แถม 1 | B1G1 | เพิ่มปริมาณ |
| ส่งฟรี | Free Shipping | ลดอุปสรรค |
| ซื้อครบ X แถม Y | Buy ฿500 Get Gift | เพิ่ม Basket Size |

### 13.2.2 สร้าง Coupon ผ่าน API

```python
import requests
import json
import hashlib
import time
from datetime import datetime, timedelta

class LineCouponManager:
    """
    จัดการ Coupon ใน LINE OA
    """
    
    def __init__(self, channel_access_token):
        self.token = channel_access_token
        self.headers = {
            "Authorization": f"Bearer {channel_access_token}",
            "Content-Type": "application/json"
        }
    
    def create_coupon_message(self, coupon_data, user_id):
        """
        ส่ง Coupon ให้ผู้ใช้
        
        Args:
            coupon_data: รายละเอียดคูปอง
            user_id: LINE User ID
        """
        
        coupon_bubble = self._build_coupon_bubble(coupon_data)
        
        message = {
            "type": "flex",
            "altText": f"คูปองพิเศษสำหรับคุณ: {coupon_data['title']}",
            "contents": coupon_bubble
        }
        
        return self._send_message(user_id, message)
    
    def _build_coupon_bubble(self, coupon):
        """สร้าง Coupon Bubble"""
        
        # สีตาม Coupon Type
        colors = {
            "discount": "#e74c3c",
            "gift": "#9b59b6",
            "special": "#f39c12",
            "birthday": "#e91e63"
        }
        
        color = colors.get(coupon.get("type", "discount"), "#1DB446")
        
        # คำนวณวันเหลือ
        expiry = coupon.get("expiry_date")
        days_remaining = None
        if expiry:
            if isinstance(expiry, str):
                expiry = datetime.fromisoformat(expiry)
            days_remaining = (expiry - datetime.now()).days
        
        bubble = {
            "type": "bubble",
            "size": "mega",
            "header": {
                "type": "box",
                "layout": "vertical",
                "backgroundColor": color,
                "paddingAll": "20px",
                "contents": [
                    {
                        "type": "text",
                        "text": coupon.get("badge", "COUPON"),
                        "color": "#FFFFFF",
                        "size": "sm",
                        "weight": "bold"
                    },
                    {
                        "type": "text",
                        "text": coupon["title"],
                        "color": "#FFFFFF",
                        "size": "xxl",
                        "weight": "bold",
                        "wrap": True
                    }
                ]
            },
            "body": {
                "type": "box",
                "layout": "vertical",
                "paddingAll": "20px",
                "contents": [
                    {
                        "type": "text",
                        "text": coupon.get("description", ""),
                        "size": "sm",
                        "color": "#555555",
                        "wrap": True
                    },
                    {
                        "type": "separator",
                        "margin": "lg",
                        "color": "#EEEEEE"
                    },
                    {
                        "type": "box",
                        "layout": "horizontal",
                        "margin": "lg",
                        "contents": [
                            {
                                "type": "text",
                                "text": "รหัส:",
                                "size": "sm",
                                "color": "#888888",
                                "flex": 0
                            },
                            {
                                "type": "text",
                                "text": coupon.get("code", ""),
                                "size": "xl",
                                "weight": "bold",
                                "color": color,
                                "flex": 1,
                                "align": "end"
                            }
                        ]
                    }
                ]
            },
            "footer": {
                "type": "box",
                "layout": "vertical",
                "paddingAll": "sm",
                "contents": [
                    {
                        "type": "button",
                        "style": "primary",
                        "color": color,
                        "action": {
                            "type": "uri",
                            "label": coupon.get("cta_text", "ใช้คูปอง"),
                            "uri": coupon.get("use_url", "#")
                        }
                    }
                ]
            }
        }
        
        # เพิ่ม countdown ถ้ามีวันหมดอายุ
        if days_remaining is not None:
            urgency_text = f"หมดอายุใน {days_remaining} วัน" if days_remaining > 0 else "หมดอายุวันนี้!"
            urgency_color = "#e74c3c" if days_remaining <= 3 else "#888888"
            
            bubble["body"]["contents"].append({
                "type": "text",
                "text": urgency_text,
                "size": "xs",
                "color": urgency_color,
                "margin": "sm",
                "align": "end"
            })
        
        return bubble
    
    def _send_message(self, to, message):
        """ส่งข้อความ"""
        response = requests.post(
            "https://api.line.me/v2/bot/message/push",
            headers=self.headers,
            json={"to": to, "messages": [message]}
        )
        return response.status_code == 200

# ตัวอย่างการใช้งาน
manager = LineCouponManager("YOUR_ACCESS_TOKEN")

coupon = {
    "type": "discount",
    "title": "ลด 100 บาท!",
    "description": "สำหรับการสั่งซื้อขั้นต่ำ 500 บาท ใช้ได้ถึง 31 ธ.ค. นี้",
    "code": "SAVE100",
    "badge": "💝 EXCLUSIVE",
    "cta_text": "ใช้คูปองเลย",
    "use_url": "https://shop.example.com?coupon=SAVE100",
    "expiry_date": datetime.now() + timedelta(days=14)
}

manager.create_coupon_message(coupon, "Uxxxxxxxxxx")
```

---

## 13.3 ประเภทและเงื่อนไขคูปอง

### 13.3.1 Discount Coupon (คูปองส่วนลด)

#### เงื่อนไขที่กำหนดได้

```python
discount_coupon_conditions = {
    # เงื่อนไขการใช้
    "min_purchase": 500,          # ขั้นต่ำการซื้อ (บาท)
    "max_discount": 200,          # ส่วนลดสูงสุด (บาท) สำหรับ %
    
    # เงื่อนไขสินค้า
    "applicable_categories": ["เสื้อผ้า", "กระเป๋า"],  # หมวดที่ใช้ได้
    "excluded_items": ["item_001", "item_002"],         # สินค้าที่ยกเว้น
    "brand_restriction": ["Nike", "Adidas"],            # แบรนด์เฉพาะ
    
    # เงื่อนไขเวลา
    "valid_from": "2024-01-01",
    "valid_until": "2024-01-31",
    "valid_days": ["Mon", "Tue", "Wed", "Thu", "Fri"],  # วันที่ใช้ได้
    "valid_hours": {"start": "10:00", "end": "22:00"},  # ช่วงเวลา
    
    # เงื่อนไขการใช้
    "usage_limit_per_user": 1,    # ใช้ได้กี่ครั้งต่อคน
    "total_usage_limit": 1000,    # จำนวนรวมทั้งหมด
    "new_customer_only": False,   # เฉพาะลูกค้าใหม่
    
    # เงื่อนไขช่องทาง
    "channel": "online",          # "online", "offline", "both"
    "platform": ["web", "app"]    # ช่องทางที่ใช้ได้
}
```

### 13.3.2 Gift Coupon (คูปองของแถม)

```python
gift_coupon_template = {
    "type": "gift",
    "title": "แลกรับกระเป๋าผ้าฟรี!",
    "description": "ซื้อสินค้าครบ ฿1,000 รับกระเป๋าผ้า Limited Edition ฟรี 1 ใบ",
    "gift_details": {
        "name": "กระเป๋าผ้า Cotton Limited Edition",
        "value": 299,           # มูลค่าของแถม
        "quantity": 1,          # จำนวน
        "sku": "BAG-COTTON-001" # รหัสสินค้า
    },
    "conditions": {
        "min_purchase": 1000,
        "while_stocks_last": True,
        "stock_quantity": 500    # จำนวนที่มี
    }
}
```

### 13.3.3 Special Coupon (คูปองพิเศษ)

```python
special_coupon_examples = [
    {
        "type": "special",
        "subtype": "priority_service",
        "title": "ช่อง VIP สำหรับคุณ",
        "description": "ใช้คูปองนี้เพื่อรับบริการก่อนในแถว VIP",
        "use_case": "บริการที่มีคิว เช่น สปา, ร้านอาหาร"
    },
    {
        "type": "special",
        "subtype": "free_upgrade",
        "title": "อัปเกรดห้องฟรี!",
        "description": "จองห้อง Standard รับการอัปเกรดเป็น Deluxe ฟรี",
        "use_case": "โรงแรม, บริการพรีเมียม"
    },
    {
        "type": "special",
        "subtype": "exclusive_access",
        "title": "เข้าชม Pre-Sale ก่อนใคร",
        "description": "ลูกค้า VIP เท่านั้น! ช้อปสินค้า New Collection ก่อนวันเปิดตัว 24 ชั่วโมง",
        "use_case": "แฟชั่น, ช้อปปิ้ง"
    }
]
```

---

## 13.4 การตั้งค่าวันหมดอายุและเงื่อนไข

### กลยุทธ์วันหมดอายุที่มีประสิทธิภาพ

| กลยุทธ์ | ระยะเวลา | ผลลัพธ์ |
|---------|---------|---------|
| Flash Coupon | 24-48 ชั่วโมง | สร้าง Urgency สูง |
| Weekly Coupon | 7 วัน | สมดุลระหว่าง Urgency และ Convenience |
| Monthly Coupon | 30 วัน | เหมาะกับ Loyalty Program |
| Birthday Coupon | 7-14 วัน (ช่วงวันเกิด) | Personalized Experience |
| Seasonal Coupon | 3-6 เดือน | วางแผนล่วงหน้า |

### การจัดการ Coupon Expiry

```python
from datetime import datetime, timedelta
import pytz

def calculate_expiry_date(strategy, created_at=None):
    """
    คำนวณวันหมดอายุตาม Strategy
    """
    
    if created_at is None:
        created_at = datetime.now(pytz.timezone("Asia/Bangkok"))
    
    expiry_rules = {
        "flash": timedelta(hours=24),
        "flash_2day": timedelta(hours=48),
        "weekly": timedelta(days=7),
        "monthly": timedelta(days=30),
        "quarterly": timedelta(days=90)
    }
    
    if strategy in expiry_rules:
        return created_at + expiry_rules[strategy]
    
    return None

def get_coupon_status(coupon):
    """
    ตรวจสอบสถานะคูปอง
    """
    
    now = datetime.now(pytz.timezone("Asia/Bangkok"))
    expiry = coupon.get("expiry_date")
    
    if isinstance(expiry, str):
        expiry = datetime.fromisoformat(expiry).replace(tzinfo=pytz.utc)
        expiry = expiry.astimezone(pytz.timezone("Asia/Bangkok"))
    
    if now > expiry:
        return "expired"
    
    days_remaining = (expiry - now).days
    
    if days_remaining <= 1:
        return "expiring_soon"  # หมดอายุวันนี้/พรุ่งนี้
    elif days_remaining <= 3:
        return "urgent"          # เหลือน้อยกว่า 3 วัน
    else:
        return "active"
    
    # ตรวจสอบ Usage Limit
    usage_limit = coupon.get("usage_limit_per_user", float("inf"))
    current_usage = coupon.get("current_usage", 0)
    
    if current_usage >= usage_limit:
        return "used"
    
    total_limit = coupon.get("total_usage_limit", float("inf"))
    total_used = coupon.get("total_used", 0)
    
    if total_used >= total_limit:
        return "exhausted"
    
    return "active"

# ตัวอย่างการส่ง Reminder ก่อนคูปองหมดอายุ
def send_coupon_expiry_reminder(coupons, access_token):
    """
    ส่ง Reminder สำหรับคูปองที่ใกล้หมดอายุ
    """
    
    manager = LineCouponManager(access_token)
    
    for coupon in coupons:
        status = get_coupon_status(coupon)
        user_id = coupon.get("user_id")
        
        if status == "expiring_soon":
            # ส่ง Urgent Reminder
            reminder_message = {
                "type": "text",
                "text": f"⚠️ คูปอง '{coupon['title']}' ของคุณจะหมดอายุในวันนี้!\nรีบใช้ก่อนที่จะสายเกินไป 🏃"
            }
            
        elif status == "urgent":
            # ส่ง Regular Reminder
            expiry = coupon.get("expiry_date")
            if isinstance(expiry, datetime):
                expiry_str = expiry.strftime("%d/%m/%Y")
            else:
                expiry_str = str(expiry)
                
            reminder_message = {
                "type": "text", 
                "text": f"📅 แจ้งเตือน: คูปอง '{coupon['title']}' จะหมดอายุวันที่ {expiry_str}\nอย่าลืมใช้นะคะ!"
            }
        else:
            continue
        
        # ส่ง Coupon Card + Reminder
        import requests
        requests.post(
            "https://api.line.me/v2/bot/message/push",
            headers=manager.headers,
            json={"to": user_id, "messages": [reminder_message]}
        )
```

---

## 13.5 Reward Card (Loyalty Stamps)

### 13.5.1 ภาพรวม Reward Card

Reward Card เป็นระบบสะสมแสตมป์ดิจิทัล คล้ายกับ Loyalty Card แบบกระดาษ แต่สะดวกกว่า ติดตามได้ง่ายกว่า และป้องกันการทุจริตได้

### วิธีทำงานของ Reward Card

```
1. ลูกค้าซื้อสินค้า/บริการ
2. ได้รับแสตมป์ (อัตโนมัติหรือ Manual)
3. สะสมครบตามเป้า
4. แลกรับรางวัล
5. เริ่มต้นรอบใหม่
```

### 13.5.2 การสร้าง Reward Card ใน LINE OA Manager

#### ขั้นตอนการสร้าง

```
1. เข้า LINE OA Manager
2. คลิก "Messages" > "Reward Card" หรือ "Stamp Card"
3. คลิก "+ Create"
4. กำหนดรายละเอียด:

  ชื่อการ์ด: [ชื่อโปรแกรม เช่น "Loyalty Stamps 2024"]
  คำอธิบาย: [อธิบายโปรแกรม]
  
  จำนวนช่องแสตมป์:
  ○ 5 ช่อง
  ○ 8 ช่อง
  ○ 10 ช่อง
  ○ Custom (3-20 ช่อง)
  
  รูปแสตมป์:
  - Stamp ว่าง: อัปโหลดรูป
  - Stamp เต็ม: อัปโหลดรูป
  - พื้นหลัง: อัปโหลดรูป

5. กำหนดรางวัล:
  ช่องที่ X: [รางวัล] (สามารถตั้งหลายช่วงได้)

6. บันทึก
```

### 13.5.3 Stamp Card API Integration

```python
class StampCardManager:
    """
    จัดการระบบสะสมแสตมป์
    """
    
    def __init__(self, access_token, db_connection):
        self.token = access_token
        self.db = db_connection
        self.headers = {
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json"
        }
    
    def get_user_stamps(self, user_id, card_id):
        """ดึงข้อมูลแสตมป์ของผู้ใช้"""
        
        cursor = self.db.cursor()
        cursor.execute(
            "SELECT stamps, total_stamps, last_stamp_date FROM stamp_cards WHERE user_id = ? AND card_id = ?",
            (user_id, card_id)
        )
        result = cursor.fetchone()
        
        if result:
            return {
                "stamps": result[0],
                "total_stamps": result[1],
                "last_stamp_date": result[2]
            }
        else:
            # สร้างการ์ดใหม่สำหรับผู้ใช้
            cursor.execute(
                "INSERT INTO stamp_cards (user_id, card_id, stamps, total_stamps) VALUES (?, ?, 0, 0)",
                (user_id, card_id)
            )
            self.db.commit()
            return {"stamps": 0, "total_stamps": 0, "last_stamp_date": None}
    
    def add_stamp(self, user_id, card_id, amount=1, reason="purchase"):
        """
        เพิ่มแสตมป์ให้ผู้ใช้
        
        Args:
            user_id: LINE User ID
            card_id: รหัส Stamp Card
            amount: จำนวนแสตมป์
            reason: เหตุผล (purchase, bonus, referral)
        
        Returns:
            dict ผลลัพธ์และรางวัลที่ได้รับ (ถ้ามี)
        """
        
        card_config = self._get_card_config(card_id)
        user_stamps = self.get_user_stamps(user_id, card_id)
        
        old_stamps = user_stamps["stamps"]
        new_stamps = old_stamps + amount
        
        rewards_earned = []
        
        # ตรวจสอบ Milestones
        for milestone in card_config["milestones"]:
            milestone_stamps = milestone["stamps_required"]
            
            # ตรวจสอบว่าถึง Milestone ใหม่หรือไม่
            if old_stamps < milestone_stamps <= new_stamps:
                rewards_earned.append(milestone["reward"])
        
        # ถ้าครบรอบ
        max_stamps = card_config["max_stamps"]
        if new_stamps >= max_stamps:
            # แลกรางวัลหลัก
            rewards_earned.append(card_config["main_reward"])
            new_stamps = new_stamps % max_stamps  # Reset หรือ Carry Over
        
        # อัปเดต Database
        cursor = self.db.cursor()
        cursor.execute(
            """UPDATE stamp_cards 
               SET stamps = ?, total_stamps = total_stamps + ?, last_stamp_date = ?
               WHERE user_id = ? AND card_id = ?""",
            (new_stamps, amount, datetime.now().isoformat(), user_id, card_id)
        )
        self.db.commit()
        
        # Log Transaction
        cursor.execute(
            """INSERT INTO stamp_transactions (user_id, card_id, amount, reason, created_at)
               VALUES (?, ?, ?, ?, ?)""",
            (user_id, card_id, amount, reason, datetime.now().isoformat())
        )
        self.db.commit()
        
        return {
            "success": True,
            "old_stamps": old_stamps,
            "new_stamps": new_stamps,
            "stamps_added": amount,
            "rewards_earned": rewards_earned,
            "next_milestone": self._get_next_milestone(new_stamps, card_config)
        }
    
    def _get_card_config(self, card_id):
        """ดึง Config ของ Stamp Card"""
        # ตัวอย่าง Config
        return {
            "id": card_id,
            "name": "Loyalty Stamps",
            "max_stamps": 10,
            "main_reward": {
                "type": "coupon",
                "value": "FREE_ITEM_001",
                "description": "รับสินค้าฟรี 1 ชิ้น"
            },
            "milestones": [
                {
                    "stamps_required": 5,
                    "reward": {
                        "type": "discount",
                        "value": "MID_DISCOUNT_10",
                        "description": "ส่วนลด 10% ครึ่งทาง"
                    }
                }
            ]
        }
    
    def _get_next_milestone(self, current_stamps, card_config):
        """คำนวณ Milestone ถัดไป"""
        
        milestones = sorted(
            card_config["milestones"],
            key=lambda x: x["stamps_required"]
        )
        
        for milestone in milestones:
            if current_stamps < milestone["stamps_required"]:
                return {
                    "stamps_needed": milestone["stamps_required"] - current_stamps,
                    "reward": milestone["reward"]["description"]
                }
        
        # ถ้าผ่าน milestones ทั้งหมดแล้ว
        stamps_to_complete = card_config["max_stamps"] - current_stamps
        return {
            "stamps_needed": stamps_to_complete,
            "reward": card_config["main_reward"]["description"]
        }
    
    def send_stamp_card_status(self, user_id, card_id):
        """ส่ง Visual ของ Stamp Card ให้ผู้ใช้"""
        
        user_stamps = self.get_user_stamps(user_id, card_id)
        card_config = self._get_card_config(card_id)
        
        stamps_count = user_stamps["stamps"]
        max_stamps = card_config["max_stamps"]
        
        # สร้างภาพแสตมป์
        filled_stamps = "🟢" * stamps_count
        empty_stamps = "⚪" * (max_stamps - stamps_count)
        stamp_visual = filled_stamps + empty_stamps
        
        next_milestone = self._get_next_milestone(stamps_count, card_config)
        
        message_text = (
            f"🎖️ Loyalty Card ของคุณ\n"
            f"{'─' * 20}\n"
            f"แสตมป์: {stamp_visual}\n"
            f"สะสม: {stamps_count}/{max_stamps}\n\n"
            f"อีก {next_milestone['stamps_needed']} แสตมป์รับ:\n"
            f"🎁 {next_milestone['reward']}"
        )
        
        import requests
        requests.post(
            "https://api.line.me/v2/bot/message/push",
            headers=self.headers,
            json={
                "to": user_id,
                "messages": [{"type": "text", "text": message_text}]
            }
        )
```

---

## 13.6 การกำหนดรางวัลสะสมแสตมป์

### โครงสร้างรางวัล (Reward Tiers)

```python
reward_structure_examples = {
    "coffee_shop": {
        "name": "Coffee Loyalty Card",
        "max_stamps": 10,
        "stamp_trigger": "purchase",  # ได้แสตมป์ทุกครั้งที่ซื้อ
        "milestones": [
            {"stamps": 3, "reward": "เครื่องดื่มราคา 50 บาทฟรี 1 แก้ว"},
            {"stamps": 7, "reward": "ขนมหวาน 1 ชิ้นฟรี"},
            {"stamps": 10, "reward": "เครื่องดื่ม Free Size ฟรี 1 แก้ว"}
        ]
    },
    
    "restaurant": {
        "name": "Dining Rewards",
        "max_stamps": 8,
        "stamp_trigger": "per_100_baht",  # ได้ 1 แสตมป์ทุก 100 บาท
        "milestones": [
            {"stamps": 4, "reward": "เครื่องดื่มฟรี 1 แก้ว"},
            {"stamps": 8, "reward": "อาหารมูลค่าไม่เกิน 200 บาทฟรี"}
        ]
    },
    
    "beauty_salon": {
        "name": "Beauty Points Card",
        "max_stamps": 5,
        "stamp_trigger": "per_service",  # ได้แสตมป์ทุกครั้งที่ใช้บริการ
        "milestones": [
            {"stamps": 3, "reward": "ส่วนลด 20% สำหรับบริการครั้งถัดไป"},
            {"stamps": 5, "reward": "บริการ Facial ฟรี 1 ครั้ง (มูลค่า 500 บาท)"}
        ]
    }
}
```

### การคำนวณแสตมป์ตาม Purchase Amount

```python
def calculate_stamps_for_purchase(amount, rules):
    """
    คำนวณจำนวนแสตมป์จากยอดซื้อ
    
    Args:
        amount: ยอดซื้อ (บาท)
        rules: กฎการให้แสตมป์
    
    Returns:
        จำนวนแสตมป์ที่ได้รับ
    """
    
    rule_type = rules.get("type")
    
    if rule_type == "flat":
        # ได้ N แสตมป์ต่อการซื้อ 1 ครั้ง
        return rules["stamps_per_purchase"]
    
    elif rule_type == "per_amount":
        # ได้ 1 แสตมป์ทุก X บาท
        amount_per_stamp = rules["amount_per_stamp"]
        return int(amount // amount_per_stamp)
    
    elif rule_type == "tiered":
        # แสตมป์ตามระดับยอดซื้อ
        stamps = 0
        for tier in sorted(rules["tiers"], key=lambda x: x["min_amount"]):
            if amount >= tier["min_amount"]:
                stamps = tier["stamps"]
        return stamps
    
    elif rule_type == "bonus_multiplier":
        # แสตมป์ x Multiplier ในวันพิเศษ
        base_stamps = int(amount // rules["base_amount"])
        multiplier = rules.get("current_multiplier", 1)
        return int(base_stamps * multiplier)
    
    return 0

# ตัวอย่าง
rules_examples = {
    "flat": {"type": "flat", "stamps_per_purchase": 1},
    "per_100": {"type": "per_amount", "amount_per_stamp": 100},
    "tiered": {
        "type": "tiered",
        "tiers": [
            {"min_amount": 0, "stamps": 1},
            {"min_amount": 500, "stamps": 2},
            {"min_amount": 1000, "stamps": 3},
            {"min_amount": 2000, "stamps": 5}
        ]
    },
    "double_day": {
        "type": "bonus_multiplier",
        "base_amount": 100,
        "current_multiplier": 2  # Double Stamp Day!
    }
}

print(calculate_stamps_for_purchase(750, rules_examples["tiered"]))    # 2
print(calculate_stamps_for_purchase(750, rules_examples["double_day"])) # 14
```

---

## 13.7 จัดการคูปองที่ยังไม่ถูกใช้และหมดอายุ

### 13.7.1 การส่ง Reminder อัตโนมัติ

```python
import schedule
import time

class CouponReminderSystem:
    """
    ระบบส่ง Reminder คูปองอัตโนมัติ
    """
    
    def __init__(self, access_token, db):
        self.token = access_token
        self.db = db
    
    def send_expiry_reminders(self):
        """
        ส่ง Reminder สำหรับคูปองที่ใกล้หมดอายุ
        เรียกใช้ทุกวัน
        """
        
        cursor = self.db.cursor()
        now = datetime.now()
        
        # คูปองที่หมดอายุใน 3 วัน
        expiry_3days = now + timedelta(days=3)
        cursor.execute(
            """SELECT c.*, u.line_user_id 
               FROM coupons c
               JOIN user_coupons uc ON c.id = uc.coupon_id
               JOIN users u ON uc.user_id = u.id
               WHERE uc.used = 0 
               AND c.expiry_date BETWEEN ? AND ?
               AND uc.reminder_3day_sent = 0""",
            (now.isoformat(), expiry_3days.isoformat())
        )
        
        reminders_3day = cursor.fetchall()
        
        for row in reminders_3day:
            self._send_reminder(
                row["line_user_id"],
                row,
                reminder_type="3_day"
            )
            
            # Mark as sent
            cursor.execute(
                "UPDATE user_coupons SET reminder_3day_sent = 1 WHERE coupon_id = ? AND user_id = ?",
                (row["id"], row["user_id"])
            )
        
        # คูปองที่หมดอายุวันนี้
        tomorrow = now + timedelta(days=1)
        cursor.execute(
            """SELECT c.*, u.line_user_id 
               FROM coupons c
               JOIN user_coupons uc ON c.id = uc.coupon_id
               JOIN users u ON uc.user_id = u.id
               WHERE uc.used = 0 
               AND c.expiry_date BETWEEN ? AND ?
               AND uc.reminder_1day_sent = 0""",
            (now.isoformat(), tomorrow.isoformat())
        )
        
        reminders_1day = cursor.fetchall()
        
        for row in reminders_1day:
            self._send_reminder(
                row["line_user_id"],
                row,
                reminder_type="1_day"
            )
            
            cursor.execute(
                "UPDATE user_coupons SET reminder_1day_sent = 1 WHERE coupon_id = ? AND user_id = ?",
                (row["id"], row["user_id"])
            )
        
        self.db.commit()
    
    def _send_reminder(self, user_id, coupon, reminder_type):
        """ส่ง Reminder Message"""
        
        if reminder_type == "3_day":
            text = (
                f"⏰ คูปองของคุณใกล้หมดอายุแล้ว!\n\n"
                f"🎁 {coupon['title']}\n"
                f"📅 หมดอายุ: {coupon['expiry_date'][:10]}\n\n"
                f"อย่าลืมใช้ก่อนหมดอายุนะคะ 😊"
            )
        else:
            text = (
                f"🚨 คูปองของคุณจะหมดอายุวันนี้!\n\n"
                f"🎁 {coupon['title']}\n"
                f"⚡ โอกาสสุดท้ายแล้ว!\n\n"
                f"รีบใช้เลยก่อนที่จะสายเกินไป 🏃‍♀️"
            )
        
        import requests
        requests.post(
            "https://api.line.me/v2/bot/message/push",
            headers={
                "Authorization": f"Bearer {self.token}",
                "Content-Type": "application/json"
            },
            json={
                "to": user_id,
                "messages": [{"type": "text", "text": text}]
            }
        )

# ตั้งเวลาส่ง Reminder ทุกวันตอน 10:00
# reminder_system = CouponReminderSystem(token, db)
# schedule.every().day.at("10:00").do(reminder_system.send_expiry_reminders)
```

### 13.7.2 การจัดการคูปองที่หมดอายุ

```python
def process_expired_coupons(db):
    """
    จัดการคูปองที่หมดอายุ
    - Mark คูปองว่า Expired
    - เก็บ Stats
    - ส่ง Notification ถ้าจำเป็น
    """
    
    cursor = db.cursor()
    now = datetime.now()
    
    # ค้นหาคูปองที่หมดอายุแต่ยังไม่ได้ Mark
    cursor.execute(
        """SELECT c.*, COUNT(uc.id) as unused_count
           FROM coupons c
           LEFT JOIN user_coupons uc ON c.id = uc.coupon_id AND uc.used = 0
           WHERE c.expiry_date < ? AND c.status = 'active'
           GROUP BY c.id""",
        (now.isoformat(),)
    )
    
    expired_coupons = cursor.fetchall()
    
    for coupon in expired_coupons:
        # อัปเดตสถานะ
        cursor.execute(
            "UPDATE coupons SET status = 'expired' WHERE id = ?",
            (coupon["id"],)
        )
        
        # บันทึก Stats
        cursor.execute(
            """INSERT INTO coupon_stats 
               (coupon_id, expired_at, unused_count, total_used)
               VALUES (?, ?, ?, ?)""",
            (
                coupon["id"],
                now.isoformat(),
                coupon["unused_count"],
                coupon["total_limit"] - coupon["unused_count"]
            )
        )
        
        print(f"Expired: {coupon['title']} - Unused: {coupon['unused_count']}")
    
    db.commit()
    
    return {
        "processed": len(expired_coupons),
        "total_unused": sum(c["unused_count"] for c in expired_coupons)
    }
```

---

## 13.8 การรวมระบบคูปองกับโปรโมชั่น

### Campaign Planning Framework

```python
class PromotionCampaign:
    """
    จัดการ Campaign ที่รวม Coupon + Reward
    """
    
    def __init__(self, campaign_config):
        self.config = campaign_config
    
    def create_campaign_bundle(self):
        """
        สร้างชุดข้อความ Campaign
        """
        
        messages = []
        
        # 1. Announcement Message
        if self.config.get("announcement"):
            messages.append({
                "type": "text",
                "text": self.config["announcement"]
            })
        
        # 2. Rich Message (ภาพโปรโมชั่น)
        if self.config.get("promo_image"):
            messages.append({
                "type": "imagemap",
                "baseUrl": self.config["promo_image"]["url"],
                "altText": self.config["promo_image"]["alt_text"],
                "baseSize": {"width": 1040, "height": 1040},
                "actions": self.config["promo_image"].get("actions", [])
            })
        
        # 3. Product Cards
        if self.config.get("products"):
            product_bubbles = [create_product_card(p) for p in self.config["products"]]
            messages.append({
                "type": "flex",
                "altText": "สินค้าในโปรโมชั่น",
                "contents": {
                    "type": "carousel",
                    "contents": product_bubbles
                }
            })
        
        # 4. Coupon (ถ้ามี)
        if self.config.get("coupon"):
            messages.append({
                "type": "flex",
                "altText": f"คูปองพิเศษ: {self.config['coupon']['code']}",
                "contents": self._build_coupon_card(self.config["coupon"])
            })
        
        return messages[:5]  # LINE จำกัด 5 messages
    
    def _build_coupon_card(self, coupon):
        """สร้าง Coupon Card Bubble"""
        return {
            "type": "bubble",
            "body": {
                "type": "box",
                "layout": "vertical",
                "contents": [
                    {"type": "text", "text": coupon["title"], "weight": "bold"},
                    {"type": "text", "text": f"CODE: {coupon['code']}", "color": "#1DB446"}
                ]
            }
        }

# ตัวอย่าง Campaign Config
summer_campaign = {
    "name": "Summer Sale 2024",
    "announcement": "☀️ Summer Sale เริ่มแล้ว! ลดสูงสุด 70%\nช้อปได้ตั้งแต่วันนี้ - 31 ก.ค. นี้",
    "promo_image": {
        "url": "https://cdn.example.com/summer-sale",
        "alt_text": "Summer Sale ลด 70%",
        "actions": [
            {
                "type": "uri",
                "linkUri": "https://shop.example.com/summer-sale",
                "area": {"x": 0, "y": 0, "width": 1040, "height": 1040}
            }
        ]
    },
    "products": [
        {"name": "ชุดว่ายน้ำ", "price": 490, "original_price": 990, "image_url": "...", "buy_url": "..."},
        {"name": "แว่นกันแดด", "price": 290, "original_price": 590, "image_url": "...", "buy_url": "..."}
    ],
    "coupon": {
        "title": "ลดเพิ่ม 10% สำหรับลูกค้า LINE",
        "code": "SUMMER10",
        "description": "ใช้ได้กับสินค้า Summer Collection ทุกรายการ",
        "expiry_date": "2024-07-31",
        "use_url": "https://shop.example.com?coupon=SUMMER10"
    }
}

campaign = PromotionCampaign(summer_campaign)
campaign_messages = campaign.create_campaign_bundle()
```

---

## 13.9 กลยุทธ์ Loyalty Program

### ระดับสมาชิก (Membership Tiers)

```python
membership_tiers = {
    "bronze": {
        "name": "Bronze Member",
        "icon": "🥉",
        "min_points": 0,
        "max_points": 999,
        "benefits": [
            "Birthday Coupon 10% off",
            "Early access to sales",
            "1 point per ฿10 spent"
        ],
        "stamp_multiplier": 1.0
    },
    "silver": {
        "name": "Silver Member",
        "icon": "🥈",
        "min_points": 1000,
        "max_points": 4999,
        "benefits": [
            "Birthday Coupon 15% off",
            "Free shipping on all orders",
            "1.5 points per ฿10 spent",
            "Exclusive member deals"
        ],
        "stamp_multiplier": 1.5
    },
    "gold": {
        "name": "Gold Member",
        "icon": "🥇",
        "min_points": 5000,
        "max_points": 14999,
        "benefits": [
            "Birthday Coupon 20% off",
            "Free express shipping",
            "2 points per ฿10 spent",
            "Priority customer service",
            "Monthly surprise gift"
        ],
        "stamp_multiplier": 2.0
    },
    "platinum": {
        "name": "Platinum Member",
        "icon": "💎",
        "min_points": 15000,
        "benefits": [
            "Birthday Coupon 25% off",
            "Free same-day delivery",
            "3 points per ฿10 spent",
            "Dedicated account manager",
            "VIP event invitations",
            "Quarterly premium gift"
        ],
        "stamp_multiplier": 3.0
    }
}

def get_member_tier(points):
    """ดึง Tier ของสมาชิกจาก Points"""
    for tier_name, tier in sorted(
        membership_tiers.items(),
        key=lambda x: x[1]["min_points"],
        reverse=True
    ):
        if points >= tier["min_points"]:
            return tier_name, tier
    return "bronze", membership_tiers["bronze"]

def send_tier_upgrade_notification(user_id, new_tier, access_token):
    """ส่งแจ้งเตือนเมื่อ Upgrade Tier"""
    
    tier = membership_tiers[new_tier]
    
    bubble = {
        "type": "bubble",
        "header": {
            "type": "box",
            "layout": "vertical",
            "backgroundColor": "#FFD700" if new_tier == "gold" else "#C0C0C0",
            "paddingAll": "20px",
            "contents": [
                {
                    "type": "text",
                    "text": "🎉 ยินดีด้วย!",
                    "color": "#FFFFFF",
                    "weight": "bold",
                    "size": "xl"
                },
                {
                    "type": "text",
                    "text": f"คุณได้รับการอัปเกรดเป็น {tier['icon']} {tier['name']}",
                    "color": "#FFFFFF",
                    "size": "lg",
                    "wrap": True
                }
            ]
        },
        "body": {
            "type": "box",
            "layout": "vertical",
            "contents": [
                {"type": "text", "text": "สิทธิประโยชน์ที่คุณได้รับ:", "weight": "bold"},
                *[
                    {"type": "text", "text": f"✓ {benefit}", "size": "sm", "color": "#555555", "margin": "xs"}
                    for benefit in tier["benefits"]
                ]
            ]
        }
    }
    
    import requests
    requests.post(
        "https://api.line.me/v2/bot/message/push",
        headers={"Authorization": f"Bearer {access_token}", "Content-Type": "application/json"},
        json={"to": user_id, "messages": [{"type": "flex", "altText": f"ยินดีด้วย! คุณเป็น {tier['name']} แล้ว!", "contents": bubble}]}
    )
```

---

## 13.10 Analytics สำหรับ Coupon & Rewards

### KPIs สำคัญ

```python
def calculate_coupon_analytics(coupon_data):
    """
    คำนวณ Analytics สำหรับ Coupon Campaign
    """
    
    total_issued = coupon_data.get("total_issued", 0)
    total_redeemed = coupon_data.get("total_redeemed", 0)
    total_revenue_with_coupon = coupon_data.get("revenue_with_coupon", 0)
    total_revenue_without_coupon = coupon_data.get("revenue_without_coupon", 0)
    total_discount_given = coupon_data.get("total_discount", 0)
    
    metrics = {
        # Redemption Rate
        "redemption_rate": (total_redeemed / total_issued * 100) if total_issued > 0 else 0,
        
        # Revenue Lift
        "revenue_lift": total_revenue_with_coupon - total_revenue_without_coupon,
        
        # ROI ของ Coupon Campaign
        "campaign_roi": (
            (total_revenue_with_coupon - total_discount_given) / total_discount_given * 100
        ) if total_discount_given > 0 else 0,
        
        # Average Discount per Redemption
        "avg_discount": total_discount_given / total_redeemed if total_redeemed > 0 else 0,
        
        # Average Order Value with Coupon
        "aov_with_coupon": total_revenue_with_coupon / total_redeemed if total_redeemed > 0 else 0
    }
    
    # Benchmark Comparison
    benchmarks = {
        "good_redemption_rate": 15,    # > 15% = ดี
        "excellent_redemption_rate": 30 # > 30% = ยอดเยี่ยม
    }
    
    insights = []
    
    if metrics["redemption_rate"] < 5:
        insights.append("Redemption Rate ต่ำมาก - ลองส่ง Reminder หรือ ปรับ Offer ให้น่าสนใจขึ้น")
    elif metrics["redemption_rate"] > 30:
        insights.append("Redemption Rate ดีมาก! Campaign นี้ประสบความสำเร็จ")
    
    if metrics["campaign_roi"] < 100:
        insights.append("ROI ต่ำกว่า 100% - ต้นทุนส่วนลดสูงกว่ารายได้ที่เพิ่มขึ้น")
    elif metrics["campaign_roi"] > 300:
        insights.append("ROI สูงมาก! Coupon Campaign นี้คุ้มค่ามาก")
    
    return {**metrics, "insights": insights}

# ตัวอย่าง
coupon_stats = {
    "total_issued": 1000,
    "total_redeemed": 234,
    "revenue_with_coupon": 351000,   # บาท
    "revenue_without_coupon": 200000,
    "total_discount": 23400          # บาท
}

analytics = calculate_coupon_analytics(coupon_stats)
print(f"Redemption Rate: {analytics['redemption_rate']:.1f}%")
print(f"Revenue Lift: ฿{analytics['revenue_lift']:,.0f}")
print(f"Campaign ROI: {analytics['campaign_roi']:.0f}%")
for insight in analytics["insights"]:
    print(f"💡 {insight}")
```

---

## 13.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Birthday Coupon System

**โจทย์:** สร้างระบบที่ส่ง Birthday Coupon อัตโนมัติให้ลูกค้าในวันเกิด

```python
def birthday_coupon_daily_job(users, access_token):
    """
    TODO: สร้างฟังก์ชันนี้
    1. ค้นหาผู้ใช้ที่วันเกิดตรงกับวันนี้
    2. สร้าง Unique Coupon Code สำหรับแต่ละคน
    3. ส่ง Coupon Card ที่มีชื่อผู้ใช้
    
    Hints:
    - รูปแบบ Coupon Code: BD{USER_ID_LAST4}{YEAR}
    - Coupon มีอายุ 7 วัน
    - ส่วนลด 15% สำหรับทุกคน (หรือ 20% สำหรับ VIP)
    """
    pass

# Test data
test_users = [
    {"user_id": "Uabc1234", "name": "คุณสมชาย", "birthday": "1990-09-30", "is_vip": False},
    {"user_id": "Udef5678", "name": "คุณสมหญิง", "birthday": "1985-09-30", "is_vip": True}
]
```

### แบบฝึกหัดที่ 2: Stamp Card Integration

**โจทย์:** สร้างระบบที่รับ Webhook เมื่อมีการซื้อสินค้าและเพิ่มแสตมป์อัตโนมัติ

```python
def handle_purchase_webhook(order_data, stamp_manager):
    """
    TODO: เมื่อมีการซื้อสินค้า
    1. ดึง User ID จาก order_data
    2. คำนวณจำนวนแสตมป์จากยอดซื้อ (ทุก 100 บาท = 1 แสตมป์)
    3. เพิ่มแสตมป์
    4. ถ้าได้รับรางวัล ส่งแจ้งเตือนทันที
    5. ส่ง Status Update ของ Stamp Card
    """
    pass
```

---

## 13.12 สรุปบทที่ 13

### Key Takeaways

1. **Coupon Types**: Discount, Gift, Special แต่ละแบบเหมาะกับวัตถุประสงค์ต่างกัน
2. **Reward Card**: ระบบสะสมแสตมป์ดิจิทัลสร้าง Customer Loyalty ได้ดี
3. **Automation**: ส่ง Reminder คูปองหมดอายุและวันเกิดอัตโนมัติ
4. **Analytics**: วัด Redemption Rate, ROI และ Revenue Lift เสมอ
5. **Tier System**: แบ่งระดับสมาชิกเพื่อ Personalize ประสบการณ์

### Coupon Best Practices

```
✓ ส่วนลดควรน่าสนใจแต่ไม่ทำให้ขาดทุน
✓ Expiry Date ควรสั้นพอที่สร้าง Urgency (7-14 วัน)
✓ เงื่อนไขต้องชัดเจน ไม่ซับซ้อน
✓ ส่ง Reminder 3 วันและ 1 วันก่อนหมดอายุ
✓ ติดตาม Redemption Rate ทุก Campaign
✓ A/B Test ระหว่าง % vs. จำนวนเงิน
```

---

*บทถัดไป: Part 14 - Timeline Posts - สร้าง Content Strategy บน LINE OA Timeline*
