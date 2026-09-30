# Part 47: Database Design สำหรับ LINE Bot

## บทนำ

การออกแบบ database ที่ดีเป็นรากฐานสำคัญของ LINE bot ที่มีประสิทธิภาพ ในบทนี้เราจะเรียนรู้ patterns ต่างๆ สำหรับ database design ที่เหมาะสมกับ use case ของ LINE bot

---

## 1. หลักการออกแบบ Database สำหรับ LINE Bot

### 1.1 ข้อมูลที่ต้องเก็บใน LINE Bot

```
ข้อมูลหลักใน LINE Bot:
┌─────────────────────────────────────────────┐
│  Users (ผู้ใช้)                              │
│  ├── LINE User ID                            │
│  ├── Profile Information                     │
│  └── Preferences                             │
├─────────────────────────────────────────────┤
│  Conversations (การสนทนา)                    │
│  ├── Conversation State                      │
│  ├── Current Step/Flow                       │
│  └── Collected Data                         │
├─────────────────────────────────────────────┤
│  Messages (ข้อความ)                          │
│  ├── Message Logs                            │
│  ├── Sent Messages                           │
│  └── Received Messages                       │
├─────────────────────────────────────────────┤
│  Business Data (ข้อมูลธุรกิจ)               │
│  ├── Orders / Bookings                       │
│  ├── Products / Services                     │
│  └── Transactions                            │
└─────────────────────────────────────────────┘
```

### 1.2 ข้อควรระวัง

- **LINE User ID** มีรูปแบบ `U` ตามด้วย 32 ตัวอักษร เช่น `Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`
- User ID **เฉพาะ** กับแต่ละ LINE Official Account (ไม่ใช่ global ID)
- ควรเก็บ `displayName` และ `pictureUrl` เพราะ user สามารถเปลี่ยนได้
- **ไม่มี email** โดย default (ต้องขอผ่าน LINE Login)

---

## 2. MySQL Schema Design

### 2.1 Users Table

```sql
-- =============================================
-- USERS TABLE - เก็บข้อมูลผู้ใช้ LINE
-- =============================================
CREATE TABLE users (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    line_user_id    VARCHAR(50) NOT NULL UNIQUE,  -- LINE User ID
    display_name    VARCHAR(255),                   -- ชื่อที่แสดงใน LINE
    picture_url     TEXT,                           -- URL รูปโปรไฟล์
    status_message  TEXT,                           -- สถานะ
    email           VARCHAR(255),                   -- email (จาก LINE Login)
    language        VARCHAR(10) DEFAULT 'th',       -- ภาษาที่ใช้
    is_blocked      BOOLEAN DEFAULT FALSE,          -- ผู้ใช้ block bot หรือไม่
    is_active       BOOLEAN DEFAULT TRUE,           -- บัญชียังใช้งานอยู่หรือไม่
    last_interaction_at DATETIME,                   -- การโต้ตอบล่าสุด
    profile_updated_at  DATETIME,                   -- อัปเดต profile ล่าสุด
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    -- Indexes
    INDEX idx_line_user_id (line_user_id),
    INDEX idx_is_active (is_active),
    INDEX idx_last_interaction (last_interaction_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='ข้อมูลผู้ใช้ LINE';

-- ตัวอย่างข้อมูล
INSERT INTO users (line_user_id, display_name, picture_url, language) 
VALUES 
    ('Uabc123def456789012345678901234', 'สมชาย ใจดี', 'https://profile.line.com/pic1.jpg', 'th'),
    ('Uxyz987fed654321098765432109876', 'สมหญิง มีสุข', 'https://profile.line.com/pic2.jpg', 'th');
```

### 2.2 User Preferences Table

```sql
-- =============================================
-- USER PREFERENCES - การตั้งค่าของผู้ใช้
-- =============================================
CREATE TABLE user_preferences (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT UNSIGNED NOT NULL,
    preference_key  VARCHAR(100) NOT NULL,   -- ชื่อ preference
    preference_value TEXT,                    -- ค่า (JSON หรือ string)
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY uk_user_pref (user_id, preference_key),
    FOREIGN KEY fk_user_pref_user (user_id) REFERENCES users(id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ตัวอย่าง: เก็บการตั้งค่าการแจ้งเตือน
INSERT INTO user_preferences (user_id, preference_key, preference_value)
VALUES 
    (1, 'notification_enabled', 'true'),
    (1, 'notification_time', '{"morning": "09:00", "evening": "18:00"}'),
    (1, 'language', 'th'),
    (1, 'theme', 'default');
```

### 2.3 Conversation State Table

```sql
-- =============================================
-- CONVERSATION STATES - สถานะการสนทนา
-- =============================================
CREATE TABLE conversation_states (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id         BIGINT UNSIGNED NOT NULL,
    session_id      VARCHAR(100),            -- Session ID สำหรับแยก conversation
    current_flow    VARCHAR(100),            -- Flow ปัจจุบัน เช่น 'order_flow'
    current_step    VARCHAR(100),            -- Step ปัจจุบัน เช่น 'select_product'
    context_data    JSON,                    -- ข้อมูล context ที่เก็บระหว่าง flow
    is_active       BOOLEAN DEFAULT TRUE,    -- Session ยัง active อยู่หรือไม่
    expires_at      DATETIME,                -- หมดอายุเมื่อไหร่
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_user_active (user_id, is_active),
    INDEX idx_session (session_id),
    INDEX idx_expires (expires_at),
    FOREIGN KEY fk_conv_state_user (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='สถานะการสนทนาปัจจุบัน';

-- ตัวอย่างข้อมูล: user กำลังอยู่ใน order flow
INSERT INTO conversation_states 
    (user_id, current_flow, current_step, context_data, expires_at)
VALUES (
    1,
    'order_flow',
    'select_size',
    JSON_OBJECT(
        'product_id', 101,
        'product_name', 'เสื้อยืด',
        'color', 'ดำ',
        'collected_at', NOW()
    ),
    DATE_ADD(NOW(), INTERVAL 30 MINUTE)
);
```

### 2.4 Message Logs Table

```sql
-- =============================================
-- MESSAGE LOGS - บันทึกข้อความทั้งหมด
-- =============================================
CREATE TABLE message_logs (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    message_id      VARCHAR(50),             -- LINE Message ID
    user_id         BIGINT UNSIGNED,         -- NULL ถ้าเป็น system message
    direction       ENUM('incoming','outgoing') NOT NULL,
    message_type    VARCHAR(50) NOT NULL,    -- text, image, sticker, etc.
    content         TEXT,                    -- เนื้อหาข้อความ
    raw_payload     JSON,                    -- Raw payload จาก LINE
    reply_token     VARCHAR(50),             -- Reply token ถ้ามี
    is_replied      BOOLEAN DEFAULT FALSE,   -- ตอบกลับแล้วหรือยัง
    source_type     ENUM('user','group','room') DEFAULT 'user',
    source_id       VARCHAR(50),             -- group_id หรือ room_id ถ้ามี
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_created (user_id, created_at),
    INDEX idx_message_id (message_id),
    INDEX idx_direction (direction),
    INDEX idx_created (created_at),
    INDEX idx_reply_token (reply_token),
    FOREIGN KEY fk_msg_user (user_id) REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  COMMENT='บันทึกข้อความทั้งหมด'
  PARTITION BY RANGE (YEAR(created_at)) (
      PARTITION p2024 VALUES LESS THAN (2025),
      PARTITION p2025 VALUES LESS THAN (2026),
      PARTITION p2026 VALUES LESS THAN (2027),
      PARTITION p_future VALUES LESS THAN MAXVALUE
  );
```

### 2.5 Sessions Table

```sql
-- =============================================
-- SESSIONS - Session Management
-- =============================================
CREATE TABLE sessions (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    session_key     VARCHAR(100) NOT NULL UNIQUE,  -- unique session key
    user_id         BIGINT UNSIGNED NOT NULL,
    session_data    JSON,                           -- ข้อมูลใน session
    ip_address      VARCHAR(45),                    -- IP Address
    user_agent      TEXT,                           -- User Agent
    last_activity   DATETIME,                       -- Activity ล่าสุด
    expires_at      DATETIME NOT NULL,              -- หมดอายุเมื่อไหร่
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_session_key (session_key),
    INDEX idx_user_id (user_id),
    INDEX idx_expires (expires_at),
    FOREIGN KEY fk_session_user (user_id) REFERENCES users(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ทำ session cleanup โดยอัตโนมัติ
CREATE EVENT IF NOT EXISTS cleanup_expired_sessions
    ON SCHEDULE EVERY 1 HOUR
    DO
        DELETE FROM sessions WHERE expires_at < NOW();
```

---

## 3. E-Commerce Bot Schema

### 3.1 Products Table

```sql
-- =============================================
-- PRODUCTS - สินค้า
-- =============================================
CREATE TABLE products (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    sku             VARCHAR(100) UNIQUE,     -- SKU สินค้า
    name            VARCHAR(500) NOT NULL,   -- ชื่อสินค้า
    description     TEXT,                    -- รายละเอียด
    price           DECIMAL(10,2) NOT NULL,  -- ราคา
    sale_price      DECIMAL(10,2),           -- ราคาลด
    stock_quantity  INT DEFAULT 0,           -- จำนวนคงเหลือ
    images          JSON,                    -- รายการรูปภาพ
    category_id     INT UNSIGNED,
    attributes      JSON,                    -- คุณสมบัติสินค้า (สี, ขนาด ฯลฯ)
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_sku (sku),
    INDEX idx_category (category_id),
    INDEX idx_active_price (is_active, price),
    FULLTEXT INDEX ft_name_desc (name, description)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Categories
CREATE TABLE product_categories (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    parent_id   INT UNSIGNED,
    name        VARCHAR(255) NOT NULL,
    slug        VARCHAR(255) UNIQUE,
    sort_order  INT DEFAULT 0,
    is_active   BOOLEAN DEFAULT TRUE,
    FOREIGN KEY fk_cat_parent (parent_id) REFERENCES product_categories(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

### 3.2 Orders Table

```sql
-- =============================================
-- ORDERS - คำสั่งซื้อ
-- =============================================
CREATE TABLE orders (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_number    VARCHAR(50) NOT NULL UNIQUE,  -- เลขที่คำสั่งซื้อ
    user_id         BIGINT UNSIGNED NOT NULL,
    status          ENUM(
        'pending',      -- รอการยืนยัน
        'confirmed',    -- ยืนยันแล้ว
        'processing',   -- กำลังเตรียมสินค้า
        'shipped',      -- ส่งแล้ว
        'delivered',    -- ส่งถึงแล้ว
        'completed',    -- เสร็จสิ้น
        'cancelled',    -- ยกเลิก
        'refunded'      -- คืนเงินแล้ว
    ) NOT NULL DEFAULT 'pending',
    
    -- ราคา
    subtotal        DECIMAL(12,2) NOT NULL DEFAULT 0,  -- ราคาก่อนส่วนลด
    discount_amount DECIMAL(12,2) DEFAULT 0,           -- ส่วนลด
    shipping_fee    DECIMAL(10,2) DEFAULT 0,           -- ค่าส่ง
    tax_amount      DECIMAL(10,2) DEFAULT 0,           -- ภาษี
    total_amount    DECIMAL(12,2) NOT NULL,             -- ยอดรวม
    
    -- ที่อยู่จัดส่ง
    shipping_name       VARCHAR(255),
    shipping_phone      VARCHAR(20),
    shipping_address    TEXT,
    shipping_district   VARCHAR(100),
    shipping_province   VARCHAR(100),
    shipping_postal     VARCHAR(10),
    
    -- ข้อมูลการชำระเงิน
    payment_method  VARCHAR(50),            -- promptpay, bank_transfer, cod
    payment_status  ENUM('unpaid','paid','partial','refunded') DEFAULT 'unpaid',
    paid_at         DATETIME,
    payment_ref     VARCHAR(100),           -- reference จาก payment gateway
    
    -- ข้อมูลการส่ง
    tracking_number VARCHAR(100),
    shipping_provider VARCHAR(50),          -- Kerry, Flash, etc.
    shipped_at      DATETIME,
    delivered_at    DATETIME,
    
    notes           TEXT,                   -- หมายเหตุจากลูกค้า
    admin_notes     TEXT,                   -- หมายเหตุจาก admin
    
    -- Metadata
    source          VARCHAR(50) DEFAULT 'line_bot',  -- แหล่งที่มาของ order
    ip_address      VARCHAR(45),
    user_agent      TEXT,
    
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_order_number (order_number),
    INDEX idx_user_created (user_id, created_at),
    INDEX idx_status (status),
    INDEX idx_payment_status (payment_status),
    INDEX idx_created (created_at),
    FOREIGN KEY fk_order_user (user_id) REFERENCES users(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- ORDER ITEMS - รายการสินค้าในคำสั่งซื้อ
-- =============================================
CREATE TABLE order_items (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id        BIGINT UNSIGNED NOT NULL,
    product_id      BIGINT UNSIGNED NOT NULL,
    product_sku     VARCHAR(100),           -- เก็บ snapshot ณ เวลาที่สั่ง
    product_name    VARCHAR(500) NOT NULL,  -- เก็บ snapshot ชื่อสินค้า
    product_image   TEXT,                   -- รูปสินค้า
    variant_info    JSON,                   -- ข้อมูล variant (สี/ขนาด)
    quantity        INT UNSIGNED NOT NULL,
    unit_price      DECIMAL(10,2) NOT NULL, -- ราคาต่อชิ้น ณ เวลาสั่ง
    discount_amount DECIMAL(10,2) DEFAULT 0,
    total_price     DECIMAL(12,2) NOT NULL,
    
    FOREIGN KEY fk_item_order (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY fk_item_product (product_id) REFERENCES products(id),
    INDEX idx_order_id (order_id),
    INDEX idx_product_id (product_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- ORDER STATUS HISTORY - ประวัติการเปลี่ยนสถานะ
-- =============================================
CREATE TABLE order_status_history (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    order_id    BIGINT UNSIGNED NOT NULL,
    old_status  VARCHAR(50),
    new_status  VARCHAR(50) NOT NULL,
    note        TEXT,
    changed_by  VARCHAR(100),               -- user_id หรือ 'system'
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY fk_history_order (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    INDEX idx_order_id (order_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- Trigger เพื่อบันทึก status history อัตโนมัติ
DELIMITER //
CREATE TRIGGER tr_order_status_change
    AFTER UPDATE ON orders
    FOR EACH ROW
BEGIN
    IF OLD.status != NEW.status THEN
        INSERT INTO order_status_history (order_id, old_status, new_status, changed_by)
        VALUES (NEW.id, OLD.status, NEW.status, 'system');
    END IF;
END//
DELIMITER ;
```

### 3.3 Carts Table (Shopping Cart)

```sql
-- =============================================
-- CARTS - ตะกร้าสินค้า (Persistent Cart)
-- =============================================
CREATE TABLE cart_items (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT UNSIGNED NOT NULL,
    product_id  BIGINT UNSIGNED NOT NULL,
    variant_id  BIGINT UNSIGNED,            -- ถ้ามี variant
    variant_info JSON,                       -- ข้อมูล variant
    quantity    INT UNSIGNED NOT NULL DEFAULT 1,
    added_at    DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    UNIQUE KEY uk_user_product_variant (user_id, product_id, variant_id),
    FOREIGN KEY fk_cart_user (user_id) REFERENCES users(id) ON DELETE CASCADE,
    FOREIGN KEY fk_cart_product (product_id) REFERENCES products(id) ON DELETE CASCADE,
    INDEX idx_user_id (user_id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 4. Appointment/Booking Bot Schema

```sql
-- =============================================
-- SERVICES - บริการที่ให้จอง
-- =============================================
CREATE TABLE services (
    id              INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    description     TEXT,
    duration_minutes INT NOT NULL DEFAULT 60,   -- ระยะเวลาต่อครั้ง (นาที)
    price           DECIMAL(10,2),
    max_bookings    INT DEFAULT 1,               -- รับได้กี่คนต่อ slot
    buffer_minutes  INT DEFAULT 0,               -- เวลาพักระหว่าง appointment
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- STAFF - เจ้าหน้าที่/ผู้ให้บริการ
-- =============================================
CREATE TABLE staff (
    id              INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    email           VARCHAR(255),
    phone           VARCHAR(20),
    line_user_id    VARCHAR(50),             -- LINE ID ของเจ้าหน้าที่
    avatar_url      TEXT,
    bio             TEXT,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- STAFF AVAILABILITY - ตารางเวลาทำงาน
-- =============================================
CREATE TABLE staff_availability (
    id          INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    staff_id    INT UNSIGNED NOT NULL,
    day_of_week TINYINT NOT NULL,  -- 0=Sunday, 1=Monday, ..., 6=Saturday
    start_time  TIME NOT NULL,
    end_time    TIME NOT NULL,
    is_available BOOLEAN DEFAULT TRUE,
    
    FOREIGN KEY fk_avail_staff (staff_id) REFERENCES staff(id) ON DELETE CASCADE,
    INDEX idx_staff_day (staff_id, day_of_week)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- APPOINTMENTS - การนัดหมาย
-- =============================================
CREATE TABLE appointments (
    id              BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    booking_ref     VARCHAR(20) NOT NULL UNIQUE,  -- รหัสการจอง
    user_id         BIGINT UNSIGNED NOT NULL,
    service_id      INT UNSIGNED NOT NULL,
    staff_id        INT UNSIGNED,
    
    -- วันเวลา
    appointment_date    DATE NOT NULL,
    start_time          TIME NOT NULL,
    end_time            TIME NOT NULL,
    
    -- ข้อมูลลูกค้า
    customer_name   VARCHAR(255),
    customer_phone  VARCHAR(20),
    customer_email  VARCHAR(255),
    customer_notes  TEXT,
    
    -- สถานะ
    status          ENUM(
        'pending',      -- รอการยืนยัน
        'confirmed',    -- ยืนยันแล้ว
        'reminded',     -- ส่ง reminder แล้ว
        'completed',    -- เสร็จแล้ว
        'cancelled',    -- ยกเลิก
        'no_show'       -- ไม่มาตามนัด
    ) NOT NULL DEFAULT 'pending',
    
    cancel_reason   TEXT,
    
    -- การชำระเงิน
    deposit_amount  DECIMAL(10,2) DEFAULT 0,
    total_amount    DECIMAL(10,2),
    payment_status  ENUM('unpaid','deposit_paid','fully_paid','refunded') DEFAULT 'unpaid',
    
    -- Reminder
    reminder_sent_at DATETIME,
    
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_user_date (user_id, appointment_date),
    INDEX idx_staff_date (staff_id, appointment_date),
    INDEX idx_date_status (appointment_date, status),
    INDEX idx_booking_ref (booking_ref),
    FOREIGN KEY fk_appt_user (user_id) REFERENCES users(id),
    FOREIGN KEY fk_appt_service (service_id) REFERENCES services(id),
    FOREIGN KEY fk_appt_staff (staff_id) REFERENCES staff(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- Function: สร้าง booking reference อัตโนมัติ
-- =============================================
DELIMITER //
CREATE FUNCTION generate_booking_ref() RETURNS VARCHAR(20) DETERMINISTIC
BEGIN
    DECLARE ref VARCHAR(20);
    SET ref = CONCAT('BK', DATE_FORMAT(NOW(), '%Y%m%d'), LPAD(FLOOR(RAND() * 9999), 4, '0'));
    RETURN ref;
END//
DELIMITER ;
```

---

## 5. Analytics Tables

```sql
-- =============================================
-- USER EVENTS - บันทึก Events ของผู้ใช้
-- =============================================
CREATE TABLE user_events (
    id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    user_id     BIGINT UNSIGNED,
    event_type  VARCHAR(100) NOT NULL,   -- 'message_sent', 'button_clicked', etc.
    event_data  JSON,                    -- ข้อมูลเพิ่มเติม
    source      VARCHAR(50),             -- 'line_bot', 'liff', 'webhook'
    ip_address  VARCHAR(45),
    created_at  DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_user_event (user_id, event_type),
    INDEX idx_event_type (event_type),
    INDEX idx_created (created_at),
    FOREIGN KEY fk_event_user (user_id) REFERENCES users(id) ON DELETE SET NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
  PARTITION BY RANGE (TO_DAYS(created_at)) (
      PARTITION p202401 VALUES LESS THAN (TO_DAYS('2024-02-01')),
      PARTITION p202402 VALUES LESS THAN (TO_DAYS('2024-03-01')),
      PARTITION p_future VALUES LESS THAN MAXVALUE
  );

-- =============================================
-- DAILY STATS - สถิติรายวัน
-- =============================================
CREATE TABLE daily_stats (
    id                  BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    stat_date           DATE NOT NULL UNIQUE,
    new_users           INT DEFAULT 0,          -- ผู้ใช้ใหม่
    active_users        INT DEFAULT 0,          -- ผู้ใช้ที่ active
    total_messages      INT DEFAULT 0,          -- ข้อความทั้งหมด
    incoming_messages   INT DEFAULT 0,          -- ข้อความที่ได้รับ
    outgoing_messages   INT DEFAULT 0,          -- ข้อความที่ส่ง
    new_orders          INT DEFAULT 0,          -- คำสั่งซื้อใหม่
    revenue             DECIMAL(15,2) DEFAULT 0, -- รายได้
    new_appointments    INT DEFAULT 0,          -- การจองใหม่
    broadcast_sent      INT DEFAULT 0,          -- broadcast ที่ส่ง
    created_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at          DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_stat_date (stat_date)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- =============================================
-- MESSAGE TEMPLATES - template ข้อความ
-- =============================================
CREATE TABLE message_templates (
    id              INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,       -- ชื่อ template
    template_key    VARCHAR(100) NOT NULL UNIQUE, -- key สำหรับ reference
    message_type    VARCHAR(50) NOT NULL,         -- text, flex, image, etc.
    content         JSON NOT NULL,                -- เนื้อหา template
    variables       JSON,                         -- ตัวแปรที่ใช้ใน template
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at      DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_template_key (template_key),
    FULLTEXT INDEX ft_name (name)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

---

## 6. MongoDB Schema Design

### 6.1 Users Collection

```javascript
// MongoDB Schema สำหรับ Users
const userSchema = {
  _id: ObjectId(),
  lineUserId: "Uabc123def456",    // String, indexed, unique
  profile: {
    displayName: "สมชาย ใจดี",
    pictureUrl: "https://...",
    statusMessage: "Hello!",
    email: "user@example.com",    // จาก LINE Login
    language: "th"
  },
  settings: {
    notifications: {
      enabled: true,
      quietHours: {
        start: "22:00",
        end: "08:00"
      },
      channels: ["order_update", "promotion"]
    },
    language: "th",
    timezone: "Asia/Bangkok"
  },
  stats: {
    totalOrders: 5,
    totalSpent: 2500.00,
    points: 250,
    rank: "Bronze",
    firstOrderAt: ISODate("2024-01-15"),
    lastOrderAt: ISODate("2024-03-20")
  },
  tags: ["vip", "frequent_buyer"],  // Array ของ tags
  metadata: {
    source: "line_bot",
    version: "2.0",
    firstSeenAt: ISODate("2024-01-01"),
    lastSeenAt: ISODate("2024-03-25")
  },
  isBlocked: false,
  isActive: true,
  createdAt: ISODate("2024-01-01"),
  updatedAt: ISODate("2024-03-25")
}

// MongoDB Indexes
db.users.createIndex({ lineUserId: 1 }, { unique: true })
db.users.createIndex({ isActive: 1, "stats.lastOrderAt": -1 })
db.users.createIndex({ tags: 1 })
db.users.createIndex({ "profile.email": 1 }, { sparse: true })
db.users.createIndex({ createdAt: -1 })
```

### 6.2 Conversations Collection

```javascript
// MongoDB Schema สำหรับ Conversations
const conversationSchema = {
  _id: ObjectId(),
  lineUserId: "Uabc123",
  
  currentState: {
    flow: "order_flow",           // ชื่อ flow ที่กำลังทำ
    step: "select_size",          // step ปัจจุบัน
    data: {                       // ข้อมูลที่เก็บระหว่าง flow
      selectedProduct: {
        id: "P001",
        name: "เสื้อยืด",
        color: "ดำ"
      }
    },
    enteredAt: ISODate("2024-03-25T10:00:00Z"),
    expiresAt: ISODate("2024-03-25T10:30:00Z")
  },
  
  history: [                      // ประวัติ state transitions
    {
      flow: "menu",
      step: "main_menu",
      enteredAt: ISODate("2024-03-25T09:55:00Z"),
      exitedAt: ISODate("2024-03-25T10:00:00Z")
    }
  ],
  
  messages: [                     // ข้อความล่าสุดใน conversation
    {
      direction: "incoming",
      type: "text",
      content: "สั่งสินค้า",
      timestamp: ISODate("2024-03-25T10:00:00Z")
    },
    {
      direction: "outgoing",
      type: "flex",
      content: { /* Flex Message */ },
      timestamp: ISODate("2024-03-25T10:00:01Z")
    }
  ],
  
  updatedAt: ISODate("2024-03-25T10:00:00Z")
}

// Index สำหรับ TTL (Auto delete หลัง 24 ชั่วโมง)
db.conversations.createIndex(
  { "currentState.expiresAt": 1 }, 
  { expireAfterSeconds: 0 }
)
db.conversations.createIndex({ lineUserId: 1 }, { unique: true })
```

### 6.3 Messages Collection

```javascript
// MongoDB Schema สำหรับ Message Logs
const messageSchema = {
  _id: ObjectId(),
  messageId: "12345678901234",   // LINE Message ID
  lineUserId: "Uabc123",
  direction: "incoming",          // incoming | outgoing
  
  message: {
    type: "text",                 // text | image | sticker | flex | etc.
    content: "สวัสดี",
    rawPayload: { /* LINE raw payload */ }
  },
  
  context: {
    replyToken: "nHuyWiB7yP5Zw52FIkcQobQuGDXCTA...",
    sourceType: "user",          // user | group | room
    sourceId: "Uabc123",
    flow: "order_flow",
    step: "select_product"
  },
  
  processing: {
    handled: true,
    handlerName: "OrderHandler",
    processingTimeMs: 45,
    repliedAt: ISODate("2024-03-25T10:00:01Z")
  },
  
  createdAt: ISODate("2024-03-25T10:00:00Z")
}

// Indexes
db.messages.createIndex({ lineUserId: 1, createdAt: -1 })
db.messages.createIndex({ messageId: 1 }, { unique: true, sparse: true })
db.messages.createIndex({ createdAt: -1 })
db.messages.createIndex({ "context.flow": 1 })

// TTL Index - ลบ messages เก่ากว่า 90 วัน
db.messages.createIndex(
  { createdAt: 1 },
  { expireAfterSeconds: 60 * 60 * 24 * 90 }
)
```

### 6.4 Orders Collection

```javascript
// MongoDB Schema สำหรับ Orders
const orderSchema = {
  _id: ObjectId(),
  orderNumber: "ORD20240325001",
  lineUserId: "Uabc123",
  
  customer: {
    name: "สมชาย ใจดี",
    phone: "0812345678",
    email: "somchai@example.com"
  },
  
  items: [
    {
      productId: "P001",
      sku: "TSHIRT-BLK-M",
      name: "เสื้อยืดสีดำ Size M",
      imageUrl: "https://...",
      variant: { color: "ดำ", size: "M" },
      quantity: 2,
      unitPrice: 299,
      totalPrice: 598
    }
  ],
  
  pricing: {
    subtotal: 598,
    discountAmount: 50,
    shippingFee: 50,
    tax: 0,
    total: 598
  },
  
  shipping: {
    name: "สมชาย ใจดี",
    phone: "0812345678",
    address: "123 ถ.สุขุมวิท",
    district: "คลองเตย",
    province: "กรุงเทพมหานคร",
    postalCode: "10110"
  },
  
  payment: {
    method: "promptpay",
    status: "paid",
    amount: 598,
    paidAt: ISODate("2024-03-25T10:15:00Z"),
    reference: "REF202403251015001",
    slipUrl: "https://..."
  },
  
  delivery: {
    trackingNumber: "KER1234567890",
    provider: "Kerry",
    shippedAt: ISODate("2024-03-26T09:00:00Z"),
    estimatedDelivery: ISODate("2024-03-27"),
    deliveredAt: null
  },
  
  status: "shipped",
  
  statusHistory: [
    {
      status: "pending",
      changedAt: ISODate("2024-03-25T10:00:00Z"),
      changedBy: "system",
      note: "Order created"
    },
    {
      status: "confirmed",
      changedAt: ISODate("2024-03-25T10:16:00Z"),
      changedBy: "system",
      note: "Payment confirmed"
    },
    {
      status: "shipped",
      changedAt: ISODate("2024-03-26T09:00:00Z"),
      changedBy: "staff_001",
      note: "Shipped via Kerry"
    }
  ],
  
  notes: "ส่งด่วน",
  source: "line_bot",
  
  createdAt: ISODate("2024-03-25T10:00:00Z"),
  updatedAt: ISODate("2024-03-26T09:00:00Z")
}

// Indexes
db.orders.createIndex({ orderNumber: 1 }, { unique: true })
db.orders.createIndex({ lineUserId: 1, createdAt: -1 })
db.orders.createIndex({ status: 1 })
db.orders.createIndex({ "payment.status": 1 })
db.orders.createIndex({ createdAt: -1 })
```

---

## 7. PostgreSQL Schema

```sql
-- PostgreSQL version พร้อม JSONB และ advanced features

-- สร้าง Extension ที่ต้องการ
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- Full text search

-- =============================================
-- USERS TABLE
-- =============================================
CREATE TABLE users (
    id              BIGSERIAL PRIMARY KEY,
    line_user_id    VARCHAR(50) NOT NULL UNIQUE,
    profile         JSONB NOT NULL DEFAULT '{}',
    settings        JSONB NOT NULL DEFAULT '{}',
    tags            TEXT[] DEFAULT '{}',
    is_blocked      BOOLEAN DEFAULT FALSE,
    is_active       BOOLEAN DEFAULT TRUE,
    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- PostgreSQL Indexes รวมถึง JSONB
CREATE INDEX idx_users_line_id ON users(line_user_id);
CREATE INDEX idx_users_active ON users(is_active);
CREATE INDEX idx_users_tags ON users USING GIN(tags);
CREATE INDEX idx_users_profile ON users USING GIN(profile);
CREATE INDEX idx_users_settings ON users USING GIN(settings);

-- Function trigger สำหรับ updated_at อัตโนมัติ
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER tr_users_updated_at
    BEFORE UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- =============================================
-- CONVERSATION STATES TABLE
-- =============================================
CREATE TABLE conversation_states (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    flow        VARCHAR(100),
    step        VARCHAR(100),
    context     JSONB DEFAULT '{}',
    is_active   BOOLEAN DEFAULT TRUE,
    expires_at  TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_conv_state_user ON conversation_states(user_id, is_active);
CREATE INDEX idx_conv_state_expires ON conversation_states(expires_at) 
    WHERE is_active = TRUE;

-- =============================================
-- ORDERS TABLE
-- =============================================
CREATE TYPE order_status AS ENUM (
    'pending', 'confirmed', 'processing', 
    'shipped', 'delivered', 'completed', 
    'cancelled', 'refunded'
);

CREATE TYPE payment_status AS ENUM (
    'unpaid', 'paid', 'partial', 'refunded'
);

CREATE TABLE orders (
    id              BIGSERIAL PRIMARY KEY,
    order_number    VARCHAR(50) NOT NULL UNIQUE,
    user_id         BIGINT NOT NULL REFERENCES users(id),
    status          order_status NOT NULL DEFAULT 'pending',
    items           JSONB NOT NULL DEFAULT '[]',
    pricing         JSONB NOT NULL DEFAULT '{}',
    shipping        JSONB DEFAULT '{}',
    payment         JSONB DEFAULT '{}',
    delivery        JSONB DEFAULT '{}',
    notes           TEXT,
    metadata        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_user ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_number ON orders(order_number);
CREATE INDEX idx_orders_items ON orders USING GIN(items);

-- View สำหรับ order summary
CREATE VIEW order_summaries AS
SELECT 
    o.id,
    o.order_number,
    o.user_id,
    u.line_user_id,
    u.profile->>'displayName' AS customer_name,
    o.status,
    (o.pricing->>'total')::DECIMAL AS total_amount,
    (o.pricing->>'itemCount')::INT AS item_count,
    o.created_at
FROM orders o
JOIN users u ON o.user_id = u.id;
```

---

## 8. Indexing Strategy

### 8.1 MySQL Indexing

```sql
-- =============================================
-- INDEXING STRATEGY สำหรับ LINE Bot
-- =============================================

-- 1. Lookup by LINE User ID (Query บ่อยที่สุด)
CREATE INDEX idx_lookup_line_id ON users(line_user_id);

-- 2. Active users ที่ต้อง broadcast
CREATE INDEX idx_active_users ON users(is_active, is_blocked, last_interaction_at);

-- 3. Orders by user (ดู order history)
CREATE INDEX idx_user_orders ON orders(user_id, created_at DESC);

-- 4. Pending orders (admin ต้องจัดการ)
CREATE INDEX idx_pending_orders ON orders(status, created_at) 
    WHERE status IN ('pending', 'confirmed', 'processing');
-- MySQL ใช้: CREATE INDEX idx_pending ON orders(status) WHERE status != 'completed';

-- 5. Message lookup
CREATE INDEX idx_msg_reply_token ON message_logs(reply_token) WHERE reply_token IS NOT NULL;

-- 6. Conversation state lookup
CREATE INDEX idx_conv_active ON conversation_states(user_id) WHERE is_active = TRUE;

-- 7. Appointment date lookup
CREATE INDEX idx_appt_date ON appointments(appointment_date, start_time, status);

-- ตรวจสอบ index usage
EXPLAIN SELECT * FROM orders WHERE user_id = 1 ORDER BY created_at DESC LIMIT 10;
SHOW INDEX FROM orders;

-- ดู slow queries
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 1;
SHOW VARIABLES LIKE 'slow_query_log_file';
```

### 8.2 Query Optimization Examples

```sql
-- =============================================
-- OPTIMIZED QUERIES
-- =============================================

-- 1. ดึงข้อมูล user พร้อม active conversation
SELECT 
    u.id,
    u.line_user_id,
    u.display_name,
    cs.current_flow,
    cs.current_step,
    cs.context_data
FROM users u
LEFT JOIN conversation_states cs ON u.id = cs.user_id AND cs.is_active = TRUE
WHERE u.line_user_id = 'Uabc123'
LIMIT 1;

-- 2. Dashboard: ออเดอร์วันนี้
SELECT 
    COUNT(*) as total_orders,
    SUM(total_amount) as total_revenue,
    SUM(CASE WHEN status = 'pending' THEN 1 ELSE 0 END) as pending_count
FROM orders
WHERE DATE(created_at) = CURDATE();

-- 3. User activity summary (เร็วกว่าการ join หลายตาราง)
SELECT 
    u.line_user_id,
    u.display_name,
    COUNT(DISTINCT o.id) as total_orders,
    COALESCE(SUM(o.total_amount), 0) as total_spent,
    MAX(o.created_at) as last_order_date
FROM users u
LEFT JOIN orders o ON u.id = o.user_id AND o.status NOT IN ('cancelled', 'refunded')
GROUP BY u.id, u.line_user_id, u.display_name
ORDER BY total_spent DESC
LIMIT 100;

-- 4. ดึง upcoming appointments (ใช้ index)
SELECT 
    a.*,
    u.display_name,
    u.line_user_id,
    s.name as service_name,
    st.name as staff_name
FROM appointments a
JOIN users u ON a.user_id = u.id
JOIN services s ON a.service_id = s.id
LEFT JOIN staff st ON a.staff_id = st.id
WHERE a.appointment_date = CURDATE()
    AND a.status = 'confirmed'
ORDER BY a.start_time;
```

---

## 9. Data Retention Policy

```sql
-- =============================================
-- DATA RETENTION - นโยบายเก็บข้อมูล
-- =============================================

-- Policy:
-- - Message logs: เก็บ 90 วัน
-- - Conversation states: เก็บ 24 ชั่วโมงหลัง expire
-- - Sessions: เก็บ 30 วัน
-- - User events: เก็บ 1 ปี
-- - Orders: เก็บถาวร (ตามกฎหมาย)
-- - Analytics: เก็บ 2 ปี

-- Stored Procedure สำหรับ cleanup
DELIMITER //
CREATE PROCEDURE sp_cleanup_old_data()
BEGIN
    DECLARE rows_affected INT;
    
    -- ลบ message logs เก่ากว่า 90 วัน
    DELETE FROM message_logs 
    WHERE created_at < DATE_SUB(NOW(), INTERVAL 90 DAY);
    SET rows_affected = ROW_COUNT();
    SELECT CONCAT('Deleted message_logs: ', rows_affected) as log;
    
    -- ลบ expired conversation states
    UPDATE conversation_states 
    SET is_active = FALSE 
    WHERE expires_at < NOW() AND is_active = TRUE;
    
    DELETE FROM conversation_states
    WHERE is_active = FALSE 
    AND updated_at < DATE_SUB(NOW(), INTERVAL 24 HOUR);
    
    -- ลบ expired sessions
    DELETE FROM sessions 
    WHERE expires_at < NOW();
    
    -- ลบ user events เก่ากว่า 1 ปี
    DELETE FROM user_events 
    WHERE created_at < DATE_SUB(NOW(), INTERVAL 1 YEAR);
    
    SELECT 'Cleanup completed' as status;
END//
DELIMITER ;

-- สร้าง Event สำหรับ cleanup อัตโนมัติทุกวัน เวลา 02:00 น.
CREATE EVENT IF NOT EXISTS ev_daily_cleanup
    ON SCHEDULE EVERY 1 DAY
    STARTS '2024-01-01 02:00:00'
    DO CALL sp_cleanup_old_data();
```

---

## 10. Node.js Database Helper

```javascript
// database/mysql.js
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
  waitForConnections: true,
  connectionLimit: 20,
  queueLimit: 0,
  charset: 'utf8mb4',
  timezone: '+07:00',
  // สำคัญสำหรับ JSON columns
  typeCast: function(field, next) {
    if (field.type === 'JSON') {
      const value = field.string();
      return value ? JSON.parse(value) : null;
    }
    return next();
  }
});

// Helper: Find or create user
async function findOrCreateUser(lineUserId, profile = {}) {
  const conn = await pool.getConnection();
  try {
    await conn.beginTransaction();

    const [rows] = await conn.execute(
      'SELECT * FROM users WHERE line_user_id = ? FOR UPDATE',
      [lineUserId]
    );

    if (rows.length > 0) {
      // อัปเดต profile ถ้ามีการเปลี่ยนแปลง
      if (profile.displayName && rows[0].display_name !== profile.displayName) {
        await conn.execute(
          `UPDATE users 
           SET display_name = ?, picture_url = ?, 
               last_interaction_at = NOW(), profile_updated_at = NOW()
           WHERE id = ?`,
          [profile.displayName, profile.pictureUrl, rows[0].id]
        );
      } else {
        await conn.execute(
          'UPDATE users SET last_interaction_at = NOW() WHERE id = ?',
          [rows[0].id]
        );
      }
      await conn.commit();
      return rows[0];
    }

    // สร้าง user ใหม่
    const [result] = await conn.execute(
      `INSERT INTO users 
       (line_user_id, display_name, picture_url, last_interaction_at) 
       VALUES (?, ?, ?, NOW())`,
      [lineUserId, profile.displayName, profile.pictureUrl]
    );

    const [newUser] = await conn.execute(
      'SELECT * FROM users WHERE id = ?',
      [result.insertId]
    );

    await conn.commit();
    return newUser[0];
  } catch (error) {
    await conn.rollback();
    throw error;
  } finally {
    conn.release();
  }
}

// Helper: Get/Update conversation state
async function getConversationState(userId) {
  const [rows] = await pool.execute(
    `SELECT * FROM conversation_states 
     WHERE user_id = ? AND is_active = TRUE 
     AND (expires_at IS NULL OR expires_at > NOW())
     ORDER BY updated_at DESC LIMIT 1`,
    [userId]
  );
  return rows[0] || null;
}

async function setConversationState(userId, flow, step, contextData) {
  const expiresAt = new Date(Date.now() + 30 * 60 * 1000); // 30 นาที

  const [existing] = await pool.execute(
    'SELECT id FROM conversation_states WHERE user_id = ? AND is_active = TRUE',
    [userId]
  );

  if (existing.length > 0) {
    await pool.execute(
      `UPDATE conversation_states 
       SET current_flow = ?, current_step = ?, 
           context_data = ?, expires_at = ?
       WHERE id = ?`,
      [flow, step, JSON.stringify(contextData), expiresAt, existing[0].id]
    );
  } else {
    await pool.execute(
      `INSERT INTO conversation_states 
       (user_id, current_flow, current_step, context_data, expires_at)
       VALUES (?, ?, ?, ?, ?)`,
      [userId, flow, step, JSON.stringify(contextData), expiresAt]
    );
  }
}

async function clearConversationState(userId) {
  await pool.execute(
    'UPDATE conversation_states SET is_active = FALSE WHERE user_id = ?',
    [userId]
  );
}

module.exports = {
  pool,
  findOrCreateUser,
  getConversationState,
  setConversationState,
  clearConversationState,
};
```

---

## สรุป

การออกแบบ database สำหรับ LINE Bot ที่ดีต้องคำนึงถึง:

1. **Users**: เก็บ LINE User ID เป็น unique key, เก็บ profile snapshot
2. **Conversation State**: ต้องมี expiration, รองรับ concurrent access
3. **Message Logs**: partitioning ตามวันที่, TTL policy
4. **Business Data**: ออกแบบตาม use case (E-commerce, Booking)
5. **Analytics**: แยก table สำหรับ reporting
6. **Indexing**: index ที่เหมาะสมกับ query patterns
7. **Data Retention**: นโยบายเก็บและลบข้อมูล

ทั้ง MySQL, MongoDB และ PostgreSQL ต่างมีจุดแข็งต่างกัน การเลือกใช้ขึ้นอยู่กับ requirements และ team expertise
