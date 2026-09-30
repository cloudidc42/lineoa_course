# Part 68: ระบบสั่งอาหารบน LINE OA

## บทนำ

ในบทนี้เราจะสร้างระบบสั่งอาหาร Food Delivery แบบครบวงจรบน LINE OA รองรับหลายร้านค้า ระบบ delivery พร้อม tracking และ Kitchen Display System

## สิ่งที่จะได้เรียนรู้

- Multi-restaurant support
- Menu management พร้อม customization
- Order flow ครบขั้นตอน
- Delivery zone management
- Real-time order tracking
- Kitchen Display System (KDS)
- Driver management
- Payment integration
- Rating & reviews

---

## 1. Database Schema

```sql
CREATE DATABASE IF NOT EXISTS lineoa_food CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE lineoa_food;

-- ลูกค้า
CREATE TABLE customers (
  id INT PRIMARY KEY AUTO_INCREMENT,
  line_user_id VARCHAR(100) UNIQUE NOT NULL,
  display_name VARCHAR(200),
  picture_url TEXT,
  phone VARCHAR(20),
  email VARCHAR(200),
  default_address JSON,
  saved_addresses JSON DEFAULT '[]',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_line_user_id (line_user_id)
);

-- ร้านอาหาร
CREATE TABLE restaurants (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  description TEXT,
  logo_url TEXT,
  cover_url TEXT,
  address TEXT,
  lat DECIMAL(10,8),
  lng DECIMAL(11,8),
  phone VARCHAR(20),
  opening_hours JSON,
  min_order_amount DECIMAL(10,2) DEFAULT 0,
  delivery_fee DECIMAL(10,2) DEFAULT 0,
  free_delivery_above DECIMAL(10,2),
  prep_time_minutes INT DEFAULT 20,
  rating DECIMAL(3,2) DEFAULT 0,
  review_count INT DEFAULT 0,
  is_active TINYINT(1) DEFAULT 1,
  is_open TINYINT(1) DEFAULT 1,
  tags JSON,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_is_active (is_active)
);

-- หมวดหมู่อาหาร
CREATE TABLE menu_categories (
  id INT PRIMARY KEY AUTO_INCREMENT,
  restaurant_id INT NOT NULL,
  name VARCHAR(200) NOT NULL,
  description TEXT,
  sort_order INT DEFAULT 0,
  is_active TINYINT(1) DEFAULT 1,
  FOREIGN KEY (restaurant_id) REFERENCES restaurants(id) ON DELETE CASCADE
);

-- เมนูอาหาร
CREATE TABLE menu_items (
  id INT PRIMARY KEY AUTO_INCREMENT,
  restaurant_id INT NOT NULL,
  category_id INT,
  name VARCHAR(300) NOT NULL,
  description TEXT,
  price DECIMAL(10,2) NOT NULL,
  image_url TEXT,
  options JSON,
  addons JSON,
  is_available TINYINT(1) DEFAULT 1,
  is_popular TINYINT(1) DEFAULT 0,
  sold_count INT DEFAULT 0,
  prep_time_minutes INT DEFAULT 10,
  calories INT,
  tags JSON,
  sort_order INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (restaurant_id) REFERENCES restaurants(id) ON DELETE CASCADE,
  FOREIGN KEY (category_id) REFERENCES menu_categories(id) ON DELETE SET NULL,
  INDEX idx_restaurant_id (restaurant_id),
  INDEX idx_is_available (is_available)
);

-- ออเดอร์
CREATE TABLE orders (
  id INT PRIMARY KEY AUTO_INCREMENT,
  order_number VARCHAR(20) UNIQUE NOT NULL,
  customer_id INT NOT NULL,
  restaurant_id INT NOT NULL,
  driver_id INT,
  status ENUM('pending','confirmed','preparing','ready','picked_up','delivering','delivered','cancelled') DEFAULT 'pending',
  delivery_address JSON NOT NULL,
  subtotal DECIMAL(12,2) NOT NULL,
  delivery_fee DECIMAL(10,2) DEFAULT 0,
  discount DECIMAL(10,2) DEFAULT 0,
  total_amount DECIMAL(12,2) NOT NULL,
  payment_method VARCHAR(50),
  payment_status ENUM('pending','paid','failed') DEFAULT 'pending',
  payment_transaction_id VARCHAR(200),
  special_instructions TEXT,
  estimated_delivery_time TIMESTAMP NULL,
  actual_delivery_time TIMESTAMP NULL,
  driver_lat DECIMAL(10,8),
  driver_lng DECIMAL(11,8),
  rating INT,
  review TEXT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (customer_id) REFERENCES customers(id),
  FOREIGN KEY (restaurant_id) REFERENCES restaurants(id),
  INDEX idx_status (status),
  INDEX idx_customer_id (customer_id),
  INDEX idx_restaurant_id (restaurant_id)
);

-- รายการอาหารในออเดอร์
CREATE TABLE order_items (
  id INT PRIMARY KEY AUTO_INCREMENT,
  order_id INT NOT NULL,
  menu_item_id INT NOT NULL,
  name VARCHAR(300) NOT NULL,
  price DECIMAL(10,2) NOT NULL,
  quantity INT NOT NULL DEFAULT 1,
  options JSON,
  addons JSON,
  special_instructions TEXT,
  total_price DECIMAL(12,2) NOT NULL,
  FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
  FOREIGN KEY (menu_item_id) REFERENCES menu_items(id),
  INDEX idx_order_id (order_id)
);

-- คนขับรถ
CREATE TABLE drivers (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  phone VARCHAR(20) NOT NULL,
  vehicle_type VARCHAR(50),
  vehicle_plate VARCHAR(20),
  profile_image TEXT,
  current_lat DECIMAL(10,8),
  current_lng DECIMAL(11,8),
  status ENUM('offline','available','busy') DEFAULT 'offline',
  rating DECIMAL(3,2) DEFAULT 5.0,
  total_deliveries INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- โซนจัดส่ง
CREATE TABLE delivery_zones (
  id INT PRIMARY KEY AUTO_INCREMENT,
  restaurant_id INT NOT NULL,
  name VARCHAR(200) NOT NULL,
  area_polygon JSON,
  delivery_fee DECIMAL(10,2) NOT NULL,
  min_order DECIMAL(10,2) DEFAULT 0,
  max_distance_km DECIMAL(5,2),
  estimated_minutes INT DEFAULT 30,
  is_active TINYINT(1) DEFAULT 1,
  FOREIGN KEY (restaurant_id) REFERENCES restaurants(id) ON DELETE CASCADE
);

-- รีวิว
CREATE TABLE reviews (
  id INT PRIMARY KEY AUTO_INCREMENT,
  order_id INT NOT NULL UNIQUE,
  customer_id INT NOT NULL,
  restaurant_id INT NOT NULL,
  driver_id INT,
  food_rating INT NOT NULL CHECK (food_rating BETWEEN 1 AND 5),
  delivery_rating INT CHECK (delivery_rating BETWEEN 1 AND 5),
  comment TEXT,
  images JSON,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (order_id) REFERENCES orders(id),
  FOREIGN KEY (customer_id) REFERENCES customers(id),
  FOREIGN KEY (restaurant_id) REFERENCES restaurants(id)
);

-- ข้อมูลตัวอย่าง
INSERT INTO restaurants (name, description, address, prep_time_minutes, delivery_fee, rating) VALUES
('ร้านข้าวมันไก่เจ้าดัง', 'ข้าวมันไก่รสชาติต้นตำรับ', '123 ถนนสุขุมวิท กรุงเทพ', 15, 30, 4.8),
('ก๋วยเตี๋ยวเรือแม่น้ำ', 'ก๋วยเตี๋ยวเรือแท้ๆ', '456 ถนนพหลโยธิน กรุงเทพ', 10, 25, 4.6);

INSERT INTO menu_categories (restaurant_id, name) VALUES
(1, 'เมนูยอดนิยม'), (1, 'อาหารหลัก'), (1, 'เครื่องดื่ม'),
(2, 'ก๋วยเตี๋ยว'), (2, 'ข้าวหน้า'), (2, 'เครื่องดื่ม');

INSERT INTO menu_items (restaurant_id, category_id, name, price, is_popular, options) VALUES
(1, 1, 'ข้าวมันไก่ต้ม', 60, 1, '[{"name":"ขนาด","required":true,"choices":[{"label":"ธรรมดา","price":0},{"label":"พิเศษ","price":10}]}]'),
(1, 1, 'ข้าวมันไก่ทอด', 70, 1, NULL),
(1, 3, 'น้ำเต้าหู้เย็น', 25, 0, NULL),
(2, 4, 'ก๋วยเตี๋ยวน้ำ', 55, 1, '[{"name":"เส้น","required":true,"choices":[{"label":"เส้นใหญ่","price":0},{"label":"เส้นเล็ก","price":0},{"label":"วุ้นเส้น","price":0}]}]'),
(2, 4, 'ก๋วยเตี๋ยวแห้ง', 55, 0, NULL);
```

## 2. โครงสร้างโปรเจค

```
lineoa-food/
├── src/
│   ├── config/
│   │   └── database.js
│   ├── services/
│   │   ├── restaurantService.js
│   │   ├── orderService.js
│   │   ├── cartService.js
│   │   └── deliveryService.js
│   ├── messages/
│   │   ├── restaurantMessages.js
│   │   ├── menuMessages.js
│   │   └── orderMessages.js
│   ├── handlers/
│   │   ├── postbackHandler.js
│   │   └── messageHandler.js
│   └── routes/
│       ├── webhook.js
│       ├── kds.js
│       └── admin.js
├── socket/
│   └── trackingServer.js
└── index.js
```

## 3. Restaurant Service

```javascript
// src/services/restaurantService.js
const db = require('../config/database');

class RestaurantService {
  // ดึงร้านอาหารที่เปิดอยู่
  async getOpenRestaurants(page = 1, limit = 8) {
    const offset = (page - 1) * limit;
    const now = new Date();
    const dayOfWeek = now.toLocaleDateString('en-US', { weekday: 'long' }).toLowerCase();
    const currentTime = now.toTimeString().slice(0, 5);

    const [restaurants] = await db.query(
      `SELECT * FROM restaurants WHERE is_active = 1 AND is_open = 1
       ORDER BY rating DESC LIMIT ? OFFSET ?`,
      [limit, offset]
    );

    // กรองร้านที่เปิดตามเวลา
    const openRestaurants = restaurants.filter(r => {
      if (!r.opening_hours) return true;
      const hours = JSON.parse(r.opening_hours);
      const todayHours = hours[dayOfWeek];
      if (!todayHours) return false;
      return currentTime >= todayHours.open && currentTime <= todayHours.close;
    });

    return openRestaurants;
  }

  // ค้นหาร้านอาหาร
  async searchRestaurants(keyword) {
    const searchTerm = `%${keyword}%`;
    const [restaurants] = await db.query(
      `SELECT * FROM restaurants 
       WHERE is_active = 1 AND (name LIKE ? OR description LIKE ?)
       ORDER BY rating DESC LIMIT 10`,
      [searchTerm, searchTerm]
    );
    return restaurants;
  }

  // ดึงเมนูของร้าน
  async getRestaurantMenu(restaurantId) {
    const [[restaurant]] = await db.query(
      'SELECT * FROM restaurants WHERE id = ? AND is_active = 1',
      [restaurantId]
    );
    if (!restaurant) return null;

    const [categories] = await db.query(
      `SELECT c.*, GROUP_CONCAT(mi.id) as item_ids
       FROM menu_categories c
       LEFT JOIN menu_items mi ON c.id = mi.category_id AND mi.is_available = 1
       WHERE c.restaurant_id = ? AND c.is_active = 1
       GROUP BY c.id
       ORDER BY c.sort_order`,
      [restaurantId]
    );

    const [items] = await db.query(
      `SELECT * FROM menu_items 
       WHERE restaurant_id = ? AND is_available = 1
       ORDER BY category_id, is_popular DESC, sort_order`,
      [restaurantId]
    );

    restaurant.categories = categories.map(cat => ({
      ...cat,
      items: items.filter(item => item.category_id === cat.id)
    }));

    return restaurant;
  }

  // ดึงเมนูยอดนิยม
  async getPopularItems(restaurantId) {
    const [items] = await db.query(
      `SELECT * FROM menu_items 
       WHERE restaurant_id = ? AND is_popular = 1 AND is_available = 1
       ORDER BY sold_count DESC LIMIT 6`,
      [restaurantId]
    );
    return items;
  }
}

module.exports = new RestaurantService();
```

## 4. Cart Service (Redis-based)

```javascript
// src/services/cartService.js
const redis = require('../config/redis');
const db = require('../config/database');

const CART_TTL = 3600; // 1 ชั่วโมง

class CartService {
  getCartKey(userId) {
    return `cart:${userId}`;
  }

  // ดึงตะกร้า
  async getCart(userId) {
    const key = this.getCartKey(userId);
    const data = await redis.get(key);
    if (!data) return { restaurantId: null, items: [] };
    return JSON.parse(data);
  }

  // เพิ่มรายการ
  async addItem(userId, restaurantId, menuItemId, quantity, options, addons, specialInstructions) {
    const [[menuItem]] = await db.query(
      'SELECT * FROM menu_items WHERE id = ? AND is_available = 1',
      [menuItemId]
    );
    if (!menuItem) throw new Error('Menu item not found or unavailable');

    const cart = await this.getCart(userId);

    // ถ้าตะกร้ามีสินค้าจากร้านอื่น ถามยืนยัน
    if (cart.restaurantId && cart.restaurantId !== restaurantId && cart.items.length > 0) {
      return { needsClear: true, currentRestaurantId: cart.restaurantId };
    }

    // คำนวณราคา
    let itemPrice = parseFloat(menuItem.price);
    const selectedOptions = options || {};
    const selectedAddons = addons || [];

    if (menuItem.options) {
      const optionDefs = JSON.parse(menuItem.options);
      for (const optDef of optionDefs) {
        if (selectedOptions[optDef.name]) {
          const choice = optDef.choices.find(c => c.label === selectedOptions[optDef.name]);
          if (choice) itemPrice += parseFloat(choice.price || 0);
        }
      }
    }

    if (menuItem.addons) {
      const addonDefs = JSON.parse(menuItem.addons);
      for (const addonId of selectedAddons) {
        const addon = addonDefs.find(a => a.id === addonId);
        if (addon) itemPrice += parseFloat(addon.price || 0);
      }
    }

    const cartItemId = `${menuItemId}_${Date.now()}`;
    const newItem = {
      id: cartItemId,
      menuItemId,
      name: menuItem.name,
      imageUrl: menuItem.image_url,
      unitPrice: itemPrice,
      quantity,
      options: selectedOptions,
      addons: selectedAddons,
      specialInstructions: specialInstructions || '',
      total: itemPrice * quantity
    };

    cart.restaurantId = restaurantId;
    cart.items.push(newItem);

    await redis.setex(this.getCartKey(userId), CART_TTL, JSON.stringify(cart));
    return { cart };
  }

  // อัปเดตจำนวน
  async updateQuantity(userId, cartItemId, quantity) {
    const cart = await this.getCart(userId);
    const item = cart.items.find(i => i.id === cartItemId);
    if (!item) throw new Error('Item not found in cart');

    if (quantity <= 0) {
      cart.items = cart.items.filter(i => i.id !== cartItemId);
    } else {
      item.quantity = quantity;
      item.total = item.unitPrice * quantity;
    }

    if (cart.items.length === 0) cart.restaurantId = null;

    await redis.setex(this.getCartKey(userId), CART_TTL, JSON.stringify(cart));
    return cart;
  }

  // ล้างตะกร้า
  async clearCart(userId) {
    await redis.del(this.getCartKey(userId));
  }

  // คำนวณยอดรวม
  getCartSummary(cart) {
    const subtotal = cart.items.reduce((sum, item) => sum + item.total, 0);
    return { subtotal, itemCount: cart.items.reduce((sum, i) => sum + i.quantity, 0) };
  }
}

module.exports = new CartService();
```

## 5. Order Service

```javascript
// src/services/orderService.js
const db = require('../config/database');
const cartService = require('./cartService');

function generateOrderNumber() {
  const ts = Date.now().toString().slice(-8);
  return `FD${ts}`;
}

class OrderService {
  async createOrder(userId, deliveryAddress, specialInstructions = '') {
    const [[customer]] = await db.query('SELECT * FROM customers WHERE line_user_id = ?', [userId]);
    if (!customer) throw new Error('Customer not found');

    const cart = await cartService.getCart(userId);
    if (!cart.items || cart.items.length === 0) throw new Error('Cart is empty');

    const [[restaurant]] = await db.query(
      'SELECT * FROM restaurants WHERE id = ? AND is_active = 1 AND is_open = 1',
      [cart.restaurantId]
    );
    if (!restaurant) throw new Error('Restaurant is not available');

    const summary = cartService.getCartSummary(cart);
    const deliveryFee = parseFloat(restaurant.delivery_fee);
    const total = summary.subtotal + deliveryFee;

    if (summary.subtotal < parseFloat(restaurant.min_order_amount)) {
      throw new Error(`ยอดขั้นต่ำ ฿${restaurant.min_order_amount}`);
    }

    const connection = await db.getConnection();
    try {
      await connection.beginTransaction();

      const orderNumber = generateOrderNumber();

      const [orderResult] = await connection.query(
        `INSERT INTO orders 
         (order_number, customer_id, restaurant_id, delivery_address, subtotal, delivery_fee, total_amount, special_instructions)
         VALUES (?, ?, ?, ?, ?, ?, ?, ?)`,
        [orderNumber, customer.id, cart.restaurantId, JSON.stringify(deliveryAddress),
         summary.subtotal, deliveryFee, total, specialInstructions]
      );
      const orderId = orderResult.insertId;

      for (const item of cart.items) {
        await connection.query(
          `INSERT INTO order_items (order_id, menu_item_id, name, price, quantity, options, addons, special_instructions, total_price)
           VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)`,
          [orderId, item.menuItemId, item.name, item.unitPrice, item.quantity,
           JSON.stringify(item.options), JSON.stringify(item.addons),
           item.specialInstructions, item.total]
        );
      }

      await connection.commit();

      // ล้างตะกร้า
      await cartService.clearCart(userId);

      return this.getOrder(orderId);
    } catch (err) {
      await connection.rollback();
      throw err;
    } finally {
      connection.release();
    }
  }

  async getOrder(orderId) {
    const [[order]] = await db.query(
      `SELECT o.*, r.name as restaurant_name, r.logo_url, r.prep_time_minutes,
              c.display_name as customer_name, c.phone as customer_phone, c.line_user_id,
              d.name as driver_name, d.phone as driver_phone, d.vehicle_plate
       FROM orders o
       JOIN restaurants r ON o.restaurant_id = r.id
       JOIN customers c ON o.customer_id = c.id
       LEFT JOIN drivers d ON o.driver_id = d.id
       WHERE o.id = ?`,
      [orderId]
    );
    if (!order) return null;

    const [items] = await db.query('SELECT * FROM order_items WHERE order_id = ?', [orderId]);
    order.items = items;
    return order;
  }

  async getOrderByNumber(orderNumber) {
    const [[order]] = await db.query('SELECT id FROM orders WHERE order_number = ?', [orderNumber]);
    if (!order) return null;
    return this.getOrder(order.id);
  }

  async getUserOrders(userId, limit = 10) {
    const [[customer]] = await db.query('SELECT id FROM customers WHERE line_user_id = ?', [userId]);
    if (!customer) return [];

    const [orders] = await db.query(
      `SELECT o.*, r.name as restaurant_name, r.logo_url,
              COUNT(oi.id) as item_count
       FROM orders o
       JOIN restaurants r ON o.restaurant_id = r.id
       LEFT JOIN order_items oi ON o.id = oi.order_id
       WHERE o.customer_id = ?
       GROUP BY o.id
       ORDER BY o.created_at DESC LIMIT ?`,
      [customer.id, limit]
    );
    return orders;
  }

  async updateStatus(orderId, status, extraData = {}) {
    const validStatuses = ['confirmed', 'preparing', 'ready', 'picked_up', 'delivering', 'delivered', 'cancelled'];
    if (!validStatuses.includes(status)) throw new Error('Invalid status');

    const updates = { status };
    if (status === 'delivered') updates.actual_delivery_time = new Date();
    if (extraData.driverId) updates.driver_id = extraData.driverId;
    if (extraData.estimatedTime) updates.estimated_delivery_time = extraData.estimatedTime;

    const setClause = Object.keys(updates).map(k => `${k} = ?`).join(', ');
    await db.query(
      `UPDATE orders SET ${setClause} WHERE id = ?`,
      [...Object.values(updates), orderId]
    );

    // แจ้งเตือนลูกค้า
    const order = await this.getOrder(orderId);
    if (order) {
      await this.notifyCustomer(order, status);
    }

    return order;
  }

  async notifyCustomer(order, status) {
    const line = require('@line/bot-sdk');
    const client = new line.Client({
      channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
    });

    const messages = {
      confirmed: `✅ ร้าน ${order.restaurant_name} รับออเดอร์ของคุณแล้ว!\nกำลังเตรียมอาหาร...`,
      preparing: `👨‍🍳 กำลังปรุงอาหารของคุณ\nออเดอร์ ${order.order_number}`,
      ready: `🎉 อาหารพร้อมแล้ว! กำลังรอคนขับมารับ`,
      picked_up: `🛵 คนขับรับอาหารแล้ว!\nกำลังนำส่ง...`,
      delivering: `🚀 ${order.driver_name || 'คนขับ'} กำลังเดินทางมาหาคุณ`,
      delivered: `🎊 ส่งอาหารแล้ว! ขอให้อร่อยนะครับ\n\nอยากให้รีวิวมั้ยคะ?`,
      cancelled: `❌ ออเดอร์ ${order.order_number} ถูกยกเลิก`
    };

    if (messages[status]) {
      try {
        const msgs = [{ type: 'text', text: messages[status] }];

        if (status === 'delivered') {
          msgs.push(createReviewPrompt(order));
        }

        await client.pushMessage(order.line_user_id, msgs);
      } catch (err) {
        console.error('Notification error:', err.message);
      }
    }
  }
}

function createReviewPrompt(order) {
  return {
    type: 'flex',
    altText: 'รีวิวอาหาร',
    contents: {
      type: 'bubble',
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          { type: 'text', text: '⭐ ให้คะแนนออเดอร์นี้', weight: 'bold', size: 'md' },
          { type: 'text', text: `จาก ${order.restaurant_name}`, size: 'sm', color: '#666666', margin: 'sm' },
          {
            type: 'box', layout: 'horizontal', margin: 'lg', spacing: 'md',
            contents: [1, 2, 3, 4, 5].map(rating => ({
              type: 'button',
              action: {
                type: 'postback',
                label: '⭐'.repeat(rating),
                data: `action=rate_order&order_id=${order.id}&rating=${rating}`
              },
              style: 'secondary',
              height: 'sm',
              flex: 1
            }))
          }
        ]
      }
    }
  };
}

module.exports = new OrderService();
```

## 6. Flex Messages

```javascript
// src/messages/restaurantMessages.js
function createRestaurantList(restaurants) {
  if (!restaurants.length) {
    return { type: 'text', text: '😔 ไม่พบร้านอาหารที่เปิดอยู่ในขณะนี้' };
  }

  const bubbles = restaurants.map(r => ({
    type: 'bubble',
    hero: {
      type: 'image',
      url: r.cover_url || r.logo_url || 'https://via.placeholder.com/600x300',
      size: 'full',
      aspectRatio: '20:13',
      aspectMode: 'cover'
    },
    body: {
      type: 'box',
      layout: 'vertical',
      contents: [
        {
          type: 'box', layout: 'horizontal',
          contents: [
            {
              type: 'image',
              url: r.logo_url || 'https://via.placeholder.com/60',
              size: 'xs',
              aspectMode: 'cover',
              flex: 0
            },
            {
              type: 'box', layout: 'vertical', margin: 'sm',
              contents: [
                { type: 'text', text: r.name, weight: 'bold', size: 'md', wrap: true },
                { type: 'text', text: r.description || '', size: 'xs', color: '#666666', maxLines: 1, wrap: true }
              ]
            }
          ]
        },
        {
          type: 'box', layout: 'horizontal', margin: 'md', spacing: 'md',
          contents: [
            { type: 'text', text: `⭐ ${r.rating}`, size: 'xs', flex: 1 },
            { type: 'text', text: `⏱ ${r.prep_time_minutes} นาที`, size: 'xs', flex: 1 },
            { type: 'text', text: `🛵 ฿${r.delivery_fee}`, size: 'xs', flex: 1 }
          ]
        },
        r.min_order_amount > 0 ? {
          type: 'text',
          text: `ขั้นต่ำ ฿${r.min_order_amount}`,
          size: 'xs',
          color: '#999999',
          margin: 'xs'
        } : null
      ].filter(Boolean)
    },
    footer: {
      type: 'box',
      layout: 'vertical',
      contents: [{
        type: 'button',
        action: {
          type: 'postback',
          label: '🍽️ ดูเมนู',
          data: `action=view_menu&restaurant_id=${r.id}`
        },
        style: 'primary',
        color: '#FF6B35'
      }]
    }
  }));

  return {
    type: 'flex',
    altText: 'ร้านอาหารที่เปิดอยู่',
    contents: { type: 'carousel', contents: bubbles }
  };
}

module.exports = { createRestaurantList };
```

```javascript
// src/messages/menuMessages.js
function createMenuCarousel(restaurant, category) {
  const items = category.items || [];
  if (!items.length) return null;

  const bubbles = items.map(item => createMenuItemCard(item, restaurant.id));

  return {
    type: 'flex',
    altText: `เมนู ${category.name}`,
    contents: { type: 'carousel', contents: bubbles }
  };
}

function createMenuItemCard(item, restaurantId) {
  const options = item.options ? JSON.parse(item.options) : [];
  const addons = item.addons ? JSON.parse(item.addons) : [];
  const hasCustomization = options.length > 0 || addons.length > 0;

  return {
    type: 'bubble',
    size: 'kilo',
    hero: item.image_url ? {
      type: 'image',
      url: item.image_url,
      size: 'full',
      aspectRatio: '20:13',
      aspectMode: 'cover'
    } : null,
    body: {
      type: 'box',
      layout: 'vertical',
      contents: [
        {
          type: 'box', layout: 'horizontal',
          contents: [
            { type: 'text', text: item.name, weight: 'bold', size: 'sm', flex: 4, wrap: true },
            item.is_popular ? { type: 'text', text: '🔥', size: 'sm', flex: 0 } : null
          ].filter(Boolean)
        },
        item.description ? {
          type: 'text', text: item.description, size: 'xs', color: '#666666',
          wrap: true, maxLines: 2, margin: 'xs'
        } : null,
        {
          type: 'text',
          text: `฿${parseFloat(item.price).toLocaleString('th-TH')}`,
          weight: 'bold',
          color: '#FF6B35',
          size: 'md',
          margin: 'sm'
        }
      ].filter(Boolean)
    },
    footer: {
      type: 'box',
      layout: 'vertical',
      contents: [{
        type: 'button',
        action: hasCustomization ? {
          type: 'postback',
          label: '➕ เพิ่มลงตะกร้า',
          data: `action=customize_item&item_id=${item.id}&restaurant_id=${restaurantId}`
        } : {
          type: 'postback',
          label: '➕ เพิ่มลงตะกร้า',
          data: `action=add_to_cart&item_id=${item.id}&restaurant_id=${restaurantId}&qty=1`
        },
        style: 'primary',
        color: '#FF6B35',
        height: 'sm'
      }]
    }
  };
}

function createCustomizationMenu(item) {
  const options = item.options ? JSON.parse(item.options) : [];
  const addons = item.addons ? JSON.parse(item.addons) : [];

  const optionContents = options.map(opt => ({
    type: 'box',
    layout: 'vertical',
    contents: [
      { type: 'text', text: `${opt.required ? '* ' : ''}${opt.name}`, weight: 'bold', size: 'sm' },
      {
        type: 'box', layout: 'horizontal', spacing: 'xs', margin: 'xs',
        contents: opt.choices.map(choice => ({
          type: 'button',
          action: {
            type: 'postback',
            label: choice.price > 0 ? `${choice.label} +฿${choice.price}` : choice.label,
            data: `action=select_option&item_id=${item.id}&option=${encodeURIComponent(opt.name)}&choice=${encodeURIComponent(choice.label)}`
          },
          style: 'secondary',
          height: 'sm',
          flex: 1
        }))
      }
    ]
  }));

  const addonContents = addons.length > 0 ? [
    { type: 'separator', margin: 'md' },
    { type: 'text', text: 'เพิ่มเติม:', weight: 'bold', size: 'sm', margin: 'md' },
    {
      type: 'box', layout: 'vertical', spacing: 'xs',
      contents: addons.map(addon => ({
        type: 'button',
        action: {
          type: 'postback',
          label: `${addon.name} +฿${addon.price}`,
          data: `action=toggle_addon&item_id=${item.id}&addon_id=${addon.id}`
        },
        style: 'secondary',
        height: 'sm'
      }))
    }
  ] : [];

  return {
    type: 'flex',
    altText: `ตั้งค่า ${item.name}`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box',
        layout: 'vertical',
        contents: [
          { type: 'text', text: item.name, weight: 'bold', color: '#FFFFFF', size: 'md' },
          { type: 'text', text: `฿${parseFloat(item.price).toLocaleString('th-TH')}`, color: '#FFFFFF', size: 'sm' }
        ],
        backgroundColor: '#FF6B35'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [...optionContents, ...addonContents]
      },
      footer: {
        type: 'box',
        layout: 'vertical',
        contents: [{
          type: 'button',
          action: {
            type: 'postback',
            label: '✅ เพิ่มลงตะกร้า',
            data: `action=add_customized&item_id=${item.id}`
          },
          style: 'primary',
          color: '#FF6B35'
        }]
      }
    }
  };
}

function createCartView(cart, restaurant) {
  if (!cart.items || cart.items.length === 0) {
    return {
      type: 'flex',
      altText: 'ตะกร้าว่างเปล่า',
      contents: {
        type: 'bubble',
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            { type: 'text', text: '🛒 ตะกร้าว่างเปล่า', align: 'center', weight: 'bold' },
            { type: 'text', text: 'เพิ่มอาหารที่ต้องการ', color: '#999999', align: 'center', margin: 'md', size: 'sm' }
          ]
        }
      }
    };
  }

  const subtotal = cart.items.reduce((sum, i) => sum + i.total, 0);
  const deliveryFee = restaurant ? parseFloat(restaurant.delivery_fee) : 0;
  const total = subtotal + deliveryFee;

  const itemContents = cart.items.map(item => ({
    type: 'box',
    layout: 'horizontal',
    margin: 'sm',
    contents: [
      {
        type: 'box', layout: 'vertical', flex: 4,
        contents: [
          { type: 'text', text: item.name, size: 'sm', weight: 'bold', wrap: true, maxLines: 2 },
          Object.keys(item.options || {}).length > 0 ? {
            type: 'text',
            text: Object.values(item.options).join(', '),
            size: 'xs', color: '#999999'
          } : null,
          { type: 'text', text: `฿${item.unitPrice} x ${item.quantity}`, size: 'xs', color: '#666666' }
        ].filter(Boolean)
      },
      {
        type: 'box', layout: 'vertical', flex: 2, alignItems: 'flex-end',
        contents: [
          { type: 'text', text: `฿${item.total}`, weight: 'bold', size: 'sm', align: 'end' },
          {
            type: 'button',
            action: {
              type: 'postback',
              label: '🗑',
              data: `action=remove_cart_item&item_id=${item.id}`
            },
            style: 'secondary',
            height: 'sm'
          }
        ]
      }
    ]
  }));

  return {
    type: 'flex',
    altText: `ตะกร้า ${cart.items.length} รายการ`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box', layout: 'vertical',
        contents: [{ type: 'text', text: `🛒 ตะกร้า (${cart.items.length} รายการ)`, weight: 'bold', color: '#FFFFFF' }],
        backgroundColor: '#FF6B35'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          restaurant ? { type: 'text', text: restaurant.name, size: 'sm', color: '#666666' } : null,
          ...itemContents,
          { type: 'separator', margin: 'md' },
          {
            type: 'box', layout: 'horizontal', margin: 'md',
            contents: [
              { type: 'text', text: 'ราคาอาหาร:', size: 'sm', flex: 3 },
              { type: 'text', text: `฿${subtotal}`, size: 'sm', align: 'end', flex: 2 }
            ]
          },
          {
            type: 'box', layout: 'horizontal', margin: 'xs',
            contents: [
              { type: 'text', text: 'ค่าส่ง:', size: 'sm', flex: 3 },
              { type: 'text', text: `฿${deliveryFee}`, size: 'sm', align: 'end', flex: 2 }
            ]
          },
          { type: 'separator', margin: 'sm' },
          {
            type: 'box', layout: 'horizontal', margin: 'sm',
            contents: [
              { type: 'text', text: 'รวม:', size: 'md', weight: 'bold', flex: 3 },
              { type: 'text', text: `฿${total}`, size: 'md', weight: 'bold', color: '#FF6B35', align: 'end', flex: 2 }
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
            action: { type: 'postback', label: '➕ เพิ่มอาหาร', data: `action=view_menu&restaurant_id=${cart.restaurantId}` },
            style: 'secondary', height: 'sm'
          },
          {
            type: 'button',
            action: { type: 'postback', label: '🛵 สั่งอาหาร', data: 'action=checkout' },
            style: 'primary', color: '#FF6B35'
          }
        ]
      }
    }
  };
}

module.exports = { createMenuCarousel, createMenuItemCard, createCustomizationMenu, createCartView };
```

## 7. Order Status Messages

```javascript
// src/messages/orderMessages.js
function createOrderConfirmation(order) {
  const address = typeof order.delivery_address === 'string'
    ? JSON.parse(order.delivery_address)
    : order.delivery_address;

  const itemsText = order.items.map(i => `• ${i.name} x${i.quantity} ฿${i.total_price}`).join('\n');

  return {
    type: 'flex',
    altText: `✅ สั่งอาหารแล้ว ${order.order_number}`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box', layout: 'vertical',
        contents: [
          { type: 'text', text: '✅ สั่งอาหารสำเร็จ!', weight: 'bold', color: '#FFFFFF', size: 'lg' },
          { type: 'text', text: `ออเดอร์: ${order.order_number}`, color: '#FFFFFF', size: 'sm' }
        ],
        backgroundColor: '#27AE60'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          { type: 'text', text: order.restaurant_name, weight: 'bold', size: 'md' },
          { type: 'text', text: itemsText, size: 'xs', color: '#666666', wrap: true, margin: 'sm' },
          { type: 'separator', margin: 'md' },
          {
            type: 'box', layout: 'horizontal', margin: 'md',
            contents: [
              { type: 'text', text: 'รวมทั้งสิ้น:', size: 'md', weight: 'bold', flex: 3 },
              { type: 'text', text: `฿${order.total_amount}`, size: 'md', weight: 'bold', color: '#FF6B35', align: 'end', flex: 2 }
            ]
          },
          { type: 'separator', margin: 'md' },
          {
            type: 'box', layout: 'horizontal', margin: 'md',
            contents: [
              { type: 'text', text: '📍 ส่งถึง:', size: 'sm', color: '#666666', flex: 2 },
              { type: 'text', text: address.raw || address.full || 'ที่อยู่ที่บันทึก', size: 'sm', flex: 4, wrap: true }
            ]
          },
          {
            type: 'box', layout: 'horizontal', margin: 'xs',
            contents: [
              { type: 'text', text: '⏱ เวลาโดยประมาณ:', size: 'sm', color: '#666666', flex: 2 },
              { type: 'text', text: `${order.prep_time_minutes + 20} นาที`, size: 'sm', weight: 'bold', flex: 4 }
            ]
          }
        ]
      },
      footer: {
        type: 'box',
        layout: 'vertical',
        contents: [{
          type: 'button',
          action: {
            type: 'postback',
            label: '🔍 ติดตามออเดอร์',
            data: `action=track_order&order_id=${order.id}`
          },
          style: 'primary',
          color: '#FF6B35'
        }]
      }
    }
  };
}

function createOrderTracker(order) {
  const steps = [
    { key: 'pending', label: 'รับออเดอร์', icon: '📋' },
    { key: 'confirmed', label: 'ยืนยัน', icon: '✅' },
    { key: 'preparing', label: 'กำลังปรุง', icon: '👨‍🍳' },
    { key: 'ready', label: 'พร้อมส่ง', icon: '🎉' },
    { key: 'picked_up', label: 'รับอาหาร', icon: '🛵' },
    { key: 'delivering', label: 'กำลังส่ง', icon: '🚀' },
    { key: 'delivered', label: 'ส่งแล้ว', icon: '🏠' }
  ];

  const currentIndex = steps.findIndex(s => s.key === order.status);

  const stepContents = steps.map((step, index) => {
    const isDone = index <= currentIndex;
    const isCurrent = index === currentIndex;

    return {
      type: 'box',
      layout: 'horizontal',
      margin: 'xs',
      contents: [
        {
          type: 'text',
          text: isDone ? step.icon : '○',
          size: 'sm',
          flex: 0,
          color: isDone ? '#27AE60' : '#CCCCCC'
        },
        {
          type: 'text',
          text: step.label,
          size: 'sm',
          margin: 'sm',
          weight: isCurrent ? 'bold' : 'regular',
          color: isDone ? '#2D3748' : '#CCCCCC',
          flex: 1
        },
        isCurrent ? {
          type: 'text',
          text: '← ตอนนี้',
          size: 'xs',
          color: '#27AE60',
          flex: 0
        } : { type: 'filler' }
      ]
    };
  });

  return {
    type: 'flex',
    altText: `ติดตามออเดอร์ ${order.order_number}`,
    contents: {
      type: 'bubble',
      header: {
        type: 'box', layout: 'vertical',
        contents: [
          { type: 'text', text: `🔍 ออเดอร์ ${order.order_number}`, weight: 'bold', color: '#FFFFFF' },
          { type: 'text', text: order.restaurant_name, size: 'sm', color: '#FFFFFF', opacity: 0.8 }
        ],
        backgroundColor: '#FF6B35'
      },
      body: {
        type: 'box',
        layout: 'vertical',
        contents: [
          ...stepContents,
          order.driver_name ? {
            type: 'box', layout: 'horizontal', margin: 'lg',
            contents: [
              { type: 'text', text: `🛵 ${order.driver_name}`, size: 'sm', flex: 3 },
              { type: 'text', text: order.driver_phone || '', size: 'sm', flex: 2, align: 'end' }
            ]
          } : null,
          order.estimated_delivery_time ? {
            type: 'text',
            text: `ประมาณถึง: ${new Date(order.estimated_delivery_time).toLocaleTimeString('th-TH', {hour:'2-digit', minute:'2-digit'})}`,
            size: 'sm', color: '#666666', margin: 'md', align: 'center'
          } : null
        ].filter(Boolean)
      }
    }
  };
}

module.exports = { createOrderConfirmation, createOrderTracker };
```

## 8. Postback Handler

```javascript
// src/handlers/postbackHandler.js
const db = require('../config/database');
const redis = require('../config/redis');
const restaurantService = require('../services/restaurantService');
const cartService = require('../services/cartService');
const orderService = require('../services/orderService');
const { createRestaurantList } = require('../messages/restaurantMessages');
const { createMenuCarousel, createCustomizationMenu, createCartView } = require('../messages/menuMessages');
const { createOrderConfirmation, createOrderTracker } = require('../messages/orderMessages');

async function getOrCreateCustomer(lineUserId, client) {
  const [[existing]] = await db.query('SELECT * FROM customers WHERE line_user_id = ?', [lineUserId]);
  if (existing) return existing;

  const profile = await client.getProfile(lineUserId);
  const [result] = await db.query(
    'INSERT INTO customers (line_user_id, display_name, picture_url) VALUES (?, ?, ?)',
    [lineUserId, profile.displayName, profile.pictureUrl]
  );
  const [[customer]] = await db.query('SELECT * FROM customers WHERE id = ?', [result.insertId]);
  return customer;
}

async function handlePostback(client, event) {
  const userId = event.source.userId;
  const params = new URLSearchParams(event.postback.data);
  const action = params.get('action');

  const customer = await getOrCreateCustomer(userId, client);

  try {
    switch (action) {
      case 'restaurants': {
        const restaurants = await restaurantService.getOpenRestaurants();
        await client.replyMessage(event.replyToken, createRestaurantList(restaurants));
        break;
      }

      case 'view_menu': {
        const restaurantId = parseInt(params.get('restaurant_id'));
        const restaurant = await restaurantService.getRestaurantMenu(restaurantId);
        if (!restaurant) {
          await client.replyMessage(event.replyToken, { type: 'text', text: 'ไม่พบร้านอาหาร' });
          break;
        }

        const messages = [];

        // แสดงเมนูยอดนิยมก่อน
        const popularItems = await restaurantService.getPopularItems(restaurantId);
        if (popularItems.length > 0) {
          messages.push(createMenuCarousel(restaurant, { name: '🔥 เมนูยอดนิยม', items: popularItems }));
        }

        // แสดงทีละ category
        for (const cat of restaurant.categories.slice(0, 3)) {
          if (cat.items && cat.items.length > 0) {
            messages.push(createMenuCarousel(restaurant, cat));
          }
        }

        if (messages.length === 0) {
          await client.replyMessage(event.replyToken, { type: 'text', text: 'ไม่มีเมนู' });
        } else {
          await client.replyMessage(event.replyToken, messages.slice(0, 5));
        }
        break;
      }

      case 'add_to_cart': {
        const itemId = parseInt(params.get('item_id'));
        const restaurantId = parseInt(params.get('restaurant_id'));
        const qty = parseInt(params.get('qty') || '1');

        const result = await cartService.addItem(userId, restaurantId, itemId, qty, {}, [], '');

        if (result.needsClear) {
          // บอกให้ยืนยัน
          await redis.setex(`pending_add:${userId}`, 300, JSON.stringify({ itemId, restaurantId, qty }));
          await client.replyMessage(event.replyToken, {
            type: 'flex',
            altText: 'มีสินค้าจากร้านอื่นในตะกร้า',
            contents: {
              type: 'bubble',
              body: {
                type: 'box', layout: 'vertical',
                contents: [
                  { type: 'text', text: '⚠️ ตะกร้ามีอาหารจากร้านอื่น', weight: 'bold' },
                  { type: 'text', text: 'ต้องการล้างตะกร้าและเพิ่มจากร้านนี้?', size: 'sm', margin: 'md', color: '#666666', wrap: true }
                ]
              },
              footer: {
                type: 'box', layout: 'horizontal', spacing: 'sm',
                contents: [
                  {
                    type: 'button',
                    action: { type: 'postback', label: 'ล้างตะกร้า', data: `action=clear_and_add&item_id=${itemId}&restaurant_id=${restaurantId}&qty=${qty}` },
                    style: 'primary', color: '#FF6B35', flex: 1
                  },
                  {
                    type: 'button',
                    action: { type: 'postback', label: 'ยกเลิก', data: 'action=view_cart' },
                    style: 'secondary', flex: 1
                  }
                ]
              }
            }
          });
          break;
        }

        const [[rest]] = await db.query('SELECT * FROM restaurants WHERE id = ?', [restaurantId]);
        await client.replyMessage(event.replyToken, [
          { type: 'text', text: '✅ เพิ่มลงตะกร้าแล้ว!' },
          createCartView(result.cart, rest)
        ]);
        break;
      }

      case 'clear_and_add': {
        await cartService.clearCart(userId);
        const itemId = parseInt(params.get('item_id'));
        const restaurantId = parseInt(params.get('restaurant_id'));
        const qty = parseInt(params.get('qty') || '1');

        const result = await cartService.addItem(userId, restaurantId, itemId, qty, {}, [], '');
        const [[rest]] = await db.query('SELECT * FROM restaurants WHERE id = ?', [restaurantId]);
        await client.replyMessage(event.replyToken, createCartView(result.cart, rest));
        break;
      }

      case 'view_cart': {
        const cart = await cartService.getCart(userId);
        let restaurant = null;
        if (cart.restaurantId) {
          [[restaurant]] = await db.query('SELECT * FROM restaurants WHERE id = ?', [cart.restaurantId]);
        }
        await client.replyMessage(event.replyToken, createCartView(cart, restaurant));
        break;
      }

      case 'remove_cart_item': {
        const itemId = params.get('item_id');
        const cart = await cartService.updateQuantity(userId, itemId, 0);
        let restaurant = null;
        if (cart.restaurantId) {
          [[restaurant]] = await db.query('SELECT * FROM restaurants WHERE id = ?', [cart.restaurantId]);
        }
        await client.replyMessage(event.replyToken, createCartView(cart, restaurant));
        break;
      }

      case 'checkout': {
        const cart = await cartService.getCart(userId);
        if (!cart.items || cart.items.length === 0) {
          await client.replyMessage(event.replyToken, { type: 'text', text: '❌ ตะกร้าว่างเปล่า' });
          break;
        }

        await redis.setex(`state:${userId}`, 600, JSON.stringify({ state: 'waiting_address' }));
        await client.replyMessage(event.replyToken, {
          type: 'flex',
          altText: 'ระบุที่อยู่จัดส่ง',
          contents: {
            type: 'bubble',
            body: {
              type: 'box', layout: 'vertical',
              contents: [
                { type: 'text', text: '📍 ที่อยู่จัดส่ง', weight: 'bold', size: 'lg' },
                { type: 'text', text: 'กรุณาพิมพ์ที่อยู่จัดส่งของคุณ', size: 'sm', color: '#666666', margin: 'md', wrap: true }
              ]
            },
            footer: {
              type: 'box', layout: 'vertical',
              contents: customer.default_address ? [{
                type: 'button',
                action: { type: 'postback', label: 'ใช้ที่อยู่เดิม', data: 'action=use_saved_address' },
                style: 'primary', color: '#FF6B35'
              }] : []
            }
          }
        });
        break;
      }

      case 'use_saved_address': {
        if (!customer.default_address) {
          await client.replyMessage(event.replyToken, { type: 'text', text: 'ไม่พบที่อยู่ที่บันทึก' });
          break;
        }
        await placeOrder(client, event, userId, JSON.parse(customer.default_address));
        break;
      }

      case 'track_order': {
        const orderId = parseInt(params.get('order_id'));
        const order = await orderService.getOrder(orderId);
        if (!order) {
          await client.replyMessage(event.replyToken, { type: 'text', text: 'ไม่พบออเดอร์' });
          break;
        }
        await client.replyMessage(event.replyToken, createOrderTracker(order));
        break;
      }

      case 'rate_order': {
        const orderId = parseInt(params.get('order_id'));
        const rating = parseInt(params.get('rating'));

        await db.query(
          'UPDATE orders SET rating = ? WHERE id = ?',
          [rating, orderId]
        );

        const stars = '⭐'.repeat(rating);
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: `${stars}\nขอบคุณที่ให้คะแนน ${rating}/5 ดาวครับ!`
        });
        break;
      }

      default:
        await client.replyMessage(event.replyToken, {
          type: 'text',
          text: 'ไม่เข้าใจคำสั่ง'
        });
    }
  } catch (error) {
    console.error('Postback error:', error);
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: `❌ ${error.message}`
    });
  }
}

async function placeOrder(client, event, userId, deliveryAddress) {
  try {
    const order = await orderService.createOrder(userId, deliveryAddress);
    await client.replyMessage(event.replyToken, createOrderConfirmation(order));
  } catch (err) {
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: `❌ สั่งอาหารไม่สำเร็จ: ${err.message}`
    });
  }
}

module.exports = { handlePostback };
```

## 9. Kitchen Display System (KDS) API

```javascript
// src/routes/kds.js
const express = require('express');
const db = require('../config/database');
const orderService = require('../services/orderService');

const router = express.Router();

// Auth middleware
function kdsAuth(req, res, next) {
  if (req.headers['x-kds-token'] !== process.env.KDS_TOKEN) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
}

router.use(kdsAuth);

// ดึงออเดอร์ที่ต้องเตรียม (สำหรับหน้าจอครัว)
router.get('/orders', async (req, res) => {
  const restaurantId = req.query.restaurant_id;

  const [orders] = await db.query(
    `SELECT o.*, c.display_name as customer_name,
            COUNT(oi.id) as item_count
     FROM orders o
     JOIN customers c ON o.customer_id = c.id
     LEFT JOIN order_items oi ON o.id = oi.order_id
     WHERE o.restaurant_id = ?
       AND o.status IN ('confirmed', 'preparing')
     GROUP BY o.id
     ORDER BY o.created_at ASC`,
    [restaurantId]
  );

  // ดึง items ของแต่ละ order
  const ordersWithItems = await Promise.all(orders.map(async order => {
    const [items] = await db.query(
      'SELECT * FROM order_items WHERE order_id = ?',
      [order.id]
    );
    return { ...order, items };
  }));

  res.json({ orders: ordersWithItems });
});

// อัปเดตสถานะ
router.put('/orders/:id/status', async (req, res) => {
  const { id } = req.params;
  const { status } = req.body;

  try {
    const order = await orderService.updateStatus(parseInt(id), status);
    res.json({ order });
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

// Socket.io events สำหรับ real-time KDS
router.get('/stream', (req, res) => {
  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');

  const restaurantId = req.query.restaurant_id;

  const sendOrders = async () => {
    const [orders] = await db.query(
      `SELECT o.order_number, o.status, o.created_at,
              GROUP_CONCAT(oi.name ORDER BY oi.id SEPARATOR ', ') as items_summary
       FROM orders o
       LEFT JOIN order_items oi ON o.id = oi.order_id
       WHERE o.restaurant_id = ? AND o.status IN ('confirmed','preparing')
       GROUP BY o.id
       ORDER BY o.created_at`,
      [restaurantId]
    );

    res.write(`data: ${JSON.stringify(orders)}\n\n`);
  };

  sendOrders();
  const interval = setInterval(sendOrders, 10000); // ทุก 10 วินาที

  req.on('close', () => clearInterval(interval));
});

module.exports = router;
```

## 10. Driver Tracking

```javascript
// src/routes/driver.js
const express = require('express');
const db = require('../config/database');
const orderService = require('../services/orderService');
const line = require('@line/bot-sdk');

const router = express.Router();
const lineClient = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

// Driver auth
function driverAuth(req, res, next) {
  const driverId = req.headers['x-driver-id'];
  if (!driverId) return res.status(401).json({ error: 'Unauthorized' });
  req.driverId = parseInt(driverId);
  next();
}

router.use(driverAuth);

// อัปเดตตำแหน่ง
router.post('/location', async (req, res) => {
  const { lat, lng } = req.body;

  await db.query(
    'UPDATE drivers SET current_lat = ?, current_lng = ? WHERE id = ?',
    [lat, lng, req.driverId]
  );

  // อัปเดตตำแหน่งในออเดอร์ที่กำลัง deliver
  await db.query(
    "UPDATE orders SET driver_lat = ?, driver_lng = ? WHERE driver_id = ? AND status = 'delivering'",
    [lat, lng, req.driverId]
  );

  res.json({ ok: true });
});

// รับออเดอร์
router.post('/orders/:id/accept', async (req, res) => {
  const { id } = req.params;

  await db.query(
    "UPDATE orders SET driver_id = ?, status = 'picked_up' WHERE id = ? AND status = 'ready'",
    [req.driverId, id]
  );

  await orderService.updateStatus(parseInt(id), 'picked_up');
  res.json({ ok: true });
});

// ยืนยันส่งแล้ว
router.post('/orders/:id/deliver', async (req, res) => {
  const { id } = req.params;
  const order = await orderService.updateStatus(parseInt(id), 'delivered');

  // อัปเดตสถิติ driver
  await db.query(
    'UPDATE drivers SET total_deliveries = total_deliveries + 1 WHERE id = ?',
    [req.driverId]
  );

  res.json({ order });
});

// ดูออเดอร์ที่รับได้
router.get('/available-orders', async (req, res) => {
  const { lat, lng } = req.query;

  const [orders] = await db.query(
    `SELECT o.*, r.name as restaurant_name, r.address as restaurant_address,
            r.lat as restaurant_lat, r.lng as restaurant_lng
     FROM orders o
     JOIN restaurants r ON o.restaurant_id = r.id
     WHERE o.status = 'ready' AND o.driver_id IS NULL
     ORDER BY o.created_at ASC
     LIMIT 10`
  );

  res.json({ orders });
});

module.exports = router;
```

## 11. Admin Routes

```javascript
// src/routes/admin.js
const express = require('express');
const db = require('../config/database');

const router = express.Router();

function adminAuth(req, res, next) {
  if (req.headers['x-admin-token'] !== process.env.ADMIN_TOKEN) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
}

router.use(adminAuth);

// Dashboard stats
router.get('/stats', async (req, res) => {
  const [[today]] = await db.query(
    `SELECT COUNT(*) as orders, COALESCE(SUM(total_amount), 0) as revenue
     FROM orders WHERE DATE(created_at) = CURDATE() AND status != 'cancelled'`
  );

  const [[pending]] = await db.query(
    "SELECT COUNT(*) as count FROM orders WHERE status IN ('pending','confirmed','preparing')"
  );

  res.json({ today, pending: pending.count });
});

// จัดการเมนู
router.post('/menu-items', async (req, res) => {
  const { restaurant_id, category_id, name, description, price, image_url, options, addons } = req.body;
  const [result] = await db.query(
    `INSERT INTO menu_items (restaurant_id, category_id, name, description, price, image_url, options, addons)
     VALUES (?, ?, ?, ?, ?, ?, ?, ?)`,
    [restaurant_id, category_id, name, description, price, image_url,
     options ? JSON.stringify(options) : null,
     addons ? JSON.stringify(addons) : null]
  );
  res.json({ id: result.insertId });
});

router.put('/menu-items/:id', async (req, res) => {
  const { id } = req.params;
  const { is_available, price, name } = req.body;

  const updates = {};
  if (is_available !== undefined) updates.is_available = is_available;
  if (price !== undefined) updates.price = price;
  if (name) updates.name = name;

  if (Object.keys(updates).length) {
    const setClause = Object.keys(updates).map(k => `${k} = ?`).join(', ');
    await db.query(`UPDATE menu_items SET ${setClause} WHERE id = ?`, [...Object.values(updates), id]);
  }

  res.json({ ok: true });
});

// Toggle restaurant open/close
router.put('/restaurants/:id/toggle', async (req, res) => {
  const { id } = req.params;
  await db.query('UPDATE restaurants SET is_open = !is_open WHERE id = ?', [id]);
  res.json({ ok: true });
});

// รายการออเดอร์
router.get('/orders', async (req, res) => {
  const { status, restaurant_id, date } = req.query;
  let where = '1=1';
  const params = [];

  if (status) { where += ' AND o.status = ?'; params.push(status); }
  if (restaurant_id) { where += ' AND o.restaurant_id = ?'; params.push(restaurant_id); }
  if (date) { where += ' AND DATE(o.created_at) = ?'; params.push(date); }

  const [orders] = await db.query(
    `SELECT o.*, r.name as restaurant_name, c.display_name as customer_name
     FROM orders o
     JOIN restaurants r ON o.restaurant_id = r.id
     JOIN customers c ON o.customer_id = c.id
     WHERE ${where}
     ORDER BY o.created_at DESC LIMIT 50`,
    params
  );

  res.json({ orders });
});

module.exports = router;
```

## 12. Webhook + Main

```javascript
// src/routes/webhook.js
const express = require('express');
const line = require('@line/bot-sdk');
const redis = require('../config/redis');
const db = require('../config/database');
const { handlePostback } = require('../handlers/postbackHandler');
const { createRestaurantList } = require('../messages/restaurantMessages');
const restaurantService = require('../services/restaurantService');
const cartService = require('../services/cartService');
const orderService = require('../services/orderService');
const { createOrderConfirmation } = require('../messages/orderMessages');

const router = express.Router();
const lineConfig = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};
const client = new line.Client(lineConfig);

router.post('/', line.middleware(lineConfig), async (req, res) => {
  await Promise.all(req.body.events.map(handleEvent));
  res.json({ ok: true });
});

async function handleEvent(event) {
  const userId = event.source.userId;

  if (event.type === 'follow') {
    const profile = await client.getProfile(userId);
    const [[existing]] = await db.query('SELECT id FROM customers WHERE line_user_id = ?', [userId]);
    if (!existing) {
      await db.query(
        'INSERT INTO customers (line_user_id, display_name, picture_url) VALUES (?, ?, ?)',
        [userId, profile.displayName, profile.pictureUrl]
      );
    }
    await client.replyMessage(event.replyToken, {
      type: 'text',
      text: `สวัสดีครับ ${profile.displayName}! 🍔\n\nยินดีต้อนรับสู่ระบบสั่งอาหาร\nพิมพ์ "ร้านอาหาร" เพื่อดูร้านที่เปิดอยู่`
    });
    return;
  }

  if (event.type === 'postback') {
    await handlePostback(client, event);
    return;
  }

  if (event.type === 'message' && event.message.type === 'text') {
    const text = event.message.text.trim();

    const stateData = await redis.get(`state:${userId}`);
    const state = stateData ? JSON.parse(stateData) : null;

    if (state?.state === 'waiting_address') {
      await redis.del(`state:${userId}`);
      const address = { raw: text };
      await db.query('UPDATE customers SET default_address = ? WHERE line_user_id = ?', [JSON.stringify(address), userId]);

      try {
        const order = await orderService.createOrder(userId, address);
        await client.replyMessage(event.replyToken, createOrderConfirmation(order));
      } catch (err) {
        await client.replyMessage(event.replyToken, { type: 'text', text: `❌ ${err.message}` });
      }
      return;
    }

    const commands = {
      'ร้านอาหาร': async () => {
        const restaurants = await restaurantService.getOpenRestaurants();
        await client.replyMessage(event.replyToken, createRestaurantList(restaurants));
      },
      'ตะกร้า': async () => {
        const cart = await cartService.getCart(userId);
        let restaurant = null;
        if (cart.restaurantId) {
          [[restaurant]] = await db.query('SELECT * FROM restaurants WHERE id = ?', [cart.restaurantId]);
        }
        const { createCartView } = require('../messages/menuMessages');
        await client.replyMessage(event.replyToken, createCartView(cart, restaurant));
      },
      'ออเดอร์': async () => {
        const orders = await orderService.getUserOrders(userId, 5);
        if (orders.length === 0) {
          await client.replyMessage(event.replyToken, { type: 'text', text: 'ยังไม่มีออเดอร์' });
        } else {
          const text = orders.map(o => `${o.order_number} - ${o.restaurant_name} (${o.status})`).join('\n');
          await client.replyMessage(event.replyToken, { type: 'text', text: `📦 ออเดอร์ของคุณ:\n${text}` });
        }
      }
    };

    if (commands[text]) {
      await commands[text]();
    } else {
      await client.replyMessage(event.replyToken, {
        type: 'text',
        text: '🍔 พิมพ์ "ร้านอาหาร" เพื่อดูร้านที่เปิดอยู่\n🛒 พิมพ์ "ตะกร้า" เพื่อดูสินค้าในตะกร้า'
      });
    }
  }
}

module.exports = router;
```

```javascript
// index.js
require('dotenv').config();
const express = require('express');
const app = express();

app.use(express.json());
app.use('/webhook', require('./src/routes/webhook'));
app.use('/admin', require('./src/routes/admin'));
app.use('/kds', require('./src/routes/kds'));
app.use('/driver', require('./src/routes/driver'));

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`Food ordering bot on port ${PORT}`));
```

## 13. package.json

```json
{
  "name": "lineoa-food",
  "version": "1.0.0",
  "dependencies": {
    "@line/bot-sdk": "^8.0.0",
    "dotenv": "^16.0.0",
    "express": "^4.18.0",
    "ioredis": "^5.3.0",
    "mysql2": "^3.6.0",
    "socket.io": "^4.6.0"
  }
}
```

## สรุป

ระบบสั่งอาหารบน LINE OA ที่สร้างในบทนี้ครอบคลุม:

1. **Multi-Restaurant** - รองรับหลายร้านอาหาร
2. **Smart Menu** - เมนูพร้อม options/addons
3. **Redis Cart** - ตะกร้าอาหาร session-based
4. **Order Flow** - ขั้นตอนสั่งอาหารครบวงจร
5. **KDS** - Kitchen Display System API
6. **Driver App** - API สำหรับคนขับ
7. **Real-time Tracking** - ติดตามออเดอร์
8. **Notifications** - แจ้งเตือนทุกขั้นตอน
9. **Rating System** - ให้คะแนนหลังรับอาหาร
10. **Admin Panel** - จัดการจากฝั่ง admin
