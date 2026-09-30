# Part 41: รู้จักกับ LIFF (LINE Front-end Framework)

## สารบัญ

1. [LIFF คืออะไร?](#liff-คืออะไร)
2. [LIFF vs Regular Web](#liff-vs-regular-web)
3. [ฟีเจอร์หลักของ LIFF](#ฟีเจอร์หลักของ-liff)
4. [LIFF URL Types](#liff-url-types)
5. [ขนาด LIFF และการใช้งาน](#ขนาด-liff-และการใช้งาน)
6. [การสร้าง LIFF App ใน Developer Console](#การสร้าง-liff-app)
7. [LIFF ID และ Endpoint URL](#liff-id-และ-endpoint-url)
8. [การติดตั้ง LIFF SDK](#การติดตั้ง-liff-sdk)
9. [Hello World LIFF App](#hello-world-liff-app)
10. [เปิด LIFF จาก Rich Menu](#เปิด-liff-จาก-rich-menu)
11. [เปิด LIFF จาก Flex Message](#เปิด-liff-จาก-flex-message)
12. [Best Practices](#best-practices)
13. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. LIFF คืออะไร?

**LIFF (LINE Front-end Framework)** คือ Web Application Platform ที่ทำงานภายใน LINE App โดยตรง ทำให้นักพัฒนาสามารถสร้าง Web App ที่มีความสามารถเฉพาะตัวในการเข้าถึงข้อมูล LINE ของผู้ใช้ได้

### ความเป็นมาของ LIFF

LINE ได้พัฒนา LIFF ขึ้นมาเพื่อตอบสนองความต้องการของนักพัฒนาที่ต้องการสร้าง User Experience ที่ดีกว่าการใช้แค่ Bot Message เพียงอย่างเดียว ด้วย LIFF ผู้ใช้สามารถโต้ตอบกับฟอร์ม, ดูข้อมูล, และทำธุรกรรมต่างๆ ได้โดยไม่ต้องออกจาก LINE

### จุดเด่นของ LIFF

```
┌─────────────────────────────────────────────────────────────┐
│                     LINE Application                         │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                    LIFF Webview                        │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │              Your Web Application                │  │  │
│  │  │                                                  │  │  │
│  │  │  • ใช้ HTML/CSS/JavaScript ปกติ                  │  │  │
│  │  │  • เข้าถึงข้อมูล LINE Profile                   │  │  │
│  │  │  • ส่งข้อความใน Chat                            │  │  │
│  │  │  • แชร์ Content ให้เพื่อน                       │  │  │
│  │  │  • ใช้ LINE Login โดยอัตโนมัติ                  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### สิ่งที่ LIFF ทำได้

| ความสามารถ | รายละเอียด |
|-----------|-----------|
| **Profile Access** | ดึงชื่อ, รูปโปรไฟล์, User ID ของผู้ใช้ |
| **Send Message** | ส่งข้อความในนาม User ไปยัง Chat |
| **Share Content** | แชร์ข้อความหรือ Flex Message ให้เพื่อน |
| **QR Code Scanner** | สแกน QR Code โดยไม่ต้องขอ Permission พิเศษ |
| **Bluetooth** | เชื่อมต่อ Bluetooth Devices (BLE) |
| **Window Control** | เปิด/ปิด LIFF Window |
| **External Links** | เปิด URL ภายนอก |
| **Access Token** | รับ Token สำหรับ API Calls |

---

## 2. LIFF vs Regular Web

### ความแตกต่างหลัก

```
Regular Web Browser               LIFF in LINE App
─────────────────                 ─────────────────
• ไม่รู้ว่าใครใช้งาน              • รู้ User ID ทันที
• ต้อง Login เอง                  • Auto Login ด้วย LINE Account
• ส่งข้อความไม่ได้               • ส่งข้อความใน Chat ได้
• แชร์ผ่าน Browser Share API      • แชร์ด้วย LINE Native Share
• ไม่มี QR Scanner                • มี QR Scanner ในตัว
• Standard Web APIs                • LINE LIFF SDK APIs เพิ่มเติม
```

### เปรียบเทียบ User Experience

**Regular Web Flow:**
```
1. User เห็นลิงก์ใน LINE Chat
2. คลิกลิงก์ → เปิด Browser ภายนอก
3. ต้อง Login แยก (Google, Facebook, Email)
4. ใช้งาน
5. ถ้าต้องการแชร์ → ออกจากแอป → Copy URL → กลับ LINE
```

**LIFF Flow:**
```
1. User กด Button ใน LINE Chat หรือ Rich Menu
2. LIFF เปิดทันทีใน LINE App
3. Login อัตโนมัติด้วย LINE Account
4. ใช้งาน
5. ถ้าต้องการแชร์ → กด Share → เลือกเพื่อนใน LINE ได้เลย
```

### เมื่อไหร่ควรใช้ LIFF?

```javascript
// ✅ ควรใช้ LIFF เมื่อ:
const shouldUseLiff = [
  "ต้องการข้อมูล User จาก LINE",
  "ต้องการให้ User ส่งข้อความใน Chat",
  "ต้องการ Native LINE UX (Share, QR)",
  "ต้องการ Single Sign-On ด้วย LINE",
  "ต้องการ Form ที่ซับซ้อนกว่า Quick Reply"
];

// ❌ ไม่จำเป็นต้องใช้ LIFF เมื่อ:
const dontNeedLiff = [
  "แสดงข้อมูลแบบ Read-only ธรรมดา",
  "Bot Reply เพียงอย่างเดียวเพียงพอ",
  "ต้องการ SEO (LIFF ไม่ Support)",
  "ผู้ใช้ที่ไม่มี LINE"
];
```

---

## 3. ฟีเจอร์หลักของ LIFF

### 3.1 LIFF SDK APIs ที่สำคัญ

```javascript
// กลุ่ม Initialization
liff.init()              // เริ่มต้น LIFF SDK
liff.ready              // Promise ที่ resolve เมื่อ init สำเร็จ

// กลุ่ม Authentication
liff.isLoggedIn()        // ตรวจสอบว่า Login แล้วหรือไม่
liff.login()             // เริ่มกระบวนการ Login
liff.logout()            // Logout
liff.getProfile()        // ดึงข้อมูล Profile
liff.getAccessToken()    // รับ Access Token
liff.getIDToken()        // รับ ID Token (JWT)

// กลุ่ม Context
liff.getContext()        // ข้อมูล Chat Context
liff.getOS()             // ระบบปฏิบัติการ (ios/android/web)
liff.getLanguage()       // ภาษาที่ใช้
liff.getVersion()        // LIFF SDK Version
liff.isInClient()        // อยู่ใน LINE App หรือไม่

// กลุ่ม Messaging
liff.sendMessages()      // ส่งข้อความใน Current Chat
liff.shareTargetPicker() // แชร์ไปยัง Chat อื่น

// กลุ่ม Window
liff.openWindow()        // เปิด URL ใหม่
liff.closeWindow()       // ปิด LIFF Window

// กลุ่ม QR Code
liff.scanCode()          // เปิด QR Scanner (เฉพาะ LINE App)
liff.scanCodeV2()        // QR Scanner Version ใหม่

// กลุ่ม Bluetooth
liff.bluetooth.requestDevice()  // ขอเชื่อมต่อ BLE Device
```

### 3.2 LIFF Context Types

LIFF สามารถเปิดได้จากหลาย Context:

```javascript
const contextTypes = {
  "utou":     "การสนทนาแบบ 1-on-1 กับ Official Account",
  "room":     "การสนทนาในห้อง (ปัจจุบัน deprecated)",
  "group":    "การสนทนาในกลุ่ม",
  "square":   "LINE SQUARE (Community)",
  "squareThread": "Thread ใน LINE SQUARE",
  "external": "เปิดจาก Browser ภายนอก LINE App",
  "none":     "ไม่มี Context (เปิดโดยตรง)"
};
```

---

## 4. LIFF URL Types

LIFF รองรับ URL รูปแบบต่างๆ:

### 4.1 รูปแบบ URL

```
https://liff.line.me/{liffId}
https://liff.line.me/{liffId}/path/to/page
https://liff.line.me/{liffId}?param=value
https://liff.line.me/{liffId}/path?param=value#hash
```

### 4.2 LIFF URL vs Custom URL

```javascript
// LIFF URL มาตรฐาน
const liffUrl = "https://liff.line.me/1234567890-AbcdEfgh";

// LIFF URL พร้อม Path (Deep Linking)
const liffUrlWithPath = "https://liff.line.me/1234567890-AbcdEfgh/product/123";

// LIFF URL พร้อม Query String
const liffUrlWithQuery = "https://liff.line.me/1234567890-AbcdEfgh?order=456&from=chatbot";

// การดึง Query String ใน LIFF App
const urlParams = new URLSearchParams(window.location.search);
const orderId = urlParams.get('order'); // "456"
const from = urlParams.get('from');     // "chatbot"
```

### 4.3 การ Handle Deep Links

```html
<!DOCTYPE html>
<html>
<head>
    <title>LIFF Deep Link Handler</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <script charset="utf-8" src="https://static.line-scdn.net/liff/edge/2/sdk.js"></script>
</head>
<body>
<script>
    // อ่าน Path และ Query String
    function parseLiffUrl() {
        const currentPath = window.location.pathname;
        const queryString = window.location.search;
        const params = new URLSearchParams(queryString);
        
        // ตัวอย่าง: /product/123?color=red
        const pathParts = currentPath.split('/').filter(Boolean);
        
        return {
            path: currentPath,
            segments: pathParts,
            params: Object.fromEntries(params.entries()),
            hash: window.location.hash
        };
    }
    
    liff.init({ liffId: 'YOUR_LIFF_ID' })
        .then(() => {
            const urlInfo = parseLiffUrl();
            console.log('URL Info:', urlInfo);
            
            // Routing based on path
            if (urlInfo.segments[0] === 'product') {
                showProduct(urlInfo.segments[1]);
            } else if (urlInfo.segments[0] === 'order') {
                showOrder(urlInfo.segments[1]);
            } else {
                showHome();
            }
        });
    
    function showProduct(productId) {
        document.body.innerHTML = `<h1>Product: ${productId}</h1>`;
    }
    
    function showOrder(orderId) {
        document.body.innerHTML = `<h1>Order: ${orderId}</h1>`;
    }
    
    function showHome() {
        document.body.innerHTML = `<h1>Home Page</h1>`;
    }
</script>
</body>
</html>
```

---

## 5. ขนาด LIFF และการใช้งาน

### 5.1 ประเภทขนาด LIFF

LINE กำหนดขนาดของ LIFF Window ไว้ 3 แบบ:

```
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   FULL Screen    │  │   TALL Screen    │  │ COMPACT Screen   │
│                  │  │                  │  │                  │
│  ██████████████  │  │  ░░░░░░░░░░░░░░  │  │  ░░░░░░░░░░░░░░  │
│  ██████████████  │  │  ░░░░░░░░░░░░░░  │  │  ████████████████│
│  ██████████████  │  │  ████████████████│  │  ████████████████│
│  ██████████████  │  │  ████████████████│  │  ████████████████│
│  ██████████████  │  │  ████████████████│  │  ████████████████│
│  ██████████████  │  │  ████████████████│  │                  │
│  ██████████████  │  │  ████████████████│  │                  │
│  ██████████████  │  │                  │  │                  │
│  100% height     │  │  75% height      │  │  50% height      │
└──────────────────┘  └──────────────────┘  └──────────────────┘
   ██ = LIFF area         ░░ = Chat area visible
```

### 5.2 รายละเอียดแต่ละขนาด

| ขนาด | ความสูง | เห็น Chat | ใช้งานเมื่อ |
|------|---------|-----------|------------|
| **full** | 100% | ไม่เห็น | Form ซับซ้อน, E-commerce, แอปครบวงจร |
| **tall** | 75% | เห็นบางส่วน | รายการสินค้า, Profile, ข้อมูลหลายรายการ |
| **compact** | 50% | เห็นครึ่งหนึ่ง | Quick Action, ยืนยันรายการ, Mini Form |

### 5.3 การเลือกขนาดที่เหมาะสม

```javascript
// Use Case Analysis

// ✅ Full Screen - ใช้สำหรับ:
const fullScreenUseCases = [
    "แบบฟอร์มสมัครสมาชิก",
    "หน้าชำระเงิน",
    "แกลเลอรีรูปภาพ",
    "Mini Web App ครบวงจร",
    "Game Interface",
    "Dashboard"
];

// ✅ Tall Screen - ใช้สำหรับ:
const tallScreenUseCases = [
    "รายการสินค้าในตะกร้า",
    "ข้อมูลโปรไฟล์ผู้ใช้",
    "รายการคำสั่งซื้อ",
    "เมนูอาหาร",
    "ตารางเวลานัด"
];

// ✅ Compact Screen - ใช้สำหรับ:
const compactUseCases = [
    "เลือกเวลา/วันที่",
    "ยืนยันการสั่งซื้อ",
    "Rating Form",
    "คูปองส่วนลด",
    "Quick Poll/Survey"
];
```

### 5.4 CSS Tips สำหรับแต่ละขนาด

```css
/* CSS สำหรับ LIFF Full Screen */
.liff-full {
    height: 100vh;
    overflow-y: auto;
    /* ใช้ scroll ได้เต็มที่ */
}

/* CSS สำหรับ LIFF Tall/Compact */
.liff-modal {
    height: 100%;
    overflow-y: auto;
    /* Content ไม่ควรสูงเกิน window */
}

/* Safe Area สำหรับ iPhone Notch */
.liff-container {
    padding-bottom: env(safe-area-inset-bottom);
    padding-top: env(safe-area-inset-top);
}

/* ปรับ Footer ให้ติดล่างเสมอ */
.liff-footer {
    position: fixed;
    bottom: 0;
    left: 0;
    right: 0;
    padding-bottom: env(safe-area-inset-bottom);
    background: white;
    z-index: 100;
}
```

---

## 6. การสร้าง LIFF App ใน Developer Console

### 6.1 ขั้นตอนการสร้าง

**Step 1: เข้า LINE Developers Console**
```
1. ไปที่ https://developers.line.biz/console/
2. Login ด้วย LINE Account
3. เลือก Provider ที่ต้องการ
4. เลือก Channel ที่เป็น LINE Login หรือ Messaging API
```

**Step 2: เพิ่ม LIFF App**
```
1. คลิกที่ Tab "LIFF"
2. คลิก "Add"
3. กรอกข้อมูล:
   - LIFF app name: ชื่อแอป (แสดงให้ผู้ใช้เห็น)
   - Size: full / tall / compact
   - Endpoint URL: URL ของ Web App (ต้องเป็น HTTPS)
   - Scope: openid, profile, chat_message.write (เลือกตามต้องการ)
   - Bot link feature: On/Off (ถ้าต้องการ add Official Account เป็น Friend ด้วย)
4. คลิก "Add"
```

**Step 3: รับ LIFF ID**
```
หลังสร้างแล้วจะได้ LIFF ID ในรูปแบบ:
1234567890-AbcdEfgh

LIFF URL จะเป็น:
https://liff.line.me/1234567890-AbcdEfgh
```

### 6.2 Scope ที่ควรเลือก

```javascript
const liffScopes = {
    "openid": {
        required: true,
        description: "จำเป็นสำหรับ Authentication พื้นฐาน",
        provides: ["user_id", "id_token"]
    },
    "profile": {
        recommended: true,
        description: "เข้าถึงข้อมูล Profile",
        provides: ["displayName", "pictureUrl", "statusMessage"]
    },
    "chat_message.write": {
        optional: true,
        description: "ส่งข้อความใน Chat ได้",
        provides: ["liff.sendMessages()"]
    },
    "email": {
        optional: true,
        description: "เข้าถึงอีเมล (ต้องขอ approval จาก LINE)",
        provides: ["email in ID Token"]
    }
};
```

### 6.3 Bot Link Feature

```
Bot Link Feature คือการที่เมื่อผู้ใช้เปิด LIFF App ครั้งแรก
จะมีหน้า Consent ให้ Add Official Account เป็น Friend ด้วย

Options:
- Off: ไม่แสดงหน้า Add Friend
- On (Aggressive): แสดงทุกครั้งที่เปิด LIFF
- On (Normal): แสดงเฉพาะครั้งแรก หรือเมื่อยังไม่ได้ Add Friend
```

---

## 7. LIFF ID และ Endpoint URL

### 7.1 โครงสร้าง LIFF ID

```
1234567890-AbcdEfgh
│          │
│          └── Random String (8 chars)
└── Channel ID
```

### 7.2 การตั้งค่า Endpoint URL สำหรับ Development

```bash
# ใช้ ngrok สำหรับ Local Development
ngrok http 3000

# จะได้ URL ประมาณ:
# https://abc123.ngrok.io

# ตั้งค่าใน LIFF Console:
# Endpoint URL: https://abc123.ngrok.io
```

### 7.3 Endpoint URL Requirements

```
✅ ต้องมี:
- HTTPS เท่านั้น (HTTP ไม่ได้)
- Domain ที่ Valid
- Port ไม่ต้องระบุถ้าใช้ 443

❌ ไม่ได้:
- localhost (ใช้ ngrok แทน)
- IP Address โดยตรง
- HTTP URL
- Self-signed certificate (ต้องเป็น SSL จริง)

✅ ตัวอย่าง URL ที่ถูกต้อง:
https://myapp.example.com
https://my-liff-app.vercel.app
https://myapp.netlify.app
https://abc123.ngrok.io
```

### 7.4 Multiple LIFF IDs per App

```javascript
// แอปหนึ่งสามารถมีหลาย LIFF ID ได้
const liffApps = {
    main: {
        id: "1234567890-MainPage",
        size: "full",
        url: "https://myapp.com/"
    },
    product: {
        id: "1234567890-Product",
        size: "tall",
        url: "https://myapp.com/product"
    },
    checkout: {
        id: "1234567890-Checkout",
        size: "full",
        url: "https://myapp.com/checkout"
    },
    rating: {
        id: "1234567890-Rating",
        size: "compact",
        url: "https://myapp.com/rating"
    }
};
```

---

## 8. การติดตั้ง LIFF SDK

### 8.1 วิธีที่ 1: CDN (แนะนำสำหรับ Beginner)

```html
<!-- ใน <head> ของ HTML file -->
<script charset="utf-8" src="https://static.line-scdn.net/liff/edge/2/sdk.js"></script>
```

**ข้อดีข้อเสีย CDN:**
```
✅ ง่าย ไม่ต้องติดตั้ง
✅ ได้ Version ล่าสุดอัตโนมัติ
✅ เหมาะกับ HTML ธรรมดา
❌ ไม่มี TypeScript Support
❌ ต้องใช้ Global Object (window.liff)
```

### 8.2 วิธีที่ 2: npm Package (แนะนำสำหรับ Framework)

```bash
# สำหรับ npm
npm install @line/liff

# สำหรับ yarn
yarn add @line/liff

# สำหรับ pnpm
pnpm add @line/liff
```

```javascript
// ES Modules
import liff from '@line/liff';

// CommonJS
const liff = require('@line/liff');

// TypeScript
import liff from '@line/liff';
// TypeScript Types มาพร้อมกัน
```

### 8.3 ตรวจสอบ Version

```javascript
// ดู Version ที่ใช้อยู่
console.log(liff.getVersion());
// "2.22.0" หรือ Version ล่าสุด

// ตรวจสอบผ่าน npm
npm list @line/liff
```

### 8.4 CDN Versions

```html
<!-- Version ล่าสุด (แนะนำ) -->
<script src="https://static.line-scdn.net/liff/edge/2/sdk.js"></script>

<!-- Version เฉพาะ (stable) -->
<script src="https://static.line-scdn.net/liff/edge/versions/2.22.0/sdk.js"></script>

<!-- Version เก่า (ไม่แนะนำ) -->
<script src="https://d.line-scdn.net/liff/1.0/sdk.js"></script>
```

---

## 9. Hello World LIFF App

### 9.1 LIFF App แบบ HTML ธรรมดา

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, viewport-fit=cover">
    <title>Hello LIFF</title>
    
    <!-- LIFF SDK -->
    <script charset="utf-8" src="https://static.line-scdn.net/liff/edge/2/sdk.js"></script>
    
    <style>
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
        }
        
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 
                         'Helvetica Neue', Arial, sans-serif;
            background: #f0f4f8;
            min-height: 100vh;
            display: flex;
            flex-direction: column;
        }
        
        .loading {
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            gap: 16px;
        }
        
        .spinner {
            width: 48px;
            height: 48px;
            border: 4px solid #e2e8f0;
            border-top-color: #06c755;
            border-radius: 50%;
            animation: spin 0.8s linear infinite;
        }
        
        @keyframes spin {
            to { transform: rotate(360deg); }
        }
        
        .loading-text {
            color: #64748b;
            font-size: 14px;
        }
        
        .error-container {
            display: none;
            padding: 24px;
            text-align: center;
        }
        
        .error-icon {
            font-size: 48px;
            margin-bottom: 16px;
        }
        
        .error-title {
            font-size: 18px;
            font-weight: 600;
            color: #dc2626;
            margin-bottom: 8px;
        }
        
        .error-message {
            font-size: 14px;
            color: #64748b;
        }
        
        .main-container {
            display: none;
            padding: 24px 16px;
            max-width: 480px;
            margin: 0 auto;
            width: 100%;
        }
        
        .header {
            background: linear-gradient(135deg, #06c755 0%, #04a040 100%);
            color: white;
            padding: 32px 24px;
            border-radius: 16px;
            text-align: center;
            margin-bottom: 24px;
            box-shadow: 0 4px 6px rgba(6, 199, 85, 0.2);
        }
        
        .header h1 {
            font-size: 24px;
            font-weight: 700;
            margin-bottom: 8px;
        }
        
        .header p {
            font-size: 14px;
            opacity: 0.9;
        }
        
        .profile-card {
            background: white;
            border-radius: 16px;
            padding: 24px;
            display: flex;
            align-items: center;
            gap: 16px;
            margin-bottom: 16px;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        }
        
        .profile-avatar {
            width: 64px;
            height: 64px;
            border-radius: 50%;
            border: 3px solid #06c755;
            object-fit: cover;
        }
        
        .profile-avatar-placeholder {
            width: 64px;
            height: 64px;
            border-radius: 50%;
            background: #e2e8f0;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 24px;
        }
        
        .profile-info h2 {
            font-size: 18px;
            font-weight: 600;
            color: #1a202c;
            margin-bottom: 4px;
        }
        
        .profile-info p {
            font-size: 12px;
            color: #64748b;
            word-break: break-all;
        }
        
        .info-grid {
            display: grid;
            gap: 12px;
            margin-bottom: 24px;
        }
        
        .info-item {
            background: white;
            border-radius: 12px;
            padding: 16px;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        }
        
        .info-label {
            font-size: 11px;
            font-weight: 600;
            text-transform: uppercase;
            letter-spacing: 0.05em;
            color: #94a3b8;
            margin-bottom: 6px;
        }
        
        .info-value {
            font-size: 14px;
            color: #1a202c;
            font-family: 'Courier New', monospace;
            word-break: break-all;
        }
        
        .info-badge {
            display: inline-flex;
            align-items: center;
            gap: 4px;
            padding: 4px 10px;
            border-radius: 20px;
            font-size: 12px;
            font-weight: 600;
        }
        
        .badge-green {
            background: #dcfce7;
            color: #166534;
        }
        
        .badge-blue {
            background: #dbeafe;
            color: #1d4ed8;
        }
        
        .badge-orange {
            background: #fed7aa;
            color: #c2410c;
        }
        
        .action-buttons {
            display: flex;
            flex-direction: column;
            gap: 12px;
        }
        
        .btn {
            padding: 14px 24px;
            border-radius: 12px;
            font-size: 15px;
            font-weight: 600;
            border: none;
            cursor: pointer;
            transition: all 0.2s;
            display: flex;
            align-items: center;
            justify-content: center;
            gap: 8px;
        }
        
        .btn:active {
            transform: scale(0.98);
        }
        
        .btn-primary {
            background: #06c755;
            color: white;
            box-shadow: 0 2px 4px rgba(6, 199, 85, 0.3);
        }
        
        .btn-primary:hover {
            background: #04a040;
        }
        
        .btn-secondary {
            background: white;
            color: #06c755;
            border: 2px solid #06c755;
        }
        
        .btn-danger {
            background: white;
            color: #dc2626;
            border: 2px solid #fca5a5;
        }
        
        .btn:disabled {
            opacity: 0.5;
            cursor: not-allowed;
        }
        
        .success-toast {
            position: fixed;
            bottom: 24px;
            left: 50%;
            transform: translateX(-50%) translateY(100px);
            background: #1a202c;
            color: white;
            padding: 12px 20px;
            border-radius: 8px;
            font-size: 14px;
            transition: transform 0.3s;
            z-index: 1000;
            white-space: nowrap;
        }
        
        .success-toast.show {
            transform: translateX(-50%) translateY(0);
        }
    </style>
</head>
<body>

    <!-- Loading State -->
    <div class="loading" id="loading">
        <div class="spinner"></div>
        <p class="loading-text">กำลังโหลด LIFF...</p>
    </div>
    
    <!-- Error State -->
    <div class="error-container" id="errorContainer">
        <div class="error-icon">⚠️</div>
        <h2 class="error-title">เกิดข้อผิดพลาด</h2>
        <p class="error-message" id="errorMessage"></p>
    </div>
    
    <!-- Main Content -->
    <div class="main-container" id="mainContainer">
        <!-- Header -->
        <div class="header">
            <h1>🎉 Hello LIFF!</h1>
            <p>Welcome to LINE Front-end Framework</p>
        </div>
        
        <!-- Profile Card -->
        <div class="profile-card" id="profileCard" style="display:none">
            <img id="profileAvatar" class="profile-avatar" src="" alt="Profile">
            <div class="profile-info">
                <h2 id="profileName">-</h2>
                <p id="profileUserId">-</p>
            </div>
        </div>
        
        <!-- Not Logged In State -->
        <div id="notLoggedIn" style="display:none; margin-bottom:16px;">
            <div class="info-item" style="text-align:center;">
                <div style="font-size:48px; margin-bottom:12px;">👤</div>
                <p style="color:#64748b; margin-bottom:16px;">กรุณา Login ด้วย LINE Account</p>
                <button class="btn btn-primary" onclick="handleLogin()">
                    Login ด้วย LINE
                </button>
            </div>
        </div>
        
        <!-- LIFF Info Grid -->
        <div class="info-grid">
            <div class="info-item">
                <div class="info-label">LIFF Environment</div>
                <div class="info-value" id="liffEnvironment">-</div>
            </div>
            <div class="info-item">
                <div class="info-label">Operating System</div>
                <div class="info-value" id="liffOS">-</div>
            </div>
            <div class="info-item">
                <div class="info-label">Language</div>
                <div class="info-value" id="liffLanguage">-</div>
            </div>
            <div class="info-item">
                <div class="info-label">LIFF SDK Version</div>
                <div class="info-value" id="liffVersion">-</div>
            </div>
            <div class="info-item">
                <div class="info-label">Chat Context</div>
                <div class="info-value" id="liffContext">-</div>
            </div>
            <div class="info-item">
                <div class="info-label">Login Status</div>
                <div class="info-value" id="loginStatus">-</div>
            </div>
        </div>
        
        <!-- Action Buttons -->
        <div class="action-buttons" id="actionButtons" style="display:none">
            <button class="btn btn-primary" onclick="sendHelloMessage()">
                💬 ส่งข้อความ "Hello!"
            </button>
            <button class="btn btn-secondary" onclick="copyUserId()">
                📋 Copy User ID
            </button>
            <button class="btn btn-danger" onclick="handleLogout()">
                🚪 Logout
            </button>
        </div>
    </div>
    
    <!-- Toast Notification -->
    <div class="success-toast" id="toast"></div>

    <script>
        const LIFF_ID = 'YOUR_LIFF_ID'; // แทนด้วย LIFF ID จริง
        
        // แสดง Toast Message
        function showToast(message) {
            const toast = document.getElementById('toast');
            toast.textContent = message;
            toast.classList.add('show');
            setTimeout(() => toast.classList.remove('show'), 3000);
        }
        
        // แสดง Error
        function showError(message) {
            document.getElementById('loading').style.display = 'none';
            document.getElementById('errorContainer').style.display = 'block';
            document.getElementById('errorMessage').textContent = message;
        }
        
        // แสดง Main Content
        function showMain() {
            document.getElementById('loading').style.display = 'none';
            document.getElementById('mainContainer').style.display = 'block';
        }
        
        // อัพเดต LIFF Info
        function updateLiffInfo() {
            // Environment
            const isInClient = liff.isInClient();
            const envEl = document.getElementById('liffEnvironment');
            if (isInClient) {
                envEl.innerHTML = '<span class="info-badge badge-green">✓ LINE App</span>';
            } else {
                envEl.innerHTML = '<span class="info-badge badge-orange">⚠ External Browser</span>';
            }
            
            // OS
            const os = liff.getOS();
            const osEl = document.getElementById('liffOS');
            const osIcon = os === 'ios' ? '🍎' : os === 'android' ? '🤖' : '💻';
            osEl.innerHTML = `${osIcon} ${os || 'unknown'}`;
            
            // Language
            document.getElementById('liffLanguage').textContent = liff.getLanguage() || '-';
            
            // Version
            document.getElementById('liffVersion').textContent = liff.getVersion() || '-';
            
            // Context
            const context = liff.getContext();
            const contextEl = document.getElementById('liffContext');
            if (context) {
                contextEl.innerHTML = `<span class="info-badge badge-blue">${context.type}</span>`;
            } else {
                contextEl.textContent = 'No context';
            }
        }
        
        // อัพเดต Login Status
        async function updateLoginStatus() {
            const isLoggedIn = liff.isLoggedIn();
            const statusEl = document.getElementById('loginStatus');
            
            if (isLoggedIn) {
                statusEl.innerHTML = '<span class="info-badge badge-green">✓ Logged In</span>';
                
                // แสดง Profile
                try {
                    const profile = await liff.getProfile();
                    document.getElementById('profileAvatar').src = profile.pictureUrl || '';
                    document.getElementById('profileName').textContent = profile.displayName;
                    document.getElementById('profileUserId').textContent = `ID: ${profile.userId}`;
                    document.getElementById('profileCard').style.display = 'flex';
                    document.getElementById('actionButtons').style.display = 'flex';
                    document.getElementById('notLoggedIn').style.display = 'none';
                    
                    // เก็บ User ID สำหรับ Copy
                    window._userId = profile.userId;
                } catch (err) {
                    console.error('Get profile error:', err);
                }
            } else {
                statusEl.innerHTML = '<span class="info-badge badge-orange">✗ Not Logged In</span>';
                document.getElementById('profileCard').style.display = 'none';
                document.getElementById('actionButtons').style.display = 'none';
                document.getElementById('notLoggedIn').style.display = 'block';
            }
        }
        
        // Login Handler
        function handleLogin() {
            liff.login({
                redirectUri: window.location.href
            });
        }
        
        // Logout Handler
        async function handleLogout() {
            if (confirm('ต้องการ Logout หรือไม่?')) {
                liff.logout();
                window.location.reload();
            }
        }
        
        // ส่งข้อความ Hello
        async function sendHelloMessage() {
            if (!liff.isInClient()) {
                showToast('⚠️ ฟีเจอร์นี้ใช้ได้เฉพาะใน LINE App');
                return;
            }
            
            try {
                await liff.sendMessages([
                    {
                        type: 'text',
                        text: 'Hello from LIFF! 👋\nสวัสดีจาก LINE Front-end Framework'
                    }
                ]);
                showToast('✅ ส่งข้อความสำเร็จ!');
                setTimeout(() => liff.closeWindow(), 1500);
            } catch (error) {
                console.error('Send message error:', error);
                showToast('❌ ไม่สามารถส่งข้อความได้');
            }
        }
        
        // Copy User ID
        function copyUserId() {
            if (!window._userId) return;
            
            if (navigator.clipboard) {
                navigator.clipboard.writeText(window._userId)
                    .then(() => showToast('✅ Copy User ID แล้ว!'))
                    .catch(() => showToast('❌ ไม่สามารถ Copy ได้'));
            } else {
                // Fallback สำหรับ Browser เก่า
                const el = document.createElement('textarea');
                el.value = window._userId;
                document.body.appendChild(el);
                el.select();
                document.execCommand('copy');
                document.body.removeChild(el);
                showToast('✅ Copy User ID แล้ว!');
            }
        }
        
        // Initialize LIFF
        async function initLiff() {
            try {
                await liff.init({ liffId: LIFF_ID });
                
                // แสดง Main Content
                showMain();
                
                // อัพเดต LIFF Info
                updateLiffInfo();
                
                // อัพเดต Login Status
                await updateLoginStatus();
                
            } catch (error) {
                console.error('LIFF init error:', error);
                showError(`LIFF Initialization Failed: ${error.message || error}`);
            }
        }
        
        // Start
        initLiff();
    </script>
</body>
</html>
```

### 9.2 Node.js Backend สำหรับ LIFF

```javascript
// server.js - Express server เสิร์ฟ LIFF files
const express = require('express');
const path = require('path');
const app = express();

// Serve static files
app.use(express.static(path.join(__dirname, 'public')));

// Handle all routes (SPA)
app.get('*', (req, res) => {
    res.sendFile(path.join(__dirname, 'public', 'index.html'));
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`LIFF App running on port ${PORT}`);
    console.log(`Local: http://localhost:${PORT}`);
});
```

```json
// package.json
{
  "name": "hello-liff",
  "version": "1.0.0",
  "description": "Hello World LIFF App",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.2"
  },
  "devDependencies": {
    "nodemon": "^3.0.0"
  }
}
```

### 9.3 โครงสร้างโปรเจกต์

```
hello-liff/
├── server.js
├── package.json
├── .env
└── public/
    ├── index.html     ← LIFF App หลัก
    ├── css/
    │   └── style.css
    └── js/
        └── app.js
```

---

## 10. เปิด LIFF จาก Rich Menu

### 10.1 การตั้งค่า Rich Menu ให้เปิด LIFF

```javascript
// สร้าง Rich Menu ที่มี LIFF Button ด้วย Node.js
const line = require('@line/bot-sdk');

const client = new line.messagingApi.MessagingApiClient({
    channelAccessToken: process.env.CHANNEL_ACCESS_TOKEN
});

async function createRichMenuWithLiff() {
    const richMenu = {
        size: {
            width: 2500,
            height: 843
        },
        selected: true,
        name: "Main Menu with LIFF",
        chatBarText: "เมนู",
        areas: [
            // พื้นที่ซ้าย - เปิด LIFF
            {
                bounds: { x: 0, y: 0, width: 833, height: 843 },
                action: {
                    type: "uri",
                    uri: "https://liff.line.me/1234567890-AbcdEfgh",
                    label: "เปิดแอป"
                }
            },
            // พื้นที่กลาง - เปิด LIFF พร้อม Parameter
            {
                bounds: { x: 833, y: 0, width: 834, height: 843 },
                action: {
                    type: "uri",
                    uri: "https://liff.line.me/1234567890-AbcdEfgh?page=menu",
                    label: "เมนู"
                }
            },
            // พื้นที่ขวา - เปิด LIFF หน้าอื่น
            {
                bounds: { x: 1667, y: 0, width: 833, height: 843 },
                action: {
                    type: "uri",
                    uri: "https://liff.line.me/1234567890-Profile",
                    label: "โปรไฟล์"
                }
            }
        ]
    };
    
    try {
        const result = await client.createRichMenu(richMenu);
        console.log('Rich Menu Created:', result.richMenuId);
        
        // Upload Image
        const fs = require('fs');
        const imageBuffer = fs.readFileSync('./richmenu-image.png');
        await client.setRichMenuImage(result.richMenuId, imageBuffer);
        
        // Set as Default
        await client.setDefaultRichMenu(result.richMenuId);
        
        console.log('Rich Menu with LIFF set successfully!');
        return result.richMenuId;
    } catch (error) {
        console.error('Error:', error);
    }
}

createRichMenuWithLiff();
```

### 10.2 Rich Menu JSON โดยตรง

```json
{
  "size": {
    "width": 2500,
    "height": 843
  },
  "selected": true,
  "name": "LIFF Menu",
  "chatBarText": "เมนูหลัก",
  "areas": [
    {
      "bounds": { "x": 0, "y": 0, "width": 1250, "height": 843 },
      "action": {
        "type": "uri",
        "uri": "https://liff.line.me/YOUR_LIFF_ID",
        "label": "เปิดแอป LIFF"
      }
    },
    {
      "bounds": { "x": 1250, "y": 0, "width": 1250, "height": 843 },
      "action": {
        "type": "message",
        "text": "ช่วยเหลือ",
        "label": "ช่วยเหลือ"
      }
    }
  ]
}
```

### 10.3 LIFF รับ Parameter จาก Rich Menu

```html
<!-- LIFF App ที่รับ Parameter จาก Rich Menu URL -->
<script>
    liff.init({ liffId: 'YOUR_LIFF_ID' })
        .then(() => {
            // ดึง Query Parameters ที่ส่งมาจาก Rich Menu
            const params = new URLSearchParams(window.location.search);
            const page = params.get('page');
            const from = params.get('from');
            
            console.log('Opened from:', from); // 'richmenu'
            console.log('Page requested:', page);
            
            // Routing ตาม page parameter
            switch (page) {
                case 'menu':
                    showMenuPage();
                    break;
                case 'order':
                    showOrderPage();
                    break;
                case 'profile':
                    showProfilePage();
                    break;
                default:
                    showHomePage();
            }
        });
</script>
```

---

## 11. เปิด LIFF จาก Flex Message

### 11.1 Flex Message พร้อม LIFF Button

```javascript
// ส่ง Flex Message ที่มีปุ่มเปิด LIFF
const liffFlexMessage = {
    type: 'flex',
    altText: 'สินค้าแนะนำ',
    contents: {
        type: 'bubble',
        size: 'mega',
        header: {
            type: 'box',
            layout: 'vertical',
            contents: [
                {
                    type: 'text',
                    text: '🛍️ สินค้าแนะนำ',
                    weight: 'bold',
                    size: 'xl',
                    color: '#ffffff'
                }
            ],
            backgroundColor: '#06c755',
            paddingAll: '20px'
        },
        hero: {
            type: 'image',
            url: 'https://example.com/product.jpg',
            size: 'full',
            aspectRatio: '20:13',
            aspectMode: 'cover'
        },
        body: {
            type: 'box',
            layout: 'vertical',
            contents: [
                {
                    type: 'text',
                    text: 'สินค้า Premium Collection',
                    weight: 'bold',
                    size: 'lg',
                    wrap: true
                },
                {
                    type: 'box',
                    layout: 'baseline',
                    margin: 'md',
                    contents: [
                        {
                            type: 'text',
                            text: '฿1,299',
                            size: 'xl',
                            color: '#06c755',
                            weight: 'bold',
                            flex: 0
                        },
                        {
                            type: 'text',
                            text: '฿2,500',
                            size: 'sm',
                            color: '#aaaaaa',
                            decoration: 'line-through',
                            margin: 'sm'
                        }
                    ]
                },
                {
                    type: 'text',
                    text: 'ลดราคาพิเศษ 48% เฉพาะสัปดาห์นี้',
                    size: 'sm',
                    color: '#999999',
                    margin: 'md',
                    wrap: true
                }
            ]
        },
        footer: {
            type: 'box',
            layout: 'vertical',
            spacing: 'sm',
            contents: [
                {
                    type: 'button',
                    style: 'primary',
                    action: {
                        type: 'uri',
                        label: '🛒 ดูรายละเอียดและสั่งซื้อ',
                        uri: 'https://liff.line.me/YOUR_LIFF_ID/product/PRD001'
                    },
                    color: '#06c755',
                    height: 'md'
                },
                {
                    type: 'button',
                    style: 'secondary',
                    action: {
                        type: 'uri',
                        label: '❤️ เพิ่มในรายการโปรด',
                        uri: 'https://liff.line.me/YOUR_LIFF_ID/wishlist/add/PRD001'
                    },
                    height: 'md'
                }
            ]
        }
    }
};

// ส่งข้อความ
client.replyMessage({
    replyToken: event.replyToken,
    messages: [liffFlexMessage]
});
```

### 11.2 Carousel Flex Message พร้อม LIFF

```javascript
// Carousel ที่แต่ละ Card เปิด LIFF ต่างกัน
function createProductCarousel(products) {
    const bubbles = products.map(product => ({
        type: 'bubble',
        hero: {
            type: 'image',
            url: product.imageUrl,
            size: 'full',
            aspectRatio: '1:1',
            aspectMode: 'cover',
            action: {
                type: 'uri',
                uri: `https://liff.line.me/YOUR_LIFF_ID/product/${product.id}`
            }
        },
        body: {
            type: 'box',
            layout: 'vertical',
            contents: [
                {
                    type: 'text',
                    text: product.name,
                    weight: 'bold',
                    size: 'md',
                    wrap: true,
                    maxLines: 2
                },
                {
                    type: 'text',
                    text: `฿${product.price.toLocaleString()}`,
                    color: '#06c755',
                    weight: 'bold',
                    size: 'lg',
                    margin: 'sm'
                }
            ]
        },
        footer: {
            type: 'box',
            layout: 'vertical',
            contents: [
                {
                    type: 'button',
                    action: {
                        type: 'uri',
                        label: 'สั่งซื้อเลย',
                        uri: `https://liff.line.me/YOUR_LIFF_ID/order?product=${product.id}`
                    },
                    style: 'primary',
                    color: '#06c755'
                }
            ]
        }
    }));
    
    return {
        type: 'flex',
        altText: 'รายการสินค้า',
        contents: {
            type: 'carousel',
            contents: bubbles
        }
    };
}

// ตัวอย่างการใช้งาน
const products = [
    { id: 'P001', name: 'สินค้า A', price: 599, imageUrl: 'https://...' },
    { id: 'P002', name: 'สินค้า B', price: 899, imageUrl: 'https://...' },
    { id: 'P003', name: 'สินค้า C', price: 1299, imageUrl: 'https://...' }
];

const carouselMessage = createProductCarousel(products);
```

### 11.3 LIFF App ที่รับ Product ID

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Product Detail</title>
    <script charset="utf-8" src="https://static.line-scdn.net/liff/edge/2/sdk.js"></script>
    <style>
        body {
            font-family: -apple-system, sans-serif;
            margin: 0;
            background: #f5f5f5;
        }
        
        .product-container {
            max-width: 480px;
            margin: 0 auto;
            background: white;
            min-height: 100vh;
        }
        
        .product-image {
            width: 100%;
            aspect-ratio: 1;
            object-fit: cover;
        }
        
        .product-info {
            padding: 20px;
        }
        
        .product-name {
            font-size: 20px;
            font-weight: 700;
            margin-bottom: 8px;
        }
        
        .product-price {
            font-size: 24px;
            color: #06c755;
            font-weight: 700;
            margin-bottom: 16px;
        }
        
        .product-desc {
            color: #666;
            line-height: 1.6;
            margin-bottom: 24px;
        }
        
        .add-to-cart-btn {
            width: 100%;
            padding: 16px;
            background: #06c755;
            color: white;
            font-size: 16px;
            font-weight: 700;
            border: none;
            border-radius: 8px;
            cursor: pointer;
        }
        
        .add-to-cart-btn:active {
            background: #05b34c;
        }
    </style>
</head>
<body>
    <div class="product-container" id="productContainer">
        <div id="loadingState" style="padding:40px;text-align:center;">
            กำลังโหลด...
        </div>
    </div>
    
    <script>
        const LIFF_ID = 'YOUR_LIFF_ID';
        
        // Mock API - ในชีวิตจริงเรียก Backend
        const mockProducts = {
            'PRD001': {
                name: 'สินค้า Premium A',
                price: 1299,
                image: 'https://via.placeholder.com/480x480/06c755/ffffff?text=Product+A',
                description: 'สินค้าคุณภาพสูง ทำจากวัสดุพรีเมียม รับประกัน 2 ปี'
            },
            'P001': {
                name: 'สินค้า A',
                price: 599,
                image: 'https://via.placeholder.com/480x480/3b82f6/ffffff?text=Product+B',
                description: 'สินค้าคุณภาพดี ราคาเหมาะสม'
            }
        };
        
        async function loadProduct(productId) {
            // จำลองการโหลดข้อมูล
            const product = mockProducts[productId] || {
                name: 'Product Not Found',
                price: 0,
                image: 'https://via.placeholder.com/480x480/cccccc/ffffff?text=Not+Found',
                description: 'ไม่พบสินค้า'
            };
            
            const container = document.getElementById('productContainer');
            container.innerHTML = `
                <img class="product-image" src="${product.image}" alt="${product.name}">
                <div class="product-info">
                    <h1 class="product-name">${product.name}</h1>
                    <div class="product-price">฿${product.price.toLocaleString()}</div>
                    <p class="product-desc">${product.description}</p>
                    <button class="add-to-cart-btn" onclick="addToCart('${productId}')">
                        🛒 เพิ่มลงตะกร้า
                    </button>
                </div>
            `;
        }
        
        async function addToCart(productId) {
            if (!liff.isInClient()) {
                alert('กรุณาเปิดจาก LINE App');
                return;
            }
            
            const profile = await liff.getProfile();
            
            try {
                await liff.sendMessages([{
                    type: 'text',
                    text: `🛒 เพิ่ม ${productId} ลงตะกร้าแล้ว!\nUser: ${profile.displayName}`
                }]);
                
                liff.closeWindow();
            } catch (error) {
                console.error(error);
                alert('เกิดข้อผิดพลาด กรุณาลองใหม่');
            }
        }
        
        // Init
        liff.init({ liffId: LIFF_ID })
            .then(async () => {
                // ดึง Product ID จาก Path หรือ Query String
                const path = window.location.pathname;
                const params = new URLSearchParams(window.location.search);
                
                let productId = params.get('product');
                
                // รองรับ Path format: /product/PRD001
                if (!productId) {
                    const pathMatch = path.match(/\/product\/([^\/\?]+)/);
                    if (pathMatch) productId = pathMatch[1];
                }
                
                if (productId) {
                    await loadProduct(productId);
                } else {
                    document.getElementById('loadingState').textContent = 
                        'ไม่พบข้อมูลสินค้า';
                }
            })
            .catch(error => {
                console.error('LIFF init error:', error);
                document.getElementById('loadingState').textContent = 
                    `Error: ${error.message}`;
            });
    </script>
</body>
</html>
```

---

## 12. Best Practices

### 12.1 การจัดการ Loading State

```javascript
// Pattern ที่ดี: แสดง Loading จนกว่า LIFF จะ Ready
class LiffApp {
    constructor(liffId) {
        this.liffId = liffId;
        this.initialized = false;
    }
    
    async init() {
        this.showLoading();
        
        try {
            await liff.init({ liffId: this.liffId });
            this.initialized = true;
            this.hideLoading();
            await this.onReady();
        } catch (error) {
            this.hideLoading();
            this.showError(error);
        }
    }
    
    showLoading() {
        document.getElementById('loading').style.display = 'flex';
        document.getElementById('content').style.display = 'none';
    }
    
    hideLoading() {
        document.getElementById('loading').style.display = 'none';
        document.getElementById('content').style.display = 'block';
    }
    
    showError(error) {
        document.getElementById('error').style.display = 'block';
        document.getElementById('error').textContent = error.message;
    }
    
    async onReady() {
        // Override ใน Subclass
    }
}
```

### 12.2 การรองรับ External Browser

```javascript
// บางครั้งผู้ใช้เปิด LIFF URL ใน Browser ภายนอก
function handleExternalBrowser() {
    if (!liff.isInClient()) {
        // แสดงข้อความแนะนำ
        const isAndroid = /Android/i.test(navigator.userAgent);
        const isIOS = /iPhone|iPad/i.test(navigator.userAgent);
        
        if (isAndroid || isIOS) {
            // มือถือ - แนะนำให้เปิดใน LINE
            showOpenInLineButton();
        } else {
            // Desktop - บางฟีเจอร์ไม่ทำงาน
            showDesktopWarning();
        }
    }
}

function showOpenInLineButton() {
    const banner = document.createElement('div');
    banner.style.cssText = `
        position: fixed;
        bottom: 0;
        left: 0;
        right: 0;
        background: #06c755;
        color: white;
        padding: 16px;
        text-align: center;
        z-index: 1000;
    `;
    banner.innerHTML = `
        <p style="margin:0 0 8px">เปิดใน LINE เพื่อใช้งานได้ครบฟีเจอร์</p>
        <a href="line://app/YOUR_LIFF_ID" 
           style="color:white;font-weight:bold;font-size:16px;">
           เปิดใน LINE App →
        </a>
    `;
    document.body.appendChild(banner);
}
```

### 12.3 Security Considerations

```javascript
// ✅ Best Practices ด้าน Security

// 1. ตรวจสอบ Token บน Server เสมอ ไม่เชื่อ Client
async function secureOperation() {
    const idToken = liff.getIDToken();
    
    // ส่ง token ไป verify บน server
    const response = await fetch('/api/verify', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ idToken })
    });
    
    if (!response.ok) {
        throw new Error('Token verification failed');
    }
    
    const { userId } = await response.json();
    return userId;
}

// 2. ไม่เก็บ Access Token ใน localStorage
// ❌ อย่าทำแบบนี้:
// localStorage.setItem('accessToken', liff.getAccessToken());

// ✅ ทำแบบนี้แทน: ส่ง token ไป backend ทันที
async function useAccessToken() {
    const token = liff.getAccessToken();
    // ใช้ทันที ไม่เก็บ
    const userInfo = await fetch('https://api.line.me/v2/profile', {
        headers: { Authorization: `Bearer ${token}` }
    }).then(r => r.json());
    return userInfo;
}

// 3. Validate Input ทั้ง Client และ Server Side
function sanitizeInput(input) {
    return String(input)
        .replace(/[<>&"']/g, char => ({
            '<': '&lt;', '>': '&gt;', '&': '&amp;',
            '"': '&quot;', "'": '&#x27;'
        }[char]));
}
```

### 12.4 Performance Tips

```javascript
// ✅ Performance Best Practices

// 1. Init LIFF เร็วที่สุด - ใส่ Script ใน <head> หรือ defer
// <script src="liff-sdk.js" defer></script>

// 2. Cache Profile ใน Session
let cachedProfile = null;

async function getProfile() {
    if (!cachedProfile) {
        cachedProfile = await liff.getProfile();
    }
    return cachedProfile;
}

// 3. Lazy Load Content
async function loadContentOnDemand(type) {
    switch (type) {
        case 'products':
            const { renderProducts } = await import('./products.js');
            renderProducts();
            break;
        case 'profile':
            const { renderProfile } = await import('./profile.js');
            renderProfile();
            break;
    }
}

// 4. ใช้ Service Worker สำหรับ Offline Support
// public/sw.js
if ('serviceWorker' in navigator) {
    navigator.serviceWorker.register('/sw.js');
}
```

---

## 13. แบบฝึกหัด

### แบบฝึกหัดที่ 1: LIFF Profile Viewer

สร้าง LIFF App ที่:
- แสดงรูปโปรไฟล์, ชื่อ, และ User ID
- มีปุ่ม Copy User ID
- แสดง Device Info (OS, Language)
- ออกแบบ UI สวยงามด้วย CSS

**Hints:**
```javascript
// APIs ที่ต้องใช้:
liff.init()
liff.getProfile()
liff.getOS()
liff.getLanguage()
liff.isLoggedIn()
```

### แบบฝึกหัดที่ 2: Mini Order Form

สร้าง LIFF App (Full Screen) ที่:
- มีฟอร์มรับชื่อ, ที่อยู่, จำนวน
- Validate ข้อมูลก่อน Submit
- เมื่อ Submit ส่งข้อความสรุปใน Chat
- ปิด LIFF หลังส่งเสร็จ

**Template เริ่มต้น:**
```html
<!DOCTYPE html>
<html>
<!-- เขียน HTML Form ที่สวยงาม -->
<!-- ใช้ liff.sendMessages() เมื่อ Submit -->
<!-- ใช้ liff.closeWindow() หลังส่ง -->
</html>
```

### แบบฝึกหัดที่ 3: LIFF + Rich Menu Integration

1. สร้าง LIFF App ที่มี 3 หน้า: Home, Products, Profile
2. สร้าง Rich Menu ที่มี 3 ปุ่มเปิดแต่ละหน้า
3. แต่ละหน้าแสดงข้อมูลที่แตกต่างกัน
4. ใช้ Query Parameter สำหรับ Routing

---

## สรุป

ในบทนี้เราได้เรียนรู้:

| หัวข้อ | สาระสำคัญ |
|--------|-----------|
| LIFF คืออะไร | Web App Platform ภายใน LINE App |
| LIFF vs Browser | Authentication อัตโนมัติ, Native LINE Features |
| ขนาด LIFF | Full (100%), Tall (75%), Compact (50%) |
| LIFF SDK | CDN และ npm package |
| Hello World | HTML + LIFF SDK พื้นฐาน |
| Rich Menu | เปิด LIFF ด้วย URI Action |
| Flex Message | ปุ่มเปิด LIFF พร้อม Parameters |

### ขั้นตอนต่อไป

- **Part 42**: LIFF Authentication - liff.getProfile(), liff.getIDToken()
- **Part 43**: LIFF Send Messages - liff.sendMessages(), shareTargetPicker()
- **Part 44**: LIFF Advanced Features - QR Scanner, Bluetooth, Plugins
- **Part 45**: LIFF + React - Full React Application

---

*หมายเหตุ: แทน `YOUR_LIFF_ID` ด้วย LIFF ID จริงจาก LINE Developers Console*
