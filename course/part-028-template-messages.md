# ส่วนที่ 28: Template Messages

---

## สารบัญ

1. [ภาพรวม Template Messages](#1-ภาพรวม-template-messages)
2. [Button Template](#2-button-template)
3. [Confirm Template](#3-confirm-template)
4. [Carousel Template](#4-carousel-template)
5. [Image Carousel Template](#5-image-carousel-template)
6. [Action Types](#6-action-types)
7. [Code Examples ใน Node.js](#7-code-examples-ใน-nodejs)
8. [Code Examples ใน Python](#8-code-examples-ใน-python)
9. [Best Practices](#9-best-practices)
10. [แบบฝึกหัดท้ายบท](#10-แบบฝึกหัดท้ายบท)

---

## 1. ภาพรวม Template Messages

Template Messages คือข้อความที่มีโครงสร้างสำเร็จรูป ประกอบด้วยรูปภาพ ข้อความ และปุ่มกด ทำให้ Bot สามารถสร้าง Interactive UI ได้โดยไม่ต้องใช้ Flex Message ที่ซับซ้อน

### ประเภท Template Messages

| Template | ปุ่มสูงสุด | มีรูป | ใช้งาน |
|---------|---------|------|-------|
| Buttons | 4 | ✅ (optional) | เมนูเลือก |
| Confirm | 2 (fixed) | ❌ | ยืนยัน yes/no |
| Carousel | 10 columns × 3 buttons | ✅ | แสดงหลายรายการ |
| Image Carousel | 10 columns | ✅ | Gallery รูปภาพ |

### ข้อจำกัดทั่วไปของ Template Messages

- Template Messages รองรับเฉพาะ **LINE iOS/Android** เท่านั้น
- บน Desktop และ Web บางรายการอาจแสดงไม่ครบ
- สำหรับ advanced UI ควรใช้ **Flex Message** แทน

---

## 2. Button Template

Button Template แสดงรูปภาพ (optional), ชื่อ, ข้อความ และปุ่มกด สูงสุด 4 ปุ่ม

### โครงสร้าง Button Template

```json
{
  "type": "template",
  "altText": "This is a buttons template",
  "template": {
    "type": "buttons",
    "thumbnailImageUrl": "https://example.com/image.jpg",
    "imageAspectRatio": "rectangle",
    "imageSize": "cover",
    "imageBackgroundColor": "#FFFFFF",
    "title": "เลือกบริการ",
    "text": "กรุณาเลือกบริการที่ต้องการ",
    "defaultAction": {
      "type": "uri",
      "label": "ดูเพิ่มเติม",
      "uri": "https://example.com"
    },
    "actions": [
      {
        "type": "postback",
        "label": "ซื้อเลย",
        "data": "action=buy&itemId=123"
      },
      {
        "type": "postback",
        "label": "เพิ่มใน Wishlist",
        "data": "action=addWishlist&itemId=123"
      },
      {
        "type": "uri",
        "label": "ดูรายละเอียด",
        "uri": "https://example.com/products/123"
      },
      {
        "type": "message",
        "label": "ติดต่อเรา",
        "text": "ฉันต้องการติดต่อเพื่อสอบถามเกี่ยวกับสินค้า"
      }
    ]
  }
}
```

### Parameters ของ Button Template

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| thumbnailImageUrl | String | No | URL รูปภาพ thumbnail |
| imageAspectRatio | String | No | "rectangle" (51:26) หรือ "square" |
| imageSize | String | No | "cover" หรือ "contain" |
| imageBackgroundColor | String | No | สีพื้นหลัง (hex) |
| title | String | No | ชื่อ (สูงสุด 40 ตัวอักษร) |
| text | String | Yes | ข้อความ (สูงสุด 160 ตัวอักษร) |
| defaultAction | Object | No | Action เมื่อแตะที่รูปหรือชื่อ |
| actions | Array | Yes | ปุ่ม (1-4 ปุ่ม) |

### ตัวอย่าง Button Templates สำหรับ Use Cases ต่างๆ

#### Menu หลัก

```json
{
  "type": "template",
  "altText": "เมนูหลัก",
  "template": {
    "type": "buttons",
    "title": "เมนูหลัก",
    "text": "เลือกสิ่งที่คุณต้องการ",
    "actions": [
      {
        "type": "message",
        "label": "🛍️ ดูสินค้า",
        "text": "ดูสินค้า"
      },
      {
        "type": "message",
        "label": "📦 ตรวจสอบคำสั่งซื้อ",
        "text": "ตรวจสอบคำสั่งซื้อ"
      },
      {
        "type": "message",
        "label": "💬 ติดต่อ Support",
        "text": "ติดต่อ support"
      },
      {
        "type": "uri",
        "label": "🌐 เว็บไซต์",
        "uri": "https://example.com"
      }
    ]
  }
}
```

#### Product Card

```json
{
  "type": "template",
  "altText": "สินค้าแนะนำ: นาฬิกา Smart Watch",
  "template": {
    "type": "buttons",
    "thumbnailImageUrl": "https://cdn.example.com/products/smartwatch.jpg",
    "imageAspectRatio": "rectangle",
    "imageSize": "cover",
    "title": "Smart Watch Pro X1",
    "text": "นาฬิกาอัจฉริยะ กันน้ำ 50m\nแบตเตอรี่ 14 วัน\nราคา 4,990 บาท",
    "defaultAction": {
      "type": "uri",
      "label": "ดูรายละเอียด",
      "uri": "https://example.com/products/smartwatch-pro-x1"
    },
    "actions": [
      {
        "type": "postback",
        "label": "🛒 หยิบใส่ตะกร้า",
        "data": "action=addCart&productId=SW001&price=4990"
      },
      {
        "type": "postback",
        "label": "❤️ บันทึก Wishlist",
        "data": "action=addWishlist&productId=SW001"
      },
      {
        "type": "uri",
        "label": "📋 รายละเอียดเพิ่มเติม",
        "uri": "https://example.com/products/smartwatch-pro-x1"
      }
    ]
  }
}
```

#### Appointment Booking

```json
{
  "type": "template",
  "altText": "จองคิวนัดหมาย",
  "template": {
    "type": "buttons",
    "thumbnailImageUrl": "https://cdn.example.com/clinic-banner.jpg",
    "title": "จองคิวนัดหมาย",
    "text": "เลือกประเภทการนัดหมายที่ต้องการ",
    "actions": [
      {
        "type": "datetimepicker",
        "label": "📅 เลือกวันและเวลา",
        "data": "action=selectDateTime&type=appointment",
        "mode": "datetime",
        "initial": "2024-12-01T10:00",
        "max": "2025-12-31T23:59",
        "min": "2024-01-01T00:00"
      },
      {
        "type": "postback",
        "label": "📋 ดูคิวของฉัน",
        "data": "action=viewAppointments"
      },
      {
        "type": "postback",
        "label": "❌ ยกเลิกนัดหมาย",
        "data": "action=cancelAppointment"
      }
    ]
  }
}
```

---

## 3. Confirm Template

Confirm Template ใช้สำหรับการยืนยัน Yes/No มีปุ่ม 2 ปุ่มเสมอ เหมาะสำหรับการถามยืนยันก่อนทำรายการสำคัญ

### โครงสร้าง Confirm Template

```json
{
  "type": "template",
  "altText": "ยืนยันคำสั่งซื้อ",
  "template": {
    "type": "confirm",
    "text": "ยืนยันการสั่งซื้อสินค้า?\n\nสินค้า: Smart Watch Pro X1\nราคา: 4,990 บาท",
    "actions": [
      {
        "type": "postback",
        "label": "✅ ยืนยัน",
        "data": "action=confirm&orderId=12345"
      },
      {
        "type": "postback",
        "label": "❌ ยกเลิก",
        "data": "action=cancel&orderId=12345"
      }
    ]
  }
}
```

### Parameters ของ Confirm Template

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| text | String | Yes | ข้อความ (สูงสุด 240 ตัวอักษร) |
| actions | Array | Yes | ปุ่ม (ต้องมีพอดี 2 ปุ่ม) |

### ตัวอย่าง Confirm Templates

#### ยืนยันการสั่งซื้อ

```json
{
  "type": "template",
  "altText": "ยืนยันคำสั่งซื้อ #12345",
  "template": {
    "type": "confirm",
    "text": "ยืนยันคำสั่งซื้อ #12345\n\n📦 Smart Watch Pro X1 × 1\n💰 ยอดรวม: 4,990 บาท\n🚚 ส่งที่: กรุงเทพฯ",
    "actions": [
      {
        "type": "postback",
        "label": "✅ ยืนยันคำสั่งซื้อ",
        "data": "action=confirmOrder&orderId=12345"
      },
      {
        "type": "postback",
        "label": "✏️ แก้ไข",
        "data": "action=editOrder&orderId=12345"
      }
    ]
  }
}
```

#### ยืนยันการยกเลิก

```json
{
  "type": "template",
  "altText": "ยืนยันการยกเลิก",
  "template": {
    "type": "confirm",
    "text": "คุณต้องการยกเลิกคำสั่งซื้อ #12345 ใช่หรือไม่?\n\nหากยกเลิกแล้วไม่สามารถกู้คืนได้",
    "actions": [
      {
        "type": "postback",
        "label": "ใช่ ยกเลิก",
        "data": "action=confirmCancel&orderId=12345"
      },
      {
        "type": "postback",
        "label": "ไม่ กลับไป",
        "data": "action=keepOrder&orderId=12345"
      }
    ]
  }
}
```

#### Subscribe/Unsubscribe

```json
{
  "type": "template",
  "altText": "สมัครรับข่าวสาร",
  "template": {
    "type": "confirm",
    "text": "ต้องการรับข่าวสารและโปรโมชั่นจากเราหรือไม่?\n\nคุณสามารถยกเลิกได้ทุกเมื่อ",
    "actions": [
      {
        "type": "postback",
        "label": "✅ ใช่ ต้องการ",
        "data": "action=subscribe&type=newsletter"
      },
      {
        "type": "postback",
        "label": "❌ ไม่ ขอบคุณ",
        "data": "action=skipSubscribe"
      }
    ]
  }
}
```

---

## 4. Carousel Template

Carousel Template แสดงข้อมูลหลายรายการในแนวนอน (scroll ได้) แต่ละ column มีรูปภาพ ชื่อ และปุ่ม

### โครงสร้าง Carousel Template

```json
{
  "type": "template",
  "altText": "สินค้าแนะนำ",
  "template": {
    "type": "carousel",
    "imageAspectRatio": "rectangle",
    "imageSize": "cover",
    "columns": [
      {
        "thumbnailImageUrl": "https://example.com/product1.jpg",
        "imageBackgroundColor": "#FFFFFF",
        "title": "สินค้า 1",
        "text": "รายละเอียดสินค้า 1",
        "defaultAction": {
          "type": "uri",
          "label": "ดูรายละเอียด",
          "uri": "https://example.com/1"
        },
        "actions": [
          {
            "type": "postback",
            "label": "ซื้อ",
            "data": "action=buy&id=1"
          },
          {
            "type": "uri",
            "label": "ดูเพิ่มเติม",
            "uri": "https://example.com/1"
          }
        ]
      },
      {
        "thumbnailImageUrl": "https://example.com/product2.jpg",
        "title": "สินค้า 2",
        "text": "รายละเอียดสินค้า 2",
        "actions": [
          {
            "type": "postback",
            "label": "ซื้อ",
            "data": "action=buy&id=2"
          },
          {
            "type": "uri",
            "label": "ดูเพิ่มเติม",
            "uri": "https://example.com/2"
          }
        ]
      }
    ]
  }
}
```

### Parameters ของ Carousel

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| imageAspectRatio | String | No | "rectangle" หรือ "square" |
| imageSize | String | No | "cover" หรือ "contain" |
| columns | Array | Yes | รายการ column (1-10) |

### Parameters ของแต่ละ Column

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| thumbnailImageUrl | String | No | URL รูปภาพ |
| imageBackgroundColor | String | No | สีพื้นหลัง (hex) |
| title | String | No | ชื่อ (สูงสุด 40 ตัวอักษร) |
| text | String | Yes | ข้อความ (สูงสุด 120 ตัวอักษร) |
| defaultAction | Object | No | Action เมื่อแตะ column |
| actions | Array | Yes | ปุ่ม (1-3 ปุ่ม) |

### ตัวอย่าง Carousel สำหรับ E-commerce

```json
{
  "type": "template",
  "altText": "สินค้าแนะนำประจำสัปดาห์",
  "template": {
    "type": "carousel",
    "imageAspectRatio": "rectangle",
    "imageSize": "cover",
    "columns": [
      {
        "thumbnailImageUrl": "https://cdn.shop.com/products/watch.jpg",
        "imageBackgroundColor": "#F8F8F8",
        "title": "Smart Watch Pro X1",
        "text": "กันน้ำ 50m | แบต 14 วัน\n💰 4,990 บาท (ลด 30%)",
        "defaultAction": {
          "type": "uri",
          "label": "ดูสินค้า",
          "uri": "https://shop.com/products/sw-pro-x1"
        },
        "actions": [
          {
            "type": "postback",
            "label": "🛒 ซื้อเลย",
            "data": "action=buy&productId=SW001&price=4990"
          },
          {
            "type": "postback",
            "label": "❤️ Wishlist",
            "data": "action=wishlist&productId=SW001"
          },
          {
            "type": "uri",
            "label": "📋 รายละเอียด",
            "uri": "https://shop.com/products/sw-pro-x1"
          }
        ]
      },
      {
        "thumbnailImageUrl": "https://cdn.shop.com/products/earbuds.jpg",
        "imageBackgroundColor": "#F8F8F8",
        "title": "TWS Earbuds Ultra",
        "text": "ANC | 30hr battery\n💰 2,490 บาท",
        "actions": [
          {
            "type": "postback",
            "label": "🛒 ซื้อเลย",
            "data": "action=buy&productId=EB002&price=2490"
          },
          {
            "type": "postback",
            "label": "❤️ Wishlist",
            "data": "action=wishlist&productId=EB002"
          },
          {
            "type": "uri",
            "label": "📋 รายละเอียด",
            "uri": "https://shop.com/products/tws-ultra"
          }
        ]
      },
      {
        "thumbnailImageUrl": "https://cdn.shop.com/products/powerbank.jpg",
        "imageBackgroundColor": "#F8F8F8",
        "title": "Power Bank 20000mAh",
        "text": "PD 65W | 3 ช่องชาร์จ\n💰 1,290 บาท",
        "actions": [
          {
            "type": "postback",
            "label": "🛒 ซื้อเลย",
            "data": "action=buy&productId=PB003&price=1290"
          },
          {
            "type": "postback",
            "label": "❤️ Wishlist",
            "data": "action=wishlist&productId=PB003"
          },
          {
            "type": "uri",
            "label": "📋 รายละเอียด",
            "uri": "https://shop.com/products/powerbank-20k"
          }
        ]
      }
    ]
  }
}
```

### Carousel สำหรับ Service Menu

```json
{
  "type": "template",
  "altText": "บริการของเรา",
  "template": {
    "type": "carousel",
    "columns": [
      {
        "thumbnailImageUrl": "https://cdn.example.com/services/delivery.jpg",
        "title": "🚚 บริการส่งด่วน",
        "text": "ส่งภายใน 2 ชั่วโมง\nทุกวันทุกเวลา",
        "actions": [
          {
            "type": "postback",
            "label": "สั่งส่งด่วน",
            "data": "service=express_delivery"
          }
        ]
      },
      {
        "thumbnailImageUrl": "https://cdn.example.com/services/repair.jpg",
        "title": "🔧 บริการซ่อม",
        "text": "ซ่อมอุปกรณ์ทุกชนิด\nรับประกัน 6 เดือน",
        "actions": [
          {
            "type": "postback",
            "label": "นัดซ่อม",
            "data": "service=repair"
          }
        ]
      },
      {
        "thumbnailImageUrl": "https://cdn.example.com/services/consult.jpg",
        "title": "💬 ปรึกษาผู้เชี่ยวชาญ",
        "text": "ฟรี ไม่มีค่าใช้จ่าย\nตลอด 24/7",
        "actions": [
          {
            "type": "postback",
            "label": "ขอคำปรึกษา",
            "data": "service=consult"
          }
        ]
      }
    ]
  }
}
```

---

## 5. Image Carousel Template

Image Carousel Template แสดงเฉพาะรูปภาพหลายรูปในแนวนอน แต่ละรูปมี action 1 อย่าง

### โครงสร้าง Image Carousel Template

```json
{
  "type": "template",
  "altText": "โปรโมชั่น",
  "template": {
    "type": "image_carousel",
    "columns": [
      {
        "imageUrl": "https://example.com/promo1.jpg",
        "action": {
          "type": "uri",
          "label": "ดูโปรโมชั่น 1",
          "uri": "https://example.com/promo1"
        }
      },
      {
        "imageUrl": "https://example.com/promo2.jpg",
        "action": {
          "type": "postback",
          "label": "โปรโมชั่น 2",
          "data": "action=promo&id=2"
        }
      },
      {
        "imageUrl": "https://example.com/promo3.jpg",
        "action": {
          "type": "message",
          "label": "โปรโมชั่น 3",
          "text": "ต้องการข้อมูลเกี่ยวกับโปรโมชั่น 3"
        }
      }
    ]
  }
}
```

### Parameters ของ Image Carousel

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| columns | Array | Yes | รายการ column (1-10) |
| imageUrl | String | Yes | URL รูปภาพ (สี่เหลี่ยมจัตุรัส) |
| action | Object | Yes | Action เมื่อแตะรูป |

### ข้อกำหนดรูปภาพ Image Carousel

- **Format**: JPEG หรือ PNG
- **Size สูงสุด**: 1 MB
- **Aspect Ratio**: 1:1 (สี่เหลี่ยมจัตุรัส)
- **Resolution แนะนำ**: 1024x1024

### ตัวอย่าง Image Carousel สำหรับ Gallery

```json
{
  "type": "template",
  "altText": "คอลเลกชันฤดูกาลใหม่",
  "template": {
    "type": "image_carousel",
    "columns": [
      {
        "imageUrl": "https://cdn.fashion.com/spring/look1.jpg",
        "action": {
          "type": "uri",
          "label": "ดู Look 1",
          "uri": "https://fashion.com/spring/look1"
        }
      },
      {
        "imageUrl": "https://cdn.fashion.com/spring/look2.jpg",
        "action": {
          "type": "uri",
          "label": "ดู Look 2",
          "uri": "https://fashion.com/spring/look2"
        }
      },
      {
        "imageUrl": "https://cdn.fashion.com/spring/look3.jpg",
        "action": {
          "type": "postback",
          "label": "สนใจ Look 3",
          "data": "action=inquiry&look=3"
        }
      },
      {
        "imageUrl": "https://cdn.fashion.com/spring/look4.jpg",
        "action": {
          "type": "message",
          "label": "ถามเรื่อง Look 4",
          "text": "ขอข้อมูล Look 4 จากคอลเลกชันฤดูใบไม้ผลิ"
        }
      }
    ]
  }
}
```

---

## 6. Action Types

Template Messages รองรับ Action ต่างๆ สำหรับปุ่มและ defaultAction

### 1. URI Action

เปิด URL ใน Browser หรือ App

```json
{
  "type": "uri",
  "label": "เปิดเว็บไซต์",
  "uri": "https://example.com",
  "altUri": {
    "desktop": "https://desktop.example.com"
  }
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| type | String | Yes | "uri" |
| label | String | Yes | ชื่อปุ่ม (สูงสุด 20 ตัวอักษร) |
| uri | String | Yes | URL ที่จะเปิด |
| altUri | Object | No | URL สำรองสำหรับ Desktop |

URI Schemes ที่รองรับ:
- `https://` - เปิดใน LINE Browser
- `tel:` - โทรออก
- `mailto:` - ส่งอีเมล
- `line://` - เปิด LINE app

```json
// โทรออก
{
  "type": "uri",
  "label": "โทรหาเรา",
  "uri": "tel:+6621234567"
}

// ส่งอีเมล
{
  "type": "uri",
  "label": "ส่งอีเมล",
  "uri": "mailto:support@example.com"
}

// เปิด LINE OA อีกอัน
{
  "type": "uri",
  "label": "ติดต่อ Support",
  "uri": "https://line.me/R/ti/p/@support"
}
```

### 2. Message Action

ส่งข้อความในนามผู้ใช้

```json
{
  "type": "message",
  "label": "ดูเมนู",
  "text": "แสดงเมนูทั้งหมด"
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| type | String | Yes | "message" |
| label | String | Yes | ชื่อปุ่ม |
| text | String | Yes | ข้อความที่จะส่ง (สูงสุด 300 ตัวอักษร) |

### 3. Postback Action

ส่งข้อมูล (data) กลับมายัง Bot โดยที่ผู้ใช้ไม่เห็น

```json
{
  "type": "postback",
  "label": "เพิ่มในตะกร้า",
  "data": "action=add_cart&product_id=123&qty=1",
  "displayText": "เพิ่มสินค้าในตะกร้าแล้ว",
  "inputOption": "openKeyboard",
  "fillInText": "จำนวน: "
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| type | String | Yes | "postback" |
| label | String | Yes | ชื่อปุ่ม |
| data | String | Yes | ข้อมูล postback (สูงสุด 300 ตัวอักษร) |
| displayText | String | No | ข้อความที่แสดงใน chat (สูงสุด 300 ตัวอักษร) |
| inputOption | String | No | "closeRichMenu", "openRichMenu", "openKeyboard", "openVoice" |
| fillInText | String | No | ข้อความที่ fill ใน text input |

### 4. Datetime Picker Action

เปิด datetime picker ให้ผู้ใช้เลือกวันที่/เวลา

```json
{
  "type": "datetimepicker",
  "label": "เลือกวันนัดหมาย",
  "data": "action=selectDate&type=appointment",
  "mode": "datetime",
  "initial": "2024-12-01T10:00",
  "max": "2025-12-31T23:59",
  "min": "2024-11-01T00:00"
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| type | String | Yes | "datetimepicker" |
| label | String | Yes | ชื่อปุ่ม |
| data | String | Yes | ข้อมูล postback |
| mode | String | Yes | "date", "time", "datetime" |
| initial | String | No | ค่าเริ่มต้น |
| max | String | No | ค่าสูงสุด |
| min | String | No | ค่าต่ำสุด |

Format ตาม mode:
- `date`: `yyyy-MM-dd` (เช่น 2024-12-01)
- `time`: `HH:mm` (เช่น 10:00)
- `datetime`: `yyyy-MM-ddTHH:mm` (เช่น 2024-12-01T10:00)

### 5. Camera Action

เปิดกล้องถ่ายภาพ

```json
{
  "type": "camera",
  "label": "ถ่ายรูป"
}
```

### 6. Camera Roll Action

เปิด Gallery รูปภาพ

```json
{
  "type": "cameraRoll",
  "label": "เลือกรูปจาก Gallery"
}
```

### 7. Location Action

เปิด Location picker

```json
{
  "type": "location",
  "label": "ส่งตำแหน่งของฉัน"
}
```

---

## 7. Code Examples ใน Node.js

### templates/buttonTemplate.js

```javascript
'use strict';

/**
 * สร้าง Button Template Message
 */
function createButtonTemplate({
  title,
  text,
  thumbnailUrl,
  imageAspectRatio = 'rectangle',
  imageSize = 'cover',
  defaultAction,
  actions
}) {
  const template = {
    type: 'buttons',
    text,
    actions: actions.slice(0, 4)  // สูงสุด 4 ปุ่ม
  };

  if (title) template.title = title;
  if (thumbnailUrl) {
    template.thumbnailImageUrl = thumbnailUrl;
    template.imageAspectRatio = imageAspectRatio;
    template.imageSize = imageSize;
  }
  if (defaultAction) template.defaultAction = defaultAction;

  return {
    type: 'template',
    altText: title || text,
    template
  };
}

/**
 * สร้าง Confirm Template Message
 */
function createConfirmTemplate(text, confirmAction, cancelAction) {
  return {
    type: 'template',
    altText: text,
    template: {
      type: 'confirm',
      text,
      actions: [confirmAction, cancelAction]
    }
  };
}

/**
 * สร้าง Carousel Template Message
 */
function createCarouselTemplate(columns, options = {}) {
  const template = {
    type: 'carousel',
    columns: columns.slice(0, 10)  // สูงสุด 10 columns
  };

  if (options.imageAspectRatio) template.imageAspectRatio = options.imageAspectRatio;
  if (options.imageSize) template.imageSize = options.imageSize;

  return {
    type: 'template',
    altText: options.altText || 'Carousel',
    template
  };
}

/**
 * สร้าง Image Carousel Template Message
 */
function createImageCarouselTemplate(columns, altText = 'Image Carousel') {
  return {
    type: 'template',
    altText,
    template: {
      type: 'image_carousel',
      columns: columns.slice(0, 10)
    }
  };
}

// Action builders
function uriAction(label, uri, altDesktopUri) {
  const action = { type: 'uri', label, uri };
  if (altDesktopUri) action.altUri = { desktop: altDesktopUri };
  return action;
}

function messageAction(label, text) {
  return { type: 'message', label, text };
}

function postbackAction(label, data, displayText, inputOption) {
  const action = { type: 'postback', label, data };
  if (displayText) action.displayText = displayText;
  if (inputOption) action.inputOption = inputOption;
  return action;
}

function datetimePickerAction(label, data, mode, options = {}) {
  const action = { type: 'datetimepicker', label, data, mode };
  if (options.initial) action.initial = options.initial;
  if (options.max) action.max = options.max;
  if (options.min) action.min = options.min;
  return action;
}

function cameraAction(label) {
  return { type: 'camera', label };
}

function cameraRollAction(label) {
  return { type: 'cameraRoll', label };
}

function locationAction(label) {
  return { type: 'location', label };
}

module.exports = {
  createButtonTemplate,
  createConfirmTemplate,
  createCarouselTemplate,
  createImageCarouselTemplate,
  uriAction,
  messageAction,
  postbackAction,
  datetimePickerAction,
  cameraAction,
  cameraRollAction,
  locationAction
};
```

### handlers/shopHandler.js - ตัวอย่างร้านค้า

```javascript
'use strict';

const {
  createButtonTemplate,
  createConfirmTemplate,
  createCarouselTemplate,
  createImageCarouselTemplate,
  uriAction,
  messageAction,
  postbackAction,
  datetimePickerAction,
  cameraAction,
  cameraRollAction
} = require('../templates/buttonTemplate');

// ฐานข้อมูลสินค้า (ตัวอย่าง)
const PRODUCTS = [
  {
    id: 'P001',
    name: 'Smart Watch Pro X1',
    price: 4990,
    description: 'กันน้ำ 50m | แบต 14 วัน | GPS',
    imageUrl: 'https://picsum.photos/id/100/800/500',
    url: 'https://shop.example.com/products/P001'
  },
  {
    id: 'P002',
    name: 'TWS Earbuds Ultra',
    price: 2490,
    description: 'ANC | 30hr battery | Hi-Fi',
    imageUrl: 'https://picsum.photos/id/101/800/500',
    url: 'https://shop.example.com/products/P002'
  },
  {
    id: 'P003',
    name: 'Power Bank 20000mAh',
    price: 1290,
    description: 'PD 65W | 3 ช่องชาร์จ | Fast Charge',
    imageUrl: 'https://picsum.photos/id/102/800/500',
    url: 'https://shop.example.com/products/P003'
  }
];

/**
 * ส่ง Main Menu
 */
async function sendMainMenu(replyToken, client) {
  const menuTemplate = createButtonTemplate({
    title: '🏪 ร้านของเรา',
    text: 'ยินดีต้อนรับ! เลือกสิ่งที่ต้องการ',
    thumbnailUrl: 'https://picsum.photos/id/1/800/500',
    actions: [
      messageAction('🛍️ ดูสินค้าทั้งหมด', 'ดูสินค้า'),
      messageAction('📦 ตรวจสอบคำสั่งซื้อ', 'คำสั่งซื้อของฉัน'),
      messageAction('🎁 โปรโมชั่น', 'ดูโปรโมชั่น'),
      uriAction('🌐 เยี่ยมชมเว็บ', 'https://shop.example.com')
    ]
  });

  return client.replyMessage({
    replyToken,
    messages: [menuTemplate]
  });
}

/**
 * ส่ง Product Carousel
 */
async function sendProductCarousel(replyToken, client) {
  const columns = PRODUCTS.map(product => ({
    thumbnailImageUrl: product.imageUrl,
    imageBackgroundColor: '#F8F8F8',
    title: product.name,
    text: `${product.description}\n💰 ${product.price.toLocaleString()} บาท`,
    defaultAction: uriAction('ดูรายละเอียด', product.url),
    actions: [
      postbackAction('🛒 ซื้อเลย', `action=buy&id=${product.id}&price=${product.price}`, `เพิ่ม ${product.name} ในตะกร้า`),
      postbackAction('❤️ Wishlist', `action=wishlist&id=${product.id}`),
      uriAction('📋 รายละเอียด', product.url)
    ]
  }));

  const carousel = createCarouselTemplate(columns, {
    altText: 'สินค้าแนะนำ',
    imageAspectRatio: 'rectangle',
    imageSize: 'cover'
  });

  return client.replyMessage({
    replyToken,
    messages: [carousel]
  });
}

/**
 * ส่ง Confirm สำหรับการซื้อ
 */
async function sendPurchaseConfirm(replyToken, client, productId, quantity = 1) {
  const product = PRODUCTS.find(p => p.id === productId);

  if (!product) {
    return client.replyMessage({
      replyToken,
      messages: [{ type: 'text', text: 'ไม่พบสินค้านี้' }]
    });
  }

  const total = product.price * quantity;

  const confirmTemplate = createConfirmTemplate(
    `ยืนยันการสั่งซื้อ?\n\n📦 ${product.name} × ${quantity}\n💰 ยอดรวม: ${total.toLocaleString()} บาท`,
    postbackAction('✅ ยืนยัน', `action=confirm_buy&id=${productId}&qty=${quantity}`, 'ยืนยันคำสั่งซื้อแล้ว'),
    postbackAction('❌ ยกเลิก', `action=cancel_buy&id=${productId}`, 'ยกเลิกแล้ว')
  );

  return client.replyMessage({
    replyToken,
    messages: [confirmTemplate]
  });
}

/**
 * ส่ง Appointment Booking
 */
async function sendAppointmentBooking(replyToken, client) {
  const now = new Date();
  const minDate = now.toISOString().slice(0, 16);
  const maxDate = new Date(now.getTime() + 90 * 24 * 60 * 60 * 1000).toISOString().slice(0, 16);

  const bookingTemplate = createButtonTemplate({
    title: '📅 จองนัดหมาย',
    text: 'เลือกวันและเวลาที่สะดวก',
    thumbnailUrl: 'https://picsum.photos/id/50/800/500',
    actions: [
      datetimePickerAction('📅 เลือกวันและเวลา', 'action=book&type=new', 'datetime', {
        initial: minDate,
        min: minDate,
        max: maxDate
      }),
      postbackAction('📋 ดูนัดหมายของฉัน', 'action=view_appointments'),
      postbackAction('❌ ยกเลิกนัดหมาย', 'action=cancel_appointment')
    ]
  });

  return client.replyMessage({
    replyToken,
    messages: [bookingTemplate]
  });
}

/**
 * ส่ง Image Carousel สำหรับ Promotions
 */
async function sendPromotionGallery(replyToken, client) {
  const promotions = [
    { imageUrl: 'https://picsum.photos/id/200/1040/1040', id: 'PROMO1' },
    { imageUrl: 'https://picsum.photos/id/201/1040/1040', id: 'PROMO2' },
    { imageUrl: 'https://picsum.photos/id/202/1040/1040', id: 'PROMO3' },
    { imageUrl: 'https://picsum.photos/id/203/1040/1040', id: 'PROMO4' }
  ];

  const columns = promotions.map(promo => ({
    imageUrl: promo.imageUrl,
    action: postbackAction(`โปรโมชั่น ${promo.id}`, `action=view_promo&id=${promo.id}`)
  }));

  const imageCarousel = createImageCarouselTemplate(columns, 'โปรโมชั่นสุดพิเศษ');

  return client.replyMessage({
    replyToken,
    messages: [imageCarousel]
  });
}

/**
 * จัดการ Postback Event
 */
async function handlePostback(event, client) {
  const { replyToken, postback } = event;
  const params = new URLSearchParams(postback.data);
  const action = params.get('action');

  switch (action) {
    case 'buy':
      const productId = params.get('id');
      const qty = parseInt(params.get('qty') || '1');
      return sendPurchaseConfirm(replyToken, client, productId, qty);

    case 'confirm_buy':
      const confirmedId = params.get('id');
      const confirmedQty = parseInt(params.get('qty'));
      return processOrder(replyToken, client, confirmedId, confirmedQty);

    case 'cancel_buy':
      return client.replyMessage({
        replyToken,
        messages: [{ type: 'text', text: '❌ ยกเลิกการสั่งซื้อแล้ว' }]
      });

    case 'wishlist':
      return client.replyMessage({
        replyToken,
        messages: [{ type: 'text', text: '❤️ เพิ่มใน Wishlist แล้ว!' }]
      });

    case 'book':
      const selectedDate = postback.params?.datetime;
      return handleDatetimePicker(replyToken, client, selectedDate);

    default:
      return client.replyMessage({
        replyToken,
        messages: [{ type: 'text', text: `รับทราบ: ${postback.data}` }]
      });
  }
}

/**
 * ประมวลผลคำสั่งซื้อ
 */
async function processOrder(replyToken, client, productId, quantity) {
  // สร้างคำสั่งซื้อ
  const orderId = Date.now().toString().slice(-8);
  const product = PRODUCTS.find(p => p.id === productId);

  return client.replyMessage({
    replyToken,
    messages: [
      {
        type: 'text',
        text: `✅ สั่งซื้อสำเร็จ!\n\n🔖 หมายเลขคำสั่งซื้อ: #${orderId}\n📦 ${product.name} × ${quantity}\n💰 ยอดรวม: ${(product.price * quantity).toLocaleString()} บาท\n\nเราจะส่งข้อความยืนยันให้เร็วๆ นี้`
      }
    ]
  });
}

/**
 * จัดการ Datetime Picker Result
 */
async function handleDatetimePicker(replyToken, client, datetime) {
  if (!datetime) {
    return client.replyMessage({
      replyToken,
      messages: [{ type: 'text', text: '❌ กรุณาเลือกวันที่และเวลา' }]
    });
  }

  const selectedDate = new Date(datetime);
  const formattedDate = selectedDate.toLocaleString('th-TH', {
    timeZone: 'Asia/Bangkok',
    year: 'numeric',
    month: 'long',
    day: 'numeric',
    hour: '2-digit',
    minute: '2-digit'
  });

  return client.replyMessage({
    replyToken,
    messages: [
      {
        type: 'text',
        text: `✅ บันทึกนัดหมายแล้ว!\n\n📅 วันที่: ${formattedDate}\n\nเราจะส่งการยืนยันให้คุณทาง LINE`
      }
    ]
  });
}

module.exports = {
  sendMainMenu,
  sendProductCarousel,
  sendPurchaseConfirm,
  sendAppointmentBooking,
  sendPromotionGallery,
  handlePostback
};
```

### app.js - Integration ทั้งหมด

```javascript
'use strict';

require('dotenv').config();
const express = require('express');
const line = require('@line/bot-sdk');
const shopHandler = require('./handlers/shopHandler');

const app = express();

const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};

const client = new line.messagingApi.MessagingApiClient({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

app.post('/webhook', line.middleware(config), async (req, res) => {
  try {
    await Promise.all(req.body.events.map(event => handleEvent(event)));
    res.json({ status: 'ok' });
  } catch (err) {
    console.error(err);
    res.status(500).end();
  }
});

async function handleEvent(event) {
  const { type, replyToken } = event;

  if (type === 'message' && event.message.type === 'text') {
    const text = event.message.text.trim();

    switch (text) {
      case 'เมนู':
      case 'menu':
        return shopHandler.sendMainMenu(replyToken, client);

      case 'ดูสินค้า':
        return shopHandler.sendProductCarousel(replyToken, client);

      case 'ดูโปรโมชั่น':
        return shopHandler.sendPromotionGallery(replyToken, client);

      case 'นัดหมาย':
        return shopHandler.sendAppointmentBooking(replyToken, client);

      default:
        return client.replyMessage({
          replyToken,
          messages: [{ type: 'text', text: 'พิมพ์ "เมนู" เพื่อเริ่มต้น' }]
        });
    }
  }

  if (type === 'postback') {
    return shopHandler.handlePostback(event, client);
  }
}

app.listen(3000, () => console.log('Shop bot running on port 3000'));
```

---

## 8. Code Examples ใน Python

### templates/template_builder.py

```python
from typing import Optional, List, Dict, Any
from linebot.v3.messaging import (
    TemplateMessage, ButtonsTemplate, ConfirmTemplate,
    CarouselTemplate, CarouselColumn, ImageCarouselTemplate,
    ImageCarouselColumn, URIAction, MessageAction, PostbackAction,
    DatetimePickerAction, CameraAction, CameraRollAction,
    LocationAction
)


def button_template(
    text: str,
    actions: List,
    title: Optional[str] = None,
    thumbnail_url: Optional[str] = None,
    image_aspect_ratio: str = 'rectangle',
    image_size: str = 'cover',
    default_action=None
) -> TemplateMessage:
    """สร้าง Button Template Message"""
    template = ButtonsTemplate(
        text=text,
        actions=actions[:4]  # สูงสุด 4 ปุ่ม
    )

    if title:
        template.title = title
    if thumbnail_url:
        template.thumbnail_image_url = thumbnail_url
        template.image_aspect_ratio = image_aspect_ratio
        template.image_size = image_size
    if default_action:
        template.default_action = default_action

    return TemplateMessage(
        alt_text=title or text,
        template=template
    )


def confirm_template(
    text: str,
    confirm_action,
    cancel_action
) -> TemplateMessage:
    """สร้าง Confirm Template Message"""
    return TemplateMessage(
        alt_text=text[:100],
        template=ConfirmTemplate(
            text=text,
            actions=[confirm_action, cancel_action]
        )
    )


def carousel_template(
    columns: List[Dict],
    alt_text: str = 'Carousel',
    image_aspect_ratio: str = 'rectangle',
    image_size: str = 'cover'
) -> TemplateMessage:
    """สร้าง Carousel Template Message"""
    carousel_columns = []

    for col in columns[:10]:  # สูงสุด 10 columns
        column = CarouselColumn(
            text=col['text'],
            actions=col['actions'][:3]  # สูงสุด 3 ปุ่ม
        )

        if col.get('thumbnail_image_url'):
            column.thumbnail_image_url = col['thumbnail_image_url']
        if col.get('title'):
            column.title = col['title']
        if col.get('default_action'):
            column.default_action = col['default_action']

        carousel_columns.append(column)

    template = CarouselTemplate(columns=carousel_columns)
    template.image_aspect_ratio = image_aspect_ratio
    template.image_size = image_size

    return TemplateMessage(
        alt_text=alt_text,
        template=template
    )


def image_carousel_template(
    columns: List[Dict],
    alt_text: str = 'Image Carousel'
) -> TemplateMessage:
    """สร้าง Image Carousel Template Message"""
    carousel_columns = [
        ImageCarouselColumn(
            image_url=col['image_url'],
            action=col['action']
        )
        for col in columns[:10]
    ]

    return TemplateMessage(
        alt_text=alt_text,
        template=ImageCarouselTemplate(columns=carousel_columns)
    )


# Action builders
def uri_action(label: str, uri: str) -> URIAction:
    return URIAction(type='uri', label=label, uri=uri)


def message_action(label: str, text: str) -> MessageAction:
    return MessageAction(type='message', label=label, text=text)


def postback_action(
    label: str,
    data: str,
    display_text: Optional[str] = None
) -> PostbackAction:
    action = PostbackAction(type='postback', label=label, data=data)
    if display_text:
        action.display_text = display_text
    return action


def datetime_picker_action(
    label: str,
    data: str,
    mode: str,
    initial: Optional[str] = None,
    min_val: Optional[str] = None,
    max_val: Optional[str] = None
) -> DatetimePickerAction:
    action = DatetimePickerAction(type='datetimepicker', label=label, data=data, mode=mode)
    if initial:
        action.initial = initial
    if min_val:
        action.min = min_val
    if max_val:
        action.max = max_val
    return action


def camera_action(label: str) -> CameraAction:
    return CameraAction(type='camera', label=label)


def camera_roll_action(label: str) -> CameraRollAction:
    return CameraRollAction(type='cameraRoll', label=label)


def location_action(label: str) -> LocationAction:
    return LocationAction(type='location', label=label)
```

### handlers/shop_handler.py

```python
import logging
from datetime import datetime, timedelta
from templates.template_builder import (
    button_template, confirm_template, carousel_template,
    image_carousel_template, uri_action, message_action,
    postback_action, datetime_picker_action
)
from services.line_service import reply_message

logger = logging.getLogger(__name__)

PRODUCTS = [
    {
        'id': 'P001',
        'name': 'Smart Watch Pro X1',
        'price': 4990,
        'description': 'กันน้ำ 50m | แบต 14 วัน | GPS',
        'image_url': 'https://picsum.photos/id/100/800/500',
        'url': 'https://shop.example.com/products/P001'
    },
    {
        'id': 'P002',
        'name': 'TWS Earbuds Ultra',
        'price': 2490,
        'description': 'ANC | 30hr battery | Hi-Fi',
        'image_url': 'https://picsum.photos/id/101/800/500',
        'url': 'https://shop.example.com/products/P002'
    },
    {
        'id': 'P003',
        'name': 'Power Bank 20000mAh',
        'price': 1290,
        'description': 'PD 65W | 3 ช่องชาร์จ',
        'image_url': 'https://picsum.photos/id/102/800/500',
        'url': 'https://shop.example.com/products/P003'
    }
]


def send_main_menu(reply_token: str):
    """ส่ง Main Menu"""
    menu = button_template(
        title='🏪 ร้านของเรา',
        text='ยินดีต้อนรับ! เลือกสิ่งที่ต้องการ',
        thumbnail_url='https://picsum.photos/id/1/800/500',
        actions=[
            message_action('🛍️ ดูสินค้าทั้งหมด', 'ดูสินค้า'),
            message_action('📦 ตรวจสอบคำสั่งซื้อ', 'คำสั่งซื้อของฉัน'),
            message_action('🎁 โปรโมชั่น', 'ดูโปรโมชั่น'),
            uri_action('🌐 เยี่ยมชมเว็บ', 'https://shop.example.com')
        ]
    )
    reply_message(reply_token, [menu])


def send_product_carousel(reply_token: str):
    """ส่ง Product Carousel"""
    columns = [
        {
            'thumbnail_image_url': p['image_url'],
            'title': p['name'],
            'text': f"{p['description']}\n💰 {p['price']:,} บาท",
            'default_action': uri_action('ดูรายละเอียด', p['url']),
            'actions': [
                postback_action('🛒 ซื้อเลย', f"action=buy&id={p['id']}&price={p['price']}",
                               f"เพิ่ม {p['name']} ในตะกร้า"),
                postback_action('❤️ Wishlist', f"action=wishlist&id={p['id']}"),
                uri_action('📋 รายละเอียด', p['url'])
            ]
        }
        for p in PRODUCTS
    ]

    carousel = carousel_template(
        columns=columns,
        alt_text='สินค้าแนะนำ',
        image_aspect_ratio='rectangle',
        image_size='cover'
    )

    reply_message(reply_token, [carousel])


def send_purchase_confirm(reply_token: str, product_id: str, quantity: int = 1):
    """ส่ง Confirm สำหรับการซื้อ"""
    product = next((p for p in PRODUCTS if p['id'] == product_id), None)

    if not product:
        reply_message(reply_token, 'ไม่พบสินค้านี้')
        return

    total = product['price'] * quantity

    confirm = confirm_template(
        text=f"ยืนยันการสั่งซื้อ?\n\n📦 {product['name']} × {quantity}\n💰 ยอดรวม: {total:,} บาท",
        confirm_action=postback_action('✅ ยืนยัน',
                                       f"action=confirm_buy&id={product_id}&qty={quantity}",
                                       'ยืนยันคำสั่งซื้อแล้ว'),
        cancel_action=postback_action('❌ ยกเลิก',
                                      f"action=cancel_buy&id={product_id}",
                                      'ยกเลิกแล้ว')
    )

    reply_message(reply_token, [confirm])


def send_appointment_booking(reply_token: str):
    """ส่ง Appointment Booking"""
    now = datetime.now()
    min_date = now.strftime('%Y-%m-%dT%H:%M')
    max_date = (now + timedelta(days=90)).strftime('%Y-%m-%dT%H:%M')

    booking = button_template(
        title='📅 จองนัดหมาย',
        text='เลือกวันและเวลาที่สะดวก',
        thumbnail_url='https://picsum.photos/id/50/800/500',
        actions=[
            datetime_picker_action(
                label='📅 เลือกวันและเวลา',
                data='action=book&type=new',
                mode='datetime',
                initial=min_date,
                min_val=min_date,
                max_val=max_date
            ),
            postback_action('📋 ดูนัดหมายของฉัน', 'action=view_appointments'),
            postback_action('❌ ยกเลิกนัดหมาย', 'action=cancel_appointment')
        ]
    )

    reply_message(reply_token, [booking])


def handle_postback(event):
    """จัดการ Postback Event"""
    from urllib.parse import parse_qs
    
    reply_token = event.reply_token
    data = event.postback.data
    
    params = dict(parse_qs(data))
    action = params.get('action', [''])[0]
    
    if action == 'buy':
        product_id = params.get('id', [''])[0]
        qty = int(params.get('qty', ['1'])[0])
        send_purchase_confirm(reply_token, product_id, qty)
    
    elif action == 'confirm_buy':
        product_id = params.get('id', [''])[0]
        qty = int(params.get('qty', ['1'])[0])
        product = next((p for p in PRODUCTS if p['id'] == product_id), None)
        if product:
            order_id = str(int(datetime.now().timestamp()))[-8:]
            reply_message(reply_token, [
                f"✅ สั่งซื้อสำเร็จ!\n\n🔖 หมายเลข: #{order_id}\n📦 {product['name']} × {qty}\n💰 {product['price'] * qty:,} บาท"
            ])
    
    elif action == 'wishlist':
        reply_message(reply_token, '❤️ เพิ่มใน Wishlist แล้ว!')
    
    elif action == 'book':
        datetime_selected = event.postback.params.get('datetime', '') if event.postback.params else ''
        if datetime_selected:
            reply_message(reply_token, f'✅ บันทึกนัดหมาย {datetime_selected} แล้ว!')
        else:
            reply_message(reply_token, '❌ กรุณาเลือกวันที่')
    
    else:
        reply_message(reply_token, f'รับทราบ: {data}')
```

---

## 9. Best Practices

### เมื่อไหร่ควรใช้ Template แต่ละประเภท

| Template | เหมาะกับ |
|---------|---------|
| Buttons | เมนูเลือก, Product card, Quick actions |
| Confirm | ยืนยันคำสั่ง, Yes/No questions |
| Carousel | Product listing, Service menu, News |
| Image Carousel | Photo gallery, Campaign banners |

### Tips สำหรับ UX ที่ดี

1. **ปุ่มมีความหมายชัดเจน** - ใช้คำกริยา + Object (เช่น "ซื้อสินค้า" ไม่ใช่ "กด")
2. **จำกัดตัวเลือก** - ไม่ควรมีปุ่มเกิน 4 ปุ่ม
3. **ใช้ emoji** - ช่วยให้ปุ่มดูน่าคลิก
4. **Alt text สำคัญ** - ควรอธิบาย template ได้ดี
5. **Test บนมือถือ** - Templates ออกแบบสำหรับมือถือ

### Template vs Flex Message

| ปัจจัย | Template | Flex Message |
|-------|---------|-------------|
| ความง่าย | ง่ายกว่า | ซับซ้อนกว่า |
| ความยืดหยุ่น | จำกัด | ไม่จำกัด |
| รองรับ Web | บางส่วน | ดีกว่า |
| Learning curve | ต่ำ | สูง |
| แนะนำใช้ | Basic UI | Advanced UI |

---

## 10. แบบฝึกหัดท้ายบท

### แบบฝึกหัดที่ 1: Restaurant Menu Bot
สร้าง Bot สำหรับร้านอาหารที่มี:
- Button Template: เมนูหลัก (อาหาร/เครื่องดื่ม/ของหวาน/เรียกพนักงาน)
- Carousel Template: แสดงรายการอาหาร 5 อย่างพร้อมราคา
- Confirm Template: ยืนยันคำสั่งอาหาร

### แบบฝึกหัดที่ 2: Event Registration Bot
สร้าง Bot ลงทะเบียนงานอีเวนต์ที่:
- แสดง Carousel Template ของงานอีเวนต์ 3 งาน
- มีปุ่ม Datetime Picker เพื่อเลือกรอบเวลา
- ยืนยันการลงทะเบียนด้วย Confirm Template

### แบบฝึกหัดที่ 3: Survey Bot
สร้าง Bot สำรวจความพึงพอใจ:
- ถามคำถาม 3 ข้อ
- แต่ละข้อใช้ Confirm Template หรือ Button Template
- สรุปผลการสำรวจ

---

**สรุปบทเรียน:**

Template Messages ใน LINE Messaging API มี 4 ประเภทหลัก:

1. **Button Template** - แสดงรูป + ปุ่มสูงสุด 4 ปุ่ม
2. **Confirm Template** - ถามใช่/ไม่ใช่ พร้อม 2 ปุ่ม
3. **Carousel Template** - แสดงหลายรายการแบบ scroll
4. **Image Carousel** - Gallery รูปภาพแบบ scroll

Action Types ที่รองรับ: URI, Message, Postback, Datetime Picker, Camera, Camera Roll, Location
