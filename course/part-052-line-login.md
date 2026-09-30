# Part 52: LINE Login

## สารบัญ

1. [LINE Login Overview](#line-login-overview)
2. [OAuth 2.0 และ OpenID Connect](#oauth-20-และ-openid-connect)
3. [การสร้าง LINE Login Channel](#การสร้าง-line-login-channel)
4. [Web Login Flow](#web-login-flow)
5. [Mobile Login Flow](#mobile-login-flow)
6. [Scopes ที่รองรับ](#scopes-ที่รองรับ)
7. [Access Token และ ID Token](#access-token-และ-id-token)
8. [Verifying ID Token](#verifying-id-token)
9. [Refreshing Tokens](#refreshing-tokens)
10. [Revoking Tokens](#revoking-tokens)
11. [Social Login Button](#social-login-button)
12. [Node.js/Express LINE Login](#nodejsexpress-line-login)
13. [React + Node.js Backend](#react--nodejs-backend)
14. [Session Management](#session-management)
15. [Linking LINE Login กับบัญชีที่มีอยู่](#linking-line-login-กับบัญชีที่มีอยู่)
16. [Security Best Practices](#security-best-practices)

---

## 1. LINE Login Overview

LINE Login ช่วยให้ผู้ใช้สามารถ Login เข้าเว็บไซต์หรือแอปของคุณด้วยบัญชี LINE โดยไม่ต้องสมัครสมาชิกใหม่

### ประโยชน์ของ LINE Login

```
┌─────────────────────────────────────────────────────────┐
│                   LINE Login Benefits                    │
├─────────────────────────────────────────────────────────┤
│ ✓ ผู้ใช้ไม่ต้องจำ Password ใหม่                          │
│ ✓ ได้ข้อมูล Profile, Email, Phone จาก LINE              │
│ ✓ เชื่อมต่อกับ LINE OA ได้โดยตรง                        │
│ ✓ รองรับทั้ง Web และ Mobile                              │
│ ✓ ระบบ Social Login ที่ผู้ใช้ไทยคุ้นเคย                 │
└─────────────────────────────────────────────────────────┘
```

### LINE Login vs LINE OA Messaging API

| ฟีเจอร์ | LINE Login | Messaging API |
|---------|------------|---------------|
| วัตถุประสงค์ | Authentication | Bot/Chat |
| Channel Type | LINE Login Channel | Messaging API Channel |
| ข้อมูลที่ได้ | Profile, Email, Phone | Profile เท่านั้น |
| User Consent | ผ่าน OAuth | ผ่านการ Follow OA |
| Token | Access Token + ID Token | Channel Access Token |

---

## 2. OAuth 2.0 และ OpenID Connect

LINE Login ใช้ทั้ง OAuth 2.0 และ OpenID Connect (OIDC)

### OAuth 2.0 Authorization Code Flow

```
┌──────────┐          ┌──────────────┐          ┌──────────────┐
│  Browser │          │  Your Server │          │  LINE Server │
└────┬─────┘          └──────┬───────┘          └──────┬───────┘
     │                       │                          │
     │  1. กด "Login ด้วย LINE"│                          │
     │──────────────────────>│                          │
     │                       │                          │
     │  2. Redirect ไป LINE   │                          │
     │<──────────────────────│                          │
     │                       │                          │
     │  3. LINE Login Page   │                          │
     │─────────────────────────────────────────────────>│
     │                       │                          │
     │  4. User กด Authorize │                          │
     │<─────────────────────────────────────────────────│
     │                       │                          │
     │  5. Redirect + code   │                          │
     │──────────────────────>│                          │
     │                       │                          │
     │                       │ 6. แลก code เอา Token    │
     │                       │─────────────────────────>│
     │                       │                          │
     │                       │ 7. Access + ID Token     │
     │                       │<─────────────────────────│
     │                       │                          │
     │  8. Session + Cookie  │                          │
     │<──────────────────────│                          │
     │                       │                          │
```

### OpenID Connect Endpoints

```
Authorization Endpoint:
https://access.line.me/oauth2/v2.1/authorize

Token Endpoint:
https://api.line.me/oauth2/v2.1/token

Userinfo Endpoint:
https://api.line.me/v2/profile

Token Verify Endpoint:
https://api.line.me/oauth2/v2.1/verify

Token Refresh Endpoint:
https://api.line.me/oauth2/v2.1/token

Token Revoke Endpoint:
https://api.line.me/oauth2/v2.1/revoke
```

---

## 3. การสร้าง LINE Login Channel

### ขั้นตอนการสร้าง

1. เข้าไปที่ LINE Developers Console: https://developers.line.biz/
2. เลือก Provider ที่ต้องการ
3. คลิก "Create a new channel"
4. เลือก "LINE Login"
5. กรอกข้อมูล:
   - **Channel name**: ชื่อ Channel
   - **Channel description**: คำอธิบาย
   - **App types**: Web app / Mobile app
   - **Email address**: อีเมลติดต่อ
6. ยอมรับ Terms of Use

### ตั้งค่า Callback URL

```
Callback URL ต้องตั้งค่าให้ถูกต้อง:

Development:
http://localhost:3000/auth/line/callback

Production:
https://yourdomain.com/auth/line/callback
```

### ข้อมูลที่ต้องเก็บ

```env
# .env file
LINE_LOGIN_CHANNEL_ID=1234567890
LINE_LOGIN_CHANNEL_SECRET=abcdef1234567890abcdef1234567890
LINE_LOGIN_CALLBACK_URL=https://yourdomain.com/auth/line/callback
```

---

## 4. Web Login Flow

### 4.1 สร้าง Authorization URL

```javascript
const crypto = require('crypto');
const querystring = require('querystring');

function generateLineLoginUrl(options = {}) {
  const {
    channelId = process.env.LINE_LOGIN_CHANNEL_ID,
    callbackUrl = process.env.LINE_LOGIN_CALLBACK_URL,
    scopes = ['profile', 'openid'],
    state = null,
    nonce = null,
    prompt = null, // 'consent' เพื่อบังคับ consent screen
    uiLocales = 'th', // ภาษา UI
    botPrompt = null, // 'aggressive' หรือ 'normal' - เชิญ Follow OA
    initialAmrDisplay = null,
    switchAmr = false
  } = options;

  // สร้าง state แบบ random เพื่อป้องกัน CSRF
  const stateValue = state || crypto.randomBytes(16).toString('hex');
  // สร้าง nonce สำหรับ ID Token verification
  const nonceValue = nonce || crypto.randomBytes(16).toString('hex');

  const params = {
    response_type: 'code',
    client_id: channelId,
    redirect_uri: callbackUrl,
    state: stateValue,
    scope: scopes.join('%20'),
    nonce: nonceValue,
    ui_locales: uiLocales
  };

  if (prompt) params.prompt = prompt;
  if (botPrompt) params.bot_prompt = botPrompt;
  if (initialAmrDisplay) params.initial_amr_display = initialAmrDisplay;
  if (switchAmr) params.switch_amr = 'true';

  const queryString = Object.entries(params)
    .map(([key, value]) => `${key}=${value}`)
    .join('&');

  return {
    url: `https://access.line.me/oauth2/v2.1/authorize?${queryString}`,
    state: stateValue,
    nonce: nonceValue
  };
}

// ตัวอย่าง:
const loginData = generateLineLoginUrl({
  scopes: ['profile', 'openid', 'email'],
  botPrompt: 'aggressive' // เชิญ Follow OA หลัง Login
});

console.log(loginData.url);
// https://access.line.me/oauth2/v2.1/authorize?response_type=code&...
```

### 4.2 Handle Callback

```javascript
const express = require('express');
const axios = require('axios');
const router = express.Router();

// Route สำหรับ redirect ไป LINE
router.get('/auth/line', (req, res) => {
  const loginData = generateLineLoginUrl({
    scopes: ['profile', 'openid', 'email'],
    botPrompt: 'normal'
  });

  // บันทึก state และ nonce ใน session
  req.session.lineState = loginData.state;
  req.session.lineNonce = loginData.nonce;
  req.session.returnTo = req.query.returnTo || '/';

  res.redirect(loginData.url);
});

// Callback Route
router.get('/auth/line/callback', async (req, res) => {
  const { code, state, error, error_description } = req.query;

  // จัดการ Error
  if (error) {
    console.error('LINE Login error:', error, error_description);
    return res.redirect(`/?error=${encodeURIComponent(error_description)}`);
  }

  // ตรวจสอบ State (CSRF Protection)
  if (state !== req.session.lineState) {
    console.error('State mismatch - possible CSRF attack');
    return res.status(403).send('Invalid state parameter');
  }

  try {
    // แลก code เอา Token
    const tokenData = await exchangeCodeForToken(code);

    // ตรวจสอบ ID Token
    const userInfo = await verifyIdToken(
      tokenData.id_token,
      req.session.lineNonce
    );

    // ดูข้อมูล Profile เพิ่มเติม
    const profile = await getUserProfile(tokenData.access_token);

    // บันทึกหรืออัพเดตผู้ใช้ใน Database
    const user = await findOrCreateUser({
      lineUserId: profile.userId,
      displayName: profile.displayName,
      pictureUrl: profile.pictureUrl,
      email: userInfo.email,
      accessToken: tokenData.access_token,
      refreshToken: tokenData.refresh_token,
      tokenExpiresAt: new Date(Date.now() + tokenData.expires_in * 1000)
    });

    // สร้าง Session
    req.session.userId = user.id;
    req.session.lineUserId = profile.userId;

    // ลบข้อมูลชั่วคราวออกจาก session
    delete req.session.lineState;
    delete req.session.lineNonce;

    const returnTo = req.session.returnTo || '/dashboard';
    delete req.session.returnTo;

    res.redirect(returnTo);
  } catch (error) {
    console.error('LINE Login callback error:', error);
    res.redirect('/?error=login_failed');
  }
});
```

---

## 5. Mobile Login Flow

### iOS (Swift)

LINE มี SDK สำหรับ iOS ที่ใช้ง่าย

```swift
// AppDelegate.swift
import LineSDK

@UIApplicationMain
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ application: UIApplication,
                     didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]?) -> Bool {
        LoginManager.shared.setup(channelID: "1234567890", universalLinkURL: nil)
        return true
    }

    func application(_ app: UIApplication, open url: URL,
                     options: [UIApplication.OpenURLOptionsKey: Any] = [:]) -> Bool {
        return LoginManager.shared.application(app, open: url, options: options)
    }
}

// ViewController.swift
import LineSDK

class LoginViewController: UIViewController {
    @IBAction func loginWithLine(_ sender: Any) {
        LoginManager.shared.login(
            permissions: [.profile, .openID, .email],
            in: self
        ) { result in
            switch result {
            case .success(let loginResult):
                print("User ID:", loginResult.userProfile?.userID ?? "")
                print("Display Name:", loginResult.userProfile?.displayName ?? "")
                print("Access Token:", loginResult.accessToken.value)
                // บันทึก token และ navigate ต่อ
                self.handleLoginSuccess(loginResult)
            case .failure(let error):
                print("Login failed:", error.localizedDescription)
            }
        }
    }

    func handleLoginSuccess(_ result: LoginResult) {
        // ส่ง access token ไปยัง backend
        self.sendTokenToBackend(token: result.accessToken.value)
    }

    func sendTokenToBackend(token: String) {
        let url = URL(string: "https://yourapi.com/auth/line/mobile")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/json", forHTTPHeaderField: "Content-Type")

        let body = ["accessToken": token]
        request.httpBody = try? JSONSerialization.data(withJSONObject: body)

        URLSession.shared.dataTask(with: request) { data, response, error in
            if let data = data {
                // Handle server response
                print(String(data: data, encoding: .utf8) ?? "")
            }
        }.resume()
    }
}
```

### Android (Kotlin)

```kotlin
// MainActivity.kt
import com.linecorp.linesdk.auth.LineAuthenticationConfig
import com.linecorp.linesdk.auth.LineAuthenticationParams
import com.linecorp.linesdk.auth.LineLoginApi
import com.linecorp.linesdk.Scope

class MainActivity : AppCompatActivity() {
    private val LINE_LOGIN_REQUEST = 1001

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val loginButton = findViewById<Button>(R.id.lineLoginButton)
        loginButton.setOnClickListener {
            startLineLogin()
        }
    }

    private fun startLineLogin() {
        val loginIntent = LineLoginApi.getLoginIntent(
            this,
            "1234567890", // Channel ID
            LineAuthenticationParams.Builder()
                .scopes(listOf(Scope.PROFILE, Scope.OPENID_CONNECT, Scope.OC_EMAIL))
                .nonce("your-nonce-value")
                .build()
        )
        startActivityForResult(loginIntent, LINE_LOGIN_REQUEST)
    }

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)

        if (requestCode == LINE_LOGIN_REQUEST) {
            val result = LineLoginApi.getLoginResultFromIntent(data)

            when (result.responseCode) {
                LineApiResponseCode.SUCCESS -> {
                    val userId = result.lineProfile?.userID
                    val displayName = result.lineProfile?.displayName
                    val accessToken = result.lineCredential?.accessToken?.tokenString

                    println("User ID: $userId")
                    println("Name: $displayName")
                    println("Token: $accessToken")

                    // ส่ง token ไป backend
                    sendTokenToBackend(accessToken ?: "")
                }
                LineApiResponseCode.CANCEL -> {
                    println("User cancelled login")
                }
                else -> {
                    println("Login failed: ${result.errorData.message}")
                }
            }
        }
    }

    private fun sendTokenToBackend(token: String) {
        // ใช้ Retrofit หรือ OkHttp ส่งไป backend
    }
}
```

---

## 6. Scopes ที่รองรับ

### รายการ Scopes

| Scope | ข้อมูลที่ได้ | หมายเหตุ |
|-------|------------|----------|
| `profile` | userId, displayName, pictureUrl, statusMessage | จำเป็นต้องมี |
| `openid` | ใช้กับ ID Token | ต้องมีเพื่อดู email, phone |
| `email` | อีเมล | ต้องขอ openid ด้วย |
| `phone` | เบอร์โทรศัพท์ (ต้องยืนยัน) | ต้องขอ openid ด้วย |

### ตัวอย่าง Scope Combinations

```javascript
// Scope พื้นฐาน - แค่ Profile
const basicScopes = ['profile', 'openid'];

// Scope เพิ่มเติม - รวม Email
const emailScopes = ['profile', 'openid', 'email'];

// Scope สูงสุด - รวม Phone
const fullScopes = ['profile', 'openid', 'email', 'phone'];

// ตัวอย่าง URL พร้อม Email scope
const url = `https://access.line.me/oauth2/v2.1/authorize?` +
  `response_type=code&` +
  `client_id=${channelId}&` +
  `redirect_uri=${encodeURIComponent(callbackUrl)}&` +
  `state=${state}&` +
  `scope=profile%20openid%20email&` +
  `nonce=${nonce}`;
```

### ข้อมูลที่ได้จากแต่ละ Scope

```json
// จาก /v2/profile (scope: profile)
{
  "userId": "U4af4980629...",
  "displayName": "สมชาย ใจดี",
  "pictureUrl": "https://profile.line-scdn.net/...",
  "statusMessage": "สวัสดีครับ"
}

// จาก ID Token (scope: openid + email + phone)
{
  "iss": "https://access.line.me",
  "sub": "U4af4980629...",
  "aud": "1234567890",
  "exp": 1735689600,
  "iat": 1735603200,
  "nonce": "abc123",
  "name": "สมชาย ใจดี",
  "picture": "https://profile.line-scdn.net/...",
  "email": "somchai@example.com",
  "phone_number": "+66812345678",
  "phone_number_verified": true,
  "amr": ["linesso"]
}
```

---

## 7. Access Token และ ID Token

### Exchange Code for Token

```javascript
async function exchangeCodeForToken(code) {
  const params = new URLSearchParams({
    grant_type: 'authorization_code',
    code: code,
    redirect_uri: process.env.LINE_LOGIN_CALLBACK_URL,
    client_id: process.env.LINE_LOGIN_CHANNEL_ID,
    client_secret: process.env.LINE_LOGIN_CHANNEL_SECRET
  });

  try {
    const response = await axios.post(
      'https://api.line.me/oauth2/v2.1/token',
      params.toString(),
      {
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
      }
    );

    return response.data;
  } catch (error) {
    console.error('Token exchange failed:', error.response?.data);
    throw error;
  }
}
```

### Token Response

```json
{
  "access_token": "eyJhbGciOiJIUzI1NiJ9...",
  "expires_in": 2592000,
  "id_token": "eyJhbGciOiJIUzI1NiJ9...",
  "refresh_token": "eyJhbGciOiJIUzI1NiJ9...",
  "scope": "openid profile email",
  "token_type": "Bearer"
}
```

### Get User Profile

```javascript
async function getUserProfile(accessToken) {
  try {
    const response = await axios.get('https://api.line.me/v2/profile', {
      headers: { 'Authorization': `Bearer ${accessToken}` }
    });
    return response.data;
  } catch (error) {
    console.error('Get profile failed:', error.response?.data);
    throw error;
  }
}

// Profile Response
/*
{
  "userId": "U4af4980629...",
  "displayName": "สมชาย ใจดี",
  "pictureUrl": "https://profile.line-scdn.net/...",
  "statusMessage": "สวัสดีครับ"
}
*/
```

---

## 8. Verifying ID Token

การ Verify ID Token สำคัญมากเพื่อความปลอดภัย

### Verify กับ LINE API

```javascript
async function verifyIdToken(idToken, expectedNonce = null) {
  const params = new URLSearchParams({
    id_token: idToken,
    client_id: process.env.LINE_LOGIN_CHANNEL_ID
  });

  if (expectedNonce) {
    params.append('nonce', expectedNonce);
  }

  try {
    const response = await axios.post(
      'https://api.line.me/oauth2/v2.1/verify',
      params.toString(),
      {
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
      }
    );

    return response.data;
  } catch (error) {
    const errorData = error.response?.data;
    throw new Error(`ID Token verification failed: ${errorData?.error_description || error.message}`);
  }
}
```

### Verify ด้วย JWT Library (ตรวจสอบ Local)

```javascript
const jwt = require('jsonwebtoken');
const axios = require('axios');

// ดึง Public Key จาก LINE JWKS Endpoint
let cachedJwks = null;
let jwksCacheTime = 0;
const JWKS_CACHE_TTL = 3600000; // 1 ชั่วโมง

async function getLineJwks() {
  if (cachedJwks && Date.now() - jwksCacheTime < JWKS_CACHE_TTL) {
    return cachedJwks;
  }

  const response = await axios.get('https://api.line.me/oauth2/v2.1/certs');
  cachedJwks = response.data;
  jwksCacheTime = Date.now();
  return cachedJwks;
}

async function verifyIdTokenLocally(idToken, expectedNonce = null) {
  try {
    // Decode header เพื่อดู kid
    const decoded = jwt.decode(idToken, { complete: true });
    if (!decoded) throw new Error('Invalid JWT format');

    // ดึง JWKS
    const jwks = await getLineJwks();
    const key = jwks.keys.find(k => k.kid === decoded.header.kid);

    if (!key) throw new Error('Public key not found');

    // แปลง JWK เป็น PEM (ใช้ jwk-to-pem library)
    const jwkToPem = require('jwk-to-pem');
    const pem = jwkToPem(key);

    // Verify JWT
    const payload = jwt.verify(idToken, pem, {
      algorithms: ['RS256'],
      audience: process.env.LINE_LOGIN_CHANNEL_ID,
      issuer: 'https://access.line.me'
    });

    // ตรวจสอบ nonce
    if (expectedNonce && payload.nonce !== expectedNonce) {
      throw new Error('Nonce mismatch');
    }

    // ตรวจสอบ expiration
    if (payload.exp < Math.floor(Date.now() / 1000)) {
      throw new Error('ID Token expired');
    }

    return payload;
  } catch (error) {
    throw new Error(`JWT verification failed: ${error.message}`);
  }
}
```

### ข้อมูลใน ID Token Payload

```javascript
/*
{
  "iss": "https://access.line.me",       // Issuer
  "sub": "U4af4980629...",               // User ID (same as userId)
  "aud": "1234567890",                   // Channel ID
  "exp": 1735689600,                     // Expiration time (Unix timestamp)
  "iat": 1735603200,                     // Issued at time
  "auth_time": 1735603200,               // Authentication time
  "nonce": "randomnonce123",             // Nonce ที่เราส่งไป
  "amr": ["linesso"],                    // Authentication method
  "name": "สมชาย ใจดี",                  // Display name (ถ้าขอ profile scope)
  "picture": "https://...",             // Profile picture URL
  "email": "somchai@example.com",       // Email (ถ้าขอ email scope)
  "email_verified": true,               // Email verified
  "phone_number": "+66812345678",       // Phone (ถ้าขอ phone scope)
  "phone_number_verified": true
}
*/
```

---

## 9. Refreshing Tokens

```javascript
async function refreshAccessToken(refreshToken) {
  const params = new URLSearchParams({
    grant_type: 'refresh_token',
    refresh_token: refreshToken,
    client_id: process.env.LINE_LOGIN_CHANNEL_ID,
    client_secret: process.env.LINE_LOGIN_CHANNEL_SECRET
  });

  try {
    const response = await axios.post(
      'https://api.line.me/oauth2/v2.1/token',
      params.toString(),
      {
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
      }
    );

    return response.data;
    // Response: { access_token, expires_in, id_token, refresh_token, scope, token_type }
  } catch (error) {
    if (error.response?.status === 400) {
      // Refresh token expired หรือ invalid
      throw new Error('Refresh token invalid or expired');
    }
    throw error;
  }
}

// ตัวอย่างการ Refresh Token อัตโนมัติ
async function getValidAccessToken(user) {
  const now = new Date();
  const tokenExpiry = new Date(user.tokenExpiresAt);

  // Refresh ถ้า token จะหมดอายุใน 5 นาที
  if (tokenExpiry - now < 5 * 60 * 1000) {
    console.log('Token expiring soon, refreshing...');

    const newTokens = await refreshAccessToken(user.refreshToken);

    // อัพเดต Database
    await db.users.updateOne(
      { _id: user._id },
      {
        $set: {
          accessToken: newTokens.access_token,
          refreshToken: newTokens.refresh_token,
          tokenExpiresAt: new Date(Date.now() + newTokens.expires_in * 1000)
        }
      }
    );

    return newTokens.access_token;
  }

  return user.accessToken;
}
```

### Verify Access Token

```javascript
async function verifyAccessToken(accessToken) {
  try {
    const response = await axios.get(
      `https://api.line.me/oauth2/v2.1/verify?access_token=${accessToken}`
    );
    return response.data;
    /*
    {
      "scope": "profile openid email",
      "client_id": "1234567890",
      "expires_in": 2591000
    }
    */
  } catch (error) {
    if (error.response?.status === 400) {
      return null; // Token invalid or expired
    }
    throw error;
  }
}
```

---

## 10. Revoking Tokens

```javascript
async function revokeToken(token, tokenType = 'access_token') {
  const params = new URLSearchParams({
    access_token: token,
    client_id: process.env.LINE_LOGIN_CHANNEL_ID,
    client_secret: process.env.LINE_LOGIN_CHANNEL_SECRET
  });

  try {
    await axios.post(
      'https://api.line.me/oauth2/v2.1/revoke',
      params.toString(),
      {
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
      }
    );
    console.log('Token revoked successfully');
    return true;
  } catch (error) {
    console.error('Revoke token failed:', error.response?.data);
    throw error;
  }
}

// Logout: Revoke token แล้ว clear session
router.post('/auth/logout', async (req, res) => {
  try {
    const user = await db.users.findById(req.session.userId);

    if (user?.accessToken) {
      await revokeToken(user.accessToken);
      // อัพเดต database
      await db.users.updateOne(
        { _id: user._id },
        { $unset: { accessToken: '', refreshToken: '' } }
      );
    }

    req.session.destroy();
    res.json({ success: true, message: 'Logged out successfully' });
  } catch (error) {
    console.error('Logout error:', error);
    res.status(500).json({ error: 'Logout failed' });
  }
});
```

---

## 11. Social Login Button

### HTML + CSS สำหรับ LINE Login Button

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>เข้าสู่ระบบ</title>
  <style>
    /* LINE Login Button */
    .line-login-btn {
      display: inline-flex;
      align-items: center;
      justify-content: center;
      gap: 12px;
      background-color: #00B900;
      color: white;
      border: none;
      border-radius: 4px;
      padding: 12px 24px;
      font-size: 16px;
      font-weight: bold;
      cursor: pointer;
      text-decoration: none;
      transition: background-color 0.2s;
      min-width: 200px;
    }

    .line-login-btn:hover {
      background-color: #00a000;
    }

    .line-login-btn:active {
      background-color: #009000;
    }

    .line-login-btn svg {
      width: 24px;
      height: 24px;
      fill: white;
    }

    /* Loading state */
    .line-login-btn.loading {
      opacity: 0.7;
      cursor: not-allowed;
    }

    /* Login Container */
    .login-container {
      max-width: 400px;
      margin: 100px auto;
      padding: 40px;
      background: white;
      border-radius: 12px;
      box-shadow: 0 4px 20px rgba(0,0,0,0.1);
      text-align: center;
    }

    .login-divider {
      display: flex;
      align-items: center;
      margin: 24px 0;
      color: #999;
    }

    .login-divider::before,
    .login-divider::after {
      content: '';
      flex: 1;
      border-bottom: 1px solid #ddd;
    }

    .login-divider span {
      padding: 0 16px;
      font-size: 14px;
    }
  </style>
</head>
<body>
  <div class="login-container">
    <h2>เข้าสู่ระบบ</h2>
    <p>เข้าสู่ระบบด้วยบัญชี LINE ของคุณ</p>

    <a href="/auth/line" class="line-login-btn" id="lineLoginBtn">
      <!-- LINE Logo SVG -->
      <svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 36 36">
        <path d="M18 4C10.27 4 4 9.46 4 16.18c0 6.08 5.39 11.17 12.69 12.13.49.11 1.16.33 1.33.75.15.38.1.98.05 1.37l-.22 1.3c-.07.38-.31 1.5 1.31.82 1.62-.68 8.73-5.14 11.91-8.8A10.52 10.52 0 0 0 32 16.18C32 9.46 25.73 4 18 4z"/>
        <path d="M26.26 20.15h-4.82a.35.35 0 0 1-.35-.35v-7.5a.35.35 0 0 1 .35-.35h4.82a.35.35 0 0 1 .35.35v1.21a.35.35 0 0 1-.35.35H23v1.35h3.26a.35.35 0 0 1 .35.35v1.21a.35.35 0 0 1-.35.35H23v1.35h3.26a.35.35 0 0 1 .35.35v1.21a.35.35 0 0 1-.35.38zM13.61 20.15a.35.35 0 0 1-.35-.35v-7.5a.35.35 0 0 1 .35-.35h1.21a.35.35 0 0 1 .35.35v6.15h3.26a.35.35 0 0 1 .35.35v1.21a.35.35 0 0 1-.35.35h-4.82zM11.15 20.15H9.94a.35.35 0 0 1-.35-.35v-7.5a.35.35 0 0 1 .35-.35h1.21a.35.35 0 0 1 .35.35v7.5a.35.35 0 0 1-.35.35zM7.44 20.15H2.62a.35.35 0 0 1-.35-.35v-7.5a.35.35 0 0 1 .35-.35h1.21a.35.35 0 0 1 .35.35v6.15h3.26a.35.35 0 0 1 .35.35v1.21a.35.35 0 0 1-.35.35z" fill="#00B900"/>
      </svg>
      เข้าสู่ระบบด้วย LINE
    </a>

    <div class="login-divider">
      <span>หรือ</span>
    </div>

    <form method="POST" action="/auth/email">
      <input type="email" name="email" placeholder="อีเมล" required>
      <input type="password" name="password" placeholder="รหัสผ่าน" required>
      <button type="submit">เข้าสู่ระบบ</button>
    </form>
  </div>

  <script>
    document.getElementById('lineLoginBtn').addEventListener('click', function() {
      this.classList.add('loading');
      this.textContent = 'กำลังเปิด LINE...';
    });
  </script>
</body>
</html>
```

---

## 12. Node.js/Express LINE Login

### การตั้งค่า Express Application เต็มรูปแบบ

```javascript
// app.js
const express = require('express');
const session = require('express-session');
const MongoStore = require('connect-mongo');
const helmet = require('helmet');
const axios = require('axios');
const crypto = require('crypto');
require('dotenv').config();

const app = express();

// Security Middleware
app.use(helmet());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Session Configuration
app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  store: MongoStore.create({
    mongoUrl: process.env.MONGODB_URI,
    ttl: 24 * 60 * 60 // 24 ชั่วโมง
  }),
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 24 * 60 * 60 * 1000
  }
}));

// LINE Login Configuration
const lineLoginConfig = {
  channelId: process.env.LINE_LOGIN_CHANNEL_ID,
  channelSecret: process.env.LINE_LOGIN_CHANNEL_SECRET,
  callbackUrl: process.env.LINE_LOGIN_CALLBACK_URL,
  defaultScopes: ['profile', 'openid', 'email']
};

// ─── Routes ────────────────────────────────────────────────────

// เริ่ม LINE Login
app.get('/auth/line', (req, res) => {
  const state = crypto.randomBytes(16).toString('hex');
  const nonce = crypto.randomBytes(16).toString('hex');

  req.session.lineAuth = {
    state,
    nonce,
    returnTo: req.query.returnTo || '/',
    createdAt: Date.now()
  };

  const scope = lineLoginConfig.defaultScopes.join('%20');
  const params = new URLSearchParams({
    response_type: 'code',
    client_id: lineLoginConfig.channelId,
    redirect_uri: lineLoginConfig.callbackUrl,
    state,
    nonce,
    scope
  });

  res.redirect(`https://access.line.me/oauth2/v2.1/authorize?${params}`);
});

// Callback
app.get('/auth/line/callback', async (req, res) => {
  const { code, state, error } = req.query;
  const lineAuth = req.session.lineAuth;

  // ตรวจสอบ error
  if (error) {
    console.error('LINE auth error:', error);
    return res.redirect(`/?error=${encodeURIComponent(error)}`);
  }

  // ตรวจสอบ state และ timeout (5 นาที)
  if (!lineAuth || lineAuth.state !== state) {
    return res.status(403).send('Invalid state');
  }
  if (Date.now() - lineAuth.createdAt > 5 * 60 * 1000) {
    return res.status(403).send('Auth session expired');
  }

  try {
    // แลก code
    const tokenRes = await exchangeCode(code);

    // Verify ID Token
    const idTokenPayload = await verifyIdToken(
      tokenRes.id_token,
      lineAuth.nonce
    );

    // ดู Profile
    const profile = await getProfile(tokenRes.access_token);

    // Find or create user
    const user = await upsertUser({
      lineUserId: profile.userId,
      displayName: profile.displayName,
      pictureUrl: profile.pictureUrl,
      email: idTokenPayload.email || null,
      phone: idTokenPayload.phone_number || null,
      accessToken: tokenRes.access_token,
      refreshToken: tokenRes.refresh_token,
      tokenExpiresAt: new Date(Date.now() + tokenRes.expires_in * 1000)
    });

    // Session
    req.session.userId = user._id.toString();
    req.session.lineUserId = profile.userId;
    delete req.session.lineAuth;

    res.redirect(lineAuth.returnTo);
  } catch (err) {
    console.error('Callback error:', err);
    res.redirect('/?error=auth_failed');
  }
});

// Protected route middleware
function requireAuth(req, res, next) {
  if (req.session.userId) {
    return next();
  }
  res.redirect(`/auth/line?returnTo=${encodeURIComponent(req.path)}`);
}

// Dashboard (Protected)
app.get('/dashboard', requireAuth, async (req, res) => {
  const user = await db.users.findById(req.session.userId);
  res.render('dashboard', { user });
});

// Logout
app.post('/auth/logout', async (req, res) => {
  const user = await db.users.findById(req.session.userId);
  if (user?.accessToken) {
    await revokeToken(user.accessToken).catch(() => {});
  }
  req.session.destroy();
  res.redirect('/');
});

// ─── Helper Functions ────────────────────────────────────────────

async function exchangeCode(code) {
  const response = await axios.post(
    'https://api.line.me/oauth2/v2.1/token',
    new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri: lineLoginConfig.callbackUrl,
      client_id: lineLoginConfig.channelId,
      client_secret: lineLoginConfig.channelSecret
    }).toString(),
    { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
  );
  return response.data;
}

async function verifyIdToken(idToken, nonce) {
  const response = await axios.post(
    'https://api.line.me/oauth2/v2.1/verify',
    new URLSearchParams({
      id_token: idToken,
      client_id: lineLoginConfig.channelId,
      nonce
    }).toString(),
    { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
  );
  return response.data;
}

async function getProfile(accessToken) {
  const response = await axios.get('https://api.line.me/v2/profile', {
    headers: { 'Authorization': `Bearer ${accessToken}` }
  });
  return response.data;
}

async function revokeToken(accessToken) {
  await axios.post(
    'https://api.line.me/oauth2/v2.1/revoke',
    new URLSearchParams({
      access_token: accessToken,
      client_id: lineLoginConfig.channelId,
      client_secret: lineLoginConfig.channelSecret
    }).toString(),
    { headers: { 'Content-Type': 'application/x-www-form-urlencoded' } }
  );
}

async function upsertUser(data) {
  return await db.users.findOneAndUpdate(
    { lineUserId: data.lineUserId },
    { $set: data },
    { upsert: true, new: true }
  );
}

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## 13. React + Node.js Backend

### Frontend (React)

```jsx
// src/components/LoginPage.jsx
import React, { useState, useEffect } from 'react';
import { useNavigate, useSearchParams } from 'react-router-dom';

const LoginPage = () => {
  const navigate = useNavigate();
  const [searchParams] = useSearchParams();
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const errorParam = searchParams.get('error');
    if (errorParam) {
      setError('การเข้าสู่ระบบล้มเหลว กรุณาลองใหม่อีกครั้ง');
    }
  }, [searchParams]);

  const handleLineLogin = async () => {
    setIsLoading(true);
    try {
      // ขอ auth URL จาก backend
      const response = await fetch('/api/auth/line/url', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ returnTo: '/dashboard' })
      });

      const { url } = await response.json();
      window.location.href = url;
    } catch (err) {
      setError('เกิดข้อผิดพลาด กรุณาลองใหม่');
      setIsLoading(false);
    }
  };

  return (
    <div className="login-page">
      <div className="login-container">
        <img src="/logo.png" alt="Logo" />
        <h1>เข้าสู่ระบบ</h1>

        {error && (
          <div className="error-message">
            ⚠️ {error}
          </div>
        )}

        <button
          className={`line-login-btn ${isLoading ? 'loading' : ''}`}
          onClick={handleLineLogin}
          disabled={isLoading}
        >
          {isLoading ? 'กำลังเปิด LINE...' : 'เข้าสู่ระบบด้วย LINE'}
        </button>
      </div>
    </div>
  );
};

export default LoginPage;
```

```jsx
// src/components/LineCallback.jsx
import React, { useEffect, useState } from 'react';
import { useNavigate, useSearchParams } from 'react-router-dom';

const LineCallback = () => {
  const navigate = useNavigate();
  const [searchParams] = useSearchParams();
  const [status, setStatus] = useState('processing');

  useEffect(() => {
    const code = searchParams.get('code');
    const state = searchParams.get('state');
    const error = searchParams.get('error');

    if (error) {
      navigate('/login?error=cancelled');
      return;
    }

    if (code && state) {
      processCallback(code, state);
    }
  }, []);

  const processCallback = async (code, state) => {
    try {
      const response = await fetch('/api/auth/line/callback', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        credentials: 'include',
        body: JSON.stringify({ code, state })
      });

      if (response.ok) {
        const { user, returnTo } = await response.json();
        // บันทึก user state
        setStatus('success');
        navigate(returnTo || '/dashboard');
      } else {
        throw new Error('Authentication failed');
      }
    } catch (err) {
      console.error('Callback error:', err);
      navigate('/login?error=failed');
    }
  };

  return (
    <div className="callback-page">
      {status === 'processing' && (
        <div className="loading">
          <div className="spinner"></div>
          <p>กำลังเข้าสู่ระบบ...</p>
        </div>
      )}
      {status === 'success' && (
        <div className="success">
          <p>เข้าสู่ระบบสำเร็จ กำลังนำทาง...</p>
        </div>
      )}
    </div>
  );
};

export default LineCallback;
```

```jsx
// src/hooks/useAuth.js
import { useState, useEffect, createContext, useContext } from 'react';

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    checkAuth();
  }, []);

  const checkAuth = async () => {
    try {
      const response = await fetch('/api/auth/me', { credentials: 'include' });
      if (response.ok) {
        const userData = await response.json();
        setUser(userData);
      }
    } catch (err) {
      console.error('Auth check failed:', err);
    } finally {
      setLoading(false);
    }
  };

  const logout = async () => {
    try {
      await fetch('/api/auth/logout', {
        method: 'POST',
        credentials: 'include'
      });
      setUser(null);
    } catch (err) {
      console.error('Logout failed:', err);
    }
  };

  return (
    <AuthContext.Provider value={{ user, loading, logout, checkAuth }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  return useContext(AuthContext);
}
```

### Backend API (Express)

```javascript
// routes/auth.js
const express = require('express');
const router = express.Router();
const crypto = require('crypto');
const axios = require('axios');

// ─── LINE Login Routes ────────────────────────────────────────────

// สร้าง URL สำหรับ Frontend
router.post('/line/url', (req, res) => {
  const { returnTo = '/' } = req.body;

  const state = crypto.randomBytes(16).toString('hex');
  const nonce = crypto.randomBytes(16).toString('hex');

  req.session.lineAuth = { state, nonce, returnTo, createdAt: Date.now() };

  const params = new URLSearchParams({
    response_type: 'code',
    client_id: process.env.LINE_LOGIN_CHANNEL_ID,
    redirect_uri: process.env.LINE_LOGIN_CALLBACK_URL,
    state,
    nonce,
    scope: 'profile openid email'
  });

  res.json({ url: `https://access.line.me/oauth2/v2.1/authorize?${params}` });
});

// Handle Callback จาก Frontend
router.post('/line/callback', async (req, res) => {
  const { code, state } = req.body;
  const lineAuth = req.session.lineAuth;

  if (!lineAuth || lineAuth.state !== state) {
    return res.status(403).json({ error: 'Invalid state' });
  }

  if (Date.now() - lineAuth.createdAt > 5 * 60 * 1000) {
    return res.status(403).json({ error: 'Session expired' });
  }

  try {
    // แลก code
    const tokenData = await exchangeCode(code);

    // Verify ID Token
    const idPayload = await verifyLineIdToken(
      tokenData.id_token,
      lineAuth.nonce
    );

    // ดู Profile
    const profile = await getLineProfile(tokenData.access_token);

    // Create/Update user
    const user = await createOrUpdateUser({
      lineUserId: profile.userId,
      displayName: profile.displayName,
      pictureUrl: profile.pictureUrl,
      email: idPayload.email,
      accessToken: tokenData.access_token,
      refreshToken: tokenData.refresh_token,
      tokenExpiresAt: new Date(Date.now() + tokenData.expires_in * 1000)
    });

    // Session
    req.session.userId = user._id.toString();
    const returnTo = lineAuth.returnTo;
    delete req.session.lineAuth;

    res.json({
      success: true,
      user: {
        id: user._id,
        displayName: user.displayName,
        pictureUrl: user.pictureUrl,
        email: user.email
      },
      returnTo
    });
  } catch (err) {
    console.error('Callback error:', err);
    res.status(500).json({ error: 'Authentication failed' });
  }
});

// Get current user
router.get('/me', requireAuth, async (req, res) => {
  const user = await User.findById(req.session.userId).select('-accessToken -refreshToken');
  res.json(user);
});

// Logout
router.post('/logout', async (req, res) => {
  const user = await User.findById(req.session.userId);
  if (user?.accessToken) {
    await revokeLineToken(user.accessToken).catch(() => {});
  }
  req.session.destroy();
  res.json({ success: true });
});

module.exports = router;
```

---

## 14. Session Management

### การจัดการ Session อย่างถูกต้อง

```javascript
// middleware/auth.js
const jwt = require('jsonwebtoken');
const User = require('../models/User');

// Middleware ตรวจสอบ Auth
async function requireAuth(req, res, next) {
  // ตรวจสอบ Session
  if (req.session?.userId) {
    try {
      const user = await User.findById(req.session.userId);
      if (user) {
        req.user = user;
        return next();
      }
    } catch (err) {
      console.error('Auth middleware error:', err);
    }
  }

  // ตรวจสอบ JWT Token (สำหรับ API)
  const authHeader = req.headers.authorization;
  if (authHeader?.startsWith('Bearer ')) {
    try {
      const token = authHeader.substring(7);
      const payload = jwt.verify(token, process.env.JWT_SECRET);
      const user = await User.findById(payload.userId);
      if (user) {
        req.user = user;
        return next();
      }
    } catch (err) {
      // Token invalid
    }
  }

  res.status(401).json({ error: 'Authentication required' });
}

// สร้าง JWT Token หลัง LINE Login
function generateJWT(userId) {
  return jwt.sign(
    { userId },
    process.env.JWT_SECRET,
    { expiresIn: '7d' }
  );
}

module.exports = { requireAuth, generateJWT };
```

### Session Store Configuration

```javascript
// config/session.js
const session = require('express-session');
const RedisStore = require('connect-redis').default;
const { createClient } = require('redis');

const redisClient = createClient({
  url: process.env.REDIS_URL
});

redisClient.connect().catch(console.error);

const sessionConfig = {
  store: new RedisStore({ client: redisClient }),
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: process.env.NODE_ENV === 'production',
    httpOnly: true,
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7 วัน
    sameSite: 'lax'
  },
  name: 'sessionId' // ไม่ใช้ชื่อ default 'connect.sid'
};

module.exports = sessionConfig;
```

---

## 15. Linking LINE Login กับบัญชีที่มีอยู่

### Scenario: ผู้ใช้มีบัญชีอยู่แล้ว ต้องการเชื่อมกับ LINE

```javascript
// routes/account.js

// เริ่มกระบวนการ Link
router.get('/settings/link-line', requireAuth, (req, res) => {
  const state = crypto.randomBytes(16).toString('hex');
  const nonce = crypto.randomBytes(16).toString('hex');

  req.session.linkState = {
    state,
    nonce,
    userId: req.user._id.toString(), // บัญชีที่กำลัง Link
    createdAt: Date.now()
  };

  const params = new URLSearchParams({
    response_type: 'code',
    client_id: process.env.LINE_LOGIN_CHANNEL_ID,
    redirect_uri: `${process.env.BASE_URL}/settings/link-line/callback`,
    state,
    nonce,
    scope: 'profile openid'
  });

  res.redirect(`https://access.line.me/oauth2/v2.1/authorize?${params}`);
});

// Callback สำหรับ Linking
router.get('/settings/link-line/callback', async (req, res) => {
  const { code, state } = req.query;
  const linkState = req.session.linkState;

  if (!linkState || linkState.state !== state) {
    return res.redirect('/settings?error=invalid_state');
  }

  try {
    const tokenData = await exchangeCode(code);
    const profile = await getLineProfile(tokenData.access_token);

    // ตรวจสอบว่า LINE ID นี้ไม่ได้ใช้กับบัญชีอื่น
    const existingUser = await User.findOne({
      lineUserId: profile.userId,
      _id: { $ne: linkState.userId }
    });

    if (existingUser) {
      return res.redirect('/settings?error=line_account_already_linked');
    }

    // เชื่อม LINE กับบัญชี
    await User.findByIdAndUpdate(linkState.userId, {
      $set: {
        lineUserId: profile.userId,
        lineDisplayName: profile.displayName,
        linePictureUrl: profile.pictureUrl,
        lineAccessToken: tokenData.access_token,
        lineRefreshToken: tokenData.refresh_token,
        lineTokenExpiresAt: new Date(Date.now() + tokenData.expires_in * 1000),
        lineLinkedAt: new Date()
      }
    });

    delete req.session.linkState;
    res.redirect('/settings?success=line_linked');
  } catch (err) {
    console.error('Link error:', err);
    res.redirect('/settings?error=link_failed');
  }
});

// Unlink LINE
router.post('/settings/unlink-line', requireAuth, async (req, res) => {
  const user = req.user;

  if (!user.lineUserId) {
    return res.json({ error: 'LINE not linked' });
  }

  // Revoke token
  if (user.lineAccessToken) {
    await revokeToken(user.lineAccessToken).catch(() => {});
  }

  await User.findByIdAndUpdate(user._id, {
    $unset: {
      lineUserId: '',
      lineDisplayName: '',
      linePictureUrl: '',
      lineAccessToken: '',
      lineRefreshToken: '',
      lineTokenExpiresAt: '',
      lineLinkedAt: ''
    }
  });

  res.json({ success: true, message: 'LINE unlinked successfully' });
});
```

### User Model ที่รองรับทั้ง Email และ LINE Login

```javascript
// models/User.js
const mongoose = require('mongoose');

const userSchema = new mongoose.Schema({
  // Email/Password Authentication
  email: { type: String, sparse: true, lowercase: true },
  password: { type: String },
  emailVerified: { type: Boolean, default: false },

  // LINE Authentication
  lineUserId: { type: String, sparse: true, unique: true },
  lineDisplayName: String,
  linePictureUrl: String,
  lineAccessToken: String,
  lineRefreshToken: String,
  lineTokenExpiresAt: Date,
  lineLinkedAt: Date,

  // Common Fields
  displayName: String,
  pictureUrl: String,
  phone: String,
  memberStatus: {
    type: String,
    enum: ['guest', 'member', 'vip', 'admin'],
    default: 'member'
  },
  createdAt: { type: Date, default: Date.now },
  lastLoginAt: Date
}, {
  timestamps: true
});

// Index
userSchema.index({ email: 1 }, { sparse: true });
userSchema.index({ lineUserId: 1 }, { sparse: true });

// Method: ตรวจสอบว่ามี LINE เชื่อมหรือยัง
userSchema.methods.hasLineLinked = function() {
  return !!this.lineUserId;
};

// Method: ตรวจสอบว่า Token หมดอายุหรือยัง
userSchema.methods.isLineTokenExpired = function() {
  if (!this.lineTokenExpiresAt) return true;
  return this.lineTokenExpiresAt < new Date();
};

const User = mongoose.model('User', userSchema);
module.exports = User;
```

---

## 16. Security Best Practices

### 1. CSRF Protection

```javascript
// ใช้ State Parameter
function generateSecureState() {
  return crypto.randomBytes(32).toString('hex');
}

// ตรวจสอบ State ทุกครั้ง
function validateState(sessionState, callbackState) {
  if (!sessionState || !callbackState) return false;
  // ใช้ timing-safe comparison เพื่อป้องกัน timing attacks
  return crypto.timingSafeEqual(
    Buffer.from(sessionState),
    Buffer.from(callbackState)
  );
}
```

### 2. HTTPS เท่านั้น

```javascript
// Force HTTPS in production
app.use((req, res, next) => {
  if (process.env.NODE_ENV === 'production' && !req.secure) {
    return res.redirect(301, `https://${req.headers.host}${req.url}`);
  }
  next();
});
```

### 3. Token Storage ที่ปลอดภัย

```javascript
const bcrypt = require('bcrypt');
const crypto = require('crypto');

// Encrypt token ก่อนบันทึกลง Database
function encryptToken(token) {
  const key = Buffer.from(process.env.ENCRYPTION_KEY, 'hex'); // 32 bytes
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-cbc', key, iv);
  const encrypted = Buffer.concat([cipher.update(token), cipher.final()]);
  return `${iv.toString('hex')}:${encrypted.toString('hex')}`;
}

function decryptToken(encryptedToken) {
  const [ivHex, encryptedHex] = encryptedToken.split(':');
  const key = Buffer.from(process.env.ENCRYPTION_KEY, 'hex');
  const iv = Buffer.from(ivHex, 'hex');
  const encrypted = Buffer.from(encryptedHex, 'hex');
  const decipher = crypto.createDecipheriv('aes-256-cbc', key, iv);
  return Buffer.concat([decipher.update(encrypted), decipher.final()]).toString();
}
```

### 4. Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

// จำกัด Login Attempts
const loginRateLimit = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 นาที
  max: 10, // 10 ครั้งต่อ 15 นาที
  message: { error: 'Too many login attempts, please try again later' },
  standardHeaders: true,
  legacyHeaders: false
});

app.use('/auth/line', loginRateLimit);
```

### 5. Validate Redirect URLs

```javascript
const ALLOWED_RETURN_PATHS = [
  '/',
  '/dashboard',
  '/profile',
  '/orders',
  '/settings'
];

function isValidReturnPath(path) {
  // ไม่ยอมรับ external URL หรือ paths ที่ไม่ได้รับอนุญาต
  if (!path.startsWith('/')) return false;
  if (path.includes('://')) return false;
  // เช็ค prefix matching
  return ALLOWED_RETURN_PATHS.some(allowed =>
    path === allowed || path.startsWith(allowed + '/')
  );
}
```

---

## สรุป LINE Login

```
LINE Login Flow:
┌─────────┐    authorize    ┌───────────┐    code     ┌──────────┐
│  User   │──────────────>  │  LINE     │──────────>  │  Your    │
│         │                 │  Server   │             │  Server  │
│         │<──────────────  │           │<──────────  │          │
└─────────┘   profile page  └───────────┘  token req  └──────────┘

ข้อมูลที่ได้:
- User ID (unique ต่อ Channel)
- Display Name + Profile Picture
- Email (ถ้าขอ scope)
- Phone (ถ้าขอ scope + ยืนยันแล้ว)
```

| สิ่งที่ต้องทำ | ความสำคัญ |
|------------|---------|
| ตรวจสอบ State | สูงมาก |
| ตรวจสอบ ID Token | สูงมาก |
| เก็บ Token อย่างปลอดภัย | สูงมาก |
| ใช้ HTTPS | สูงมาก |
| Rate Limiting | สูง |
| Refresh Token Management | ปานกลาง |

---

*ส่วนถัดไป: [Part 53: LINE Pay](part-053-line-pay.md)*
