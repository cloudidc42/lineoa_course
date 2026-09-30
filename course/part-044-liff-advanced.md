# Part 44: LIFF Advanced Features - ฟีเจอร์ขั้นสูงของ LIFF

## สารบัญ

1. [liff.scanCode() และ scanCodeV2()](#liff-scancode)
2. [liff.openWindow()](#liff-openwindow)
3. [liff.closeWindow()](#liff-closewindow)
4. [liff.getContext()](#liff-getcontext)
5. [liff.getOS()](#liff-getos)
6. [liff.getLanguage()](#liff-getlanguage)
7. [liff.getVersion()](#liff-getversion)
8. [liff.isInClient()](#liff-isinclient)
9. [liff.isLoggedIn()](#liff-isloggedin)
10. [liff.permission.query() และ requestAll()](#liff-permission)
11. [Bluetooth API](#bluetooth-api)
12. [LIFF Plugins](#liff-plugins)
13. [Multiple LIFF Apps Management](#multiple-liff-apps-management)
14. [Advanced Patterns](#advanced-patterns)
15. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. liff.scanCode() และ scanCodeV2()

### 1.1 ภาพรวม

`liff.scanCode()` ช่วยให้ LIFF App เปิด QR Code Scanner ของ LINE ได้โดยตรง โดยไม่ต้องขอ Camera Permission แยก

```
ข้อดีเหนือ Browser Camera API:
✅ ใช้ Native LINE Scanner (เร็วกว่า)
✅ ไม่ต้องขอ Camera Permission
✅ UI เหมือน LINE ดั้งเดิม
✅ รองรับ QR Code ทุกประเภท
❌ ใช้ได้เฉพาะใน LINE App (ไม่ได้ใน Browser)
❌ ไม่รองรับ Barcode (เฉพาะ QR Code เท่านั้น)
```

### 1.2 liff.scanCode() (Version 1 - Deprecated)

```javascript
// เก่า - ยังใช้ได้แต่ไม่แนะนำ
async function scanQRCode() {
    if (!liff.isInClient()) {
        alert('QR Scanner ใช้ได้เฉพาะใน LINE App');
        return;
    }
    
    try {
        const result = await liff.scanCode();
        console.log('QR Result:', result.value);
        // result.value คือ String ที่อ่านได้จาก QR
        
    } catch (error) {
        if (error.code === 'CANCEL') {
            console.log('User cancelled scan');
        } else {
            console.error('Scan error:', error);
        }
    }
}
```

### 1.3 liff.scanCodeV2() (Version 2 - แนะนำ)

```javascript
// ใหม่ - แนะนำให้ใช้แทน scanCode
async function scanQRCodeV2() {
    if (!liff.isInClient()) {
        // Fallback สำหรับ Browser
        useWebCamera();
        return;
    }
    
    try {
        const result = await liff.scanCodeV2();
        
        // result คือ Object
        console.log('Raw value:', result.value);
        
        // ประมวลผล QR Content
        processQRCode(result.value);
        
    } catch (error) {
        if (error.code === 'CANCEL') {
            // ผู้ใช้กด Cancel - ไม่ใช่ Error
            console.log('Scan cancelled by user');
        } else {
            console.error('Scan failed:', error);
            handleScanError(error);
        }
    }
}

function processQRCode(value) {
    // ตรวจสอบประเภท QR
    if (value.startsWith('http')) {
        // URL
        handleQRUrl(value);
    } else if (value.startsWith('LINE://')) {
        // LINE Deep Link
        handleLINEDeepLink(value);
    } else if (/^\d+$/.test(value)) {
        // ตัวเลข (อาจเป็น Member ID, Product Code)
        handleNumericCode(value);
    } else {
        // ข้อความทั่วไป
        handleGenericQR(value);
    }
}
```

### 1.4 QR Scanner UI Component

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>QR Scanner</title>
    <script charset="utf-8" src="https://static.line-scdn.net/liff/edge/2/sdk.js"></script>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        
        body {
            font-family: -apple-system, sans-serif;
            background: #f7f8fa;
            min-height: 100vh;
        }
        
        .app {
            max-width: 480px;
            margin: 0 auto;
            padding: 24px 16px;
        }
        
        .scanner-card {
            background: white;
            border-radius: 16px;
            overflow: hidden;
            box-shadow: 0 2px 8px rgba(0,0,0,0.1);
            margin-bottom: 16px;
        }
        
        .scanner-header {
            background: linear-gradient(135deg, #1e293b, #334155);
            padding: 24px;
            text-align: center;
            color: white;
        }
        
        .scanner-icon {
            font-size: 48px;
            margin-bottom: 12px;
        }
        
        .scanner-title {
            font-size: 20px;
            font-weight: 700;
        }
        
        .scanner-subtitle {
            font-size: 13px;
            color: #94a3b8;
            margin-top: 4px;
        }
        
        .scan-btn {
            display: block;
            width: calc(100% - 32px);
            margin: 16px;
            padding: 16px;
            background: #06c755;
            color: white;
            border: none;
            border-radius: 12px;
            font-size: 16px;
            font-weight: 700;
            cursor: pointer;
            text-align: center;
            transition: opacity 0.2s;
        }
        
        .scan-btn:active { opacity: 0.85; }
        
        .scan-btn:disabled {
            background: #9ca3af;
            cursor: not-allowed;
        }
        
        .result-section {
            background: white;
            border-radius: 16px;
            overflow: hidden;
            box-shadow: 0 2px 8px rgba(0,0,0,0.1);
        }
        
        .result-header {
            padding: 16px;
            background: #f8f9fa;
            border-bottom: 1px solid #e5e7eb;
            font-size: 13px;
            font-weight: 700;
            color: #666;
            text-transform: uppercase;
        }
        
        .result-empty {
            padding: 32px;
            text-align: center;
            color: #9ca3af;
        }
        
        .result-item {
            padding: 16px;
            border-bottom: 1px solid #f3f4f6;
        }
        
        .result-item:last-child {
            border-bottom: none;
        }
        
        .result-type {
            font-size: 11px;
            font-weight: 700;
            text-transform: uppercase;
            color: #9ca3af;
            margin-bottom: 4px;
        }
        
        .result-value {
            font-size: 14px;
            color: #1a202c;
            word-break: break-all;
            font-family: monospace;
        }
        
        .result-actions {
            display: flex;
            gap: 8px;
            margin-top: 10px;
        }
        
        .action-btn {
            flex: 1;
            padding: 8px;
            border-radius: 8px;
            font-size: 13px;
            font-weight: 600;
            cursor: pointer;
            border: none;
        }
        
        .btn-copy {
            background: #f3f4f6;
            color: #374151;
        }
        
        .btn-open {
            background: #dbeafe;
            color: #1d4ed8;
        }
        
        .btn-send {
            background: #dcfce7;
            color: #166534;
        }
        
        .history-list {
            max-height: 300px;
            overflow-y: auto;
        }
        
        .no-client-warning {
            background: #fff3cd;
            border: 1px solid #ffc107;
            border-radius: 12px;
            padding: 16px;
            margin-bottom: 16px;
            display: flex;
            gap: 12px;
            align-items: flex-start;
        }
        
        .warning-icon { font-size: 24px; }
        
        .warning-text {
            font-size: 13px;
            color: #856404;
            line-height: 1.5;
        }
    </style>
</head>
<body>

<div class="app" id="app" style="display:none">
    <!-- คำเตือนถ้าเปิดนอก LINE -->
    <div class="no-client-warning" id="clientWarning" style="display:none">
        <span class="warning-icon">⚠️</span>
        <div class="warning-text">
            QR Scanner ใช้ได้เฉพาะใน LINE App เท่านั้น<br>
            กรุณาเปิดหน้านี้จาก LINE
        </div>
    </div>
    
    <!-- Scanner Card -->
    <div class="scanner-card">
        <div class="scanner-header">
            <div class="scanner-icon">📷</div>
            <div class="scanner-title">QR Code Scanner</div>
            <div class="scanner-subtitle">สแกน QR Code เพื่อรับข้อมูล</div>
        </div>
        <button class="scan-btn" id="scanBtn" onclick="startScan()">
            🔍 เปิดกล้องสแกน QR
        </button>
    </div>
    
    <!-- Results -->
    <div class="result-section">
        <div class="result-header">ผลการสแกน</div>
        <div class="history-list" id="resultList">
            <div class="result-empty">
                <div style="font-size:48px;margin-bottom:12px">📋</div>
                <div>ยังไม่มีผลการสแกน</div>
            </div>
        </div>
    </div>
</div>

<div id="loading" style="display:flex;align-items:center;justify-content:center;min-height:100vh">
    <p>กำลังโหลด...</p>
</div>

<script>
    const LIFF_ID = 'YOUR_LIFF_ID';
    const scanHistory = [];
    
    function detectQRType(value) {
        if (value.startsWith('http://') || value.startsWith('https://')) {
            return { type: 'URL', icon: '🔗', label: 'ลิงก์เว็บ' };
        }
        if (value.startsWith('LINE://') || value.startsWith('line://')) {
            return { type: 'LINE_DEEPLINK', icon: '💬', label: 'LINE Deep Link' };
        }
        if (/^[0-9]{6,20}$/.test(value)) {
            return { type: 'NUMERIC_CODE', icon: '🔢', label: 'รหัสตัวเลข' };
        }
        if (/^[A-Z0-9]{5,20}$/.test(value)) {
            return { type: 'CODE', icon: '🏷️', label: 'รหัสสินค้า' };
        }
        return { type: 'TEXT', icon: '📝', label: 'ข้อความ' };
    }
    
    async function startScan() {
        const scanBtn = document.getElementById('scanBtn');
        scanBtn.disabled = true;
        scanBtn.textContent = '⏳ กำลังเปิดกล้อง...';
        
        try {
            let result;
            
            if (liff.isInClient()) {
                result = await liff.scanCodeV2();
            } else {
                // Mock สำหรับ Development
                result = { value: 'MOCK_QR_12345_' + Date.now() };
            }
            
            if (result?.value) {
                addToHistory(result.value);
            }
            
        } catch (error) {
            if (error.code !== 'CANCEL') {
                alert('สแกนไม่สำเร็จ: ' + (error.message || 'ไม่ทราบสาเหตุ'));
            }
        } finally {
            scanBtn.disabled = false;
            scanBtn.textContent = '🔍 เปิดกล้องสแกน QR';
        }
    }
    
    function addToHistory(value) {
        const qrInfo = detectQRType(value);
        const item = {
            value,
            ...qrInfo,
            time: new Date().toLocaleTimeString('th-TH')
        };
        
        scanHistory.unshift(item);
        renderHistory();
    }
    
    function renderHistory() {
        const list = document.getElementById('resultList');
        
        if (scanHistory.length === 0) {
            list.innerHTML = `
                <div class="result-empty">
                    <div style="font-size:48px;margin-bottom:12px">📋</div>
                    <div>ยังไม่มีผลการสแกน</div>
                </div>
            `;
            return;
        }
        
        list.innerHTML = scanHistory.map((item, index) => `
            <div class="result-item">
                <div class="result-type">${item.icon} ${item.label} · ${item.time}</div>
                <div class="result-value">${escapeHtml(item.value)}</div>
                <div class="result-actions">
                    <button class="action-btn btn-copy" onclick="copyValue(${index})">
                        📋 Copy
                    </button>
                    ${item.type === 'URL' ? `
                        <button class="action-btn btn-open" onclick="openUrl(${index})">
                            🔗 เปิด
                        </button>
                    ` : ''}
                    ${liff.isInClient() ? `
                        <button class="action-btn btn-send" onclick="sendToChat(${index})">
                            💬 ส่ง
                        </button>
                    ` : ''}
                </div>
            </div>
        `).join('');
    }
    
    function escapeHtml(text) {
        const div = document.createElement('div');
        div.appendChild(document.createTextNode(text));
        return div.innerHTML;
    }
    
    function copyValue(index) {
        const value = scanHistory[index].value;
        navigator.clipboard.writeText(value)
            .then(() => alert('Copy แล้ว!'))
            .catch(() => {
                const el = document.createElement('textarea');
                el.value = value;
                document.body.appendChild(el);
                el.select();
                document.execCommand('copy');
                document.body.removeChild(el);
                alert('Copy แล้ว!');
            });
    }
    
    function openUrl(index) {
        const url = scanHistory[index].value;
        liff.openWindow({ url, external: true });
    }
    
    async function sendToChat(index) {
        const item = scanHistory[index];
        
        try {
            await liff.sendMessages([{
                type: 'text',
                text: `🔍 ผลการสแกน QR\n━━━━━━━━━\nประเภท: ${item.label}\nข้อมูล: ${item.value}`
            }]);
            alert('ส่งไปใน Chat แล้ว!');
        } catch (error) {
            alert('ไม่สามารถส่งได้: ' + error.message);
        }
    }
    
    // Init
    liff.init({ liffId: LIFF_ID })
        .then(() => {
            document.getElementById('loading').style.display = 'none';
            document.getElementById('app').style.display = 'block';
            
            if (!liff.isInClient()) {
                document.getElementById('clientWarning').style.display = 'flex';
            }
        })
        .catch(err => {
            document.getElementById('loading').textContent = 'Error: ' + err.message;
        });
</script>
</body>
</html>
```

---

## 2. liff.openWindow()

### 2.1 การใช้งาน

```javascript
// เปิด URL ใน LIFF Browser (Default)
liff.openWindow({
    url: 'https://example.com/page',
    external: false  // เปิดใน LIFF Browser (Default)
});

// เปิด URL ใน External Browser
liff.openWindow({
    url: 'https://example.com/page',
    external: true  // เปิดใน Browser ภายนอก LINE
});

// เปิด URL อื่นใน LIFF
liff.openWindow({
    url: 'https://liff.line.me/OTHER_LIFF_ID',
    external: false
});
```

### 2.2 Use Cases

```javascript
// Use Case 1: เปิดลิงก์จากผลการสแกน QR
async function handleQRResult(url) {
    if (url.startsWith('http')) {
        // ถาม User ก่อนเปิด (Security)
        const confirm = window.confirm(`เปิดลิงก์นี้?\n${url}`);
        if (confirm) {
            liff.openWindow({ url, external: true });
        }
    }
}

// Use Case 2: เปิด Payment Page
function openPaymentPage(orderId) {
    const paymentUrl = `https://payment.example.com/pay?order=${orderId}`;
    liff.openWindow({
        url: paymentUrl,
        external: false // เปิดใน LIFF เพื่อ UX ดีกว่า
    });
}

// Use Case 3: เปิด Map
function openLocation(lat, lng) {
    // เปิด Google Maps
    const mapsUrl = `https://maps.google.com/?q=${lat},${lng}`;
    liff.openWindow({ url: mapsUrl, external: true });
    
    // หรือเปิด LINE Map (ถ้าอยู่ใน LINE App)
    if (liff.isInClient()) {
        const lineMapUrl = `line://map?lat=${lat}&lng=${lng}`;
        liff.openWindow({ url: lineMapUrl, external: false });
    }
}

// Use Case 4: Share Page (กลับไป LIFF หน้าแรก)
function navigateToHome() {
    const homeUrl = `https://liff.line.me/YOUR_LIFF_ID`;
    liff.openWindow({ url: homeUrl, external: false });
}
```

---

## 3. liff.closeWindow()

### 3.1 การใช้งาน

```javascript
// ปิด LIFF Window (กลับไป LINE Chat)
liff.closeWindow();

// ปิดหลัง Delay
setTimeout(() => liff.closeWindow(), 2000);

// ปิดพร้อมส่งข้อความ
async function sendAndClose(message) {
    try {
        await liff.sendMessages([{ type: 'text', text: message }]);
        liff.closeWindow();
    } catch (error) {
        console.error(error);
    }
}
```

### 3.2 Close Animation Pattern

```javascript
// แสดง Success แล้วปิด
function showSuccessAndClose(message = 'ดำเนินการสำเร็จ!') {
    // แทนที่ Content ด้วย Success Screen
    document.body.innerHTML = `
        <div style="
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            background: white;
            gap: 16px;
            text-align: center;
            padding: 32px;
        ">
            <div style="
                width: 80px; height: 80px;
                background: #dcfce7;
                border-radius: 50%;
                display: flex;
                align-items: center;
                justify-content: center;
                font-size: 40px;
                animation: bounceIn 0.5s;
            ">✅</div>
            <h2 style="font-size: 22px; font-weight: 700; color: #111;">${message}</h2>
            <p style="color: #666; font-size: 14px;">หน้าต่างจะปิดโดยอัตโนมัติ...</p>
            <div style="
                width: 100%;
                height: 4px;
                background: #e5e7eb;
                border-radius: 2px;
                overflow: hidden;
            ">
                <div style="
                    height: 100%;
                    background: #06c755;
                    border-radius: 2px;
                    animation: fillBar 2s linear forwards;
                    width: 0%;
                "></div>
            </div>
        </div>
        <style>
            @keyframes bounceIn {
                0% { transform: scale(0); opacity: 0; }
                50% { transform: scale(1.2); }
                100% { transform: scale(1); opacity: 1; }
            }
            @keyframes fillBar {
                to { width: 100%; }
            }
        </style>
    `;
    
    setTimeout(() => liff.closeWindow(), 2000);
}
```

---

## 4. liff.getContext()

### 4.1 Context Object Structure

```typescript
interface LiffContext {
    type: 'utou' | 'room' | 'group' | 'square' | 'squareThread' | 'external' | 'none';
    viewType: 'full' | 'tall' | 'compact';
    accessTokenHash: string;
    availability: {
        shareTargetPicker: {
            permission: boolean;
            minVer: string;
        };
        multipleLiffTransition: {
            permission: boolean;
            minVer: string;
        };
        subWindow: {
            permission: boolean;
            minVer: string;
        };
    };
    // เฉพาะ type = 'utou'
    utouId?: string;
    // เฉพาะ type = 'group'  
    groupId?: string;
    // เฉพาะ type = 'room' (deprecated)
    roomId?: string;
    // เฉพาะ type = 'square'
    squareChatId?: string;
}
```

### 4.2 การใช้งาน getContext

```javascript
async function analyzeContext() {
    const context = liff.getContext();
    
    if (!context) {
        console.log('No context - probably opened from Direct URL');
        return;
    }
    
    console.log('Context Type:', context.type);
    console.log('View Type:', context.viewType);
    
    // ตรวจสอบ Context ประเภทต่างๆ
    switch (context.type) {
        case 'utou':
            console.log('Opened from 1:1 chat with Official Account');
            console.log('Chat ID:', context.utouId);
            break;
            
        case 'group':
            console.log('Opened from Group Chat');
            console.log('Group ID:', context.groupId);
            break;
            
        case 'square':
            console.log('Opened from LINE SQUARE');
            console.log('Square Chat ID:', context.squareChatId);
            break;
            
        case 'external':
            console.log('Opened from External Browser');
            break;
            
        case 'none':
            console.log('No specific context');
            break;
    }
    
    // ตรวจสอบ Permissions
    const { shareTargetPicker } = context.availability;
    if (shareTargetPicker.permission) {
        console.log('shareTargetPicker available!');
    }
    
    return context;
}

// ใช้ Context สำหรับ Custom Behavior
function customizeBehaviorByContext() {
    const context = liff.getContext();
    
    if (!context) return;
    
    if (context.viewType === 'compact') {
        // ปรับ UI สำหรับ Compact Mode
        document.body.classList.add('compact-mode');
    }
    
    if (context.type === 'group') {
        // อยู่ใน Group - ส่งข้อความแบบ Broadcast
        showGroupOptions();
    }
}
```

### 4.3 Context-Aware UI

```javascript
// แสดง UI ตาม Context
function renderContextInfo() {
    const context = liff.getContext();
    
    const contextDescriptions = {
        'utou': { emoji: '💬', desc: 'แชท 1:1 กับ Official Account', color: '#06c755' },
        'group': { emoji: '👥', desc: 'แชทกลุ่ม', color: '#3b82f6' },
        'room': { emoji: '🏠', desc: 'ห้องแชท', color: '#8b5cf6' },
        'square': { emoji: '🔶', desc: 'LINE SQUARE', color: '#f59e0b' },
        'external': { emoji: '🌐', desc: 'Browser ภายนอก', color: '#6b7280' },
        'none': { emoji: '❓', desc: 'ไม่มี Context', color: '#9ca3af' }
    };
    
    if (!context) {
        return '<div>ไม่มี Context</div>';
    }
    
    const info = contextDescriptions[context.type] || 
                 contextDescriptions['none'];
    
    return `
        <div style="
            display: flex;
            align-items: center;
            gap: 12px;
            padding: 12px 16px;
            background: ${info.color}15;
            border-left: 3px solid ${info.color};
            border-radius: 8px;
        ">
            <span style="font-size: 24px">${info.emoji}</span>
            <div>
                <div style="font-weight: 700; color: ${info.color}">${context.type.toUpperCase()}</div>
                <div style="font-size: 12px; color: #666">${info.desc}</div>
            </div>
        </div>
    `;
}
```

---

## 5. liff.getOS()

```javascript
// รับ OS ของผู้ใช้
const os = liff.getOS();
// 'ios' | 'android' | 'web'

// Responsive ตาม OS
function handleOSSpecific() {
    const os = liff.getOS();
    
    if (os === 'ios') {
        // iOS Specific
        document.documentElement.style.setProperty(
            '--safe-bottom', 
            'env(safe-area-inset-bottom)'
        );
        // ป้องกัน double-tap zoom
        document.addEventListener('touchend', e => e.preventDefault(), { passive: false });
        
    } else if (os === 'android') {
        // Android Specific
        // Android back button
        window.addEventListener('popstate', () => {
            if (liff.isInClient()) {
                liff.closeWindow();
            }
        });
        
    } else {
        // Desktop Web
        document.body.classList.add('desktop-mode');
    }
}

// Platform-specific styles
function applyPlatformStyles() {
    const os = liff.getOS();
    const root = document.documentElement;
    
    if (os === 'ios') {
        root.classList.add('ios');
        root.style.cssText += `
            --safe-top: env(safe-area-inset-top);
            --safe-bottom: env(safe-area-inset-bottom);
        `;
    } else if (os === 'android') {
        root.classList.add('android');
    }
}
```

---

## 6. liff.getLanguage()

```javascript
// รับภาษาที่ LINE ใช้
const language = liff.getLanguage();
// 'th' | 'en' | 'ja' | 'zh-TW' | 'zh-CN' | 'ko' | ...

// Internationalization (i18n)
const translations = {
    'th': {
        greeting: 'สวัสดีครับ',
        submit: 'ยืนยัน',
        cancel: 'ยกเลิก',
        error: 'เกิดข้อผิดพลาด'
    },
    'en': {
        greeting: 'Hello',
        submit: 'Submit',
        cancel: 'Cancel',
        error: 'An error occurred'
    },
    'ja': {
        greeting: 'こんにちは',
        submit: '確認',
        cancel: 'キャンセル',
        error: 'エラーが発生しました'
    }
};

function t(key) {
    const lang = liff.getLanguage() || 'en';
    const langKey = lang.split('-')[0]; // 'zh-TW' -> 'zh'
    
    return (translations[langKey] || translations['en'])[key] || key;
}

// ใช้งาน
document.getElementById('submitBtn').textContent = t('submit');
document.getElementById('cancelBtn').textContent = t('cancel');

// Auto-detect และตั้ง HTML lang
document.documentElement.lang = liff.getLanguage() || 'en';
```

---

## 7. liff.getVersion()

```javascript
// รับ LIFF SDK Version
const version = liff.getVersion();
// '2.22.0' หรือ Version ล่าสุด

// Feature Detection ตาม Version
function checkFeatureAvailability() {
    const version = liff.getVersion();
    const [major, minor, patch] = version.split('.').map(Number);
    
    const features = {
        scanCodeV2: major >= 2 && minor >= 15,     // v2.15.0+
        shareTargetPicker: major >= 2 && minor >= 3, // v2.3.0+
        bluetooth: major >= 2 && minor >= 1         // v2.1.0+
    };
    
    return features;
}

// แสดง Version Info
function displayVersionInfo() {
    const version = liff.getVersion();
    console.log(`LIFF SDK Version: ${version}`);
    
    const features = checkFeatureAvailability();
    Object.entries(features).forEach(([feature, available]) => {
        console.log(`${feature}: ${available ? '✅' : '❌'}`);
    });
}
```

---

## 8. liff.isInClient()

```javascript
// ตรวจสอบว่าอยู่ใน LINE App หรือไม่
const inClient = liff.isInClient();
// true = อยู่ใน LINE App
// false = เปิดใน Browser ภายนอก

// Conditional Features
function renderFeatures() {
    if (liff.isInClient()) {
        // ฟีเจอร์เต็มรูปแบบ
        showFullFeatures();
    } else {
        // ฟีเจอร์จำกัดสำหรับ Browser
        showLimitedFeatures();
        showOpenInLineHint();
    }
}

function showOpenInLineHint() {
    const banner = document.createElement('div');
    banner.style.cssText = `
        position: sticky;
        top: 0;
        background: #06c755;
        color: white;
        padding: 10px 16px;
        text-align: center;
        font-size: 13px;
        z-index: 1000;
        display: flex;
        align-items: center;
        justify-content: center;
        gap: 8px;
    `;
    
    const liffUrl = window.location.href.replace(
        window.location.origin,
        'https://liff.line.me/YOUR_LIFF_ID'
    );
    
    banner.innerHTML = `
        <span>💡 เปิดใน LINE เพื่อใช้ฟีเจอร์ครบ</span>
        <a href="${liffUrl}" 
           style="color:white;font-weight:700;text-decoration:underline">
           เปิด →
        </a>
    `;
    
    document.body.prepend(banner);
}
```

---

## 9. liff.isLoggedIn()

```javascript
// ตรวจสอบ Login State
const loggedIn = liff.isLoggedIn();

// Guard Function
async function requireAuth(callback) {
    if (!liff.isLoggedIn()) {
        const shouldLogin = window.confirm('ต้อง Login ก่อน ต้องการ Login หรือไม่?');
        if (shouldLogin) {
            liff.login({ redirectUri: window.location.href });
        }
        return;
    }
    
    await callback();
}

// ใช้งาน
document.getElementById('profileBtn').addEventListener('click', () => {
    requireAuth(async () => {
        const profile = await liff.getProfile();
        showProfile(profile);
    });
});

// Reactive Auth State
class AuthState {
    constructor() {
        this._handlers = { login: [], logout: [] };
    }
    
    on(event, handler) {
        this._handlers[event]?.push(handler);
        return this;
    }
    
    check() {
        return liff.isLoggedIn();
    }
    
    async login() {
        liff.login({ redirectUri: window.location.href });
    }
    
    logout() {
        liff.logout();
        this._handlers.logout.forEach(h => h());
        window.location.reload();
    }
}

const auth = new AuthState();
auth.on('logout', () => {
    document.getElementById('app').innerHTML = '<p>Logged out</p>';
});
```

---

## 10. liff.permission.query() และ requestAll()

### 10.1 Permission System

```javascript
// Scope ที่ต้องการ Permission
const requiredScopes = [
    'openid',
    'profile',
    'chat_message.write'
];

// ตรวจสอบ Permission
async function checkPermissions() {
    const results = {};
    
    for (const scope of requiredScopes) {
        try {
            const result = await liff.permission.query(scope);
            results[scope] = result.state;
            // 'granted' | 'prompt' | 'denied'
        } catch (error) {
            results[scope] = 'error';
        }
    }
    
    return results;
}

// ขอ Permission ทั้งหมดที่ต้องการ
async function requestAllPermissions() {
    try {
        await liff.permission.requestAll();
        console.log('All permissions granted!');
    } catch (error) {
        console.error('Permission request failed:', error);
        
        if (error.code === 'CANCEL') {
            console.log('User denied some permissions');
        }
    }
}
```

### 10.2 Permission-Based Feature Toggle

```javascript
class FeatureManager {
    constructor() {
        this.permissions = {};
    }
    
    async init() {
        const scopes = ['profile', 'chat_message.write'];
        
        for (const scope of scopes) {
            try {
                const result = await liff.permission.query(scope);
                this.permissions[scope] = result.state;
            } catch (e) {
                this.permissions[scope] = 'unknown';
            }
        }
    }
    
    can(feature) {
        const featurePermissions = {
            'view_profile': ['profile'],
            'send_message': ['chat_message.write'],
            'share': ['chat_message.write']
        };
        
        const required = featurePermissions[feature] || [];
        return required.every(perm => 
            this.permissions[perm] === 'granted'
        );
    }
    
    async request(feature) {
        const denied = !this.can(feature);
        
        if (denied) {
            try {
                await liff.permission.requestAll();
                await this.init(); // Re-check
            } catch (error) {
                return false;
            }
        }
        
        return this.can(feature);
    }
}

const features = new FeatureManager();

// ใช้งาน
await features.init();

if (features.can('send_message')) {
    document.getElementById('sendBtn').style.display = 'block';
} else {
    document.getElementById('sendBtn').style.display = 'none';
}
```

---

## 11. Bluetooth API

### 11.1 ภาพรวม LIFF Bluetooth

```
LIFF Bluetooth ช่วยให้ LIFF App เชื่อมต่อ BLE (Bluetooth Low Energy) Devices ได้

Use Cases:
- อ่านข้อมูลจาก IoT Devices
- เชื่อมต่อ Wearables
- Point of Sale (POS) devices
- Smart Home Control

Requirements:
- LINE App เวอร์ชัน 9.15.0+
- ต้องมี Scope: bluetooth
- ต้องได้รับ Approval จาก LINE
- ใช้ได้เฉพาะ iOS/Android (ไม่ใช่ Desktop)
```

### 11.2 การ Request BLE Device

```javascript
// ตรวจสอบ Bluetooth Support
async function checkBluetoothSupport() {
    if (!liff.isInClient()) {
        return { supported: false, reason: 'ต้องอยู่ใน LINE App' };
    }
    
    const os = liff.getOS();
    if (os === 'web') {
        return { supported: false, reason: 'ไม่รองรับบน Desktop' };
    }
    
    return { supported: true };
}

// เชื่อมต่อ BLE Device
async function connectBLEDevice() {
    const support = await checkBluetoothSupport();
    if (!support.supported) {
        alert(support.reason);
        return;
    }
    
    try {
        // Request Device (แสดง Device Picker UI)
        const device = await liff.bluetooth.requestDevice({
            filters: [
                { services: ['battery_service'] }
                // หรือ
                // { namePrefix: 'MyDevice' }
                // { name: 'ExactDeviceName' }
            ]
        });
        
        console.log('Device found:', device.name);
        
        // เชื่อมต่อ
        const server = await device.gatt.connect();
        console.log('Connected to:', device.name);
        
        // อ่านข้อมูล
        const service = await server.getPrimaryService('battery_service');
        const characteristic = await service.getCharacteristic('battery_level');
        const value = await characteristic.readValue();
        
        const batteryLevel = value.getUint8(0);
        console.log('Battery Level:', batteryLevel + '%');
        
        return { device, batteryLevel };
        
    } catch (error) {
        if (error.code === 'CANCEL') {
            console.log('User cancelled device selection');
        } else {
            console.error('BLE Error:', error);
            throw error;
        }
    }
}

// สแกนอุปกรณ์ IoT
async function scanIoTSensor() {
    try {
        const device = await liff.bluetooth.requestDevice({
            acceptAllDevices: true, // แสดงทุก Device
            optionalServices: ['generic_access', '0000xxx-...'] // Custom UUID
        });
        
        const server = await device.gatt.connect();
        
        // ฟัง Notifications
        const service = await server.getPrimaryService('generic_access');
        const char = await service.getCharacteristic('device_name');
        
        await char.startNotifications();
        char.addEventListener('characteristicvaluechanged', (event) => {
            const data = new Uint8Array(event.target.value.buffer);
            console.log('Sensor data:', data);
            updateSensorDisplay(data);
        });
        
    } catch (error) {
        console.error('IoT scan error:', error);
    }
}
```

---

## 12. LIFF Plugins

### 12.1 LIFF Plugin คืออะไร?

```
LIFF Plugin ช่วยให้เพิ่มความสามารถให้ LIFF SDK โดย:
- เพิ่ม Custom APIs ให้ liff Object
- Override พฤติกรรม Default
- Share Code ระหว่าง LIFF Apps

Use Cases:
- Analytics Tracking
- Error Reporting
- Custom Authentication
- Shared Utility Functions
```

### 12.2 สร้าง LIFF Plugin

```javascript
// myPlugin.js - Custom LIFF Plugin

const AnalyticsPlugin = {
    name: 'analytics',
    
    // เรียกตอน liff.use(plugin)
    install(liff, options = {}) {
        const { trackingId = 'GA-XXXXX' } = options;
        
        // เพิ่ม Analytics Methods ให้ liff.$analytics
        liff.$analytics = {
            
            // Track Event
            track(event, params = {}) {
                const profile = liff.isLoggedIn() 
                    ? null // ยังไม่ได้ดึง profile
                    : null;
                
                console.log(`[Analytics] ${event}`, {
                    ...params,
                    context: liff.getContext()?.type,
                    os: liff.getOS(),
                    language: liff.getLanguage(),
                    timestamp: Date.now()
                });
                
                // ส่งไปยัง Analytics Service จริง
                if (typeof gtag !== 'undefined') {
                    gtag('event', event, params);
                }
            },
            
            // Track Page View
            pageView(pageName) {
                this.track('page_view', { 
                    page_title: pageName,
                    page_location: window.location.href
                });
            },
            
            // Track Button Click
            buttonClick(buttonName, context = {}) {
                this.track('button_click', {
                    button_name: buttonName,
                    ...context
                });
            }
        };
        
        // Auto-track Page Views
        const originalPushState = history.pushState;
        history.pushState = function(...args) {
            originalPushState.apply(this, args);
            liff.$analytics.pageView(document.title);
        };
    }
};

// ErrorPlugin
const ErrorPlugin = {
    name: 'errorHandler',
    
    install(liff) {
        liff.$error = {
            
            report(error, context = {}) {
                console.error('[Error Report]', {
                    message: error.message,
                    stack: error.stack,
                    context: {
                        liffContext: liff.getContext(),
                        os: liff.getOS(),
                        version: liff.getVersion(),
                        isLoggedIn: liff.isLoggedIn(),
                        ...context
                    },
                    timestamp: new Date().toISOString()
                });
                
                // ส่งไปยัง Error Tracking Service
                // เช่น Sentry, Bugsnag
            },
            
            // Global Error Handler
            setupGlobalHandler() {
                window.addEventListener('error', (event) => {
                    this.report(event.error, { type: 'unhandled_error' });
                });
                
                window.addEventListener('unhandledrejection', (event) => {
                    this.report(new Error(event.reason), { type: 'unhandled_rejection' });
                });
            }
        };
        
        liff.$error.setupGlobalHandler();
    }
};
```

### 12.3 การใช้งาน LIFF Plugin

```javascript
// ลงทะเบียน Plugin ก่อน init
liff.use(AnalyticsPlugin, { trackingId: 'GA-12345' });
liff.use(ErrorPlugin);

// Init
await liff.init({ liffId: 'YOUR_LIFF_ID' });

// ใช้งาน Analytics
liff.$analytics.pageView('Home');

document.getElementById('orderBtn').addEventListener('click', () => {
    liff.$analytics.buttonClick('place_order', { 
        productId: 'P001',
        price: 299
    });
    handleOrder();
});

// ใช้งาน Error Handler
try {
    await riskyOperation();
} catch (error) {
    liff.$error.report(error, { operation: 'riskyOperation' });
    showErrorUI(error);
}
```

---

## 13. Multiple LIFF Apps Management

### 13.1 จัดการ Multiple LIFF IDs

```javascript
// config/liff.js
const LIFF_APPS = {
    main: {
        id: '1234567890-MainPage',
        name: 'หน้าหลัก',
        size: 'full',
        url: '/'
    },
    products: {
        id: '1234567890-Products',
        name: 'สินค้า',
        size: 'tall',
        url: '/products'
    },
    cart: {
        id: '1234567890-Cart',
        name: 'ตะกร้าสินค้า',
        size: 'full',
        url: '/cart'
    },
    profile: {
        id: '1234567890-Profile',
        name: 'โปรไฟล์',
        size: 'tall',
        url: '/profile'
    },
    qr: {
        id: '1234567890-QRScanner',
        name: 'สแกน QR',
        size: 'full',
        url: '/qr'
    }
};

// ตัวจัดการ LIFF Apps
class LiffAppManager {
    constructor(apps) {
        this.apps = apps;
        this.currentApp = null;
    }
    
    // ตรวจสอบว่ากำลังอยู่ใน LIFF App ไหน
    detectCurrentApp() {
        const path = window.location.pathname;
        
        for (const [key, app] of Object.entries(this.apps)) {
            if (path.startsWith(app.url)) {
                this.currentApp = { key, ...app };
                return this.currentApp;
            }
        }
        
        return null;
    }
    
    // เปิด LIFF App อื่น
    openApp(appKey, params = {}) {
        const app = this.apps[appKey];
        if (!app) throw new Error(`App "${appKey}" not found`);
        
        const queryString = new URLSearchParams(params).toString();
        const url = `https://liff.line.me/${app.id}${queryString ? '?' + queryString : ''}`;
        
        liff.openWindow({ url, external: false });
    }
    
    // สร้าง LIFF URL สำหรับ Share
    getShareUrl(appKey, params = {}) {
        const app = this.apps[appKey];
        if (!app) return null;
        
        const queryString = new URLSearchParams(params).toString();
        return `https://liff.line.me/${app.id}${queryString ? '?' + queryString : ''}`;
    }
}

const liffApps = new LiffAppManager(LIFF_APPS);

// ใช้งาน
// จาก main page ไปหน้า products
document.getElementById('viewProducts').addEventListener('click', () => {
    liffApps.openApp('products', { category: 'clothing' });
});

// จาก products ไปหน้า cart
document.getElementById('viewCart').addEventListener('click', () => {
    liffApps.openApp('cart');
});

// Share Product URL
async function shareProduct(productId) {
    const shareUrl = liffApps.getShareUrl('products', { product: productId });
    
    await liff.shareTargetPicker([{
        type: 'flex',
        altText: 'ดูสินค้านี้',
        contents: {
            type: 'bubble',
            body: {
                type: 'box',
                layout: 'vertical',
                contents: [
                    { type: 'text', text: 'ดูสินค้านี้!' },
                    {
                        type: 'button',
                        action: { type: 'uri', label: 'ดูเลย', uri: shareUrl }
                    }
                ]
            }
        }
    }]);
}
```

### 13.2 LIFF Navigation Pattern

```javascript
// liff-router.js - Simple Router สำหรับ Multiple LIFF

class LiffRouter {
    constructor() {
        this.routes = new Map();
        this.currentRoute = null;
    }
    
    // ลงทะเบียน Route
    register(path, component) {
        this.routes.set(path, component);
        return this;
    }
    
    // Navigate ไป Route
    async navigate(path, state = {}) {
        const component = this.routes.get(path);
        
        if (!component) {
            console.error(`Route "${path}" not found`);
            return;
        }
        
        // Update URL โดยไม่ Reload
        history.pushState(state, '', path);
        
        // Render Component
        this.currentRoute = { path, component, state };
        await component(state);
    }
    
    // Start Router
    async start() {
        const path = window.location.pathname;
        const query = new URLSearchParams(window.location.search);
        
        const component = this.routes.get(path) || this.routes.get('/');
        
        if (component) {
            this.currentRoute = { path, component };
            await component(Object.fromEntries(query));
        }
        
        // Handle browser back/forward
        window.addEventListener('popstate', async (event) => {
            const newPath = window.location.pathname;
            const comp = this.routes.get(newPath);
            if (comp) {
                await comp(event.state || {});
            }
        });
    }
}

// ใช้งาน
const router = new LiffRouter();

router
    .register('/', renderHome)
    .register('/products', renderProducts)
    .register('/product', renderProductDetail)
    .register('/cart', renderCart)
    .register('/profile', renderProfile);

// Init LIFF แล้วเริ่ม Router
liff.init({ liffId: 'YOUR_LIFF_ID' })
    .then(() => router.start());
```

---

## 14. Advanced Patterns

### 14.1 LIFF + WebSocket Real-time Updates

```javascript
// Real-time updates ด้วย WebSocket ใน LIFF

class LiffRealtimeApp {
    constructor(liffId, wsUrl) {
        this.liffId = liffId;
        this.wsUrl = wsUrl;
        this.ws = null;
        this.userId = null;
    }
    
    async init() {
        await liff.init({ liffId: this.liffId });
        
        if (liff.isLoggedIn()) {
            const profile = await liff.getProfile();
            this.userId = profile.userId;
            
            // เชื่อมต่อ WebSocket
            this.connectWebSocket();
        }
    }
    
    connectWebSocket() {
        this.ws = new WebSocket(`${this.wsUrl}?userId=${this.userId}`);
        
        this.ws.onopen = () => {
            console.log('WebSocket connected');
            this.onReady();
        };
        
        this.ws.onmessage = (event) => {
            const data = JSON.parse(event.data);
            this.handleMessage(data);
        };
        
        this.ws.onclose = () => {
            console.log('WebSocket disconnected, reconnecting...');
            setTimeout(() => this.connectWebSocket(), 3000);
        };
    }
    
    handleMessage(data) {
        switch (data.type) {
            case 'order_update':
                updateOrderStatus(data.payload);
                break;
            case 'notification':
                showNotification(data.payload);
                break;
            case 'chat_message':
                appendChatMessage(data.payload);
                break;
        }
    }
    
    onReady() {
        // Notify server ว่า LIFF เปิดอยู่
        this.ws.send(JSON.stringify({
            type: 'liff_opened',
            context: liff.getContext()
        }));
    }
    
    send(type, payload) {
        if (this.ws?.readyState === WebSocket.OPEN) {
            this.ws.send(JSON.stringify({ type, payload }));
        }
    }
}
```

### 14.2 LIFF Environment Detection

```javascript
// Comprehensive Environment Detection

function detectLiffEnvironment() {
    const isInClient = liff.isInClient();
    const os = liff.getOS();
    const context = liff.getContext();
    const version = liff.getVersion();
    const language = liff.getLanguage();
    
    return {
        // Client
        isInClient,
        
        // Platform
        platform: isInClient ? `LINE (${os})` : `Web Browser (${os})`,
        
        // Context
        contextType: context?.type || 'none',
        viewType: context?.viewType || 'full',
        
        // SDK
        sdkVersion: version,
        
        // User Language
        language,
        
        // Capabilities
        capabilities: {
            sendMessages: isInClient && !!context && context.type !== 'none',
            shareTargetPicker: isInClient,
            scanCode: isInClient && os !== 'web',
            bluetooth: isInClient && (os === 'ios' || os === 'android'),
            closeWindow: isInClient,
            login: true,
            profile: liff.isLoggedIn()
        },
        
        // Debug
        url: window.location.href,
        timestamp: new Date().toISOString()
    };
}

// แสดง Debug Panel (Dev Mode เท่านั้น)
function showDebugPanel() {
    const env = detectLiffEnvironment();
    
    const panel = document.createElement('div');
    panel.style.cssText = `
        position: fixed;
        bottom: 0;
        left: 0;
        right: 0;
        background: #1e293b;
        color: #94a3b8;
        padding: 12px;
        font-size: 11px;
        font-family: monospace;
        max-height: 40vh;
        overflow-y: auto;
        z-index: 9999;
    `;
    
    panel.innerHTML = `
        <div style="color:#06c755;font-weight:700;margin-bottom:8px">
            LIFF Debug Info
            <button onclick="this.parentElement.parentElement.remove()" 
                    style="float:right;background:none;border:none;color:#94a3b8;cursor:pointer">✕</button>
        </div>
        <pre>${JSON.stringify(env, null, 2)}</pre>
    `;
    
    document.body.appendChild(panel);
}

// ใน Dev Mode
if (process.env.NODE_ENV === 'development') {
    // long press สำหรับ Debug Panel
    let pressTimer;
    document.addEventListener('touchstart', () => {
        pressTimer = setTimeout(showDebugPanel, 3000);
    });
    document.addEventListener('touchend', () => {
        clearTimeout(pressTimer);
    });
}
```

---

## 15. แบบฝึกหัด

### แบบฝึกหัดที่ 1: QR Check-in System

สร้างระบบ Check-in ด้วย QR Code:
1. Admin สร้าง QR Code ที่มี Event ID
2. ผู้เข้าร่วมสแกน QR ด้วย LIFF
3. ส่งผลการสแกนไป Backend
4. Backend บันทึก Check-in
5. ส่งข้อความยืนยันใน Chat

```javascript
// Hint: Flow
// 1. liff.scanCodeV2() → QR Value
// 2. Parse Event ID จาก QR Value
// 3. POST to /api/checkin { eventId, userId }
// 4. liff.sendMessages() ยืนยัน
// 5. liff.closeWindow()
```

### แบบฝึกหัดที่ 2: LIFF Environment Display

สร้างหน้า Debug ที่แสดงข้อมูลทั้งหมด:
- liff.getOS()
- liff.getLanguage()
- liff.getVersion()
- liff.isInClient()
- liff.isLoggedIn()
- liff.getContext()
- Permission states

### แบบฝึกหัดที่ 3: Multi-Page LIFF App

สร้าง LIFF App ที่มี 3 หน้า:
1. Home → แสดงเมนู + liff info
2. Scanner → QR Scanner พร้อม History
3. Profile → ข้อมูล User + Token info

ใช้ URL Routing ระหว่างหน้า

---

## สรุป APIs ในบทนี้

| API | ประเภท | ใช้งาน |
|-----|--------|--------|
| `liff.scanCodeV2()` | async | สแกน QR Code |
| `liff.openWindow()` | sync | เปิด URL |
| `liff.closeWindow()` | sync | ปิด LIFF |
| `liff.getContext()` | sync | ข้อมูล Chat Context |
| `liff.getOS()` | sync | ระบบปฏิบัติการ |
| `liff.getLanguage()` | sync | ภาษา |
| `liff.getVersion()` | sync | LIFF SDK Version |
| `liff.isInClient()` | sync | อยู่ใน LINE App? |
| `liff.isLoggedIn()` | sync | Login แล้ว? |
| `liff.permission.query()` | async | ตรวจสอบ Permission |
| `liff.permission.requestAll()` | async | ขอ Permission |
| `liff.bluetooth.requestDevice()` | async | เชื่อมต่อ BLE |
| `liff.use()` | sync | ติดตั้ง Plugin |

### ขั้นตอนต่อไป
- **Part 45**: LIFF + React - Full React Application พร้อม Custom Hooks
