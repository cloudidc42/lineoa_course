# Part 66: E-Commerce Bot บน LINE OA

## บทนำ

ในบทนี้เราจะสร้างระบบ E-Commerce เต็มรูปแบบบน LINE Official Account ครอบคลุมตั้งแต่การจัดการสินค้า ตะกร้าสินค้า การชำระเงินผ่าน LINE Pay ไปจนถึงระบบสมาชิกและคะแนนสะสม

## สิ่งที่จะได้เรียนรู้

- การออกแบบระบบ E-Commerce บน LINE
- Product Catalog พร้อม Flex Message
- Shopping Cart ด้วย Redis/MySQL
- Order Management System
- LINE Pay Integration
- ระบบ Loyalty Points
- การจัดการสต็อกสินค้า
- Admin Panel ด้วย Express.js
- Return/Refund Flow

---

## 1. สถาปัตยกรรมระบบ

```
┌─────────────────────────────────────────────────────┐
│                   LINE OA Bot                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ Product  │  │  Cart    │  │   Order          │   │
│  │ Catalog  │  │ Manager  │  │   Management     │   │
│  └──────────┘  └──────────┘  └──────────────────┘   │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐   │
│  │ LINE Pay │  │ Loyalty  │  │   Inventory      │   │
│  │ Payment  │  │ Points   │  │   Management     │   │
│  └──────────┘  └──────────┘  └──────────────────┘   │
└─────────────────────────────────────────────────────┘
         │              │              │
    ┌────▼────┐    ┌────▼────┐   ┌────▼────┐
    │  MySQL  │    │  Redis  │   │  LINE   │
    │   DB    │    │  Cache  │   │  Pay API│
    └─────────┘    └─────────┘   └─────────┘
```

## 2. Database Schema

```sql
-- ไฟล์: database/schema.sql

CREATE DATABASE IF NOT EXISTS lineoa_ecommerce CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE lineoa_ecommerce;

-- ตารางผู้ใช้
CREATE TABLE users (
  id INT PRIMARY KEY AUTO_INCREMENT,
  line_user_id VARCHAR(100) UNIQUE NOT NULL,
  display_name VARCHAR(200),
  picture_url TEXT,
  phone VARCHAR(20),
  email VARCHAR(200),
  address TEXT,
  loyalty_points INT DEFAULT 0,
  total_spent DECIMAL(12,2) DEFAULT 0,
  tier ENUM('bronze','silver','gold','platinum') DEFAULT 'bronze',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_line_user_id (line_user_id)
);

-- ตารางหมวดหมู่สินค้า
CREATE TABLE categories (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  description TEXT,
  image_url TEXT,
  parent_id INT,
  sort_order INT DEFAULT 0,
  is_active TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (parent_id) REFERENCES categories(id) ON DELETE SET NULL,
  INDEX idx_parent_id (parent_id),
  INDEX idx_is_active (is_active)
);

-- ตารางสินค้า
CREATE TABLE products (
  id INT PRIMARY KEY AUTO_INCREMENT,
  category_id INT,
  sku VARCHAR(100) UNIQUE NOT NULL,
  name VARCHAR(300) NOT NULL,
  description TEXT,
  price DECIMAL(10,2) NOT NULL,
  sale_price DECIMAL(10,2),
  cost_price DECIMAL(10,2),
  stock_quantity INT DEFAULT 0,
  low_stock_threshold INT DEFAULT 10,
  weight DECIMAL(8,2),
  images JSON,
  attributes JSON,
  is_active TINYINT(1) DEFAULT 1,
  is_featured TINYINT(1) DEFAULT 0,
  sold_count INT DEFAULT 0,
  rating DECIMAL(3,2) DEFAULT 0,
  review_count INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (category_id) REFERENCES categories(id) ON DELETE SET NULL,
  INDEX idx_category_id (category_id),
  INDEX idx_is_active (is_active),
  INDEX idx_is_featured (is_featured),
  FULLTEXT INDEX ft_name_description (name, description)
);

-- ตารางตัวเลือกสินค้า (variants)
CREATE TABLE product_variants (
  id INT PRIMARY KEY AUTO_INCREMENT,
  product_id INT NOT NULL,
  sku VARCHAR(100) UNIQUE NOT NULL,
  name VARCHAR(200) NOT NULL,
  price_modifier DECIMAL(10,2) DEFAULT 0,
  stock_quantity INT DEFAULT 0,
  attributes JSON,
  is_active TINYINT(1) DEFAULT 1,
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
  INDEX idx_product_id (product_id)
);

-- ตะกร้าสินค้า
CREATE TABLE carts (
  id INT PRIMARY KEY AUTO_INCREMENT,
  user_id INT NOT NULL,
  product_id INT NOT NULL,
  variant_id INT,
  quantity INT NOT NULL DEFAULT 1,
  price_at_time DECIMAL(10,2),
  notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
  FOREIGN KEY (product_id) REFERENCES products(id) ON DELETE CASCADE,
  FOREIGN KEY (variant_id) REFERENCES product_variants(id) ON DELETE SET NULL,
  UNIQUE KEY unique_cart_item (user_id, product_id, variant_id)
);

-- ตารางคูปอง/โปรโมชั่น
CREATE TABLE coupons (
  id INT PRIMARY KEY AUTO_INCREMENT,
  code VARCHAR(50) UNIQUE NOT NULL,
  name VARCHAR(200) NOT NULL,
  type ENUM('percentage','fixed','free_shipping','buy_x_get_y') NOT NULL,
  value DECIMAL(10,2),
  min_order_amount DECIMAL(10,2) DEFAULT 0,
  max_discount DECIMAL(10,2),
  usage_limit INT,
  used_count INT DEFAULT 0,
  per_user_limit INT DEFAULT 1,
  start_date TIMESTAMP,
  end_date TIMESTAMP,
  conditions JSON,
  is_active TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_code (code),
  INDEX idx_is_active (is_active)
);

-- การใช้คูปอง
CREATE TABLE coupon_usages (
  id INT PRIMARY KEY AUTO_INCREMENT,
  coupon_id INT NOT NULL,
  user_id INT NOT NULL,
  order_id INT,
  used_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (coupon_id) REFERENCES coupons(id),
  FOREIGN KEY (user_id) REFERENCES users(id)
);

-- ตารางออเดอร์
CREATE TABLE orders (
  id INT PRIMARY KEY AUTO_INCREMENT,
  order_number VARCHAR(50) UNIQUE NOT NULL,
  user_id INT NOT NULL,
  status ENUM('pending','confirmed','processing','shipped','delivered','cancelled','refunded') DEFAULT 'pending',
  subtotal DECIMAL(12,2) NOT NULL,
  discount_amount DECIMAL(12,2) DEFAULT 0,
  shipping_fee DECIMAL(10,2) DEFAULT 0,
  total_amount DECIMAL(12,2) NOT NULL,
  coupon_id INT,
  shipping_address JSON,
  payment_method VARCHAR(50),
  payment_status ENUM('pending','paid','failed','refunded') DEFAULT 'pending',
  payment_transaction_id VARCHAR(200),
  payment_details JSON,
  notes TEXT,
  tracking_number VARCHAR(200),
  shipped_at TIMESTAMP NULL,
  delivered_at TIMESTAMP NULL,
  cancelled_at TIMESTAMP NULL,
  cancel_reason TEXT,
  loyalty_points_earned INT DEFAULT 0,
  loyalty_points_used INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id),
  FOREIGN KEY (coupon_id) REFERENCES coupons(id) ON DELETE SET NULL,
  INDEX idx_user_id (user_id),
  INDEX idx_status (status),
  INDEX idx_order_number (order_number)
);

-- รายการสินค้าในออเดอร์
CREATE TABLE order_items (
  id INT PRIMARY KEY AUTO_INCREMENT,
  order_id INT NOT NULL,
  product_id INT NOT NULL,
  variant_id INT,
  product_name VARCHAR(300) NOT NULL,
  variant_name VARCHAR(200),
  sku VARCHAR(100),
  quantity INT NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL,
  total_price DECIMAL(12,2) NOT NULL,
  product_snapshot JSON,
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
  FOREIGN KEY (product_id) REFERENCES products(id),
  INDEX idx_order_id (order_id)
);

-- ประวัติสถานะออเดอร์
CREATE TABLE order_status_history (
  id INT PRIMARY KEY AUTO_INCREMENT,
  order_id INT NOT NULL,
  status VARCHAR(50) NOT NULL,
  comment TEXT,
  created_by VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE
);

-- ระบบ Loyalty Points
CREATE TABLE loyalty_transactions (
  id INT PRIMARY KEY AUTO_INCREMENT,
  user_id INT NOT NULL,
  type ENUM('earn','redeem','expire','adjust') NOT NULL,
  points INT NOT NULL,
  balance_after INT NOT NULL,
  reference_type VARCHAR(50),
  reference_id INT,
  description TEXT,
  expires_at TIMESTAMP NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (user_id) REFERENCES users(id),
  INDEX idx_user_id (user_id),
  INDEX idx_type (type)
);

-- รีวิวสินค้า
CREATE TABLE product_reviews (
  id INT PRIMARY KEY AUTO_INCREMENT,
  product_id INT NOT NULL,
  user_id INT NOT NULL,
  order_id INT,
  rating INT NOT NULL CHECK (rating BETWEEN 1 AND 5),
  comment TEXT,
  images JSON,
  is_verified TINYINT(1) DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (product_id) REFERENCES products(id),
  FOREIGN KEY (user_id) REFERENCES users(id),
  UNIQUE KEY unique_review (product_id, user_id, order_id),
  INDEX idx_product_id (product_id)
);

-- การคืนสินค้า/คืนเงิน
CREATE TABLE returns (
  id INT PRIMARY KEY AUTO_INCREMENT,
  order_id INT NOT NULL,
  user_id INT NOT NULL,
  status ENUM('requested','approved','rejected','received','refunded') DEFAULT 'requested',
  reason VARCHAR(200) NOT NULL,
  description TEXT,
  images JSON,
  refund_amount DECIMAL(12,2),
  refund_method VARCHAR(50),
  refund_transaction_id VARCHAR(200),
  admin_notes TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (order_id) REFERENCES orders(id),
  FOREIGN KEY (user_id) REFERENCES users(id)
);

-- Stock Movement
CREATE TABLE stock_movements (
  id INT PRIMARY KEY AUTO_INCREMENT,
  product_id INT NOT NULL,
  variant_id INT,
  type ENUM('in','out','adjust','reserve','release') NOT NULL,
  quantity INT NOT NULL,
  balance_after INT NOT NULL,
  reference_type VARCHAR(50),
  reference_id INT,
  notes TEXT,
  created_by VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (product_id) REFERENCES products(id),
  INDEX idx_product_id (product_id)
);

-- ข้อมูลตัวอย่าง
INSERT INTO categories (name, description) VALUES 
('อาหารและเครื่องดื่ม', 'สินค้าอาหารและเครื่องดื่มทุกประเภท'),
('เสื้อผ้าและแฟชั่น', 'เสื้อผ้า รองเท้า และเครื่องประดับ'),
('อิเล็กทรอนิกส์', 'สินค้าไอทีและอิเล็กทรอนิกส์'),
('สุขภาพและความงาม', 'ผลิตภัณฑ์ดูแลสุขภาพและความงาม');

INSERT INTO products (category_id, sku, name, description, price, stock_quantity, images) VALUES
(1, 'FOOD001', 'ข้าวกล่องไก่ทอด', 'ข้าวกล่องไก่ทอดกรอบ พร้อมเครื่องเคียง', 89.00, 100, '["https://example.com/food001.jpg"]'),
(2, 'CLOTH001', 'เสื้อยืด Premium Cotton', 'เสื้อยืดผ้าคอตตอน 100% ระบายอากาศดี', 290.00, 500, '["https://example.com/cloth001.jpg"]'),
(3, 'ELEC001', 'หูฟัง Bluetooth Pro', 'หูฟังบลูทูธ 5.0 ตัดเสียงรบกวน', 1290.00, 50, '["https://example.com/elec001.jpg"]'),
(4, 'BEAUTY001', 'ครีมบำรุงหน้า SPF50', 'ครีมกันแดดและบำรุงผิวในขวดเดียว', 450.00, 200, '["https://example.com/beauty001.jpg"]');
```

## 3. โครงสร้างโปรเจค

```
lineoa-ecommerce/
├── src/
│   ├── config/
│   │   ├── database.js
│   │   ├── redis.js
│   │   └── linepay.js
│   ├── handlers/
│   │   ├── messageHandler.js
│   │   ├── postbackHandler.js
│   │   └── followHandler.js
│   ├── services/
│   │   ├── productService.js
│   │   ├── cartService.js
│   │   ├── orderService.js
│   │   ├── paymentService.js
│   │   ├── loyaltyService.js
│   │   └── inventoryService.js
│   ├── messages/
│   │   ├── productMessages.js
│   │   ├── cartMessages.js
│   │   ├── orderMessages.js
│   │   └── menuMessages.js
│   ├── routes/
│   │   ├── webhook.js
│   │   ├── linepay.js
│   │   └── admin.js
│   └── utils/
│       ├── orderNumber.js
│       └── validation.js
├── admin/
│   ├── public/
│   └── views/
├── database/
│   └── schema.sql
├── package.json
└── index.js
```

## 4. การตั้งค่าหลัก

```javascript
// src/config/database.js
const mysql = require('mysql2/promise');

const pool = mysql.createPool({
  host: process.env.DB_HOST || 'localhost',
  user: process.env.DB_USER || 'root',
  password: process.env.DB_PASSWORD || '',
  database: process.env.DB_NAME || 'lineoa_ecommerce',
  waitForConnections: true,
  connectionLimit: 20,
  queueLimit: 0,
  charset: 'utf8mb4',
  timezone: '+07:00'
});

pool.on('connection', (connection) => {
  console.log('New DB connection established');
});

module.exports = pool;
```

```javascript
// src/config/redis.js
const Redis = require('ioredis');

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: process.env.REDIS_PORT || 6379,
  password: process.env.REDIS_PASSWORD || '',
  db: 0,
  keyPrefix: 'lineoa:ecom:',
  retryStrategy: (times) => Math.min(times * 50, 2000)
});

redis.on('error', (err) => console.error('Redis error:', err));
redis.on('connect', () => console.log('Redis connected'));

module.exports = redis;
```

```javascript
// src/config/linepay.js
const axios = require('axios');
const crypto = require('crypto');

const LINE_PAY_CONFIG = {
  channelId: process.env.LINEPAY_CHANNEL_ID,
  channelSecretKey: process.env.LINEPAY_CHANNEL_SECRET,
  isSandbox: process.env.NODE_ENV !== 'production',
  baseUrl: process.env.NODE_ENV === 'production'
    ? 'https://api-pay.line.me'
    : 'https://sandbox-api-pay.line.me'
};

function generateSignature(channelSecret, uri, body, nonce) {
  const text = channelSecret + uri + JSON.stringify(body) + nonce;
  return crypto.createHmac('sha256', channelSecret).update(text).digest('base64');
}

async function linePayRequest(method, uri, body = {}) {
  const nonce = crypto.randomBytes(16).toString('hex');
  const signature = generateSignature(
    LINE_PAY_CONFIG.channelSecretKey,
    uri,
    body,
    nonce
  );

  const headers = {
    'Content-Type': 'application/json',
    'X-LINE-ChannelId': LINE_PAY_CONFIG.channelId,
    'X-LINE-Authorization-Nonce': nonce,
    'X-LINE-Authorization': signature
  };

  try {
    const response = await axios({
      method,
      url: `${LINE_PAY_CONFIG.baseUrl}${uri}`,
      headers,
      data: body
    });
    return response.data;
  } catch (error) {
    console.error('LINE Pay API error:', error.response?.data || error.message);
    throw error;
  }
}

module.exports = { linePayRequest, LINE_PAY_CONFIG };
```

## 5. Product Service

```javascript
// src/services/productService.js
const db = require('../config/database');
const redis = require('../config/redis');

class ProductService {
  // ดึงสินค้าแนะนำ
  async getFeaturedProducts(limit = 6) {
    const cacheKey = `featured:${limit}`;
    const cached = await redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    const [products] = await db.query(
      `SELECT p.*, c.name as category_name
       FROM products p
       LEFT JOIN categories c ON p.category_id = c.id
       WHERE p.is_active = 1 AND p.is_featured = 1
       ORDER BY p.sold_count DESC
       LIMIT ?`,
      [limit]
    );

    await redis.setex(cacheKey, 300, JSON.stringify(products));
    return products;
  }

  // ค้นหาสินค้า
  async searchProducts(keyword, page = 1, limit = 10) {
    const offset = (page - 1) * limit;
    const searchTerm = `%${keyword}%`;

    const [products] = await db.query(
      `SELECT p.*, c.name as category_name
       FROM products p
       LEFT JOIN categories c ON p.category_id = c.id
       WHERE p.is_active = 1
         AND (p.name LIKE ? OR p.description LIKE ? OR p.sku LIKE ?)
       ORDER BY p.is_featured DESC, p.sold_count DESC
       LIMIT ? OFFSET ?`,
      [searchTerm, searchTerm, searchTerm, limit, offset]
    );

    const [[{ total }]] = await db.query(
      `SELECT COUNT(*) as total FROM products p
       WHERE p.is_active = 1
         AND (p.name LIKE ? OR p.description LIKE ? OR p.sku LIKE ?)`,
      [searchTerm, searchTerm, searchTerm]
    );

    return { products, total, page, limit, pages: Math.ceil(total / limit) };
  }

  // ดึงสินค้าตามหมวดหมู่
  async getProductsByCategory(categoryId, page = 1, limit = 10) {
    const offset = (page - 1) * limit;

    const [products] = await db.query(
      `SELECT p.*, c.name as category_name
       FROM products p
       LEFT JOIN categories c ON p.category_id = c.id
       WHERE p.is_active = 1 AND p.category_id = ?
       ORDER BY p.is_featured DESC, p.sold_count DESC
       LIMIT ? OFFSET ?`,
      [categoryId, limit, offset]
    );

    return products;
  }

  // ดึงข้อมูลสินค้าเดียว
  async getProduct(productId) {
    const cacheKey = `product:${productId}`;
    const cached = await redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    const [[product]] = await db.query(
      `SELECT p.*, c.name as category_name
       FROM products p
       LEFT JOIN categories c ON p.category_id = c.id
       WHERE p.id = ? AND p.is_active = 1`,
      [productId]
    );

    if (!product) return null;

    // ดึง variants
    const [variants] = await db.query(
      'SELECT * FROM product_variants WHERE product_id = ? AND is_active = 1',
      [productId]
    );

    product.variants = variants;

    await redis.setex(cacheKey, 300, JSON.stringify(product));
    return product;
  }

  // ดึงหมวดหมู่ทั้งหมด
  async getCategories() {
    const cached = await redis.get('categories');
    if (cached) return JSON.parse(cached);

    const [categories] = await db.query(
      'SELECT * FROM categories WHERE is_active = 1 AND parent_id IS NULL ORDER BY sort_order',
    );

    await redis.setex('categories', 600, JSON.stringify(categories));
    return categories;
  }

  // ตรวจสอบสต็อก
  async checkStock(productId, variantId, quantity) {
    if (variantId) {
      const [[variant]] = await db.query(
        'SELECT stock_quantity FROM product_variants WHERE id = ? AND product_id = ?',
        [variantId, productId]
      );
      return variant && variant.stock_quantity >= quantity;
    }

    const [[product]] = await db.query(
      'SELECT stock_quantity FROM products WHERE id = ?',
      [productId]
    );
    return product && product.stock_quantity >= quantity;
  }

  // ลดสต็อก
  async reduceStock(productId, variantId, quantity, orderId) {
    const connection = await db.getConnection();
    try {
      await connection.beginTransaction();

      if (variantId) {
        const [[variant]] = await connection.query(
          'SELECT stock_quantity FROM product_variants WHERE id = ? FOR UPDATE',
          [variantId]
        );

        if (variant.stock_quantity < quantity) {
          throw new Error('Insufficient stock');
        }

        await connection.query(
          'UPDATE product_variants SET stock_quantity = stock_quantity - ? WHERE id = ?',
          [quantity, variantId]
        );
      } else {
        const [[product]] = await connection.query(
          'SELECT stock_quantity FROM products WHERE id = ? FOR UPDATE',
          [productId]
        );

        if (product.stock_quantity < quantity) {
          throw new Error('Insufficient stock');
        }

        await connection.query(
          'UPDATE products SET stock_quantity = stock_quantity - ?, sold_count = sold_count + ? WHERE id = ?',
          [quantity, quantity, productId]
        );
      }

      // บันทึก stock movement
      const [[currentStock]] = await connection.query(
        'SELECT stock_quantity FROM products WHERE id = ?',
        [productId]
      );

      await connection.query(
        `INSERT INTO stock_movements (product_id, variant_id, type, quantity, balance_after, reference_type, reference_id)
         VALUES (?, ?, 'out', ?, ?, 'order', ?)`,
        [productId, variantId, quantity, currentStock.stock_quantity, orderId]
      );

      await connection.commit();

      // ล้าง cache
      await redis.del(`product:${productId}`);
    } catch (error) {
      await connection.rollback();
      throw error;
    } finally {
      connection.release();
    }
  }
}

module.exports = new ProductService();
```

## 6. Cart Service

```javascript
// src/services/cartService.js
const db = require('../config/database');
const redis = require('../config/redis');
const productService = require('./productService');

class CartService {
  // ดึงตะกร้าสินค้า
  async getCart(userId) {
    const [items] = await db.query(
      `SELECT c.*, p.name as product_name, p.images, p.price as current_price,
              p.stock_quantity,
              pv.name as variant_name, pv.price_modifier,
              (c.quantity * c.price_at_time) as line_total
       FROM carts c
       JOIN products p ON c.product_id = p.id
       LEFT JOIN product_variants pv ON c.variant_id = pv.id
       WHERE c.user_id = ?
       ORDER BY c.created_at DESC`,
      [userId]
    );

    const subtotal = items.reduce((sum, item) => sum + parseFloat(item.line_total), 0);

    return { items, subtotal, itemCount: items.length };
  }

  // เพิ่มสินค้าลงตะกร้า
  async addToCart(userId, productId, variantId, quantity = 1) {
    const product = await productService.getProduct(productId);
    if (!product) throw new Error('Product not found');
    if (!product.is_active) throw new Error('Product is not available');

    const hasStock = await productService.checkStock(productId, variantId, quantity);
    if (!hasStock) throw new Error('Insufficient stock');

    let price = parseFloat(product.sale_price || product.price);
    if (variantId) {
      const variant = product.variants.find(v => v.id === variantId);
      if (!variant) throw new Error('Variant not found');
      price += parseFloat(variant.price_modifier);
    }

    // ตรวจสอบว่ามีในตะกร้าแล้วหรือไม่
    const [[existing]] = await db.query(
      'SELECT id, quantity FROM carts WHERE user_id = ? AND product_id = ? AND COALESCE(variant_id, 0) = COALESCE(?, 0)',
      [userId, productId, variantId]
    );

    if (existing) {
      const newQty = existing.quantity + quantity;
      const hasEnoughStock = await productService.checkStock(productId, variantId, newQty);
      if (!hasEnoughStock) throw new Error('Insufficient stock for requested quantity');

      await db.query(
        'UPDATE carts SET quantity = ?, price_at_time = ? WHERE id = ?',
        [newQty, price, existing.id]
      );
    } else {
      await db.query(
        'INSERT INTO carts (user_id, product_id, variant_id, quantity, price_at_time) VALUES (?, ?, ?, ?, ?)',
        [userId, productId, variantId, quantity, price]
      );
    }

    return this.getCart(userId);
  }

  // อัปเดตจำนวนสินค้าในตะกร้า
  async updateCartItem(userId, cartItemId, quantity) {
    const [[item]] = await db.query(
      'SELECT * FROM carts WHERE id = ? AND user_id = ?',
      [cartItemId, userId]
    );

    if (!item) throw new Error('Cart item not found');

    if (quantity <= 0) {
      return this.removeFromCart(userId, cartItemId);
    }

    const hasStock = await productService.checkStock(item.product_id, item.variant_id, quantity);
    if (!hasStock) throw new Error('Insufficient stock');

    await db.query(
      'UPDATE carts SET quantity = ? WHERE id = ?',
      [quantity, cartItemId]
    );

    return this.getCart(userId);
  }

  // ลบสินค้าออกจากตะกร้า
  async removeFromCart(userId, cartItemId) {
    await db.query(
      'DELETE FROM carts WHERE id = ? AND user_id = ?',
      [cartItemId, userId]
    );
    return this.getCart(userId);
  }

  // ล้างตะกร้า
  async clearCart(userId) {
    await db.query('DELETE FROM carts WHERE user_id = ?', [userId]);
  }

  // คำนวณราคาพร้อมคูปอง
  async calculateTotal(userId, couponCode = null, shippingFee = 0) {
    const cart = await this.getCart(userId);
    if (cart.items.length === 0) throw new Error('Cart is empty');

    let discount = 0;
    let coupon = null;

    if (couponCode) {
      coupon = await this.validateCoupon(couponCode, userId, cart.subtotal);
      if (coupon) {
        if (coupon.type === 'percentage') {
          discount = cart.subtotal * (coupon.value / 100);
          if (coupon.max_discount) {
            discount = Math.min(discount, coupon.max_discount);
          }
        } else if (coupon.type === 'fixed') {
          discount = Math.min(coupon.value, cart.subtotal);
        } else if (coupon.type === 'free_shipping') {
          shippingFee = 0;
        }
      }
    }

    const total = cart.subtotal - discount + shippingFee;

    return {
      cart,
      coupon,
      subtotal: cart.subtotal,
      discount,
      shippingFee,
      total: Math.max(0, total)
    };
  }

  // ตรวจสอบคูปอง
  async validateCoupon(code, userId, orderAmount) {
    const [[coupon]] = await db.query(
      `SELECT c.*, 
       (SELECT COUNT(*) FROM coupon_usages WHERE coupon_id = c.id AND user_id = ?) as user_usage
       FROM coupons c
       WHERE c.code = ? AND c.is_active = 1
         AND (c.start_date IS NULL OR c.start_date <= NOW())
         AND (c.end_date IS NULL OR c.end_date >= NOW())`,
      [userId, code]
    );

    if (!coupon) throw new Error('Coupon not found or expired');
    if (coupon.min_order_amount > orderAmount) {
      throw new Error(`ต้องสั่งซื้อขั้นต่ำ ${coupon.min_order_amount} บาท`);
    }
    if (coupon.usage_limit && coupon.used_count >= coupon.usage_limit) {
      throw new Error('Coupon has reached usage limit');
    }
    if (coupon.per_user_limit && coupon.user_usage >= coupon.per_user_limit) {
      throw new Error('You have already used this coupon');
    }

    return coupon;
  }
}

module.exports = new CartService();
```

## 7. Order Service

```javascript
// src/services/orderService.js
const db = require('../config/database');
const redis = require('../config/redis');
const cartService = require('./cartService');
const productService = require('./productService');
const loyaltyService = require('./loyaltyService');
const { generateOrderNumber } = require('../utils/orderNumber');

class OrderService {
  // สร้างออเดอร์
  async createOrder(userId, shippingAddress, couponCode = null, notes = '') {
    const calculation = await cartService.calculateTotal(userId, couponCode, 50);
    const { cart, coupon, subtotal, discount, shippingFee, total } = calculation;

    const connection = await db.getConnection();
    try {
      await connection.beginTransaction();

      const orderNumber = generateOrderNumber();

      // สร้างออเดอร์
      const [orderResult] = await connection.query(
        `INSERT INTO orders 
         (order_number, user_id, status, subtotal, discount_amount, shipping_fee, total_amount, 
          coupon_id, shipping_address, payment_method, notes)
         VALUES (?, ?, 'pending', ?, ?, ?, ?, ?, ?, 'linepay', ?)`,
        [
          orderNumber, userId, subtotal, discount, shippingFee, total,
          coupon?.id || null, JSON.stringify(shippingAddress), notes
        ]
      );

      const orderId = orderResult.insertId;

      // สร้าง order items
      for (const item of cart.items) {
        await connection.query(
          `INSERT INTO order_items 
           (order_id, product_id, variant_id, product_name, variant_name, sku, quantity, unit_price, total_price, product_snapshot)
           VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)`,
          [
            orderId,
            item.product_id,
            item.variant_id,
            item.product_name,
            item.variant_name,
            item.sku,
            item.quantity,
            item.price_at_time,
            item.line_total,
            JSON.stringify({
              name: item.product_name,
              price: item.price_at_time,
              images: item.images
            })
          ]
        );
      }

      // บันทึก history
      await connection.query(
        `INSERT INTO order_status_history (order_id, status, comment, created_by)
         VALUES (?, 'pending', 'Order created', 'system')`,
        [orderId]
      );

      // อัปเดตการใช้คูปอง
      if (coupon) {
        await connection.query(
          'UPDATE coupons SET used_count = used_count + 1 WHERE id = ?',
          [coupon.id]
        );
        await connection.query(
          'INSERT INTO coupon_usages (coupon_id, user_id, order_id) VALUES (?, ?, ?)',
          [coupon.id, userId, orderId]
        );
      }

      await connection.commit();

      // ดึงออเดอร์ที่สร้าง
      const order = await this.getOrder(orderId);
      return order;
    } catch (error) {
      await connection.rollback();
      throw error;
    } finally {
      connection.release();
    }
  }

  // ดึงข้อมูลออเดอร์
  async getOrder(orderId) {
    const [[order]] = await db.query(
      `SELECT o.*, u.display_name, u.line_user_id
       FROM orders o
       JOIN users u ON o.user_id = u.id
       WHERE o.id = ?`,
      [orderId]
    );

    if (!order) return null;

    const [items] = await db.query(
      'SELECT * FROM order_items WHERE order_id = ?',
      [orderId]
    );

    order.items = items;
    return order;
  }

  // ดึงออเดอร์ตาม order number
  async getOrderByNumber(orderNumber) {
    const [[order]] = await db.query(
      'SELECT * FROM orders WHERE order_number = ?',
      [orderNumber]
    );
    if (!order) return null;
    return this.getOrder(order.id);
  }

  // ดึงออเดอร์ของผู้ใช้
  async getUserOrders(userId, page = 1, limit = 10) {
    const offset = (page - 1) * limit;

    const [orders] = await db.query(
      `SELECT o.*, COUNT(oi.id) as item_count
       FROM orders o
       LEFT JOIN order_items oi ON o.id = oi.order_id
       WHERE o.user_id = ?
       GROUP BY o.id
       ORDER BY o.created_at DESC
       LIMIT ? OFFSET ?`,
      [userId, limit, offset]
    );

    return orders;
  }

  // อัปเดตสถานะการชำระเงิน
  async updatePaymentStatus(orderId, transactionId, paymentDetails, status = 'paid') {
    const connection = await db.getConnection();
    try {
      await connection.beginTransaction();

      await connection.query(
        `UPDATE orders 
         SET payment_status = ?, payment_transaction_id = ?, payment_details = ?, status = 'confirmed'
         WHERE id = ?`,
        [status, transactionId, JSON.stringify(paymentDetails), orderId]
      );

      await connection.query(
        `INSERT INTO order_status_history (order_id, status, comment, created_by)
         VALUES (?, 'confirmed', 'Payment confirmed', 'system')`,
        [orderId]
      );

      // ลดสต็อก
      const [[order]] = await connection.query(
        'SELECT * FROM orders WHERE id = ?',
        [orderId]
      );

      const [items] = await connection.query(
        'SELECT * FROM order_items WHERE order_id = ?',
        [orderId]
      );

      for (const item of items) {
        await productService.reduceStock(item.product_id, item.variant_id, item.quantity, orderId);
      }

      // คำนวณ loyalty points (1 บาท = 1 แต้ม)
      const pointsEarned = Math.floor(parseFloat(order.total_amount));
      await loyaltyService.earnPoints(
        order.user_id,
        pointsEarned,
        'order',
        orderId,
        `คะแนนจากออเดอร์ ${order.order_number}`
      );

      await connection.query(
        'UPDATE orders SET loyalty_points_earned = ? WHERE id = ?',
        [pointsEarned, orderId]
      );

      // ล้างตะกร้า
      await connection.query(
        'DELETE FROM carts WHERE user_id = ?',
        [order.user_id]
      );

      await connection.commit();
    } catch (error) {
      await connection.rollback();
      throw error;
    } finally {
      connection.release();
    }
  }

  // ยกเลิกออเดอร์
  async cancelOrder(orderId, userId, reason) {
    const order = await this.getOrder(orderId);
    if (!order) throw new Error('Order not found');
    if (order.user_id !== userId) throw new Error('Unauthorized');

    const cancellableStatuses = ['pending', 'confirmed'];
    if (!cancellableStatuses.includes(order.status)) {
      throw new Error(`Cannot cancel order with status: ${order.status}`);
    }

    await db.query(
      `UPDATE orders SET status = 'cancelled', cancel_reason = ?, cancelled_at = NOW()
       WHERE id = ?`,
      [reason, orderId]
    );

    await db.query(
      `INSERT INTO order_status_history (order_id, status, comment, created_by)
       VALUES (?, 'cancelled', ?, ?)`,
      [orderId, reason, `user:${userId}`]
    );

    // คืนสต็อก
    for (const item of order.items) {
      await db.query(
        'UPDATE products SET stock_quantity = stock_quantity + ? WHERE id = ?',
        [item.quantity, item.product_id]
      );
    }

    // คืนคะแนน loyalty ถ้าได้รับไปแล้ว
    if (order.loyalty_points_earned > 0) {
      await loyaltyService.adjustPoints(
        order.user_id,
        -order.loyalty_points_earned,
        'order',
        orderId,
        `คืนคะแนนเนื่องจากยกเลิกออเดอร์ ${order.order_number}`
      );
    }
  }
}

module.exports = new OrderService();
```

## 8. LINE Pay Service

```javascript
// src/services/paymentService.js
const { linePayRequest } = require('../config/linepay');
const orderService = require('./orderService');

class PaymentService {
  // สร้าง payment request
  async requestPayment(order, confirmUrl, cancelUrl) {
    const packages = [{
      id: `pkg_${order.id}`,
      amount: Math.round(order.total_amount * 100) / 100,
      name: `Order ${order.order_number}`,
      products: order.items.map(item => ({
        id: String(item.product_id),
        name: item.product_name,
        quantity: item.quantity,
        price: Math.round(item.unit_price * 100) / 100
      }))
    }];

    const requestBody = {
      amount: Math.round(order.total_amount * 100) / 100,
      currency: 'THB',
      orderId: order.order_number,
      packages,
      redirectUrls: {
        confirmUrl,
        cancelUrl
      },
      options: {
        payment: {
          capture: true
        }
      }
    };

    const response = await linePayRequest('POST', '/v3/payments/request', requestBody);

    if (response.returnCode !== '0000') {
      throw new Error(`LINE Pay request failed: ${response.returnMessage}`);
    }

    return {
      transactionId: response.info.transactionId,
      paymentUrl: response.info.paymentUrl.web,
      paymentUrlMobile: response.info.paymentUrl.app
    };
  }

  // ยืนยันการชำระเงิน
  async confirmPayment(transactionId, orderId) {
    const order = await orderService.getOrder(orderId);
    if (!order) throw new Error('Order not found');

    const requestBody = {
      amount: Math.round(order.total_amount * 100) / 100,
      currency: 'THB'
    };

    const response = await linePayRequest(
      'POST',
      `/v3/payments/${transactionId}/confirm`,
      requestBody
    );

    if (response.returnCode !== '0000') {
      throw new Error(`LINE Pay confirm failed: ${response.returnMessage}`);
    }

    await orderService.updatePaymentStatus(
      orderId,
      String(transactionId),
      response.info,
      'paid'
    );

    return response.info;
  }

  // คืนเงิน
  async refundPayment(transactionId, amount, reason) {
    const requestBody = {
      refundAmount: Math.round(amount * 100) / 100
    };

    const response = await linePayRequest(
      'POST',
      `/v3/payments/${transactionId}/refund`,
      requestBody
    );

    if (response.returnCode !== '0000') {
      throw new Error(`LINE Pay refund failed: ${response.returnMessage}`);
    }

    return response.info;
  }
}

module.exports = new PaymentService();
```

## 9. Loyalty Points Service

```javascript
// src/services/loyaltyService.js
const db = require('../config/database');

const TIER_THRESHOLDS = {
  bronze: 0,
  silver: 5000,
  gold: 15000,
  platinum: 50000
};

class LoyaltyService {
  // เพิ่มคะแนน
  async earnPoints(userId, points, referenceType, referenceId, description) {
    const connection = await db.getConnection();
    try {
      await connection.beginTransaction();

      const [[user]] = await connection.query(
        'SELECT loyalty_points, total_spent FROM users WHERE id = ? FOR UPDATE',
        [userId]
      );

      const newBalance = user.loyalty_points + points;

      await connection.query(
        'UPDATE users SET loyalty_points = ? WHERE id = ?',
        [newBalance, userId]
      );

      await connection.query(
        `INSERT INTO loyalty_transactions 
         (user_id, type, points, balance_after, reference_type, reference_id, description, expires_at)
         VALUES (?, 'earn', ?, ?, ?, ?, ?, DATE_ADD(NOW(), INTERVAL 1 YEAR))`,
        [userId, points, newBalance, referenceType, referenceId, description]
      );

      // อัปเดต tier
      await this.updateTier(userId, user.total_spent, connection);

      await connection.commit();
      return newBalance;
    } catch (error) {
      await connection.rollback();
      throw error;
    } finally {
      connection.release();
    }
  }

  // ใช้คะแนน
  async redeemPoints(userId, points, orderId, description) {
    const connection = await db.getConnection();
    try {
      await connection.beginTransaction();

      const [[user]] = await connection.query(
        'SELECT loyalty_points FROM users WHERE id = ? FOR UPDATE',
        [userId]
      );

      if (user.loyalty_points < points) {
        throw new Error('Insufficient loyalty points');
      }

      const newBalance = user.loyalty_points - points;

      await connection.query(
        'UPDATE users SET loyalty_points = ? WHERE id = ?',
        [newBalance, userId]
      );

      await connection.query(
        `INSERT INTO loyalty_transactions 
         (user_id, type, points, balance_after, reference_type, reference_id, description)
         VALUES (?, 'redeem', ?, ?, 'order', ?, ?)`,
        [userId, -points, newBalance, orderId, description]
      );

      await connection.commit();
      return newBalance;
    } catch (error) {
      await connection.rollback();
      throw error;
    } finally {
      connection.release();
    }
  }

  // ปรับแต้ม (admin)
  async adjustPoints(userId, points, referenceType, referenceId, description) {
    const connection = await db.getConnection();
    try {
      await connection.beginTransaction();

      const [[user]] = await connection.query(
        'SELECT loyalty_points FROM users WHERE id = ? FOR UPDATE',
        [userId]
      );

      const newBalance = Math.max(0, user.loyalty_points + points);

      await connection.query(
        'UPDATE users SET loyalty_points = ? WHERE id = ?',
        [newBalance, userId]
      );

      await connection.query(
        `INSERT INTO loyalty_transactions 
         (user_id, type, points, balance_after, reference_type, reference_id, description)
         VALUES (?, 'adjust', ?, ?, ?, ?, ?)`,
        [userId, points, newBalance, referenceType, referenceId, description]
      );

      await connection.commit();
      return newBalance;
    } catch (error) {
      await connection.rollback();
      throw error;
    } finally {
      connection.release();
    }
  }

  // อัปเดต tier
  async updateTier(userId, totalSpent, connection) {
    const db_conn = connection || db;

    let tier = 'bronze';
    if (totalSpent >= TIER_THRESHOLDS.platinum) tier = 'platinum';
    else if (totalSpent >= TIER_THRESHOLDS.gold) tier = 'gold';
    else if (totalSpent >= TIER_THRESHOLDS.silver) tier = 'silver';

    await db_conn.query(
      'UPDATE users SET tier = ? WHERE id = ?',
      [tier, userId]
    );
  }

  // ดึงประวัติคะแนน
  async getPointsHistory(userId, limit = 20) {
    const [transactions] = await db.query(
      `SELECT * FROM loyalty_transactions
       WHERE user_id = ?
       ORDER BY created_at DESC
       LIMIT ?`,
      [userId, limit]
    );
    return transactions;
  }
}

module.exports = new LoyaltyService();
```

## 10. Flex Message Templates

```javascript
// src/messages/productMessages.js
const { PRODUCT_PER_PAGE } = process.env;

function createProductCard(product) {
  const price = product.sale_price || product.price;
  const hasDiscount = product.sale_price && product.sale_price < product.price;
  const images = JSON.parse(product.images || '[]');
  const imageUrl = images[0] || 'https://via.placeholder.com/400x300';

  return {
    type: 'bubble',
    size: 'kilo',
    hero: {
      type: 'image',
      url: imageUrl,
      size: 'full',
      aspectRatio: '20:13',
      aspectMode: 'cover',
      action: {
        type: 'postback',
        data: `action=view_product&product_id=${product.id}`
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
          size: 'sm',
          wrap: true,
          maxLines: 2
        },
        {
          type: 'box',
          layout: 'baseline',
          margin: 'sm',
          contents: hasDiscount ? [
            {
              type: 'text',
              text: `฿${parseFloat(price).toLocaleString('th-TH')}`,
              weight: 'bold',
              color: '#E53E3E',
              size: 'md',
              flex: 0
            },
            {
              type: 'text',
              text: `฿${parseFloat(product.price).toLocaleString('th-TH')}`,
              size: 'xs',
              color: '#999999',
              decoration: 'line-through',
              margin: 'sm',
              flex: 0
            }
          ] : [
            {
              type: 'text',
              text: `฿${parseFloat(price).toLocaleString('th-TH')}`,
              weight: 'bold',
              color: '#2D3748',
              size: 'md'
            }
          ]
        },
        product.stock_quantity <= 10 && product.stock_quantity > 0 ? {
          type: 'text',
          text: `เหลือเพียง ${product.stock_quantity} ชิ้น!`,
          size: 'xs',
          color: '#E53E3E',
          margin: 'sm'
        } : null,
        product.stock_quantity === 0 ? {
          type: 'text',
          text: 'สินค้าหมด',
          size: 'xs',
          color: '#999999',
          margin: 'sm'
        } : null
      ].filter(Boolean)
    },
    footer: {
      type: 'box',
      layout: 'vertical',
      spacing: 'sm',
      contents: [
        {
          type: 'button',
          style: 'primary',
          height: 'sm',
          action: {
            type: 'postback',
            label: product.stock_quantity > 0 ? '🛒 เพิ่มลงตะกร้า' : 'สินค้าหมด',
            data: `action=add_cart&product_id=${product.id}`
          },
          color: product.stock_quantity > 0 ? '#06C755' : '#999999'
        }
      ]
    }
  };
}

function createProductCatalog(products, page, totalPages, title = 'สินค้าของเรา') {
  const bubbles = products.map(p => createProductCard(p));

  // เพิ่ม navigation
  const navBubble = {
    type: 'bubble',
    size: 'nano',
    body: {
      type: 'box',
      layout: 'vertical',
      contents: [
        {
          type: 'text',
          text: title,
          weight: 'bold',
          size: 'md',
          wrap: true
        },
        {
          type: 'text',
          text: `หน้า ${page}/${totalPages}`,
          size: 'xs',
          color: '#999999',
          margin: 'sm'
        }
      ]
    },
    footer: {
      type: 'box',
      layout: 'horizontal',
      contents: [
        page > 1 ? {
          type: 'button',
          action: {
            type: 'postback',
            label: '◀ ก่อนหน้า',
            data: `action=catalog&page=${page - 1}`
          },
          style: 'secondary',
          height: 'sm',
          flex: 1
        } : { type: 'filler' },
        page < totalPages ? {
          type: 'button',
          action: {
            type: 'postback',
            label: 'ถัดไป ▶',
            data: `action=catalog&page=${page + 1}`
          },
          style: 'primary',
          height: 'sm',
          flex: 1,
          color: '#06C755'
        } : { type: 'filler' }
      ]
    }
  };

  bubbles.push(navBubble);

  return {
    type: 'flex',
    altText: title,
    contents: {
      type: 'carousel',
      contents: bubbles
    }
  };
}

function createProductDetail(product) {
  const price = product.sale_price || product.price;
  const images = JSON.parse(product.images || '[]');
  const imageUrl = images[0] || 'https://via.placeholder.com/400x300';

  const variantButtons = (product.variants || []).map(v => ({
    type: 'button',
    action: {
      type: 'postback',
      label: `${v.name} (+฿${v.price_modifier})`,
      data: `action=select_variant&product_id=${product.id}&variant_id=${v.id}`
    },
    style: 'secondary',
    height: 'sm',
    margin: 'sm'
  }));

  return {
    type: 'flex',
    altText: product.name,
    contents: {
      type: 'bubble',
      hero: {
        type: 'image',
        url: imageUrl,
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
            text: product.name,
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
                text: `฿${parseFloat(price).toLocaleString('th-TH')}`,
                weight: 'bold',
                color: '#E53E3E',
                size: 'xl',
                flex: 0
              }
            ]
          },
          {
            type: 'text',
            text: product.description || '',
            size: 'sm',
            color: '#666666',
            wrap: true,
            margin: 'md',
            maxLines: 3
          },
          {
            type: 'separator',
            margin: 'lg'
          },
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'md',
            contents: [
              {
                type: 'text',
                text: 'คงเหลือ:',
                size: 'sm',
                color: '#666666',
                flex: 2
              },
              {
                type: 'text',
                text: `${product.stock_quantity} ชิ้น`,
                size: 'sm',
                color: product.stock_quantity > 10 ? '#27AE60' : '#E53E3E',
                flex: 3
              }
            ]
          },
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'xs',
            contents: [
              {
                type: 'text',
                text: 'คะแนน:',
                size: 'sm',
                color: '#666666',
                flex: 2
              },
              {
                type: 'text',
                text: `⭐ ${product.rating} (${product.review_count} รีวิว)`,
                size: 'sm',
                flex: 3
              }
            ]
          },
          ...(variantButtons.length > 0 ? [
            { type: 'separator', margin: 'md' },
            {
              type: 'text',
              text: 'เลือกรุ่น:',
              size: 'sm',
              weight: 'bold',
              margin: 'md'
            },
            ...variantButtons
          ] : [])
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
              label: '🛒 เพิ่มลงตะกร้า',
              data: `action=add_cart&product_id=${product.id}`
            },
            color: '#06C755'
          },
          {
            type: 'button',
            style: 'secondary',
            action: {
              type: 'postback',
              label: '← กลับไปดูสินค้า',
              data: 'action=catalog&page=1'
            }
          }
        ]
      }
    }
  };
}

module.exports = { createProductCard, createProductCatalog, createProductDetail };
```

```javascript
// src/messages/cartMessages.js
function createCartMessage(cart, couponCode = null, discount = 0) {
  if (cart.items.length === 0) {
    return {
      type: 'flex',
      altText: 'ตะกร้าสินค้า',
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: '🛒 ตะกร้าของคุณว่างเปล่า',
              size: 'md',
              align: 'center',
              weight: 'bold'
            },
            {
              type: 'text',
              text: 'เพิ่มสินค้าที่คุณต้องการลงในตะกร้า',
              size: 'sm',
              color: '#999999',
              align: 'center',
              margin: 'md',
              wrap: true
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
                type: 'postback',
                label: '🛍️ ดูสินค้า',
                data: 'action=catalog&page=1'
              },
              style: 'primary',
              color: '#06C755'
            }
          ]
        }
      }
    };
  }

  const itemContents = cart.items.map(item => ({
    type: 'box',
    layout: 'horizontal',
    contents: [
      {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: item.product_name,
            size: 'sm',
            weight: 'bold',
            wrap: true,
            maxLines: 2
          },
          item.variant_name ? {
            type: 'text',
            text: item.variant_name,
            size: 'xs',
            color: '#999999'
          } : null,
          {
            type: 'text',
            text: `฿${parseFloat(item.price_at_time).toLocaleString('th-TH')} x ${item.quantity}`,
            size: 'xs',
            color: '#666666',
            margin: 'xs'
          }
        ].filter(Boolean),
        flex: 4
      },
      {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: `฿${parseFloat(item.line_total).toLocaleString('th-TH')}`,
            size: 'sm',
            weight: 'bold',
            align: 'end'
          },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '🗑',
              data: `action=remove_cart&cart_id=${item.id}`
            },
            style: 'secondary',
            height: 'sm'
          }
        ],
        flex: 2
      }
    ],
    margin: 'sm'
  }));

  const shippingFee = 50;
  const total = cart.subtotal - discount + shippingFee;

  return {
    type: 'flex',
    altText: `ตะกร้าสินค้า ${cart.itemCount} รายการ`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: `🛒 ตะกร้าสินค้า (${cart.itemCount} รายการ)`,
            weight: 'bold',
            size: 'md'
          }
        ],
        backgroundColor: '#F7F8FA'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          ...itemContents,
          { type: 'separator', margin: 'md' },
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'md',
            contents: [
              { type: 'text', text: 'ราคาสินค้า:', size: 'sm', flex: 3 },
              { type: 'text', text: `฿${cart.subtotal.toLocaleString('th-TH')}`, size: 'sm', align: 'end', flex: 2 }
            ]
          },
          discount > 0 ? {
            type: 'box',
            layout: 'horizontal',
            margin: 'xs',
            contents: [
              { type: 'text', text: 'ส่วนลด:', size: 'sm', color: '#E53E3E', flex: 3 },
              { type: 'text', text: `-฿${discount.toLocaleString('th-TH')}`, size: 'sm', color: '#E53E3E', align: 'end', flex: 2 }
            ]
          } : null,
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'xs',
            contents: [
              { type: 'text', text: 'ค่าจัดส่ง:', size: 'sm', flex: 3 },
              { type: 'text', text: `฿${shippingFee}`, size: 'sm', align: 'end', flex: 2 }
            ]
          },
          { type: 'separator', margin: 'sm' },
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'sm',
            contents: [
              { type: 'text', text: 'รวมทั้งสิ้น:', size: 'md', weight: 'bold', flex: 3 },
              { type: 'text', text: `฿${total.toLocaleString('th-TH')}`, size: 'md', weight: 'bold', color: '#E53E3E', align: 'end', flex: 2 }
            ]
          }
        ].filter(Boolean)
      },
      footer: {
        type: 'box',
        layout: 'vertical',
        spacing: 'sm',
        contents: [
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '🏷️ ใส่รหัสคูปอง',
              data: 'action=enter_coupon'
            },
            style: 'secondary',
            height: 'sm'
          },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '💳 ดำเนินการสั่งซื้อ',
              data: 'action=checkout'
            },
            style: 'primary',
            color: '#06C755',
            height: 'sm'
          }
        ]
      }
    }
  };
}

module.exports = { createCartMessage };
```

## 11. Postback Handler

```javascript
// src/handlers/postbackHandler.js
const line = require('@line/bot-sdk');
const productService = require('../services/productService');
const cartService = require('../services/cartService');
const orderService = require('../services/orderService');
const paymentService = require('../services/paymentService');
const loyaltyService = require('../services/loyaltyService');
const { createProductCatalog, createProductDetail } = require('../messages/productMessages');
const { createCartMessage } = require('../messages/cartMessages');
const db = require('../config/database');

async function getOrCreateUser(lineUserId, profileData = null) {
  const [[existing]] = await db.query(
    'SELECT * FROM users WHERE line_user_id = ?',
    [lineUserId]
  );

  if (existing) return existing;

  const [result] = await db.query(
    `INSERT INTO users (line_user_id, display_name, picture_url) VALUES (?, ?, ?)`,
    [lineUserId, profileData?.displayName || '', profileData?.pictureUrl || '']
  );

  const [[user]] = await db.query('SELECT * FROM users WHERE id = ?', [result.insertId]);
  return user;
}

async function handlePostback(client, event) {
  const userId = event.source.userId;
  const params = new URLSearchParams(event.postback.data);
  const action = params.get('action');

  const user = await getOrCreateUser(userId);

  try {
    switch (action) {
      case 'catalog': {
        const page = parseInt(params.get('page') || '1');
        const result = await productService.searchProducts('', page, 6);
        const message = createProductCatalog(result.products, page, result.pages);
        await client.replyMessage(event.replyToken, message);
        break;
      }

      case 'view_product': {
        const productId = parseInt(params.get('product_id'));
        const product = await productService.getProduct(productId);
        if (!product) {
          await client.replyMessage(event.replyToken, {
            type: 'text',
            text: 'ไม่พบสินค้าที่คุณเลือก'
          });
          break;
        }
        const message = createProductDetail(product);
        await client.replyMessage(event.replyToken, message);
        break;
      }

      case 'add_cart': {
        const productId = parseInt(params.get('product_id'));
        const variantId = params.get('variant_id') ? parseInt(params.get('variant_id')) : null;

        try {
          const cart = await cartService.addToCart(user.id, productId, variantId, 1);
          const message = createCartMessage(cart);
          await client.replyMessage(event.replyToken, [
            { type: 'text', text: '✅ เพิ่มสินค้าลงตะกร้าแล้ว!' },
            message
          ]);
        } catch (err) {
          await client.replyMessage(event.replyToken, {
            type: 'text',
            text: `❌ ${err.message}`
          });
        }
        break;
      }

      case 'view_cart': {
        const cart = await cartService.getCart(user.id);
        const message = createCartMessage(cart);
        await client.replyMessage(event.replyToken, message);
        break;
      }

      case 'remove_cart': {
        const cartId = parseInt(params.get('cart_id'));
        const cart = await cartService.removeFromCart(user.id, cartId);
        const message = createCartMessage(cart);
        await client.replyMessage(event.replyToken, [
          { type: 'text', text: '🗑️ ลบสินค้าออกจากตะกร้าแล้ว' },
          message
        ]);
        break;
      }

      case 'enter_coupon': {
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: '🏷️ กรุณาพิมพ์รหัสคูปองที่คุณต้องการใช้:\n\n(พิมพ์ "ยกเลิก" เพื่อกลับ)'
        });
        // เก็บ state ว่ากำลังรอรหัสคูปอง
        const redis = require('../config/redis');
        await redis.setex(`state:${userId}`, 300, JSON.stringify({ state: 'waiting_coupon' }));
        break;
      }

      case 'checkout': {
        const cart = await cartService.getCart(user.id);
        if (cart.items.length === 0) {
          await client.replyMessage(event.replyToken, {
            type: 'text',
            text: '❌ ตะกร้าสินค้าของคุณว่างเปล่า'
          });
          break;
        }

        // ถามที่อยู่จัดส่ง
        await client.replyMessage(event.replyToken, {
          type: 'flex',
          altText: 'ข้อมูลการจัดส่ง',
          contents: {
            type: 'bubble',
            body: {
              type: 'box',
              layout: 'vertical',
              contents: [
                {
                  type: 'text',
                  text: '📦 ข้อมูลการจัดส่ง',
                  weight: 'bold',
                  size: 'lg'
                },
                {
                  type: 'text',
                  text: 'กรุณากรอกที่อยู่จัดส่งในรูปแบบ:\n\nชื่อ-นามสกุล\nเบอร์โทรศัพท์\nที่อยู่บ้านเลขที่/ถนน\nตำบล/แขวง อำเภอ/เขต\nจังหวัด รหัสไปรษณีย์',
                  size: 'sm',
                  wrap: true,
                  margin: 'md',
                  color: '#666666'
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
                    type: 'postback',
                    label: 'ใช้ที่อยู่เดิม',
                    data: 'action=use_saved_address'
                  },
                  style: 'primary',
                  color: '#06C755'
                }
              ]
            }
          }
        });

        const redis = require('../config/redis');
        await redis.setex(`state:${userId}`, 600, JSON.stringify({ state: 'waiting_address' }));
        break;
      }

      case 'use_saved_address': {
        if (!user.address) {
          await client.replyMessage(event.replyToken, {
            type: 'text',
            text: 'ไม่พบที่อยู่ที่บันทึกไว้ กรุณากรอกที่อยู่จัดส่ง'
          });
          const redis = require('../config/redis');
          await redis.setex(`state:${userId}`, 600, JSON.stringify({ state: 'waiting_address' }));
          break;
        }

        // ดำเนินการสั่งซื้อ
        await processOrder(client, event, user, JSON.parse(user.address));
        break;
      }

      case 'confirm_order': {
        const orderId = parseInt(params.get('order_id'));
        const order = await orderService.getOrder(orderId);

        if (!order || order.user_id !== user.id) {
          await client.replyMessage(event.replyToken, { type: 'text', text: '❌ ไม่พบออเดอร์' });
          break;
        }

        // สร้าง LINE Pay payment
        const confirmUrl = `${process.env.BASE_URL}/linepay/confirm?orderId=${orderId}`;
        const cancelUrl = `${process.env.BASE_URL}/linepay/cancel?orderId=${orderId}`;

        const payment = await paymentService.requestPayment(order, confirmUrl, cancelUrl);

        await client.replyMessage(event.replyToken, {
          type: 'flex',
          altText: 'ชำระเงินผ่าน LINE Pay',
          contents: {
            type: 'bubble',
            body: {
              type: 'box',
              layout: 'vertical',
              contents: [
                {
                  type: 'text',
                  text: '💳 ชำระเงินผ่าน LINE Pay',
                  weight: 'bold',
                  size: 'lg'
                },
                {
                  type: 'text',
                  text: `ออเดอร์: ${order.order_number}`,
                  size: 'sm',
                  margin: 'md'
                },
                {
                  type: 'text',
                  text: `ยอดชำระ: ฿${parseFloat(order.total_amount).toLocaleString('th-TH')}`,
                  size: 'md',
                  weight: 'bold',
                  color: '#E53E3E',
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
                    label: '💳 ชำระเงินเลย',
                    uri: payment.paymentUrl
                  },
                  style: 'primary',
                  color: '#06C755'
                }
              ]
            }
          }
        });
        break;
      }

      case 'my_orders': {
        const orders = await orderService.getUserOrders(user.id, 1, 5);
        if (orders.length === 0) {
          await client.replyMessage(event.replyToken, {
            type: 'text',
            text: '📦 คุณยังไม่มีออเดอร์\n\nเริ่มช้อปปิ้งได้เลย!'
          });
          break;
        }

        const orderBubbles = orders.map(order => createOrderCard(order));
        await client.replyMessage(event.replyToken, {
          type: 'flex',
          altText: 'ออเดอร์ของฉัน',
          contents: {
            type: 'carousel',
            contents: orderBubbles
          }
        });
        break;
      }

      case 'my_points': {
        const history = await loyaltyService.getPointsHistory(user.id, 10);
        await client.replyMessage(event.replyToken, createPointsMessage(user, history));
        break;
      }

      default:
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: 'ไม่เข้าใจคำสั่ง กรุณาลองใหม่อีกครั้ง'
        });
    }
  } catch (error) {
    console.error('Postback handler error:', error);
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: '❌ เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง'
    });
  }
}

async function processOrder(client, event, user, shippingAddress) {
  try {
    const order = await orderService.createOrder(user.id, shippingAddress);

    await client.replyMessage(event.replyToken, {
      type: 'flex',
      altText: `สร้างออเดอร์ ${order.order_number} แล้ว`,
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: '✅ สร้างออเดอร์สำเร็จ!',
              weight: 'bold',
              size: 'lg',
              color: '#27AE60'
            },
            {
              type: 'text',
              text: `หมายเลขออเดอร์: ${order.order_number}`,
              size: 'sm',
              margin: 'md'
            },
            {
              type: 'text',
              text: `ยอดรวม: ฿${parseFloat(order.total_amount).toLocaleString('th-TH')}`,
              size: 'md',
              weight: 'bold',
              color: '#E53E3E',
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
                type: 'postback',
                label: '💳 ชำระเงินผ่าน LINE Pay',
                data: `action=confirm_order&order_id=${order.id}`
              },
              style: 'primary',
              color: '#06C755'
            }
          ]
        }
      }
    });
  } catch (error) {
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: `❌ เกิดข้อผิดพลาดในการสร้างออเดอร์: ${error.message}`
    });
  }
}

function createOrderCard(order) {
  const statusEmoji = {
    pending: '⏳',
    confirmed: '✅',
    processing: '🔧',
    shipped: '🚚',
    delivered: '📦',
    cancelled: '❌',
    refunded: '💸'
  };

  const statusText = {
    pending: 'รอชำระเงิน',
    confirmed: 'ยืนยันแล้ว',
    processing: 'กำลังดำเนินการ',
    shipped: 'จัดส่งแล้ว',
    delivered: 'ส่งถึงแล้ว',
    cancelled: 'ยกเลิกแล้ว',
    refunded: 'คืนเงินแล้ว'
  };

  return {
    type: 'bubble',
    body: {
      type: 'box',
      layout: 'vertical',
      contents: [
        {
          type: 'text',
          text: `${statusEmoji[order.status] || '📋'} ${statusText[order.status] || order.status}`,
          weight: 'bold',
          size: 'md'
        },
        {
          type: 'text',
          text: order.order_number,
          size: 'xs',
          color: '#999999',
          margin: 'xs'
        },
        {
          type: 'separator',
          margin: 'md'
        },
        {
          type: 'box',
          layout: 'horizontal',
          margin: 'md',
          contents: [
            { type: 'text', text: 'จำนวนสินค้า:', size: 'sm', flex: 3 },
            { type: 'text', text: `${order.item_count} รายการ`, size: 'sm', flex: 2, align: 'end' }
          ]
        },
        {
          type: 'box',
          layout: 'horizontal',
          margin: 'xs',
          contents: [
            { type: 'text', text: 'ยอดรวม:', size: 'sm', flex: 3 },
            { type: 'text', text: `฿${parseFloat(order.total_amount).toLocaleString('th-TH')}`, size: 'sm', weight: 'bold', flex: 2, align: 'end', color: '#E53E3E' }
          ]
        },
        {
          type: 'text',
          text: new Date(order.created_at).toLocaleDateString('th-TH'),
          size: 'xs',
          color: '#999999',
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
            type: 'postback',
            label: 'ดูรายละเอียด',
            data: `action=order_detail&order_id=${order.id}`
          },
          style: 'secondary',
          height: 'sm'
        }
      ]
    }
  };
}

function createPointsMessage(user, history) {
  const tierColors = {
    bronze: '#CD7F32',
    silver: '#C0C0C0',
    gold: '#FFD700',
    platinum: '#E5E4E2'
  };

  const tierEmoji = {
    bronze: '🥉',
    silver: '🥈',
    gold: '🥇',
    platinum: '💎'
  };

  const historyContents = history.slice(0, 5).map(t => ({
    type: 'box',
    layout: 'horizontal',
    margin: 'xs',
    contents: [
      {
        type: 'text',
        text: t.description || `${t.type} points`,
        size: 'xs',
        flex: 4,
        wrap: true,
        maxLines: 1
      },
      {
        type: 'text',
        text: `${t.points > 0 ? '+' : ''}${t.points}`,
        size: 'xs',
        color: t.points > 0 ? '#27AE60' : '#E53E3E',
        flex: 2,
        align: 'end'
      }
    ]
  }));

  return {
    type: 'flex',
    altText: `คะแนนสะสม ${user.loyalty_points} แต้ม`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: `${tierEmoji[user.tier]} ${user.tier.toUpperCase()} Member`,
            weight: 'bold',
            size: 'lg',
            color: '#FFFFFF'
          }
        ],
        backgroundColor: tierColors[user.tier] || '#06C755'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'text',
            text: `${user.loyalty_points.toLocaleString('th-TH')} แต้ม`,
            weight: 'bold',
            size: 'xxl',
            color: '#06C755',
            align: 'center'
          },
          {
            type: 'text',
            text: 'คะแนนสะสมของคุณ',
            size: 'sm',
            color: '#999999',
            align: 'center'
          },
          { type: 'separator', margin: 'lg' },
          {
            type: 'text',
            text: 'ประวัติล่าสุด:',
            size: 'sm',
            weight: 'bold',
            margin: 'md'
          },
          ...historyContents
        ]
      }
    }
  };
}

module.exports = { handlePostback };
```

## 12. Main Webhook Route

```javascript
// src/routes/webhook.js
const express = require('express');
const line = require('@line/bot-sdk');
const redis = require('../config/redis');
const db = require('../config/database');
const { handlePostback } = require('../handlers/postbackHandler');
const cartService = require('../services/cartService');
const orderService = require('../services/orderService');
const productService = require('../services/productService');
const { createProductCatalog } = require('../messages/productMessages');
const { createCartMessage } = require('../messages/cartMessages');

const router = express.Router();

const lineConfig = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};

const client = new line.Client(lineConfig);

router.post('/', line.middleware(lineConfig), async (req, res) => {
  try {
    await Promise.all(req.body.events.map(event => handleEvent(event)));
    res.json({ success: true });
  } catch (error) {
    console.error('Webhook error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

async function handleEvent(event) {
  if (event.type === 'follow') {
    await handleFollow(event);
  } else if (event.type === 'postback') {
    await handlePostback(client, event);
  } else if (event.type === 'message') {
    await handleMessage(event);
  }
}

async function handleFollow(event) {
  const userId = event.source.userId;
  try {
    const profile = await client.getProfile(userId);
    const [[existing]] = await db.query(
      'SELECT id FROM users WHERE line_user_id = ?', [userId]
    );

    if (!existing) {
      await db.query(
        'INSERT INTO users (line_user_id, display_name, picture_url) VALUES (?, ?, ?)',
        [userId, profile.displayName, profile.pictureUrl]
      );
    }

    await client.replyMessage(event.replyToken, [
      {
        type: 'text',
        text: `สวัสดีครับ ${profile.displayName}! 🎉\n\nยินดีต้อนรับสู่ร้านค้าออนไลน์ของเรา\nพิมพ์ "เมนู" เพื่อดูตัวเลือกทั้งหมด`
      },
      createMainMenu()
    ]);
  } catch (error) {
    console.error('Follow handler error:', error);
  }
}

async function handleMessage(event) {
  const userId = event.source.userId;
  const text = event.message.text?.trim() || '';

  // ตรวจสอบ state
  const stateData = await redis.get(`state:${userId}`);
  const state = stateData ? JSON.parse(stateData) : null;

  const [[user]] = await db.query(
    'SELECT * FROM users WHERE line_user_id = ?', [userId]
  );

  if (!user) {
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: 'กรุณา Unfollow และ Follow ใหม่อีกครั้ง'
    });
    return;
  }

  // Handle state-based inputs
  if (state?.state === 'waiting_coupon') {
    await redis.del(`state:${userId}`);
    if (text.toLowerCase() === 'ยกเลิก') {
      const cart = await cartService.getCart(user.id);
      await client.replyMessage(event.replyToken, createCartMessage(cart));
      return;
    }

    try {
      const coupon = await cartService.validateCoupon(text.toUpperCase(), user.id, 0);
      await redis.setex(`coupon:${userId}`, 3600, text.toUpperCase());
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: `✅ ใช้คูปอง "${text.toUpperCase()}" สำเร็จ!\n${coupon.name}`
      });
    } catch (err) {
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: `❌ ${err.message}`
      });
    }
    return;
  }

  if (state?.state === 'waiting_address') {
    await redis.del(`state:${userId}`);
    // บันทึกที่อยู่
    const address = { raw: text };
    await db.query('UPDATE users SET address = ? WHERE id = ?', [JSON.stringify(address), user.id]);

    // สร้างออเดอร์
    const couponCode = await redis.get(`coupon:${userId}`);
    try {
      const order = await orderService.createOrder(user.id, address, couponCode);
      await redis.del(`coupon:${userId}`);

      await client.replyMessage(event.replyToken, {
        type: 'flex',
        altText: `สร้างออเดอร์ ${order.order_number}`,
        contents: createOrderSummaryBubble(order)
      });
    } catch (err) {
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: `❌ ${err.message}`
      });
    }
    return;
  }

  // ตรวจสอบคำสั่งทั่วไป
  const commands = {
    'เมนู': () => client.replyMessage(event.replyToken, createMainMenu()),
    'สินค้า': async () => {
      const result = await productService.searchProducts('', 1, 6);
      const msg = createProductCatalog(result.products, 1, result.pages);
      await client.replyMessage(event.replyToken, msg);
    },
    'ตะกร้า': async () => {
      const cart = await cartService.getCart(user.id);
      await client.replyMessage(event.replyToken, createCartMessage(cart));
    },
    'ออเดอร์': async () => {
      const orders = await orderService.getUserOrders(user.id, 1, 5);
      if (orders.length === 0) {
        await client.replyMessage(event.replyToken, { type: 'text', text: 'ยังไม่มีออเดอร์' });
      } else {
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: `📦 ออเดอร์ล่าสุดของคุณ:\n${orders.map(o => `${o.order_number} - ${o.status}`).join('\n')}`
        });
      }
    },
    'แต้ม': async () => {
      const history = await loyaltyService.getPointsHistory(user.id, 10);
      await client.replyMessage(event.replyToken, createPointsMessage(user, history));
    }
  };

  if (commands[text]) {
    await commands[text]();
  } else if (text.startsWith('ค้นหา ')) {
    const keyword = text.replace('ค้นหา ', '');
    const result = await productService.searchProducts(keyword, 1, 6);
    if (result.products.length === 0) {
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: `ไม่พบสินค้า "${keyword}"`
      });
    } else {
      const msg = createProductCatalog(result.products, 1, result.pages, `ค้นหา: ${keyword}`);
      await client.replyMessage(event.replyToken, msg);
    }
  } else {
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: 'พิมพ์ "เมนู" เพื่อดูตัวเลือกทั้งหมด\nหรือพิมพ์ "ค้นหา [ชื่อสินค้า]" เพื่อค้นหาสินค้า'
    });
  }
}

function createMainMenu() {
  return {
    type: 'flex',
    altText: 'เมนูหลัก',
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [{
          type: 'text',
          text: '🛍️ ร้านค้าออนไลน์',
          weight: 'bold',
          size: 'lg',
          color: '#FFFFFF'
        }],
        backgroundColor: '#06C755'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          {
            type: 'box',
            layout: 'horizontal',
            contents: [
              createMenuButton('🛒 สินค้า', 'action=catalog&page=1'),
              createMenuButton('🔍 ค้นหา', 'action=search')
            ]
          },
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'sm',
            contents: [
              createMenuButton('🛍️ ตะกร้า', 'action=view_cart'),
              createMenuButton('📦 ออเดอร์', 'action=my_orders')
            ]
          },
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'sm',
            contents: [
              createMenuButton('⭐ คะแนน', 'action=my_points'),
              createMenuButton('👤 โปรไฟล์', 'action=my_profile')
            ]
          }
        ]
      }
    }
  };
}

function createMenuButton(label, postbackData) {
  return {
    type: 'button',
    action: {
      type: 'postback',
      label,
      data: postbackData
    },
    style: 'secondary',
    flex: 1,
    margin: 'xs'
  };
}

function createOrderSummaryBubble(order) {
  return {
    type: 'bubble',
    body: {
      type: 'box',
      layout: 'vertical',
      contents: [
        { type: 'text', text: '✅ สร้างออเดอร์แล้ว!', weight: 'bold', size: 'lg', color: '#27AE60' },
        { type: 'text', text: `หมายเลข: ${order.order_number}`, size: 'sm', margin: 'md' },
        { type: 'text', text: `ยอดรวม: ฿${parseFloat(order.total_amount).toLocaleString('th-TH')}`, size: 'md', weight: 'bold', color: '#E53E3E', margin: 'sm' }
      ]
    },
    footer: {
      type: 'box',
      layout: 'vertical',
      contents: [{
        type: 'button',
        action: { type: 'postback', label: '💳 ชำระเงิน LINE Pay', data: `action=confirm_order&order_id=${order.id}` },
        style: 'primary',
        color: '#06C755'
      }]
    }
  };
}

module.exports = router;
```

## 13. LINE Pay Callback Route

```javascript
// src/routes/linepay.js
const express = require('express');
const paymentService = require('../services/paymentService');
const orderService = require('../services/orderService');
const line = require('@line/bot-sdk');
const db = require('../config/database');

const router = express.Router();
const client = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

// Callback หลังชำระเงินสำเร็จ
router.get('/confirm', async (req, res) => {
  const { transactionId, orderId } = req.query;

  try {
    await paymentService.confirmPayment(transactionId, orderId);
    const order = await orderService.getOrder(orderId);

    // ส่งข้อความยืนยัน
    await client.pushMessage(order.line_user_id, [
      {
        type: 'text',
        text: `🎉 ชำระเงินสำเร็จ!\n\nออเดอร์ ${order.order_number}\nยอด ฿${parseFloat(order.total_amount).toLocaleString('th-TH')}\n\nเราจะดำเนินการจัดส่งสินค้าให้โดยเร็ว`
      }
    ]);

    res.redirect(`${process.env.LIFF_URL}/order-complete?orderId=${orderId}`);
  } catch (error) {
    console.error('LINE Pay confirm error:', error);
    res.redirect(`${process.env.LIFF_URL}/order-failed?orderId=${orderId}`);
  }
});

// ยกเลิกการชำระเงิน
router.get('/cancel', async (req, res) => {
  const { orderId } = req.query;

  try {
    const order = await orderService.getOrder(orderId);
    await client.pushMessage(order.line_user_id, {
      type: 'text',
      text: `❌ ยกเลิกการชำระเงิน\nออเดอร์ ${order.order_number} ยังรอชำระอยู่\n\nพิมพ์ "ออเดอร์" เพื่อดำเนินการชำระเงินใหม่`
    });
  } catch (error) {
    console.error('LINE Pay cancel error:', error);
  }

  res.redirect(`${process.env.LIFF_URL}/cart`);
});

module.exports = router;
```

## 14. Admin Panel

```javascript
// src/routes/admin.js
const express = require('express');
const db = require('../config/database');
const productService = require('../services/productService');
const orderService = require('../services/orderService');
const redis = require('../config/redis');

const router = express.Router();

// Middleware ตรวจสอบ admin
function adminAuth(req, res, next) {
  const token = req.headers['x-admin-token'] || req.query.token;
  if (token !== process.env.ADMIN_TOKEN) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
}

router.use(adminAuth);

// Dashboard stats
router.get('/stats', async (req, res) => {
  try {
    const [[todayOrders]] = await db.query(
      `SELECT COUNT(*) as count, COALESCE(SUM(total_amount), 0) as revenue
       FROM orders WHERE DATE(created_at) = CURDATE() AND payment_status = 'paid'`
    );

    const [[totalUsers]] = await db.query('SELECT COUNT(*) as count FROM users');

    const [[lowStock]] = await db.query(
      'SELECT COUNT(*) as count FROM products WHERE stock_quantity <= low_stock_threshold AND is_active = 1'
    );

    const [[pendingOrders]] = await db.query(
      "SELECT COUNT(*) as count FROM orders WHERE status IN ('pending', 'confirmed')"
    );

    res.json({
      today: {
        orders: todayOrders.count,
        revenue: todayOrders.revenue
      },
      totalUsers: totalUsers.count,
      lowStockProducts: lowStock.count,
      pendingOrders: pendingOrders.count
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// จัดการสินค้า
router.get('/products', async (req, res) => {
  const page = parseInt(req.query.page || '1');
  const limit = parseInt(req.query.limit || '20');
  const offset = (page - 1) * limit;

  const [products] = await db.query(
    'SELECT * FROM products ORDER BY created_at DESC LIMIT ? OFFSET ?',
    [limit, offset]
  );
  const [[{ total }]] = await db.query('SELECT COUNT(*) as total FROM products');

  res.json({ products, total, page, limit });
});

router.post('/products', async (req, res) => {
  const { category_id, sku, name, description, price, sale_price, stock_quantity, images } = req.body;

  try {
    const [result] = await db.query(
      `INSERT INTO products (category_id, sku, name, description, price, sale_price, stock_quantity, images)
       VALUES (?, ?, ?, ?, ?, ?, ?, ?)`,
      [category_id, sku, name, description, price, sale_price, stock_quantity, JSON.stringify(images || [])]
    );

    await redis.del('featured:6');
    res.json({ id: result.insertId, message: 'Product created' });
  } catch (error) {
    res.status(400).json({ error: error.message });
  }
});

router.put('/products/:id', async (req, res) => {
  const { id } = req.params;
  const updates = req.body;

  const allowed = ['name', 'description', 'price', 'sale_price', 'stock_quantity', 'is_active', 'is_featured'];
  const fields = Object.keys(updates).filter(k => allowed.includes(k));
  if (fields.length === 0) return res.status(400).json({ error: 'No valid fields to update' });

  const setClause = fields.map(f => `${f} = ?`).join(', ');
  const values = fields.map(f => updates[f]);
  values.push(id);

  await db.query(`UPDATE products SET ${setClause} WHERE id = ?`, values);
  await redis.del(`product:${id}`);

  res.json({ message: 'Product updated' });
});

// จัดการออเดอร์
router.get('/orders', async (req, res) => {
  const page = parseInt(req.query.page || '1');
  const limit = parseInt(req.query.limit || '20');
  const offset = (page - 1) * limit;
  const status = req.query.status;

  let whereClause = '';
  let params = [];

  if (status) {
    whereClause = 'WHERE o.status = ?';
    params.push(status);
  }

  const [orders] = await db.query(
    `SELECT o.*, u.display_name, u.line_user_id
     FROM orders o JOIN users u ON o.user_id = u.id
     ${whereClause}
     ORDER BY o.created_at DESC LIMIT ? OFFSET ?`,
    [...params, limit, offset]
  );

  res.json({ orders });
});

router.put('/orders/:id/status', async (req, res) => {
  const { id } = req.params;
  const { status, comment, trackingNumber } = req.body;

  const validStatuses = ['confirmed', 'processing', 'shipped', 'delivered', 'cancelled', 'refunded'];
  if (!validStatuses.includes(status)) {
    return res.status(400).json({ error: 'Invalid status' });
  }

  const connection = await db.getConnection();
  try {
    await connection.beginTransaction();

    let updateFields = 'status = ?';
    let values = [status];

    if (status === 'shipped' && trackingNumber) {
      updateFields += ', tracking_number = ?, shipped_at = NOW()';
      values.push(trackingNumber);
    } else if (status === 'delivered') {
      updateFields += ', delivered_at = NOW()';
    }

    values.push(id);
    await connection.query(`UPDATE orders SET ${updateFields} WHERE id = ?`, values);

    await connection.query(
      `INSERT INTO order_status_history (order_id, status, comment, created_by)
       VALUES (?, ?, ?, 'admin')`,
      [id, status, comment || '']
    );

    await connection.commit();

    // แจ้งเตือนผู้ใช้
    const [[order]] = await db.query(
      'SELECT o.*, u.line_user_id FROM orders o JOIN users u ON o.user_id = u.id WHERE o.id = ?',
      [id]
    );

    const statusMessages = {
      processing: `🔧 ออเดอร์ ${order.order_number} กำลังดำเนินการ`,
      shipped: `🚚 ออเดอร์ ${order.order_number} จัดส่งแล้ว!\nเลขพัสดุ: ${trackingNumber || 'N/A'}`,
      delivered: `✅ ออเดอร์ ${order.order_number} ส่งถึงแล้ว!\nขอบคุณที่อุดหนุนนะคะ 😊`
    };

    if (statusMessages[status]) {
      const line = require('@line/bot-sdk');
      const lineClient = new line.Client({
        channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
      });
      await lineClient.pushMessage(order.line_user_id, {
        type: 'text',
        text: statusMessages[status]
      });
    }

    res.json({ message: 'Order status updated' });
  } catch (error) {
    await connection.rollback();
    res.status(500).json({ error: error.message });
  } finally {
    connection.release();
  }
});

// รายงานยอดขาย
router.get('/reports/sales', async (req, res) => {
  const { start, end, groupBy = 'day' } = req.query;

  const dateFormat = groupBy === 'month' ? '%Y-%m' : '%Y-%m-%d';

  const [sales] = await db.query(
    `SELECT DATE_FORMAT(created_at, ?) as period,
            COUNT(*) as orders,
            COALESCE(SUM(total_amount), 0) as revenue,
            AVG(total_amount) as avg_order_value
     FROM orders
     WHERE payment_status = 'paid'
       AND created_at BETWEEN ? AND ?
     GROUP BY period
     ORDER BY period`,
    [dateFormat, start || '2024-01-01', end || new Date().toISOString().split('T')[0]]
  );

  res.json({ sales });
});

module.exports = router;
```

## 15. Return/Refund System

```javascript
// src/services/returnService.js
const db = require('../config/database');
const paymentService = require('./paymentService');
const loyaltyService = require('./loyaltyService');

class ReturnService {
  // ขอคืนสินค้า
  async requestReturn(userId, orderId, reason, description, images = []) {
    const [[order]] = await db.query(
      'SELECT * FROM orders WHERE id = ? AND user_id = ?',
      [orderId, userId]
    );

    if (!order) throw new Error('Order not found');
    if (order.status !== 'delivered') {
      throw new Error('Can only return delivered orders');
    }

    // ตรวจสอบว่าขอคืนไปแล้วหรือยัง
    const [[existing]] = await db.query(
      "SELECT id FROM returns WHERE order_id = ? AND status NOT IN ('rejected')",
      [orderId]
    );
    if (existing) throw new Error('Return request already exists');

    const [result] = await db.query(
      `INSERT INTO returns (order_id, user_id, reason, description, images, refund_amount)
       VALUES (?, ?, ?, ?, ?, ?)`,
      [orderId, userId, reason, description, JSON.stringify(images), order.total_amount]
    );

    return result.insertId;
  }

  // อนุมัติการคืนสินค้า (admin)
  async approveReturn(returnId, refundAmount, refundMethod, notes) {
    const [[ret]] = await db.query(
      'SELECT r.*, o.payment_transaction_id FROM returns r JOIN orders o ON r.order_id = o.id WHERE r.id = ?',
      [returnId]
    );

    if (!ret) throw new Error('Return not found');

    await db.query(
      `UPDATE returns SET status = 'approved', refund_amount = ?, refund_method = ?, admin_notes = ?
       WHERE id = ?`,
      [refundAmount, refundMethod, notes, returnId]
    );

    // ถ้าคืนผ่าน LINE Pay
    if (refundMethod === 'linepay' && ret.payment_transaction_id) {
      try {
        await paymentService.refundPayment(ret.payment_transaction_id, refundAmount, 'Customer return');

        await db.query(
          "UPDATE returns SET status = 'refunded' WHERE id = ?",
          [returnId]
        );

        await db.query(
          "UPDATE orders SET status = 'refunded', payment_status = 'refunded' WHERE id = ?",
          [ret.order_id]
        );
      } catch (error) {
        throw new Error(`Refund failed: ${error.message}`);
      }
    }

    return ret;
  }

  // ปฏิเสธการคืนสินค้า
  async rejectReturn(returnId, reason) {
    await db.query(
      "UPDATE returns SET status = 'rejected', admin_notes = ? WHERE id = ?",
      [reason, returnId]
    );
  }
}

module.exports = new ReturnService();
```

## 16. Order Number Generator

```javascript
// src/utils/orderNumber.js
function generateOrderNumber() {
  const now = new Date();
  const year = now.getFullYear().toString().slice(-2);
  const month = String(now.getMonth() + 1).padStart(2, '0');
  const day = String(now.getDate()).padStart(2, '0');
  const random = Math.floor(Math.random() * 100000).toString().padStart(5, '0');
  return `ORD${year}${month}${day}${random}`;
}

module.exports = { generateOrderNumber };
```

## 17. index.js หลัก

```javascript
// index.js
require('dotenv').config();
const express = require('express');
const webhookRouter = require('./src/routes/webhook');
const linepayRouter = require('./src/routes/linepay');
const adminRouter = require('./src/routes/admin');

const app = express();
app.use(express.json());

app.use('/webhook', webhookRouter);
app.use('/linepay', linepayRouter);
app.use('/admin', adminRouter);

app.get('/health', (req, res) => res.json({ status: 'ok', time: new Date() }));

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => {
  console.log(`E-Commerce Bot running on port ${PORT}`);
});
```

## 18. package.json

```json
{
  "name": "lineoa-ecommerce",
  "version": "1.0.0",
  "scripts": {
    "start": "node index.js",
    "dev": "nodemon index.js",
    "test": "jest"
  },
  "dependencies": {
    "@line/bot-sdk": "^8.0.0",
    "axios": "^1.6.0",
    "dotenv": "^16.0.0",
    "express": "^4.18.0",
    "ioredis": "^5.3.0",
    "mysql2": "^3.6.0"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "nodemon": "^3.0.0",
    "supertest": "^6.3.0"
  }
}
```

## 19. การทดสอบ

```javascript
// tests/cart.test.js
const cartService = require('../src/services/cartService');
const productService = require('../src/services/productService');

describe('Cart Service', () => {
  let testUserId;
  let testProductId;

  beforeAll(async () => {
    // สร้างข้อมูลทดสอบ
    const db = require('../src/config/database');
    const [userResult] = await db.query(
      "INSERT INTO users (line_user_id, display_name) VALUES ('test_user_001', 'Test User')"
    );
    testUserId = userResult.insertId;

    const [productResult] = await db.query(
      `INSERT INTO products (sku, name, price, stock_quantity)
       VALUES ('TEST001', 'Test Product', 100.00, 50)`
    );
    testProductId = productResult.insertId;
  });

  test('เพิ่มสินค้าลงตะกร้า', async () => {
    const cart = await cartService.addToCart(testUserId, testProductId, null, 2);
    expect(cart.items).toHaveLength(1);
    expect(cart.items[0].quantity).toBe(2);
    expect(cart.subtotal).toBe(200);
  });

  test('เพิ่มสินค้าเดิมในตะกร้า - จะรวมจำนวน', async () => {
    const cart = await cartService.addToCart(testUserId, testProductId, null, 3);
    expect(cart.items[0].quantity).toBe(5);
  });

  test('ลบสินค้าออกจากตะกร้า', async () => {
    const cart = await cartService.getCart(testUserId);
    const updatedCart = await cartService.removeFromCart(testUserId, cart.items[0].id);
    expect(updatedCart.items).toHaveLength(0);
  });

  afterAll(async () => {
    const db = require('../src/config/database');
    await db.query('DELETE FROM users WHERE id = ?', [testUserId]);
    await db.query('DELETE FROM products WHERE id = ?', [testProductId]);
    await db.end();
  });
});
```

## 20. Performance Optimization

```javascript
// src/middleware/cacheMiddleware.js
const redis = require('../config/redis');

function cacheResponse(ttl = 300) {
  return async (req, res, next) => {
    const key = `cache:${req.method}:${req.originalUrl}`;
    
    try {
      const cached = await redis.get(key);
      if (cached) {
        res.setHeader('X-Cache', 'HIT');
        return res.json(JSON.parse(cached));
      }
    } catch (err) {
      console.error('Cache read error:', err);
    }

    res.setHeader('X-Cache', 'MISS');
    const originalJson = res.json.bind(res);
    res.json = async (data) => {
      try {
        await redis.setex(key, ttl, JSON.stringify(data));
      } catch (err) {
        console.error('Cache write error:', err);
      }
      return originalJson(data);
    };

    next();
  };
}

module.exports = { cacheResponse };
```

```javascript
// Database Query Optimization - Indexes
// เพิ่ม Composite Indexes ที่ใช้บ่อย
/*
ALTER TABLE orders ADD INDEX idx_user_status (user_id, status);
ALTER TABLE order_items ADD INDEX idx_order_product (order_id, product_id);
ALTER TABLE loyalty_transactions ADD INDEX idx_user_date (user_id, created_at);
ALTER TABLE carts ADD INDEX idx_user_updated (user_id, updated_at);
*/
```

## สรุป

ในบทนี้เราได้สร้างระบบ E-Commerce บน LINE OA ที่ครอบคลุม:

1. **Database Schema** - ออกแบบฐานข้อมูลครบถ้วน
2. **Product Catalog** - แสดงสินค้าด้วย Flex Message Carousel
3. **Shopping Cart** - ระบบตะกร้าสินค้า
4. **Order Management** - จัดการออเดอร์พร้อม status tracking
5. **LINE Pay** - การชำระเงินผ่าน LINE Pay
6. **Loyalty Points** - ระบบแต้มสะสมพร้อม tier
7. **Inventory** - ติดตามสต็อก
8. **Admin Panel** - API สำหรับจัดการระบบ
9. **Return/Refund** - ขั้นตอนคืนสินค้า/เงิน
10. **Performance** - Caching ด้วย Redis
