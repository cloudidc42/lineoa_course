# Part 18: LINE OA กับ E-commerce - การเชื่อมต่อกับร้านค้าออนไลน์

## บทนำ

LINE OA ไม่ได้เป็นแค่ช่องทางการสื่อสาร แต่สามารถเป็น Sales Channel ที่ทรงพลังสำหรับ E-commerce ได้อย่างมาก ในประเทศไทย ผู้ใช้ LINE มากกว่า 50 ล้านคน ทำให้ LINE OA เป็น Platform ที่ไม่ควรมองข้ามสำหรับธุรกิจ Online

ในบทนี้ เราจะเรียนรู้การเชื่อมต่อ LINE OA กับระบบ E-commerce ต่างๆ ตั้งแต่ LINE Shopping ไปจนถึง Shopify, WooCommerce และการสร้าง Custom Integration

---

## 18.1 Overview: LINE OA ใน E-commerce Ecosystem

### บทบาทของ LINE OA ใน Customer Journey

```
Customer Journey พร้อม LINE OA:

Awareness → Consideration → Purchase → Retention → Advocacy

LINE OA ช่วยใน:
├── Awareness: Broadcast โปรโมชัน, Rich Card สินค้า
├── Consideration: Chatbot ตอบคำถาม, Product Catalog
├── Purchase: Deep Link ไปร้าน, Payment Link
├── Retention: Order Updates, Loyalty Program
└── Advocacy: Review Request, Referral Program
```

### E-commerce Integration Landscape

| Platform | Integration Level | ความยาก | เหมาะกับ |
|---------|-----------------|---------|---------|
| LINE Shopping | Native | ง่าย | ธุรกิจทุกขนาดในไทย |
| Shopify | App + API | ปานกลาง | ร้านค้าระดับ SME-Enterprise |
| WooCommerce | Plugin + API | ปานกลาง | ร้านค้า WordPress |
| Custom API | Full Custom | ยาก | ธุรกิจที่มี Dev Team |
| ZORT / Ready2Ship | Integration Platform | ง่าย-ปานกลาง | ร้านค้า Multi-Channel |
| Lazada/Shopee | Chat Commerce | ปานกลาง | Marketplace Sellers |

---

## 18.2 LINE Shopping Integration

### LINE Shopping คืออะไร

LINE Shopping เป็น Marketplace ภายใน Ecosystem ของ LINE ที่ให้ผู้ใช้ซื้อสินค้าจากภายใน LINE App ได้เลย

### การสมัคร LINE Shopping

**ข้อกำหนดเบื้องต้น:**
- มี LINE OA ที่ Verified แล้ว
- มีสินค้าที่พร้อมขาย
- มีบัญชีธนาคารไทยสำหรับรับเงิน
- ไม่ขายสินค้าต้องห้าม

**ขั้นตอนการสมัคร:**
1. เข้า https://shopping.line.me/shop/register
2. กรอกข้อมูลร้านค้า
3. เชื่อมต่อกับ LINE OA
4. ยืนยันข้อมูลธนาคาร
5. อัปโหลดสินค้า
6. รอการ Approve (1-3 วันทำการ)

### การตั้งค่า LINE Shopping ใน OA Manager

```
ขั้นตอนใน LINE OA Manager:

1. Settings → LINE Shopping
2. เชื่อมต่อ LINE Shopping Account
3. ตั้งค่า Deep Link สำหรับสินค้า
4. เปิดใช้งาน Shopping Notification
5. ตั้ง Auto-Reply สำหรับคำถามเกี่ยวกับออเดอร์
```

### LINE Shopping API

```python
# LINE Shopping API Integration

import requests
import json
import hmac
import hashlib
import base64
from datetime import datetime

class LineShopping:
    """
    LINE Shopping API Client
    """
    
    API_BASE = "https://api-shop.line.me/v2"
    
    def __init__(self, shop_id: str, access_token: str, channel_secret: str):
        self.shop_id = shop_id
        self.access_token = access_token
        self.channel_secret = channel_secret
        self.headers = {
            'Authorization': f'Bearer {access_token}',
            'Content-Type': 'application/json'
        }
    
    def get_orders(self, status: str = None, page: int = 1, limit: int = 20) -> dict:
        """
        ดึงรายการออเดอร์
        
        Args:
            status: สถานะออเดอร์ (pending, processing, shipped, delivered, cancelled)
            page: หน้าที่ต้องการ
            limit: จำนวนรายการต่อหน้า
        """
        url = f"{self.API_BASE}/shops/{self.shop_id}/orders"
        params = {
            'page': page,
            'limit': limit
        }
        if status:
            params['status'] = status
        
        response = requests.get(url, headers=self.headers, params=params)
        return response.json()
    
    def update_order_status(self, order_id: str, status: str, tracking_info: dict = None) -> dict:
        """
        อัปเดตสถานะออเดอร์
        
        Args:
            order_id: รหัสออเดอร์
            status: สถานะใหม่
            tracking_info: ข้อมูล Tracking (สำหรับสถานะ shipped)
        """
        url = f"{self.API_BASE}/shops/{self.shop_id}/orders/{order_id}/status"
        
        payload = {'status': status}
        if tracking_info:
            payload['tracking'] = tracking_info
        
        response = requests.put(url, headers=self.headers, json=payload)
        return response.json()
    
    def get_products(self, category_id: str = None, page: int = 1) -> dict:
        """
        ดึงรายการสินค้า
        """
        url = f"{self.API_BASE}/shops/{self.shop_id}/products"
        params = {'page': page, 'limit': 20}
        if category_id:
            params['category_id'] = category_id
        
        response = requests.get(url, headers=self.headers, params=params)
        return response.json()
    
    def update_stock(self, product_id: str, variant_id: str, quantity: int) -> dict:
        """
        อัปเดตจำนวน Stock
        
        Args:
            product_id: รหัสสินค้า
            variant_id: รหัส Variant (ถ้ามี)
            quantity: จำนวนใหม่
        """
        url = f"{self.API_BASE}/shops/{self.shop_id}/products/{product_id}/variants/{variant_id}/stock"
        payload = {'quantity': quantity}
        
        response = requests.put(url, headers=self.headers, json=payload)
        return response.json()
    
    def send_order_notification(self, user_id: str, order_id: str, status: str) -> bool:
        """
        ส่งแจ้งเตือนสถานะออเดอร์ผ่าน LINE Message
        
        Args:
            user_id: LINE User ID
            order_id: รหัสออเดอร์
            status: สถานะออเดอร์
        """
        # สร้างข้อความตามสถานะ
        status_messages = {
            'confirmed': f'✅ ออเดอร์ #{order_id} ได้รับการยืนยันแล้ว\nขอบคุณที่ซื้อสินค้า!',
            'processing': f'📦 ออเดอร์ #{order_id} กำลังเตรียมสินค้า\nคาดว่าจัดส่งใน 1-2 วันทำการ',
            'shipped': f'🚚 ออเดอร์ #{order_id} จัดส่งแล้ว!\nติดตามพัสดุได้ที่ลิงก์ด้านล่าง',
            'delivered': f'🎉 ออเดอร์ #{order_id} ส่งถึงแล้ว!\nหากมีปัญหาติดต่อเราได้เลย',
            'cancelled': f'❌ ออเดอร์ #{order_id} ถูกยกเลิก\nจะคืนเงินภายใน 3-5 วันทำการ'
        }
        
        message = status_messages.get(status, f'ออเดอร์ #{order_id} สถานะ: {status}')
        
        # ส่งผ่าน LINE Messaging API
        line_api_url = "https://api.line.me/v2/bot/message/push"
        payload = {
            'to': user_id,
            'messages': [{'type': 'text', 'text': message}]
        }
        
        response = requests.post(
            line_api_url,
            headers=self.headers,
            json=payload
        )
        
        return response.status_code == 200

# ตัวอย่างการใช้งาน
# shop = LineShopping(
#     shop_id='SHOP-001',
#     access_token='YOUR_ACCESS_TOKEN',
#     channel_secret='YOUR_CHANNEL_SECRET'
# )
# orders = shop.get_orders(status='pending')
# print(json.dumps(orders, indent=2, ensure_ascii=False))
```

---

## 18.3 Product Catalog Setup

### การสร้าง Product Catalog ที่มีประสิทธิภาพ

Product Catalog บน LINE OA ช่วยให้ลูกค้าเรียกดูสินค้าได้โดยไม่ต้องออกจาก LINE

#### โครงสร้าง Product Catalog

```
Product Catalog Structure:

Category Level 1: เสื้อผ้าผู้หญิง
├── Category Level 2: เสื้อ
│   ├── Product: เสื้อยืดสีขาว (SKU: TS-001)
│   │   ├── Variant: S, M, L, XL
│   │   ├── Price: 299 บาท
│   │   ├── Images: 3 รูป
│   │   └── Stock: 50 ชิ้น
│   └── Product: เสื้อเชิ้ต (SKU: SH-001)
│       ├── Variant: สีขาว, สีฟ้า
│       ├── Price: 499 บาท
│       └── ...
└── Category Level 2: กางเกง
    └── ...
```

#### Flex Message สำหรับ Product Card

```json
// Flex Message Template สำหรับ Product Card
{
  "type": "flex",
  "altText": "สินค้าแนะนำ",
  "contents": {
    "type": "bubble",
    "hero": {
      "type": "image",
      "url": "https://example.com/product-image.jpg",
      "size": "full",
      "aspectRatio": "4:3",
      "action": {
        "type": "uri",
        "uri": "https://shop.example.com/product/001"
      }
    },
    "body": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        {
          "type": "text",
          "text": "เสื้อยืดสีขาว Basic",
          "weight": "bold",
          "size": "xl",
          "wrap": true
        },
        {
          "type": "box",
          "layout": "baseline",
          "margin": "md",
          "contents": [
            {
              "type": "text",
              "text": "ราคา",
              "size": "sm",
              "color": "#999999",
              "flex": 0
            },
            {
              "type": "text",
              "text": "299 บาท",
              "weight": "bold",
              "size": "xl",
              "color": "#FF0000",
              "margin": "md",
              "flex": 0
            }
          ]
        },
        {
          "type": "box",
          "layout": "vertical",
          "margin": "lg",
          "spacing": "sm",
          "contents": [
            {
              "type": "box",
              "layout": "baseline",
              "spacing": "sm",
              "contents": [
                {
                  "type": "text",
                  "text": "ไซส์",
                  "color": "#aaaaaa",
                  "size": "sm",
                  "flex": 1
                },
                {
                  "type": "text",
                  "text": "S / M / L / XL",
                  "wrap": true,
                  "color": "#666666",
                  "size": "sm",
                  "flex": 5
                }
              ]
            },
            {
              "type": "box",
              "layout": "baseline",
              "spacing": "sm",
              "contents": [
                {
                  "type": "text",
                  "text": "สต็อก",
                  "color": "#aaaaaa",
                  "size": "sm",
                  "flex": 1
                },
                {
                  "type": "text",
                  "text": "มีสินค้า",
                  "wrap": true,
                  "color": "#00B900",
                  "size": "sm",
                  "flex": 5
                }
              ]
            }
          ]
        }
      ]
    },
    "footer": {
      "type": "box",
      "layout": "vertical",
      "spacing": "sm",
      "contents": [
        {
          "type": "button",
          "style": "primary",
          "action": {
            "type": "uri",
            "label": "ซื้อเลย",
            "uri": "https://shop.example.com/product/001?utm_source=line&utm_medium=catalog"
          }
        },
        {
          "type": "button",
          "action": {
            "type": "message",
            "label": "สอบถามข้อมูล",
            "text": "สอบถามสินค้า SKU-001"
          }
        }
      ]
    }
  }
}
```

#### Carousel สำหรับแสดงหลายสินค้า

```python
# สร้าง Product Carousel Message

def create_product_carousel(products: list) -> dict:
    """
    สร้าง Carousel Message สำหรับแสดงหลายสินค้า
    
    Args:
        products: list ของข้อมูลสินค้า
            [{id, name, price, image_url, product_url, in_stock}]
    
    Returns:
        carousel_message: Flex Message Carousel
    """
    bubbles = []
    
    for product in products[:10]:  # LINE รองรับสูงสุด 12 Bubbles
        stock_text = "มีสินค้า" if product.get('in_stock', True) else "สินค้าหมด"
        stock_color = "#00B900" if product.get('in_stock', True) else "#FF0000"
        
        bubble = {
            "type": "bubble",
            "hero": {
                "type": "image",
                "url": product['image_url'],
                "size": "full",
                "aspectRatio": "1:1",
                "action": {
                    "type": "uri",
                    "uri": product['product_url']
                }
            },
            "body": {
                "type": "box",
                "layout": "vertical",
                "contents": [
                    {
                        "type": "text",
                        "text": product['name'],
                        "weight": "bold",
                        "size": "md",
                        "wrap": True,
                        "maxLines": 2
                    },
                    {
                        "type": "box",
                        "layout": "baseline",
                        "margin": "sm",
                        "contents": [
                            {
                                "type": "text",
                                "text": f"฿{product['price']:,}",
                                "weight": "bold",
                                "color": "#FF4444",
                                "size": "lg"
                            }
                        ]
                    },
                    {
                        "type": "text",
                        "text": stock_text,
                        "color": stock_color,
                        "size": "sm",
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
                        "color": "#00B900",
                        "action": {
                            "type": "uri",
                            "label": "ซื้อเลย",
                            "uri": f"{product['product_url']}?ref=line_catalog"
                        }
                    }
                ]
            }
        }
        
        bubbles.append(bubble)
    
    return {
        "type": "flex",
        "altText": "สินค้าแนะนำ",
        "contents": {
            "type": "carousel",
            "contents": bubbles
        }
    }

# ตัวอย่างการใช้งาน
sample_products = [
    {
        'id': '001',
        'name': 'เสื้อยืดสีขาว Basic Tee',
        'price': 299,
        'image_url': 'https://example.com/product-001.jpg',
        'product_url': 'https://shop.example.com/products/001',
        'in_stock': True
    },
    {
        'id': '002',
        'name': 'กางเกงยีนส์ Slim Fit',
        'price': 790,
        'image_url': 'https://example.com/product-002.jpg',
        'product_url': 'https://shop.example.com/products/002',
        'in_stock': True
    },
    {
        'id': '003',
        'name': 'กระเป๋า Canvas Limited Edition',
        'price': 1290,
        'image_url': 'https://example.com/product-003.jpg',
        'product_url': 'https://shop.example.com/products/003',
        'in_stock': False
    }
]

carousel = create_product_carousel(sample_products)
import json
print(json.dumps(carousel, indent=2, ensure_ascii=False))
```

---

## 18.4 Order Notifications ผ่าน LINE

### ระบบแจ้งเตือนออเดอร์อัตโนมัติ

```python
# ระบบส่ง Order Notification ผ่าน LINE

import requests
import json
from datetime import datetime
from typing import Optional

class OrderNotificationSystem:
    """
    ระบบส่งแจ้งเตือนออเดอร์ผ่าน LINE
    """
    
    LINE_API_URL = "https://api.line.me/v2/bot/message/push"
    
    def __init__(self, channel_access_token: str):
        self.token = channel_access_token
        self.headers = {
            'Authorization': f'Bearer {channel_access_token}',
            'Content-Type': 'application/json'
        }
    
    def send_order_confirmed(self, user_id: str, order: dict) -> bool:
        """
        ส่งแจ้งเตือนยืนยันออเดอร์
        """
        message = self._create_order_confirmed_message(order)
        return self._send_message(user_id, message)
    
    def send_order_shipped(self, user_id: str, order: dict, tracking: dict) -> bool:
        """
        ส่งแจ้งเตือนจัดส่งสินค้า
        """
        message = self._create_shipped_message(order, tracking)
        return self._send_message(user_id, message)
    
    def send_order_delivered(self, user_id: str, order: dict) -> bool:
        """
        ส่งแจ้งเตือนสินค้าถึงปลายทาง
        """
        message = self._create_delivered_message(order)
        return self._send_message(user_id, message)
    
    def _create_order_confirmed_message(self, order: dict) -> dict:
        """
        สร้างข้อความยืนยันออเดอร์ (Flex Message)
        """
        items_text = "\n".join([
            f"• {item['name']} x{item['qty']} = ฿{item['price']:,}"
            for item in order.get('items', [])
        ])
        
        return {
            "type": "flex",
            "altText": f"ยืนยันออเดอร์ #{order['order_id']}",
            "contents": {
                "type": "bubble",
                "styles": {
                    "header": {"backgroundColor": "#00B900"},
                    "body": {"backgroundColor": "#f9f9f9"}
                },
                "header": {
                    "type": "box",
                    "layout": "vertical",
                    "contents": [
                        {
                            "type": "text",
                            "text": "✅ ยืนยันออเดอร์แล้ว!",
                            "color": "#ffffff",
                            "weight": "bold",
                            "size": "xl"
                        }
                    ]
                },
                "body": {
                    "type": "box",
                    "layout": "vertical",
                    "spacing": "md",
                    "contents": [
                        {
                            "type": "text",
                            "text": f"ออเดอร์ #{order['order_id']}",
                            "weight": "bold",
                            "size": "lg"
                        },
                        {
                            "type": "separator"
                        },
                        {
                            "type": "text",
                            "text": "รายการสินค้า:",
                            "color": "#666666",
                            "size": "sm"
                        },
                        {
                            "type": "text",
                            "text": items_text,
                            "size": "sm",
                            "wrap": True
                        },
                        {
                            "type": "separator"
                        },
                        {
                            "type": "box",
                            "layout": "horizontal",
                            "contents": [
                                {
                                    "type": "text",
                                    "text": "ยอดรวม:",
                                    "size": "md",
                                    "color": "#666666"
                                },
                                {
                                    "type": "text",
                                    "text": f"฿{order['total']:,}",
                                    "size": "md",
                                    "weight": "bold",
                                    "color": "#FF0000",
                                    "align": "end"
                                }
                            ]
                        },
                        {
                            "type": "box",
                            "layout": "horizontal",
                            "contents": [
                                {
                                    "type": "text",
                                    "text": "คาดว่าจะได้รับ:",
                                    "size": "sm",
                                    "color": "#666666"
                                },
                                {
                                    "type": "text",
                                    "text": order.get('estimated_delivery', '1-3 วันทำการ'),
                                    "size": "sm",
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
                            "action": {
                                "type": "uri",
                                "label": "ดูออเดอร์",
                                "uri": f"https://shop.example.com/orders/{order['order_id']}"
                            }
                        },
                        {
                            "type": "button",
                            "action": {
                                "type": "message",
                                "label": "ติดต่อเรา",
                                "text": f"สอบถามออเดอร์ #{order['order_id']}"
                            }
                        }
                    ]
                }
            }
        }
    
    def _create_shipped_message(self, order: dict, tracking: dict) -> dict:
        """
        สร้างข้อความแจ้งจัดส่ง
        """
        tracking_url = tracking.get('tracking_url', '')
        tracking_number = tracking.get('tracking_number', 'N/A')
        courier = tracking.get('courier', 'ไม่ระบุ')
        
        return {
            "type": "flex",
            "altText": f"จัดส่งออเดอร์ #{order['order_id']} แล้ว",
            "contents": {
                "type": "bubble",
                "hero": {
                    "type": "image",
                    "url": "https://example.com/shipping-banner.png",
                    "size": "full",
                    "aspectRatio": "20:8"
                },
                "body": {
                    "type": "box",
                    "layout": "vertical",
                    "spacing": "md",
                    "contents": [
                        {
                            "type": "text",
                            "text": "🚚 สินค้ากำลังจัดส่ง!",
                            "weight": "bold",
                            "size": "xl"
                        },
                        {
                            "type": "text",
                            "text": f"ออเดอร์ #{order['order_id']}",
                            "color": "#666666"
                        },
                        {
                            "type": "separator"
                        },
                        {
                            "type": "box",
                            "layout": "vertical",
                            "spacing": "sm",
                            "contents": [
                                {
                                    "type": "box",
                                    "layout": "horizontal",
                                    "contents": [
                                        {"type": "text", "text": "ขนส่ง:", "size": "sm", "color": "#aaaaaa", "flex": 2},
                                        {"type": "text", "text": courier, "size": "sm", "flex": 4}
                                    ]
                                },
                                {
                                    "type": "box",
                                    "layout": "horizontal",
                                    "contents": [
                                        {"type": "text", "text": "หมายเลข:", "size": "sm", "color": "#aaaaaa", "flex": 2},
                                        {"type": "text", "text": tracking_number, "size": "sm", "weight": "bold", "flex": 4}
                                    ]
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
                                "label": "ติดตามพัสดุ",
                                "uri": tracking_url or "https://tracking.example.com"
                            }
                        }
                    ]
                }
            }
        }
    
    def _create_delivered_message(self, order: dict) -> dict:
        """
        สร้างข้อความแจ้งสินค้าถึงแล้ว
        """
        return {
            "type": "flex",
            "altText": f"สินค้าถึงแล้ว #{order['order_id']}",
            "contents": {
                "type": "bubble",
                "body": {
                    "type": "box",
                    "layout": "vertical",
                    "spacing": "md",
                    "contents": [
                        {
                            "type": "text",
                            "text": "🎉 สินค้าถึงแล้ว!",
                            "weight": "bold",
                            "size": "xl"
                        },
                        {
                            "type": "text",
                            "text": f"ออเดอร์ #{order['order_id']} ส่งถึงคุณแล้ว",
                            "wrap": True,
                            "color": "#666666"
                        },
                        {
                            "type": "text",
                            "text": "ขอบคุณที่ใช้บริการ! หากสินค้าไม่ตรงกับที่สั่ง หรือมีปัญหา ติดต่อเราได้ภายใน 7 วัน",
                            "wrap": True,
                            "size": "sm",
                            "color": "#888888",
                            "margin": "md"
                        }
                    ]
                },
                "footer": {
                    "type": "box",
                    "layout": "vertical",
                    "spacing": "sm",
                    "contents": [
                        {
                            "type": "button",
                            "style": "primary",
                            "action": {
                                "type": "uri",
                                "label": "⭐ รีวิวสินค้า",
                                "uri": f"https://shop.example.com/review/{order['order_id']}"
                            }
                        },
                        {
                            "type": "button",
                            "action": {
                                "type": "message",
                                "label": "แจ้งปัญหา",
                                "text": f"มีปัญหากับออเดอร์ #{order['order_id']}"
                            }
                        }
                    ]
                }
            }
        }
    
    def _send_message(self, user_id: str, message: dict) -> bool:
        """
        ส่ง Message ผ่าน LINE API
        """
        payload = {
            'to': user_id,
            'messages': [message]
        }
        
        response = requests.post(
            self.LINE_API_URL,
            headers=self.headers,
            json=payload
        )
        
        return response.status_code == 200

# ตัวอย่าง
# notify = OrderNotificationSystem('YOUR_TOKEN')
# order_data = {
#     'order_id': 'ORD-2024-001',
#     'items': [
#         {'name': 'เสื้อยืด Basic', 'qty': 2, 'price': 598},
#         {'name': 'กางเกงยีนส์', 'qty': 1, 'price': 790}
#     ],
#     'total': 1388,
#     'estimated_delivery': '20-22 ม.ค. 2567'
# }
# notify.send_order_confirmed('Uxxxxxxxxxx', order_data)
```

---

## 18.5 Abandoned Cart Messages

### กลยุทธ์ Abandoned Cart Recovery

Abandoned Cart Rate เฉลี่ยในไทยอยู่ที่ 70-80% หมายความว่าลูกค้า 7-8 ใน 10 คนที่เพิ่มสินค้าลงตะกร้าไม่ได้ซื้อจริง LINE OA ช่วยกู้คืนยอดขายที่หายไปได้

### Abandoned Cart Sequence

```
Abandoned Cart Recovery Sequence:

[ลูกค้าเพิ่มสินค้าในตะกร้า แต่ไม่ Checkout ใน 30 นาที]
│
├── Email 1 (LINE Message): 1 ชั่วโมงหลัง Abandon
│   "สินค้าในตะกร้ารอคุณอยู่! 🛒"
│   [ดูตะกร้า] [ต่อการซื้อ]
│
├── Message 2: 24 ชั่วโมงหลัง Abandon
│   "ยังไม่แน่ใจ? เราช่วยได้! มีส่วนลด 5%"
│   [ใช้โค้ดส่วนลด] [สอบถาม]
│
└── Message 3: 72 ชั่วโมงหลัง Abandon (Final)
    "สินค้าในตะกร้ากำลังจะหมด! รีบสั่งก่อนหมดนะ"
    [ซื้อเลย 10% Off] [ไม่ขอบคุณ]
```

```python
# ระบบ Abandoned Cart Recovery

from datetime import datetime, timedelta
import json

class AbandonedCartRecovery:
    """
    ระบบกู้คืน Abandoned Cart ผ่าน LINE
    """
    
    RECOVERY_SEQUENCE = [
        {
            'delay_hours': 1,
            'type': 'reminder',
            'discount': None,
            'message_key': 'first_reminder'
        },
        {
            'delay_hours': 24,
            'type': 'incentive',
            'discount': 5,  # 5%
            'message_key': 'second_reminder'
        },
        {
            'delay_hours': 72,
            'type': 'urgency',
            'discount': 10,  # 10%
            'message_key': 'final_reminder'
        }
    ]
    
    def __init__(self, line_token: str, shop_url: str):
        self.line_token = line_token
        self.shop_url = shop_url
        self.pending_carts = {}
    
    def track_abandoned_cart(self, user_id: str, cart: dict):
        """
        บันทึก Abandoned Cart
        
        Args:
            user_id: LINE User ID
            cart: ข้อมูลตะกร้า {items, total, cart_url}
        """
        self.pending_carts[user_id] = {
            'cart': cart,
            'abandoned_at': datetime.now().isoformat(),
            'messages_sent': 0,
            'recovered': False
        }
    
    def mark_as_purchased(self, user_id: str):
        """
        Mark ว่าลูกค้าซื้อแล้ว (หยุดส่ง Recovery Messages)
        """
        if user_id in self.pending_carts:
            self.pending_carts[user_id]['recovered'] = True
    
    def get_pending_recoveries(self) -> list:
        """
        ดึงรายการ Cart ที่ต้องส่ง Recovery Message
        """
        pending = []
        now = datetime.now()
        
        for user_id, cart_data in self.pending_carts.items():
            if cart_data['recovered']:
                continue
            
            messages_sent = cart_data['messages_sent']
            if messages_sent >= len(self.RECOVERY_SEQUENCE):
                continue
            
            # ตรวจสอบว่าถึงเวลาส่ง Message ถัดไปหรือยัง
            abandoned_at = datetime.fromisoformat(cart_data['abandoned_at'])
            sequence = self.RECOVERY_SEQUENCE[messages_sent]
            send_time = abandoned_at + timedelta(hours=sequence['delay_hours'])
            
            if now >= send_time:
                pending.append({
                    'user_id': user_id,
                    'cart': cart_data['cart'],
                    'sequence': sequence,
                    'messages_sent': messages_sent
                })
        
        return pending
    
    def create_recovery_message(self, cart: dict, sequence: dict) -> dict:
        """
        สร้าง Recovery Message ตาม Sequence
        """
        items_summary = ", ".join([
            item['name'] for item in cart.get('items', [])[:2]
        ])
        if len(cart.get('items', [])) > 2:
            items_summary += f" และอีก {len(cart['items']) - 2} รายการ"
        
        cart_total = cart.get('total', 0)
        cart_url = cart.get('cart_url', self.shop_url + '/cart')
        
        if sequence['type'] == 'reminder':
            # Message 1: Soft Reminder
            return {
                "type": "flex",
                "altText": "สินค้าในตะกร้ารอคุณอยู่! 🛒",
                "contents": {
                    "type": "bubble",
                    "body": {
                        "type": "box",
                        "layout": "vertical",
                        "spacing": "md",
                        "contents": [
                            {
                                "type": "text",
                                "text": "🛒 สินค้าในตะกร้ารอคุณอยู่!",
                                "weight": "bold",
                                "size": "xl",
                                "wrap": True
                            },
                            {
                                "type": "text",
                                "text": f"คุณมีสินค้า: {items_summary}",
                                "wrap": True,
                                "color": "#666666"
                            },
                            {
                                "type": "text",
                                "text": f"ยอดรวม: ฿{cart_total:,}",
                                "weight": "bold",
                                "size": "lg",
                                "color": "#FF4444"
                            }
                        ]
                    },
                    "footer": {
                        "type": "box",
                        "layout": "vertical",
                        "spacing": "sm",
                        "contents": [
                            {
                                "type": "button",
                                "style": "primary",
                                "action": {
                                    "type": "uri",
                                    "label": "ดูตะกร้า",
                                    "uri": cart_url
                                }
                            }
                        ]
                    }
                }
            }
        
        elif sequence['type'] == 'incentive':
            # Message 2: ให้ส่วนลด
            discount_pct = sequence['discount']
            discount_amount = int(cart_total * discount_pct / 100)
            discount_code = f"CART{discount_pct}OFF"
            
            return {
                "type": "flex",
                "altText": f"มีส่วนลด {discount_pct}% สำหรับคุณ!",
                "contents": {
                    "type": "bubble",
                    "styles": {
                        "header": {"backgroundColor": "#FF6B35"}
                    },
                    "header": {
                        "type": "box",
                        "layout": "vertical",
                        "contents": [
                            {
                                "type": "text",
                                "text": f"🎁 ส่วนลดพิเศษ {discount_pct}% สำหรับคุณ!",
                                "color": "#ffffff",
                                "weight": "bold",
                                "wrap": True
                            }
                        ]
                    },
                    "body": {
                        "type": "box",
                        "layout": "vertical",
                        "spacing": "md",
                        "contents": [
                            {
                                "type": "text",
                                "text": "ยังคิดอยู่ไหม? เราให้ส่วนลดพิเศษ!",
                                "wrap": True
                            },
                            {
                                "type": "box",
                                "layout": "vertical",
                                "backgroundColor": "#FFF3E0",
                                "cornerRadius": "md",
                                "paddingAll": "md",
                                "margin": "md",
                                "contents": [
                                    {
                                        "type": "text",
                                        "text": "โค้ดส่วนลด:",
                                        "size": "sm",
                                        "color": "#888888"
                                    },
                                    {
                                        "type": "text",
                                        "text": discount_code,
                                        "size": "xxl",
                                        "weight": "bold",
                                        "color": "#FF6B35"
                                    },
                                    {
                                        "type": "text",
                                        "text": f"ลดทันที ฿{discount_amount:,}",
                                        "size": "sm",
                                        "color": "#666666"
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
                                    "label": f"ใช้โค้ดลด {discount_pct}% เลย",
                                    "uri": f"{cart_url}?coupon={discount_code}"
                                }
                            }
                        ]
                    }
                }
            }
        
        else:
            # Message 3: Urgency
            return {
                "type": "flex",
                "altText": "⚠️ สินค้ากำลังจะหมด! รีบสั่งเลย",
                "contents": {
                    "type": "bubble",
                    "body": {
                        "type": "box",
                        "layout": "vertical",
                        "spacing": "md",
                        "contents": [
                            {
                                "type": "text",
                                "text": "⚠️ สินค้าของคุณกำลังจะหมด!",
                                "weight": "bold",
                                "size": "lg",
                                "color": "#FF0000",
                                "wrap": True
                            },
                            {
                                "type": "text",
                                "text": f"มีคนสนใจสินค้าในตะกร้าคุณหลายคน รีบซื้อก่อนหมดนะ",
                                "wrap": True,
                                "color": "#666666"
                            },
                            {
                                "type": "text",
                                "text": f"ลดพิเศษ {sequence['discount']}% สำหรับการสั่งซื้อวันนี้!",
                                "weight": "bold",
                                "color": "#FF6B35"
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
                                "color": "#FF0000",
                                "action": {
                                    "type": "uri",
                                    "label": f"ซื้อเลย {sequence['discount']}% Off",
                                    "uri": f"{cart_url}?urgency=1&discount={sequence['discount']}"
                                }
                            }
                        ]
                    }
                }
            }

# ตัวอย่างการใช้งาน
recovery_system = AbandonedCartRecovery(
    line_token='YOUR_LINE_TOKEN',
    shop_url='https://shop.example.com'
)

# บันทึก Abandoned Cart
recovery_system.track_abandoned_cart(
    user_id='Uxxxxxxx001',
    cart={
        'items': [
            {'name': 'เสื้อยืด Basic White', 'price': 299, 'qty': 2},
            {'name': 'กางเกงยีนส์ Slim', 'price': 790, 'qty': 1}
        ],
        'total': 1388,
        'cart_url': 'https://shop.example.com/cart/abc123'
    }
)

print("Abandoned Cart Recovery System พร้อมใช้งาน!")
```

---

## 18.6 Customer Support ผ่าน LINE

### การตั้งค่า Customer Support System

#### Response Time Standards

| ช่องทาง | Response Time ที่แนะนำ | ยอมรับได้สูงสุด |
|---------|---------------------|----------------|
| LINE Chat | 5-15 นาที | 1 ชั่วโมง |
| LINE OA (ในเวลาทำการ) | 15-30 นาที | 2 ชั่วโมง |
| LINE OA (นอกเวลาทำการ) | Auto-reply + follow up ตอนเช้า | 12 ชั่วโมง |

#### Auto-Reply ระหว่างนอกเวลาทำการ

```
ตัวอย่าง Auto-Reply Message:

"สวัสดีครับ/ค่ะ ขอบคุณที่ติดต่อ [ชื่อร้าน]

ขณะนี้อยู่นอกเวลาทำการ (08:00-20:00 น.)

เจ้าหน้าที่จะติดต่อกลับภายใน 8.00 น. วันทำการถัดไป

สำหรับเรื่องเร่งด่วน กรุณาโทร 02-xxx-xxxx

ขอบคุณที่ใช้บริการครับ/ค่ะ 😊"
```

### FAQ Bot สำหรับ E-commerce

```python
# FAQ Chatbot สำหรับ E-commerce

class EcommerceFAQBot:
    """
    Chatbot สำหรับตอบคำถาม E-commerce บน LINE
    """
    
    FAQ_RESPONSES = {
        # คำสำคัญ: ข้อความตอบ
        'สั่งซื้อ': {
            'keywords': ['สั่ง', 'ซื้อ', 'order', 'ออเดอร์'],
            'response': """วิธีสั่งซื้อสินค้า:
1. เลือกสินค้าที่ต้องการ
2. เลือกไซส์/สี/จำนวน
3. กด "เพิ่มในตะกร้า"
4. ไปที่ตะกร้า → ยืนยันการสั่งซื้อ
5. กรอกที่อยู่จัดส่ง
6. เลือกวิธีชำระเงิน
7. ยืนยันออเดอร์

หากมีปัญหาในการสั่งซื้อ พิมพ์ "ติดต่อเจ้าหน้าที่" ได้เลยครับ""",
            'quick_replies': ['วิธีชำระเงิน', 'ระยะเวลาจัดส่ง', 'ตรวจสอบออเดอร์']
        },
        'ชำระเงิน': {
            'keywords': ['ชำระ', 'จ่าย', 'โอน', 'payment', 'pay'],
            'response': """วิธีชำระเงินที่รับ:
💳 บัตรเครดิต/เดบิต (Visa, Mastercard)
📱 QR Code Payment (PromptPay, SCB Pay, KPlus)
🏦 โอนเงินผ่านธนาคาร
🛍️ LINE Pay
💰 เก็บเงินปลายทาง (COD) - บวกค่าบริการ 30 บาท

หลังจากโอนเงิน กรุณาแจ้งสลิปผ่านช่องทางนี้ครับ""",
            'quick_replies': ['แนบสลิปโอนเงิน', 'ระยะเวลาจัดส่ง', 'ตรวจสอบออเดอร์']
        },
        'จัดส่ง': {
            'keywords': ['จัดส่ง', 'ส่ง', 'delivery', 'shipping', 'ส่งของ', 'พัสดุ'],
            'response': """ข้อมูลการจัดส่ง:
📦 ขนส่งที่ใช้: Kerry, Flash Express, J&T
⏱ ระยะเวลา: 1-3 วันทำการ (กทม.), 2-5 วัน (ต่างจังหวัด)
💰 ค่าส่ง: ฿50 (สั่งน้อยกว่า 500 บาท), ฟรี (สั่ง 500 บาทขึ้นไป)
🚚 ตัดออเดอร์: 12:00 น. ทุกวัน (จัดส่งวันเดียวกัน)

ติดตามพัสดุ: https://track.example.com""",
            'quick_replies': ['ติดตามพัสดุ', 'สั่งซื้อ', 'คืนสินค้า']
        },
        'คืนสินค้า': {
            'keywords': ['คืน', 'return', 'เปลี่ยน', 'exchange', 'รีเทิร์น'],
            'response': """นโยบายคืนสินค้า:
✅ คืนได้ภายใน 7 วัน หลังได้รับสินค้า
✅ สินค้าต้องอยู่ในสภาพเดิม ไม่ผ่านการใช้งาน
✅ มีบรรจุภัณฑ์ครบ

ขั้นตอน:
1. แจ้งผ่าน LINE นี้
2. เราจะส่ง Label คืน
3. ส่งสินค้ากลับ
4. รับเงินคืนใน 3-5 วันทำการ

สำหรับสินค้าที่ได้รับผิด/ชำรุด ค่าส่งคืนฟรีครับ""",
            'quick_replies': ['แจ้งคืนสินค้า', 'ติดต่อเจ้าหน้าที่']
        },
        'ติดต่อ': {
            'keywords': ['ติดต่อ', 'staff', 'เจ้าหน้าที่', 'คน', 'human'],
            'response': 'TRANSFER_TO_HUMAN',  # Special action
            'quick_replies': []
        }
    }
    
    def get_response(self, user_message: str) -> dict:
        """
        หาคำตอบจากข้อความ
        
        Args:
            user_message: ข้อความจากผู้ใช้
        
        Returns:
            response: dict ของ response
        """
        message_lower = user_message.lower()
        
        for category, data in self.FAQ_RESPONSES.items():
            keywords = data.get('keywords', [])
            if any(kw in message_lower for kw in keywords):
                return {
                    'found': True,
                    'category': category,
                    'response': data['response'],
                    'quick_replies': data.get('quick_replies', [])
                }
        
        # ไม่พบคำตอบ
        return {
            'found': False,
            'category': None,
            'response': """ขอโทษนะครับ/ค่ะ ไม่เข้าใจคำถาม

คุณสามารถถามเกี่ยวกับ:
• วิธีสั่งซื้อ
• วิธีชำระเงิน
• การจัดส่ง
• การคืนสินค้า
• ตรวจสอบออเดอร์

หรือพิมพ์ "ติดต่อเจ้าหน้าที่" เพื่อคุยกับคนครับ""",
            'quick_replies': ['วิธีสั่งซื้อ', 'การจัดส่ง', 'คืนสินค้า', 'ติดต่อเจ้าหน้าที่']
        }
    
    def create_quick_reply_message(self, text: str, quick_replies: list) -> dict:
        """
        สร้างข้อความพร้อม Quick Reply Buttons
        """
        items = [
            {
                "type": "action",
                "action": {
                    "type": "message",
                    "label": reply[:20],  # LINE จำกัด 20 ตัวอักษร
                    "text": reply
                }
            }
            for reply in quick_replies[:13]  # LINE รองรับสูงสุด 13 items
        ]
        
        return {
            "type": "text",
            "text": text,
            "quickReply": {
                "items": items
            }
        }

# ตัวอย่างการใช้งาน
bot = EcommerceFAQBot()

test_messages = [
    "อยากสั่งซื้อสินค้า ต้องทำอย่างไร",
    "รับโอนเงินธนาคารอะไรบ้าง",
    "สั่งแล้วกี่วันถึงจะได้รับสินค้า",
    "สวัสดีครับ"
]

for msg in test_messages:
    response = bot.get_response(msg)
    print(f"\nคำถาม: {msg}")
    print(f"หมวดหมู่: {response['category']}")
    print(f"ตอบ: {response['response'][:100]}...")
```

---

## 18.7 Integration กับ Shopify

### Shopify + LINE OA Integration

```python
# Shopify Webhook Handler สำหรับ LINE OA

import hmac
import hashlib
import json
import base64
import requests
from flask import Flask, request, jsonify

app = Flask(__name__)

SHOPIFY_SECRET = 'YOUR_SHOPIFY_WEBHOOK_SECRET'
LINE_ACCESS_TOKEN = 'YOUR_LINE_ACCESS_TOKEN'

# Database จำลอง (ในการใช้งานจริงให้ใช้ Database จริง)
USER_DB = {
    # shopify_customer_id: line_user_id
    '12345': 'Uxxxxxxxxxxx001',
    '12346': 'Uxxxxxxxxxxx002'
}

def verify_shopify_webhook(data: bytes, hmac_header: str) -> bool:
    """
    ตรวจสอบว่า Webhook มาจาก Shopify จริง
    """
    computed_hmac = base64.b64encode(
        hmac.new(
            SHOPIFY_SECRET.encode('utf-8'),
            data,
            hashlib.sha256
        ).digest()
    ).decode()
    
    return hmac.compare_digest(computed_hmac, hmac_header)

def send_line_message(user_id: str, messages: list) -> bool:
    """
    ส่ง Message ผ่าน LINE API
    """
    url = "https://api.line.me/v2/bot/message/push"
    headers = {
        'Authorization': f'Bearer {LINE_ACCESS_TOKEN}',
        'Content-Type': 'application/json'
    }
    payload = {
        'to': user_id,
        'messages': messages
    }
    
    response = requests.post(url, headers=headers, json=payload)
    return response.status_code == 200

@app.route('/webhooks/shopify/order-created', methods=['POST'])
def handle_order_created():
    """
    รับ Webhook เมื่อมีออเดอร์ใหม่จาก Shopify
    """
    # ตรวจสอบ Signature
    hmac_header = request.headers.get('X-Shopify-Hmac-Sha256', '')
    if not verify_shopify_webhook(request.data, hmac_header):
        return jsonify({'error': 'Invalid signature'}), 401
    
    order = request.get_json()
    
    # หา LINE User ID จาก Customer Email/ID
    customer_id = str(order.get('customer', {}).get('id', ''))
    line_user_id = USER_DB.get(customer_id)
    
    if not line_user_id:
        return jsonify({'status': 'no_line_user'}), 200
    
    # สร้างข้อมูลออเดอร์
    order_number = order.get('order_number')
    total = float(order.get('total_price', 0))
    items = order.get('line_items', [])
    
    items_text = "\n".join([
        f"• {item['title']} x{item['quantity']}"
        for item in items[:5]
    ])
    
    # ส่ง Message
    message = {
        "type": "flex",
        "altText": f"✅ ยืนยันออเดอร์ #{order_number}",
        "contents": {
            "type": "bubble",
            "styles": {
                "header": {"backgroundColor": "#00B900"}
            },
            "header": {
                "type": "box",
                "layout": "vertical",
                "contents": [{
                    "type": "text",
                    "text": f"✅ ออเดอร์ #{order_number} ยืนยันแล้ว!",
                    "color": "#ffffff",
                    "weight": "bold",
                    "wrap": True
                }]
            },
            "body": {
                "type": "box",
                "layout": "vertical",
                "spacing": "md",
                "contents": [
                    {
                        "type": "text",
                        "text": items_text,
                        "size": "sm",
                        "wrap": True
                    },
                    {
                        "type": "separator"
                    },
                    {
                        "type": "box",
                        "layout": "horizontal",
                        "contents": [
                            {"type": "text", "text": "ยอดชำระ:", "color": "#666666"},
                            {"type": "text", "text": f"฿{total:,.2f}", "weight": "bold", "color": "#FF4444", "align": "end"}
                        ]
                    }
                ]
            },
            "footer": {
                "type": "box",
                "layout": "vertical",
                "contents": [{
                    "type": "button",
                    "action": {
                        "type": "uri",
                        "label": "ดูออเดอร์",
                        "uri": f"https://yourshop.myshopify.com/orders/{order.get('token')}"
                    }
                }]
            }
        }
    }
    
    send_line_message(line_user_id, [message])
    return jsonify({'status': 'ok'}), 200

@app.route('/webhooks/shopify/fulfillment-created', methods=['POST'])
def handle_fulfillment_created():
    """
    รับ Webhook เมื่อมีการ Fulfill ออเดอร์ (จัดส่งแล้ว)
    """
    hmac_header = request.headers.get('X-Shopify-Hmac-Sha256', '')
    if not verify_shopify_webhook(request.data, hmac_header):
        return jsonify({'error': 'Invalid signature'}), 401
    
    fulfillment = request.get_json()
    order_id = str(fulfillment.get('order_id'))
    
    # ดึง Order Info จาก Shopify API
    tracking_company = fulfillment.get('tracking_company', 'ไม่ระบุ')
    tracking_number = fulfillment.get('tracking_number', 'N/A')
    tracking_url = fulfillment.get('tracking_url', '')
    
    # หา LINE User (ต้อง Query จาก Database จริง)
    # customer_id = get_customer_from_order(order_id)
    # line_user_id = USER_DB.get(customer_id)
    
    # ตัวอย่าง (ใน Production ให้ Query จาก DB จริง)
    line_user_id = 'Uxxxxxxxxxxx001'
    
    if not line_user_id:
        return jsonify({'status': 'no_line_user'}), 200
    
    message = {
        "type": "flex",
        "altText": "🚚 สินค้าจัดส่งแล้ว!",
        "contents": {
            "type": "bubble",
            "body": {
                "type": "box",
                "layout": "vertical",
                "spacing": "md",
                "contents": [
                    {
                        "type": "text",
                        "text": "🚚 สินค้ากำลังจัดส่ง!",
                        "weight": "bold",
                        "size": "xl"
                    },
                    {
                        "type": "box",
                        "layout": "vertical",
                        "spacing": "sm",
                        "contents": [
                            {
                                "type": "box",
                                "layout": "horizontal",
                                "contents": [
                                    {"type": "text", "text": "ขนส่ง:", "size": "sm", "color": "#888888", "flex": 2},
                                    {"type": "text", "text": tracking_company, "size": "sm", "flex": 4}
                                ]
                            },
                            {
                                "type": "box",
                                "layout": "horizontal",
                                "contents": [
                                    {"type": "text", "text": "Tracking:", "size": "sm", "color": "#888888", "flex": 2},
                                    {"type": "text", "text": tracking_number, "size": "sm", "weight": "bold", "flex": 4}
                                ]
                            }
                        ]
                    }
                ]
            },
            "footer": {
                "type": "box",
                "layout": "vertical",
                "contents": [{
                    "type": "button",
                    "style": "primary",
                    "action": {
                        "type": "uri",
                        "label": "ติดตามพัสดุ",
                        "uri": tracking_url or "https://tracking.example.com"
                    }
                }]
            }
        }
    }
    
    send_line_message(line_user_id, [message])
    return jsonify({'status': 'ok'}), 200

if __name__ == '__main__':
    app.run(debug=False, port=5000)
```

---

## 18.8 Integration กับ WooCommerce

### WooCommerce + LINE OA Setup

```php
<?php
/**
 * WooCommerce LINE OA Integration Plugin
 * File: line-oa-integration.php
 */

defined('ABSPATH') || exit;

class WooCommerce_LINE_OA {
    
    private $line_token;
    private $db;
    
    public function __construct() {
        $this->line_token = get_option('line_oa_access_token');
        $this->db = $GLOBALS['wpdb'];
        
        // Hook เข้ากับ WooCommerce Events
        add_action('woocommerce_order_status_changed', [$this, 'on_order_status_changed'], 10, 3);
        add_action('woocommerce_checkout_order_created', [$this, 'on_new_order'], 10, 1);
        add_action('woocommerce_payment_complete', [$this, 'on_payment_complete'], 10, 1);
    }
    
    /**
     * เมื่อมีออเดอร์ใหม่
     */
    public function on_new_order($order) {
        $customer_id = $order->get_customer_id();
        $line_user_id = $this->get_line_user_id($customer_id);
        
        if (!$line_user_id) return;
        
        $order_data = [
            'order_id' => $order->get_order_number(),
            'total' => $order->get_total(),
            'items' => $this->get_order_items_summary($order)
        ];
        
        $this->send_order_confirmation($line_user_id, $order_data);
    }
    
    /**
     * เมื่อสถานะออเดอร์เปลี่ยน
     */
    public function on_order_status_changed($order_id, $old_status, $new_status) {
        $order = wc_get_order($order_id);
        $customer_id = $order->get_customer_id();
        $line_user_id = $this->get_line_user_id($customer_id);
        
        if (!$line_user_id) return;
        
        // แจ้งเตือนตามสถานะที่เปลี่ยน
        switch ($new_status) {
            case 'processing':
                $this->send_text_message($line_user_id, 
                    "✅ ออเดอร์ #" . $order->get_order_number() . " ได้รับการชำระเงินแล้ว กำลังเตรียมสินค้า"
                );
                break;
                
            case 'completed':
                $tracking_number = get_post_meta($order_id, '_tracking_number', true);
                $tracking_company = get_post_meta($order_id, '_tracking_company', true);
                
                if ($tracking_number) {
                    $this->send_shipping_notification($line_user_id, $order, $tracking_company, $tracking_number);
                } else {
                    $this->send_text_message($line_user_id,
                        "🎉 ออเดอร์ #" . $order->get_order_number() . " สำเร็จแล้ว ขอบคุณที่ใช้บริการ!"
                    );
                }
                break;
                
            case 'cancelled':
                $this->send_text_message($line_user_id,
                    "❌ ออเดอร์ #" . $order->get_order_number() . " ถูกยกเลิกแล้ว จะคืนเงินภายใน 3-5 วันทำการ"
                );
                break;
        }
    }
    
    /**
     * ดึง LINE User ID จาก Customer ID
     */
    private function get_line_user_id($customer_id) {
        return get_user_meta($customer_id, 'line_user_id', true);
    }
    
    /**
     * ส่ง Text Message
     */
    private function send_text_message($user_id, $text) {
        return $this->call_line_api($user_id, [
            ['type' => 'text', 'text' => $text]
        ]);
    }
    
    /**
     * ส่ง Order Confirmation
     */
    private function send_order_confirmation($user_id, $order_data) {
        $message = [
            'type' => 'flex',
            'altText' => 'ยืนยันออเดอร์ #' . $order_data['order_id'],
            'contents' => [
                'type' => 'bubble',
                'body' => [
                    'type' => 'box',
                    'layout' => 'vertical',
                    'contents' => [
                        [
                            'type' => 'text',
                            'text' => '✅ ยืนยันออเดอร์แล้ว!',
                            'weight' => 'bold',
                            'size' => 'xl'
                        ],
                        [
                            'type' => 'text',
                            'text' => 'ออเดอร์ #' . $order_data['order_id'],
                            'color' => '#666666'
                        ],
                        [
                            'type' => 'text',
                            'text' => 'ยอดรวม: ฿' . number_format($order_data['total'], 2),
                            'weight' => 'bold',
                            'color' => '#FF4444'
                        ]
                    ]
                ]
            ]
        ];
        
        return $this->call_line_api($user_id, [$message]);
    }
    
    /**
     * เรียก LINE API
     */
    private function call_line_api($user_id, $messages) {
        $response = wp_remote_post(
            'https://api.line.me/v2/bot/message/push',
            [
                'headers' => [
                    'Authorization' => 'Bearer ' . $this->line_token,
                    'Content-Type' => 'application/json'
                ],
                'body' => json_encode([
                    'to' => $user_id,
                    'messages' => $messages
                ]),
                'timeout' => 10
            ]
        );
        
        return !is_wp_error($response) && wp_remote_retrieve_response_code($response) === 200;
    }
    
    /**
     * สรุปรายการสินค้าในออเดอร์
     */
    private function get_order_items_summary($order) {
        $items = [];
        foreach ($order->get_items() as $item) {
            $items[] = $item->get_name() . ' x' . $item->get_quantity();
        }
        return implode(', ', array_slice($items, 0, 3));
    }
}

// เริ่มต้น Plugin
add_action('plugins_loaded', function() {
    if (class_exists('WooCommerce')) {
        new WooCommerce_LINE_OA();
    }
});
```

---

## 18.9 แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Product Catalog Setup
1. สร้าง Product Catalog ใน LINE OA Manager
2. เพิ่มสินค้าอย่างน้อย 5 รายการพร้อมรูปภาพ
3. สร้าง Flex Message แสดง Product Carousel
4. ทดสอบส่ง Message หาตัวเอง

### แบบฝึกหัดที่ 2: Order Notification Flow
1. วางแผน Order Notification Flow ทั้งหมด
2. สร้าง Template Message สำหรับแต่ละสถานะ
3. ทดสอบ Message บน LINE Test Account
4. วัดผล Open Rate ของ Order Notification

### แบบฝึกหัดที่ 3: Abandoned Cart Setup
1. เลือก Platform ที่ใช้ (Shopify/WooCommerce/Custom)
2. Setup Tracking สำหรับ Abandoned Cart
3. สร้าง Recovery Sequence 3 ขั้นตอน
4. ทดสอบ Sequence ครบทุก Step

---

## สรุปบทที่ 18

ในบทนี้เราได้เรียนรู้:
- **LINE Shopping Integration** วิธีการ Setup และใช้งาน
- **Product Catalog** การสร้างและแสดงสินค้าผ่าน Flex Message
- **Order Notification** ระบบแจ้งเตือนออเดอร์อัตโนมัติ
- **Abandoned Cart Recovery** กลยุทธ์กู้คืนยอดขาย
- **Customer Support** การตั้งค่า FAQ Bot และ Auto-Reply
- **Shopify Integration** Webhook Handler พร้อมใช้งาน
- **WooCommerce Integration** PHP Plugin สำหรับ WordPress

**บทต่อไป:** Part 19 - Customer Service Setup

---

*อัปเดตล่าสุด: กันยายน 2026*
*เวอร์ชัน: 1.0*
