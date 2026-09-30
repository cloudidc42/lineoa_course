# Part 35: Dynamic Flex Messages

## สารบัญ

1. [Building Flex Messages Programmatically](#building-flex-messages-programmatically)
2. [Template Functions ใน Node.js](#template-functions-nodejs)
3. [Generating Flex from Database](#generating-flex-from-database)
4. [Dynamic Product Cards from API](#dynamic-product-cards-from-api)
5. [Receipt Generation](#receipt-generation)
6. [Notification Cards](#notification-cards)
7. [Error Message Cards](#error-message-cards)
8. [Success/Confirmation Cards](#success-confirmation-cards)
9. [Flex Message Builder Utility](#flex-message-builder-utility)
10. [Python Implementation](#python-implementation)
11. [Testing Strategies](#testing-strategies)
12. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Building Flex Messages Programmatically

การสร้าง Flex Message แบบ dynamic หมายถึงการสร้าง JSON structure โดยใช้ code แทนที่จะเขียน JSON ตายตัว ซึ่งช่วยให้:

- **Reusability**: ใช้ template function ซ้ำได้
- **Maintainability**: แก้ไขแค่ที่เดียว ทุกที่ได้รับผล
- **Flexibility**: ปรับแต่ง output ตาม data ที่รับมา
- **Type Safety**: ตรวจสอบ input ได้ก่อนส่ง

### Design Principles

```
1. Separation of Concerns
   - Data layer: ดึงข้อมูล
   - Template layer: สร้าง Flex structure
   - Sending layer: ส่ง message

2. Immutability
   - ไม่แก้ไข input data
   - return object ใหม่เสมอ

3. Default Values
   - กำหนด default สำหรับทุก optional field
   - ป้องกัน undefined/null errors

4. Validation
   - ตรวจสอบ required fields
   - sanitize input data
```

### โครงสร้างโค้ด (Node.js)

```javascript
// src/flex/
// ├── components/
// │   ├── header.js
// │   ├── hero.js
// │   ├── body.js
// │   └── footer.js
// ├── templates/
// │   ├── productCard.js
// │   ├── receipt.js
// │   ├── notification.js
// │   └── error.js
// ├── builder.js
// └── index.js
```

---

## 2. Template Functions ใน Node.js

### Component Factory Functions

```javascript
// src/flex/components/header.js

/**
 * สร้าง header box component
 * @param {Object} options
 * @param {string} options.text - ข้อความใน header
 * @param {string} [options.backgroundColor='#ffffff'] - สีพื้นหลัง
 * @param {string} [options.textColor='#333333'] - สีตัวอักษร
 * @param {string} [options.size='sm'] - ขนาดตัวอักษร
 * @param {string} [options.badge] - badge text (optional)
 * @param {string} [options.badgeColor='#FF5722'] - สี badge
 */
function createHeader({
  text,
  backgroundColor = '#ffffff',
  textColor = '#333333',
  size = 'sm',
  badge = null,
  badgeColor = '#FF5722',
} = {}) {
  if (!text) return null;

  const contents = [
    {
      type: 'text',
      text,
      weight: 'bold',
      size,
      color: textColor,
      flex: 1,
    },
  ];

  if (badge) {
    contents.push({
      type: 'box',
      layout: 'vertical',
      flex: 0,
      paddingAll: '3px 8px',
      backgroundColor: badgeColor,
      cornerRadius: '4px',
      contents: [
        {
          type: 'text',
          text: badge,
          size: 'xxs',
          color: '#FFFFFF',
          weight: 'bold',
        },
      ],
    });
  }

  return {
    type: 'box',
    layout: 'horizontal',
    paddingAll: badge ? '10px 16px' : '12px 16px',
    backgroundColor,
    contents,
  };
}

module.exports = { createHeader };
```

```javascript
// src/flex/components/hero.js

/**
 * สร้าง hero image component
 * @param {Object} options
 * @param {string} options.url - URL ของรูปภาพ
 * @param {string} [options.aspectRatio='20:13'] - aspect ratio
 * @param {string} [options.aspectMode='cover'] - cover หรือ fit
 * @param {Object} [options.action] - action เมื่อแตะ
 * @param {boolean} [options.animated=false] - animated GIF
 */
function createHero({
  url,
  aspectRatio = '20:13',
  aspectMode = 'cover',
  action = null,
  animated = false,
  placeholder = 'https://via.placeholder.com/400x260/eeeeee/888888?text=No+Image',
} = {}) {
  if (!url && !placeholder) return null;

  const hero = {
    type: 'image',
    url: url || placeholder,
    size: 'full',
    aspectRatio,
    aspectMode,
  };

  if (animated) hero.animated = true;
  if (action) hero.action = action;

  return hero;
}

/**
 * สร้าง hero box ที่มี overlay text
 */
function createHeroWithOverlay({
  imageUrl,
  overlayText,
  overlaySubtext = null,
  aspectRatio = '20:13',
} = {}) {
  return {
    type: 'box',
    layout: 'vertical',
    contents: [
      {
        type: 'image',
        url: imageUrl,
        size: 'full',
        aspectRatio,
        aspectMode: 'cover',
      },
      {
        type: 'box',
        layout: 'vertical',
        position: 'absolute',
        offsetBottom: '0px',
        offsetStart: '0px',
        offsetEnd: '0px',
        paddingAll: '12px',
        backgroundColor: '#00000060',
        contents: [
          {
            type: 'text',
            text: overlayText,
            color: '#FFFFFF',
            weight: 'bold',
            size: 'md',
            wrap: true,
          },
          ...(overlaySubtext
            ? [
                {
                  type: 'text',
                  text: overlaySubtext,
                  color: '#ffffffcc',
                  size: 'sm',
                  margin: 'xs',
                },
              ]
            : []),
        ],
      },
    ],
  };
}

module.exports = { createHero, createHeroWithOverlay };
```

```javascript
// src/flex/components/body.js

/**
 * สร้าง info row (label: value)
 */
function createInfoRow({
  label,
  value,
  labelColor = '#888888',
  valueColor = '#333333',
  labelFlex = 2,
  valueFlex = 3,
  valueWrap = false,
  margin = 'none',
} = {}) {
  return {
    type: 'box',
    layout: 'horizontal',
    margin,
    contents: [
      {
        type: 'text',
        text: label,
        size: 'sm',
        color: labelColor,
        flex: labelFlex,
      },
      {
        type: 'text',
        text: value,
        size: 'sm',
        color: valueColor,
        flex: valueFlex,
        wrap: valueWrap,
      },
    ],
  };
}

/**
 * สร้าง price row
 */
function createPriceRow({
  label = 'ราคา',
  price,
  originalPrice = null,
  currency = '฿',
  priceColor = '#FF5722',
  showDiscount = true,
} = {}) {
  const discount =
    originalPrice && showDiscount
      ? Math.round(((originalPrice - price) / originalPrice) * 100)
      : 0;

  const priceContents = [
    {
      type: 'text',
      text: label,
      size: 'sm',
      color: '#888888',
      flex: 0,
    },
    {
      type: 'filler',
    },
    {
      type: 'text',
      text: `${currency}${price.toLocaleString()}`,
      weight: 'bold',
      size: 'xl',
      color: priceColor,
      flex: 0,
    },
  ];

  const contents = [
    {
      type: 'box',
      layout: 'horizontal',
      alignItems: 'center',
      contents: priceContents,
    },
  ];

  if (originalPrice) {
    contents.push({
      type: 'box',
      layout: 'horizontal',
      contents: [
        { type: 'filler' },
        {
          type: 'text',
          text: `${currency}${originalPrice.toLocaleString()}`,
          color: '#aaaaaa',
          decoration: 'line-through',
          size: 'sm',
          flex: 0,
        },
        ...(discount > 0
          ? [
              {
                type: 'box',
                layout: 'vertical',
                flex: 0,
                paddingAll: '2px 6px',
                backgroundColor: '#F44336',
                cornerRadius: '4px',
                margin: 'sm',
                contents: [
                  {
                    type: 'text',
                    text: `-${discount}%`,
                    size: 'xxs',
                    color: '#FFFFFF',
                    weight: 'bold',
                  },
                ],
              },
            ]
          : []),
      ],
    });
  }

  return { type: 'box', layout: 'vertical', contents };
}

/**
 * สร้าง rating bar
 */
function createRatingBar({
  rating,
  maxRating = 5,
  reviewCount = null,
  starUrl = 'https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png',
  emptyStarUrl = 'https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gray_star_28.png',
  size = 'xs',
} = {}) {
  const fullStars = Math.floor(rating);
  const hasHalfStar = rating % 1 >= 0.5;
  const emptyStars = maxRating - fullStars - (hasHalfStar ? 1 : 0);

  const contents = [
    ...Array(fullStars).fill(null).map(() => ({
      type: 'icon',
      url: starUrl,
      size,
    })),
    ...(hasHalfStar
      ? [{ type: 'icon', url: starUrl, size }]
      : []),
    ...Array(emptyStars).fill(null).map(() => ({
      type: 'icon',
      url: emptyStarUrl,
      size,
    })),
  ];

  if (reviewCount !== null) {
    contents.push({
      type: 'text',
      text: `${rating} (${reviewCount.toLocaleString()})`,
      size,
      color: '#aaaaaa',
      margin: 'xs',
    });
  }

  return {
    type: 'box',
    layout: 'baseline',
    spacing: 'xxs',
    contents,
  };
}

/**
 * สร้าง progress bar
 */
function createProgressBar({
  progress,
  total = 100,
  label = null,
  barColor = '#1565C0',
  backgroundColor = '#E0E0E0',
  height = '8px',
  showPercentage = true,
} = {}) {
  const percentage = Math.min(Math.round((progress / total) * 100), 100);
  const remaining = 100 - percentage;

  const contents = [
    ...(label || showPercentage
      ? [
          {
            type: 'box',
            layout: 'horizontal',
            margin: 'none',
            contents: [
              ...(label
                ? [{ type: 'text', text: label, size: 'xs', color: '#888888', flex: 1 }]
                : [{ type: 'filler' }]),
              ...(showPercentage
                ? [
                    {
                      type: 'text',
                      text: `${percentage}%`,
                      size: 'xs',
                      color: barColor,
                      weight: 'bold',
                      flex: 0,
                    },
                  ]
                : []),
            ],
          },
        ]
      : []),
    {
      type: 'box',
      layout: 'horizontal',
      height,
      backgroundColor,
      cornerRadius: height,
      margin: label || showPercentage ? 'xs' : 'none',
      contents: [
        {
          type: 'box',
          layout: 'vertical',
          backgroundColor: barColor,
          cornerRadius: height,
          flex: percentage,
          contents: [],
        },
        ...(remaining > 0
          ? [
              {
                type: 'box',
                layout: 'vertical',
                flex: remaining,
                contents: [],
              },
            ]
          : []),
      ],
    },
  ];

  return { type: 'box', layout: 'vertical', spacing: 'xs', contents };
}

/**
 * สร้าง separator
 */
function createSeparator({ margin = 'md', color = '#eeeeee' } = {}) {
  return { type: 'separator', margin, color };
}

module.exports = {
  createInfoRow,
  createPriceRow,
  createRatingBar,
  createProgressBar,
  createSeparator,
};
```

```javascript
// src/flex/components/footer.js

/**
 * สร้าง single button footer
 */
function createSingleButtonFooter({
  label,
  action,
  style = 'primary',
  color = null,
  height = 'sm',
} = {}) {
  const button = {
    type: 'button',
    style,
    height,
    action,
  };

  if (color) button.color = color;

  return {
    type: 'box',
    layout: 'vertical',
    paddingAll: '12px',
    contents: [button],
  };
}

/**
 * สร้าง dual button footer
 */
function createDualButtonFooter({
  primaryLabel,
  primaryAction,
  primaryColor = null,
  secondaryLabel,
  secondaryAction,
  height = 'sm',
} = {}) {
  const primaryBtn = {
    type: 'button',
    style: 'primary',
    height,
    flex: 1,
    action: primaryAction,
  };
  if (primaryColor) primaryBtn.color = primaryColor;

  return {
    type: 'box',
    layout: 'horizontal',
    spacing: 'sm',
    paddingAll: '12px',
    contents: [
      {
        type: 'button',
        style: 'secondary',
        height,
        flex: 1,
        action: secondaryAction,
      },
      primaryBtn,
    ],
  };
}

module.exports = { createSingleButtonFooter, createDualButtonFooter };
```

---

## 3. Generating Flex from Database

### Database Schema Example (MySQL)

```sql
-- products table
CREATE TABLE products (
  id VARCHAR(36) PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  price DECIMAL(10, 2) NOT NULL,
  original_price DECIMAL(10, 2),
  category VARCHAR(100),
  image_url VARCHAR(500),
  rating DECIMAL(3, 1) DEFAULT 0,
  review_count INT DEFAULT 0,
  stock_count INT DEFAULT 0,
  is_active BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- orders table
CREATE TABLE orders (
  id VARCHAR(36) PRIMARY KEY,
  user_id VARCHAR(100) NOT NULL,
  status ENUM('pending', 'confirmed', 'processing', 'shipped', 'delivered', 'cancelled'),
  total_amount DECIMAL(10, 2) NOT NULL,
  discount_amount DECIMAL(10, 2) DEFAULT 0,
  shipping_fee DECIMAL(10, 2) DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- order_items table
CREATE TABLE order_items (
  id VARCHAR(36) PRIMARY KEY,
  order_id VARCHAR(36) REFERENCES orders(id),
  product_id VARCHAR(36) REFERENCES products(id),
  product_name VARCHAR(255) NOT NULL,
  quantity INT NOT NULL,
  unit_price DECIMAL(10, 2) NOT NULL,
  subtotal DECIMAL(10, 2) NOT NULL
);
```

### Repository Pattern

```javascript
// src/repositories/productRepository.js
const db = require('../database'); // your DB connection

class ProductRepository {
  async findById(id) {
    const [rows] = await db.query(
      'SELECT * FROM products WHERE id = ? AND is_active = TRUE',
      [id]
    );
    return rows[0] || null;
  }

  async findByCategory(category, limit = 10, offset = 0) {
    const [rows] = await db.query(
      `SELECT * FROM products 
       WHERE category = ? AND is_active = TRUE 
       ORDER BY rating DESC, review_count DESC
       LIMIT ? OFFSET ?`,
      [category, limit, offset]
    );
    return rows;
  }

  async findFeatured(limit = 6) {
    const [rows] = await db.query(
      `SELECT * FROM products 
       WHERE is_active = TRUE AND stock_count > 0
       ORDER BY (rating * LOG(review_count + 1)) DESC
       LIMIT ?`,
      [limit]
    );
    return rows;
  }

  async countByCategory(category) {
    const [rows] = await db.query(
      'SELECT COUNT(*) as count FROM products WHERE category = ? AND is_active = TRUE',
      [category]
    );
    return rows[0].count;
  }
}

module.exports = new ProductRepository();
```

### Service Layer

```javascript
// src/services/flexMessageService.js
const productRepository = require('../repositories/productRepository');
const orderRepository = require('../repositories/orderRepository');
const { createProductCarousel } = require('../flex/templates/productCard');
const { createReceiptBubble } = require('../flex/templates/receipt');
const { createNotificationBubble } = require('../flex/templates/notification');

class FlexMessageService {
  /**
   * สร้าง product carousel จาก database
   */
  async createProductCarouselFromDB(category, page = 1, pageSize = 5) {
    const offset = (page - 1) * pageSize;
    
    const [products, totalCount] = await Promise.all([
      productRepository.findByCategory(category, pageSize + 1, offset), // +1 เพื่อเช็คว่ามีหน้าถัดไปไหม
      productRepository.countByCategory(category),
    ]);

    const hasMore = products.length > pageSize;
    const displayProducts = products.slice(0, pageSize);

    return createProductCarousel(displayProducts, {
      showViewAll: hasMore,
      totalCount,
      category,
      currentPage: page,
    });
  }

  /**
   * สร้าง receipt จาก order ใน database
   */
  async createReceiptFromDB(orderId) {
    const order = await orderRepository.findById(orderId);
    if (!order) throw new Error(`Order ${orderId} not found`);

    const items = await orderRepository.findItemsByOrderId(orderId);

    return createReceiptBubble({
      orderId: order.id,
      status: order.status,
      items: items.map(item => ({
        name: item.product_name,
        quantity: item.quantity,
        price: item.unit_price,
        subtotal: item.subtotal,
      })),
      subtotal: order.total_amount - order.shipping_fee + order.discount_amount,
      shippingFee: order.shipping_fee,
      discount: order.discount_amount,
      total: order.total_amount,
      createdAt: order.created_at,
    });
  }
}

module.exports = new FlexMessageService();
```

---

## 4. Dynamic Product Cards from API

### ดึงข้อมูลจาก External API

```javascript
// src/services/productApiService.js
const axios = require('axios');

class ProductApiService {
  constructor() {
    this.baseUrl = process.env.PRODUCT_API_URL;
    this.apiKey = process.env.PRODUCT_API_KEY;
    this.client = axios.create({
      baseURL: this.baseUrl,
      headers: { 'X-API-Key': this.apiKey },
      timeout: 5000,
    });
  }

  async getProducts(params = {}) {
    try {
      const response = await this.client.get('/products', { params });
      return response.data;
    } catch (error) {
      console.error('Product API Error:', error.message);
      throw new Error('ไม่สามารถดึงข้อมูลสินค้าได้ กรุณาลองใหม่อีกครั้ง');
    }
  }

  async getProductById(id) {
    try {
      const response = await this.client.get(`/products/${id}`);
      return response.data;
    } catch (error) {
      if (error.response?.status === 404) {
        return null;
      }
      throw error;
    }
  }
}

module.exports = new ProductApiService();
```

### Template: Product Card

```javascript
// src/flex/templates/productCard.js
const { createHeader } = require('../components/header');
const { createHero } = require('../components/hero');
const { createRatingBar, createPriceRow, createSeparator } = require('../components/body');
const { createDualButtonFooter } = require('../components/footer');

/**
 * สร้าง product card bubble
 */
function createProductCard(product) {
  // Validate required fields
  if (!product || !product.name || product.price === undefined) {
    throw new Error('Product must have name and price');
  }

  // Normalize data
  const data = {
    id: product.id || product.productId || 'unknown',
    name: product.name || 'ไม่ระบุชื่อ',
    description: product.description || '',
    price: Number(product.price) || 0,
    originalPrice: product.originalPrice ? Number(product.originalPrice) : null,
    imageUrl: product.imageUrl || product.image_url || null,
    rating: Number(product.rating) || 0,
    reviewCount: Number(product.reviewCount || product.review_count) || 0,
    category: product.category || '',
    url: product.url || product.productUrl || `https://example.com/products/${product.id}`,
    inStock: product.inStock !== false && product.stock_count !== 0,
    badge: product.badge || determineBadge(product),
  };

  const discount = data.originalPrice
    ? Math.round(((data.originalPrice - data.price) / data.originalPrice) * 100)
    : 0;

  // Build header
  const header = data.badge
    ? createHeader({
        text: data.badge,
        backgroundColor: getBadgeBackground(data.badge),
        textColor: getBadgeTextColor(data.badge),
      })
    : null;

  // Build hero
  const hero = createHero({
    url: data.imageUrl,
    action: { type: 'uri', uri: data.url },
  });

  // Build body
  const bodyContents = [
    {
      type: 'text',
      text: data.name,
      weight: 'bold',
      size: 'lg',
      maxLines: 2,
      wrap: true,
    },
  ];

  if (data.category) {
    bodyContents.push({
      type: 'text',
      text: data.category,
      size: 'xs',
      color: '#999999',
    });
  }

  if (data.rating > 0) {
    bodyContents.push(createRatingBar({
      rating: data.rating,
      reviewCount: data.reviewCount,
      margin: 'sm',
    }));
  }

  if (data.description) {
    bodyContents.push({
      type: 'text',
      text: data.description,
      size: 'sm',
      color: '#666666',
      wrap: true,
      maxLines: 2,
      margin: 'sm',
    });
  }

  bodyContents.push(createSeparator({ margin: 'md' }));
  bodyContents.push(createPriceRow({
    price: data.price,
    originalPrice: data.originalPrice,
    margin: 'md',
  }));

  if (!data.inStock) {
    bodyContents.push({
      type: 'text',
      text: '❌ สินค้าหมด',
      size: 'xs',
      color: '#F44336',
      margin: 'xs',
    });
  }

  const body = {
    type: 'box',
    layout: 'vertical',
    paddingAll: '16px',
    spacing: 'sm',
    contents: bodyContents,
  };

  // Build footer
  const footer = createDualButtonFooter({
    primaryLabel: data.inStock ? '🛒 ซื้อเลย' : '❌ หมด',
    primaryAction: data.inStock
      ? {
          type: 'postback',
          label: '🛒 ซื้อเลย',
          data: `action=buy&id=${data.id}`,
          displayText: `เพิ่ม ${data.name} ลงตะกร้า`,
        }
      : {
          type: 'message',
          label: 'แจ้งเตือนเมื่อมีสินค้า',
          text: `แจ้งเตือนสินค้า ${data.id}`,
        },
    primaryColor: data.inStock ? '#FF5722' : '#9E9E9E',
    secondaryLabel: 'ดูรายละเอียด',
    secondaryAction: { type: 'uri', label: 'ดูรายละเอียด', uri: data.url },
  });

  // Assemble bubble
  const bubble = {
    type: 'bubble',
    size: 'mega',
    hero,
    body,
    footer,
    styles: {
      footer: { separator: true, separatorColor: '#eeeeee' },
    },
  };

  if (header) {
    bubble.header = header;
    bubble.styles.header = { backgroundColor: getBadgeBackground(data.badge) };
  }

  return bubble;
}

// Helper functions
function determineBadge(product) {
  if (product.isFeatured) return '🔥 แนะนำ';
  if (product.isNew) return '✨ ใหม่';
  const discount =
    product.originalPrice
      ? Math.round(((product.originalPrice - product.price) / product.originalPrice) * 100)
      : 0;
  if (discount >= 20) return `ลด ${discount}%`;
  return null;
}

function getBadgeBackground(badge) {
  if (!badge) return '#ffffff';
  if (badge.includes('🔥') || badge.includes('ลด')) return '#FFF3F0';
  if (badge.includes('✨')) return '#E3F2FD';
  return '#F5F5F5';
}

function getBadgeTextColor(badge) {
  if (!badge) return '#333333';
  if (badge.includes('🔥') || badge.includes('ลด')) return '#FF5722';
  if (badge.includes('✨')) return '#1565C0';
  return '#555555';
}

/**
 * สร้าง product carousel
 */
function createProductCarousel(products, options = {}) {
  const {
    showViewAll = false,
    totalCount = 0,
    category = 'all',
    maxCards = 10,
  } = options;

  if (!products || products.length === 0) {
    return {
      type: 'text',
      text: 'ไม่พบสินค้าในขณะนี้ กรุณาลองใหม่อีกครั้ง',
    };
  }

  const bubbles = products
    .slice(0, showViewAll ? maxCards - 1 : maxCards)
    .map(product => createProductCard(product));

  if (showViewAll && products.length >= maxCards) {
    bubbles.push(createViewAllCard(category, totalCount));
  }

  return {
    type: 'flex',
    altText: `สินค้า ${products.length} รายการ`,
    contents: {
      type: 'carousel',
      contents: bubbles,
    },
  };
}

function createViewAllCard(category, totalCount) {
  return {
    type: 'bubble',
    size: 'mega',
    body: {
      type: 'box',
      layout: 'vertical',
      justifyContent: 'center',
      alignItems: 'center',
      spacing: 'md',
      contents: [
        { type: 'text', text: '→', size: '5xl', color: '#cccccc', align: 'center' },
        { type: 'text', text: 'ดูสินค้าทั้งหมด', size: 'md', color: '#888888', align: 'center', weight: 'bold' },
        { type: 'text', text: `${totalCount.toLocaleString()} รายการ`, size: 'sm', color: '#aaaaaa', align: 'center' },
      ],
    },
    footer: {
      type: 'box',
      layout: 'vertical',
      paddingAll: '12px',
      contents: [{
        type: 'button',
        style: 'secondary',
        action: { type: 'uri', label: `ดูทั้งหมด`, uri: `https://example.com/products?cat=${category}` },
      }],
    },
  };
}

module.exports = { createProductCard, createProductCarousel };
```

---

## 5. Receipt Generation

```javascript
// src/flex/templates/receipt.js

/**
 * สร้าง receipt bubble จากข้อมูล order
 * @param {Object} order
 * @param {string} order.orderId
 * @param {string} order.status - pending, confirmed, processing, shipped, delivered, cancelled
 * @param {Array} order.items - [{name, quantity, price, subtotal}]
 * @param {number} order.subtotal
 * @param {number} order.shippingFee
 * @param {number} order.discount
 * @param {number} order.total
 * @param {Date|string} order.createdAt
 * @param {Object} [order.shippingAddress]
 * @param {string} [order.paymentMethod]
 */
function createReceiptBubble(order) {
  const {
    orderId,
    status = 'confirmed',
    items = [],
    subtotal,
    shippingFee = 0,
    discount = 0,
    total,
    createdAt,
    shippingAddress,
    paymentMethod,
  } = order;

  const statusConfig = getStatusConfig(status);
  const dateStr = formatDate(createdAt);

  // Header
  const header = {
    type: 'box',
    layout: 'vertical',
    paddingAll: '20px',
    paddingBottom: '16px',
    contents: [
      {
        type: 'box',
        layout: 'horizontal',
        contents: [
          { type: 'text', text: '🧾 ใบเสร็จรับเงิน', weight: 'bold', size: 'lg', color: '#1a1a1a', flex: 1 },
          {
            type: 'box',
            layout: 'vertical',
            flex: 0,
            paddingAll: '3px 8px',
            backgroundColor: statusConfig.bgColor,
            cornerRadius: '4px',
            contents: [{ type: 'text', text: statusConfig.label, size: 'xs', color: statusConfig.textColor, weight: 'bold' }],
          },
        ],
      },
      { type: 'text', text: `คำสั่งซื้อ #${orderId}`, size: 'xs', color: '#888888', margin: 'xs' },
      { type: 'text', text: dateStr, size: 'xs', color: '#888888' },
    ],
  };

  // Body items
  const itemRows = items.map(item => ({
    type: 'box',
    layout: 'horizontal',
    contents: [
      { type: 'text', text: item.name, size: 'sm', color: '#333333', flex: 3, wrap: true, maxLines: 2 },
      { type: 'text', text: `x${item.quantity}`, size: 'sm', color: '#888888', flex: 1, align: 'center' },
      { type: 'text', text: `฿${item.subtotal.toLocaleString()}`, size: 'sm', color: '#333333', flex: 1, align: 'end' },
    ],
  }));

  // Summary rows
  const summaryRows = [
    createSummaryRow('รวมสินค้า', `฿${subtotal.toLocaleString()}`),
    ...(shippingFee > 0 ? [createSummaryRow('ค่าจัดส่ง', `฿${shippingFee.toLocaleString()}`)] : []),
    ...(shippingFee === 0 ? [createSummaryRow('ค่าจัดส่ง', 'ฟรี!', '#4CAF50')] : []),
    ...(discount > 0 ? [createSummaryRow('ส่วนลด', `-฿${discount.toLocaleString()}`, '#4CAF50')] : []),
  ];

  const bodyContents = [
    { type: 'text', text: 'รายการสินค้า', weight: 'bold', size: 'sm', color: '#555555' },
    { type: 'separator', color: '#eeeeee' },
    ...itemRows,
    { type: 'separator', color: '#eeeeee', margin: 'md' },
    ...summaryRows,
    { type: 'separator', color: '#eeeeee', margin: 'sm' },
    // Total
    {
      type: 'box',
      layout: 'horizontal',
      margin: 'sm',
      contents: [
        { type: 'text', text: 'ยอดชำระรวม', weight: 'bold', size: 'md', color: '#1a1a1a', flex: 1 },
        { type: 'text', text: `฿${total.toLocaleString()}`, weight: 'bold', size: 'xl', color: '#FF5722', flex: 0 },
      ],
    },
  ];

  // Shipping address
  if (shippingAddress) {
    bodyContents.push({ type: 'separator', color: '#eeeeee', margin: 'md' });
    bodyContents.push({
      type: 'box',
      layout: 'vertical',
      backgroundColor: '#F5F5F5',
      paddingAll: '12px',
      cornerRadius: '8px',
      margin: 'md',
      contents: [
        { type: 'text', text: '📍 ที่อยู่จัดส่ง', size: 'xs', color: '#888888', weight: 'bold' },
        { type: 'text', text: shippingAddress.name, size: 'sm', color: '#333333', margin: 'xs' },
        { type: 'text', text: shippingAddress.address, size: 'xs', color: '#666666', wrap: true },
        { type: 'text', text: `โทร: ${shippingAddress.phone}`, size: 'xs', color: '#666666', margin: 'xs' },
      ],
    });
  }

  const body = {
    type: 'box',
    layout: 'vertical',
    paddingAll: '20px',
    spacing: 'sm',
    contents: bodyContents,
  };

  // Footer buttons based on status
  const footerButtons = getReceiptFooterButtons(orderId, status);

  return {
    type: 'flex',
    altText: `ใบเสร็จ #${orderId} - รวม ฿${total.toLocaleString()} (${statusConfig.label})`,
    contents: {
      type: 'bubble',
      size: 'mega',
      header,
      body,
      footer: {
        type: 'box',
        layout: 'vertical',
        spacing: 'sm',
        paddingAll: '16px',
        contents: footerButtons,
      },
      styles: {
        header: { backgroundColor: '#FAFAFA' },
        footer: { backgroundColor: '#FAFAFA', separator: true, separatorColor: '#eeeeee' },
      },
    },
  };
}

function createSummaryRow(label, value, valueColor = '#333333') {
  return {
    type: 'box',
    layout: 'horizontal',
    contents: [
      { type: 'text', text: label, size: 'sm', color: '#555555', flex: 1 },
      { type: 'text', text: value, size: 'sm', color: valueColor, flex: 0 },
    ],
  };
}

function getStatusConfig(status) {
  const configs = {
    pending: { label: 'รอยืนยัน', bgColor: '#FFF8E1', textColor: '#F57F17' },
    confirmed: { label: 'ยืนยันแล้ว', bgColor: '#E3F2FD', textColor: '#1565C0' },
    processing: { label: 'กำลังเตรียม', bgColor: '#E8EAF6', textColor: '#3949AB' },
    shipped: { label: 'จัดส่งแล้ว', bgColor: '#E0F7FA', textColor: '#00695C' },
    delivered: { label: 'ส่งถึงแล้ว', bgColor: '#E8F5E9', textColor: '#2E7D32' },
    cancelled: { label: 'ยกเลิก', bgColor: '#FFEBEE', textColor: '#C62828' },
  };
  return configs[status] || configs.confirmed;
}

function getReceiptFooterButtons(orderId, status) {
  const buttons = [];

  if (['confirmed', 'processing', 'shipped'].includes(status)) {
    buttons.push({
      type: 'button',
      style: 'primary',
      height: 'sm',
      action: { type: 'postback', label: '📍 ติดตามพัสดุ', data: `action=track&orderId=${orderId}` },
    });
  }

  buttons.push({
    type: 'button',
    style: 'secondary',
    height: 'sm',
    action: { type: 'uri', label: '📄 ดาวน์โหลดใบเสร็จ', uri: `https://example.com/receipts/${orderId}` },
  });

  if (status === 'delivered') {
    buttons.push({
      type: 'button',
      style: 'secondary',
      height: 'sm',
      action: { type: 'postback', label: '⭐ รีวิวสินค้า', data: `action=review&orderId=${orderId}` },
    });
  }

  return buttons;
}

function formatDate(date) {
  const d = new Date(date);
  const months = ['ม.ค.', 'ก.พ.', 'มี.ค.', 'เม.ย.', 'พ.ค.', 'มิ.ย.', 'ก.ค.', 'ส.ค.', 'ก.ย.', 'ต.ค.', 'พ.ย.', 'ธ.ค.'];
  return `${d.getDate()} ${months[d.getMonth()]} ${d.getFullYear() + 543}, ${String(d.getHours()).padStart(2, '0')}:${String(d.getMinutes()).padStart(2, '0')} น.`;
}

module.exports = { createReceiptBubble };
```

---

## 6. Notification Cards

```javascript
// src/flex/templates/notification.js

const NOTIFICATION_TYPES = {
  ORDER: { icon: '🛒', color: '#1565C0', bgColor: '#E3F2FD' },
  PAYMENT: { icon: '💳', color: '#2E7D32', bgColor: '#E8F5E9' },
  SHIPPING: { icon: '🚚', color: '#E65100', bgColor: '#FFF3E0' },
  PROMO: { icon: '🎉', color: '#6A1B9A', bgColor: '#F3E5F5' },
  ALERT: { icon: '⚠️', color: '#F57F17', bgColor: '#FFF8E1' },
  ERROR: { icon: '❌', color: '#C62828', bgColor: '#FFEBEE' },
  SUCCESS: { icon: '✅', color: '#2E7D32', bgColor: '#E8F5E9' },
  INFO: { icon: 'ℹ️', color: '#1565C0', bgColor: '#E3F2FD' },
};

/**
 * สร้าง notification bubble
 */
function createNotificationBubble({
  type = 'INFO',
  title,
  message,
  detail = null,
  timestamp = new Date(),
  action = null,
  actionLabel = 'ดูรายละเอียด',
  imageUrl = null,
} = {}) {
  const config = NOTIFICATION_TYPES[type] || NOTIFICATION_TYPES.INFO;
  const timeStr = formatRelativeTime(timestamp);

  const bodyContents = [
    // Icon + title row
    {
      type: 'box',
      layout: 'horizontal',
      spacing: 'md',
      contents: [
        ...(imageUrl
          ? [
              {
                type: 'image',
                url: imageUrl,
                size: 'sm',
                aspectRatio: '1:1',
                aspectMode: 'cover',
                flex: 0,
              },
            ]
          : [
              {
                type: 'box',
                layout: 'vertical',
                flex: 0,
                width: '40px',
                height: '40px',
                backgroundColor: config.bgColor,
                cornerRadius: '50%',
                justifyContent: 'center',
                alignItems: 'center',
                contents: [
                  { type: 'text', text: config.icon, size: 'lg', align: 'center' },
                ],
              },
            ]),
        {
          type: 'box',
          layout: 'vertical',
          flex: 1,
          contents: [
            { type: 'text', text: title, weight: 'bold', size: 'sm', color: '#1a1a1a', maxLines: 2, wrap: true },
            { type: 'text', text: message, size: 'xs', color: '#666666', wrap: true, maxLines: 3, margin: 'xs' },
          ],
        },
        {
          type: 'text',
          text: timeStr,
          size: 'xxs',
          color: '#aaaaaa',
          flex: 0,
          gravity: 'top',
        },
      ],
    },
  ];

  if (detail) {
    bodyContents.push({ type: 'separator', margin: 'md', color: '#eeeeee' });
    bodyContents.push({
      type: 'box',
      layout: 'vertical',
      backgroundColor: config.bgColor,
      paddingAll: '10px',
      cornerRadius: '6px',
      margin: 'md',
      contents: [
        { type: 'text', text: detail, size: 'xs', color: config.color, wrap: true },
      ],
    });
  }

  const bubble = {
    type: 'bubble',
    size: 'kilo',
    body: {
      type: 'box',
      layout: 'vertical',
      paddingAll: '16px',
      spacing: 'md',
      contents: bodyContents,
    },
  };

  if (action) {
    bubble.footer = {
      type: 'box',
      layout: 'vertical',
      paddingAll: '12px',
      contents: [
        {
          type: 'button',
          style: 'link',
          height: 'sm',
          action: { ...action, label: actionLabel },
        },
      ],
    };
    bubble.styles = { footer: { separator: true, separatorColor: '#eeeeee' } };
  }

  return {
    type: 'flex',
    altText: `[${type}] ${title}: ${message}`,
    contents: bubble,
  };
}

/**
 * สร้าง order notification
 */
function createOrderNotification(order) {
  const statusMessages = {
    confirmed: { title: 'ยืนยันคำสั่งซื้อแล้ว', type: 'ORDER' },
    processing: { title: 'กำลังเตรียมสินค้า', type: 'ORDER' },
    shipped: { title: 'จัดส่งสินค้าแล้ว', type: 'SHIPPING' },
    delivered: { title: 'ส่งสินค้าสำเร็จ!', type: 'SUCCESS' },
    cancelled: { title: 'ยกเลิกคำสั่งซื้อ', type: 'ERROR' },
  };

  const config = statusMessages[order.status] || statusMessages.confirmed;

  return createNotificationBubble({
    type: config.type,
    title: config.title,
    message: `คำสั่งซื้อ #${order.orderId} - ฿${order.total.toLocaleString()}`,
    detail: order.trackingNumber ? `เลขพัสดุ: ${order.trackingNumber}` : null,
    timestamp: order.updatedAt,
    action: {
      type: 'postback',
      data: `action=viewOrder&orderId=${order.orderId}`,
    },
    actionLabel: 'ดูรายละเอียดคำสั่งซื้อ',
  });
}

function formatRelativeTime(date) {
  const now = new Date();
  const diff = now - new Date(date);
  const minutes = Math.floor(diff / 60000);
  const hours = Math.floor(diff / 3600000);
  const days = Math.floor(diff / 86400000);

  if (minutes < 1) return 'เพิ่งเมื่อกี้';
  if (minutes < 60) return `${minutes} นาทีที่แล้ว`;
  if (hours < 24) return `${hours} ชั่วโมงที่แล้ว`;
  if (days < 7) return `${days} วันที่แล้ว`;
  return new Date(date).toLocaleDateString('th-TH');
}

module.exports = { createNotificationBubble, createOrderNotification };
```

---

## 7. Error Message Cards

```javascript
// src/flex/templates/error.js

/**
 * สร้าง error message bubble
 */
function createErrorCard({
  title = 'เกิดข้อผิดพลาด',
  message,
  errorCode = null,
  suggestion = null,
  retryAction = null,
  helpUrl = null,
} = {}) {
  const bodyContents = [
    {
      type: 'box',
      layout: 'vertical',
      alignItems: 'center',
      spacing: 'sm',
      contents: [
        { type: 'text', text: '❌', size: '4xl', align: 'center' },
        { type: 'text', text: title, weight: 'bold', size: 'lg', align: 'center', color: '#C62828' },
        { type: 'text', text: message, size: 'sm', color: '#666666', align: 'center', wrap: true, maxLines: 4 },
      ],
    },
  ];

  if (errorCode) {
    bodyContents.push({
      type: 'box',
      layout: 'vertical',
      backgroundColor: '#FFEBEE',
      paddingAll: '10px',
      cornerRadius: '6px',
      margin: 'md',
      contents: [
        { type: 'text', text: `รหัสข้อผิดพลาด: ${errorCode}`, size: 'xs', color: '#C62828', align: 'center' },
      ],
    });
  }

  if (suggestion) {
    bodyContents.push({
      type: 'box',
      layout: 'vertical',
      backgroundColor: '#FFF8E1',
      paddingAll: '10px',
      cornerRadius: '6px',
      margin: 'md',
      contents: [
        { type: 'text', text: '💡 แนะนำ:', size: 'xs', color: '#F57F17', weight: 'bold' },
        { type: 'text', text: suggestion, size: 'xs', color: '#E65100', wrap: true, margin: 'xs' },
      ],
    });
  }

  const footerButtons = [];

  if (retryAction) {
    footerButtons.push({
      type: 'button',
      style: 'primary',
      color: '#EF5350',
      height: 'sm',
      action: { ...retryAction, label: '🔄 ลองใหม่' },
    });
  }

  if (helpUrl) {
    footerButtons.push({
      type: 'button',
      style: 'secondary',
      height: 'sm',
      action: { type: 'uri', label: '❓ ติดต่อฝ่ายสนับสนุน', uri: helpUrl },
    });
  }

  return {
    type: 'flex',
    altText: `ข้อผิดพลาด: ${title} - ${message}`,
    contents: {
      type: 'bubble',
      size: 'kilo',
      body: {
        type: 'box',
        layout: 'vertical',
        paddingAll: '20px',
        spacing: 'md',
        contents: bodyContents,
      },
      ...(footerButtons.length > 0
        ? {
            footer: {
              type: 'box',
              layout: 'vertical',
              spacing: 'sm',
              paddingAll: '16px',
              contents: footerButtons,
            },
            styles: { footer: { separator: true, separatorColor: '#eeeeee' } },
          }
        : {}),
    },
  };
}

// Error presets
const ErrorCards = {
  notFound: (item = 'ข้อมูล') =>
    createErrorCard({
      title: 'ไม่พบข้อมูล',
      message: `ขออภัย ไม่พบ${item}ที่คุณต้องการ`,
      suggestion: 'ลองค้นหาด้วยคำอื่น หรือกลับไปหน้าหลัก',
    }),

  serverError: () =>
    createErrorCard({
      title: 'เซิร์ฟเวอร์ขัดข้อง',
      message: 'ขณะนี้ระบบมีปัญหาชั่วคราว กรุณาลองใหม่อีกครั้งในภายหลัง',
      errorCode: 'SERVER_ERROR_500',
      helpUrl: 'https://example.com/support',
    }),

  networkError: (retryData) =>
    createErrorCard({
      title: 'เชื่อมต่อไม่ได้',
      message: 'ไม่สามารถเชื่อมต่อกับเซิร์ฟเวอร์ได้',
      suggestion: 'ตรวจสอบการเชื่อมต่ออินเทอร์เน็ตของคุณ',
      retryAction: retryData ? { type: 'postback', data: retryData } : null,
    }),

  unauthorized: () =>
    createErrorCard({
      title: 'ไม่มีสิทธิ์เข้าถึง',
      message: 'คุณไม่มีสิทธิ์ดำเนินการนี้ กรุณาเข้าสู่ระบบก่อน',
      suggestion: 'กด "เข้าสู่ระบบ" เพื่อยืนยันตัวตน',
      retryAction: { type: 'uri', uri: 'https://example.com/login' },
    }),
};

module.exports = { createErrorCard, ErrorCards };
```

---

## 8. Success/Confirmation Cards

```javascript
// src/flex/templates/success.js

/**
 * สร้าง success/confirmation bubble
 */
function createSuccessCard({
  title = 'สำเร็จแล้ว!',
  message,
  details = [],
  primaryAction = null,
  primaryLabel = 'ตกลง',
  secondaryAction = null,
  secondaryLabel = 'กลับหน้าหลัก',
  animatedIcon = false,
} = {}) {
  const bodyContents = [
    {
      type: 'box',
      layout: 'vertical',
      alignItems: 'center',
      spacing: 'sm',
      contents: [
        { type: 'text', text: '✅', size: '4xl', align: 'center' },
        { type: 'text', text: title, weight: 'bold', size: 'xl', align: 'center', color: '#2E7D32' },
        { type: 'text', text: message, size: 'sm', color: '#555555', align: 'center', wrap: true },
      ],
    },
  ];

  if (details.length > 0) {
    bodyContents.push({ type: 'separator', margin: 'lg', color: '#eeeeee' });
    bodyContents.push({
      type: 'box',
      layout: 'vertical',
      margin: 'lg',
      spacing: 'sm',
      contents: details.map(({ label, value, valueColor }) => ({
        type: 'box',
        layout: 'horizontal',
        contents: [
          { type: 'text', text: label, size: 'sm', color: '#888888', flex: 2 },
          { type: 'text', text: value, size: 'sm', color: valueColor || '#333333', flex: 3, wrap: true },
        ],
      })),
    });
  }

  const footerButtons = [];
  if (primaryAction) {
    footerButtons.push({
      type: 'button',
      style: 'primary',
      color: '#4CAF50',
      height: 'sm',
      action: { ...primaryAction, label: primaryLabel },
    });
  }
  if (secondaryAction) {
    footerButtons.push({
      type: 'button',
      style: 'secondary',
      height: 'sm',
      action: { ...secondaryAction, label: secondaryLabel },
    });
  }

  return {
    type: 'flex',
    altText: `✅ ${title}: ${message}`,
    contents: {
      type: 'bubble',
      size: 'kilo',
      body: { type: 'box', layout: 'vertical', paddingAll: '24px', spacing: 'md', contents: bodyContents },
      ...(footerButtons.length > 0
        ? {
            footer: {
              type: 'box',
              layout: 'vertical',
              spacing: 'sm',
              paddingAll: '16px',
              contents: footerButtons,
            },
          }
        : {}),
    },
  };
}

// Success presets
const SuccessCards = {
  orderPlaced: (orderId, total) =>
    createSuccessCard({
      title: 'สั่งซื้อสำเร็จ!',
      message: 'เราได้รับคำสั่งซื้อของคุณแล้ว',
      details: [
        { label: 'เลขคำสั่งซื้อ', value: `#${orderId}` },
        { label: 'ยอดรวม', value: `฿${total.toLocaleString()}`, valueColor: '#FF5722' },
      ],
      primaryAction: { type: 'postback', data: `action=viewOrder&id=${orderId}` },
      primaryLabel: '📦 ติดตามคำสั่งซื้อ',
    }),

  registrationComplete: (name) =>
    createSuccessCard({
      title: 'ลงทะเบียนสำเร็จ!',
      message: `ยินดีต้อนรับ ${name}! บัญชีของคุณพร้อมใช้งานแล้ว`,
      primaryAction: { type: 'postback', data: 'action=goHome' },
      primaryLabel: '🏠 เริ่มใช้งาน',
    }),

  paymentSuccess: (amount, method) =>
    createSuccessCard({
      title: 'ชำระเงินสำเร็จ!',
      message: 'การชำระเงินของคุณได้รับการยืนยันแล้ว',
      details: [
        { label: 'จำนวนเงิน', value: `฿${amount.toLocaleString()}`, valueColor: '#4CAF50' },
        { label: 'วิธีชำระ', value: method },
      ],
    }),
};

module.exports = { createSuccessCard, SuccessCards };
```

---

## 9. Flex Message Builder Utility

```javascript
// src/flex/builder.js
/**
 * FlexMessageBuilder - utility class สำหรับสร้าง Flex Message แบบ fluent API
 */
class FlexMessageBuilder {
  constructor() {
    this._type = 'flex';
    this._altText = '';
    this._bubble = {
      type: 'bubble',
      size: 'mega',
    };
  }

  // === Bubble Settings ===

  size(size) {
    this._bubble.size = size;
    return this;
  }

  altText(text) {
    this._altText = text;
    return this;
  }

  // === Header ===

  header({ text, backgroundColor, textColor, badge, badgeColor } = {}) {
    this._bubble.header = {
      type: 'box',
      layout: 'horizontal',
      paddingAll: '12px 16px',
      ...(backgroundColor ? { backgroundColor } : {}),
      contents: [
        {
          type: 'text',
          text,
          weight: 'bold',
          size: 'sm',
          color: textColor || '#333333',
          flex: 1,
        },
        ...(badge
          ? [
              {
                type: 'box',
                layout: 'vertical',
                flex: 0,
                paddingAll: '3px 8px',
                backgroundColor: badgeColor || '#FF5722',
                cornerRadius: '4px',
                contents: [
                  { type: 'text', text: badge, size: 'xxs', color: '#FFFFFF', weight: 'bold' },
                ],
              },
            ]
          : []),
      ],
    };
    return this;
  }

  // === Hero ===

  heroImage({ url, aspectRatio = '20:13', aspectMode = 'cover', action, animated } = {}) {
    this._bubble.hero = {
      type: 'image',
      url,
      size: 'full',
      aspectRatio,
      aspectMode,
      ...(action ? { action } : {}),
      ...(animated ? { animated: true } : {}),
    };
    return this;
  }

  heroBox(contents, options = {}) {
    this._bubble.hero = {
      type: 'box',
      layout: 'vertical',
      ...options,
      contents,
    };
    return this;
  }

  // === Body ===

  body(contents, options = {}) {
    this._bubble.body = {
      type: 'box',
      layout: 'vertical',
      paddingAll: '16px',
      spacing: 'md',
      ...options,
      contents,
    };
    return this;
  }

  // === Footer ===

  footer(contents, options = {}) {
    this._bubble.footer = {
      type: 'box',
      layout: 'vertical',
      spacing: 'sm',
      paddingAll: '12px',
      ...options,
      contents,
    };
    return this;
  }

  primaryButton({ label, action, color }) {
    const btn = { type: 'button', style: 'primary', height: 'sm', action };
    if (color) btn.color = color;
    this._ensureFooter();
    this._bubble.footer.contents.push(btn);
    return this;
  }

  secondaryButton({ label, action }) {
    this._ensureFooter();
    this._bubble.footer.contents.push({
      type: 'button',
      style: 'secondary',
      height: 'sm',
      action: { ...action, label },
    });
    return this;
  }

  // === Styles ===

  styles(styles) {
    this._bubble.styles = { ...this._bubble.styles, ...styles };
    return this;
  }

  // === Build ===

  build() {
    if (!this._altText) {
      throw new Error('altText is required');
    }
    return {
      type: this._type,
      altText: this._altText,
      contents: this._bubble,
    };
  }

  // === Static helpers ===

  static text(text, options = {}) {
    return { type: 'text', text, ...options };
  }

  static box(layout, contents, options = {}) {
    return { type: 'box', layout, contents, ...options };
  }

  static image(url, options = {}) {
    return { type: 'image', url, size: 'full', aspectRatio: '20:13', aspectMode: 'cover', ...options };
  }

  static button(label, action, style = 'primary', options = {}) {
    return { type: 'button', style, height: 'sm', action: { ...action, label }, ...options };
  }

  static separator(options = {}) {
    return { type: 'separator', color: '#eeeeee', ...options };
  }

  static filler() {
    return { type: 'filler' };
  }

  // === Private ===

  _ensureFooter() {
    if (!this._bubble.footer) {
      this._bubble.footer = {
        type: 'box',
        layout: 'vertical',
        spacing: 'sm',
        paddingAll: '12px',
        contents: [],
      };
    }
  }
}

// ตัวอย่างการใช้งาน Builder
function exampleUsage() {
  const { text, box, image, button, separator, filler } = FlexMessageBuilder;

  const msg = new FlexMessageBuilder()
    .altText('สินค้า: Nike Air Max 90')
    .size('mega')
    .heroImage({
      url: 'https://example.com/nike.jpg',
      action: { type: 'uri', uri: 'https://example.com/products/nike' },
    })
    .body([
      text('Nike Air Max 90', { weight: 'bold', size: 'xl', maxLines: 2, wrap: true }),
      text('Nike', { size: 'xs', color: '#999999' }),
      separator(),
      box('horizontal', [
        text('ราคา', { size: 'sm', color: '#888888', flex: 0 }),
        filler(),
        text('฿3,500', { weight: 'bold', size: 'xl', color: '#FF5722', flex: 0 }),
      ], { margin: 'md' }),
    ])
    .footer([
      box('horizontal', [
        button('ดูรายละเอียด', { type: 'uri', uri: 'https://example.com' }, 'secondary', { flex: 1 }),
        button('ซื้อเลย', { type: 'postback', data: 'action=buy&id=Nike-AM90' }, 'primary', { flex: 1, color: '#FF5722' }),
      ], { layout: 'horizontal', spacing: 'sm' }),
    ])
    .styles({ footer: { separator: true, separatorColor: '#eeeeee' } })
    .build();

  return msg;
}

module.exports = FlexMessageBuilder;
```

---

## 10. Python Implementation

```python
# flex_builder.py
from typing import Optional, List, Dict, Any, Union
from dataclasses import dataclass, field
from datetime import datetime
import json


@dataclass
class FlexComponent:
    """Base class for Flex components"""
    
    @staticmethod
    def text(text: str, **kwargs) -> dict:
        return {"type": "text", "text": text, **kwargs}
    
    @staticmethod
    def box(layout: str, contents: list, **kwargs) -> dict:
        return {"type": "box", "layout": layout, "contents": contents, **kwargs}
    
    @staticmethod
    def image(url: str, **kwargs) -> dict:
        defaults = {"size": "full", "aspectRatio": "20:13", "aspectMode": "cover"}
        return {"type": "image", "url": url, **defaults, **kwargs}
    
    @staticmethod
    def button(label: str, action: dict, style: str = "primary", **kwargs) -> dict:
        return {"type": "button", "style": style, "height": "sm", 
                "action": {**action, "label": label}, **kwargs}
    
    @staticmethod
    def separator(**kwargs) -> dict:
        return {"type": "separator", "color": "#eeeeee", **kwargs}
    
    @staticmethod
    def filler() -> dict:
        return {"type": "filler"}


class FlexMessageBuilder:
    """Fluent builder สำหรับสร้าง Flex Message"""
    
    def __init__(self):
        self._alt_text = ""
        self._bubble = {"type": "bubble", "size": "mega"}
        self.fc = FlexComponent()
    
    def alt_text(self, text: str) -> 'FlexMessageBuilder':
        self._alt_text = text
        return self
    
    def size(self, size: str) -> 'FlexMessageBuilder':
        self._bubble['size'] = size
        return self
    
    def header(self, text: str, background_color: str = None, 
               text_color: str = "#333333", badge: str = None, 
               badge_color: str = "#FF5722") -> 'FlexMessageBuilder':
        contents = [self.fc.text(text, weight="bold", size="sm", color=text_color, flex=1)]
        
        if badge:
            contents.append({
                "type": "box",
                "layout": "vertical",
                "flex": 0,
                "paddingAll": "3px 8px",
                "backgroundColor": badge_color,
                "cornerRadius": "4px",
                "contents": [self.fc.text(badge, size="xxs", color="#FFFFFF", weight="bold")]
            })
        
        header = {
            "type": "box",
            "layout": "horizontal",
            "paddingAll": "12px 16px",
            "contents": contents
        }
        if background_color:
            header['backgroundColor'] = background_color
        
        self._bubble['header'] = header
        return self
    
    def hero_image(self, url: str, aspect_ratio: str = "20:13",
                   aspect_mode: str = "cover", action: dict = None) -> 'FlexMessageBuilder':
        hero = self.fc.image(url, aspectRatio=aspect_ratio, aspectMode=aspect_mode)
        if action:
            hero['action'] = action
        self._bubble['hero'] = hero
        return self
    
    def body(self, contents: list, **kwargs) -> 'FlexMessageBuilder':
        self._bubble['body'] = {
            "type": "box",
            "layout": "vertical",
            "paddingAll": "16px",
            "spacing": "md",
            **kwargs,
            "contents": contents
        }
        return self
    
    def footer(self, contents: list, **kwargs) -> 'FlexMessageBuilder':
        self._bubble['footer'] = {
            "type": "box",
            "layout": "vertical",
            "spacing": "sm",
            "paddingAll": "12px",
            **kwargs,
            "contents": contents
        }
        return self
    
    def styles(self, **kwargs) -> 'FlexMessageBuilder':
        if 'styles' not in self._bubble:
            self._bubble['styles'] = {}
        self._bubble['styles'].update(kwargs)
        return self
    
    def build(self) -> dict:
        if not self._alt_text:
            raise ValueError("altText is required")
        return {
            "type": "flex",
            "altText": self._alt_text,
            "contents": self._bubble
        }


# Template functions
def create_product_card_python(product: dict) -> dict:
    """สร้าง product card Flex Message"""
    fc = FlexComponent()
    
    price = product.get('price', 0)
    original_price = product.get('original_price')
    discount = 0
    if original_price:
        discount = round(((original_price - price) / original_price) * 100)
    
    body_contents = [
        fc.text(product.get('name', ''), weight="bold", size="xl", maxLines=2, wrap=True),
        fc.text(product.get('category', ''), size="xs", color="#999999"),
        fc.separator(margin="md"),
    ]
    
    # Price row
    price_row_contents = [
        fc.text("฿{:,}".format(price), weight="bold", size="xl", color="#FF5722", flex=1)
    ]
    
    if original_price:
        price_row_contents.append(
            fc.text("฿{:,}".format(original_price), 
                   color="#aaaaaa", decoration="line-through", size="sm", flex=0)
        )
    
    body_contents.append(fc.box("horizontal", price_row_contents, margin="md", alignItems="flex-end"))
    
    # Build message
    builder = FlexMessageBuilder()
    
    alt_text = f"สินค้า: {product.get('name')} - ราคา ฿{price:,}"
    if discount > 0:
        alt_text += f" (ลด {discount}%)"
    
    builder.alt_text(alt_text).size("mega")
    
    if product.get('image_url'):
        builder.hero_image(
            product['image_url'],
            action={"type": "uri", "uri": product.get('url', 'https://example.com')}
        )
    
    if discount > 0:
        builder.header(f"🔥 สินค้าแนะนำ", badge=f"ลด {discount}%", 
                      background_color="#FFF3F0", text_color="#FF5722", 
                      badge_color="#FF5722")
    
    builder.body(body_contents)
    
    footer_contents = [
        fc.box("horizontal", [
            fc.button("ดูรายละเอียด", 
                     {"type": "uri", "uri": product.get('url', 'https://example.com')},
                     "secondary", flex=1),
            fc.button("🛒 ซื้อเลย",
                     {"type": "postback", "data": f"action=buy&id={product.get('id', '')}",
                      "displayText": f"เพิ่ม {product.get('name', '')} ลงตะกร้า"},
                     "primary", color="#FF5722", flex=1)
        ], layout="horizontal", spacing="sm")
    ]
    
    builder.footer(footer_contents)
    builder.styles(footer={"separator": True, "separatorColor": "#eeeeee"})
    
    return builder.build()


# Usage example
if __name__ == '__main__':
    product = {
        'id': 'P001',
        'name': 'Nike Air Max 90',
        'category': 'รองเท้า',
        'description': 'รองเท้าวิ่งรุ่นยอดนิยม',
        'price': 3500,
        'original_price': 4500,
        'image_url': None,
        'url': 'https://example.com/products/P001'
    }
    
    flex_msg = create_product_card_python(product)
    print(json.dumps(flex_msg, ensure_ascii=False, indent=2))
```

---

## 11. Testing Strategies

### Unit Tests (Node.js)

```javascript
// tests/flex/productCard.test.js
const { createProductCard, createProductCarousel } = require('../../src/flex/templates/productCard');

describe('ProductCard Template', () => {
  const mockProduct = {
    id: 'P001',
    name: 'Test Product',
    price: 100,
    originalPrice: 150,
    category: 'Test',
    rating: 4.5,
    reviewCount: 100,
    imageUrl: 'https://example.com/image.jpg',
    url: 'https://example.com/products/P001',
    inStock: true,
  };

  describe('createProductCard', () => {
    it('should return valid Flex Message structure', () => {
      const result = createProductCard(mockProduct);
      
      expect(result.type).toBe('flex');
      expect(result.altText).toContain('Test Product');
      expect(result.contents.type).toBe('bubble');
    });

    it('should show discount badge when originalPrice > price', () => {
      const result = createProductCard(mockProduct);
      expect(result.altText).toContain('33%'); // 33% discount
    });

    it('should handle missing optional fields', () => {
      const minimalProduct = { id: 'P002', name: 'Minimal', price: 50 };
      expect(() => createProductCard(minimalProduct)).not.toThrow();
    });

    it('should throw when required fields missing', () => {
      expect(() => createProductCard({ id: 'P003' })).toThrow();
      expect(() => createProductCard({ name: 'Test' })).toThrow();
    });

    it('should show out-of-stock indicator', () => {
      const outOfStock = { ...mockProduct, inStock: false };
      const result = createProductCard(outOfStock);
      const bodyText = JSON.stringify(result);
      expect(bodyText).toContain('หมด');
    });
  });

  describe('createProductCarousel', () => {
    it('should create carousel with multiple products', () => {
      const products = [mockProduct, { ...mockProduct, id: 'P002', name: 'Product 2' }];
      const result = createProductCarousel(products);
      
      expect(result.contents.type).toBe('carousel');
      expect(result.contents.contents.length).toBe(2);
    });

    it('should return text message for empty array', () => {
      const result = createProductCarousel([]);
      expect(result.type).toBe('text');
    });

    it('should add view all card when showViewAll and enough products', () => {
      const products = Array(10).fill(mockProduct).map((p, i) => ({ ...p, id: `P${i}` }));
      const result = createProductCarousel(products, { showViewAll: true, maxCards: 10 });
      
      // Should have 9 products + 1 view all = 10 total
      expect(result.contents.contents.length).toBeLessThanOrEqual(10);
    });
  });
});
```

### Validation Helper

```javascript
// src/flex/validator.js
function validateFlexMessage(message) {
  const errors = [];

  if (!message) {
    errors.push('Message is null or undefined');
    return errors;
  }

  if (message.type !== 'flex') {
    errors.push(`Invalid type: ${message.type} (expected "flex")`);
  }

  if (!message.altText) {
    errors.push('Missing altText');
  } else if (message.altText.length > 400) {
    errors.push(`altText too long: ${message.altText.length} chars (max 400)`);
  }

  if (!message.contents) {
    errors.push('Missing contents');
  } else {
    const containerType = message.contents.type;
    if (!['bubble', 'carousel'].includes(containerType)) {
      errors.push(`Invalid container type: ${containerType}`);
    }

    if (containerType === 'carousel') {
      const items = message.contents.contents;
      if (!Array.isArray(items) || items.length === 0) {
        errors.push('Carousel contents must be a non-empty array');
      } else if (items.length > 12) {
        errors.push(`Carousel has too many items: ${items.length} (max 12)`);
      }
    }
  }

  return errors;
}

function assertValidFlexMessage(message) {
  const errors = validateFlexMessage(message);
  if (errors.length > 0) {
    throw new Error(`Invalid Flex Message:\n${errors.join('\n')}`);
  }
  return true;
}

module.exports = { validateFlexMessage, assertValidFlexMessage };
```

---

## 12. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Price Alert Card (ง่าย)

สร้าง notification card สำหรับแจ้งเตือนราคาสินค้าลดลง:
- แสดงชื่อสินค้า
- ราคาเดิม vs ราคาใหม่
- เปอร์เซ็นต์ที่ลด
- ปุ่ม "ซื้อเลย" ก่อนหมดโปรโมชั่น

### แบบฝึกหัดที่ 2: Survey Card (ปานกลาง)

สร้าง survey card ด้วย FlexMessageBuilder ที่:
- แสดงคำถาม 1 ข้อ
- มีตัวเลือก 4 ข้อ (ใช้ button + postback)
- Progress bar แสดงความคืบหน้า (เช่น "คำถามที่ 3/10")
- Timer countdown

### แบบฝึกหัดที่ 3: Complete E-Commerce Flow (ยาก)

สร้างระบบครบวงจรที่:
1. ผู้ใช้พิมพ์ชื่อสินค้า
2. ค้นหาจาก mock database
3. แสดง carousel ผลการค้นหา
4. กด "ซื้อเลย" → แสดง confirmation card
5. กด "ยืนยัน" → แสดง receipt bubble
6. มีปุ่ม "ติดตามพัสดุ" ใน receipt

---

## สรุป

ใน Part นี้เราได้เรียนรู้การสร้าง Dynamic Flex Messages:

1. **Design Principles** สำหรับ programmatic Flex generation
2. **Component Factory Functions** แบบ reusable (header, hero, body, footer)
3. **Database Integration** ด้วย Repository pattern
4. **External API Integration** พร้อม error handling
5. **Template Functions**: Product Card, Receipt, Notification, Error, Success
6. **FlexMessageBuilder** utility class (Node.js + Python)
7. **Testing Strategies** สำหรับ Flex templates

### Key Takeaways

```
💡 แยก data layer ออกจาก template layer เสมอ
💡 ใช้ default values สำหรับทุก optional field
💡 Validate input ก่อนสร้าง Flex Message
💡 Test templates ด้วย unit tests
💡 ใช้ Builder pattern สำหรับ complex messages
💡 Cache Flex JSON ที่สร้างแล้วเมื่อเป็นไปได้
💡 ทดสอบใน Simulator ก่อน deploy เสมอ
```

---

## Resources

- [Flex Message Official Documentation](https://developers.line.biz/en/docs/messaging-api/flex-message-elements/)
- [Flex Message Simulator](https://developers.line.biz/flex-simulator/)
- [LINE Bot SDK Node.js](https://github.com/line/line-bot-sdk-nodejs)
- [LINE Bot SDK Python](https://github.com/line/line-bot-sdk-python)
- [Jest Testing Framework](https://jestjs.io/)
- [Pytest Python Testing](https://pytest.org/)
