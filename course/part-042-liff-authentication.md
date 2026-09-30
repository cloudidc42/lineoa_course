# Part 42: LIFF Authentication - การยืนยันตัวตนด้วย LINE Login

## สารบัญ

1. [LINE Login และ LIFF](#line-login-และ-liff)
2. [liff.init() Setup](#liff-init-setup)
3. [liff.getProfile() - ดึงข้อมูลผู้ใช้](#liff-getprofile)
4. [liff.getAccessToken()](#liff-getaccesstoken)
5. [liff.getIDToken()](#liff-getidtoken)
6. [การ Verify Token บน Server](#การ-verify-token-บน-server)
7. [การตรวจสอบ Login State](#การตรวจสอบ-login-state)
8. [liff.login() และ liff.logout()](#liff-login-และ-liff-logout)
9. [การจัดการ Unauthenticated State](#การจัดการ-unauthenticated-state)
10. [User Registration ครั้งแรก](#user-registration-ครั้งแรก)
11. [JWT Verification](#jwt-verification)
12. [Complete Authentication Flow](#complete-authentication-flow)
13. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. LINE Login และ LIFF

### 1.1 ความสัมพันธ์ระหว่าง LINE Login และ LIFF

**LINE Login** คือระบบ Authentication ของ LINE ที่ช่วยให้ผู้ใช้ Login ด้วย LINE Account บนเว็บไซต์หรือแอปภายนอก

**LIFF** ใช้ LINE Login โดยอัตโนมัติ เมื่อเปิด LIFF ใน LINE App ผู้ใช้จะถูก Authenticate โดยอัตโนมัติ

```
┌─────────────────────────────────────────────────────────────┐
│                    Authentication Flow                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  เปิดจาก LINE App:                                          │
│  User Opens LIFF → Auto Login → liff.init() → Ready ✅     │
│                                                              │
│  เปิดจาก External Browser:                                  │
│  User Opens LIFF → liff.init() → Not Logged In →            │
│  liff.login() → LINE Login Page → Redirect Back → Ready ✅  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 1.2 Scope ที่ต้องการสำหรับ Authentication

```javascript
// ตารางแสดง Scope และสิ่งที่ได้
const scopeDetails = {
    openid: {
        required: true,
        provides: {
            userId: "รหัสประจำตัวผู้ใช้ (ไม่ซ้ำกัน)",
            idToken: "JWT Token สำหรับ Verify"
        },
        example: "U4af4980629..."
    },
    profile: {
        optional: false, // แนะนำให้เปิด
        provides: {
            displayName: "ชื่อที่แสดงใน LINE",
            pictureUrl: "URL รูปโปรไฟล์",
            statusMessage: "ข้อความสถานะ"
        },
        example: "สมชาย ใจดี"
    },
    email: {
        optional: true,
        note: "ต้องได้รับการ Approve จาก LINE",
        provides: {
            email: "อีเมลที่ผูกกับบัญชี LINE"
        }
    }
};
```

### 1.3 Consent Screen

```
เมื่อผู้ใช้เปิด LIFF ครั้งแรก (หรือ Scope เปลี่ยน):

┌─────────────────────────────────┐
│         LINE Login              │
│                                 │
│  [App Icon] My LIFF App         │
│                                 │
│  แอปนี้ขอเข้าถึง:               │
│  ✓ ข้อมูลโปรไฟล์ของคุณ         │
│  ✓ User ID                      │
│  ✓ ส่งข้อความในนามคุณ          │
│                                 │
│  [อนุญาต]     [ปฏิเสธ]          │
└─────────────────────────────────┘
```

---

## 2. liff.init() Setup

### 2.1 Basic Init

```javascript
// การใช้งานพื้นฐาน
liff.init({ liffId: 'YOUR_LIFF_ID' })
    .then(() => {
        console.log('LIFF initialized!');
        // เริ่มใช้งาน LIFF APIs ได้
    })
    .catch((error) => {
        console.error('LIFF init failed:', error);
    });
```

### 2.2 Init พร้อม Options

```javascript
// Init พร้อม Options ครบถ้วน
await liff.init({
    liffId: 'YOUR_LIFF_ID',
    
    // withLoginOnExternalBrowser: true/false
    // true = Login อัตโนมัติเมื่อเปิดใน External Browser
    // false = ต้อง login() เอง (Default: false)
    withLoginOnExternalBrowser: true,
    
    // mock: สำหรับ Development เท่านั้น
    // ไม่มีใน Production
});
```

### 2.3 Init Error Handling

```javascript
// Error Types ที่อาจเกิดขึ้น
const liffErrors = {
    "INIT_FAILED": "LIFF ID ไม่ถูกต้อง หรือ Channel ถูกปิด",
    "INVALID_ARGUMENT": "ค่า liffId ผิดรูปแบบ",
    "FORBIDDEN": "Domain ไม่ตรงกับที่ตั้งค่าไว้",
    "NETWORK_ERROR": "ปัญหาการเชื่อมต่อ Internet"
};

async function initLiffWithErrorHandling(liffId) {
    try {
        await liff.init({ liffId });
        return { success: true };
    } catch (error) {
        console.error('Init failed:', error);
        
        // จัดการ Error แต่ละประเภท
        if (error.code === 'INIT_FAILED') {
            return { 
                success: false, 
                message: 'กรุณาตรวจสอบ LIFF ID' 
            };
        }
        
        if (error.code === 'FORBIDDEN') {
            return { 
                success: false, 
                message: 'Domain ไม่ได้รับอนุญาต' 
            };
        }
        
        return { 
            success: false, 
            message: `เกิดข้อผิดพลาด: ${error.message}` 
        };
    }
}
```

### 2.4 Init Pattern สำหรับ Single Page App

```javascript
// Pattern: Init ครั้งเดียว แล้วใช้ทั้งแอป
class LiffManager {
    constructor() {
        this._initialized = false;
        this._initPromise = null;
    }
    
    // เรียกครั้งเดียว ไม่ว่า Component ไหนจะเรียก
    async init(liffId) {
        if (this._initialized) return;
        
        // ป้องกัน init ซ้ำ
        if (!this._initPromise) {
            this._initPromise = liff.init({ liffId });
        }
        
        await this._initPromise;
        this._initialized = true;
    }
    
    isInitialized() {
        return this._initialized;
    }
}

// Singleton
const liffManager = new LiffManager();

// ใช้งาน
await liffManager.init('YOUR_LIFF_ID');
```

---

## 3. liff.getProfile() - ดึงข้อมูลผู้ใช้

### 3.1 Profile Object Structure

```typescript
// TypeScript Interface ของ Profile
interface LiffProfile {
    userId: string;           // "Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
    displayName: string;      // "ชื่อผู้ใช้"
    pictureUrl: string;       // "https://profile.line-scdn.net/..."
    statusMessage?: string;   // "ข้อความสถานะ" (อาจไม่มี)
}
```

### 3.2 การใช้งาน getProfile

```javascript
async function getUserProfile() {
    // ต้อง init และ login ก่อน
    if (!liff.isLoggedIn()) {
        throw new Error('User is not logged in');
    }
    
    try {
        const profile = await liff.getProfile();
        
        console.log('User ID:', profile.userId);
        // "U4af4980629... (ตัวเลขยาว 33 หลัก)"
        
        console.log('Name:', profile.displayName);
        // "สมชาย ใจดี"
        
        console.log('Picture:', profile.pictureUrl);
        // "https://profile.line-scdn.net/0h..."
        
        console.log('Status:', profile.statusMessage);
        // "Hello World!" หรือ undefined
        
        return profile;
    } catch (error) {
        console.error('Get profile error:', error);
        throw error;
    }
}
```

### 3.3 แสดง Profile ใน HTML

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>LIFF Profile</title>
    <script charset="utf-8" src="https://static.line-scdn.net/liff/edge/2/sdk.js"></script>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        
        body {
            font-family: -apple-system, BlinkMacSystemFont, sans-serif;
            background: #f7f8fa;
            min-height: 100vh;
        }
        
        .page {
            max-width: 480px;
            margin: 0 auto;
            padding: 0 0 80px;
        }
        
        /* Header */
        .page-header {
            background: linear-gradient(135deg, #06c755, #00a040);
            padding: 40px 20px 60px;
            text-align: center;
            color: white;
            position: relative;
        }
        
        .page-header h1 {
            font-size: 22px;
            font-weight: 700;
        }
        
        /* Profile Card */
        .profile-card {
            background: white;
            margin: -40px 16px 0;
            border-radius: 20px;
            padding: 24px;
            box-shadow: 0 8px 24px rgba(0,0,0,0.12);
            text-align: center;
            position: relative;
            z-index: 1;
        }
        
        .avatar-container {
            position: relative;
            display: inline-block;
            margin-bottom: 16px;
        }
        
        .avatar {
            width: 100px;
            height: 100px;
            border-radius: 50%;
            border: 4px solid #06c755;
            object-fit: cover;
        }
        
        .avatar-placeholder {
            width: 100px;
            height: 100px;
            border-radius: 50%;
            background: #e5e7eb;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 40px;
        }
        
        .online-badge {
            position: absolute;
            bottom: 4px;
            right: 4px;
            width: 20px;
            height: 20px;
            background: #06c755;
            border-radius: 50%;
            border: 3px solid white;
        }
        
        .display-name {
            font-size: 22px;
            font-weight: 700;
            color: #111;
            margin-bottom: 6px;
        }
        
        .status-message {
            font-size: 14px;
            color: #666;
            margin-bottom: 16px;
            font-style: italic;
        }
        
        .user-id-box {
            background: #f3f4f6;
            border-radius: 10px;
            padding: 12px;
            font-family: monospace;
            font-size: 11px;
            color: #444;
            word-break: break-all;
            text-align: left;
        }
        
        .user-id-label {
            font-size: 10px;
            font-weight: 700;
            text-transform: uppercase;
            color: #9ca3af;
            margin-bottom: 4px;
            font-family: sans-serif;
        }
        
        /* Stats Row */
        .stats-row {
            display: grid;
            grid-template-columns: repeat(3, 1fr);
            gap: 1px;
            background: #e5e7eb;
            border-radius: 16px;
            overflow: hidden;
            margin: 16px 16px 0;
        }
        
        .stat-item {
            background: white;
            padding: 16px 8px;
            text-align: center;
        }
        
        .stat-value {
            font-size: 20px;
            font-weight: 700;
            color: #06c755;
        }
        
        .stat-label {
            font-size: 11px;
            color: #9ca3af;
            margin-top: 4px;
        }
        
        /* Info Section */
        .section {
            background: white;
            border-radius: 16px;
            margin: 12px 16px 0;
            overflow: hidden;
        }
        
        .section-title {
            padding: 14px 16px;
            font-size: 13px;
            font-weight: 700;
            color: #9ca3af;
            text-transform: uppercase;
            letter-spacing: 0.05em;
            border-bottom: 1px solid #f3f4f6;
        }
        
        .info-row {
            padding: 14px 16px;
            display: flex;
            align-items: center;
            gap: 12px;
            border-bottom: 1px solid #f3f4f6;
        }
        
        .info-row:last-child {
            border-bottom: none;
        }
        
        .info-icon {
            width: 36px;
            height: 36px;
            border-radius: 10px;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 18px;
            flex-shrink: 0;
        }
        
        .icon-green { background: #d1fae5; }
        .icon-blue { background: #dbeafe; }
        .icon-purple { background: #ede9fe; }
        .icon-orange { background: #fed7aa; }
        
        .info-content {
            flex: 1;
        }
        
        .info-label {
            font-size: 12px;
            color: #9ca3af;
        }
        
        .info-value {
            font-size: 14px;
            font-weight: 600;
            color: #111;
            margin-top: 2px;
        }
        
        /* Token Section */
        .token-box {
            background: #1e293b;
            color: #94a3b8;
            padding: 12px;
            margin: 12px 16px;
            border-radius: 12px;
            font-family: monospace;
            font-size: 11px;
            word-break: break-all;
            position: relative;
        }
        
        .token-label {
            color: #64748b;
            font-size: 10px;
            text-transform: uppercase;
            margin-bottom: 6px;
            font-family: sans-serif;
        }
        
        .token-value {
            color: #06c755;
        }
        
        .copy-btn {
            position: absolute;
            top: 8px;
            right: 8px;
            background: #334155;
            color: #94a3b8;
            border: none;
            border-radius: 6px;
            padding: 4px 8px;
            font-size: 11px;
            cursor: pointer;
        }
        
        /* Actions */
        .action-bar {
            position: fixed;
            bottom: 0;
            left: 0;
            right: 0;
            background: white;
            border-top: 1px solid #e5e7eb;
            padding: 12px 16px;
            padding-bottom: max(12px, env(safe-area-inset-bottom));
            display: flex;
            gap: 12px;
        }
        
        .btn {
            flex: 1;
            padding: 14px;
            border-radius: 12px;
            font-size: 14px;
            font-weight: 700;
            border: none;
            cursor: pointer;
            transition: opacity 0.2s;
        }
        
        .btn:active { opacity: 0.8; }
        
        .btn-primary {
            background: #06c755;
            color: white;
        }
        
        .btn-outline {
            background: white;
            color: #374151;
            border: 1.5px solid #d1d5db;
        }
        
        /* Loading */
        .loading-screen {
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            gap: 16px;
        }
        
        .spinner {
            width: 40px;
            height: 40px;
            border: 3px solid #e5e7eb;
            border-top-color: #06c755;
            border-radius: 50%;
            animation: spin 0.7s linear infinite;
        }
        
        @keyframes spin { to { transform: rotate(360deg); } }
        
        /* Not Logged In */
        .not-logged-in {
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            padding: 32px;
            text-align: center;
        }
        
        .not-logged-in .icon {
            font-size: 80px;
            margin-bottom: 24px;
        }
        
        .not-logged-in h2 {
            font-size: 20px;
            font-weight: 700;
            margin-bottom: 8px;
        }
        
        .not-logged-in p {
            color: #666;
            margin-bottom: 32px;
            line-height: 1.6;
        }
        
        .login-btn {
            background: #06c755;
            color: white;
            border: none;
            padding: 16px 32px;
            border-radius: 12px;
            font-size: 16px;
            font-weight: 700;
            cursor: pointer;
        }
    </style>
</head>
<body>

    <!-- Loading -->
    <div id="loading" class="loading-screen">
        <div class="spinner"></div>
        <p style="color:#666">กำลังโหลด...</p>
    </div>
    
    <!-- Not Logged In -->
    <div id="notLoggedIn" class="not-logged-in" style="display:none">
        <div class="icon">🔒</div>
        <h2>กรุณา Login</h2>
        <p>เพื่อดูข้อมูลโปรไฟล์ของคุณ<br>กรุณา Login ด้วย LINE Account</p>
        <button class="login-btn" onclick="handleLogin()">
            Login ด้วย LINE
        </button>
    </div>
    
    <!-- Main Content -->
    <div id="content" class="page" style="display:none">
        <div class="page-header">
            <h1>โปรไฟล์ของฉัน</h1>
        </div>
        
        <!-- Profile Card -->
        <div class="profile-card">
            <div class="avatar-container">
                <img id="avatar" class="avatar" src="" alt="Avatar" 
                     onerror="this.style.display='none'; this.nextElementSibling.style.display='flex'">
                <div class="avatar-placeholder" style="display:none">👤</div>
                <div class="online-badge"></div>
            </div>
            <div class="display-name" id="displayName">-</div>
            <div class="status-message" id="statusMessage"></div>
            <div class="user-id-box">
                <div class="user-id-label">LINE User ID</div>
                <div id="userId">-</div>
            </div>
        </div>
        
        <!-- Environment Info -->
        <div class="section" style="margin-top:16px">
            <div class="section-title">ข้อมูล LIFF</div>
            <div class="info-row">
                <div class="info-icon icon-green">📱</div>
                <div class="info-content">
                    <div class="info-label">Environment</div>
                    <div class="info-value" id="environment">-</div>
                </div>
            </div>
            <div class="info-row">
                <div class="info-icon icon-blue">💻</div>
                <div class="info-content">
                    <div class="info-label">ระบบปฏิบัติการ</div>
                    <div class="info-value" id="osInfo">-</div>
                </div>
            </div>
            <div class="info-row">
                <div class="info-icon icon-purple">🌐</div>
                <div class="info-content">
                    <div class="info-label">ภาษา</div>
                    <div class="info-value" id="language">-</div>
                </div>
            </div>
            <div class="info-row">
                <div class="info-icon icon-orange">💬</div>
                <div class="info-content">
                    <div class="info-label">Chat Context</div>
                    <div class="info-value" id="chatContext">-</div>
                </div>
            </div>
        </div>
        
        <!-- Token Info -->
        <div class="token-box">
            <div class="token-label">Access Token</div>
            <button class="copy-btn" onclick="copyToken('access')">Copy</button>
            <div class="token-value" id="accessToken">-</div>
        </div>
        
        <!-- Action Bar -->
        <div class="action-bar">
            <button class="btn btn-outline" onclick="handleLogout()">Logout</button>
            <button class="btn btn-primary" onclick="refreshProfile()">รีเฟรช</button>
        </div>
    </div>

    <script>
        const LIFF_ID = 'YOUR_LIFF_ID';
        let currentProfile = null;
        
        function handleLogin() {
            liff.login({ redirectUri: window.location.href });
        }
        
        function handleLogout() {
            if (confirm('ต้องการ Logout หรือไม่?')) {
                liff.logout();
                showNotLoggedIn();
            }
        }
        
        function showLoading() {
            document.getElementById('loading').style.display = 'flex';
            document.getElementById('notLoggedIn').style.display = 'none';
            document.getElementById('content').style.display = 'none';
        }
        
        function showNotLoggedIn() {
            document.getElementById('loading').style.display = 'none';
            document.getElementById('notLoggedIn').style.display = 'flex';
            document.getElementById('content').style.display = 'none';
        }
        
        function showContent() {
            document.getElementById('loading').style.display = 'none';
            document.getElementById('notLoggedIn').style.display = 'none';
            document.getElementById('content').style.display = 'block';
        }
        
        async function loadProfile() {
            try {
                const profile = await liff.getProfile();
                currentProfile = profile;
                
                // แสดง Avatar
                const avatar = document.getElementById('avatar');
                if (profile.pictureUrl) {
                    avatar.src = profile.pictureUrl;
                }
                
                // ข้อมูล Profile
                document.getElementById('displayName').textContent = profile.displayName;
                document.getElementById('userId').textContent = profile.userId;
                
                if (profile.statusMessage) {
                    document.getElementById('statusMessage').textContent = 
                        `"${profile.statusMessage}"`;
                }
                
                // LIFF Info
                document.getElementById('environment').textContent = 
                    liff.isInClient() ? 'LINE App ✓' : 'External Browser';
                document.getElementById('osInfo').textContent = 
                    liff.getOS() || 'Unknown';
                document.getElementById('language').textContent = 
                    liff.getLanguage() || 'Unknown';
                
                const ctx = liff.getContext();
                document.getElementById('chatContext').textContent = 
                    ctx ? `${ctx.type}` : 'None';
                
                // Access Token (แสดงแบบ Masked)
                const token = liff.getAccessToken();
                if (token) {
                    const masked = token.substring(0, 20) + '...' + 
                                   token.substring(token.length - 10);
                    document.getElementById('accessToken').textContent = masked;
                    window._accessToken = token;
                }
                
                showContent();
            } catch (error) {
                console.error('Load profile error:', error);
                alert('ไม่สามารถโหลดข้อมูลได้: ' + error.message);
            }
        }
        
        function copyToken(type) {
            const token = type === 'access' ? window._accessToken : null;
            if (!token) return;
            
            navigator.clipboard.writeText(token)
                .then(() => alert('Copy แล้ว!'))
                .catch(() => alert('ไม่สามารถ Copy ได้'));
        }
        
        async function refreshProfile() {
            showLoading();
            await loadProfile();
        }
        
        // Init
        liff.init({ liffId: LIFF_ID })
            .then(async () => {
                if (liff.isLoggedIn()) {
                    await loadProfile();
                } else {
                    showNotLoggedIn();
                }
            })
            .catch(error => {
                document.getElementById('loading').innerHTML = 
                    `<p style="color:red">Error: ${error.message}</p>`;
            });
    </script>
</body>
</html>
```

---

## 4. liff.getAccessToken()

### 4.1 Access Token คืออะไร?

```
Access Token ใน LIFF คือ Short-lived Token ที่ใช้:
- เรียก LINE APIs โดยตรง
- ดึงข้อมูล Profile จาก LINE Social API
- ไม่ควรเก็บใน Client (ควรส่งไป Server ทันที)

อายุของ Token: ประมาณ 12 ชั่วโมง (หรือจนกว่า LIFF จะปิด)
```

### 4.2 การใช้งาน Access Token

```javascript
// ดึง Access Token
const accessToken = liff.getAccessToken();
console.log('Access Token:', accessToken);
// "eyJhbGciOiJIUzI1NiJ9..."

// ใช้เรียก LINE Social API โดยตรง
async function getProfileWithToken() {
    const token = liff.getAccessToken();
    
    const response = await fetch('https://api.line.me/v2/profile', {
        headers: {
            'Authorization': `Bearer ${token}`
        }
    });
    
    if (!response.ok) {
        throw new Error('API call failed');
    }
    
    return await response.json();
}

// ส่ง Token ไป Backend
async function authenticateWithBackend() {
    const accessToken = liff.getAccessToken();
    
    const response = await fetch('/api/auth/line', {
        method: 'POST',
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({ accessToken })
    });
    
    const { sessionToken } = await response.json();
    
    // เก็บ Session Token แทน Access Token
    sessionStorage.setItem('sessionToken', sessionToken);
    
    return sessionToken;
}
```

### 4.3 การ Verify Access Token บน Server

```javascript
// Backend: verify access token
const verifyLineAccessToken = async (accessToken) => {
    // Verify ด้วย LINE API
    const response = await fetch(
        `https://api.line.me/oauth2/v2.1/verify?access_token=${accessToken}`
    );
    
    if (!response.ok) {
        throw new Error('Token verification failed');
    }
    
    const data = await response.json();
    
    // ตรวจสอบ client_id ตรงกับ Channel ID
    if (data.client_id !== process.env.LINE_CHANNEL_ID) {
        throw new Error('Token belongs to different channel');
    }
    
    return data;
};
```

---

## 5. liff.getIDToken()

### 5.1 ID Token คืออะไร?

```
ID Token คือ JWT (JSON Web Token) ที่:
- มีข้อมูล User ฝังอยู่ใน Payload
- ลงนาม (Sign) ด้วย Channel Secret
- ใช้สำหรับ Verify บน Server โดยไม่ต้องเรียก API เพิ่ม
- มีอายุสั้น (ควรใช้ทันที)

โครงสร้าง JWT:
Header.Payload.Signature
```

### 5.2 โครงสร้าง ID Token

```javascript
// ตัวอย่าง Decoded ID Token
const decodedPayload = {
    iss: "https://access.line.me",          // Issuer
    sub: "U4af4980629...",                  // User ID
    aud: "1234567890",                      // Channel ID (aud = audience)
    exp: 1698765432,                        // Expiry timestamp
    iat: 1698761832,                        // Issued at timestamp
    nonce: "abc123",                        // Nonce (ถ้าใช้)
    
    // Scope: profile
    name: "สมชาย ใจดี",                   // Display Name
    picture: "https://profile.line-scdn.net/...",
    
    // Scope: email (ถ้าได้รับ approval)
    email: "user@example.com"
};
```

### 5.3 การใช้งาน getIDToken

```javascript
// ดึง ID Token
const idToken = liff.getIDToken();
console.log('ID Token:', idToken);
// "eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJVNGFmNDk4..."

// Decode ดู Payload (ไม่ปลอดภัย - อย่า Trust ค่านี้โดยไม่ Verify)
function decodeJwtPayload(token) {
    try {
        const parts = token.split('.');
        if (parts.length !== 3) throw new Error('Invalid token format');
        
        // Decode base64url
        const payload = parts[1]
            .replace(/-/g, '+')
            .replace(/_/g, '/');
        
        return JSON.parse(atob(payload));
    } catch (error) {
        console.error('Decode error:', error);
        return null;
    }
}

// ⚠️ ใช้สำหรับ Debug เท่านั้น!
// ต้อง Verify บน Server ก่อนใช้งานจริง
const payload = decodeJwtPayload(idToken);
console.log('User ID:', payload?.sub);
```

---

## 6. การ Verify Token บน Server

### 6.1 ทำไมต้อง Verify บน Server?

```
Client-side JavaScript สามารถถูกแก้ไขได้
→ ข้อมูลจาก Client ไม่น่าเชื่อถือ
→ ต้อง Verify Token บน Server เสมอ

Security Rule:
❌ อย่า Trust User ID จาก Client โดยตรง
✅ รับ Token → Verify บน Server → ใช้ User ID จาก Verified Result
```

### 6.2 Verify ด้วย LINE Verify API

```javascript
// server/routes/auth.js
const express = require('express');
const axios = require('axios');
const router = express.Router();

// Verify Access Token
router.post('/verify-access-token', async (req, res) => {
    const { accessToken } = req.body;
    
    if (!accessToken) {
        return res.status(400).json({ error: 'Access token required' });
    }
    
    try {
        // เรียก LINE API เพื่อ Verify
        const verifyResponse = await axios.get(
            `https://api.line.me/oauth2/v2.1/verify`,
            {
                params: { access_token: accessToken }
            }
        );
        
        const tokenInfo = verifyResponse.data;
        
        // ตรวจสอบ Channel ID
        if (tokenInfo.client_id !== process.env.LINE_CHANNEL_ID) {
            return res.status(401).json({ error: 'Token belongs to different channel' });
        }
        
        // ตรวจสอบ Expiry
        if (tokenInfo.expires_in <= 0) {
            return res.status(401).json({ error: 'Token expired' });
        }
        
        // ดึง Profile ด้วย Token
        const profileResponse = await axios.get(
            'https://api.line.me/v2/profile',
            {
                headers: { Authorization: `Bearer ${accessToken}` }
            }
        );
        
        const profile = profileResponse.data;
        
        res.json({
            verified: true,
            userId: profile.userId,
            displayName: profile.displayName,
            pictureUrl: profile.pictureUrl
        });
        
    } catch (error) {
        console.error('Verify error:', error.response?.data || error.message);
        res.status(401).json({ error: 'Token verification failed' });
    }
});

module.exports = router;
```

### 6.3 Verify ID Token

```javascript
// server/routes/auth.js - Verify ID Token

router.post('/verify-id-token', async (req, res) => {
    const { idToken } = req.body;
    
    if (!idToken) {
        return res.status(400).json({ error: 'ID token required' });
    }
    
    try {
        // Verify ด้วย LINE API
        const verifyResponse = await axios.post(
            'https://api.line.me/oauth2/v2.1/verify',
            new URLSearchParams({
                id_token: idToken,
                client_id: process.env.LINE_CHANNEL_ID
            }),
            {
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded'
                }
            }
        );
        
        const payload = verifyResponse.data;
        
        // ตรวจสอบ aud (audience) ตรงกับ Channel ID
        if (payload.aud !== process.env.LINE_CHANNEL_ID) {
            return res.status(401).json({ error: 'Invalid audience' });
        }
        
        // ตรวจสอบ iss (issuer)
        if (payload.iss !== 'https://access.line.me') {
            return res.status(401).json({ error: 'Invalid issuer' });
        }
        
        res.json({
            verified: true,
            userId: payload.sub,
            displayName: payload.name,
            pictureUrl: payload.picture,
            email: payload.email || null,
            issuedAt: new Date(payload.iat * 1000),
            expiresAt: new Date(payload.exp * 1000)
        });
        
    } catch (error) {
        console.error('ID token verify error:', error.response?.data || error.message);
        res.status(401).json({ error: 'ID token verification failed' });
    }
});
```

---

## 7. การตรวจสอบ Login State

### 7.1 liff.isLoggedIn()

```javascript
// ตรวจสอบก่อน getProfile
if (liff.isLoggedIn()) {
    const profile = await liff.getProfile();
    // ใช้งาน profile ได้
} else {
    // ยังไม่ได้ Login
    liff.login();
}

// Pattern ที่ดี: ตรวจสอบทุกครั้งก่อนใช้ Protected APIs
async function safeGetProfile() {
    // 1. ตรวจสอบว่า Init แล้ว
    // (liff.init() ต้องเรียกก่อน)
    
    // 2. ตรวจสอบว่า Login แล้ว
    if (!liff.isLoggedIn()) {
        throw new Error('USER_NOT_LOGGED_IN');
    }
    
    // 3. ดึง Profile
    return await liff.getProfile();
}
```

### 7.2 Handling Login State Changes

```javascript
class AuthManager {
    constructor() {
        this.profile = null;
        this.listeners = [];
    }
    
    async init(liffId) {
        await liff.init({ liffId });
        
        if (liff.isLoggedIn()) {
            await this.loadProfile();
        }
    }
    
    async loadProfile() {
        try {
            this.profile = await liff.getProfile();
            this.notifyListeners('login', this.profile);
        } catch (error) {
            console.error('Load profile error:', error);
        }
    }
    
    login() {
        liff.login({ redirectUri: window.location.href });
    }
    
    logout() {
        liff.logout();
        this.profile = null;
        this.notifyListeners('logout', null);
    }
    
    isLoggedIn() {
        return liff.isLoggedIn();
    }
    
    getProfile() {
        return this.profile;
    }
    
    onAuthChange(callback) {
        this.listeners.push(callback);
    }
    
    notifyListeners(event, data) {
        this.listeners.forEach(cb => cb(event, data));
    }
}

const auth = new AuthManager();

// ใช้งาน
auth.onAuthChange((event, profile) => {
    if (event === 'login') {
        console.log('User logged in:', profile.displayName);
        renderProfile(profile);
    } else if (event === 'logout') {
        console.log('User logged out');
        renderLoginScreen();
    }
});

await auth.init('YOUR_LIFF_ID');
```

---

## 8. liff.login() และ liff.logout()

### 8.1 liff.login()

```javascript
// Login พื้นฐาน
liff.login();

// Login พร้อม Redirect URL
liff.login({
    redirectUri: 'https://your-app.com/liff/?page=profile'
});

// Login และ Redirect ไปหน้าเดิม
liff.login({
    redirectUri: window.location.href
});

// กำหนด State สำหรับ CSRF Protection
liff.login({
    state: 'random_csrf_token_here'
});
```

### 8.2 liff.logout()

```javascript
// Logout ธรรมดา
liff.logout();

// Logout แล้ว Reload หน้า
function logoutAndReload() {
    liff.logout();
    window.location.reload();
}

// Logout พร้อม Cleanup
async function fullLogout() {
    try {
        // 1. Clear local data
        localStorage.removeItem('userPreferences');
        sessionStorage.clear();
        
        // 2. แจ้ง Backend (Optional)
        await fetch('/api/auth/logout', {
            method: 'POST',
            headers: { 'Authorization': `Bearer ${sessionStorage.getItem('token')}` }
        });
        
        // 3. Logout จาก LIFF
        liff.logout();
        
        // 4. Redirect ไปหน้า Login
        window.location.href = '/login';
    } catch (error) {
        console.error('Logout error:', error);
        // Force logout ถ้าเกิด error
        liff.logout();
        window.location.reload();
    }
}
```

### 8.3 Login/Logout UI Component

```html
<!-- Auth Component -->
<div id="authSection">
    <!-- แสดงเมื่อยังไม่ Login -->
    <div id="loginSection" style="display:none">
        <div style="text-align:center; padding:40px 20px">
            <div style="font-size:64px; margin-bottom:16px">👤</div>
            <h2 style="margin-bottom:8px">เข้าสู่ระบบ</h2>
            <p style="color:#666; margin-bottom:24px">
                Login ด้วย LINE เพื่อเข้าถึงฟีเจอร์ทั้งหมด
            </p>
            <button id="loginBtn" 
                    style="background:#06c755;color:white;border:none;
                           padding:16px 40px;border-radius:12px;
                           font-size:16px;font-weight:700;cursor:pointer;">
                Login ด้วย LINE
            </button>
        </div>
    </div>
    
    <!-- แสดงเมื่อ Login แล้ว -->
    <div id="userSection" style="display:none">
        <div style="display:flex;align-items:center;gap:12px;padding:16px">
            <img id="userAvatar" 
                 style="width:48px;height:48px;border-radius:50%;object-fit:cover"
                 src="" alt="Avatar">
            <div style="flex:1">
                <div id="userName" style="font-weight:700"></div>
                <div style="font-size:12px;color:#666">LINE Member</div>
            </div>
            <button id="logoutBtn"
                    style="background:none;border:1px solid #ccc;
                           padding:8px 16px;border-radius:8px;cursor:pointer;
                           font-size:13px;color:#666">
                Logout
            </button>
        </div>
    </div>
</div>

<script>
    document.getElementById('loginBtn').addEventListener('click', () => {
        liff.login({ redirectUri: window.location.href });
    });
    
    document.getElementById('logoutBtn').addEventListener('click', () => {
        if (confirm('ต้องการ Logout หรือไม่?')) {
            liff.logout();
            window.location.reload();
        }
    });
    
    async function updateAuthUI() {
        if (liff.isLoggedIn()) {
            const profile = await liff.getProfile();
            
            document.getElementById('loginSection').style.display = 'none';
            document.getElementById('userSection').style.display = 'block';
            document.getElementById('userAvatar').src = profile.pictureUrl || '';
            document.getElementById('userName').textContent = profile.displayName;
        } else {
            document.getElementById('loginSection').style.display = 'block';
            document.getElementById('userSection').style.display = 'none';
        }
    }
    
    // เรียกหลัง liff.init()
    updateAuthUI();
</script>
```

---

## 9. การจัดการ Unauthenticated State

### 9.1 Patterns สำหรับ Unauthenticated Users

```javascript
// Pattern 1: Auto Login (แนะนำสำหรับ LIFF ใน LINE App)
async function initWithAutoLogin(liffId) {
    await liff.init({
        liffId,
        withLoginOnExternalBrowser: true // Auto login ใน browser ด้วย
    });
    
    // ถ้าเปิดใน LINE App จะ Login อัตโนมัติ
    if (!liff.isLoggedIn()) {
        // เฉพาะกรณีเปิดจาก External Browser
        liff.login();
        return; // หยุดรอ Redirect กลับมา
    }
    
    // ถึงตรงนี้ = Login แล้ว
    const profile = await liff.getProfile();
    renderApp(profile);
}

// Pattern 2: Show Login Prompt
async function initWithLoginPrompt(liffId) {
    await liff.init({ liffId });
    
    if (!liff.isLoggedIn()) {
        renderLoginPrompt(); // แสดงหน้า Login
        return;
    }
    
    const profile = await liff.getProfile();
    renderApp(profile);
}

// Pattern 3: Allow Guest Access
async function initWithGuestAccess(liffId) {
    await liff.init({ liffId });
    
    if (!liff.isLoggedIn()) {
        renderGuestMode(); // แสดงแบบ Guest (ฟีเจอร์จำกัด)
        return;
    }
    
    const profile = await liff.getProfile();
    renderFullApp(profile);
}
```

### 9.2 Protected Route Pattern

```javascript
// Middleware สำหรับ Protected Features
function requireLogin(action) {
    return async function(...args) {
        if (!liff.isLoggedIn()) {
            const confirmed = confirm(
                'ฟีเจอร์นี้ต้องการ Login ก่อน\nต้องการ Login หรือไม่?'
            );
            if (confirmed) {
                liff.login({ redirectUri: window.location.href });
            }
            return;
        }
        return await action(...args);
    };
}

// ใช้งาน
const safeAddToCart = requireLogin(async (productId) => {
    // ถึงตรงนี้ = Login แล้ว
    const profile = await liff.getProfile();
    await addProductToCart(productId, profile.userId);
    liff.sendMessages([{ type: 'text', text: 'เพิ่มลงตะกร้าแล้ว!' }]);
});

// เรียกใช้ - ถ้ายังไม่ Login จะ prompt
document.getElementById('addToCartBtn').addEventListener('click', () => {
    safeAddToCart('PRODUCT_001');
});
```

---

## 10. User Registration ครั้งแรก

### 10.1 First-Time User Flow

```
ขั้นตอน:
1. User เปิด LIFF
2. LIFF Init + Auto Login
3. ส่ง ID Token ไป Backend
4. Backend Verify Token
5. ตรวจสอบ Database ว่ามี User นี้แล้วหรือไม่
6. ถ้าไม่มี → สร้าง User ใหม่ (Registration)
7. ถ้ามี → Update Profile (Profile อาจเปลี่ยน)
8. Return Session Token
9. Frontend ใช้ Session Token สำหรับ API Calls ต่อไป
```

### 10.2 Backend: User Registration

```javascript
// server/services/userService.js
const db = require('../db'); // Database connection

class UserService {
    
    // หา User หรือสร้างใหม่
    async findOrCreate(lineProfile) {
        const { userId, displayName, pictureUrl } = lineProfile;
        
        // ค้นหา User
        let user = await db.users.findOne({ 
            where: { lineUserId: userId } 
        });
        
        if (!user) {
            // สร้าง User ใหม่
            user = await db.users.create({
                lineUserId: userId,
                displayName: displayName,
                pictureUrl: pictureUrl,
                registeredAt: new Date(),
                lastLoginAt: new Date(),
                isActive: true
            });
            
            console.log(`New user registered: ${displayName} (${userId})`);
            
            // ส่ง Welcome Message (Optional)
            await this.sendWelcomeMessage(userId);
            
        } else {
            // Update Profile ถ้าเปลี่ยน
            const updates = {};
            
            if (user.displayName !== displayName) {
                updates.displayName = displayName;
            }
            if (user.pictureUrl !== pictureUrl) {
                updates.pictureUrl = pictureUrl;
            }
            
            if (Object.keys(updates).length > 0) {
                updates.lastLoginAt = new Date();
                await user.update(updates);
            }
        }
        
        return user;
    }
    
    async sendWelcomeMessage(userId) {
        const line = require('../lineClient');
        
        try {
            await line.pushMessage({
                to: userId,
                messages: [{
                    type: 'text',
                    text: `ยินดีต้อนรับสู่บริการของเรา! 🎉\n` +
                          `คุณได้ลงทะเบียนเรียบร้อยแล้ว`
                }]
            });
        } catch (error) {
            console.error('Welcome message error:', error);
        }
    }
}

module.exports = new UserService();
```

### 10.3 Backend: Auth Route

```javascript
// server/routes/auth.js
const express = require('express');
const axios = require('axios');
const jwt = require('jsonwebtoken');
const userService = require('../services/userService');
const router = express.Router();

router.post('/liff-login', async (req, res) => {
    const { idToken } = req.body;
    
    if (!idToken) {
        return res.status(400).json({ 
            success: false, 
            error: 'ID token required' 
        });
    }
    
    try {
        // Step 1: Verify ID Token กับ LINE
        const verifyResponse = await axios.post(
            'https://api.line.me/oauth2/v2.1/verify',
            new URLSearchParams({
                id_token: idToken,
                client_id: process.env.LINE_CHANNEL_ID
            }),
            {
                headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
            }
        );
        
        const linePayload = verifyResponse.data;
        
        // Step 2: Validate Token
        if (linePayload.aud !== process.env.LINE_CHANNEL_ID) {
            return res.status(401).json({ 
                success: false, 
                error: 'Invalid token audience' 
            });
        }
        
        // Step 3: Find or Create User
        const user = await userService.findOrCreate({
            userId: linePayload.sub,
            displayName: linePayload.name,
            pictureUrl: linePayload.picture,
            email: linePayload.email
        });
        
        // Step 4: Create Session Token (JWT)
        const sessionToken = jwt.sign(
            {
                userId: user.id,
                lineUserId: user.lineUserId,
                displayName: user.displayName
            },
            process.env.JWT_SECRET,
            { expiresIn: '7d' }
        );
        
        // Step 5: Return Success
        res.json({
            success: true,
            token: sessionToken,
            user: {
                id: user.id,
                displayName: user.displayName,
                pictureUrl: user.pictureUrl,
                isNewUser: user.registeredAt === user.updatedAt
            }
        });
        
    } catch (error) {
        console.error('Login error:', error.response?.data || error.message);
        res.status(401).json({ 
            success: false, 
            error: 'Authentication failed' 
        });
    }
});

module.exports = router;
```

---

## 11. JWT Verification

### 11.1 ติดตั้ง Dependencies

```bash
npm install jsonwebtoken axios dotenv
```

### 11.2 JWT Middleware

```javascript
// server/middleware/auth.js
const jwt = require('jsonwebtoken');

// Middleware: ตรวจสอบ Session Token
function authenticateToken(req, res, next) {
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1]; // "Bearer TOKEN"
    
    if (!token) {
        return res.status(401).json({ 
            success: false, 
            error: 'Access token required' 
        });
    }
    
    try {
        const decoded = jwt.verify(token, process.env.JWT_SECRET);
        req.user = decoded; // เพิ่ม user info ใน request
        next();
    } catch (error) {
        if (error.name === 'TokenExpiredError') {
            return res.status(401).json({ 
                success: false, 
                error: 'Token expired' 
            });
        }
        return res.status(403).json({ 
            success: false, 
            error: 'Invalid token' 
        });
    }
}

module.exports = { authenticateToken };
```

### 11.3 Protected Routes

```javascript
// server/routes/api.js
const express = require('express');
const { authenticateToken } = require('../middleware/auth');
const router = express.Router();

// Protected: ดูโปรไฟล์
router.get('/profile', authenticateToken, async (req, res) => {
    // req.user มีข้อมูลจาก JWT
    const { userId, lineUserId, displayName } = req.user;
    
    const user = await db.users.findById(userId);
    
    res.json({
        success: true,
        profile: {
            id: user.id,
            displayName: user.displayName,
            pictureUrl: user.pictureUrl,
            email: user.email,
            registeredAt: user.registeredAt
        }
    });
});

// Protected: อัพเดตโปรไฟล์
router.put('/profile', authenticateToken, async (req, res) => {
    const { userId } = req.user;
    const { phone, address } = req.body;
    
    await db.users.update(
        { phone, address },
        { where: { id: userId } }
    );
    
    res.json({ success: true, message: 'Profile updated' });
});

module.exports = router;
```

---

## 12. Complete Authentication Flow

### 12.1 Frontend: Complete Auth Module

```javascript
// liff-auth.js - Complete Authentication Module

class LiffAuth {
    constructor({ liffId, apiBaseUrl }) {
        this.liffId = liffId;
        this.apiBaseUrl = apiBaseUrl;
        this.sessionToken = null;
        this.user = null;
        this._initialized = false;
    }
    
    // Initialize และ Auto Login
    async init() {
        if (this._initialized) return;
        
        try {
            // 1. Init LIFF
            await liff.init({ 
                liffId: this.liffId,
                withLoginOnExternalBrowser: true
            });
            
            this._initialized = true;
            
            // 2. ถ้าไม่ได้ Login ให้ Login
            if (!liff.isLoggedIn()) {
                liff.login({ redirectUri: window.location.href });
                return; // Wait for redirect
            }
            
            // 3. แลก LIFF Token กับ Session Token
            await this.authenticate();
            
        } catch (error) {
            console.error('Init error:', error);
            throw error;
        }
    }
    
    // แลก ID Token กับ Session Token
    async authenticate() {
        // ตรวจสอบ Session Token ที่บันทึกไว้
        const existingToken = this.getStoredToken();
        if (existingToken && this.isTokenValid(existingToken)) {
            this.sessionToken = existingToken;
            this.user = this.decodeToken(existingToken);
            return;
        }
        
        // ขอ Session Token ใหม่จาก Backend
        const idToken = liff.getIDToken();
        
        const response = await fetch(`${this.apiBaseUrl}/auth/liff-login`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ idToken })
        });
        
        if (!response.ok) {
            const error = await response.json();
            throw new Error(error.error || 'Authentication failed');
        }
        
        const { token, user } = await response.json();
        
        // เก็บ Token
        this.sessionToken = token;
        this.user = user;
        this.storeToken(token);
        
        console.log('Authenticated:', user.displayName);
    }
    
    // HTTP Request พร้อม Auth Header
    async request(path, options = {}) {
        if (!this.sessionToken) {
            throw new Error('Not authenticated');
        }
        
        const response = await fetch(`${this.apiBaseUrl}${path}`, {
            ...options,
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${this.sessionToken}`,
                ...options.headers
            }
        });
        
        // ถ้า Token หมดอายุ
        if (response.status === 401) {
            this.clearToken();
            await this.authenticate(); // ขอ Token ใหม่
            return this.request(path, options); // Retry
        }
        
        return response;
    }
    
    // Convenience methods
    async get(path) {
        const res = await this.request(path);
        return res.json();
    }
    
    async post(path, data) {
        const res = await this.request(path, {
            method: 'POST',
            body: JSON.stringify(data)
        });
        return res.json();
    }
    
    // Token Storage
    storeToken(token) {
        try {
            sessionStorage.setItem('liff_session_token', token);
        } catch (e) {}
    }
    
    getStoredToken() {
        try {
            return sessionStorage.getItem('liff_session_token');
        } catch (e) {
            return null;
        }
    }
    
    clearToken() {
        try {
            sessionStorage.removeItem('liff_session_token');
        } catch (e) {}
        this.sessionToken = null;
        this.user = null;
    }
    
    // Token Validation
    isTokenValid(token) {
        try {
            const payload = this.decodeToken(token);
            // Check expiry (ให้ Margin 5 นาที)
            return payload.exp > (Date.now() / 1000) + 300;
        } catch (e) {
            return false;
        }
    }
    
    decodeToken(token) {
        const payload = token.split('.')[1]
            .replace(/-/g, '+')
            .replace(/_/g, '/');
        return JSON.parse(atob(payload));
    }
    
    // Public getters
    getUser() { return this.user; }
    getToken() { return this.sessionToken; }
    isAuthenticated() { return !!this.sessionToken && !!this.user; }
    
    // Logout
    logout() {
        this.clearToken();
        liff.logout();
        window.location.reload();
    }
}

// การใช้งาน
const auth = new LiffAuth({
    liffId: 'YOUR_LIFF_ID',
    apiBaseUrl: 'https://your-api.com/api'
});

// Main App
async function main() {
    const loadingEl = document.getElementById('loading');
    const appEl = document.getElementById('app');
    
    try {
        // Init และ Authenticate
        await auth.init();
        
        // ได้รับ User Info
        const user = auth.getUser();
        console.log('Welcome:', user?.displayName);
        
        // เรียก Protected API
        const profile = await auth.get('/profile');
        console.log('Profile:', profile);
        
        // แสดง App
        loadingEl.style.display = 'none';
        appEl.style.display = 'block';
        
        renderApp(user, profile);
        
    } catch (error) {
        console.error('App init error:', error);
        loadingEl.innerHTML = `
            <p style="color:red">เกิดข้อผิดพลาด: ${error.message}</p>
            <button onclick="window.location.reload()">ลองใหม่</button>
        `;
    }
}

function renderApp(user, profile) {
    const appEl = document.getElementById('app');
    appEl.innerHTML = `
        <div style="padding:20px">
            <img src="${user?.pictureUrl || ''}" 
                 style="width:80px;height:80px;border-radius:50%" alt="Avatar">
            <h2>สวัสดี, ${user?.displayName}!</h2>
            <p>ยินดีต้อนรับ${user?.isNewUser ? ' (สมาชิกใหม่)' : ''}</p>
            <button onclick="auth.logout()">Logout</button>
        </div>
    `;
}

main();
```

### 12.2 Complete Server Setup

```javascript
// server/index.js - Complete Server

require('dotenv').config();
const express = require('express');
const cors = require('cors');
const path = require('path');
const authRouter = require('./routes/auth');
const apiRouter = require('./routes/api');

const app = express();

// Middleware
app.use(cors({
    origin: process.env.FRONTEND_URL || '*',
    credentials: true
}));
app.use(express.json());
app.use(express.static(path.join(__dirname, '../public')));

// Routes
app.use('/api/auth', authRouter);
app.use('/api', apiRouter);

// Health Check
app.get('/health', (req, res) => {
    res.json({ status: 'ok', timestamp: new Date() });
});

// SPA Fallback
app.get('*', (req, res) => {
    res.sendFile(path.join(__dirname, '../public/index.html'));
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
    console.log(`Server running on port ${PORT}`);
});
```

```bash
# .env file
LINE_CHANNEL_ID=1234567890
LINE_CHANNEL_SECRET=your_channel_secret_here
LINE_ACCESS_TOKEN=your_access_token_here
JWT_SECRET=your_very_secure_random_secret_here_at_least_32_chars
PORT=3000
FRONTEND_URL=https://your-app.com
DATABASE_URL=postgresql://user:pass@localhost:5432/mydb
```

---

## 13. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Basic Profile Page

สร้าง LIFF App ที่:
1. แสดง Loading Screen ระหว่าง Init
2. ถ้ายังไม่ Login แสดงปุ่ม Login
3. เมื่อ Login แล้ว แสดง Profile พร้อม Avatar
4. มีปุ่ม Logout

### แบบฝึกหัดที่ 2: Token Inspector

สร้าง LIFF App ที่แสดง:
1. Access Token (Masked)
2. ID Token (Decoded Payload)
3. การ Verify Token ผ่าน API
4. ปุ่ม Copy Token

**Hint:**
```javascript
// Decode JWT Payload
function decodeJwt(token) {
    const base64Url = token.split('.')[1];
    const base64 = base64Url.replace(/-/g, '+').replace(/_/g, '/');
    return JSON.parse(window.atob(base64));
}
```

### แบบฝึกหัดที่ 3: Registration Flow

สร้างระบบ Full Authentication:
1. **Frontend**: LIFF App ส่ง ID Token ไป Backend
2. **Backend**: Verify Token + สร้าง User ในฐานข้อมูล
3. **Backend**: Return JWT Session Token
4. **Frontend**: ใช้ Session Token เรียก Protected API
5. แสดงหน้า Welcome สำหรับ New User

---

## สรุป

| หัวข้อ | API / Method |
|--------|-------------|
| ตรวจ Login | `liff.isLoggedIn()` |
| Login | `liff.login()` |
| Logout | `liff.logout()` |
| ดึง Profile | `liff.getProfile()` |
| รับ Access Token | `liff.getAccessToken()` |
| รับ ID Token | `liff.getIDToken()` |
| Verify Token | LINE Verify API |
| User Registration | Backend + JWT |

### Key Security Points:
1. **ตรวจสอบ Token บน Server เสมอ** - Client-side ไม่น่าเชื่อถือ
2. **ไม่เก็บ Access Token** - ใช้ทันที หรือแลกเป็น Session Token
3. **ตรวจสอบ Channel ID** - ป้องกัน Token ของ App อื่น
4. **ใช้ HTTPS เท่านั้น** - ทั้ง Frontend และ Backend

### ขั้นตอนต่อไป
- **Part 43**: LIFF Send Messages - liff.sendMessages(), shareTargetPicker()
