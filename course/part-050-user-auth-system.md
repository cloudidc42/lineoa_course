# Part 50: User Authentication System สำหรับ LINE Bot

## บทนำ

การสร้างระบบ Authentication สำหรับ LINE bot เป็นเรื่องที่ต้องออกแบบอย่างรอบคอบ เพราะมีหลาย components ที่ต้องทำงานร่วมกัน ได้แก่:

- **LINE User ID** เป็น identifier หลัก
- **LINE Login OAuth 2.0** สำหรับ authentication ผ่าน LIFF
- **JWT Tokens** สำหรับ stateless API authentication
- **Account Linking** เชื่อม LINE account กับ account ระบบ
- **Permission Levels** จัดการสิทธิ์ของผู้ใช้ต่างๆ

---

## 1. Overview ของ Authentication Flow

### 1.1 Authentication Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     LINE OA Authentication                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                   │
│  User         LINE App      LIFF App      Backend API           │
│   │              │              │              │                 │
│   │──"สวัสดี"──►│              │              │                 │
│   │              │──webhook──► Bot            │                 │
│   │              │         ───────────────────►│ find/create user│
│   │              │         ◄───────────────────│ return user data│
│   │              │              │              │                 │
│   │              │              │              │                 │
│   │ Open LIFF ──►│              │              │                 │
│   │              │──launch──►  LIFF            │                 │
│   │              │         ──────── liff.init()│                 │
│   │              │         ──────── liff.login()                 │
│   │              │              │              │                 │
│   │   LINE Login ◄─────────────┤              │                 │
│   │──login──────►│              │              │                 │
│   │              │──auth code──►│              │                 │
│   │              │         exchange code for:  │                 │
│   │              │         - access_token      │                 │
│   │              │         - id_token          │                 │
│   │              │         ─────────────────►  │                 │
│   │              │         verify token / create JWT             │
│   │              │         ◄─────────────────  │ JWT Token       │
│   │              │              │              │                 │
│   │              │         API requests with JWT                 │
│   │              │         ─────────────────►  │ verify JWT      │
│   │              │         ◄─────────────────  │ protected data  │
│   │◄─────────────────────────────────────────  │                 │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 Token Types

```
1. LINE Access Token
   - ได้จาก LINE Platform
   - ใช้เรียก LINE API (getProfile, sendMessage)
   - อายุ: 30 วัน
   - ไม่ควรส่งไปยัง backend ของเรา

2. LINE ID Token (JWT)
   - ข้อมูล user identity
   - ใช้ verify ว่าเป็น user จริง
   - ควรส่งไป backend เพื่อ verify

3. Our Application JWT
   - สร้างโดย backend ของเรา
   - ใช้ใน API requests
   - อายุที่กำหนดเอง
   - มี permission claims
```

---

## 2. Database Schema

### 2.1 Users Table (แบบ Complete)

```sql
-- =============================================
-- USERS - ข้อมูลผู้ใช้หลัก
-- =============================================
CREATE TABLE users (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    
    -- LINE Identity
    line_user_id    VARCHAR(50) UNIQUE,
    line_display_name VARCHAR(255),
    line_picture_url  TEXT,
    
    -- Application Identity (optional - สำหรับ account linking)
    username        VARCHAR(100) UNIQUE,
    email           VARCHAR(255) UNIQUE,
    phone           VARCHAR(20),
    first_name      VARCHAR(100),
    last_name       VARCHAR(100),
    
    -- Auth
    password_hash   VARCHAR(255),        -- สำหรับ users ที่ register ด้วย email
    role            ENUM('admin','staff','user','guest') DEFAULT 'user',
    permissions     JSON,                -- custom permissions
    
    -- Status
    is_active       BOOLEAN DEFAULT TRUE,
    is_verified     BOOLEAN DEFAULT FALSE,
    is_blocked      BOOLEAN DEFAULT FALSE,
    blocked_reason  TEXT,
    
    -- Profile
    profile_image   TEXT,
    bio             TEXT,
    
    -- LINE-specific
    line_language   VARCHAR(10) DEFAULT 'th',
    is_line_blocked BOOLEAN DEFAULT FALSE,  -- user block LINE OA
    
    -- Timestamps
    email_verified_at   DATETIME,
    phone_verified_at   DATETIME,
    last_login_at       DATETIME,
    last_seen_at        DATETIME,
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    deleted_at          DATETIME,           -- soft delete
    
    INDEX idx_line_user_id (line_user_id),
    INDEX idx_email (email),
    INDEX idx_username (username),
    INDEX idx_role (role),
    INDEX idx_active (is_active),
    INDEX idx_created (created_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- REFRESH TOKENS - สำหรับ JWT refresh
-- =============================================
CREATE TABLE refresh_tokens (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT UNSIGNED NOT NULL,
    token_hash  VARCHAR(64) NOT NULL UNIQUE,  -- SHA-256 hash ของ token
    device_info TEXT,                          -- device ที่ใช้
    ip_address  VARCHAR(45),
    user_agent  TEXT,
    is_revoked  BOOLEAN DEFAULT FALSE,
    last_used_at DATETIME,
    expires_at  DATETIME NOT NULL,
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY fk_rt_user (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_token_hash (token_hash),
    INDEX idx_user_id (user_id),
    INDEX idx_expires (expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- ACCOUNT LINK TOKENS - สำหรับ LINE Account Linking
-- =============================================
CREATE TABLE account_link_tokens (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT UNSIGNED NOT NULL,       -- user ในระบบของเรา
    link_token  VARCHAR(100) NOT NULL UNIQUE,   -- token จาก LINE
    nonce       VARCHAR(100),                    -- nonce สำหรับ verify
    is_used     BOOLEAN DEFAULT FALSE,
    expires_at  DATETIME NOT NULL,
    used_at     DATETIME,
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY fk_alt_user (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_link_token (link_token),
    INDEX idx_expires (expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- LOGIN LOGS - บันทึกการ login
-- =============================================
CREATE TABLE login_logs (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT UNSIGNED,
    login_type  ENUM('line_login','email','phone','token_refresh') NOT NULL,
    status      ENUM('success','failed','blocked') NOT NULL,
    ip_address  VARCHAR(45),
    user_agent  TEXT,
    failure_reason TEXT,
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY fk_ll_user (user_id) REFERENCES users(id) ON DELETE SET NULL,
    INDEX idx_user_id (user_id),
    INDEX idx_created (created_at),
    INDEX idx_status (status)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- PERMISSIONS - การจัดการ permissions
-- =============================================
CREATE TABLE permissions (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(100) NOT NULL UNIQUE,  -- 'orders.create', 'users.admin'
    description TEXT,
    group_name  VARCHAR(50),                    -- จัดกลุ่ม permissions
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE roles (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name        VARCHAR(50) NOT NULL UNIQUE,   -- 'admin', 'staff', 'user'
    description TEXT,
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE TABLE role_permissions (
    role_id         INT UNSIGNED NOT NULL,
    permission_id   INT UNSIGNED NOT NULL,
    PRIMARY KEY (role_id, permission_id),
    FOREIGN KEY fk_rp_role (role_id) REFERENCES roles(id) ON DELETE CASCADE,
    FOREIGN KEY fk_rp_perm (permission_id) REFERENCES permissions(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ตัวอย่างข้อมูล permissions
INSERT INTO permissions (name, description, group_name) VALUES
    ('orders.read', 'ดูออเดอร์', 'orders'),
    ('orders.create', 'สร้างออเดอร์', 'orders'),
    ('orders.update', 'แก้ไขออเดอร์', 'orders'),
    ('orders.delete', 'ลบออเดอร์', 'orders'),
    ('users.read', 'ดูข้อมูลผู้ใช้', 'users'),
    ('users.update', 'แก้ไขข้อมูลผู้ใช้', 'users'),
    ('users.admin', 'จัดการผู้ใช้ทั้งหมด', 'users'),
    ('products.manage', 'จัดการสินค้า', 'products'),
    ('analytics.view', 'ดูรายงาน', 'analytics'),
    ('broadcast.send', 'ส่ง broadcast', 'messaging');

INSERT INTO roles (name, description) VALUES
    ('admin', 'ผู้ดูแลระบบทั้งหมด'),
    ('staff', 'พนักงาน'),
    ('user', 'ผู้ใช้ทั่วไป');
```

---

## 3. Node.js + Express Authentication System

### 3.1 JWT Utilities

```javascript
// src/auth/jwt.js
const jwt = require('jsonwebtoken');
const crypto = require('crypto');

const JWT_SECRET = process.env.JWT_SECRET;
const JWT_REFRESH_SECRET = process.env.JWT_REFRESH_SECRET;

if (!JWT_SECRET || !JWT_REFRESH_SECRET) {
  throw new Error('JWT secrets must be configured');
}

/**
 * สร้าง Access Token (อายุสั้น)
 */
function generateAccessToken(user) {
  return jwt.sign(
    {
      sub: user.id.toString(),
      lineUserId: user.line_user_id,
      role: user.role,
      permissions: user.permissions || [],
      iat: Math.floor(Date.now() / 1000),
    },
    JWT_SECRET,
    {
      expiresIn: '1h',
      issuer: 'lineoa-app',
      audience: 'lineoa-api',
    }
  );
}

/**
 * สร้าง Refresh Token (อายุยาว)
 */
function generateRefreshToken(userId) {
  const token = crypto.randomBytes(64).toString('hex');
  const expiresAt = new Date(Date.now() + 30 * 24 * 60 * 60 * 1000); // 30 วัน
  return { token, expiresAt };
}

/**
 * Hash refresh token สำหรับเก็บ DB
 */
function hashToken(token) {
  return crypto.createHash('sha256').update(token).digest('hex');
}

/**
 * Verify Access Token
 */
function verifyAccessToken(token) {
  try {
    return jwt.verify(token, JWT_SECRET, {
      issuer: 'lineoa-app',
      audience: 'lineoa-api',
    });
  } catch (error) {
    if (error.name === 'TokenExpiredError') {
      throw new Error('Token expired');
    }
    throw new Error('Invalid token');
  }
}

/**
 * Decode token โดยไม่ verify (สำหรับดู payload ที่หมดอายุ)
 */
function decodeToken(token) {
  return jwt.decode(token);
}

module.exports = {
  generateAccessToken,
  generateRefreshToken,
  hashToken,
  verifyAccessToken,
  decodeToken,
};
```

### 3.2 LINE Token Verification

```javascript
// src/auth/lineVerify.js
const axios = require('axios');
const crypto = require('crypto');

/**
 * Verify LINE ID Token
 * ตรวจสอบว่า token มาจาก LINE จริงและ user เป็นคนที่ claim
 */
async function verifyLineIdToken(idToken, clientId) {
  try {
    // เรียก LINE API เพื่อ verify token
    const response = await axios.post(
      'https://api.line.me/oauth2/v2.1/verify',
      new URLSearchParams({
        id_token: idToken,
        client_id: clientId,
      }),
      {
        headers: {
          'Content-Type': 'application/x-www-form-urlencoded',
        },
      }
    );

    return {
      valid: true,
      data: response.data,
    };
  } catch (error) {
    return {
      valid: false,
      error: error.response?.data?.error_description || error.message,
    };
  }
}

/**
 * Verify LINE Access Token
 */
async function verifyLineAccessToken(accessToken) {
  try {
    const response = await axios.get(
      'https://api.line.me/oauth2/v2.1/verify',
      {
        params: { access_token: accessToken },
      }
    );

    const data = response.data;

    // ตรวจสอบว่า token ยังไม่หมดอายุ
    if (data.expires_in <= 0) {
      return { valid: false, error: 'Token expired' };
    }

    // ตรวจสอบว่า client_id ตรงกัน
    if (data.client_id !== process.env.LINE_CHANNEL_ID) {
      return { valid: false, error: 'Invalid client' };
    }

    return { valid: true, data };
  } catch (error) {
    return {
      valid: false,
      error: error.response?.data?.error_description || error.message,
    };
  }
}

/**
 * Get LINE user profile ด้วย Access Token
 */
async function getLineProfile(accessToken) {
  const response = await axios.get('https://api.line.me/v2/profile', {
    headers: { Authorization: `Bearer ${accessToken}` },
  });
  return response.data;
}

/**
 * Verify LINE Webhook Signature
 */
function verifyWebhookSignature(body, signature) {
  const channelSecret = process.env.LINE_CHANNEL_SECRET;
  const hash = crypto
    .createHmac('sha256', channelSecret)
    .update(typeof body === 'string' ? body : JSON.stringify(body))
    .digest('base64');
  return hash === signature;
}

module.exports = {
  verifyLineIdToken,
  verifyLineAccessToken,
  getLineProfile,
  verifyWebhookSignature,
};
```

### 3.3 AuthService

```javascript
// src/auth/AuthService.js
const bcrypt = require('bcryptjs');
const { pool } = require('../database/mysql');
const {
  generateAccessToken,
  generateRefreshToken,
  hashToken,
  verifyAccessToken,
} = require('./jwt');
const {
  verifyLineIdToken,
  verifyLineAccessToken,
  getLineProfile,
} = require('./lineVerify');

class AuthService {

  /**
   * Login/Register ด้วย LINE (จาก LIFF)
   */
  async loginWithLine(accessToken, idToken) {
    // 1. Verify LINE Access Token
    const tokenVerification = await verifyLineAccessToken(accessToken);
    if (!tokenVerification.valid) {
      throw new Error('Invalid LINE access token');
    }

    // 2. Verify LINE ID Token
    const idTokenVerification = await verifyLineIdToken(
      idToken,
      process.env.LINE_CHANNEL_ID
    );
    if (!idTokenVerification.valid) {
      throw new Error('Invalid LINE ID token');
    }

    // 3. ดึงข้อมูล profile
    const lineProfile = await getLineProfile(accessToken);

    // 4. Find or create user
    const user = await this.findOrCreateLineUser(lineProfile, idTokenVerification.data);

    // 5. สร้าง tokens
    const accessJwt = generateAccessToken(user);
    const { token: refreshToken, expiresAt } = generateRefreshToken(user.id);

    // 6. บันทึก refresh token
    await this.saveRefreshToken(user.id, refreshToken, expiresAt);

    // 7. Log login
    await this.logLogin(user.id, 'line_login', 'success');

    return {
      user: this.sanitizeUser(user),
      accessToken: accessJwt,
      refreshToken,
      expiresAt,
    };
  }

  /**
   * Login ด้วย Email/Password
   */
  async loginWithEmail(email, password) {
    const [users] = await pool.execute(
      'SELECT * FROM users WHERE email = ? AND deleted_at IS NULL LIMIT 1',
      [email]
    );

    const user = users[0];

    if (!user || !user.password_hash) {
      await this.logLogin(null, 'email', 'failed', 'User not found');
      throw new Error('Invalid credentials');
    }

    if (!user.is_active) {
      throw new Error('Account is disabled');
    }

    if (user.is_blocked) {
      throw new Error(`Account is blocked: ${user.blocked_reason || ''}`);
    }

    const isValidPassword = await bcrypt.compare(password, user.password_hash);
    if (!isValidPassword) {
      await this.logLogin(user.id, 'email', 'failed', 'Invalid password');
      throw new Error('Invalid credentials');
    }

    // สร้าง tokens
    const accessToken = generateAccessToken(user);
    const { token: refreshToken, expiresAt } = generateRefreshToken(user.id);
    await this.saveRefreshToken(user.id, refreshToken, expiresAt);

    // อัปเดต last login
    await pool.execute(
      'UPDATE users SET last_login_at = NOW() WHERE id = ?',
      [user.id]
    );

    await this.logLogin(user.id, 'email', 'success');

    return {
      user: this.sanitizeUser(user),
      accessToken,
      refreshToken,
      expiresAt,
    };
  }

  /**
   * Refresh Access Token
   */
  async refreshAccessToken(refreshTokenValue) {
    const tokenHash = hashToken(refreshTokenValue);

    const [tokens] = await pool.execute(
      `SELECT rt.*, u.* 
       FROM refresh_tokens rt
       JOIN users u ON rt.user_id = u.id
       WHERE rt.token_hash = ? 
         AND rt.is_revoked = FALSE 
         AND rt.expires_at > NOW()
         AND u.is_active = TRUE
       LIMIT 1`,
      [tokenHash]
    );

    if (tokens.length === 0) {
      throw new Error('Invalid or expired refresh token');
    }

    const { user_id, ...userData } = tokens[0];

    // Update last used
    await pool.execute(
      'UPDATE refresh_tokens SET last_used_at = NOW() WHERE token_hash = ?',
      [tokenHash]
    );

    const accessToken = generateAccessToken(userData);
    await this.logLogin(user_id, 'token_refresh', 'success');

    return { accessToken };
  }

  /**
   * Logout
   */
  async logout(userId, refreshTokenValue) {
    if (refreshTokenValue) {
      const tokenHash = hashToken(refreshTokenValue);
      await pool.execute(
        'UPDATE refresh_tokens SET is_revoked = TRUE WHERE token_hash = ? AND user_id = ?',
        [tokenHash, userId]
      );
    }
  }

  /**
   * Logout from all devices
   */
  async logoutAll(userId) {
    await pool.execute(
      'UPDATE refresh_tokens SET is_revoked = TRUE WHERE user_id = ?',
      [userId]
    );
  }

  /**
   * Register ด้วย Email
   */
  async registerWithEmail(email, password, name) {
    // ตรวจสอบ email ซ้ำ
    const [existing] = await pool.execute(
      'SELECT id FROM users WHERE email = ? LIMIT 1',
      [email]
    );

    if (existing.length > 0) {
      throw new Error('Email already registered');
    }

    // Hash password
    const passwordHash = await bcrypt.hash(password, 12);

    // สร้าง user
    const [result] = await pool.execute(
      `INSERT INTO users (email, password_hash, first_name, role, is_verified)
       VALUES (?, ?, ?, 'user', FALSE)`,
      [email, passwordHash, name]
    );

    const userId = result.insertId;

    // TODO: ส่ง verification email

    const [users] = await pool.execute('SELECT * FROM users WHERE id = ?', [userId]);
    return this.sanitizeUser(users[0]);
  }

  /**
   * สร้างหรือดึง user จาก LINE profile
   */
  async findOrCreateLineUser(lineProfile, idTokenData) {
    const conn = await pool.getConnection();
    try {
      await conn.beginTransaction();

      const [existing] = await conn.execute(
        'SELECT * FROM users WHERE line_user_id = ? FOR UPDATE',
        [lineProfile.userId]
      );

      let user;
      if (existing.length > 0) {
        // อัปเดต profile
        await conn.execute(
          `UPDATE users 
           SET line_display_name = ?,
               line_picture_url = ?,
               last_login_at = NOW(),
               last_seen_at = NOW()
           WHERE id = ?`,
          [lineProfile.displayName, lineProfile.pictureUrl, existing[0].id]
        );
        user = { ...existing[0], line_display_name: lineProfile.displayName };
      } else {
        // สร้าง user ใหม่
        const [result] = await conn.execute(
          `INSERT INTO users 
           (line_user_id, line_display_name, line_picture_url, 
            email, role, is_active, last_login_at)
           VALUES (?, ?, ?, ?, 'user', TRUE, NOW())`,
          [
            lineProfile.userId,
            lineProfile.displayName,
            lineProfile.pictureUrl,
            idTokenData.email || null,
          ]
        );

        const [newUser] = await conn.execute(
          'SELECT * FROM users WHERE id = ?',
          [result.insertId]
        );
        user = newUser[0];
      }

      await conn.commit();
      return user;
    } catch (error) {
      await conn.rollback();
      throw error;
    } finally {
      conn.release();
    }
  }

  /**
   * บันทึก Refresh Token
   */
  async saveRefreshToken(userId, token, expiresAt, deviceInfo = null) {
    const tokenHash = hashToken(token);
    await pool.execute(
      `INSERT INTO refresh_tokens (user_id, token_hash, device_info, expires_at)
       VALUES (?, ?, ?, ?)`,
      [userId, tokenHash, deviceInfo, expiresAt]
    );
  }

  /**
   * Log การ login
   */
  async logLogin(userId, loginType, status, failureReason = null) {
    await pool.execute(
      `INSERT INTO login_logs (user_id, login_type, status, failure_reason)
       VALUES (?, ?, ?, ?)`,
      [userId, loginType, status, failureReason]
    );
  }

  /**
   * ตัด sensitive fields ออก
   */
  sanitizeUser(user) {
    const { password_hash, ...safeUser } = user;
    return safeUser;
  }
}

module.exports = new AuthService();
```

### 3.4 Authentication Middleware

```javascript
// src/middleware/auth.js
const { verifyAccessToken } = require('../auth/jwt');
const { pool } = require('../database/mysql');

/**
 * Middleware ตรวจสอบ JWT token
 */
async function authenticate(req, res, next) {
  try {
    const authHeader = req.headers.authorization;

    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({
        success: false,
        error: 'Unauthorized',
        message: 'No token provided',
      });
    }

    const token = authHeader.substring(7);
    const decoded = verifyAccessToken(token);

    // โหลด user จาก DB (เพื่อดู role/permissions ล่าสุด)
    const [users] = await pool.execute(
      `SELECT id, line_user_id, email, role, permissions, is_active, is_blocked
       FROM users WHERE id = ? LIMIT 1`,
      [decoded.sub]
    );

    if (users.length === 0) {
      return res.status(401).json({
        success: false,
        error: 'User not found',
      });
    }

    const user = users[0];

    if (!user.is_active) {
      return res.status(403).json({
        success: false,
        error: 'Account disabled',
      });
    }

    if (user.is_blocked) {
      return res.status(403).json({
        success: false,
        error: 'Account blocked',
      });
    }

    req.user = user;
    req.token = decoded;
    next();
  } catch (error) {
    if (error.message === 'Token expired') {
      return res.status(401).json({
        success: false,
        error: 'TokenExpired',
        message: 'Access token expired, please refresh',
      });
    }

    return res.status(401).json({
      success: false,
      error: 'InvalidToken',
      message: 'Invalid token',
    });
  }
}

/**
 * Middleware ตรวจสอบ Role
 */
function requireRole(...roles) {
  return (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    if (!roles.includes(req.user.role)) {
      return res.status(403).json({
        success: false,
        error: 'Forbidden',
        message: `Required role: ${roles.join(' or ')}`,
      });
    }

    next();
  };
}

/**
 * Middleware ตรวจสอบ Permission
 */
function requirePermission(permission) {
  return async (req, res, next) => {
    if (!req.user) {
      return res.status(401).json({ error: 'Unauthorized' });
    }

    // Admin มีทุก permission
    if (req.user.role === 'admin') {
      return next();
    }

    // ตรวจสอบ custom permissions
    const userPermissions = req.user.permissions || [];
    if (userPermissions.includes(permission)) {
      return next();
    }

    // ตรวจสอบ role-based permissions จาก DB
    const [perms] = await pool.execute(
      `SELECT p.name FROM permissions p
       JOIN role_permissions rp ON p.id = rp.permission_id
       JOIN roles r ON rp.role_id = r.id
       WHERE r.name = ? AND p.name = ?`,
      [req.user.role, permission]
    );

    if (perms.length > 0) {
      return next();
    }

    return res.status(403).json({
      success: false,
      error: 'Forbidden',
      message: `Required permission: ${permission}`,
    });
  };
}

/**
 * Optional Auth - ไม่บังคับ login
 */
async function optionalAuth(req, res, next) {
  const authHeader = req.headers.authorization;

  if (!authHeader || !authHeader.startsWith('Bearer ')) {
    req.user = null;
    return next();
  }

  try {
    const token = authHeader.substring(7);
    const decoded = verifyAccessToken(token);

    const [users] = await pool.execute(
      'SELECT id, line_user_id, role FROM users WHERE id = ? AND is_active = TRUE LIMIT 1',
      [decoded.sub]
    );

    req.user = users[0] || null;
  } catch {
    req.user = null;
  }

  next();
}

module.exports = { authenticate, requireRole, requirePermission, optionalAuth };
```

### 3.5 Auth Routes

```javascript
// src/routes/auth.js
const express = require('express');
const router = express.Router();
const rateLimit = require('express-rate-limit');
const AuthService = require('../auth/AuthService');
const { authenticate } = require('../middleware/auth');

// Rate limiting สำหรับ auth endpoints
const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 นาที
  max: 10,
  message: { error: 'Too many attempts, please try again later' },
  standardHeaders: true,
  legacyHeaders: false,
});

const loginLimiter = rateLimit({
  windowMs: 5 * 60 * 1000,  // 5 นาที
  max: 5,
  message: { error: 'Too many login attempts' },
});

/**
 * POST /auth/line
 * Login ด้วย LINE (จาก LIFF)
 */
router.post('/line', authLimiter, async (req, res) => {
  try {
    const { accessToken, idToken } = req.body;

    if (!accessToken || !idToken) {
      return res.status(400).json({
        success: false,
        error: 'accessToken and idToken are required',
      });
    }

    const result = await AuthService.loginWithLine(accessToken, idToken);

    res.json({ success: true, ...result });
  } catch (error) {
    console.error('LINE login error:', error);
    res.status(401).json({
      success: false,
      error: error.message,
    });
  }
});

/**
 * POST /auth/email
 * Login ด้วย Email/Password
 */
router.post('/email', loginLimiter, async (req, res) => {
  try {
    const { email, password } = req.body;

    if (!email || !password) {
      return res.status(400).json({ error: 'Email and password required' });
    }

    const result = await AuthService.loginWithEmail(email, password);
    res.json({ success: true, ...result });
  } catch (error) {
    res.status(401).json({ success: false, error: error.message });
  }
});

/**
 * POST /auth/register
 * Register ด้วย Email
 */
router.post('/register', authLimiter, async (req, res) => {
  try {
    const { email, password, name } = req.body;

    if (!email || !password || !name) {
      return res.status(400).json({ error: 'All fields required' });
    }

    if (password.length < 8) {
      return res.status(400).json({ error: 'Password must be at least 8 characters' });
    }

    const user = await AuthService.registerWithEmail(email, password, name);
    res.status(201).json({ success: true, user });
  } catch (error) {
    res.status(400).json({ success: false, error: error.message });
  }
});

/**
 * POST /auth/refresh
 * Refresh Access Token
 */
router.post('/refresh', authLimiter, async (req, res) => {
  try {
    const { refreshToken } = req.body;

    if (!refreshToken) {
      return res.status(400).json({ error: 'refreshToken required' });
    }

    const result = await AuthService.refreshAccessToken(refreshToken);
    res.json({ success: true, ...result });
  } catch (error) {
    res.status(401).json({ success: false, error: error.message });
  }
});

/**
 * POST /auth/logout
 * Logout
 */
router.post('/logout', authenticate, async (req, res) => {
  try {
    const { refreshToken } = req.body;
    await AuthService.logout(req.user.id, refreshToken);
    res.json({ success: true, message: 'Logged out successfully' });
  } catch (error) {
    res.status(500).json({ success: false, error: error.message });
  }
});

/**
 * GET /auth/me
 * ดูข้อมูล user ปัจจุบัน
 */
router.get('/me', authenticate, (req, res) => {
  res.json({ success: true, user: req.user });
});

module.exports = router;
```

---

## 4. LINE Account Linking

### 4.1 Account Linking Flow

```
Account Linking ช่วยเชื่อม LINE account กับ account ในระบบของเรา

Flow:
┌──────────────────────────────────────────────────────────────────┐
│                    Account Linking Flow                           │
│                                                                    │
│  Bot              LINE Platform         Our Backend    Our App   │
│   │                    │                    │             │       │
│   │◄──"เชื่อมบัญชี"──User                   │             │       │
│   │                    │                    │             │       │
│   │──request link token────────────────────►│             │       │
│   │◄──link token──────────────────────────  │             │       │
│   │                    │                    │             │       │
│   │──send link URL────►│                    │             │       │
│   │                    │──open URL─────────────────────► │       │
│   │                    │         user login on Our App ◄─│       │
│   │                    │         link LINE account       │       │
│   │                    │◄─── callback with nonce ────────│       │
│   │                    │                    │             │       │
│   │◄── link result ────│                    │             │       │
│   │                    │                    │             │       │
│   │──save link─────────────────────────────►│             │       │
│   │◄──success──────────────────────────────│             │       │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 Account Linking Implementation

```javascript
// src/auth/AccountLinking.js
const line = require('@line/bot-sdk');
const { pool } = require('../database/mysql');
const crypto = require('crypto');

const lineClient = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
});

class AccountLinkingService {

  /**
   * เริ่มต้น account linking - สร้าง link token
   */
  async initiateLink(lineUserId) {
    // สร้าง link token จาก LINE
    const response = await lineClient.getLinkToken(lineUserId);
    const linkToken = response.linkToken;

    // หา user ในระบบหรือสร้างใหม่
    const [users] = await pool.execute(
      'SELECT id FROM users WHERE line_user_id = ? LIMIT 1',
      [lineUserId]
    );

    let userId;
    if (users.length === 0) {
      const [result] = await pool.execute(
        `INSERT INTO users (line_user_id, role) VALUES (?, 'user')`,
        [lineUserId]
      );
      userId = result.insertId;
    } else {
      userId = users[0].id;
    }

    // บันทึก link token
    const expiresAt = new Date(Date.now() + 10 * 60 * 1000); // 10 นาที
    await pool.execute(
      `INSERT INTO account_link_tokens (user_id, link_token, expires_at)
       VALUES (?, ?, ?)`,
      [userId, linkToken, expiresAt]
    );

    // URL สำหรับ link
    const linkUrl = `${process.env.APP_URL}/account/link?linkToken=${linkToken}`;

    return { linkToken, linkUrl, userId };
  }

  /**
   * Complete account linking - เมื่อ user login บน app แล้ว
   */
  async completeLinkWithExistingAccount(linkToken, existingUserId) {
    // ดึงข้อมูล link token
    const [tokens] = await pool.execute(
      `SELECT * FROM account_link_tokens
       WHERE link_token = ?
         AND is_used = FALSE
         AND expires_at > NOW()
       LIMIT 1`,
      [linkToken]
    );

    if (tokens.length === 0) {
      throw new Error('Invalid or expired link token');
    }

    const linkTokenData = tokens[0];

    // ตรวจสอบว่า existingUser มีอยู่
    const [existingUsers] = await pool.execute(
      'SELECT * FROM users WHERE id = ? LIMIT 1',
      [existingUserId]
    );

    if (existingUsers.length === 0) {
      throw new Error('Target account not found');
    }

    const targetUser = existingUsers[0];

    // ดึง LINE user ที่ต้องการ link
    const [lineUsers] = await pool.execute(
      'SELECT * FROM users WHERE id = ? LIMIT 1',
      [linkTokenData.user_id]
    );

    if (lineUsers.length === 0) {
      throw new Error('LINE user not found');
    }

    const lineUser = lineUsers[0];

    const conn = await pool.getConnection();
    try {
      await conn.beginTransaction();

      // ถ้า user เป็น account ชั่วคราว (มีแค่ line_user_id)
      if (lineUser.id !== existingUserId) {
        // ย้าย LINE data ไปยัง existing account
        await conn.execute(
          `UPDATE users 
           SET line_user_id = ?,
               line_display_name = ?,
               line_picture_url = ?
           WHERE id = ?`,
          [
            lineUser.line_user_id,
            lineUser.line_display_name,
            lineUser.line_picture_url,
            existingUserId
          ]
        );

        // ลบ account ชั่วคราว
        await conn.execute(
          'DELETE FROM users WHERE id = ?',
          [lineUser.id]
        );
      }

      // Mark token as used
      await conn.execute(
        'UPDATE account_link_tokens SET is_used = TRUE, used_at = NOW() WHERE id = ?',
        [linkTokenData.id]
      );

      await conn.commit();

      return {
        success: true,
        userId: existingUserId,
        lineUserId: lineUser.line_user_id,
      };
    } catch (error) {
      await conn.rollback();
      throw error;
    } finally {
      conn.release();
    }
  }

  /**
   * Handle Account Link webhook event
   */
  async handleLinkWebhook(event) {
    if (event.type !== 'accountLink') return;

    const { userId } = event.source;
    const { result, nonce } = event.link;

    if (result === 'ok') {
      // Link สำเร็จ
      await this.confirmLink(userId, nonce);

      await lineClient.replyMessage(event.replyToken, [{
        type: 'text',
        text: '✅ เชื่อมบัญชีสำเร็จแล้วครับ! คุณสามารถใช้บริการทั้งหมดได้แล้ว',
      }]);
    } else {
      await lineClient.replyMessage(event.replyToken, [{
        type: 'text',
        text: '❌ การเชื่อมบัญชีล้มเหลว กรุณาลองใหม่อีกครั้ง',
      }]);
    }
  }

  /**
   * Unlink account
   */
  async unlinkAccount(userId) {
    const [users] = await pool.execute(
      'SELECT line_user_id FROM users WHERE id = ? LIMIT 1',
      [userId]
    );

    if (users.length === 0 || !users[0].line_user_id) {
      throw new Error('No linked LINE account found');
    }

    await pool.execute(
      `UPDATE users 
       SET line_user_id = NULL, line_display_name = NULL, line_picture_url = NULL 
       WHERE id = ?`,
      [userId]
    );

    return { success: true };
  }
}

module.exports = new AccountLinkingService();
```

---

## 5. Python + FastAPI Authentication System

```python
# auth/main.py
from fastapi import FastAPI, HTTPException, Depends, Security
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel, EmailStr
from typing import Optional, List
import jwt
import bcrypt
import httpx
import secrets
import hashlib
from datetime import datetime, timedelta
import databases
import sqlalchemy

# =============================================
# DATABASE SETUP
# =============================================
DATABASE_URL = "mysql+aiomysql://user:password@localhost/linebot"
database = databases.Database(DATABASE_URL)

# =============================================
# MODELS
# =============================================
class LineLoginRequest(BaseModel):
    access_token: str
    id_token: str

class EmailLoginRequest(BaseModel):
    email: EmailStr
    password: str

class RegisterRequest(BaseModel):
    email: EmailStr
    password: str
    name: str

class RefreshTokenRequest(BaseModel):
    refresh_token: str

class TokenResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str = "bearer"
    expires_in: int = 3600

class UserResponse(BaseModel):
    id: int
    line_user_id: Optional[str]
    email: Optional[str]
    first_name: Optional[str]
    role: str
    is_active: bool

# =============================================
# JWT UTILITIES
# =============================================
JWT_SECRET = "your-secret-key-change-this"
JWT_ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE = timedelta(hours=1)
REFRESH_TOKEN_EXPIRE = timedelta(days=30)

def create_access_token(user_data: dict) -> str:
    payload = {
        "sub": str(user_data["id"]),
        "line_user_id": user_data.get("line_user_id"),
        "role": user_data.get("role", "user"),
        "exp": datetime.utcnow() + ACCESS_TOKEN_EXPIRE,
        "iat": datetime.utcnow(),
        "type": "access"
    }
    return jwt.encode(payload, JWT_SECRET, algorithm=JWT_ALGORITHM)

def create_refresh_token() -> tuple[str, datetime]:
    token = secrets.token_hex(64)
    expires_at = datetime.utcnow() + REFRESH_TOKEN_EXPIRE
    return token, expires_at

def hash_token(token: str) -> str:
    return hashlib.sha256(token.encode()).hexdigest()

def verify_access_token(token: str) -> dict:
    try:
        payload = jwt.decode(token, JWT_SECRET, algorithms=[JWT_ALGORITHM])
        if payload.get("type") != "access":
            raise HTTPException(status_code=401, detail="Invalid token type")
        return payload
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

# =============================================
# LINE VERIFICATION
# =============================================
LINE_CHANNEL_ID = "your-line-channel-id"

async def verify_line_access_token(access_token: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "https://api.line.me/oauth2/v2.1/verify",
            params={"access_token": access_token}
        )
        if response.status_code != 200:
            raise HTTPException(status_code=401, detail="Invalid LINE access token")
        
        data = response.json()
        if data.get("client_id") != LINE_CHANNEL_ID:
            raise HTTPException(status_code=401, detail="Token client mismatch")
        
        return data

async def get_line_profile(access_token: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(
            "https://api.line.me/v2/profile",
            headers={"Authorization": f"Bearer {access_token}"}
        )
        response.raise_for_status()
        return response.json()

# =============================================
# APP SETUP
# =============================================
app = FastAPI(title="LINE Bot Auth API")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

security = HTTPBearer()

# =============================================
# DEPENDENCIES
# =============================================
async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Security(security)
):
    token = credentials.credentials
    payload = verify_access_token(token)
    
    user_id = int(payload["sub"])
    
    query = "SELECT * FROM users WHERE id = :id AND is_active = TRUE"
    user = await database.fetch_one(query=query, values={"id": user_id})
    
    if not user:
        raise HTTPException(status_code=401, detail="User not found")
    
    if user["is_blocked"]:
        raise HTTPException(status_code=403, detail="Account blocked")
    
    return dict(user)

def require_role(*roles: str):
    async def role_checker(user: dict = Depends(get_current_user)):
        if user["role"] not in roles:
            raise HTTPException(
                status_code=403,
                detail=f"Required role: {', '.join(roles)}"
            )
        return user
    return role_checker

# =============================================
# AUTH ENDPOINTS
# =============================================
@app.post("/auth/line", response_model=TokenResponse)
async def login_with_line(request: LineLoginRequest):
    """Login ด้วย LINE"""
    # Verify token
    await verify_line_access_token(request.access_token)
    
    # Get profile
    profile = await get_line_profile(request.access_token)
    
    # Find or create user
    user = await find_or_create_line_user(profile)
    
    # Generate tokens
    access_token = create_access_token(user)
    refresh_token, expires_at = create_refresh_token()
    
    # Save refresh token
    await save_refresh_token(user["id"], refresh_token, expires_at)
    
    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token,
    )

@app.post("/auth/email", response_model=TokenResponse)
async def login_with_email(request: EmailLoginRequest):
    """Login ด้วย Email/Password"""
    query = "SELECT * FROM users WHERE email = :email AND deleted_at IS NULL LIMIT 1"
    user = await database.fetch_one(query=query, values={"email": request.email})
    
    if not user or not user["password_hash"]:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    if not bcrypt.checkpw(
        request.password.encode(), 
        user["password_hash"].encode()
    ):
        raise HTTPException(status_code=401, detail="Invalid credentials")
    
    if not user["is_active"]:
        raise HTTPException(status_code=403, detail="Account disabled")
    
    user_dict = dict(user)
    access_token = create_access_token(user_dict)
    refresh_token, expires_at = create_refresh_token()
    await save_refresh_token(user_dict["id"], refresh_token, expires_at)
    
    return TokenResponse(
        access_token=access_token,
        refresh_token=refresh_token,
    )

@app.post("/auth/register")
async def register(request: RegisterRequest):
    """Register ด้วย Email"""
    # ตรวจสอบ email ซ้ำ
    existing = await database.fetch_one(
        "SELECT id FROM users WHERE email = :email",
        values={"email": request.email}
    )
    if existing:
        raise HTTPException(status_code=400, detail="Email already registered")
    
    # Hash password
    password_hash = bcrypt.hashpw(
        request.password.encode(), bcrypt.gensalt(12)
    ).decode()
    
    # สร้าง user
    query = """INSERT INTO users (email, password_hash, first_name, role) 
               VALUES (:email, :password_hash, :name, 'user')"""
    user_id = await database.execute(
        query=query,
        values={
            "email": request.email,
            "password_hash": password_hash,
            "name": request.name,
        }
    )
    
    return {"success": True, "user_id": user_id}

@app.post("/auth/refresh")
async def refresh_token(request: RefreshTokenRequest):
    """Refresh Access Token"""
    token_hash = hash_token(request.refresh_token)
    
    query = """
        SELECT rt.*, u.* FROM refresh_tokens rt
        JOIN users u ON rt.user_id = u.id
        WHERE rt.token_hash = :hash
          AND rt.is_revoked = FALSE
          AND rt.expires_at > NOW()
          AND u.is_active = TRUE
        LIMIT 1
    """
    result = await database.fetch_one(query=query, values={"hash": token_hash})
    
    if not result:
        raise HTTPException(status_code=401, detail="Invalid refresh token")
    
    user_data = dict(result)
    access_token = create_access_token(user_data)
    
    return {"access_token": access_token, "token_type": "bearer"}

@app.get("/auth/me", response_model=UserResponse)
async def get_me(user: dict = Depends(get_current_user)):
    """ดูข้อมูล user ปัจจุบัน"""
    return UserResponse(**user)

@app.post("/auth/logout")
async def logout(
    request: RefreshTokenRequest,
    user: dict = Depends(get_current_user)
):
    """Logout"""
    if request.refresh_token:
        token_hash = hash_token(request.refresh_token)
        await database.execute(
            "UPDATE refresh_tokens SET is_revoked = TRUE WHERE token_hash = :hash AND user_id = :uid",
            values={"hash": token_hash, "uid": user["id"]}
        )
    return {"success": True}

# =============================================
# PROTECTED ROUTES ตัวอย่าง
# =============================================
@app.get("/admin/users")
async def list_users(user: dict = Depends(require_role("admin"))):
    """เฉพาะ admin เท่านั้น"""
    users = await database.fetch_all("SELECT id, email, role FROM users")
    return {"users": [dict(u) for u in users]}

@app.get("/orders")
async def get_orders(user: dict = Depends(get_current_user)):
    """Authenticated users"""
    orders = await database.fetch_all(
        "SELECT * FROM orders WHERE user_id = :uid ORDER BY created_at DESC",
        values={"uid": user["id"]}
    )
    return {"orders": [dict(o) for o in orders]}

# =============================================
# HELPER FUNCTIONS
# =============================================
async def find_or_create_line_user(profile: dict) -> dict:
    """หาหรือสร้าง user จาก LINE profile"""
    existing = await database.fetch_one(
        "SELECT * FROM users WHERE line_user_id = :uid",
        values={"uid": profile["userId"]}
    )
    
    if existing:
        await database.execute(
            """UPDATE users SET line_display_name = :name, 
               line_picture_url = :pic, last_login_at = NOW()
               WHERE id = :id""",
            values={
                "name": profile["displayName"],
                "pic": profile.get("pictureUrl"),
                "id": existing["id"]
            }
        )
        return {**dict(existing), "line_display_name": profile["displayName"]}
    
    user_id = await database.execute(
        """INSERT INTO users (line_user_id, line_display_name, line_picture_url, role)
           VALUES (:uid, :name, :pic, 'user')""",
        values={
            "uid": profile["userId"],
            "name": profile["displayName"],
            "pic": profile.get("pictureUrl"),
        }
    )
    
    return await database.fetch_one(
        "SELECT * FROM users WHERE id = :id",
        values={"id": user_id}
    )

async def save_refresh_token(user_id: int, token: str, expires_at: datetime):
    """บันทึก refresh token"""
    token_hash = hash_token(token)
    await database.execute(
        """INSERT INTO refresh_tokens (user_id, token_hash, expires_at)
           VALUES (:uid, :hash, :expires)""",
        values={"uid": user_id, "hash": token_hash, "expires": expires_at}
    )

# Startup event
@app.on_event("startup")
async def startup():
    await database.connect()

@app.on_event("shutdown")
async def shutdown():
    await database.disconnect()
```

---

## 6. Security Best Practices

```javascript
// =============================================
// SECURITY BEST PRACTICES
// =============================================

// 1. Password hashing ด้วย bcrypt (cost factor 12+)
const bcrypt = require('bcryptjs');
const SALT_ROUNDS = 12;

async function hashPassword(password) {
  return bcrypt.hash(password, SALT_ROUNDS);
}

// 2. JWT Secret ที่แข็งแกร่ง
// สร้างด้วย: node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"

// 3. Rate Limiting ป้องกัน brute force
const rateLimit = require('express-rate-limit');
const loginLimiter = rateLimit({
  windowMs: 15 * 60 * 1000,
  max: 5,
  skipSuccessfulRequests: true,  // นับเฉพาะ failed requests
});

// 4. HTTPS Only - ตั้งค่า Secure Cookies
app.use(require('helmet')());

// 5. Input Validation
const Joi = require('joi');
const loginSchema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().min(8).max(128).required(),
});

// 6. SQL Injection Prevention - ใช้ parameterized queries
// ❌ อย่าทำ:
// const query = `SELECT * FROM users WHERE email = '${email}'`;

// ✅ ทำแบบนี้:
// pool.execute('SELECT * FROM users WHERE email = ?', [email]);

// 7. Token Rotation - เปลี่ยน refresh token ทุกครั้ง
async function rotateRefreshToken(oldToken, userId) {
  // Revoke old token
  await pool.execute(
    'UPDATE refresh_tokens SET is_revoked = TRUE WHERE token_hash = ?',
    [hashToken(oldToken)]
  );
  
  // Create new token
  const { token, expiresAt } = generateRefreshToken(userId);
  await saveRefreshToken(userId, token, expiresAt);
  
  return { token, expiresAt };
}

// 8. Secure Headers
const helmet = require('helmet');
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "https://static.line-scdn.net"],
      styleSrc: ["'self'", "'unsafe-inline'"],
    },
  },
  hsts: {
    maxAge: 31536000,
    includeSubDomains: true,
    preload: true,
  },
}));
```

---

## 7. Testing Authentication

```javascript
// tests/auth.test.js
const request = require('supertest');
const app = require('../src/app');

describe('Authentication API', () => {
  describe('POST /auth/email', () => {
    it('should login with valid credentials', async () => {
      const response = await request(app)
        .post('/auth/email')
        .send({ email: 'test@example.com', password: 'password123' });
      
      expect(response.status).toBe(200);
      expect(response.body.accessToken).toBeDefined();
      expect(response.body.refreshToken).toBeDefined();
      expect(response.body.user.password_hash).toBeUndefined();
    });

    it('should reject invalid credentials', async () => {
      const response = await request(app)
        .post('/auth/email')
        .send({ email: 'test@example.com', password: 'wrongpassword' });
      
      expect(response.status).toBe(401);
    });

    it('should rate limit after 5 failed attempts', async () => {
      for (let i = 0; i < 5; i++) {
        await request(app)
          .post('/auth/email')
          .send({ email: 'test@example.com', password: 'wrong' });
      }
      
      const response = await request(app)
        .post('/auth/email')
        .send({ email: 'test@example.com', password: 'wrong' });
      
      expect(response.status).toBe(429);
    });
  });

  describe('GET /auth/me', () => {
    it('should return user when authenticated', async () => {
      // Login first
      const loginRes = await request(app)
        .post('/auth/email')
        .send({ email: 'test@example.com', password: 'password123' });
      
      const { accessToken } = loginRes.body;
      
      const meRes = await request(app)
        .get('/auth/me')
        .set('Authorization', `Bearer ${accessToken}`);
      
      expect(meRes.status).toBe(200);
      expect(meRes.body.user.email).toBe('test@example.com');
    });

    it('should reject without token', async () => {
      const response = await request(app).get('/auth/me');
      expect(response.status).toBe(401);
    });
  });
});
```

---

## สรุป

ระบบ Authentication สำหรับ LINE bot ที่สมบูรณ์ต้องประกอบด้วย:

1. **LINE User ID** เป็น identifier หลัก - unique และเชื่อถือได้
2. **LINE ID Token Verification** - ตรวจสอบ identity จาก LINE เสมอ
3. **JWT Access Token** - stateless, อายุสั้น (1 ชั่วโมง)
4. **Refresh Token** - เก็บใน DB, อายุยาว (30 วัน), rotate ทุกครั้ง
5. **Account Linking** - เชื่อม LINE กับ account ในระบบ
6. **Role-Based Access Control** - admin, staff, user, guest
7. **Rate Limiting** - ป้องกัน brute force
8. **Security Headers** - Helmet, HTTPS, HSTS
9. **Login Logs** - audit trail สำหรับ security review
10. **Token Revocation** - logout จาก device ทั้งหมดได้

การ implement ที่ดีทำให้ LINE bot มีความปลอดภัยสูง ผู้ใช้ที่น่าเชื่อถือ และระบบที่ maintainable ในระยะยาว
