# Part 77: PDPA Compliance สำหรับ LINE OA ในประเทศไทย

## บทนำ

พระราชบัญญัติคุ้มครองข้อมูลส่วนบุคคล พ.ศ. 2562 (PDPA - Personal Data Protection Act) มีผลบังคับใช้อย่างเต็มรูปแบบในประเทศไทยตั้งแต่วันที่ 1 มิถุนายน 2565 

หากคุณให้บริการ LINE OA ในประเทศไทย คุณต้องปฏิบัติตาม PDPA อย่างเคร่งครัด มิฉะนั้นอาจถูกปรับสูงสุดถึง 5 ล้านบาท หรือจำคุกไม่เกิน 1 ปี

```
PDPA Overview:
┌──────────────────────────────────────────────────────────┐
│                 PDPA สำหรับ LINE OA                       │
│                                                           │
│  ข้อมูลที่ LINE เก็บ:                                     │
│  • User ID (ไม่ระบุตัวตน)                                 │
│  • Display Name                                           │
│  • Profile Picture URL                                    │
│  • Language Setting                                       │
│  • Status Message                                         │
│                                                           │
│  ข้อมูลที่ Bot ของคุณเก็บ:                                │
│  • ข้อความที่ผู้ใช้ส่ง                                    │
│  • ข้อมูลการสั่งซื้อ                                      │
│  • ข้อมูลติดต่อ (เบอร์โทร, อีเมล)                        │
│  • ประวัติการใช้งาน                                       │
│  • ข้อมูล Location (ถ้าเปิดใช้)                           │
└──────────────────────────────────────────────────────────┘
```

---

## 1. ข้อมูลที่ LINE เก็บรวบรวม

### 1.1 ประเภทข้อมูลจาก LINE Messaging API

```javascript
// ตัวอย่างข้อมูลที่ได้จาก LINE Profile API
const userProfile = {
  userId: "Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx", // ไม่ใช่ข้อมูลส่วนบุคคลโดยตรง
  displayName: "สมชาย ใจดี",                    // ชื่อเล่นใน LINE
  pictureUrl: "https://profile.line-scdn.net/...", // รูปโปรไฟล์
  statusMessage: "Hello World",                   // สถานะ
  language: "th"                                  // ภาษา
};

// ข้อมูลจาก Webhook Events
const webhookEventData = {
  // Message Events
  message: {
    id: "12345",
    type: "text",
    text: "สวัสดี"                              // เนื้อหาข้อความ
  },
  
  // Location Message
  location: {
    title: "บ้านของฉัน",
    address: "123 ถนนสุขุมวิท กรุงเทพฯ",       // ข้อมูล Location = sensitive!
    latitude: 13.7563,
    longitude: 100.5018
  },
  
  // Source
  source: {
    type: "user",
    userId: "Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
  }
};
```

### 1.2 การจำแนกประเภทข้อมูล

```javascript
// models/dataClassification.js

const DATA_CATEGORIES = {
  // ข้อมูลส่วนบุคคลทั่วไป
  PERSONAL: {
    displayName: 'ชื่อที่แสดงใน LINE',
    phoneNumber: 'หมายเลขโทรศัพท์',
    email: 'อีเมล',
    address: 'ที่อยู่',
    birthDate: 'วันเดือนปีเกิด'
  },
  
  // ข้อมูลส่วนบุคคลที่อ่อนไหว (Sensitive Data)
  SENSITIVE: {
    healthData: 'ข้อมูลสุขภาพ',
    financialData: 'ข้อมูลทางการเงิน',
    locationHistory: 'ประวัติตำแหน่งที่ตั้ง',
    biometricData: 'ข้อมูลชีวมิติ'
  },
  
  // ข้อมูลที่ไม่ใช่ส่วนบุคคล
  NON_PERSONAL: {
    lineUserId: 'LINE User ID (เป็น pseudonymous)',
    sessionData: 'ข้อมูล session',
    analyticsData: 'ข้อมูล analytics แบบ aggregate'
  }
};

// กำหนด retention policy ตามประเภทข้อมูล
const RETENTION_POLICY = {
  messages: 90,        // 90 วัน
  orders: 2555,        // 7 ปี (ตามกฎหมายบัญชี)
  userProfile: 365,    // 1 ปีหลังจากไม่ active
  locationData: 30,    // 30 วัน
  sensitiveData: 90    // 90 วัน
};

module.exports = { DATA_CATEGORIES, RETENTION_POLICY };
```

---

## 2. Consent Management

### 2.1 Consent Flow Design

```
Consent Collection Flow:
┌─────────────────────────────────────────────────────────────┐
│  User เริ่มใช้งาน LINE OA ครั้งแรก                          │
│           ↓                                                  │
│  แสดง Privacy Notice                                        │
│           ↓                                                  │
│  ขอ Consent สำหรับแต่ละวัตถุประสงค์                         │
│    ✓ การให้บริการ (จำเป็น)                                   │
│    ✓ การส่ง Newsletter (ตัวเลือก)                           │
│    ✓ การวิเคราะห์พฤติกรรม (ตัวเลือก)                       │
│           ↓                                                  │
│  บันทึก Consent พร้อม timestamp                            │
│           ↓                                                  │
│  เริ่มให้บริการ                                              │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Database Schema สำหรับ Consent Management

```sql
-- migrations/create_consent_tables.sql

-- ตาราง consent records
CREATE TABLE user_consents (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  line_user_id VARCHAR(50) NOT NULL,
  
  -- ประเภทของ consent
  consent_type ENUM(
    'service_provision',    -- จำเป็นสำหรับบริการ
    'marketing',            -- การตลาด
    'analytics',            -- การวิเคราะห์
    'third_party_sharing',  -- แชร์กับ third party
    'location'              -- ข้อมูลตำแหน่ง
  ) NOT NULL,
  
  -- สถานะ
  status ENUM('granted', 'denied', 'withdrawn') NOT NULL,
  
  -- เวอร์ชันนโยบาย
  policy_version VARCHAR(10) NOT NULL,
  
  -- วันที่และเวลา
  consented_at DATETIME NOT NULL,
  expires_at DATETIME,
  withdrawn_at DATETIME,
  
  -- บันทึกเพิ่มเติม
  ip_address VARCHAR(45),
  user_agent TEXT,
  method ENUM('liff_form', 'chat_button', 'api') NOT NULL,
  
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  
  -- Index
  INDEX idx_line_user_id (line_user_id),
  INDEX idx_consent_type_status (consent_type, status),
  INDEX idx_consented_at (consented_at)
);

-- ตาราง consent audit log (ไม่เคยลบ)
CREATE TABLE consent_audit_log (
  id BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  line_user_id VARCHAR(50) NOT NULL,
  consent_type VARCHAR(50) NOT NULL,
  action ENUM('granted', 'denied', 'withdrawn', 'expired') NOT NULL,
  policy_version VARCHAR(10) NOT NULL,
  performed_at DATETIME NOT NULL,
  performed_by VARCHAR(50),  -- 'user' หรือ system
  reason TEXT,
  metadata JSON,
  
  INDEX idx_line_user_id (line_user_id),
  INDEX idx_performed_at (performed_at)
);

-- ตาราง data processing purposes
CREATE TABLE processing_purposes (
  id INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  description TEXT NOT NULL,
  legal_basis ENUM(
    'consent',
    'contract',
    'legal_obligation', 
    'vital_interest',
    'public_task',
    'legitimate_interest'
  ) NOT NULL,
  is_required BOOLEAN DEFAULT FALSE,
  retention_days INT UNSIGNED,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- เพิ่มข้อมูล purposes
INSERT INTO processing_purposes 
  (name, description, legal_basis, is_required, retention_days)
VALUES 
  ('service_provision', 'การให้บริการ LINE OA', 'contract', TRUE, 365),
  ('marketing', 'การส่งโปรโมชั่นและข่าวสาร', 'consent', FALSE, 365),
  ('analytics', 'การวิเคราะห์พฤติกรรมการใช้งาน', 'consent', FALSE, 90),
  ('third_party_sharing', 'การแชร์ข้อมูลกับพาร์ทเนอร์', 'consent', FALSE, 90);
```

### 2.3 Consent Management Service

```javascript
// services/consentManager.js

class ConsentManager {
  constructor(db) {
    this.db = db;
    this.currentPolicyVersion = process.env.PRIVACY_POLICY_VERSION || '1.0';
  }
  
  /**
   * ตรวจสอบว่า user มี consent สำหรับวัตถุประสงค์ที่ต้องการหรือไม่
   */
  async hasConsent(lineUserId, consentType) {
    const [rows] = await this.db.execute(
      `SELECT id FROM user_consents 
       WHERE line_user_id = ? 
       AND consent_type = ?
       AND status = 'granted'
       AND (expires_at IS NULL OR expires_at > NOW())
       ORDER BY consented_at DESC
       LIMIT 1`,
      [lineUserId, consentType]
    );
    
    return rows.length > 0;
  }
  
  /**
   * บันทึก consent ที่ได้รับ
   */
  async recordConsent(lineUserId, consentType, status, method, metadata = {}) {
    const connection = await this.db.getConnection();
    
    try {
      await connection.beginTransaction();
      
      // อัปเดต consent record
      await connection.execute(
        `INSERT INTO user_consents 
         (line_user_id, consent_type, status, policy_version, consented_at, method)
         VALUES (?, ?, ?, ?, NOW(), ?)
         ON DUPLICATE KEY UPDATE
           status = VALUES(status),
           consented_at = VALUES(consented_at),
           withdrawn_at = IF(VALUES(status) = 'withdrawn', NOW(), NULL)`,
        [lineUserId, consentType, status, this.currentPolicyVersion, method]
      );
      
      // บันทึก audit log (ไม่เคยลบ)
      await connection.execute(
        `INSERT INTO consent_audit_log 
         (line_user_id, consent_type, action, policy_version, performed_at, performed_by, metadata)
         VALUES (?, ?, ?, ?, NOW(), 'user', ?)`,
        [lineUserId, consentType, status, this.currentPolicyVersion, 
         JSON.stringify(metadata)]
      );
      
      await connection.commit();
      
      console.log(`Consent recorded: ${lineUserId} ${consentType} ${status}`);
    } catch (err) {
      await connection.rollback();
      throw err;
    } finally {
      connection.release();
    }
  }
  
  /**
   * ดึงสรุป consent ของ user ทั้งหมด
   */
  async getUserConsentSummary(lineUserId) {
    const [rows] = await this.db.execute(
      `SELECT 
         consent_type,
         status,
         consented_at,
         policy_version
       FROM user_consents uc1
       WHERE line_user_id = ?
       AND consented_at = (
         SELECT MAX(consented_at) 
         FROM user_consents uc2
         WHERE uc2.line_user_id = uc1.line_user_id
         AND uc2.consent_type = uc1.consent_type
       )`,
      [lineUserId]
    );
    
    return rows.reduce((acc, row) => {
      acc[row.consent_type] = {
        status: row.status,
        grantedAt: row.consented_at,
        policyVersion: row.policy_version
      };
      return acc;
    }, {});
  }
  
  /**
   * ถอน consent ทั้งหมด (เมื่อ user ขอลบข้อมูล)
   */
  async withdrawAllConsents(lineUserId, reason = 'user_request') {
    await this.db.execute(
      `UPDATE user_consents 
       SET status = 'withdrawn', withdrawn_at = NOW()
       WHERE line_user_id = ? AND status = 'granted'`,
      [lineUserId]
    );
    
    await this.db.execute(
      `INSERT INTO consent_audit_log 
       (line_user_id, consent_type, action, policy_version, performed_at, performed_by, reason)
       SELECT 
         line_user_id, consent_type, 'withdrawn', ?, NOW(), 'user', ?
       FROM user_consents
       WHERE line_user_id = ?`,
      [this.currentPolicyVersion, reason, lineUserId]
    );
  }
  
  /**
   * ตรวจสอบว่า user ต้องได้รับ consent ใหม่หรือไม่
   * (เมื่อ policy version เปลี่ยน)
   */
  async needsConsentUpdate(lineUserId) {
    const [rows] = await this.db.execute(
      `SELECT COUNT(*) as count
       FROM user_consents
       WHERE line_user_id = ?
       AND status = 'granted'
       AND policy_version = ?
       AND consent_type = 'service_provision'`,
      [lineUserId, this.currentPolicyVersion]
    );
    
    return rows[0].count === 0;
  }
}

module.exports = ConsentManager;
```

### 2.4 Consent Collection ผ่าน LIFF

```javascript
// liff/consent/consentForm.js

class ConsentForm {
  constructor() {
    this.consentData = {};
  }
  
  async init() {
    await liff.init({ liffId: process.env.LIFF_ID_CONSENT });
    
    // ตรวจสอบว่า user มี consent แล้วหรือไม่
    const profile = await liff.getProfile();
    const hasExistingConsent = await this.checkExistingConsent(profile.userId);
    
    if (hasExistingConsent) {
      // แสดงหน้าจัดการ consent
      this.renderConsentManagement(profile);
    } else {
      // แสดงหน้าขอ consent ครั้งแรก
      this.renderInitialConsent(profile);
    }
  }
  
  renderInitialConsent(profile) {
    const html = `
      <div class="consent-container">
        <div class="consent-header">
          <img src="${profile.pictureUrl}" alt="Profile" class="profile-pic">
          <h2>สวัสดีคุณ ${this.escapeHtml(profile.displayName)}</h2>
        </div>
        
        <div class="privacy-notice">
          <h3>🔒 นโยบายความเป็นส่วนตัว</h3>
          <p>
            เพื่อให้บริการที่ดีที่สุดแก่คุณ เราจำเป็นต้องเก็บรวบรวมและประมวลผลข้อมูลส่วนบุคคลของคุณ
            ตามพระราชบัญญัติคุ้มครองข้อมูลส่วนบุคคล พ.ศ. 2562 (PDPA)
          </p>
          
          <details>
            <summary>ดูรายละเอียดข้อมูลที่เราเก็บ</summary>
            <ul>
              <li>ชื่อที่แสดงใน LINE และรูปโปรไฟล์</li>
              <li>ข้อความที่คุณส่งหาเรา</li>
              <li>ประวัติการสั่งซื้อ</li>
              <li>ข้อมูลติดต่อที่คุณให้ไว้</li>
            </ul>
          </details>
        </div>
        
        <form id="consentForm">
          <!-- Required consent - ปิดไม่ได้ -->
          <div class="consent-item required">
            <label class="consent-label">
              <input type="checkbox" checked disabled name="service_provision">
              <span class="checkmark"></span>
              <div class="consent-text">
                <strong>การให้บริการ (จำเป็น)</strong>
                <p>เพื่อให้บริการ LINE OA อย่างครบถ้วน รวมถึงการตอบคำถามและประมวลผลคำสั่ง</p>
              </div>
            </label>
          </div>
          
          <!-- Optional consent -->
          <div class="consent-item">
            <label class="consent-label">
              <input type="checkbox" name="marketing" id="marketingConsent">
              <span class="checkmark"></span>
              <div class="consent-text">
                <strong>การรับข่าวสารและโปรโมชั่น (ตัวเลือก)</strong>
                <p>เพื่อส่งข้อมูลโปรโมชั่น ข่าวสาร และสิทธิพิเศษต่าง ๆ ให้คุณ</p>
              </div>
            </label>
          </div>
          
          <div class="consent-item">
            <label class="consent-label">
              <input type="checkbox" name="analytics" id="analyticsConsent">
              <span class="checkmark"></span>
              <div class="consent-text">
                <strong>การวิเคราะห์พฤติกรรม (ตัวเลือก)</strong>
                <p>เพื่อปรับปรุงบริการให้ตรงกับความต้องการของคุณมากขึ้น</p>
              </div>
            </label>
          </div>
          
          <div class="consent-links">
            <a href="/privacy-policy" target="_blank">นโยบายความเป็นส่วนตัว</a> |
            <a href="/terms-of-service" target="_blank">ข้อกำหนดการใช้งาน</a>
          </div>
          
          <div class="consent-buttons">
            <button type="submit" class="btn-accept">ยืนยันและเริ่มใช้งาน</button>
            <button type="button" class="btn-decline" onclick="handleDecline()">
              ไม่ยอมรับ
            </button>
          </div>
        </form>
      </div>
    `;
    
    document.getElementById('app').innerHTML = html;
    
    document.getElementById('consentForm').addEventListener('submit', (e) => {
      e.preventDefault();
      this.submitConsent(profile.userId);
    });
  }
  
  async submitConsent(userId) {
    const formData = new FormData(document.getElementById('consentForm'));
    
    const consents = {
      service_provision: true, // เสมอ true
      marketing: formData.get('marketing') === 'on',
      analytics: formData.get('analytics') === 'on'
    };
    
    // ขอ ID Token เพื่อ verify ที่ server
    const idToken = liff.getIDToken();
    
    try {
      const response = await fetch('/api/consent/record', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${idToken}`
        },
        body: JSON.stringify({
          consents,
          method: 'liff_form',
          timestamp: new Date().toISOString()
        })
      });
      
      if (response.ok) {
        // ปิด LIFF และกลับไปที่ LINE Chat
        liff.closeWindow();
      }
    } catch (err) {
      console.error('Failed to record consent:', err);
      alert('เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง');
    }
  }
  
  escapeHtml(str) {
    const div = document.createElement('div');
    div.textContent = str;
    return div.innerHTML;
  }
}

const form = new ConsentForm();
form.init();
```

### 2.5 Consent Collection ผ่าน Chat Bot

```javascript
// handlers/consentHandler.js

const CONSENT_FLOW_MESSAGES = {
  initial: {
    type: 'flex',
    altText: 'ยืนยันนโยบายความเป็นส่วนตัว',
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [{
          type: 'text',
          text: '🔒 นโยบายความเป็นส่วนตัว',
          weight: 'bold',
          size: 'lg',
          color: '#1DB446'
        }]
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: 'เราเก็บข้อมูลดังต่อไปนี้:',
            wrap: true
          },
          {
            type: 'text',
            text: '• ชื่อและรูปโปรไฟล์ LINE\n• ข้อความที่คุณส่ง\n• ประวัติการสั่งซื้อ',
            wrap: true,
            size: 'sm',
            color: '#555555',
            margin: 'sm'
          },
          {
            type: 'text',
            text: 'เพื่อดูรายละเอียดเพิ่มเติม กรุณาอ่านนโยบายความเป็นส่วนตัวของเรา',
            wrap: true,
            size: 'sm',
            margin: 'md'
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
              type: 'postback',
              label: '✅ ยอมรับทั้งหมด',
              data: 'action=consent&type=all&value=grant'
            }
          },
          {
            type: 'button',
            style: 'secondary',
            action: {
              type: 'postback',
              label: '⚙️ เลือกเอง',
              data: 'action=consent&type=custom'
            }
          },
          {
            type: 'button',
            style: 'link',
            action: {
              type: 'uri',
              label: 'อ่านนโยบายความเป็นส่วนตัว',
              uri: 'https://your-domain.com/privacy-policy'
            }
          }
        ]
      }
    }
  }
};

async function handleConsentFlow(event) {
  const { replyToken, source, postback } = event;
  const userId = source.userId;
  
  const data = new URLSearchParams(postback?.data || '');
  const action = data.get('action');
  const type = data.get('type');
  const value = data.get('value');
  
  if (!action || action !== 'consent') return;
  
  if (type === 'all' && value === 'grant') {
    // บันทึก consent ทั้งหมด
    await consentManager.recordConsent(userId, 'service_provision', 'granted', 'chat_button');
    await consentManager.recordConsent(userId, 'marketing', 'granted', 'chat_button');
    await consentManager.recordConsent(userId, 'analytics', 'granted', 'chat_button');
    
    await lineClient.replyMessage(replyToken, {
      type: 'text',
      text: '✅ ขอบคุณที่ยอมรับนโยบายของเรา\nคุณสามารถจัดการการยินยอมได้ตลอดเวลาโดยพิมพ์ "การตั้งค่าความเป็นส่วนตัว"'
    });
  } else if (type === 'custom') {
    // ส่งลิงก์ไปยัง LIFF form
    await lineClient.replyMessage(replyToken, {
      type: 'text',
      text: `กรุณาเลือกการตั้งค่าความเป็นส่วนตัวของคุณ: ${process.env.LIFF_CONSENT_URL}`
    });
  }
}

module.exports = { handleConsentFlow };
```

---

## 3. Privacy Policy Requirements

### 3.1 Privacy Policy Template (ภาษาไทย)

```markdown
# นโยบายความเป็นส่วนตัว
## [ชื่อบริษัท] - LINE Official Account

**วันที่มีผลบังคับใช้**: [วันที่]
**เวอร์ชัน**: 1.0

---

## 1. บทนำ

[ชื่อบริษัท] ("เรา", "บริษัท") ให้ความสำคัญกับการปกป้องข้อมูลส่วนบุคคลของคุณ 
นโยบายความเป็นส่วนตัวนี้อธิบายว่าเราเก็บรวบรวม ใช้ และเปิดเผยข้อมูลส่วนบุคคล
ของคุณเมื่อคุณใช้บริการ LINE Official Account ของเรา

## 2. ข้อมูลที่เราเก็บรวบรวม

### 2.1 ข้อมูลจาก LINE Platform
เมื่อคุณติดต่อเราผ่าน LINE OA เราจะได้รับข้อมูลดังนี้:
- LINE User ID (รหัสที่ LINE กำหนดให้)
- ชื่อที่แสดงใน LINE
- รูปโปรไฟล์ LINE
- ข้อความที่คุณส่งมาหาเรา
- ข้อมูลตำแหน่ง (เฉพาะเมื่อคุณแชร์ด้วยตนเอง)

### 2.2 ข้อมูลที่คุณให้เพิ่มเติม
- ชื่อ-นามสกุล
- หมายเลขโทรศัพท์
- อีเมล
- ที่อยู่จัดส่ง

### 2.3 ข้อมูลการใช้งาน
- ประวัติการสั่งซื้อ
- ประวัติการใช้บริการ
- การตอบสนองต่อโปรโมชั่น

## 3. วัตถุประสงค์การใช้ข้อมูล

| วัตถุประสงค์ | ฐานทางกฎหมาย | ระยะเวลาเก็บข้อมูล |
|-------------|--------------|-------------------|
| การให้บริการ | สัญญา | ตลอดอายุสัญญา + 1 ปี |
| การส่งโปรโมชั่น | ความยินยอม | 1 ปี |
| การวิเคราะห์ | ความยินยอม | 90 วัน |
| การปฏิบัติตามกฎหมาย | พันธะทางกฎหมาย | ตามที่กฎหมายกำหนด |

## 4. การแบ่งปันข้อมูล

เราจะไม่ขายหรือแบ่งปันข้อมูลส่วนบุคคลของคุณกับบุคคลภายนอก 
ยกเว้นในกรณีต่อไปนี้:

- ผู้ให้บริการที่เราจ้างให้ประมวลผลข้อมูลในนามเรา (Data Processors)
- เมื่อได้รับความยินยอมจากคุณ
- เมื่อมีพันธะทางกฎหมาย

## 5. สิทธิของคุณตาม PDPA

คุณมีสิทธิดังนี้:
- **สิทธิในการเข้าถึง**: ขอดูข้อมูลที่เราเก็บ
- **สิทธิในการแก้ไข**: ขอแก้ไขข้อมูลที่ไม่ถูกต้อง
- **สิทธิในการลบ**: ขอลบข้อมูล ("Right to be Forgotten")
- **สิทธิในการคัดค้าน**: คัดค้านการประมวลผลข้อมูล
- **สิทธิในการโอนย้ายข้อมูล**: ขอรับสำเนาข้อมูลในรูปแบบที่ใช้งานได้
- **สิทธิในการถอนความยินยอม**: ถอนความยินยอมได้ทุกเมื่อ

## 6. การติดต่อ

ผู้ควบคุมข้อมูลส่วนบุคคล (Data Controller):
- [ชื่อบริษัท]
- ที่อยู่: [ที่อยู่]
- อีเมล: privacy@your-company.com
- โทรศัพท์: [เบอร์โทรศัพท์]

เจ้าหน้าที่คุ้มครองข้อมูลส่วนบุคคล (DPO):
- อีเมล: dpo@your-company.com
```

---

## 4. Data Storage Guidelines

### 4.1 การเก็บข้อมูลอย่างปลอดภัย

```javascript
// models/UserData.js (Sequelize)
const { DataTypes } = require('sequelize');
const crypto = require('crypto');

// Encryption helper
const ENCRYPTION_KEY = Buffer.from(process.env.DATA_ENCRYPTION_KEY, 'hex');

function encrypt(text) {
  const iv = crypto.randomBytes(16);
  const cipher = crypto.createCipheriv('aes-256-gcm', ENCRYPTION_KEY, iv);
  let encrypted = cipher.update(text, 'utf8', 'hex');
  encrypted += cipher.final('hex');
  const authTag = cipher.getAuthTag();
  return `${iv.toString('hex')}:${authTag.toString('hex')}:${encrypted}`;
}

function decrypt(encryptedText) {
  const [ivHex, authTagHex, encrypted] = encryptedText.split(':');
  const iv = Buffer.from(ivHex, 'hex');
  const authTag = Buffer.from(authTagHex, 'hex');
  const decipher = crypto.createDecipheriv('aes-256-gcm', ENCRYPTION_KEY, iv);
  decipher.setAuthTag(authTag);
  let decrypted = decipher.update(encrypted, 'hex', 'utf8');
  decrypted += decipher.final('utf8');
  return decrypted;
}

const UserData = sequelize.define('UserData', {
  id: {
    type: DataTypes.BIGINT.UNSIGNED,
    autoIncrement: true,
    primaryKey: true
  },
  lineUserId: {
    type: DataTypes.STRING(50),
    allowNull: false,
    unique: true
  },
  // ข้อมูลที่ต้องเข้ารหัส
  phoneNumber: {
    type: DataTypes.TEXT,
    get() {
      const raw = this.getDataValue('phoneNumber');
      return raw ? decrypt(raw) : null;
    },
    set(value) {
      this.setDataValue('phoneNumber', value ? encrypt(value) : null);
    }
  },
  email: {
    type: DataTypes.TEXT,
    get() {
      const raw = this.getDataValue('email');
      return raw ? decrypt(raw) : null;
    },
    set(value) {
      this.setDataValue('email', value ? encrypt(value) : null);
    }
  },
  // Hash สำหรับ lookup (ไม่เปิดเผยค่าจริง)
  phoneHash: {
    type: DataTypes.STRING(64),
    comment: 'SHA-256 hash of phone number for lookups'
  },
  // Pseudonymization
  internalId: {
    type: DataTypes.UUID,
    defaultValue: DataTypes.UUIDV4,
    comment: 'Internal ID for analytics (not linked to real identity)'
  },
  // Data retention
  lastActiveAt: {
    type: DataTypes.DATE
  },
  dataExpiresAt: {
    type: DataTypes.DATE,
    comment: 'Data will be deleted after this date'
  }
}, {
  tableName: 'user_data',
  timestamps: true,
  paranoid: true  // soft delete
});

module.exports = UserData;
```

---

## 5. User Data Deletion (Right to be Forgotten)

### 5.1 Data Deletion Handler

```javascript
// services/dataDeleteService.js

class DataDeleteService {
  constructor(db, lineClient) {
    this.db = db;
    this.lineClient = lineClient;
  }
  
  /**
   * ลบข้อมูลทั้งหมดของ user ตาม PDPA Right to Erasure
   */
  async deleteUserData(lineUserId, requestedBy = 'user') {
    const deletionId = crypto.randomUUID();
    const startedAt = new Date();
    
    console.log(`Starting data deletion for user: ${lineUserId}, deletion ID: ${deletionId}`);
    
    // บันทึก deletion request ก่อน
    await this.createDeletionRecord(lineUserId, deletionId, requestedBy);
    
    try {
      const connection = await this.db.getConnection();
      
      try {
        await connection.beginTransaction();
        
        // 1. ลบ messages
        const [msgResult] = await connection.execute(
          'DELETE FROM messages WHERE line_user_id = ?',
          [lineUserId]
        );
        
        // 2. Anonymize orders (ไม่ลบเพราะต้องเก็บตามกฎหมาย แต่ anonymize)
        await connection.execute(
          `UPDATE orders SET 
             line_user_id = 'DELETED',
             customer_name = 'Deleted User',
             phone_number = NULL,
             email = NULL,
             shipping_address = 'REDACTED'
           WHERE line_user_id = ?`,
          [lineUserId]
        );
        
        // 3. ลบ user profile data
        await connection.execute(
          'DELETE FROM user_data WHERE line_user_id = ?',
          [lineUserId]
        );
        
        // 4. ถอน consents ทั้งหมด
        await connection.execute(
          `UPDATE user_consents SET 
             status = 'withdrawn', 
             withdrawn_at = NOW()
           WHERE line_user_id = ?`,
          [lineUserId]
        );
        
        // 5. ลบ session data
        await redisClient.del(`session:${lineUserId}`);
        await redisClient.del(`token:user:${lineUserId}`);
        
        await connection.commit();
        
        // อัปเดต deletion record
        await this.updateDeletionRecord(deletionId, 'completed', {
          messagesDeleted: msgResult.affectedRows,
          completedAt: new Date()
        });
        
        // แจ้ง user
        await this.notifyUserOfDeletion(lineUserId, deletionId);
        
        console.log(`Data deletion completed for ${lineUserId}, ID: ${deletionId}`);
        
        return { success: true, deletionId };
        
      } catch (err) {
        await connection.rollback();
        throw err;
      } finally {
        connection.release();
      }
      
    } catch (err) {
      await this.updateDeletionRecord(deletionId, 'failed', {
        error: err.message
      });
      throw err;
    }
  }
  
  async createDeletionRecord(lineUserId, deletionId, requestedBy) {
    await this.db.execute(
      `INSERT INTO data_deletion_requests 
       (deletion_id, line_user_id, requested_by, status, requested_at)
       VALUES (?, ?, ?, 'processing', NOW())`,
      [deletionId, lineUserId, requestedBy]
    );
  }
  
  async updateDeletionRecord(deletionId, status, metadata) {
    await this.db.execute(
      `UPDATE data_deletion_requests 
       SET status = ?, metadata = ?, updated_at = NOW()
       WHERE deletion_id = ?`,
      [status, JSON.stringify(metadata), deletionId]
    );
  }
  
  async notifyUserOfDeletion(lineUserId, deletionId) {
    // ลองส่งข้อความก่อนลบ
    try {
      await this.lineClient.pushMessage(lineUserId, {
        type: 'text',
        text: `✅ เราได้ลบข้อมูลส่วนบุคคลของคุณออกจากระบบแล้ว\n\nรหัสอ้างอิง: ${deletionId}\nหากคุณมีข้อสงสัย กรุณาติดต่อเราที่ privacy@your-company.com`
      });
    } catch (err) {
      // User อาจ block bot แล้ว ไม่ต้อง throw
      console.log(`Could not notify user ${lineUserId} about deletion`);
    }
  }
}

module.exports = DataDeleteService;
```

### 5.2 Bot Command สำหรับการลบข้อมูล

```javascript
// handlers/privacyHandler.js

async function handlePrivacyCommand(event) {
  const { replyToken, source, message } = event;
  const userId = source.userId;
  const text = message.text.trim().toLowerCase();
  
  if (text === 'ลบข้อมูล' || text === 'delete my data') {
    // ขอการยืนยันก่อนลบ
    await lineClient.replyMessage(replyToken, {
      type: 'flex',
      altText: 'ยืนยันการลบข้อมูล',
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: '⚠️ ยืนยันการลบข้อมูล',
              weight: 'bold',
              size: 'lg',
              color: '#FF4444'
            },
            {
              type: 'text',
              text: 'การลบข้อมูลจะทำให้:\n• ประวัติการสนทนาถูกลบทั้งหมด\n• ข้อมูลส่วนตัวของคุณถูกลบออกจากระบบ\n• คุณอาจต้องลงทะเบียนใหม่\n\nข้อมูลคำสั่งซื้อจะถูกเก็บไว้ตามที่กฎหมายกำหนด แต่จะไม่สามารถระบุตัวตนคุณได้',
              wrap: true,
              size: 'sm',
              margin: 'md'
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
              color: '#FF4444',
              action: {
                type: 'postback',
                label: '🗑️ ยืนยันลบข้อมูล',
                data: 'action=delete_data&confirmed=true'
              }
            },
            {
              type: 'button',
              style: 'secondary',
              action: {
                type: 'postback',
                label: '❌ ยกเลิก',
                data: 'action=delete_data&confirmed=false'
              }
            }
          ]
        }
      }
    });
  }
  
  if (text === 'การตั้งค่าความเป็นส่วนตัว' || text === 'privacy settings') {
    const summary = await consentManager.getUserConsentSummary(userId);
    await showPrivacySettings(replyToken, summary);
  }
}

async function handleDeleteConfirmation(event) {
  const { replyToken, source, postback } = event;
  const userId = source.userId;
  const data = new URLSearchParams(postback.data);
  
  if (data.get('action') !== 'delete_data') return;
  
  if (data.get('confirmed') === 'true') {
    try {
      const result = await deleteService.deleteUserData(userId, 'user_request');
      
      await lineClient.replyMessage(replyToken, {
        type: 'text',
        text: `✅ ดำเนินการลบข้อมูลสำเร็จ\nรหัสอ้างอิง: ${result.deletionId}`
      });
    } catch (err) {
      await lineClient.replyMessage(replyToken, {
        type: 'text',
        text: '❌ เกิดข้อผิดพลาดในการลบข้อมูล กรุณาติดต่อ privacy@your-company.com'
      });
    }
  } else {
    await lineClient.replyMessage(replyToken, {
      type: 'text',
      text: '✅ ยกเลิกการลบข้อมูลแล้ว'
    });
  }
}
```

---

## 6. Data Export (Right to Data Portability)

### 6.1 Data Export Handler

```javascript
// services/dataExportService.js

class DataExportService {
  constructor(db) {
    this.db = db;
  }
  
  /**
   * ส่งออกข้อมูลทั้งหมดของ user ในรูปแบบ JSON/CSV
   */
  async exportUserData(lineUserId) {
    const exportId = crypto.randomUUID();
    
    try {
      // เก็บข้อมูลทั้งหมด
      const [profile] = await this.db.execute(
        'SELECT * FROM user_data WHERE line_user_id = ?',
        [lineUserId]
      );
      
      const [messages] = await this.db.execute(
        `SELECT type, content, created_at 
         FROM messages 
         WHERE line_user_id = ? 
         ORDER BY created_at DESC
         LIMIT 1000`,
        [lineUserId]
      );
      
      const [orders] = await this.db.execute(
        `SELECT order_id, items, total_amount, status, created_at
         FROM orders 
         WHERE line_user_id = ?
         ORDER BY created_at DESC`,
        [lineUserId]
      );
      
      const [consents] = await this.db.execute(
        `SELECT consent_type, status, consented_at, policy_version
         FROM user_consents
         WHERE line_user_id = ?`,
        [lineUserId]
      );
      
      const exportData = {
        exportInfo: {
          exportId,
          exportDate: new Date().toISOString(),
          lineUserId,
          format: '1.0'
        },
        personalData: {
          profile: profile[0] || {},
          consents
        },
        activityData: {
          messages: messages.map(m => ({
            type: m.type,
            content: m.content,
            timestamp: m.created_at
          })),
          orders: orders.map(o => ({
            orderId: o.order_id,
            items: o.items,
            total: o.total_amount,
            status: o.status,
            date: o.created_at
          }))
        }
      };
      
      // บันทึก export request
      await this.db.execute(
        `INSERT INTO data_export_requests 
         (export_id, line_user_id, status, requested_at)
         VALUES (?, ?, 'completed', NOW())`,
        [exportId, lineUserId]
      );
      
      return exportData;
      
    } catch (err) {
      await this.db.execute(
        `INSERT INTO data_export_requests 
         (export_id, line_user_id, status, error, requested_at)
         VALUES (?, ?, 'failed', ?, NOW())`,
        [exportId, lineUserId, err.message]
      );
      throw err;
    }
  }
}

// API endpoint สำหรับ data export
app.get('/api/my-data', authenticateUser, async (req, res) => {
  const lineUserId = req.user.lineUserId;
  
  try {
    const exportData = await dataExportService.exportUserData(lineUserId);
    
    // ส่งกลับเป็น JSON หรือดาวน์โหลดเป็นไฟล์
    if (req.query.format === 'download') {
      res.setHeader('Content-Disposition', `attachment; filename="my-data-${Date.now()}.json"`);
      res.setHeader('Content-Type', 'application/json');
      res.send(JSON.stringify(exportData, null, 2));
    } else {
      res.json(exportData);
    }
    
  } catch (err) {
    res.status(500).json({ error: 'ไม่สามารถส่งออกข้อมูลได้ในขณะนี้' });
  }
});
```

---

## 7. Audit Trails

### 7.1 Audit Logging System

```javascript
// services/auditLogger.js

class AuditLogger {
  constructor(db) {
    this.db = db;
  }
  
  /**
   * บันทึก audit log ทุกการกระทำที่เกี่ยวกับข้อมูลส่วนบุคคล
   */
  async log(action, details) {
    const {
      actorId,    // ผู้กระทำ (user หรือ system)
      actorType,  // 'user', 'admin', 'system'
      entityId,   // ID ของข้อมูลที่ถูกกระทำ
      entityType, // ประเภทข้อมูล
      oldData,    // ข้อมูลก่อนเปลี่ยน
      newData,    // ข้อมูลหลังเปลี่ยน
      ipAddress,
      userAgent,
      success,
      failureReason
    } = details;
    
    try {
      await this.db.execute(
        `INSERT INTO audit_logs 
         (action, actor_id, actor_type, entity_id, entity_type,
          old_data, new_data, ip_address, user_agent, 
          success, failure_reason, performed_at)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, NOW())`,
        [
          action,
          actorId,
          actorType,
          entityId,
          entityType,
          oldData ? JSON.stringify(oldData) : null,
          newData ? JSON.stringify(newData) : null,
          ipAddress,
          userAgent,
          success ? 1 : 0,
          failureReason || null
        ]
      );
    } catch (err) {
      // Audit log ไม่ควร throw error ที่ทำให้ business logic ล้มเหลว
      console.error('Failed to write audit log:', err);
    }
  }
  
  /**
   * ค้นหา audit logs สำหรับ DPO
   */
  async searchLogs(filters) {
    const conditions = [];
    const params = [];
    
    if (filters.actorId) {
      conditions.push('actor_id = ?');
      params.push(filters.actorId);
    }
    
    if (filters.action) {
      conditions.push('action = ?');
      params.push(filters.action);
    }
    
    if (filters.dateFrom) {
      conditions.push('performed_at >= ?');
      params.push(filters.dateFrom);
    }
    
    if (filters.dateTo) {
      conditions.push('performed_at <= ?');
      params.push(filters.dateTo);
    }
    
    const where = conditions.length > 0 
      ? 'WHERE ' + conditions.join(' AND ')
      : '';
    
    const [rows] = await this.db.execute(
      `SELECT * FROM audit_logs ${where} 
       ORDER BY performed_at DESC 
       LIMIT 1000`,
      params
    );
    
    return rows;
  }
}

module.exports = AuditLogger;
```

---

## 8. Data Breach Procedures

### 8.1 Incident Response สำหรับ Data Breach

```javascript
// services/dataBreach.js

class DataBreachHandler {
  /**
   * ขั้นตอนเมื่อพบ Data Breach
   * 
   * ตาม PDPA:
   * - แจ้ง PDPC ภายใน 72 ชั่วโมง (ถ้าเป็น high risk)
   * - แจ้งเจ้าของข้อมูลโดยไม่ชักช้า
   */
  async handleBreach(breachDetails) {
    const {
      discoveredAt,
      type,           // 'unauthorized_access', 'data_leak', 'ransomware', etc.
      affectedData,   // ประเภทข้อมูลที่ถูก breach
      affectedUsers,  // จำนวน users ที่ได้รับผลกระทบ
      severity        // 'high', 'medium', 'low'
    } = breachDetails;
    
    // 1. บันทึก breach
    const breachId = await this.recordBreach(breachDetails);
    
    // 2. ประเมินความเสี่ยง
    const riskLevel = this.assessRisk(breachDetails);
    
    // 3. ดำเนินการตาม severity
    if (riskLevel === 'high') {
      // ต้องแจ้ง PDPC ภายใน 72 ชั่วโมง
      await this.notifyPDPC(breachId, breachDetails);
      
      // แจ้ง affected users
      await this.notifyAffectedUsers(breachId, breachDetails);
    }
    
    // 4. บันทึก action ที่ดำเนินการ
    await this.recordBreachActions(breachId, {
      containmentActions: [],
      notificationsSent: [],
      remediationSteps: []
    });
    
    return breachId;
  }
  
  assessRisk(breach) {
    const { affectedData, affectedUsers } = breach;
    
    // High risk factors
    const highRiskData = ['password', 'financial', 'health', 'id_card', 'credit_card'];
    const hasHighRiskData = affectedData.some(d => highRiskData.includes(d));
    
    if (hasHighRiskData || affectedUsers > 1000) return 'high';
    if (affectedUsers > 100) return 'medium';
    return 'low';
  }
  
  async notifyPDPC(breachId, breach) {
    // สร้างรายงานสำหรับส่งให้ PDPC
    const report = {
      notificationDate: new Date().toISOString(),
      breachDiscoveryDate: breach.discoveredAt,
      breachType: breach.type,
      affectedDataTypes: breach.affectedData,
      estimatedAffectedPersons: breach.affectedUsers,
      breachDescription: breach.description,
      consequencesDescription: breach.consequences,
      measuresToAddress: breach.measures,
      dpoContact: {
        name: process.env.DPO_NAME,
        email: process.env.DPO_EMAIL,
        phone: process.env.DPO_PHONE
      }
    };
    
    console.log('PDPC Notification Report:', JSON.stringify(report, null, 2));
    // ส่งรายงานไปยัง PDPC ตามช่องทางที่กำหนด
    // https://www.pdpc.or.th/
  }
  
  async notifyAffectedUsers(breachId, breach) {
    const affectedUserIds = await this.getAffectedUserIds(breach);
    
    for (const userId of affectedUserIds) {
      await lineClient.pushMessage(userId, {
        type: 'text',
        text: `
🔒 แจ้งเหตุการณ์ด้านความปลอดภัย

เรียน ผู้ใช้บริการที่เคารพ,

เราพบว่าข้อมูลบางส่วนของท่านอาจได้รับผลกระทบจากเหตุการณ์ด้านความปลอดภัยที่เกิดขึ้นเมื่อวันที่ ${breach.discoveredAt}

ข้อมูลที่อาจได้รับผลกระทบ: ${breach.affectedData.join(', ')}

มาตรการที่เราดำเนินการ:
${breach.measures}

คำแนะนำ:
• เปลี่ยนรหัสผ่านทันที
• ระวังอีเมลหรือ SMS ที่น่าสงสัย

หากมีข้อสงสัย ติดต่อ: ${process.env.DPO_EMAIL}

รหัสอ้างอิงเหตุการณ์: ${breachId}
        `.trim()
      });
    }
  }
}

module.exports = DataBreachHandler;
```

---

## 9. Cookie Consent สำหรับ LIFF

### 9.1 Cookie Banner Implementation

```javascript
// liff/cookieConsent/cookieBanner.js

class CookieConsentManager {
  constructor() {
    this.CONSENT_KEY = 'liff_cookie_consent';
    this.CONSENT_VERSION = '1.0';
    this.cookieCategories = {
      necessary: {
        label: 'จำเป็น',
        description: 'คุกกี้ที่จำเป็นสำหรับการทำงานของเว็บไซต์',
        required: true,
        defaultValue: true
      },
      analytics: {
        label: 'การวิเคราะห์',
        description: 'ช่วยให้เราเข้าใจว่าผู้ใช้ใช้งานเว็บไซต์อย่างไร',
        required: false,
        defaultValue: false
      },
      marketing: {
        label: 'การตลาด',
        description: 'ใช้สำหรับแสดงโฆษณาที่เกี่ยวข้องกับคุณ',
        required: false,
        defaultValue: false
      }
    };
  }
  
  /**
   * ตรวจสอบว่ามี consent แล้วหรือไม่
   */
  hasConsent() {
    try {
      const stored = localStorage.getItem(this.CONSENT_KEY);
      if (!stored) return false;
      
      const data = JSON.parse(stored);
      return data.version === this.CONSENT_VERSION;
    } catch {
      return false;
    }
  }
  
  /**
   * แสดง Cookie Banner
   */
  showBanner() {
    const banner = document.createElement('div');
    banner.id = 'cookie-banner';
    banner.innerHTML = `
      <div class="cookie-banner">
        <div class="cookie-content">
          <h3>🍪 เราใช้คุกกี้</h3>
          <p>เราใช้คุกกี้เพื่อให้บริการที่ดีที่สุดแก่คุณ กรุณาอ่าน
             <a href="/privacy-policy" target="_blank">นโยบายความเป็นส่วนตัว</a>
          </p>
          
          <div class="cookie-options" id="cookieOptions" style="display:none">
            ${this.renderCategoryOptions()}
          </div>
        </div>
        
        <div class="cookie-actions">
          <button onclick="cookieConsent.acceptAll()" class="btn-accept-all">
            ยอมรับทั้งหมด
          </button>
          <button onclick="cookieConsent.acceptNecessary()" class="btn-necessary">
            เฉพาะที่จำเป็น
          </button>
          <button onclick="cookieConsent.showOptions()" class="btn-customize">
            ปรับแต่ง
          </button>
        </div>
      </div>
    `;
    
    document.body.appendChild(banner);
  }
  
  renderCategoryOptions() {
    return Object.entries(this.cookieCategories).map(([key, cat]) => `
      <div class="cookie-category">
        <label class="toggle-label">
          <input type="checkbox" 
                 id="cookie_${key}" 
                 ${cat.defaultValue ? 'checked' : ''} 
                 ${cat.required ? 'disabled' : ''}>
          <span class="toggle-slider"></span>
          <div>
            <strong>${cat.label}</strong>
            <small>${cat.description}</small>
          </div>
        </label>
      </div>
    `).join('');
  }
  
  acceptAll() {
    const consents = Object.keys(this.cookieCategories).reduce((acc, key) => {
      acc[key] = true;
      return acc;
    }, {});
    
    this.saveConsent(consents);
    this.hideBanner();
    this.applyConsent(consents);
  }
  
  acceptNecessary() {
    const consents = Object.keys(this.cookieCategories).reduce((acc, key) => {
      acc[key] = this.cookieCategories[key].required;
      return acc;
    }, {});
    
    this.saveConsent(consents);
    this.hideBanner();
    this.applyConsent(consents);
  }
  
  saveConsent(consents) {
    try {
      localStorage.setItem(this.CONSENT_KEY, JSON.stringify({
        version: this.CONSENT_VERSION,
        consents,
        timestamp: new Date().toISOString()
      }));
    } catch (err) {
      console.error('Cannot save cookie consent:', err);
    }
  }
  
  applyConsent(consents) {
    // เปิดใช้งาน analytics ถ้า consent
    if (consents.analytics) {
      this.enableAnalytics();
    }
    
    // เปิดใช้งาน marketing ถ้า consent
    if (consents.marketing) {
      this.enableMarketing();
    }
  }
  
  enableAnalytics() {
    // เปิดใช้ Google Analytics, etc.
    console.log('Analytics enabled');
  }
  
  hideBanner() {
    const banner = document.getElementById('cookie-banner');
    if (banner) banner.remove();
  }
  
  showOptions() {
    const options = document.getElementById('cookieOptions');
    if (options) options.style.display = 'block';
  }
}

const cookieConsent = new CookieConsentManager();

// ตรวจสอบเมื่อ page โหลด
if (!cookieConsent.hasConsent()) {
  cookieConsent.showBanner();
}
```

---

## 10. Legal Templates

### 10.1 Privacy Notice สั้น ๆ สำหรับส่งใน Chat

```javascript
// templates/privacyNotice.js

const PRIVACY_NOTICES = {
  initial: `
🔒 การคุ้มครองข้อมูลส่วนบุคคล (PDPA)

เราเก็บ: ชื่อ LINE, รูปโปรไฟล์, ข้อความที่คุณส่ง
วัตถุประสงค์: เพื่อให้บริการแก่คุณ
สิทธิของคุณ:
• พิมพ์ "ดูข้อมูล" - ดูข้อมูลที่เราเก็บ
• พิมพ์ "ลบข้อมูล" - ลบข้อมูลทั้งหมด
• พิมพ์ "นโยบาย" - ดูนโยบายเพิ่มเติม

เราจะไม่ขายข้อมูลของคุณ ✅
  `.trim(),
  
  marketingConsent: `
📢 รับโปรโมชั่นพิเศษ

คุณต้องการรับข่าวสารและโปรโมชั่นพิเศษจากเราหรือไม่?
(คุณสามารถยกเลิกได้ทุกเมื่อ)
  `.trim(),
  
  dataRights: `
⚖️ สิทธิของคุณตาม PDPA

1. สิทธิเข้าถึง - พิมพ์ "ดูข้อมูล"
2. สิทธิแก้ไข - พิมพ์ "แก้ไขข้อมูล"  
3. สิทธิลบ - พิมพ์ "ลบข้อมูล"
4. สิทธิโอนย้าย - พิมพ์ "ส่งออกข้อมูล"
5. สิทธิคัดค้าน - พิมพ์ "หยุดโปรโมชั่น"

ติดต่อ DPO: dpo@your-company.com
  `.trim()
};

module.exports = PRIVACY_NOTICES;
```

---

## สรุปบทที่ 77

### PDPA Compliance Checklist

```
PDPA Checklist สำหรับ LINE OA:
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  ✅ มี Privacy Policy ที่ชัดเจนและครบถ้วน                    │
│  ✅ เก็บ Consent พร้อม timestamp และ version                  │
│  ✅ มีระบบถอน Consent ได้ง่าย                                 │
│  ✅ มีระบบลบข้อมูลตาม Right to Erasure                       │
│  ✅ มีระบบ Export ข้อมูลตาม Right to Portability             │
│  ✅ เก็บ Audit Log ทุกการกระทำที่เกี่ยวกับ Personal Data    │
│  ✅ มี Data Breach Response Plan                             │
│  ✅ Cookie Consent สำหรับ LIFF                               │
│  ✅ เข้ารหัสข้อมูลสำคัญใน database                          │
│  ✅ กำหนด Data Retention Policy                              │
│  ✅ มี DPO (Data Protection Officer) หรือผู้รับผิดชอบ        │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

**ต่อไป**: [Part 78 - Monitoring & Alerting](./part-078-monitoring-alerting.md)
