# Part 32: Flex Bubble Container โดยละเอียด

## สารบัญ

1. [Bubble Container Structure](#bubble-container-structure)
2. [Size Options](#size-options)
3. [Direction (ltr/rtl)](#direction)
4. [Header Component](#header-component)
5. [Hero Component](#hero-component)
6. [Body Component](#body-component)
7. [Footer Component](#footer-component)
8. [Styles Object](#styles-object)
9. [Complete Example: Product Card Bubble](#product-card-bubble)
10. [Complete Example: Receipt Bubble](#receipt-bubble)
11. [Complete Example: Event Invitation Bubble](#event-invitation-bubble)
12. [Best Practices](#best-practices)
13. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Bubble Container Structure

Bubble เป็น container พื้นฐานของ Flex Message ที่แสดงเนื้อหาในรูปแบบการ์ดเดียว

### โครงสร้างสมบูรณ์

```json
{
  "type": "bubble",
  "size": "mega",
  "direction": "ltr",
  "header": {
    "type": "box",
    "layout": "vertical",
    "contents": []
  },
  "hero": {
    "type": "image",
    "url": "https://example.com/hero.jpg",
    "size": "full",
    "aspectRatio": "20:13",
    "aspectMode": "cover"
  },
  "body": {
    "type": "box",
    "layout": "vertical",
    "contents": []
  },
  "footer": {
    "type": "box",
    "layout": "vertical",
    "contents": []
  },
  "styles": {
    "header": {
      "backgroundColor": "#ffffff",
      "separator": false,
      "separatorColor": "#cccccc"
    },
    "hero": {
      "separator": false,
      "separatorColor": "#cccccc"
    },
    "body": {
      "backgroundColor": "#ffffff",
      "separator": false,
      "separatorColor": "#cccccc"
    },
    "footer": {
      "backgroundColor": "#ffffff",
      "separator": true,
      "separatorColor": "#eeeeee"
    }
  }
}
```

### Properties ทั้งหมดของ Bubble

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| type | string | ✅ | - | ต้องเป็น "bubble" |
| size | string | ❌ | "mega" | nano, micro, kilo, mega, giga |
| direction | string | ❌ | "ltr" | "ltr" หรือ "rtl" |
| header | object | ❌ | - | Box component สำหรับส่วนหัว |
| hero | object | ❌ | - | Image, Box หรือ Video |
| body | object | ❌ | - | Box component สำหรับเนื้อหาหลัก |
| footer | object | ❌ | - | Box component สำหรับส่วนล่าง |
| styles | object | ❌ | - | กำหนด style ของแต่ละ section |
| action | object | ❌ | - | Action เมื่อแตะที่ bubble ทั้งหมด |

### Bubble Sections Layout

```
┌──────────────────────────────┐
│  header                      │  ← optional
│  (Box component)             │
├──────────────────────────────┤
│  hero                        │  ← optional
│  (Image/Box/Video)           │
├──────────────────────────────┤
│  body                        │  ← optional
│  (Box component)             │
│                              │
├──────────────────────────────┤
│  footer                      │  ← optional
│  (Box component)             │
└──────────────────────────────┘
```

> **หมายเหตุ:** ทุก section เป็น optional แต่ bubble ที่ดีควรมี body หรือ hero อย่างน้อยหนึ่งอย่าง

---

## 2. Size Options

Size ของ Bubble กำหนดความกว้างของการ์ดบนหน้าจอ

### ขนาดที่รองรับ

| Size | Width (approx) | Use Case |
|------|----------------|----------|
| nano | ~120px | ไอคอน, badge เล็กๆ |
| micro | ~160px | ข้อมูลสั้นๆ |
| kilo | ~220px (default เดิม) | ข้อมูลทั่วไป |
| mega | ~300px (recommended) | สินค้า, บทความ, ข้อมูลหลัก |
| giga | ~400px | เนื้อหาที่ต้องการพื้นที่มาก |

### ตัวอย่างการกำหนด Size

```json
// Nano - ใช้สำหรับแท็กหรือ badge
{
  "type": "bubble",
  "size": "nano",
  "body": {
    "type": "box",
    "layout": "vertical",
    "contents": [
      {
        "type": "text",
        "text": "NEW",
        "weight": "bold",
        "size": "sm",
        "color": "#FFFFFF",
        "align": "center"
      }
    ],
    "backgroundColor": "#FF5722",
    "paddingAll": "8px"
  }
}
```

```json
// Mega - ใช้สำหรับ product card ทั่วไป
{
  "type": "bubble",
  "size": "mega",
  "hero": {
    "type": "image",
    "url": "https://example.com/product.jpg",
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
        "text": "ชื่อสินค้า",
        "weight": "bold",
        "size": "xl"
      }
    ]
  }
}
```

```json
// Giga - ใช้สำหรับเนื้อหาที่ต้องการพื้นที่มาก
{
  "type": "bubble",
  "size": "giga",
  "body": {
    "type": "box",
    "layout": "vertical",
    "contents": [
      {
        "type": "text",
        "text": "บทความยาว",
        "weight": "bold",
        "size": "xxl"
      },
      {
        "type": "text",
        "text": "เนื้อหาบทความที่ต้องการพื้นที่แสดงผลมาก...",
        "size": "md",
        "wrap": true,
        "margin": "md"
      }
    ]
  }
}
```

### Visual Comparison ของแต่ละ Size

```
nano:
┌────┐
│    │
└────┘

micro:
┌──────┐
│      │
└──────┘

kilo:
┌──────────┐
│          │
└──────────┘

mega (recommended):
┌──────────────┐
│              │
└──────────────┘

giga:
┌──────────────────┐
│                  │
└──────────────────┘
```

---

## 3. Direction

Direction กำหนดทิศทางการไหลของ content ใน bubble

### ltr (Left to Right) - Default

ใช้กับภาษาที่อ่านจากซ้ายไปขวา เช่น ไทย, อังกฤษ

```json
{
  "type": "bubble",
  "direction": "ltr",
  "body": {
    "type": "box",
    "layout": "horizontal",
    "contents": [
      {
        "type": "text",
        "text": "ชื่อ:"
      },
      {
        "type": "text",
        "text": "สมชาย"
      }
    ]
  }
}
```

ผลลัพธ์: `ชื่อ:  สมชาย`

### rtl (Right to Left)

ใช้กับภาษาที่อ่านจากขวาไปซ้าย เช่น ภาษาอาหรับ, ฮีบรู

```json
{
  "type": "bubble",
  "direction": "rtl",
  "body": {
    "type": "box",
    "layout": "horizontal",
    "contents": [
      {
        "type": "text",
        "text": ":الاسم"
      },
      {
        "type": "text",
        "text": "محمد"
      }
    ]
  }
}
```

ผลลัพธ์: `محمد  :الاسم`

---

## 4. Header Component

Header เป็นส่วนบนสุดของ bubble มักใช้สำหรับแสดง title หรือ category

### Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| type | string | ✅ | ต้องเป็น "box" |
| layout | string | ✅ | "vertical", "horizontal", "baseline" |
| contents | array | ✅ | array ของ components |

### ตัวอย่าง Header พื้นฐาน

```json
"header": {
  "type": "box",
  "layout": "vertical",
  "paddingAll": "20px",
  "backgroundColor": "#0D8348",
  "contents": [
    {
      "type": "text",
      "text": "🛒 ยืนยันคำสั่งซื้อ",
      "weight": "bold",
      "color": "#FFFFFF",
      "size": "lg"
    }
  ]
}
```

### ตัวอย่าง Header ที่ซับซ้อน

```json
"header": {
  "type": "box",
  "layout": "horizontal",
  "paddingAll": "16px",
  "backgroundColor": "#1a237e",
  "contents": [
    {
      "type": "image",
      "url": "https://example.com/logo.png",
      "size": "xxs",
      "flex": 0,
      "margin": "none"
    },
    {
      "type": "box",
      "layout": "vertical",
      "flex": 1,
      "margin": "sm",
      "contents": [
        {
          "type": "text",
          "text": "ร้านค้าของเรา",
          "weight": "bold",
          "color": "#FFFFFF",
          "size": "md"
        },
        {
          "type": "text",
          "text": "เปิดบริการ 09:00 - 21:00 น.",
          "color": "#ffffffcc",
          "size": "xs"
        }
      ]
    },
    {
      "type": "button",
      "flex": 0,
      "style": "link",
      "action": {
        "type": "uri",
        "label": "ดูทั้งหมด",
        "uri": "https://example.com"
      },
      "color": "#FFFFFF"
    }
  ]
}
```

### ตัวอย่าง Header แบบ Gradient (ใช้ Box ซ้อนกัน)

```json
"header": {
  "type": "box",
  "layout": "vertical",
  "paddingAll": "20px",
  "contents": [
    {
      "type": "text",
      "text": "SALE 50% OFF",
      "weight": "bold",
      "size": "xxl",
      "color": "#FFFFFF",
      "align": "center"
    },
    {
      "type": "text",
      "text": "วันนี้ - 31 ธันวาคม 2025",
      "size": "sm",
      "color": "#ffffffcc",
      "align": "center",
      "margin": "sm"
    }
  ],
  "backgroundColor": "#E91E63"
}
```

---

## 5. Hero Component

Hero เป็น section สำหรับแสดงรูปภาพหลักหรือ video โดยเฉพาะ

### Image Hero

```json
"hero": {
  "type": "image",
  "url": "https://example.com/hero-image.jpg",
  "size": "full",
  "aspectRatio": "20:13",
  "aspectMode": "cover",
  "action": {
    "type": "uri",
    "uri": "https://example.com"
  }
}
```

### Image Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| type | string | ✅ | "image" |
| url | string | ✅ | URL ของรูปภาพ (HTTPS เท่านั้น) |
| size | string | ❌ | xxs, xs, sm, md, lg, xl, xxl, 3xl, 4xl, 5xl, full |
| aspectRatio | string | ❌ | "width:height" เช่น "20:13", "1:1" |
| aspectMode | string | ❌ | "cover" (crop) หรือ "fit" (letterbox) |
| action | object | ❌ | Action เมื่อแตะรูปภาพ |
| animated | boolean | ❌ | true สำหรับ animated GIF |

### aspectRatio Options ที่ใช้บ่อย

```
1:1    → สี่เหลี่ยมจัตุรัส
20:13  → landscape (แนะนำสำหรับ hero)
3:4    → portrait
9:16   → full-screen mobile
16:9   → widescreen
```

### ตัวอย่างการใช้ aspectMode

```json
// cover - crop รูปให้พอดีกับพื้นที่ (ไม่เห็นขอบขาว)
{
  "type": "image",
  "url": "https://example.com/image.jpg",
  "size": "full",
  "aspectRatio": "20:13",
  "aspectMode": "cover"
}

// fit - ย่อรูปให้เห็นทั้งรูป (อาจมีขอบขาว)
{
  "type": "image",
  "url": "https://example.com/image.jpg",
  "size": "full",
  "aspectRatio": "20:13",
  "aspectMode": "fit"
}
```

```
aspectMode: "cover"        aspectMode: "fit"
┌───────────────────┐      ┌───────────────────┐
│  [═══════════════]│      │                   │
│  [  Image crops  ]│      │  [Image fit in]   │
│  [═══════════════]│      │                   │
└───────────────────┘      └───────────────────┘
```

### Box Hero (กำหนด hero ด้วย Box)

เมื่อต้องการ hero ที่ซับซ้อนกว่าแค่รูปภาพเดียว

```json
"hero": {
  "type": "box",
  "layout": "vertical",
  "contents": [
    {
      "type": "image",
      "url": "https://example.com/banner.jpg",
      "size": "full",
      "aspectRatio": "20:8",
      "aspectMode": "cover"
    },
    {
      "type": "box",
      "layout": "horizontal",
      "position": "absolute",
      "offsetBottom": "0px",
      "offsetStart": "0px",
      "offsetEnd": "0px",
      "paddingAll": "12px",
      "backgroundColor": "#00000080",
      "contents": [
        {
          "type": "text",
          "text": "Flash Sale! 🔥",
          "color": "#FFFFFF",
          "weight": "bold",
          "size": "lg"
        }
      ]
    }
  ]
}
```

### Video Hero

```json
"hero": {
  "type": "video",
  "url": "https://example.com/video.mp4",
  "previewUrl": "https://example.com/preview.jpg",
  "altContent": {
    "type": "image",
    "url": "https://example.com/preview.jpg",
    "size": "full",
    "aspectRatio": "20:13",
    "aspectMode": "cover"
  },
  "action": {
    "type": "uri",
    "label": "ดูวิดีโอ",
    "uri": "https://example.com/video"
  }
}
```

---

## 6. Body Component

Body เป็นส่วนหลักของ bubble สำหรับแสดงเนื้อหาหลัก ต้องเป็น Box component เสมอ

### Body ต้องเป็น Box

```json
"body": {
  "type": "box",        // ต้องเป็น "box" เสมอ
  "layout": "vertical", // หรือ "horizontal" หรือ "baseline"
  "paddingAll": "20px",
  "spacing": "md",
  "contents": [
    // components ต่างๆ
  ]
}
```

### Box Layout Options

**vertical** - จัด content แนวตั้ง (บนลงล่าง)
```json
{
  "type": "box",
  "layout": "vertical",
  "contents": [
    { "type": "text", "text": "บรรทัดที่ 1" },
    { "type": "text", "text": "บรรทัดที่ 2" },
    { "type": "text", "text": "บรรทัดที่ 3" }
  ]
}
```
ผลลัพธ์:
```
บรรทัดที่ 1
บรรทัดที่ 2
บรรทัดที่ 3
```

**horizontal** - จัด content แนวนอน (ซ้ายไปขวา)
```json
{
  "type": "box",
  "layout": "horizontal",
  "contents": [
    { "type": "text", "text": "ซ้าย" },
    { "type": "text", "text": "กลาง" },
    { "type": "text", "text": "ขวา" }
  ]
}
```
ผลลัพธ์: `ซ้าย   กลาง   ขวา`

**baseline** - จัด content แนวนอน โดย align ที่ baseline (ดีสำหรับ text + icon)
```json
{
  "type": "box",
  "layout": "baseline",
  "contents": [
    { "type": "icon", "url": "https://example.com/star.png", "size": "sm" },
    { "type": "text", "text": "4.9 (100 รีวิว)", "size": "sm" }
  ]
}
```

### Body Box Properties สำคัญ

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| layout | string | - | "vertical", "horizontal", "baseline" |
| spacing | string | "none" | ระยะห่างระหว่าง items: none, xs, sm, md, lg, xl, xxl |
| paddingAll | string | - | padding รอบ box ทั้งหมด |
| paddingTop | string | - | padding ด้านบน |
| paddingBottom | string | - | padding ด้านล่าง |
| paddingStart | string | - | padding ด้านซ้าย (ltr) |
| paddingEnd | string | - | padding ด้านขวา (ltr) |
| backgroundColor | string | - | สีพื้นหลัง (#RRGGBB หรือ #RRGGBBAA) |
| cornerRadius | string | - | มุมโค้ง เช่น "4px", "50%" |
| borderWidth | string | - | ความหนาของ border |
| borderColor | string | - | สีของ border |

### ตัวอย่าง Body ที่สมบูรณ์

```json
"body": {
  "type": "box",
  "layout": "vertical",
  "paddingAll": "20px",
  "spacing": "md",
  "contents": [
    // ชื่อสินค้า
    {
      "type": "text",
      "text": "iPhone 15 Pro Max",
      "weight": "bold",
      "size": "xl",
      "color": "#1a1a1a",
      "wrap": true,
      "maxLines": 2
    },
    // Brand
    {
      "type": "text",
      "text": "Apple",
      "size": "sm",
      "color": "#888888"
    },
    // Rating Box
    {
      "type": "box",
      "layout": "baseline",
      "spacing": "sm",
      "contents": [
        {
          "type": "icon",
          "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
          "size": "sm"
        },
        {
          "type": "icon",
          "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
          "size": "sm"
        },
        {
          "type": "icon",
          "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
          "size": "sm"
        },
        {
          "type": "icon",
          "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
          "size": "sm"
        },
        {
          "type": "icon",
          "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
          "size": "sm"
        },
        {
          "type": "text",
          "text": "5.0 (1,234)",
          "size": "sm",
          "color": "#888888",
          "margin": "sm"
        }
      ]
    },
    // Separator
    {
      "type": "separator",
      "margin": "none",
      "color": "#eeeeee"
    },
    // Details
    {
      "type": "box",
      "layout": "vertical",
      "spacing": "sm",
      "contents": [
        {
          "type": "box",
          "layout": "horizontal",
          "contents": [
            {
              "type": "text",
              "text": "สี",
              "size": "sm",
              "color": "#888888",
              "flex": 1
            },
            {
              "type": "text",
              "text": "Natural Titanium",
              "size": "sm",
              "color": "#333333",
              "flex": 2
            }
          ]
        },
        {
          "type": "box",
          "layout": "horizontal",
          "contents": [
            {
              "type": "text",
              "text": "ความจุ",
              "size": "sm",
              "color": "#888888",
              "flex": 1
            },
            {
              "type": "text",
              "text": "256 GB",
              "size": "sm",
              "color": "#333333",
              "flex": 2
            }
          ]
        }
      ]
    },
    // Price
    {
      "type": "box",
      "layout": "horizontal",
      "margin": "md",
      "contents": [
        {
          "type": "text",
          "text": "ราคา",
          "size": "sm",
          "color": "#888888",
          "flex": 1
        },
        {
          "type": "text",
          "text": "฿49,900",
          "size": "xl",
          "weight": "bold",
          "color": "#FF5722",
          "flex": 0
        }
      ]
    }
  ]
}
```

---

## 7. Footer Component

Footer เป็นส่วนล่างของ bubble มักใช้สำหรับปุ่ม action ต้องเป็น Box component เสมอ

### Footer พื้นฐาน

```json
"footer": {
  "type": "box",
  "layout": "vertical",
  "spacing": "sm",
  "paddingAll": "20px",
  "contents": [
    {
      "type": "button",
      "style": "primary",
      "action": {
        "type": "uri",
        "label": "สั่งซื้อเลย",
        "uri": "https://example.com/order"
      }
    }
  ]
}
```

### Footer แบบ 2 ปุ่ม

```json
"footer": {
  "type": "box",
  "layout": "horizontal",
  "spacing": "sm",
  "paddingAll": "16px",
  "contents": [
    {
      "type": "button",
      "style": "secondary",
      "height": "sm",
      "flex": 1,
      "action": {
        "type": "uri",
        "label": "ดูรายละเอียด",
        "uri": "https://example.com/detail"
      }
    },
    {
      "type": "button",
      "style": "primary",
      "height": "sm",
      "flex": 1,
      "color": "#FF5722",
      "action": {
        "type": "postback",
        "label": "เพิ่มลงตะกร้า",
        "data": "action=addCart&id=123",
        "displayText": "เพิ่มสินค้าลงตะกร้า"
      }
    }
  ]
}
```

### Footer แบบ 3 ปุ่ม

```json
"footer": {
  "type": "box",
  "layout": "vertical",
  "spacing": "sm",
  "paddingAll": "16px",
  "contents": [
    {
      "type": "button",
      "style": "primary",
      "action": {
        "type": "postback",
        "label": "🛒 สั่งซื้อ",
        "data": "action=buy&id=123"
      }
    },
    {
      "type": "box",
      "layout": "horizontal",
      "spacing": "sm",
      "contents": [
        {
          "type": "button",
          "style": "secondary",
          "height": "sm",
          "flex": 1,
          "action": {
            "type": "postback",
            "label": "❤️ Wishlist",
            "data": "action=wishlist&id=123"
          }
        },
        {
          "type": "button",
          "style": "secondary",
          "height": "sm",
          "flex": 1,
          "action": {
            "type": "uri",
            "label": "📤 แชร์",
            "uri": "https://example.com/share/123"
          }
        }
      ]
    }
  ]
}
```

---

## 8. Styles Object

Styles object ช่วยให้กำหนด backgroundColor และ separator ของแต่ละ section ได้

### โครงสร้าง Styles

```json
"styles": {
  "header": {
    "backgroundColor": "#ffffff",
    "separator": false,
    "separatorColor": "#cccccc"
  },
  "hero": {
    "separator": false,
    "separatorColor": "#cccccc"
  },
  "body": {
    "backgroundColor": "#ffffff",
    "separator": false,
    "separatorColor": "#cccccc"
  },
  "footer": {
    "backgroundColor": "#f5f5f5",
    "separator": true,
    "separatorColor": "#eeeeee"
  }
}
```

### Style Properties

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| backgroundColor | string | "#ffffff" | สีพื้นหลังของ section |
| separator | boolean | false | แสดงเส้นคั่นระหว่าง section |
| separatorColor | string | "#cccccc" | สีของเส้นคั่น |

> **หมายเหตุ:** separator ใน styles จะเพิ่มเส้นคั่นระหว่าง section ที่กำหนด separator: true กับ section ก่อนหน้า

### ตัวอย่างการใช้ Styles

```json
// Styles สำหรับ Dark Mode
"styles": {
  "header": {
    "backgroundColor": "#212121"
  },
  "body": {
    "backgroundColor": "#303030"
  },
  "footer": {
    "backgroundColor": "#212121",
    "separator": true,
    "separatorColor": "#424242"
  }
}

// Styles สำหรับ Brand Theme
"styles": {
  "header": {
    "backgroundColor": "#1565C0"
  },
  "body": {
    "backgroundColor": "#FFFFFF"
  },
  "footer": {
    "backgroundColor": "#E3F2FD",
    "separator": true,
    "separatorColor": "#BBDEFB"
  }
}
```

---

## 9. Complete Example: Product Card Bubble

สร้าง product card ที่สมบูรณ์พร้อม header, hero, body, footer, และ styles

```json
{
  "type": "flex",
  "altText": "สินค้า: Nike Air Max 90 - ราคา ฿3,500",
  "contents": {
    "type": "bubble",
    "size": "mega",
    "header": {
      "type": "box",
      "layout": "horizontal",
      "paddingAll": "12px",
      "paddingStart": "20px",
      "paddingEnd": "20px",
      "contents": [
        {
          "type": "text",
          "text": "🔥 สินค้าแนะนำ",
          "weight": "bold",
          "size": "sm",
          "color": "#FF5722",
          "flex": 1
        },
        {
          "type": "text",
          "text": "ลด 22%",
          "size": "xs",
          "color": "#FFFFFF",
          "backgroundColor": "#F44336",
          "padding": "4px 8px",
          "flex": 0
        }
      ]
    },
    "hero": {
      "type": "image",
      "url": "https://via.placeholder.com/400x260/f5f5f5/333333?text=Nike+Air+Max+90",
      "size": "full",
      "aspectRatio": "20:13",
      "aspectMode": "cover",
      "action": {
        "type": "uri",
        "uri": "https://example.com/products/nike-am90"
      }
    },
    "body": {
      "type": "box",
      "layout": "vertical",
      "paddingAll": "20px",
      "spacing": "sm",
      "contents": [
        {
          "type": "text",
          "text": "Nike Air Max 90 Essential",
          "weight": "bold",
          "size": "xl",
          "color": "#1a1a1a",
          "maxLines": 2,
          "wrap": true
        },
        {
          "type": "text",
          "text": "Nike",
          "size": "xs",
          "color": "#999999"
        },
        {
          "type": "box",
          "layout": "baseline",
          "spacing": "xs",
          "margin": "sm",
          "contents": [
            {
              "type": "icon",
              "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
              "size": "xs"
            },
            {
              "type": "icon",
              "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
              "size": "xs"
            },
            {
              "type": "icon",
              "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
              "size": "xs"
            },
            {
              "type": "icon",
              "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
              "size": "xs"
            },
            {
              "type": "icon",
              "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
              "size": "xs"
            },
            {
              "type": "text",
              "text": "4.9",
              "size": "xs",
              "color": "#FF9800",
              "flex": 0,
              "weight": "bold"
            },
            {
              "type": "text",
              "text": "(2,341 รีวิว)",
              "size": "xs",
              "color": "#aaaaaa",
              "margin": "xs"
            }
          ]
        },
        {
          "type": "text",
          "text": "รองเท้าวิ่งรุ่นยอดนิยม ดีไซน์คลาสสิก ระบายอากาศดี สวมใส่สบาย เหมาะกับทุกโอกาส",
          "size": "sm",
          "color": "#666666",
          "wrap": true,
          "maxLines": 3,
          "margin": "sm"
        },
        {
          "type": "separator",
          "margin": "lg",
          "color": "#eeeeee"
        },
        {
          "type": "box",
          "layout": "horizontal",
          "margin": "md",
          "alignItems": "center",
          "contents": [
            {
              "type": "box",
              "layout": "vertical",
              "flex": 1,
              "contents": [
                {
                  "type": "text",
                  "text": "฿3,500",
                  "weight": "bold",
                  "size": "xxl",
                  "color": "#FF5722"
                },
                {
                  "type": "text",
                  "text": "฿4,500",
                  "size": "sm",
                  "color": "#aaaaaa",
                  "decoration": "line-through"
                }
              ]
            },
            {
              "type": "box",
              "layout": "vertical",
              "flex": 0,
              "paddingAll": "6px 12px",
              "backgroundColor": "#FF5722",
              "cornerRadius": "4px",
              "contents": [
                {
                  "type": "text",
                  "text": "ลด 22%",
                  "size": "sm",
                  "color": "#FFFFFF",
                  "weight": "bold"
                }
              ]
            }
          ]
        },
        {
          "type": "box",
          "layout": "horizontal",
          "margin": "sm",
          "spacing": "sm",
          "contents": [
            {
              "type": "text",
              "text": "✓ มีสินค้า",
              "size": "xs",
              "color": "#4CAF50",
              "flex": 0
            },
            {
              "type": "text",
              "text": "🚚 ส่งฟรีเมื่อซื้อครบ ฿1,000",
              "size": "xs",
              "color": "#2196F3",
              "flex": 1,
              "align": "end"
            }
          ]
        }
      ]
    },
    "footer": {
      "type": "box",
      "layout": "horizontal",
      "spacing": "sm",
      "paddingAll": "16px",
      "contents": [
        {
          "type": "button",
          "style": "secondary",
          "height": "sm",
          "flex": 1,
          "action": {
            "type": "postback",
            "label": "❤️ บันทึก",
            "data": "action=wishlist&productId=NIKE-AM90",
            "displayText": "บันทึกสินค้าแล้ว"
          }
        },
        {
          "type": "button",
          "style": "primary",
          "height": "sm",
          "flex": 2,
          "color": "#FF5722",
          "action": {
            "type": "postback",
            "label": "🛒 เพิ่มลงตะกร้า",
            "data": "action=addCart&productId=NIKE-AM90",
            "displayText": "เพิ่มสินค้าลงตะกร้าแล้ว"
          }
        }
      ]
    },
    "styles": {
      "header": {
        "backgroundColor": "#FFF8F6"
      },
      "body": {
        "backgroundColor": "#FFFFFF"
      },
      "footer": {
        "backgroundColor": "#FFFFFF",
        "separator": true,
        "separatorColor": "#eeeeee"
      }
    }
  }
}
```

---

## 10. Complete Example: Receipt Bubble

ตัวอย่าง Receipt ที่แสดงรายการสั่งซื้อ

```json
{
  "type": "flex",
  "altText": "ใบเสร็จ #ORD-20250115-001 - รวม ฿890",
  "contents": {
    "type": "bubble",
    "size": "mega",
    "header": {
      "type": "box",
      "layout": "vertical",
      "paddingAll": "20px",
      "paddingBottom": "16px",
      "contents": [
        {
          "type": "box",
          "layout": "horizontal",
          "contents": [
            {
              "type": "text",
              "text": "🧾 ใบเสร็จรับเงิน",
              "weight": "bold",
              "size": "lg",
              "color": "#1a1a1a",
              "flex": 1
            },
            {
              "type": "box",
              "layout": "vertical",
              "flex": 0,
              "paddingAll": "4px 8px",
              "backgroundColor": "#4CAF50",
              "cornerRadius": "4px",
              "contents": [
                {
                  "type": "text",
                  "text": "ชำระแล้ว",
                  "size": "xs",
                  "color": "#FFFFFF",
                  "weight": "bold"
                }
              ]
            }
          ]
        },
        {
          "type": "text",
          "text": "คำสั่งซื้อ #ORD-20250115-001",
          "size": "xs",
          "color": "#888888",
          "margin": "xs"
        },
        {
          "type": "text",
          "text": "15 มกราคม 2025, 14:30 น.",
          "size": "xs",
          "color": "#888888"
        }
      ]
    },
    "body": {
      "type": "box",
      "layout": "vertical",
      "paddingAll": "20px",
      "spacing": "md",
      "contents": [
        {
          "type": "text",
          "text": "รายการสินค้า",
          "weight": "bold",
          "size": "sm",
          "color": "#555555"
        },
        {
          "type": "separator",
          "color": "#eeeeee"
        },
        {
          "type": "box",
          "layout": "horizontal",
          "contents": [
            {
              "type": "text",
              "text": "กาแฟลาเต้ (L)",
              "size": "sm",
              "color": "#333333",
              "flex": 3
            },
            {
              "type": "text",
              "text": "x1",
              "size": "sm",
              "color": "#888888",
              "flex": 1,
              "align": "center"
            },
            {
              "type": "text",
              "text": "฿95",
              "size": "sm",
              "color": "#333333",
              "flex": 1,
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
              "text": "ครัวซองต์เนย",
              "size": "sm",
              "color": "#333333",
              "flex": 3
            },
            {
              "type": "text",
              "text": "x2",
              "size": "sm",
              "color": "#888888",
              "flex": 1,
              "align": "center"
            },
            {
              "type": "text",
              "text": "฿160",
              "size": "sm",
              "color": "#333333",
              "flex": 1,
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
              "text": "เค้กช็อคโกแลต",
              "size": "sm",
              "color": "#333333",
              "flex": 3
            },
            {
              "type": "text",
              "text": "x1",
              "size": "sm",
              "color": "#888888",
              "flex": 1,
              "align": "center"
            },
            {
              "type": "text",
              "text": "฿180",
              "size": "sm",
              "color": "#333333",
              "flex": 1,
              "align": "end"
            }
          ]
        },
        {
          "type": "separator",
          "color": "#eeeeee"
        },
        {
          "type": "box",
          "layout": "horizontal",
          "contents": [
            {
              "type": "text",
              "text": "รวมสินค้า",
              "size": "sm",
              "color": "#555555",
              "flex": 1
            },
            {
              "type": "text",
              "text": "฿435",
              "size": "sm",
              "color": "#333333",
              "flex": 0
            }
          ]
        },
        {
          "type": "box",
          "layout": "horizontal",
          "contents": [
            {
              "type": "text",
              "text": "ค่าจัดส่ง",
              "size": "sm",
              "color": "#555555",
              "flex": 1
            },
            {
              "type": "text",
              "text": "฿50",
              "size": "sm",
              "color": "#333333",
              "flex": 0
            }
          ]
        },
        {
          "type": "box",
          "layout": "horizontal",
          "contents": [
            {
              "type": "text",
              "text": "ส่วนลดโค้ด SAVE50",
              "size": "sm",
              "color": "#4CAF50",
              "flex": 1
            },
            {
              "type": "text",
              "text": "-฿50",
              "size": "sm",
              "color": "#4CAF50",
              "flex": 0
            }
          ]
        },
        {
          "type": "separator",
          "color": "#eeeeee"
        },
        {
          "type": "box",
          "layout": "horizontal",
          "contents": [
            {
              "type": "text",
              "text": "ยอดชำระรวม",
              "weight": "bold",
              "size": "md",
              "color": "#1a1a1a",
              "flex": 1
            },
            {
              "type": "text",
              "text": "฿435",
              "weight": "bold",
              "size": "xl",
              "color": "#FF5722",
              "flex": 0
            }
          ]
        },
        {
          "type": "separator",
          "color": "#eeeeee"
        },
        {
          "type": "box",
          "layout": "vertical",
          "backgroundColor": "#F5F5F5",
          "paddingAll": "12px",
          "cornerRadius": "8px",
          "contents": [
            {
              "type": "text",
              "text": "ที่อยู่จัดส่ง",
              "size": "xs",
              "color": "#888888",
              "weight": "bold"
            },
            {
              "type": "text",
              "text": "สมชาย ใจดี",
              "size": "sm",
              "color": "#333333",
              "margin": "xs"
            },
            {
              "type": "text",
              "text": "123/45 ถ.สุขุมวิท แขวงคลองเตย กทม 10110",
              "size": "xs",
              "color": "#666666",
              "wrap": true
            }
          ]
        }
      ]
    },
    "footer": {
      "type": "box",
      "layout": "vertical",
      "spacing": "sm",
      "paddingAll": "16px",
      "contents": [
        {
          "type": "button",
          "style": "primary",
          "height": "sm",
          "action": {
            "type": "postback",
            "label": "📍 ติดตามพัสดุ",
            "data": "action=trackOrder&orderId=ORD-20250115-001"
          }
        },
        {
          "type": "button",
          "style": "secondary",
          "height": "sm",
          "action": {
            "type": "uri",
            "label": "📄 ดาวน์โหลดใบเสร็จ",
            "uri": "https://example.com/receipts/ORD-20250115-001"
          }
        }
      ]
    },
    "styles": {
      "header": {
        "backgroundColor": "#FAFAFA",
        "separator": false
      },
      "footer": {
        "backgroundColor": "#FAFAFA",
        "separator": true,
        "separatorColor": "#eeeeee"
      }
    }
  }
}
```

---

## 11. Complete Example: Event Invitation Bubble

ตัวอย่างการ์ดเชิญร่วมงาน Event

```json
{
  "type": "flex",
  "altText": "เชิญร่วมงาน LINE Developer Summit 2025 - 20 มกราคม 2025",
  "contents": {
    "type": "bubble",
    "size": "mega",
    "hero": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        {
          "type": "image",
          "url": "https://via.placeholder.com/400x200/1565C0/FFFFFF?text=LINE+Developer+Summit+2025",
          "size": "full",
          "aspectRatio": "20:10",
          "aspectMode": "cover"
        },
        {
          "type": "box",
          "layout": "horizontal",
          "position": "absolute",
          "offsetBottom": "0px",
          "offsetStart": "0px",
          "offsetEnd": "0px",
          "paddingAll": "12px",
          "backgroundColor": "#00000060",
          "contents": [
            {
              "type": "text",
              "text": "LINE Developer Summit 2025",
              "color": "#FFFFFF",
              "weight": "bold",
              "size": "md",
              "wrap": true,
              "flex": 1
            }
          ]
        },
        {
          "type": "box",
          "layout": "vertical",
          "position": "absolute",
          "offsetTop": "12px",
          "offsetEnd": "12px",
          "paddingAll": "6px 12px",
          "backgroundColor": "#FF5722",
          "cornerRadius": "20px",
          "contents": [
            {
              "type": "text",
              "text": "🔥 HOT",
              "size": "xs",
              "color": "#FFFFFF",
              "weight": "bold"
            }
          ]
        }
      ]
    },
    "body": {
      "type": "box",
      "layout": "vertical",
      "paddingAll": "20px",
      "spacing": "md",
      "contents": [
        {
          "type": "box",
          "layout": "horizontal",
          "spacing": "sm",
          "contents": [
            {
              "type": "box",
              "layout": "vertical",
              "flex": 0,
              "paddingAll": "8px 12px",
              "backgroundColor": "#E3F2FD",
              "cornerRadius": "8px",
              "contents": [
                {
                  "type": "text",
                  "text": "20",
                  "weight": "bold",
                  "size": "xl",
                  "color": "#1565C0",
                  "align": "center"
                },
                {
                  "type": "text",
                  "text": "ม.ค.",
                  "size": "xs",
                  "color": "#1565C0",
                  "align": "center"
                }
              ]
            },
            {
              "type": "box",
              "layout": "vertical",
              "flex": 1,
              "justifyContent": "center",
              "contents": [
                {
                  "type": "text",
                  "text": "📅 วันจันทร์ที่ 20 มกราคม 2025",
                  "size": "sm",
                  "color": "#333333"
                },
                {
                  "type": "text",
                  "text": "⏰ 09:00 - 18:00 น.",
                  "size": "sm",
                  "color": "#555555",
                  "margin": "xs"
                }
              ]
            }
          ]
        },
        {
          "type": "box",
          "layout": "horizontal",
          "spacing": "sm",
          "contents": [
            {
              "type": "text",
              "text": "📍",
              "size": "sm",
              "flex": 0
            },
            {
              "type": "text",
              "text": "BITEC บางนา, Hall 5-6\nกรุงเทพมหานคร",
              "size": "sm",
              "color": "#555555",
              "wrap": true,
              "flex": 1
            }
          ]
        },
        {
          "type": "separator",
          "color": "#eeeeee"
        },
        {
          "type": "text",
          "text": "หัวข้อการบรรยาย",
          "weight": "bold",
          "size": "sm",
          "color": "#555555"
        },
        {
          "type": "box",
          "layout": "vertical",
          "spacing": "xs",
          "contents": [
            {
              "type": "text",
              "text": "• Flex Message Advanced Techniques",
              "size": "sm",
              "color": "#333333"
            },
            {
              "type": "text",
              "text": "• LINE LIFF 2.0 Updates",
              "size": "sm",
              "color": "#333333"
            },
            {
              "type": "text",
              "text": "• Bot vs AI: Building Smarter Bots",
              "size": "sm",
              "color": "#333333"
            },
            {
              "type": "text",
              "text": "• Workshop: Building Real-world Bot",
              "size": "sm",
              "color": "#333333"
            }
          ]
        },
        {
          "type": "separator",
          "color": "#eeeeee"
        },
        {
          "type": "box",
          "layout": "horizontal",
          "contents": [
            {
              "type": "box",
              "layout": "vertical",
              "flex": 1,
              "contents": [
                {
                  "type": "text",
                  "text": "👥 ลงทะเบียนแล้ว",
                  "size": "xs",
                  "color": "#888888"
                },
                {
                  "type": "text",
                  "text": "347 / 500 คน",
                  "size": "sm",
                  "color": "#333333",
                  "weight": "bold"
                }
              ]
            },
            {
              "type": "box",
              "layout": "vertical",
              "flex": 1,
              "contents": [
                {
                  "type": "text",
                  "text": "💰 ค่าลงทะเบียน",
                  "size": "xs",
                  "color": "#888888"
                },
                {
                  "type": "text",
                  "text": "ฟรี!",
                  "size": "sm",
                  "color": "#4CAF50",
                  "weight": "bold"
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
      "paddingAll": "16px",
      "contents": [
        {
          "type": "button",
          "style": "primary",
          "color": "#1565C0",
          "action": {
            "type": "uri",
            "label": "📝 สมัครเข้าร่วม (ฟรี!)",
            "uri": "https://example.com/events/lds2025/register"
          }
        },
        {
          "type": "box",
          "layout": "horizontal",
          "spacing": "sm",
          "contents": [
            {
              "type": "button",
              "style": "secondary",
              "height": "sm",
              "flex": 1,
              "action": {
                "type": "uri",
                "label": "🗺️ แผนที่",
                "uri": "https://maps.google.com/?q=BITEC+Bangkok"
              }
            },
            {
              "type": "button",
              "style": "secondary",
              "height": "sm",
              "flex": 1,
              "action": {
                "type": "uri",
                "label": "📤 แชร์งาน",
                "uri": "https://example.com/share/lds2025"
              }
            }
          ]
        }
      ]
    },
    "styles": {
      "body": {
        "backgroundColor": "#FFFFFF"
      },
      "footer": {
        "backgroundColor": "#F8F9FA",
        "separator": true,
        "separatorColor": "#dee2e6"
      }
    }
  }
}
```

---

## 12. Best Practices

### Design Guidelines

```
✅ ใช้ size "mega" สำหรับ content ทั่วไป
✅ กำหนด aspectRatio ให้ hero image เสมอ
✅ ใช้ paddingAll แทน paddingTop/Bottom/Start/End เมื่อทำได้
✅ ใช้ spacing แทนการเพิ่ม margin ทีละ element
✅ กำหนด cornerRadius สำหรับ box ที่เป็น badge/tag
✅ ใช้ separator เพื่อแบ่งเนื้อหาที่ต่างประเภท
✅ header สีเข้ม = text สีขาว
✅ ทดสอบทั้ง iOS และ Android เสมอ
```

### Performance

```
⚡ รูปภาพขนาดเล็กกว่า 1MB สำหรับ hero
⚡ ไม่ควร nest box เกิน 5 ระดับ
⚡ ใช้ maxLines เพื่อจำกัดการ render
⚡ หลีกเลี่ยง icon เยอะเกินไปใน carousel
```

### Accessibility

```
♿ altText อธิบายเนื้อหาสำคัญ
♿ button label ชัดเจน ไม่คลุมเครือ
♿ contrast ratio ระหว่าง text และ background
♿ ไม่ใช้สีเป็นเพียงสิ่งเดียวในการสื่อสาร
```

---

## 13. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Business Card

สร้าง bubble แสดง business card ที่มี:
- Hero: ภาพโปรไฟล์ (1:1 ratio)
- Body: ชื่อ, ตำแหน่ง, บริษัท, อีเมล, เบอร์โทร
- Footer: ปุ่ม "โทรเลย" และ "ส่งอีเมล"

### แบบฝึกหัดที่ 2: Hotel Room Card

สร้าง bubble แสดงห้องพักโรงแรมที่มี:
- Hero: รูปห้องพัก (20:13)
- Header: ชื่อโรงแรม
- Body: ชื่อห้อง, จำนวนที่นอน, สิ่งอำนวยความสะดวก, ราคาต่อคืน
- Footer: ปุ่ม "จองเลย" และ "ดูรูปเพิ่มเติม"

### แบบฝึกหัดที่ 3: Task Status Card

สร้าง bubble แสดงสถานะ task ใน project:
- Header: ชื่อ project + status badge
- Body: ชื่อ task, ผู้รับผิดชอบ, deadline, progress bar (ใช้ box)
- Footer: ปุ่ม "อัพเดทสถานะ" และ "ดูรายละเอียด"

---

## สรุป

ใน Part นี้เราได้เรียนรู้ Bubble Container อย่างละเอียด:

1. **โครงสร้าง** และ properties ทั้งหมดของ Bubble
2. **Size options** ตั้งแต่ nano ถึง giga
3. **Direction** สำหรับภาษาที่อ่านจากขวาไปซ้าย
4. **Header, Hero, Body, Footer** และการใช้งานแต่ละส่วน
5. **Styles object** สำหรับ customize สีพื้นหลังและ separator
6. **3 ตัวอย่างสมบูรณ์**: Product Card, Receipt, Event Invitation

ใน Part ต่อไปเราจะเรียนรู้ Components ภายใน Bubble อย่างละเอียด เช่น Box, Button, Image, Text, Icon, Video

---

## Resources

- [Flex Message Bubble Documentation](https://developers.line.biz/en/docs/messaging-api/flex-message-elements/#bubble)
- [Flex Message Simulator](https://developers.line.biz/flex-simulator/)
- [Flex Message Layout Guide](https://developers.line.biz/en/docs/messaging-api/flex-message-layout/)
