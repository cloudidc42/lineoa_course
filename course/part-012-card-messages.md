# Part 12: Card Message - นำเสนอสินค้าและบริการอย่างมืออาชีพ

## บทนำ

Card Message เป็นรูปแบบข้อความที่ทรงพลังสำหรับการนำเสนอสินค้าหรือบริการหลายรายการในข้อความเดียว มี 3 ประเภทหลัก ได้แก่ Product Card, Image Carousel และ Coupon Card ซึ่งแต่ละประเภทเหมาะกับการใช้งานที่แตกต่างกัน บทนี้จะครอบคลุมการสร้าง ปรับแต่ง และวิเคราะห์ผลลัพธ์ของ Card Message อย่างครบถ้วน

---

## 12.1 ประเภทของ Card Message

### ภาพรวมประเภท Card Message

| ประเภท | คำอธิบาย | การใช้งาน |
|--------|----------|-----------|
| Product Card | แสดงสินค้าพร้อมราคาและปุ่มซื้อ | E-commerce, ร้านค้าออนไลน์ |
| Image Carousel | สไลด์ภาพหลายรูปในหน้าต่างเดียว | Portfolio, Gallery, สินค้าหลายชิ้น |
| Coupon Card | คูปองส่วนลดหรือข้อเสนอพิเศษ | โปรโมชั่น, Flash Sale |

### เปรียบเทียบ Card Types

| คุณสมบัติ | Product Card | Image Carousel | Coupon Card |
|-----------|-------------|----------------|-------------|
| จำนวน Card/Message | 1-10 | 1-10 | 1-10 |
| รูปภาพ | 1 ต่อ Card | 1 ต่อ Card | 1 ต่อ Card |
| ราคาสินค้า | ✓ | ✗ | ✗ |
| รหัสคูปอง | ✗ | ✗ | ✓ |
| ปุ่ม Action | สูงสุด 3 ปุ่ม | สูงสุด 3 ปุ่ม | สูงสุด 3 ปุ่ม |
| ข้อความ | ✓ | ✓ | ✓ |

---

## 12.2 Product Card (การ์ดสินค้า)

### โครงสร้างของ Product Card

```
┌─────────────────────────────┐
│                             │
│         [รูปสินค้า]          │
│                             │
├─────────────────────────────┤
│ ชื่อสินค้า                    │
│ คำอธิบายสั้นๆ                 │
│                             │
│ ฿ 299 ~~฿ 599~~             │
│                             │
│ [ซื้อเลย]  [ดูรายละเอียด]      │
└─────────────────────────────┘
```

### ข้อกำหนดภาพสำหรับ Product Card

| คุณสมบัติ | ข้อกำหนด |
|-----------|---------|
| ขนาดแนะนำ | 1024 x 1024 px |
| Aspect Ratio | 1:1 (Square) |
| ขนาดไฟล์ | สูงสุด 10 MB |
| Format | JPEG, PNG |
| ขนาดต่ำสุด | 512 x 512 px |

### การสร้าง Product Card ใน LINE OA Manager

#### ขั้นตอนการสร้าง

```
1. เข้า LINE OA Manager
2. คลิก "Messages" > "Card Message"
3. คลิก "+ Create"
4. เลือกประเภท "Product Card"
5. กำหนดรายละเอียด:
   - Title: ชื่อสินค้า (สูงสุด 100 ตัวอักษร)
   - Description: คำอธิบาย (สูงสุด 400 ตัวอักษร)
   - Image: อัปโหลดภาพ
   - Price: ราคาสินค้า
   - Original Price: ราคาเดิม (ถ้ามี)
   - Currency: สกุลเงิน (฿ THB)
6. เพิ่ม Action Buttons (สูงสุด 3 ปุ่ม)
7. Preview และ Save
```

### Product Card ผ่าน Flex Message API

```python
def create_product_card(product):
    """
    สร้าง Product Card จากข้อมูลสินค้า
    
    Args:
        product: dict ที่มี name, description, price, original_price, image_url, buy_url
    """
    
    card = {
        "type": "bubble",
        "hero": {
            "type": "image",
            "url": product["image_url"],
            "size": "full",
            "aspectRatio": "1:1",
            "aspectMode": "cover",
            "action": {
                "type": "uri",
                "uri": product["buy_url"]
            }
        },
        "body": {
            "type": "box",
            "layout": "vertical",
            "contents": [
                {
                    "type": "text",
                    "text": product["name"],
                    "weight": "bold",
                    "size": "xl",
                    "wrap": True
                },
                {
                    "type": "text",
                    "text": product["description"],
                    "size": "sm",
                    "color": "#666666",
                    "wrap": True,
                    "margin": "sm"
                },
                {
                    "type": "box",
                    "layout": "baseline",
                    "margin": "md",
                    "contents": [
                        {
                            "type": "text",
                            "text": f"฿{product['price']:,.0f}",
                            "weight": "bold",
                            "size": "xl",
                            "color": "#e74c3c",
                            "flex": 1
                        }
                    ]
                }
            ]
        },
        "footer": {
            "type": "box",
            "layout": "horizontal",
            "spacing": "sm",
            "contents": [
                {
                    "type": "button",
                    "style": "primary",
                    "action": {
                        "type": "uri",
                        "label": "ซื้อเลย",
                        "uri": product["buy_url"]
                    },
                    "color": "#1DB446",
                    "flex": 1
                },
                {
                    "type": "button",
                    "style": "secondary",
                    "action": {
                        "type": "uri",
                        "label": "ดูรายละเอียด",
                        "uri": product["detail_url"]
                    },
                    "flex": 1
                }
            ]
        }
    }
    
    # เพิ่ม Original Price ถ้ามี
    if product.get("original_price"):
        original_price_text = {
            "type": "text",
            "text": f"฿{product['original_price']:,.0f}",
            "size": "sm",
            "color": "#999999",
            "decoration": "line-through",
            "margin": "xs"
        }
        card["body"]["contents"][2]["contents"].append(original_price_text)
    
    return card

# ตัวอย่างการใช้งาน
product_data = {
    "name": "กระเป๋าหนังแท้ Premium",
    "description": "กระเป๋าหนังแท้คุณภาพสูง ทนทาน ใช้งานได้ทุกโอกาส มีให้เลือก 5 สี",
    "price": 1290,
    "original_price": 2590,
    "image_url": "https://cdn.example.com/products/bag-001.jpg",
    "buy_url": "https://shop.example.com/buy/bag-001",
    "detail_url": "https://shop.example.com/product/bag-001"
}

card = create_product_card(product_data)
```

### การสร้าง Product Card Carousel (หลายสินค้า)

```python
import requests
import json

def send_product_carousel(access_token, to, products):
    """
    ส่ง Product Cards หลายรายการในรูปแบบ Carousel
    
    Args:
        access_token: LINE Channel Access Token
        to: User ID
        products: List ของ product dictionaries (สูงสุด 10 รายการ)
    """
    
    # จำกัดจำนวน products ไม่เกิน 10 รายการ
    products = products[:10]
    
    # สร้าง Card สำหรับแต่ละสินค้า
    bubbles = [create_product_card(p) for p in products]
    
    message = {
        "type": "flex",
        "altText": f"สินค้าแนะนำ {len(products)} รายการ",
        "contents": {
            "type": "carousel",
            "contents": bubbles
        }
    }
    
    payload = {
        "to": to,
        "messages": [message]
    }
    
    response = requests.post(
        "https://api.line.me/v2/bot/message/push",
        headers={
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json"
        },
        json=payload
    )
    
    return response.json()

# ตัวอย่างข้อมูลสินค้าหลายรายการ
products = [
    {
        "name": "เสื้อเชิ้ตผ้าลินิน",
        "description": "เสื้อเชิ้ตผ้าลินินแท้ ระบายอากาศดี เหมาะกับอากาศร้อน",
        "price": 890,
        "original_price": 1290,
        "image_url": "https://cdn.example.com/shirt.jpg",
        "buy_url": "https://shop.example.com/buy/shirt",
        "detail_url": "https://shop.example.com/product/shirt"
    },
    {
        "name": "กางเกงขายาว Slim Fit",
        "description": "กางเกงขายาวทรง Slim Fit ผ้าสีดำ ใส่ได้ทั้งทำงานและไปเที่ยว",
        "price": 690,
        "original_price": 990,
        "image_url": "https://cdn.example.com/pants.jpg",
        "buy_url": "https://shop.example.com/buy/pants",
        "detail_url": "https://shop.example.com/product/pants"
    },
    {
        "name": "รองเท้าผ้าใบ Sport",
        "description": "รองเท้าผ้าใบน้ำหนักเบา เหมาะสำหรับออกกำลังกายและทำกิจกรรม",
        "price": 1490,
        "original_price": 2200,
        "image_url": "https://cdn.example.com/shoes.jpg",
        "buy_url": "https://shop.example.com/buy/shoes",
        "detail_url": "https://shop.example.com/product/shoes"
    }
]

result = send_product_carousel("YOUR_TOKEN", "Uxxxxxxxx", products)
```

---

## 12.3 Image Carousel

### ภาพรวม Image Carousel

Image Carousel เป็นรูปแบบที่แสดงภาพหลายรูปในลักษณะ Slide ที่ผู้ใช้สามารถเลื่อนดูได้ เหมาะสำหรับการแสดงผลงาน, Collection หรือ Category ต่างๆ

### โครงสร้าง Image Carousel

```
┌──────────────────────────────────────────┐
│  ◀  ┌─────────────┐  ┌─────────────┐  ▶  │
│     │   [ภาพ 1]   │  │   [ภาพ 2]   │     │
│     │             │  │             │     │
│     │   ชื่อ 1    │  │   ชื่อ 2    │     │
│     │  คำอธิบาย 1  │  │  คำอธิบาย 2  │     │
│     │  [ปุ่ม 1]   │  │  [ปุ่ม 1]   │     │
│     └─────────────┘  └─────────────┘     │
└──────────────────────────────────────────┘
```

### ข้อกำหนดของ Image Carousel

| คุณสมบัติ | ข้อกำหนด |
|-----------|---------|
| จำนวน Card สูงสุด | 10 |
| ขนาดภาพแนะนำ | 1024 x 1024 px |
| Aspect Ratio | 1:1 (ทุก Card ต้องเท่ากัน) |
| ขนาดไฟล์สูงสุด | 10 MB |
| จำนวนปุ่ม | สูงสุด 3 ปุ่ม |
| ความยาว Title | สูงสุด 40 ตัวอักษร |
| ความยาว Text | สูงสุด 120 ตัวอักษร |

### สร้าง Image Carousel ผ่าน Flex Message

```python
def create_image_carousel(items):
    """
    สร้าง Image Carousel
    
    Args:
        items: List ของ item dict ที่มี title, description, image_url, buttons
    """
    
    def create_carousel_item(item):
        """สร้าง Bubble สำหรับแต่ละ item"""
        
        # สร้าง Buttons
        button_contents = []
        for btn in item.get("buttons", [])[:3]:  # จำกัด 3 ปุ่ม
            button = {
                "type": "button",
                "action": {
                    "type": btn.get("type", "uri"),
                    "label": btn["label"]
                }
            }
            
            if btn.get("type") == "uri":
                button["action"]["uri"] = btn["url"]
                button["style"] = "primary"
                button["color"] = btn.get("color", "#1DB446")
            elif btn.get("type") == "message":
                button["action"]["text"] = btn["text"]
                button["style"] = "secondary"
            
            button_contents.append(button)
        
        bubble = {
            "type": "bubble",
            "hero": {
                "type": "image",
                "url": item["image_url"],
                "size": "full",
                "aspectRatio": "1:1",
                "aspectMode": "cover"
            },
            "body": {
                "type": "box",
                "layout": "vertical",
                "contents": [
                    {
                        "type": "text",
                        "text": item.get("title", ""),
                        "weight": "bold",
                        "size": "lg",
                        "wrap": True
                    },
                    {
                        "type": "text",
                        "text": item.get("description", ""),
                        "size": "sm",
                        "color": "#555555",
                        "wrap": True,
                        "margin": "sm"
                    }
                ]
            }
        }
        
        if button_contents:
            bubble["footer"] = {
                "type": "box",
                "layout": "vertical",
                "spacing": "sm",
                "contents": button_contents
            }
        
        return bubble
    
    # สร้าง Carousel
    carousel = {
        "type": "flex",
        "altText": f"ดูทั้งหมด {len(items)} รายการ",
        "contents": {
            "type": "carousel",
            "contents": [create_carousel_item(item) for item in items[:10]]
        }
    }
    
    return carousel

# ตัวอย่างการใช้งาน - Category Showcase
categories = [
    {
        "title": "แฟชั่นผู้หญิง",
        "description": "เสื้อผ้าผู้หญิงรุ่นใหม่ล่าสุด มีให้เลือกหลากหลายสไตล์",
        "image_url": "https://cdn.example.com/women-fashion.jpg",
        "buttons": [
            {"label": "ดูคอลเลคชั่น", "type": "uri", "url": "https://shop.com/women"},
            {"label": "New Arrivals", "type": "uri", "url": "https://shop.com/women/new", "color": "#FF6B6B"}
        ]
    },
    {
        "title": "แฟชั่นผู้ชาย",
        "description": "เสื้อผ้าผู้ชายสไตล์ Modern ใส่สบาย ดูดี",
        "image_url": "https://cdn.example.com/men-fashion.jpg",
        "buttons": [
            {"label": "ดูคอลเลคชั่น", "type": "uri", "url": "https://shop.com/men"},
            {"label": "Best Sellers", "type": "uri", "url": "https://shop.com/men/best"}
        ]
    },
    {
        "title": "กระเป๋าและเครื่องประดับ",
        "description": "กระเป๋าแบรนด์ดัง เครื่องประดับคุณภาพสูง",
        "image_url": "https://cdn.example.com/accessories.jpg",
        "buttons": [
            {"label": "ช้อปเลย", "type": "uri", "url": "https://shop.com/accessories"}
        ]
    }
]

carousel_message = create_image_carousel(categories)
```

### Image Carousel สำหรับ Portfolio / Gallery

```python
def create_portfolio_carousel(portfolio_items):
    """
    สร้าง Portfolio Carousel สำหรับธุรกิจบริการ
    """
    
    def create_portfolio_bubble(item):
        return {
            "type": "bubble",
            "hero": {
                "type": "image",
                "url": item["image_url"],
                "size": "full",
                "aspectRatio": "1:1",
                "aspectMode": "cover",
                "action": {
                    "type": "uri",
                    "uri": item.get("detail_url", "#")
                }
            },
            "body": {
                "type": "box",
                "layout": "vertical",
                "paddingAll": "sm",
                "contents": [
                    {
                        "type": "text",
                        "text": item["title"],
                        "weight": "bold",
                        "size": "md",
                        "wrap": True
                    },
                    {
                        "type": "box",
                        "layout": "horizontal",
                        "margin": "xs",
                        "contents": [
                            {
                                "type": "text",
                                "text": f"📍 {item.get('location', '')}",
                                "size": "xs",
                                "color": "#888888",
                                "flex": 1
                            },
                            {
                                "type": "text",
                                "text": item.get("year", ""),
                                "size": "xs",
                                "color": "#888888",
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
                        "action": {
                            "type": "uri",
                            "label": "ดูผลงาน",
                            "uri": item.get("detail_url", "#")
                        }
                    }
                ]
            }
        }
    
    return {
        "type": "flex",
        "altText": "ผลงานของเรา",
        "contents": {
            "type": "carousel",
            "contents": [create_portfolio_bubble(p) for p in portfolio_items[:10]]
        }
    }
```

---

## 12.4 Coupon Card

### ภาพรวม Coupon Card

Coupon Card ใช้สำหรับแจกคูปองส่วนลดหรือข้อเสนอพิเศษ มีรูปแบบที่ชัดเจนและดึงดูดใจ

### โครงสร้าง Coupon Card

```
┌─────────────────────────────┐
│         [รูปคูปอง]           │
├─────────────────────────────┤
│ ชื่อโปรโมชั่น                  │
│ รายละเอียดเงื่อนไข              │
├- - - - - - - - - - - - - - -┤
│ รหัสคูปอง: SAVE50           │
│ หมดอายุ: 31 ธ.ค. 2567       │
├─────────────────────────────┤
│ [ใช้คูปองเลย]  [แชร์ให้เพื่อน]  │
└─────────────────────────────┘
```

### สร้าง Coupon Card ผ่าน Flex Message

```python
from datetime import datetime, timedelta

def create_coupon_card(coupon_data):
    """
    สร้าง Coupon Card
    
    Args:
        coupon_data: dict ที่มี title, description, code, discount,
                     expiry_date, conditions, use_url, share_url
    """
    
    expiry = coupon_data.get("expiry_date", "")
    if isinstance(expiry, datetime):
        expiry = expiry.strftime("%d %b %Y")
    
    bubble = {
        "type": "bubble",
        "size": "mega",
        "hero": {
            "type": "image",
            "url": coupon_data.get("image_url", "https://example.com/coupon-bg.jpg"),
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
                    "text": coupon_data["title"],
                    "weight": "bold",
                    "size": "xl",
                    "color": "#e74c3c",
                    "wrap": True
                },
                {
                    "type": "text",
                    "text": coupon_data.get("description", ""),
                    "size": "sm",
                    "color": "#555555",
                    "wrap": True,
                    "margin": "sm"
                },
                {
                    "type": "separator",
                    "margin": "md",
                    "color": "#DDDDDD"
                },
                {
                    "type": "box",
                    "layout": "vertical",
                    "margin": "md",
                    "paddingAll": "md",
                    "backgroundColor": "#F8F8F8",
                    "cornerRadius": "md",
                    "contents": [
                        {
                            "type": "box",
                            "layout": "horizontal",
                            "contents": [
                                {
                                    "type": "text",
                                    "text": "รหัสคูปอง:",
                                    "size": "sm",
                                    "color": "#888888",
                                    "flex": 0
                                },
                                {
                                    "type": "text",
                                    "text": coupon_data.get("code", ""),
                                    "size": "lg",
                                    "weight": "bold",
                                    "color": "#1DB446",
                                    "flex": 1,
                                    "align": "end"
                                }
                            ]
                        },
                        {
                            "type": "box",
                            "layout": "horizontal",
                            "margin": "sm",
                            "contents": [
                                {
                                    "type": "text",
                                    "text": "หมดอายุ:",
                                    "size": "xs",
                                    "color": "#888888",
                                    "flex": 0
                                },
                                {
                                    "type": "text",
                                    "text": expiry,
                                    "size": "xs",
                                    "color": "#e74c3c",
                                    "flex": 1,
                                    "align": "end"
                                }
                            ]
                        }
                    ]
                }
            ]
        },
        "footer": {
            "type": "box",
            "layout": "horizontal",
            "spacing": "sm",
            "contents": [
                {
                    "type": "button",
                    "style": "primary",
                    "action": {
                        "type": "uri",
                        "label": "ใช้คูปองเลย",
                        "uri": coupon_data.get("use_url", "#")
                    },
                    "color": "#e74c3c",
                    "flex": 2
                },
                {
                    "type": "button",
                    "style": "secondary",
                    "action": {
                        "type": "uri",
                        "label": "แชร์",
                        "uri": coupon_data.get("share_url", "#")
                    },
                    "flex": 1
                }
            ]
        }
    }
    
    return bubble

# ตัวอย่างการใช้งาน
coupon = {
    "title": "ลด 200 บาท! สำหรับการซื้อขั้นต่ำ 999 บาท",
    "description": "ใช้ได้กับสินค้าทุกหมวดหมู่ ยกเว้นสินค้า Flash Deal",
    "code": "SAVE200",
    "discount": 200,
    "image_url": "https://cdn.example.com/coupon-banner.jpg",
    "expiry_date": datetime.now() + timedelta(days=7),
    "use_url": "https://shop.example.com/checkout?coupon=SAVE200",
    "share_url": "https://shop.example.com/share/coupon/SAVE200"
}

coupon_card = create_coupon_card(coupon)
```

---

## 12.5 การรวม Card Types หลายแบบ

### Multiple Card Combinations

ในข้อความเดียวสามารถส่ง Card หลายประเภทรวมกันได้

```python
def send_combined_messages(access_token, user_id, products, coupon):
    """
    ส่งข้อความผสมที่มีทั้ง Product Cards และ Coupon Card
    """
    import requests
    
    messages = []
    
    # Message 1: แนะนำสินค้า (Carousel)
    if products:
        product_bubbles = [create_product_card(p) for p in products[:5]]
        messages.append({
            "type": "flex",
            "altText": f"สินค้าแนะนำ {len(product_bubbles)} รายการ",
            "contents": {
                "type": "carousel",
                "contents": product_bubbles
            }
        })
    
    # Message 2: Coupon Card
    if coupon:
        messages.append({
            "type": "flex",
            "altText": f"คูปองส่วนลด: {coupon.get('code', '')}",
            "contents": create_coupon_card(coupon)
        })
    
    # ส่งข้อความ (LINE จำกัด 5 messages ต่อ request)
    payload = {
        "to": user_id,
        "messages": messages[:5]
    }
    
    response = requests.post(
        "https://api.line.me/v2/bot/message/push",
        headers={
            "Authorization": f"Bearer {access_token}",
            "Content-Type": "application/json"
        },
        json=payload
    )
    
    return response.status_code == 200
```

### Strategy: Product Discovery Flow

```
ขั้นตอน:
1. ผู้ใช้ส่ง "ดูสินค้า" หรือกด Rich Menu
2. Bot ตอบด้วย Image Carousel แสดง Categories
3. ผู้ใช้เลือก Category ที่สนใจ
4. Bot ตอบด้วย Product Card Carousel
5. ผู้ใช้เลือกสินค้าและคลิก "ซื้อเลย"
6. เปิดหน้า Checkout พร้อม Coupon อัตโนมัติ

Implementation:
```

```python
# Webhook Handler สำหรับ Product Discovery Flow
def handle_product_discovery(event, access_token):
    """
    จัดการ Flow การค้นหาสินค้า
    """
    user_id = event["source"]["userId"]
    message_text = event.get("message", {}).get("text", "")
    
    if message_text == "ดูสินค้า" or message_text == "browse":
        # ส่ง Category Carousel
        categories = get_product_categories()  # ดึงจาก Database
        carousel = create_image_carousel([{
            "title": cat["name"],
            "description": cat["description"],
            "image_url": cat["image_url"],
            "buttons": [
                {
                    "label": "ดูสินค้า",
                    "type": "message",
                    "text": f"category:{cat['id']}"
                }
            ]
        } for cat in categories])
        
        send_message(access_token, user_id, carousel)
    
    elif message_text.startswith("category:"):
        category_id = message_text.split(":")[1]
        
        # ส่ง Product Carousel
        products = get_products_by_category(category_id)
        product_messages = [create_product_card(p) for p in products]
        
        carousel = {
            "type": "flex",
            "altText": f"สินค้าในหมวดหมู่",
            "contents": {
                "type": "carousel",
                "contents": product_messages
            }
        }
        
        send_message(access_token, user_id, carousel)
```

---

## 12.6 Call-to-Action Buttons

### ประเภท CTA Button ที่นิยม

| ชื่อปุ่ม | Action Type | ใช้งาน |
|---------|------------|-------|
| ซื้อเลย / Buy Now | URI | ไปหน้า Checkout |
| ดูรายละเอียด | URI | ไปหน้า Product Detail |
| หยิบใส่ตะกร้า | Message | ส่ง Text ไปยัง Bot |
| โทรหาเรา | URI (tel:) | โทรศัพท์ |
| นัดหมาย | URI | ไปหน้าจองนัด |
| แชร์ให้เพื่อน | URI | LINE Share |
| รับคูปอง | URI | ไปหน้ารับคูปอง |

### Button Styles และ Colors

```python
# Primary Button (โดดเด่น)
primary_button = {
    "type": "button",
    "style": "primary",
    "color": "#1DB446",  # สีเขียว LINE
    "action": {...}
}

# Secondary Button (รอง)
secondary_button = {
    "type": "button",
    "style": "secondary",
    "action": {...}
}

# Link Button (ลิงก์)
link_button = {
    "type": "button",
    "style": "link",
    "action": {...}
}

# Custom Color Button
custom_button = {
    "type": "button",
    "style": "primary",
    "color": "#FF6B35",  # สีส้ม
    "action": {...},
    "height": "sm"  # ปุ่มเล็ก
}
```

### Button Action Types

```python
# 1. URI Action - เปิด URL
uri_action = {
    "type": "uri",
    "label": "ซื้อเลย",
    "uri": "https://shop.example.com/buy/123"
}

# 2. Message Action - ส่งข้อความ
message_action = {
    "type": "message",
    "label": "สอบถามราคา",
    "text": "ขอราคาสินค้า #123"
}

# 3. Postback Action - ส่ง Data ลับๆ
postback_action = {
    "type": "postback",
    "label": "หยิบใส่ตะกร้า",
    "data": "action=add_cart&product_id=123&variant=blue",
    "displayText": "เพิ่มสินค้าลงตะกร้า"
}

# 4. Share Action - แชร์ข้อความ
share_action = {
    "type": "share"
}

# 5. Tel Action - โทรศัพท์
tel_action = {
    "type": "uri",
    "label": "โทรสั่ง",
    "uri": "tel:02-123-4567"
}

# 6. Location Action
location_action = {
    "type": "location",
    "label": "ส่งที่อยู่จัดส่ง"
}
```

---

## 12.7 Deep Linking กับ Card Messages

### Deep Link Patterns สำหรับ LINE

```python
# เปิดแอป LINE SHOPPING
line_shopping = "line://app/1655960601-BP2E0MnK"

# เปิด LIFF App
liff_url = "https://liff.line.me/LIFF_APP_ID/path/to/page"

# เปิด LINE Pay
line_pay = "linepay://pay?channelId=XXX&transactionId=YYY"

# เปิด Camera ใน LINE
line_camera = "line://camera/nativeCamera"

# เปิดหน้า Friend Profile
friend_profile = "line://profile?id=FRIEND_ID"

# เปิด LINE VOOM (Video Content)
line_voom = "https://www.line.me/voom/channel/{channelId}"
```

### LIFF (LINE Front-end Framework) URL

```python
def create_liff_deep_link(liff_id, path, params=None):
    """
    สร้าง LIFF Deep Link
    
    Args:
        liff_id: LIFF App ID จาก LINE Developers Console
        path: Path ใน LIFF App
        params: Query parameters
    
    Returns:
        LIFF URL
    """
    base_url = f"https://liff.line.me/{liff_id}{path}"
    
    if params:
        query_string = "&".join([f"{k}={v}" for k, v in params.items()])
        base_url += f"?{query_string}"
    
    return base_url

# ตัวอย่าง
liff_url = create_liff_deep_link(
    liff_id="1234567890-AbCdEfGh",
    path="/product",
    params={"id": "123", "from": "line_card"}
)
# ผลลัพธ์: https://liff.line.me/1234567890-AbCdEfGh/product?id=123&from=line_card
```

### Deep Link สำหรับ E-commerce

```python
# URL Schemes สำหรับแอปยอดนิยม

# Shopee
shopee_deep_link = "shopee://product?shopid=12345&itemid=67890"
shopee_web = "https://shopee.co.th/product/12345/67890"

# Lazada
lazada_deep_link = "lazada://product?itemId=12345"
lazada_web = "https://www.lazada.co.th/products/xxx-i12345.html"

# JD Central
jd_web = "https://www.jd.co.th/product/12345678.html"

# ฿ Market
baht_market = "https://www.bahtmarket.com/item/12345"

# Universal Link (ทำงานทั้ง Web และ App)
def create_universal_link(product_id, source="line"):
    return f"https://myshop.com/p/{product_id}?from={source}"
```

---

## 12.8 Analytics สำหรับ Card Message

### การ Track Performance ผ่าน UTM

```python
def add_tracking_to_card(card, campaign_name, card_position):
    """
    เพิ่ม UTM Tracking Parameters ให้กับ Card
    """
    import urllib.parse
    
    def add_utm_to_url(url, campaign, position):
        """เพิ่ม UTM Parameters ให้ URL"""
        separator = "&" if "?" in url else "?"
        utm_params = {
            "utm_source": "line",
            "utm_medium": "card_message",
            "utm_campaign": campaign,
            "utm_content": f"card_{position}"
        }
        utm_string = urllib.parse.urlencode(utm_params)
        return f"{url}{separator}{utm_string}"
    
    # ค้นหาและแก้ไข URI Actions ใน Card
    def process_contents(contents):
        if isinstance(contents, dict):
            if contents.get("type") == "button":
                action = contents.get("action", {})
                if action.get("type") == "uri":
                    action["uri"] = add_utm_to_url(
                        action["uri"], campaign_name, card_position
                    )
            
            for key, value in contents.items():
                if isinstance(value, (dict, list)):
                    process_contents(value)
        
        elif isinstance(contents, list):
            for item in contents:
                process_contents(item)
    
    process_contents(card)
    return card

# ตัวอย่างการใช้งาน
product_card = create_product_card(product_data)
tracked_card = add_tracking_to_card(
    product_card,
    campaign_name="summer_sale_2024",
    card_position=1
)
```

### Analytics Dashboard สำหรับ Card Performance

```python
def calculate_card_metrics(analytics_data):
    """
    คำนวณ Metrics สำหรับ Card Message Performance
    """
    
    total_sent = analytics_data.get("sent", 0)
    total_opened = analytics_data.get("opened", 0)
    total_button_clicks = analytics_data.get("button_clicks", {})
    total_conversions = analytics_data.get("conversions", 0)
    revenue = analytics_data.get("revenue", 0)
    
    metrics = {
        "open_rate": (total_opened / total_sent * 100) if total_sent > 0 else 0,
        "total_clicks": sum(total_button_clicks.values()),
        "ctr": (sum(total_button_clicks.values()) / total_sent * 100) if total_sent > 0 else 0,
        "conversion_rate": (total_conversions / sum(total_button_clicks.values()) * 100) 
                           if sum(total_button_clicks.values()) > 0 else 0,
        "revenue_per_message": revenue / total_sent if total_sent > 0 else 0,
        "button_breakdown": total_button_clicks
    }
    
    # Performance Rating
    rating = "ดีมาก" if metrics["ctr"] >= 5 else \
             "ดี" if metrics["ctr"] >= 2 else \
             "พอใช้" if metrics["ctr"] >= 1 else "ต้องปรับปรุง"
    
    return {**metrics, "rating": rating}

# ตัวอย่าง Analytics Data
sample_analytics = {
    "sent": 5000,
    "opened": 2800,
    "button_clicks": {
        "ซื้อเลย": 342,
        "ดูรายละเอียด": 156,
        "ถามราคา": 89
    },
    "conversions": 45,
    "revenue": 67500
}

result = calculate_card_metrics(sample_analytics)
print(f"Open Rate: {result['open_rate']:.1f}%")
print(f"CTR: {result['ctr']:.1f}%")
print(f"Conversion Rate: {result['conversion_rate']:.1f}%")
print(f"Revenue per Message: ฿{result['revenue_per_message']:.2f}")
print(f"Rating: {result['rating']}")
```

### ผลลัพธ์ที่คาดหวังสำหรับ Card Messages

| Metric | ต่ำกว่าเกณฑ์ | เกณฑ์ปกติ | ดีมาก |
|--------|------------|----------|-------|
| Open Rate | < 30% | 30-50% | > 50% |
| CTR | < 2% | 2-5% | > 5% |
| Conversion Rate | < 1% | 1-3% | > 3% |
| Revenue/Message | < ฿5 | ฿5-20 | > ฿20 |

---

## 12.9 ตัวอย่างจริงสำหรับธุรกิจไทย

### 12.9.1 ร้านขายเสื้อผ้า

```python
# Weekly New Arrivals Campaign
def send_weekly_new_arrivals(access_token, user_ids):
    """
    ส่งสินค้าใหม่ประจำสัปดาห์
    """
    
    new_arrivals = [
        {
            "name": "ชุดเดรสลายดอกไม้",
            "description": "ผ้า Chiffon เนื้อนุ่ม ระบายอากาศดี มี 3 สี",
            "price": 790,
            "original_price": 1200,
            "image_url": "https://cdn.fashion.co.th/dress-floral.jpg",
            "buy_url": "https://fashion.co.th/buy/dress-floral?utm_source=line&utm_campaign=new_arrival",
            "detail_url": "https://fashion.co.th/product/dress-floral"
        },
        {
            "name": "เสื้อครอป Cotton 100%",
            "description": "ผ้า Cotton แท้ ซักง่าย ไม่หด มีให้เลือก 8 สี",
            "price": 390,
            "original_price": 590,
            "image_url": "https://cdn.fashion.co.th/crop-top.jpg",
            "buy_url": "https://fashion.co.th/buy/crop-top?utm_source=line&utm_campaign=new_arrival",
            "detail_url": "https://fashion.co.th/product/crop-top"
        },
        {
            "name": "กางเกงขาม้า High Waist",
            "description": "ทรง Flared Pants เอวสูง เก็บทรงสวย แมตช์ได้ทุก Look",
            "price": 550,
            "original_price": 890,
            "image_url": "https://cdn.fashion.co.th/wide-pants.jpg",
            "buy_url": "https://fashion.co.th/buy/wide-pants?utm_source=line&utm_campaign=new_arrival",
            "detail_url": "https://fashion.co.th/product/wide-pants"
        }
    ]
    
    # สร้าง Header Message ก่อน
    header_message = {
        "type": "text",
        "text": "✨ New Arrivals ประจำสัปดาห์!\nสินค้าใหม่เพิ่งเข้า ช้อปก่อนหมด! 🛍️"
    }
    
    # สร้าง Product Carousel
    product_bubbles = [create_product_card(p) for p in new_arrivals]
    carousel_message = {
        "type": "flex",
        "altText": "New Arrivals ประจำสัปดาห์",
        "contents": {
            "type": "carousel",
            "contents": product_bubbles
        }
    }
    
    # ส่ง Multicast
    import requests
    
    # ส่งครั้งละ 500 คน (LINE Limit)
    for i in range(0, len(user_ids), 500):
        batch = user_ids[i:i+500]
        
        requests.post(
            "https://api.line.me/v2/bot/message/multicast",
            headers={
                "Authorization": f"Bearer {access_token}",
                "Content-Type": "application/json"
            },
            json={
                "to": batch,
                "messages": [header_message, carousel_message]
            }
        )
```

### 12.9.2 ธุรกิจอาหาร - Set Lunch Campaign

```python
def create_lunch_set_cards(menu_items, coupon_code=None):
    """
    สร้าง Card สำหรับเมนู Set Lunch
    """
    
    bubbles = []
    
    for item in menu_items:
        bubble = {
            "type": "bubble",
            "hero": {
                "type": "image",
                "url": item["image_url"],
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
                        "text": item["name"],
                        "weight": "bold",
                        "size": "xl"
                    },
                    {
                        "type": "text",
                        "text": item.get("description", ""),
                        "size": "sm",
                        "color": "#555555",
                        "wrap": True,
                        "margin": "sm"
                    },
                    {
                        "type": "box",
                        "layout": "horizontal",
                        "margin": "md",
                        "contents": [
                            {
                                "type": "text",
                                "text": "ราคา Set:",
                                "size": "sm",
                                "color": "#888888"
                            },
                            {
                                "type": "text",
                                "text": f"฿{item['price']}",
                                "size": "xl",
                                "weight": "bold",
                                "color": "#1DB446",
                                "align": "end"
                            }
                        ]
                    },
                    {
                        "type": "text",
                        "text": f"รวม: {', '.join(item.get('includes', []))}",
                        "size": "xs",
                        "color": "#888888",
                        "wrap": True,
                        "margin": "sm"
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
                            "label": "สั่งเดลิเวอรี่",
                            "uri": f"https://restaurant.com/order/{item['id']}?coupon={coupon_code or ''}"
                        }
                    },
                    {
                        "type": "button",
                        "style": "secondary",
                        "margin": "sm",
                        "action": {
                            "type": "message",
                            "label": "จองโต๊ะ",
                            "text": f"จองโต๊ะ Set {item['name']}"
                        }
                    }
                ]
            }
        }
        
        bubbles.append(bubble)
    
    return {
        "type": "flex",
        "altText": "Set Lunch พิเศษวันนี้",
        "contents": {
            "type": "carousel",
            "contents": bubbles
        }
    }

# ข้อมูลเมนู
lunch_menus = [
    {
        "id": "set-a",
        "name": "Set A - ผัดไทยกุ้ง",
        "description": "ผัดไทยกุ้งสด + ต้มยำน้ำใส + ข้าว + เครื่องดื่ม",
        "price": 189,
        "image_url": "https://cdn.restaurant.com/set-a.jpg",
        "includes": ["ผัดไทยกุ้งสด", "ต้มยำน้ำใส", "ข้าวสวย", "น้ำเปล่า"]
    },
    {
        "id": "set-b",
        "name": "Set B - แกงเขียวหวาน",
        "description": "แกงเขียวหวานไก่ + ไข่เจียวหมูสับ + ข้าว + เครื่องดื่ม",
        "price": 169,
        "image_url": "https://cdn.restaurant.com/set-b.jpg",
        "includes": ["แกงเขียวหวานไก่", "ไข่เจียวหมูสับ", "ข้าวสวย", "ชาไทยเย็น"]
    }
]

lunch_carousel = create_lunch_set_cards(lunch_menus, coupon_code="LUNCH10")
```

---

## 12.10 การจัดการ Card Message ด้วย Python Class

```python
class CardMessageBuilder:
    """
    Class สำหรับสร้าง Card Messages อย่างมีระบบ
    """
    
    def __init__(self, access_token):
        self.access_token = access_token
        self.messages = []
    
    def add_product_carousel(self, products, title=None):
        """เพิ่ม Product Carousel"""
        
        bubbles = [create_product_card(p) for p in products[:10]]
        
        message = {
            "type": "flex",
            "altText": title or f"สินค้าแนะนำ {len(bubbles)} รายการ",
            "contents": {
                "type": "carousel",
                "contents": bubbles
            }
        }
        
        self.messages.append(message)
        return self
    
    def add_coupon(self, coupon_data):
        """เพิ่ม Coupon Card"""
        
        message = {
            "type": "flex",
            "altText": f"คูปอง: {coupon_data.get('title', '')}",
            "contents": create_coupon_card(coupon_data)
        }
        
        self.messages.append(message)
        return self
    
    def add_text(self, text):
        """เพิ่มข้อความธรรมดา"""
        self.messages.append({"type": "text", "text": text})
        return self
    
    def add_image_carousel(self, items, title=None):
        """เพิ่ม Image Carousel"""
        self.messages.append(create_image_carousel(items))
        return self
    
    def send_to(self, user_id):
        """ส่งข้อความทั้งหมดให้ผู้ใช้"""
        import requests
        
        if not self.messages:
            raise ValueError("ไม่มีข้อความที่จะส่ง")
        
        # LINE จำกัด 5 messages ต่อ request
        for i in range(0, len(self.messages), 5):
            batch = self.messages[i:i+5]
            
            response = requests.post(
                "https://api.line.me/v2/bot/message/push",
                headers={
                    "Authorization": f"Bearer {self.access_token}",
                    "Content-Type": "application/json"
                },
                json={
                    "to": user_id,
                    "messages": batch
                }
            )
            
            if response.status_code != 200:
                print(f"Error: {response.json()}")
                return False
        
        self.messages = []  # Clear messages after sending
        return True
    
    def multicast_to(self, user_ids):
        """ส่งข้อความให้ผู้ใช้หลายคน"""
        import requests
        
        # ส่งครั้งละ 500 คน
        for i in range(0, len(user_ids), 500):
            batch_users = user_ids[i:i+500]
            
            for j in range(0, len(self.messages), 5):
                batch_msgs = self.messages[j:j+5]
                
                requests.post(
                    "https://api.line.me/v2/bot/message/multicast",
                    headers={
                        "Authorization": f"Bearer {self.access_token}",
                        "Content-Type": "application/json"
                    },
                    json={
                        "to": batch_users,
                        "messages": batch_msgs
                    }
                )
        
        self.messages = []
        return True

# ตัวอย่างการใช้งาน
builder = CardMessageBuilder("YOUR_ACCESS_TOKEN")

result = (
    builder
    .add_text("🎉 โปรโมชั่นพิเศษสุดสัปดาห์นี้!")
    .add_product_carousel(products, "สินค้าแนะนำ")
    .add_coupon({
        "title": "ใช้โค้ดรับส่วนลด 15%",
        "code": "WEEKEND15",
        "use_url": "https://shop.example.com"
    })
    .send_to("Uxxxxxxxxxx")
)
```

---

## 12.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Product Card สำหรับร้านค้า

**โจทย์:** สร้าง Product Card Carousel สำหรับร้านขายอุปกรณ์อิเล็กทรอนิกส์ แสดงสินค้า 4 รายการ

**โจทย์ย่อย:**
1. แต่ละ Card มี 2 ปุ่ม: "สั่งซื้อ" และ "เปรียบเทียบ"
2. แสดงราคาเดิมและราคาที่ลดแล้ว
3. เพิ่ม UTM Parameters ให้ทุก URL

```python
# Template ให้ฝึก
electronics = [
    {"name": "TWS Earbuds Pro", "price": 1290, "original_price": 2490, "image_url": "..."},
    {"name": "Smart Watch Series X", "price": 3990, "original_price": 6990, "image_url": "..."},
    {"name": "Portable Charger 20000mAh", "price": 890, "original_price": 1490, "image_url": "..."},
    {"name": "USB-C Hub 7-in-1", "price": 690, "original_price": 1200, "image_url": "..."}
]

# TODO: สร้าง Product Card Carousel
# 1. Loop ผ่าน electronics
# 2. สร้าง Bubble ให้แต่ละ item
# 3. เพิ่ม UTM Parameters
# 4. รวมเป็น Carousel
```

### แบบฝึกหัดที่ 2: Coupon Card Campaign

**โจทย์:** สร้าง Coupon Card สำหรับ Birthday Campaign ที่มีรหัส Unique สำหรับแต่ละลูกค้า

```python
def create_birthday_coupon(user_data, base_discount=20):
    """
    TODO: สร้าง Birthday Coupon Card
    
    - ชื่อคูปอง: "สุขสันต์วันเกิด [ชื่อลูกค้า]!"
    - ส่วนลด: base_discount % หรือมากกว่าสำหรับ VIP
    - รหัสคูปอง: BD-[USER_ID]-[YEAR]
    - หมดอายุ: วันเกิด + 7 วัน
    """
    pass

# Test
user = {
    "name": "คุณสมชาย",
    "user_id": "U12345",
    "is_vip": True,
    "birthday": "1990-09-15"
}
birthday_coupon = create_birthday_coupon(user)
```

---

## 12.12 สรุปบทที่ 12

### Key Takeaways

1. **Card Message มี 3 ประเภทหลัก**: Product Card, Image Carousel, Coupon Card
2. **Product Card** เหมาะสำหรับแสดงสินค้าพร้อมราคาและปุ่มซื้อ
3. **Image Carousel** เหมาะสำหรับแสดง Category หรือ Portfolio
4. **Coupon Card** เหมาะสำหรับโปรโมชั่นและส่วนลด
5. **สูงสุด 10 Cards** ต่อข้อความ และ **3 ปุ่ม** ต่อ Card
6. **UTM Parameters** ช่วยติดตามประสิทธิภาพได้อย่างแม่นยำ

### Checklist ก่อนส่ง Card Message

```
□ ภาพทุกรูปขนาดเท่ากัน (Aspect Ratio เดียวกัน)
□ URL ทุกปุ่มทดสอบแล้ว
□ UTM Parameters ครบถ้วน
□ ชื่อสินค้าและราคาถูกต้อง
□ ทดสอบบน iOS และ Android
□ กำหนดเวลาส่งที่เหมาะสม
□ ตั้ง Alt Text ที่น่าสนใจ
```

---

*บทถัดไป: Part 13 - Coupons & Rewards - สร้างความภักดีของลูกค้าด้วยระบบคูปองและสะสมแต้ม*
