# Part 33: Flex Message Components โดยละเอียด

## สารบัญ

1. [Box Component](#box-component)
2. [Button Component](#button-component)
3. [Image Component](#image-component)
4. [Text Component](#text-component)
5. [Icon Component](#icon-component)
6. [Video Component](#video-component)
7. [Separator Component](#separator-component)
8. [Filler Component](#filler-component)
9. [Spacer Component](#spacer-component)
10. [Span Component](#span-component)
11. [Component Combinations](#component-combinations)
12. [Common Patterns](#common-patterns)
13. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Box Component

Box เป็น layout container หลักใน Flex Message ทำหน้าที่จัด layout ของ child components

### Properties ทั้งหมด

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| type | string | ✅ | - | "box" |
| layout | string | ✅ | - | "vertical", "horizontal", "baseline" |
| contents | array | ✅ | - | array ของ child components |
| flex | number | ❌ | - | proportion ใน parent flex container |
| spacing | string | ❌ | "none" | ระยะห่างระหว่าง items |
| margin | string | ❌ | - | margin รอบ box |
| padding | string | ❌ | - | padding ภายใน box |
| paddingAll | string | ❌ | - | padding ทุกด้าน (shorthand) |
| paddingTop | string | ❌ | - | padding ด้านบน |
| paddingBottom | string | ❌ | - | padding ด้านล่าง |
| paddingStart | string | ❌ | - | padding ด้านซ้าย (ltr) |
| paddingEnd | string | ❌ | - | padding ด้านขวา (ltr) |
| backgroundColor | string | ❌ | - | สีพื้นหลัง |
| borderColor | string | ❌ | - | สีขอบ |
| borderWidth | string | ❌ | - | ความหนาขอบ |
| cornerRadius | string | ❌ | - | มุมโค้ง |
| width | string | ❌ | - | ความกว้าง (px หรือ %) |
| height | string | ❌ | - | ความสูง (px หรือ %) |
| maxWidth | string | ❌ | - | ความกว้างสูงสุด |
| maxHeight | string | ❌ | - | ความสูงสูงสุด |
| position | string | ❌ | "relative" | "relative" หรือ "absolute" |
| offsetTop | string | ❌ | - | offset จากด้านบน (absolute positioning) |
| offsetBottom | string | ❌ | - | offset จากด้านล่าง |
| offsetStart | string | ❌ | - | offset จากด้านซ้าย |
| offsetEnd | string | ❌ | - | offset จากด้านขวา |
| action | object | ❌ | - | action เมื่อแตะ box |
| justifyContent | string | ❌ | - | "flex-start", "center", "flex-end", "space-between", "space-around", "space-evenly" |
| alignItems | string | ❌ | - | "flex-start", "center", "flex-end" |
| background | object | ❌ | - | background gradient |

### Layout: Vertical

จัด items แนวตั้ง (บนลงล่าง)

```json
{
  "type": "box",
  "layout": "vertical",
  "spacing": "md",
  "contents": [
    { "type": "text", "text": "รายการที่ 1" },
    { "type": "text", "text": "รายการที่ 2" },
    { "type": "text", "text": "รายการที่ 3" }
  ]
}
```

ผลลัพธ์:
```
รายการที่ 1
รายการที่ 2
รายการที่ 3
```

### Layout: Horizontal

จัด items แนวนอน (ซ้ายไปขวา)

```json
{
  "type": "box",
  "layout": "horizontal",
  "contents": [
    {
      "type": "text",
      "text": "ซ้าย",
      "flex": 1
    },
    {
      "type": "text",
      "text": "กลาง",
      "flex": 1,
      "align": "center"
    },
    {
      "type": "text",
      "text": "ขวา",
      "flex": 1,
      "align": "end"
    }
  ]
}
```

ผลลัพธ์: `ซ้าย        กลาง        ขวา`

### Layout: Baseline

จัด items แนวนอนโดย align ที่ baseline ของ text (เหมาะสำหรับ icon + text)

```json
{
  "type": "box",
  "layout": "baseline",
  "spacing": "xs",
  "contents": [
    {
      "type": "icon",
      "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png",
      "size": "sm"
    },
    {
      "type": "text",
      "text": "4.9 (100 รีวิว)",
      "size": "sm",
      "color": "#888888"
    }
  ]
}
```

### Spacing Values

```
none → 0px
xs   → 4px
sm   → 8px
md   → 12px (default สำหรับ spacing)
lg   → 16px
xl   → 20px
xxl  → 24px
```

### Padding Values

Padding ใช้ค่าเดียวกับ Spacing รวมถึงค่า pixel เช่น "4px", "8px 16px"

```json
// ใช้ keyword
"paddingAll": "md"

// ใช้ pixel
"paddingAll": "16px"

// ใช้ top/right/bottom/left
"paddingTop": "16px",
"paddingBottom": "8px",
"paddingStart": "20px",
"paddingEnd": "20px"
```

### Flex Property

flex กำหนดสัดส่วนพื้นที่ของ item ใน parent flex container

```json
// แบ่งพื้นที่ 1:2:1
{
  "type": "box",
  "layout": "horizontal",
  "contents": [
    { "type": "text", "text": "A", "flex": 1 },  // 25%
    { "type": "text", "text": "B", "flex": 2 },  // 50%
    { "type": "text", "text": "C", "flex": 1 }   // 25%
  ]
}
```

```json
// ปิด flex (ใช้ขนาดตาม content)
{
  "type": "box",
  "layout": "horizontal",
  "contents": [
    { "type": "text", "text": "Label:", "flex": 0 },  // ขนาดตาม text
    { "type": "text", "text": "Value" }                // กินพื้นที่ที่เหลือ
  ]
}
```

### Position: Absolute

ใช้สำหรับวาง element ทับบน element อื่น

```json
{
  "type": "box",
  "layout": "vertical",
  "contents": [
    {
      "type": "image",
      "url": "https://example.com/product.jpg",
      "size": "full",
      "aspectRatio": "1:1",
      "aspectMode": "cover"
    },
    {
      "type": "box",
      "layout": "vertical",
      "position": "absolute",
      "offsetTop": "8px",
      "offsetEnd": "8px",
      "paddingAll": "4px 8px",
      "backgroundColor": "#FF5722",
      "cornerRadius": "4px",
      "contents": [
        {
          "type": "text",
          "text": "HOT",
          "size": "xs",
          "color": "#FFFFFF",
          "weight": "bold"
        }
      ]
    }
  ]
}
```

### Background Gradient

```json
{
  "type": "box",
  "layout": "vertical",
  "paddingAll": "20px",
  "background": {
    "type": "linearGradient",
    "angle": "135deg",
    "startColor": "#1565C0",
    "endColor": "#E91E63",
    "centerColor": "#9C27B0",
    "centerPosition": "50%"
  },
  "contents": [
    {
      "type": "text",
      "text": "Gradient Background",
      "color": "#FFFFFF",
      "weight": "bold"
    }
  ]
}
```

### JustifyContent

จัด items ตาม main axis ของ horizontal layout

```json
// space-between - กระจายห่างเท่ากัน
{
  "type": "box",
  "layout": "horizontal",
  "justifyContent": "space-between",
  "contents": [
    { "type": "text", "text": "A", "flex": 0 },
    { "type": "text", "text": "B", "flex": 0 },
    { "type": "text", "text": "C", "flex": 0 }
  ]
}
// ผลลัพธ์: A              B              C
```

### ตัวอย่าง Box ที่ซับซ้อน: Progress Bar

```json
{
  "type": "box",
  "layout": "vertical",
  "spacing": "xs",
  "contents": [
    {
      "type": "box",
      "layout": "horizontal",
      "contents": [
        {
          "type": "text",
          "text": "ความคืบหน้า",
          "size": "xs",
          "color": "#888888",
          "flex": 1
        },
        {
          "type": "text",
          "text": "75%",
          "size": "xs",
          "color": "#1565C0",
          "weight": "bold",
          "flex": 0
        }
      ]
    },
    {
      "type": "box",
      "layout": "horizontal",
      "height": "6px",
      "backgroundColor": "#E0E0E0",
      "cornerRadius": "3px",
      "contents": [
        {
          "type": "box",
          "layout": "vertical",
          "backgroundColor": "#1565C0",
          "cornerRadius": "3px",
          "flex": 75,
          "contents": []
        },
        {
          "type": "box",
          "layout": "vertical",
          "flex": 25,
          "contents": []
        }
      ]
    }
  ]
}
```

---

## 2. Button Component

Button ใช้สำหรับสร้างปุ่มกด action ต่างๆ

### Properties ทั้งหมด

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| type | string | ✅ | - | "button" |
| action | object | ✅ | - | action เมื่อกดปุ่ม |
| flex | number | ❌ | 1 | proportion ใน parent |
| margin | string | ❌ | - | margin รอบปุ่ม |
| height | string | ❌ | "md" | "sm" หรือ "md" |
| style | string | ❌ | "link" | "primary", "secondary", "link" |
| color | string | ❌ | - | สีปุ่ม (สำหรับ primary และ secondary) |
| gravity | string | ❌ | "top" | "top", "bottom", "center" |
| position | string | ❌ | "relative" | "relative" หรือ "absolute" |
| offsetTop/Bottom/Start/End | string | ❌ | - | สำหรับ absolute position |
| scaling | boolean | ❌ | false | scale to fit container |
| adjustMode | string | ❌ | - | "shrink-to-fit" |

### Button Styles

**primary** - ปุ่มสีเข้ม (default: สีน้ำเงินของ LINE)
```json
{
  "type": "button",
  "style": "primary",
  "action": {
    "type": "uri",
    "label": "ซื้อเลย",
    "uri": "https://example.com"
  }
}
```

ผลลัพธ์:
```
┌─────────────────────────────┐
│           ซื้อเลย            │  ← สีน้ำเงิน background, text ขาว
└─────────────────────────────┘
```

**secondary** - ปุ่มสีอ่อน (outline style)
```json
{
  "type": "button",
  "style": "secondary",
  "action": {
    "type": "uri",
    "label": "ดูเพิ่มเติม",
    "uri": "https://example.com"
  }
}
```

ผลลัพธ์:
```
┌─────────────────────────────┐
│         ดูเพิ่มเติม          │  ← ขอบสีเทา, text สีเทา
└─────────────────────────────┘
```

**link** - ปุ่มแบบ text link
```json
{
  "type": "button",
  "style": "link",
  "action": {
    "type": "uri",
    "label": "นโยบายความเป็นส่วนตัว",
    "uri": "https://example.com/privacy"
  }
}
```

ผลลัพธ์:
```
         นโยบายความเป็นส่วนตัว          ← text เท่านั้น ไม่มี background
```

### Button Height

```json
// sm - ขนาดเล็ก (ใช้ใน footer row)
{
  "type": "button",
  "style": "primary",
  "height": "sm",
  "action": { ... }
}

// md - ขนาดปกติ (default)
{
  "type": "button",
  "style": "primary",
  "height": "md",
  "action": { ... }
}
```

### Custom Button Color

```json
// Override สีของ primary button
{
  "type": "button",
  "style": "primary",
  "color": "#FF5722",      // สีส้ม-แดง
  "action": {
    "type": "postback",
    "label": "สั่งซื้อ",
    "data": "action=buy"
  }
}

// Override สีของ secondary button
{
  "type": "button",
  "style": "secondary",
  "color": "#1565C0",      // กำหนดสีขอบและ text
  "action": { ... }
}
```

### Action Types

**uri** - เปิด URL
```json
{
  "type": "uri",
  "label": "เปิดเว็บไซต์",
  "uri": "https://example.com"
}
```

**postback** - ส่ง postback event ไปที่ webhook
```json
{
  "type": "postback",
  "label": "เพิ่มลงตะกร้า",
  "data": "action=addCart&productId=123",
  "displayText": "เพิ่มสินค้าแล้ว"  // text ที่แสดงใน chat (optional)
}
```

**message** - ส่ง text message
```json
{
  "type": "message",
  "label": "สอบถามเพิ่มเติม",
  "text": "ฉันต้องการสอบถามเกี่ยวกับสินค้า"
}
```

**datetimepicker** - เปิด date/time picker
```json
{
  "type": "datetimepicker",
  "label": "เลือกวันที่",
  "data": "action=selectDate",
  "mode": "date",           // "date", "time", "datetime"
  "initial": "2025-01-15",  // ค่าเริ่มต้น
  "min": "2025-01-01",
  "max": "2025-12-31"
}
```

**camera** - เปิดกล้อง
```json
{
  "type": "camera",
  "label": "ถ่ายภาพ"
}
```

**cameraRoll** - เปิด photo gallery
```json
{
  "type": "cameraRoll",
  "label": "เลือกภาพ"
}
```

**location** - ส่ง location
```json
{
  "type": "location",
  "label": "ส่งที่อยู่"
}
```

### ตัวอย่าง Button Row แบบต่างๆ

```json
// 2 ปุ่ม side by side
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
        "label": "❌ ปฏิเสธ",
        "data": "action=reject"
      }
    },
    {
      "type": "button",
      "style": "primary",
      "height": "sm",
      "flex": 1,
      "color": "#4CAF50",
      "action": {
        "type": "postback",
        "label": "✅ ยืนยัน",
        "data": "action=confirm"
      }
    }
  ]
}
```

---

## 3. Image Component

Image ใช้แสดงรูปภาพใน Flex Message

### Properties ทั้งหมด

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| type | string | ✅ | - | "image" |
| url | string | ✅ | - | URL ของรูปภาพ (HTTPS) |
| flex | number | ❌ | 1 | proportion ใน parent |
| margin | string | ❌ | - | margin รอบรูปภาพ |
| align | string | ❌ | "center" | "start", "end", "center" |
| gravity | string | ❌ | "top" | "top", "bottom", "center" |
| size | string | ❌ | "md" | xxs, xs, sm, md, lg, xl, xxl, 3xl, 4xl, 5xl, full |
| aspectRatio | string | ❌ | "1:1" | "w:h" เช่น "20:13" |
| aspectMode | string | ❌ | "fit" | "cover" หรือ "fit" |
| backgroundColor | string | ❌ | - | สีพื้นหลังของกรอบรูป |
| action | object | ❌ | - | action เมื่อแตะรูป |
| animated | boolean | ❌ | false | true สำหรับ animated GIF |
| position | string | ❌ | "relative" | "relative" หรือ "absolute" |
| offsetTop/Bottom/Start/End | string | ❌ | - | สำหรับ absolute position |

### Image Sizes

```
xxs  → ~28px   ใช้สำหรับ small icon
xs   → ~36px   ใช้สำหรับ small avatar
sm   → ~48px   ใช้สำหรับ avatar
md   → ~60px   ขนาดกลาง
lg   → ~80px   ขนาดใหญ่
xl   → ~100px  ใช้สำหรับ product thumbnail
xxl  → ~180px  ใช้สำหรับ large thumbnail
3xl  → ~220px
4xl  → ~260px
5xl  → ~300px
full → เต็มความกว้าง (ใช้บ่อยสุดสำหรับ hero)
```

### aspectRatio ที่ใช้บ่อย

```
1:1   → สี่เหลี่ยมจัตุรัส (profile, product)
4:3   → landscape ทั่วไป
16:9  → widescreen
20:13 → LINE hero image (แนะนำ)
2:3   → portrait
3:4   → portrait tall
9:16  → full-screen portrait
```

### aspectMode

```json
// cover: crop รูปให้เต็มพื้นที่ (ตัดขอบ)
{
  "type": "image",
  "url": "...",
  "aspectRatio": "20:13",
  "aspectMode": "cover"   // รูปเต็มพื้นที่ ไม่มีขอบขาว
}

// fit: ย่อรูปให้เห็นทั้งรูป (มีขอบขาวถ้า ratio ไม่ตรง)
{
  "type": "image",
  "url": "...",
  "aspectRatio": "20:13",
  "aspectMode": "fit"     // เห็นทั้งรูป แต่อาจมีขอบขาว
}
```

### Animated GIF

```json
{
  "type": "image",
  "url": "https://example.com/animation.gif",
  "size": "full",
  "aspectRatio": "1:1",
  "aspectMode": "cover",
  "animated": true      // ← ต้องระบุ animated: true
}
```

**ข้อจำกัด GIF:**
- ไฟล์ไม่เกิน 1MB
- resolution ไม่เกิน 240x240 px
- ต้องเป็น GIF format เท่านั้น (ไม่ใช่ MP4)

### Image ในรูปแบบต่างๆ

```json
// Avatar + name pattern
{
  "type": "box",
  "layout": "horizontal",
  "spacing": "sm",
  "contents": [
    {
      "type": "image",
      "url": "https://example.com/avatar.jpg",
      "size": "xs",
      "aspectRatio": "1:1",
      "aspectMode": "cover",
      "flex": 0,
      "cornerRadius": "50%"  // ← ทำให้เป็นวงกลม!
    },
    {
      "type": "box",
      "layout": "vertical",
      "flex": 1,
      "contents": [
        {
          "type": "text",
          "text": "สมชาย ใจดี",
          "weight": "bold",
          "size": "sm"
        },
        {
          "type": "text",
          "text": "2 นาทีที่แล้ว",
          "size": "xs",
          "color": "#888888"
        }
      ]
    }
  ]
}
```

```json
// Image grid 2x2
{
  "type": "box",
  "layout": "vertical",
  "spacing": "xs",
  "contents": [
    {
      "type": "box",
      "layout": "horizontal",
      "spacing": "xs",
      "contents": [
        {
          "type": "image",
          "url": "https://example.com/img1.jpg",
          "size": "full",
          "aspectRatio": "1:1",
          "aspectMode": "cover",
          "flex": 1
        },
        {
          "type": "image",
          "url": "https://example.com/img2.jpg",
          "size": "full",
          "aspectRatio": "1:1",
          "aspectMode": "cover",
          "flex": 1
        }
      ]
    },
    {
      "type": "box",
      "layout": "horizontal",
      "spacing": "xs",
      "contents": [
        {
          "type": "image",
          "url": "https://example.com/img3.jpg",
          "size": "full",
          "aspectRatio": "1:1",
          "aspectMode": "cover",
          "flex": 1
        },
        {
          "type": "image",
          "url": "https://example.com/img4.jpg",
          "size": "full",
          "aspectRatio": "1:1",
          "aspectMode": "cover",
          "flex": 1
        }
      ]
    }
  ]
}
```

---

## 4. Text Component

Text เป็น component สำหรับแสดงข้อความ

### Properties ทั้งหมด

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| type | string | ✅ | - | "text" |
| text | string | ✅/❌ | - | ข้อความ (required ถ้าไม่มี contents) |
| contents | array | ❌ | - | array ของ span (แทน text) |
| adjustMode | string | ❌ | - | "shrink-to-fit" |
| flex | number | ❌ | 1 | proportion ใน parent |
| margin | string | ❌ | - | margin รอบ text |
| position | string | ❌ | "relative" | "relative" หรือ "absolute" |
| size | string | ❌ | "md" | xxs, xs, sm, md, lg, xl, xxl, 3xl, 4xl, 5xl |
| align | string | ❌ | "start" | "start", "end", "center" |
| gravity | string | ❌ | "top" | "top", "bottom", "center" |
| wrap | boolean | ❌ | false | ตัดบรรทัดอัตโนมัติ |
| maxLines | number | ❌ | - | จำกัดจำนวนบรรทัดสูงสุด |
| weight | string | ❌ | "regular" | "regular" หรือ "bold" |
| color | string | ❌ | - | สีตัวอักษร (#RRGGBB) |
| style | string | ❌ | "normal" | "normal", "italic" |
| decoration | string | ❌ | "none" | "none", "underline", "line-through" |
| lineSpacing | string | ❌ | - | ระยะห่างระหว่างบรรทัด |
| offsetTop/Bottom/Start/End | string | ❌ | - | สำหรับ absolute position |
| action | object | ❌ | - | action เมื่อแตะ text |

### Text Sizes

```
xxs  → 9px   ขนาดเล็กมาก
xs   → 10px  ขนาดเล็ก
sm   → 12px  ขนาดเล็กปกติ
md   → 14px  ขนาดปกติ (default)
lg   → 16px  ขนาดกลาง
xl   → 18px  ขนาดใหญ่
xxl  → 20px  ขนาดใหญ่มาก
3xl  → 24px  heading
4xl  → 28px  large heading
5xl  → 32px  extra large heading
```

### Text Properties ที่สำคัญ

**weight (bold)**
```json
{ "type": "text", "text": "หัวข้อ", "weight": "bold" }
```

**color (hex)**
```json
{ "type": "text", "text": "ข้อความสีแดง", "color": "#FF0000" }
{ "type": "text", "text": "ข้อความโปร่งใส 50%", "color": "#FF000080" }
```

**wrap (line break)**
```json
{
  "type": "text",
  "text": "ข้อความยาวๆ ที่อาจต้องตัดบรรทัด เพื่อให้แสดงผลครบ",
  "wrap": true,        // อนุญาตตัดบรรทัด
  "maxLines": 3        // จำกัดสูงสุด 3 บรรทัด
}
```

**align**
```json
{ "type": "text", "text": "ชิดซ้าย", "align": "start" }
{ "type": "text", "text": "กึ่งกลาง", "align": "center" }
{ "type": "text", "text": "ชิดขวา", "align": "end" }
```

**style (italic)**
```json
{ "type": "text", "text": "ข้อความเอียง", "style": "italic" }
```

**decoration**
```json
{ "type": "text", "text": "ขีดเส้นใต้", "decoration": "underline" }
{ "type": "text", "text": "ขีดทับ ราคาเดิม", "decoration": "line-through" }
```

**lineSpacing**
```json
{
  "type": "text",
  "text": "ข้อความหลายบรรทัด\nที่มีระยะห่างบรรทัดที่กำหนดเอง",
  "wrap": true,
  "lineSpacing": "20px"  // หรือ "8pt"
}
```

### Text Multiline ด้วย \n

```json
{
  "type": "text",
  "text": "บรรทัดที่ 1\nบรรทัดที่ 2\nบรรทัดที่ 3",
  "wrap": true
}
```

### Text ที่มี Clickable Action

```json
{
  "type": "text",
  "text": "คลิกที่นี่",
  "color": "#1565C0",
  "decoration": "underline",
  "action": {
    "type": "uri",
    "uri": "https://example.com"
  }
}
```

---

## 5. Icon Component

Icon ใช้แสดงรูปไอคอนขนาดเล็ก มักใช้ร่วมกับ text ใน baseline layout

### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| type | string | ✅ | - | "icon" |
| url | string | ✅ | - | URL ของไอคอน (HTTPS, PNG/JPG) |
| margin | string | ❌ | - | margin รอบ icon |
| size | string | ❌ | "md" | xxs, xs, sm, md, lg, xl, xxl, 3xl, 4xl, 5xl |
| aspectRatio | string | ❌ | "1:1" | ratio ของ icon |
| position | string | ❌ | "relative" | "relative" หรือ "absolute" |
| offsetTop/Bottom/Start/End | string | ❌ | - | สำหรับ absolute position |

### ตัวอย่างการใช้ Icon

```json
// Star rating
{
  "type": "box",
  "layout": "baseline",
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
      "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gray_star_28.png",
      "size": "sm"
    },
    {
      "type": "icon",
      "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gray_star_28.png",
      "size": "sm"
    },
    {
      "type": "text",
      "text": "3.0 (50 รีวิว)",
      "size": "sm",
      "color": "#888888",
      "margin": "xs"
    }
  ]
}
```

### LINE Official Icon URLs

```
Gold Star (full):
https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png

Gray Star (empty):
https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gray_star_28.png

Note: ใช้ icon URL ของ LINE โดยตรงสำหรับ star rating
สำหรับ icon อื่น ควร host เองบน CDN ที่รองรับ HTTPS
```

### Custom Icons

```json
// Custom icon จาก CDN ของคุณ
{
  "type": "icon",
  "url": "https://cdn.example.com/icons/phone.png",
  "size": "sm"
}

// Icon ใน baseline layout
{
  "type": "box",
  "layout": "baseline",
  "spacing": "xs",
  "contents": [
    {
      "type": "icon",
      "url": "https://cdn.example.com/icons/phone.png",
      "size": "xs"
    },
    {
      "type": "text",
      "text": "02-123-4567",
      "size": "sm",
      "color": "#333333"
    }
  ]
}
```

---

## 6. Video Component

Video ใช้แสดง video player ใน Flex Message (ใช้ใน hero section เท่านั้น)

### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| type | string | ✅ | - | "video" |
| url | string | ✅ | - | URL ของ video (HTTPS, MP4) |
| previewUrl | string | ✅ | - | URL ของ thumbnail รูป (HTTPS, JPG/PNG) |
| altContent | object | ✅ | - | content แทนเมื่อ video ไม่รองรับ |
| action | object | ✅ | - | action เมื่อกด video |

### ตัวอย่าง Video Component

```json
{
  "type": "flex",
  "altText": "ดูวิดีโอสินค้า",
  "contents": {
    "type": "bubble",
    "hero": {
      "type": "video",
      "url": "https://example.com/product-demo.mp4",
      "previewUrl": "https://example.com/product-thumbnail.jpg",
      "altContent": {
        "type": "image",
        "url": "https://example.com/product-thumbnail.jpg",
        "size": "full",
        "aspectRatio": "20:13",
        "aspectMode": "cover"
      },
      "action": {
        "type": "uri",
        "label": "ดูวิดีโอ",
        "uri": "https://example.com/products/video/123"
      }
    },
    "body": {
      "type": "box",
      "layout": "vertical",
      "contents": [
        {
          "type": "text",
          "text": "วิดีโอสาธิตการใช้งาน",
          "weight": "bold",
          "size": "lg"
        }
      ]
    }
  }
}
```

### Video Specifications

```
Format: MP4
Max size: 200MB
Recommended: H.264 codec, AAC audio
HTTPS required
```

---

## 7. Separator Component

Separator ใช้สร้างเส้นแบ่งแนวนอนระหว่าง components

### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| type | string | ✅ | - | "separator" |
| margin | string | ❌ | - | margin รอบ separator |
| color | string | ❌ | "#cccccc" | สีของเส้น |

### ตัวอย่างการใช้ Separator

```json
// Separator พื้นฐาน
{
  "type": "separator"
}

// Separator พร้อม margin
{
  "type": "separator",
  "margin": "lg",
  "color": "#eeeeee"
}

// Separator สีเข้ม
{
  "type": "separator",
  "margin": "md",
  "color": "#333333"
}
```

### การใช้ Separator ใน context

```json
{
  "type": "box",
  "layout": "vertical",
  "contents": [
    { "type": "text", "text": "หัวข้อที่ 1", "weight": "bold" },
    { "type": "text", "text": "รายละเอียดส่วนที่ 1", "size": "sm" },
    
    { "type": "separator", "margin": "md", "color": "#eeeeee" },  // ← แบ่งส่วน
    
    { "type": "text", "text": "หัวข้อที่ 2", "weight": "bold", "margin": "md" },
    { "type": "text", "text": "รายละเอียดส่วนที่ 2", "size": "sm" }
  ]
}
```

---

## 8. Filler Component

Filler ใช้เติมพื้นที่ว่างระหว่าง components ใน horizontal layout

### Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| type | string | ✅ | - | "filler" |
| flex | number | ❌ | 1 | proportion ของพื้นที่ว่าง |

### ตัวอย่างการใช้ Filler

```json
// ดัน text ไปชิดขวา
{
  "type": "box",
  "layout": "horizontal",
  "contents": [
    {
      "type": "text",
      "text": "Label",
      "size": "sm",
      "flex": 0
    },
    {
      "type": "filler"    // เติมพื้นที่ว่าง
    },
    {
      "type": "text",
      "text": "Value",
      "size": "sm",
      "flex": 0
    }
  ]
}
// ผลลัพธ์: Label                              Value
```

```json
// Center แบบ manual
{
  "type": "box",
  "layout": "horizontal",
  "contents": [
    { "type": "filler" },                     // พื้นที่ซ้าย
    { "type": "text", "text": "กึ่งกลาง", "flex": 0 },
    { "type": "filler" }                      // พื้นที่ขวา
  ]
}
```

---

## 9. Spacer Component

> **⚠️ Deprecated:** Spacer ถูก deprecated แล้ว ควรใช้ margin property แทน

```json
// ❌ เก่า - ไม่แนะนำ
{ "type": "spacer", "size": "md" }

// ✅ ใหม่ - แนะนำ
// ใช้ margin บน component ถัดไป
{ "type": "text", "text": "...", "margin": "md" }
```

---

## 10. Span Component

Span ใช้สำหรับสร้าง inline text ที่มี style ต่างกันภายใน text เดียวกัน (mixed styling)

### ใช้เมื่อต้องการ**ข้อความหลายสไตล์ในบรรทัดเดียวกัน**

```json
{
  "type": "text",
  "contents": [
    {
      "type": "span",
      "text": "ราคา: "
    },
    {
      "type": "span",
      "text": "฿1,290",
      "color": "#FF5722",
      "weight": "bold",
      "size": "lg"
    },
    {
      "type": "span",
      "text": " (ลด 20%)",
      "color": "#4CAF50",
      "size": "sm"
    }
  ]
}
```

ผลลัพธ์: `ราคา: ฿1,290 (ลด 20%)`
(ราคาสีส้ม bold, ส่วนลดสีเขียวเล็ก)

### Span Properties

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| type | string | ✅ | - | "span" |
| text | string | ✅ | - | ข้อความ |
| color | string | ❌ | - | สีตัวอักษร |
| size | string | ❌ | - | ขนาด |
| weight | string | ❌ | - | "regular" หรือ "bold" |
| style | string | ❌ | - | "normal" หรือ "italic" |
| decoration | string | ❌ | - | "underline", "line-through" |
| lineSpacing | string | ❌ | - | ระยะห่างบรรทัด |
| action | object | ❌ | - | action เมื่อแตะ span |

### ตัวอย่าง Span ที่ซับซ้อน

```json
// Span แสดงสถานะ
{
  "type": "text",
  "contents": [
    { "type": "span", "text": "สถานะ: " },
    {
      "type": "span",
      "text": "✅ ชำระแล้ว",
      "color": "#4CAF50",
      "weight": "bold"
    }
  ]
}

// Span แสดงราคาพร้อมส่วนลด
{
  "type": "text",
  "contents": [
    { "type": "span", "text": "ราคาเดิม: " },
    {
      "type": "span",
      "text": "฿500",
      "color": "#aaaaaa",
      "decoration": "line-through"
    },
    { "type": "span", "text": " → ราคาพิเศษ: " },
    {
      "type": "span",
      "text": "฿350",
      "color": "#FF5722",
      "weight": "bold",
      "size": "lg"
    }
  ]
}
```

---

## 11. Component Combinations

### Pattern: Info Row (Label + Value)

```json
{
  "type": "box",
  "layout": "horizontal",
  "contents": [
    {
      "type": "text",
      "text": "ชื่อผู้รับ",
      "size": "sm",
      "color": "#888888",
      "flex": 2
    },
    {
      "type": "text",
      "text": "สมชาย ใจดี",
      "size": "sm",
      "color": "#333333",
      "flex": 3,
      "wrap": true
    }
  ]
}
```

### Pattern: Badge

```json
{
  "type": "box",
  "layout": "vertical",
  "flex": 0,
  "paddingAll": "4px 10px",
  "backgroundColor": "#FF5722",
  "cornerRadius": "20px",
  "contents": [
    {
      "type": "text",
      "text": "ใหม่",
      "size": "xs",
      "color": "#FFFFFF",
      "weight": "bold"
    }
  ]
}
```

### Pattern: Avatar + Info

```json
{
  "type": "box",
  "layout": "horizontal",
  "spacing": "md",
  "alignItems": "center",
  "contents": [
    {
      "type": "image",
      "url": "https://example.com/avatar.jpg",
      "size": "sm",
      "aspectRatio": "1:1",
      "aspectMode": "cover",
      "flex": 0
    },
    {
      "type": "box",
      "layout": "vertical",
      "flex": 1,
      "contents": [
        {
          "type": "text",
          "text": "สมชาย ใจดี",
          "weight": "bold",
          "size": "sm"
        },
        {
          "type": "text",
          "text": "นักพัฒนาซอฟต์แวร์",
          "size": "xs",
          "color": "#888888"
        }
      ]
    }
  ]
}
```

### Pattern: Status Chip

```json
{
  "type": "box",
  "layout": "horizontal",
  "spacing": "xs",
  "flex": 0,
  "paddingAll": "4px 10px",
  "backgroundColor": "#E8F5E9",
  "cornerRadius": "20px",
  "contents": [
    {
      "type": "text",
      "text": "●",
      "size": "xs",
      "color": "#4CAF50",
      "flex": 0
    },
    {
      "type": "text",
      "text": "พร้อมบริการ",
      "size": "xs",
      "color": "#2E7D32",
      "flex": 0
    }
  ]
}
```

### Pattern: Price Tag

```json
{
  "type": "box",
  "layout": "horizontal",
  "alignItems": "flex-end",
  "contents": [
    {
      "type": "text",
      "text": "฿",
      "size": "sm",
      "color": "#FF5722",
      "flex": 0,
      "margin": "none"
    },
    {
      "type": "text",
      "text": "1,290",
      "size": "xxl",
      "weight": "bold",
      "color": "#FF5722",
      "flex": 0
    },
    {
      "type": "text",
      "text": ".00",
      "size": "sm",
      "color": "#FF5722",
      "flex": 0
    }
  ]
}
```

### Pattern: Countdown / Stats Row

```json
{
  "type": "box",
  "layout": "horizontal",
  "contents": [
    {
      "type": "box",
      "layout": "vertical",
      "flex": 1,
      "alignItems": "center",
      "contents": [
        {
          "type": "text",
          "text": "1,234",
          "weight": "bold",
          "size": "xl",
          "align": "center"
        },
        {
          "type": "text",
          "text": "Followers",
          "size": "xs",
          "color": "#888888",
          "align": "center"
        }
      ]
    },
    {
      "type": "separator"
    },
    {
      "type": "box",
      "layout": "vertical",
      "flex": 1,
      "alignItems": "center",
      "contents": [
        {
          "type": "text",
          "text": "567",
          "weight": "bold",
          "size": "xl",
          "align": "center"
        },
        {
          "type": "text",
          "text": "Following",
          "size": "xs",
          "color": "#888888",
          "align": "center"
        }
      ]
    },
    {
      "type": "separator"
    },
    {
      "type": "box",
      "layout": "vertical",
      "flex": 1,
      "alignItems": "center",
      "contents": [
        {
          "type": "text",
          "text": "89",
          "weight": "bold",
          "size": "xl",
          "align": "center"
        },
        {
          "type": "text",
          "text": "Posts",
          "size": "xs",
          "color": "#888888",
          "align": "center"
        }
      ]
    }
  ]
}
```

---

## 12. Common Patterns

### Pattern: Notification Card

```json
{
  "type": "flex",
  "altText": "แจ้งเตือน: มีคำสั่งซื้อใหม่",
  "contents": {
    "type": "bubble",
    "size": "kilo",
    "body": {
      "type": "box",
      "layout": "horizontal",
      "spacing": "md",
      "paddingAll": "16px",
      "contents": [
        {
          "type": "box",
          "layout": "vertical",
          "flex": 0,
          "justifyContent": "center",
          "contents": [
            {
              "type": "image",
              "url": "https://cdn.example.com/icons/order.png",
              "size": "md",
              "aspectRatio": "1:1"
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
              "text": "คำสั่งซื้อใหม่",
              "weight": "bold",
              "size": "sm",
              "color": "#1a1a1a"
            },
            {
              "type": "text",
              "text": "#ORD-001 จาก สมชาย ใจดี",
              "size": "xs",
              "color": "#666666",
              "wrap": true,
              "margin": "xs"
            },
            {
              "type": "text",
              "text": "2 นาทีที่แล้ว",
              "size": "xxs",
              "color": "#aaaaaa",
              "margin": "xs"
            }
          ]
        }
      ]
    }
  }
}
```

### Pattern: Rating Card

```json
{
  "type": "flex",
  "altText": "รีวิวจากลูกค้า",
  "contents": {
    "type": "bubble",
    "size": "mega",
    "body": {
      "type": "box",
      "layout": "vertical",
      "paddingAll": "20px",
      "spacing": "md",
      "contents": [
        {
          "type": "box",
          "layout": "horizontal",
          "spacing": "md",
          "alignItems": "center",
          "contents": [
            {
              "type": "image",
              "url": "https://example.com/avatar.jpg",
              "size": "sm",
              "aspectRatio": "1:1",
              "aspectMode": "cover",
              "flex": 0
            },
            {
              "type": "box",
              "layout": "vertical",
              "flex": 1,
              "contents": [
                {
                  "type": "text",
                  "text": "นิรนาม",
                  "weight": "bold",
                  "size": "sm"
                },
                {
                  "type": "box",
                  "layout": "baseline",
                  "spacing": "xxs",
                  "contents": [
                    { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                    { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                    { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                    { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                    { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" }
                  ]
                }
              ]
            },
            {
              "type": "text",
              "text": "3 วันที่แล้ว",
              "size": "xxs",
              "color": "#aaaaaa",
              "flex": 0,
              "gravity": "bottom"
            }
          ]
        },
        {
          "type": "text",
          "text": "สินค้าดีมาก ส่งเร็ว บรรจุภัณฑ์ดี จะกลับมาซื้ออีกแน่นอน แนะนำเลยครับ!",
          "size": "sm",
          "color": "#333333",
          "wrap": true
        }
      ]
    }
  }
}
```

### Pattern: Timeline Step

```json
{
  "type": "box",
  "layout": "horizontal",
  "spacing": "md",
  "contents": [
    {
      "type": "box",
      "layout": "vertical",
      "flex": 0,
      "alignItems": "center",
      "contents": [
        {
          "type": "box",
          "layout": "vertical",
          "width": "24px",
          "height": "24px",
          "backgroundColor": "#4CAF50",
          "cornerRadius": "50%",
          "alignItems": "center",
          "justifyContent": "center",
          "contents": [
            {
              "type": "text",
              "text": "✓",
              "size": "xs",
              "color": "#FFFFFF"
            }
          ]
        },
        {
          "type": "box",
          "layout": "vertical",
          "width": "2px",
          "height": "30px",
          "backgroundColor": "#4CAF50",
          "contents": []
        }
      ]
    },
    {
      "type": "box",
      "layout": "vertical",
      "flex": 1,
      "paddingBottom": "md",
      "contents": [
        {
          "type": "text",
          "text": "รับออเดอร์แล้ว",
          "weight": "bold",
          "size": "sm",
          "color": "#4CAF50"
        },
        {
          "type": "text",
          "text": "15 ม.ค. 2025, 10:30 น.",
          "size": "xs",
          "color": "#888888"
        }
      ]
    }
  ]
}
```

---

## 13. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Custom Stats Box (ง่าย)

สร้าง box แสดงสถิติ dashboard ที่มี:
- 3 stats: ยอดขาย, จำนวนออเดอร์, ลูกค้าใหม่
- แต่ละ stat มีตัวเลขและ label
- ใช้ separator คั่นระหว่าง stats
- สีตัวเลขแตกต่างกัน

### แบบฝึกหัดที่ 2: Article Card (ปานกลาง)

สร้าง bubble สำหรับ article ที่มี:
- Image hero (ภาพบทความ)
- Category badge (สีต่างกันตาม category)
- หัวข้อ (bold, ขนาดใหญ่)
- ตัดย่อ (2-3 บรรทัด)
- Avatar ผู้เขียน + ชื่อ + วันที่
- ปุ่มอ่านต่อ

### แบบฝึกหัดที่ 3: Food Order Card (ยาก)

สร้าง bubble สำหรับร้านอาหารที่มี:
- Hero: รูปอาหาร
- Badge "เปิดอยู่" / "ปิด" (สีเขียว/แดง)
- ชื่อร้าน, ประเภทอาหาร
- Rating bar (ใช้ box แทน progress bar)
- ข้อมูล: เวลาส่ง, ค่าส่ง, MOQ
- 3 เมนูแนะนำ (ชื่อ + ราคา)
- ปุ่ม "สั่งอาหาร"

---

## สรุป

ใน Part นี้เราได้เรียนรู้ Components ทั้งหมดของ Flex Message อย่างละเอียด:

1. **Box** - Layout container หลัก (vertical/horizontal/baseline)
2. **Button** - ปุ่ม action (primary/secondary/link)
3. **Image** - รูปภาพพร้อม aspectRatio และ aspectMode
4. **Text** - ข้อความพร้อม styling เต็มรูปแบบ
5. **Icon** - ไอคอนขนาดเล็กสำหรับ inline
6. **Video** - วิดีโอใน hero section
7. **Separator** - เส้นแบ่ง
8. **Filler** - พื้นที่ว่าง
9. **Span** - Inline text หลาย style

ใน Part ต่อไปเราจะเรียนรู้การสร้าง Carousel Container

---

## Resources

- [Flex Message Components Reference](https://developers.line.biz/en/reference/messaging-api/#flex-message-components)
- [Flex Message Layout Guide](https://developers.line.biz/en/docs/messaging-api/flex-message-layout/)
- [Flex Message Simulator](https://developers.line.biz/flex-simulator/)
