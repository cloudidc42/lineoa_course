# Part 34: Flex Carousel Container

## สารบัญ

1. [Carousel Container Structure](#carousel-container-structure)
2. [Creating Multi-Card Carousel](#creating-multi-card-carousel)
3. [Card Navigation](#card-navigation)
4. [Consistent Card Sizing](#consistent-card-sizing)
5. [Mixed Content Carousels](#mixed-content-carousels)
6. [Use Case: Product Showcase](#product-showcase)
7. [Use Case: Menu Cards](#menu-cards)
8. [Use Case: News Articles](#news-articles)
9. [Use Case: Event Listing](#event-listing)
10. [Complete Node.js Examples](#nodejs-examples)
11. [Complete Python Examples](#python-examples)
12. [Best Practices](#best-practices)
13. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. Carousel Container Structure

Carousel เป็น container ที่รวม Bubble หลายอันเข้าด้วยกัน ทำให้ผู้ใช้สามารถ scroll ดูได้

### โครงสร้างพื้นฐาน

```json
{
  "type": "flex",
  "altText": "ข้อความอธิบาย carousel",
  "contents": {
    "type": "carousel",
    "contents": [
      { "type": "bubble", ... },
      { "type": "bubble", ... },
      { "type": "bubble", ... }
    ]
  }
}
```

### Properties ของ Carousel

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| type | string | ✅ | - | "carousel" |
| contents | array | ✅ | - | array ของ bubble objects |

> **หมายเหตุ:** Carousel ไม่มี styles property เหมือน bubble

### ข้อจำกัด Carousel

```
สูงสุด 12 bubbles ต่อ carousel
ทุก bubble ควรมีขนาด (size) เท่ากัน
ทุก bubble ควรมี hero ถ้าตัวใดตัวหนึ่งมี
Carousel ไม่รองรับ giga size bubble
```

### วิธีการแสดงผล

```
Mobile:
┌────────┐ ┌────────┐ ┌────────┐
│ Card 1 │ │ Card 2 │ │ Card 3 │
│        │ │        │ │        │
└────────┘ └────────┘ └────────┘
          ←  scroll  →

โปรดทราบ: แสดงทีละ 1-2 cards ตามขนาดหน้าจอ
```

---

## 2. Creating Multi-Card Carousel

### ตัวอย่าง Carousel 3 cards พื้นฐาน

```json
{
  "type": "flex",
  "altText": "ดูสินค้าแนะนำ 3 รายการ",
  "contents": {
    "type": "carousel",
    "contents": [
      {
        "type": "bubble",
        "size": "mega",
        "hero": {
          "type": "image",
          "url": "https://via.placeholder.com/400x260/FF5722/FFFFFF?text=Product+1",
          "size": "full",
          "aspectRatio": "20:13",
          "aspectMode": "cover"
        },
        "body": {
          "type": "box",
          "layout": "vertical",
          "spacing": "sm",
          "paddingAll": "16px",
          "contents": [
            {
              "type": "text",
              "text": "สินค้า A",
              "weight": "bold",
              "size": "lg"
            },
            {
              "type": "text",
              "text": "฿299",
              "weight": "bold",
              "color": "#FF5722",
              "size": "xl"
            }
          ]
        },
        "footer": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "12px",
          "contents": [
            {
              "type": "button",
              "style": "primary",
              "height": "sm",
              "action": {
                "type": "postback",
                "label": "ซื้อเลย",
                "data": "action=buy&id=A"
              }
            }
          ]
        }
      },
      {
        "type": "bubble",
        "size": "mega",
        "hero": {
          "type": "image",
          "url": "https://via.placeholder.com/400x260/2196F3/FFFFFF?text=Product+2",
          "size": "full",
          "aspectRatio": "20:13",
          "aspectMode": "cover"
        },
        "body": {
          "type": "box",
          "layout": "vertical",
          "spacing": "sm",
          "paddingAll": "16px",
          "contents": [
            {
              "type": "text",
              "text": "สินค้า B",
              "weight": "bold",
              "size": "lg"
            },
            {
              "type": "text",
              "text": "฿499",
              "weight": "bold",
              "color": "#FF5722",
              "size": "xl"
            }
          ]
        },
        "footer": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "12px",
          "contents": [
            {
              "type": "button",
              "style": "primary",
              "height": "sm",
              "action": {
                "type": "postback",
                "label": "ซื้อเลย",
                "data": "action=buy&id=B"
              }
            }
          ]
        }
      },
      {
        "type": "bubble",
        "size": "mega",
        "hero": {
          "type": "image",
          "url": "https://via.placeholder.com/400x260/4CAF50/FFFFFF?text=Product+3",
          "size": "full",
          "aspectRatio": "20:13",
          "aspectMode": "cover"
        },
        "body": {
          "type": "box",
          "layout": "vertical",
          "spacing": "sm",
          "paddingAll": "16px",
          "contents": [
            {
              "type": "text",
              "text": "สินค้า C",
              "weight": "bold",
              "size": "lg"
            },
            {
              "type": "text",
              "text": "฿699",
              "weight": "bold",
              "color": "#FF5722",
              "size": "xl"
            }
          ]
        },
        "footer": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "12px",
          "contents": [
            {
              "type": "button",
              "style": "primary",
              "height": "sm",
              "action": {
                "type": "postback",
                "label": "ซื้อเลย",
                "data": "action=buy&id=C"
              }
            }
          ]
        }
      }
    ]
  }
}
```

### สร้าง Carousel แบบ Dynamic (Node.js)

```javascript
function createCarousel(items) {
  const bubbles = items.map(item => ({
    type: 'bubble',
    size: 'mega',
    hero: {
      type: 'image',
      url: item.imageUrl,
      size: 'full',
      aspectRatio: '20:13',
      aspectMode: 'cover',
    },
    body: {
      type: 'box',
      layout: 'vertical',
      spacing: 'sm',
      paddingAll: '16px',
      contents: [
        { type: 'text', text: item.name, weight: 'bold', size: 'lg' },
        { type: 'text', text: `฿${item.price.toLocaleString()}`, color: '#FF5722', size: 'xl', weight: 'bold' },
      ],
    },
    footer: {
      type: 'box',
      layout: 'vertical',
      paddingAll: '12px',
      contents: [{
        type: 'button',
        style: 'primary',
        height: 'sm',
        action: { type: 'postback', label: 'ซื้อเลย', data: `action=buy&id=${item.id}` },
      }],
    },
  }));

  return {
    type: 'flex',
    altText: `สินค้าแนะนำ ${items.length} รายการ`,
    contents: {
      type: 'carousel',
      contents: bubbles,
    },
  };
}
```

---

## 3. Card Navigation

การนำทางระหว่าง cards ใน carousel ผ่าน actions

### Navigation ด้วย Postback

```json
// ใน bubble แรก: ปุ่มไปหน้าถัดไป
{
  "type": "button",
  "style": "secondary",
  "height": "sm",
  "action": {
    "type": "postback",
    "label": "ดูเพิ่มเติม →",
    "data": "action=nextPage&page=2&category=shoes"
  }
}
```

### Navigation ด้วย Postback Handler (Node.js)

```javascript
async function handlePostback(event) {
  const data = new URLSearchParams(event.postback.data);
  const action = data.get('action');
  
  if (action === 'nextPage') {
    const page = parseInt(data.get('page'));
    const category = data.get('category');
    
    // ดึงข้อมูลหน้าถัดไปจาก database
    const products = await getProductsByPage(category, page, 5);
    
    if (products.length === 0) {
      return client.replyMessage(event.replyToken, {
        type: 'text',
        text: 'ไม่มีสินค้าเพิ่มเติมแล้ว',
      });
    }
    
    const carousel = createProductCarousel(products, page);
    return client.replyMessage(event.replyToken, carousel);
  }
}
```

### Navigation ด้วย URL Action

```json
// ไปหน้า product detail
{
  "type": "button",
  "style": "link",
  "action": {
    "type": "uri",
    "label": "ดูรายละเอียดทั้งหมด",
    "uri": "https://example.com/products?category=shoes"
  }
}
```

---

## 4. Consistent Card Sizing

การทำให้ทุก card ใน carousel มีขนาดเท่ากันเป็นสิ่งสำคัญสำหรับ UI ที่สวยงาม

### ทำไม Consistent Sizing จึงสำคัญ?

```
❌ ไม่ consistent:
┌────────┐ ┌──────────────┐ ┌──────┐
│        │ │              │ │      │
│ Card 1 │ │   Card 2     │ │Card 3│
│        │ │  (สูงกว่า)   │ │      │
└────────┘ └──────────────┘ └──────┘

✅ Consistent:
┌────────┐ ┌────────┐ ┌────────┐
│        │ │        │ │        │
│ Card 1 │ │ Card 2 │ │ Card 3 │
│        │ │        │ │        │
└────────┘ └────────┘ └────────┘
```

### เทคนิค: ใช้ height ที่แน่นอน

```json
// กำหนด height ให้ body box
{
  "type": "box",
  "layout": "vertical",
  "height": "120px",  // ← กำหนดความสูงที่แน่นอน
  "contents": [
    {
      "type": "text",
      "text": "เนื้อหาที่อาจมีความยาวต่างกัน",
      "wrap": true,
      "maxLines": 3     // ← จำกัดบรรทัด
    }
  ]
}
```

### เทคนิค: Fixed hero ratio

```json
// ทุก bubble ใช้ aspectRatio เดียวกัน
{
  "type": "image",
  "url": "...",
  "size": "full",
  "aspectRatio": "20:13",  // ← ทุก card ใช้ ratio นี้เหมือนกัน
  "aspectMode": "cover"
}
```

### เทคนิค: Consistent content structure

```javascript
// Node.js: สร้าง bubble template ที่ consistent
function createProductBubble(product) {
  // กำหนด description ให้มีความยาวไม่เกิน 80 ตัวอักษร
  const description = product.description.length > 80
    ? product.description.substring(0, 77) + '...'
    : product.description;

  return {
    type: 'bubble',
    size: 'mega',
    hero: {
      type: 'image',
      url: product.imageUrl || 'https://via.placeholder.com/400x260',
      size: 'full',
      aspectRatio: '20:13',   // consistent
      aspectMode: 'cover',
    },
    body: {
      type: 'box',
      layout: 'vertical',
      paddingAll: '16px',
      height: '160px',          // consistent height
      contents: [
        {
          type: 'text',
          text: product.name,
          weight: 'bold',
          size: 'md',
          maxLines: 2,           // จำกัด 2 บรรทัด
          wrap: true,
        },
        {
          type: 'text',
          text: description,
          size: 'sm',
          color: '#666666',
          wrap: true,
          maxLines: 2,           // จำกัด 2 บรรทัด
          margin: 'sm',
        },
        {
          type: 'filler',        // เติมพื้นที่ว่างถ้าข้อความสั้น
        },
        {
          type: 'text',
          text: `฿${product.price.toLocaleString()}`,
          weight: 'bold',
          color: '#FF5722',
          size: 'xl',
        },
      ],
    },
    footer: {
      type: 'box',
      layout: 'vertical',
      paddingAll: '12px',
      contents: [
        {
          type: 'button',
          style: 'primary',
          height: 'sm',
          action: {
            type: 'postback',
            label: 'เพิ่มลงตะกร้า',
            data: `action=addCart&id=${product.id}`,
          },
        },
      ],
    },
  };
}
```

---

## 5. Mixed Content Carousels

Carousel ที่มี cards หลายประเภท เช่น บาง card เป็นสินค้า บางเป็น "ดูทั้งหมด"

### Pattern: Product List + View All Card

```json
{
  "type": "flex",
  "altText": "สินค้าแนะนำ",
  "contents": {
    "type": "carousel",
    "contents": [
      {
        "type": "bubble",
        "size": "mega",
        "hero": { ... },
        "body": { ... },
        "footer": { ... }
      },
      {
        "type": "bubble",
        "size": "mega",
        "hero": { ... },
        "body": { ... },
        "footer": { ... }
      },
      {
        "type": "bubble",
        "size": "mega",
        "body": {
          "type": "box",
          "layout": "vertical",
          "justifyContent": "center",
          "alignItems": "center",
          "height": "200px",
          "contents": [
            {
              "type": "text",
              "text": "→",
              "size": "5xl",
              "color": "#cccccc",
              "align": "center"
            },
            {
              "type": "text",
              "text": "ดูสินค้าทั้งหมด",
              "size": "md",
              "color": "#888888",
              "align": "center",
              "margin": "md"
            }
          ]
        },
        "footer": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "12px",
          "contents": [
            {
              "type": "button",
              "style": "secondary",
              "action": {
                "type": "uri",
                "label": "ดูทั้งหมด",
                "uri": "https://example.com/products"
              }
            }
          ]
        }
      }
    ]
  }
}
```

---

## 6. Use Case: Product Showcase

ตัวอย่าง carousel สำหรับแสดงสินค้าแนะนำ

```json
{
  "type": "flex",
  "altText": "สินค้าแนะนำประจำสัปดาห์",
  "contents": {
    "type": "carousel",
    "contents": [
      {
        "type": "bubble",
        "size": "mega",
        "header": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "10px 16px",
          "contents": [
            {
              "type": "text",
              "text": "🔥 ขายดีอันดับ 1",
              "size": "xs",
              "color": "#FF5722",
              "weight": "bold"
            }
          ]
        },
        "hero": {
          "type": "image",
          "url": "https://via.placeholder.com/400x260/FF5722/FFFFFF?text=Top+Seller",
          "size": "full",
          "aspectRatio": "20:13",
          "aspectMode": "cover",
          "action": {
            "type": "uri",
            "uri": "https://example.com/products/001"
          }
        },
        "body": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "16px",
          "spacing": "sm",
          "contents": [
            {
              "type": "text",
              "text": "Nike Air Max 90",
              "weight": "bold",
              "size": "lg",
              "maxLines": 1
            },
            {
              "type": "box",
              "layout": "baseline",
              "spacing": "xs",
              "contents": [
                { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                { "type": "text", "text": "5.0 (2,341)", "size": "xs", "color": "#aaaaaa", "margin": "xs" }
              ]
            },
            {
              "type": "box",
              "layout": "horizontal",
              "margin": "sm",
              "contents": [
                {
                  "type": "text",
                  "text": "฿3,500",
                  "weight": "bold",
                  "color": "#FF5722",
                  "size": "xl",
                  "flex": 1
                },
                {
                  "type": "text",
                  "text": "฿4,500",
                  "color": "#aaaaaa",
                  "decoration": "line-through",
                  "size": "sm",
                  "gravity": "bottom",
                  "flex": 0
                }
              ]
            }
          ]
        },
        "footer": {
          "type": "box",
          "layout": "horizontal",
          "spacing": "sm",
          "paddingAll": "12px",
          "contents": [
            {
              "type": "button",
              "style": "secondary",
              "height": "sm",
              "flex": 1,
              "action": {
                "type": "uri",
                "label": "ดูรายละเอียด",
                "uri": "https://example.com/products/001"
              }
            },
            {
              "type": "button",
              "style": "primary",
              "height": "sm",
              "flex": 1,
              "color": "#FF5722",
              "action": {
                "type": "postback",
                "label": "🛒 ซื้อเลย",
                "data": "action=buy&id=001"
              }
            }
          ]
        },
        "styles": {
          "header": { "backgroundColor": "#FFF3F0" }
        }
      },
      {
        "type": "bubble",
        "size": "mega",
        "header": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "10px 16px",
          "contents": [
            {
              "type": "text",
              "text": "💎 พรีเมียม",
              "size": "xs",
              "color": "#9C27B0",
              "weight": "bold"
            }
          ]
        },
        "hero": {
          "type": "image",
          "url": "https://via.placeholder.com/400x260/9C27B0/FFFFFF?text=Premium",
          "size": "full",
          "aspectRatio": "20:13",
          "aspectMode": "cover",
          "action": {
            "type": "uri",
            "uri": "https://example.com/products/002"
          }
        },
        "body": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "16px",
          "spacing": "sm",
          "contents": [
            {
              "type": "text",
              "text": "Adidas Ultraboost 22",
              "weight": "bold",
              "size": "lg",
              "maxLines": 1
            },
            {
              "type": "box",
              "layout": "baseline",
              "spacing": "xs",
              "contents": [
                { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png", "size": "xs" },
                { "type": "icon", "url": "https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gray_star_28.png", "size": "xs" },
                { "type": "text", "text": "4.0 (1,567)", "size": "xs", "color": "#aaaaaa", "margin": "xs" }
              ]
            },
            {
              "type": "text",
              "text": "฿6,990",
              "weight": "bold",
              "color": "#9C27B0",
              "size": "xl",
              "margin": "sm"
            }
          ]
        },
        "footer": {
          "type": "box",
          "layout": "horizontal",
          "spacing": "sm",
          "paddingAll": "12px",
          "contents": [
            {
              "type": "button",
              "style": "secondary",
              "height": "sm",
              "flex": 1,
              "action": {
                "type": "uri",
                "label": "ดูรายละเอียด",
                "uri": "https://example.com/products/002"
              }
            },
            {
              "type": "button",
              "style": "primary",
              "height": "sm",
              "flex": 1,
              "color": "#9C27B0",
              "action": {
                "type": "postback",
                "label": "🛒 ซื้อเลย",
                "data": "action=buy&id=002"
              }
            }
          ]
        },
        "styles": {
          "header": { "backgroundColor": "#F3E5F5" }
        }
      }
    ]
  }
}
```

---

## 7. Use Case: Menu Cards

ตัวอย่าง carousel สำหรับเมนูอาหาร

```json
{
  "type": "flex",
  "altText": "เมนูอาหารแนะนำ",
  "contents": {
    "type": "carousel",
    "contents": [
      {
        "type": "bubble",
        "size": "mega",
        "hero": {
          "type": "image",
          "url": "https://via.placeholder.com/400x300/FF7043/FFFFFF?text=Coffee",
          "size": "full",
          "aspectRatio": "4:3",
          "aspectMode": "cover"
        },
        "body": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "16px",
          "spacing": "xs",
          "contents": [
            {
              "type": "box",
              "layout": "horizontal",
              "contents": [
                {
                  "type": "text",
                  "text": "กาแฟลาเต้",
                  "weight": "bold",
                  "size": "lg",
                  "flex": 1
                },
                {
                  "type": "box",
                  "layout": "vertical",
                  "flex": 0,
                  "paddingAll": "2px 8px",
                  "backgroundColor": "#FF7043",
                  "cornerRadius": "10px",
                  "contents": [
                    {
                      "type": "text",
                      "text": "HOT",
                      "size": "xxs",
                      "color": "#FFFFFF",
                      "weight": "bold"
                    }
                  ]
                }
              ]
            },
            {
              "type": "text",
              "text": "กาแฟเอสเปรสโซ่ผสมนมสด รสชาติเข้มข้นกลมกล่อม",
              "size": "sm",
              "color": "#666666",
              "wrap": true,
              "maxLines": 2
            },
            {
              "type": "box",
              "layout": "horizontal",
              "margin": "sm",
              "alignItems": "center",
              "contents": [
                {
                  "type": "text",
                  "text": "🌶️",
                  "size": "sm",
                  "flex": 0
                },
                {
                  "type": "text",
                  "text": "ไม่เผ็ด",
                  "size": "xs",
                  "color": "#888888",
                  "flex": 1,
                  "margin": "xs"
                },
                {
                  "type": "text",
                  "text": "⏱️ 5 นาที",
                  "size": "xs",
                  "color": "#888888",
                  "flex": 0
                }
              ]
            },
            {
              "type": "box",
              "layout": "horizontal",
              "margin": "md",
              "alignItems": "center",
              "contents": [
                {
                  "type": "text",
                  "text": "฿65",
                  "weight": "bold",
                  "color": "#FF7043",
                  "size": "xl",
                  "flex": 1
                },
                {
                  "type": "box",
                  "layout": "horizontal",
                  "flex": 0,
                  "spacing": "sm",
                  "contents": [
                    {
                      "type": "button",
                      "style": "secondary",
                      "height": "sm",
                      "flex": 0,
                      "action": {
                        "type": "postback",
                        "label": "−",
                        "data": "action=decrease&item=latte"
                      }
                    },
                    {
                      "type": "text",
                      "text": "1",
                      "size": "md",
                      "weight": "bold",
                      "flex": 0,
                      "gravity": "center",
                      "margin": "sm"
                    },
                    {
                      "type": "button",
                      "style": "primary",
                      "height": "sm",
                      "flex": 0,
                      "color": "#FF7043",
                      "action": {
                        "type": "postback",
                        "label": "+",
                        "data": "action=increase&item=latte"
                      }
                    }
                  ]
                }
              ]
            }
          ]
        }
      },
      {
        "type": "bubble",
        "size": "mega",
        "hero": {
          "type": "image",
          "url": "https://via.placeholder.com/400x300/8BC34A/FFFFFF?text=Salad",
          "size": "full",
          "aspectRatio": "4:3",
          "aspectMode": "cover"
        },
        "body": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "16px",
          "spacing": "xs",
          "contents": [
            {
              "type": "box",
              "layout": "horizontal",
              "contents": [
                {
                  "type": "text",
                  "text": "สลัดผัก",
                  "weight": "bold",
                  "size": "lg",
                  "flex": 1
                },
                {
                  "type": "box",
                  "layout": "vertical",
                  "flex": 0,
                  "paddingAll": "2px 8px",
                  "backgroundColor": "#8BC34A",
                  "cornerRadius": "10px",
                  "contents": [
                    {
                      "type": "text",
                      "text": "VEGAN",
                      "size": "xxs",
                      "color": "#FFFFFF",
                      "weight": "bold"
                    }
                  ]
                }
              ]
            },
            {
              "type": "text",
              "text": "ผักสดหลากสี ซอส Caesar น้ำมันมะกอก",
              "size": "sm",
              "color": "#666666",
              "wrap": true,
              "maxLines": 2
            },
            {
              "type": "box",
              "layout": "horizontal",
              "margin": "sm",
              "alignItems": "center",
              "contents": [
                {
                  "type": "text",
                  "text": "🥗",
                  "size": "sm",
                  "flex": 0
                },
                {
                  "type": "text",
                  "text": "Low Calorie",
                  "size": "xs",
                  "color": "#888888",
                  "flex": 1,
                  "margin": "xs"
                },
                {
                  "type": "text",
                  "text": "⏱️ 3 นาที",
                  "size": "xs",
                  "color": "#888888",
                  "flex": 0
                }
              ]
            },
            {
              "type": "text",
              "text": "฿120",
              "weight": "bold",
              "color": "#8BC34A",
              "size": "xl",
              "margin": "md"
            }
          ]
        },
        "footer": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "12px",
          "contents": [
            {
              "type": "button",
              "style": "primary",
              "height": "sm",
              "color": "#8BC34A",
              "action": {
                "type": "postback",
                "label": "🛒 สั่งเดี๋ยวนี้",
                "data": "action=order&item=salad"
              }
            }
          ]
        }
      }
    ]
  }
}
```

---

## 8. Use Case: News Articles

ตัวอย่าง carousel สำหรับแสดงข่าว

```json
{
  "type": "flex",
  "altText": "ข่าวประชาสัมพันธ์ล่าสุด",
  "contents": {
    "type": "carousel",
    "contents": [
      {
        "type": "bubble",
        "size": "mega",
        "hero": {
          "type": "image",
          "url": "https://via.placeholder.com/400x200/1565C0/FFFFFF?text=Tech+News",
          "size": "full",
          "aspectRatio": "20:10",
          "aspectMode": "cover",
          "action": {
            "type": "uri",
            "uri": "https://example.com/news/001"
          }
        },
        "body": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "16px",
          "spacing": "sm",
          "contents": [
            {
              "type": "box",
              "layout": "horizontal",
              "spacing": "sm",
              "contents": [
                {
                  "type": "box",
                  "layout": "vertical",
                  "flex": 0,
                  "paddingAll": "2px 8px",
                  "backgroundColor": "#1565C0",
                  "cornerRadius": "4px",
                  "contents": [
                    {
                      "type": "text",
                      "text": "เทคโนโลยี",
                      "size": "xxs",
                      "color": "#FFFFFF"
                    }
                  ]
                },
                {
                  "type": "text",
                  "text": "15 ม.ค. 2025",
                  "size": "xs",
                  "color": "#aaaaaa",
                  "gravity": "center"
                }
              ]
            },
            {
              "type": "text",
              "text": "LINE ประกาศ API ใหม่สำหรับนักพัฒนาในเอเชียตะวันออกเฉียงใต้",
              "weight": "bold",
              "size": "md",
              "wrap": true,
              "maxLines": 3
            },
            {
              "type": "text",
              "text": "LINE Corporation ได้ประกาศเปิดตัว API ชุดใหม่ที่จะช่วยให้นักพัฒนาสร้างประสบการณ์ใหม่...",
              "size": "sm",
              "color": "#666666",
              "wrap": true,
              "maxLines": 2
            },
            {
              "type": "box",
              "layout": "horizontal",
              "margin": "md",
              "contents": [
                {
                  "type": "image",
                  "url": "https://via.placeholder.com/30x30/888888/FFFFFF?text=A",
                  "size": "xxs",
                  "aspectRatio": "1:1",
                  "aspectMode": "cover",
                  "flex": 0
                },
                {
                  "type": "text",
                  "text": "สมชาย นักข่าว",
                  "size": "xs",
                  "color": "#888888",
                  "gravity": "center",
                  "margin": "sm",
                  "flex": 1
                },
                {
                  "type": "text",
                  "text": "5 นาทีอ่าน",
                  "size": "xs",
                  "color": "#aaaaaa",
                  "gravity": "center",
                  "flex": 0
                }
              ]
            }
          ]
        },
        "footer": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "12px",
          "contents": [
            {
              "type": "button",
              "style": "link",
              "action": {
                "type": "uri",
                "label": "อ่านต่อ →",
                "uri": "https://example.com/news/001"
              }
            }
          ]
        }
      }
    ]
  }
}
```

---

## 9. Use Case: Event Listing

ตัวอย่าง carousel สำหรับรายการ events

```json
{
  "type": "flex",
  "altText": "กิจกรรมที่น่าสนใจ",
  "contents": {
    "type": "carousel",
    "contents": [
      {
        "type": "bubble",
        "size": "mega",
        "hero": {
          "type": "box",
          "layout": "vertical",
          "contents": [
            {
              "type": "image",
              "url": "https://via.placeholder.com/400x200/3F51B5/FFFFFF?text=Workshop",
              "size": "full",
              "aspectRatio": "20:10",
              "aspectMode": "cover"
            }
          ]
        },
        "body": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "16px",
          "spacing": "sm",
          "contents": [
            {
              "type": "box",
              "layout": "horizontal",
              "spacing": "sm",
              "alignItems": "center",
              "contents": [
                {
                  "type": "box",
                  "layout": "vertical",
                  "flex": 0,
                  "paddingAll": "6px 10px",
                  "backgroundColor": "#E8EAF6",
                  "cornerRadius": "6px",
                  "alignItems": "center",
                  "contents": [
                    {
                      "type": "text",
                      "text": "20",
                      "weight": "bold",
                      "size": "lg",
                      "color": "#3F51B5",
                      "align": "center"
                    },
                    {
                      "type": "text",
                      "text": "ม.ค.",
                      "size": "xs",
                      "color": "#3F51B5",
                      "align": "center"
                    }
                  ]
                },
                {
                  "type": "box",
                  "layout": "vertical",
                  "flex": 1,
                  "contents": [
                    {
                      "type": "text",
                      "text": "LINE Bot Workshop",
                      "weight": "bold",
                      "size": "md",
                      "maxLines": 1
                    },
                    {
                      "type": "text",
                      "text": "📍 Central World, Bangkok",
                      "size": "xs",
                      "color": "#666666"
                    }
                  ]
                }
              ]
            },
            {
              "type": "text",
              "text": "เรียนรู้การสร้าง LINE Bot ตั้งแต่เริ่มต้น พร้อม hands-on workshop",
              "size": "sm",
              "color": "#555555",
              "wrap": true,
              "maxLines": 2
            },
            {
              "type": "box",
              "layout": "horizontal",
              "margin": "sm",
              "contents": [
                {
                  "type": "text",
                  "text": "⏰ 09:00 - 17:00",
                  "size": "xs",
                  "color": "#888888",
                  "flex": 1
                },
                {
                  "type": "text",
                  "text": "👥 45/60 คน",
                  "size": "xs",
                  "color": "#888888",
                  "flex": 0
                }
              ]
            },
            {
              "type": "box",
              "layout": "horizontal",
              "margin": "sm",
              "contents": [
                {
                  "type": "text",
                  "text": "💰 ฟรี!",
                  "size": "sm",
                  "color": "#4CAF50",
                  "weight": "bold",
                  "flex": 1
                },
                {
                  "type": "box",
                  "layout": "vertical",
                  "flex": 0,
                  "paddingAll": "2px 8px",
                  "backgroundColor": "#FFEB3B",
                  "cornerRadius": "4px",
                  "contents": [
                    {
                      "type": "text",
                      "text": "เหลือ 15 ที่!",
                      "size": "xxs",
                      "color": "#333333",
                      "weight": "bold"
                    }
                  ]
                }
              ]
            }
          ]
        },
        "footer": {
          "type": "box",
          "layout": "vertical",
          "paddingAll": "12px",
          "contents": [
            {
              "type": "button",
              "style": "primary",
              "color": "#3F51B5",
              "height": "sm",
              "action": {
                "type": "uri",
                "label": "📝 สมัครเข้าร่วม",
                "uri": "https://example.com/events/workshop/register"
              }
            }
          ]
        }
      }
    ]
  }
}
```

---

## 10. Complete Node.js Examples

### ตัวอย่างสมบูรณ์: Product Carousel with Database

```javascript
// product-carousel.js
const line = require('@line/bot-sdk');

const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET,
};
const client = new line.Client(config);

/**
 * สร้าง product bubble
 */
function createProductBubble(product, index) {
  const discount = product.originalPrice
    ? Math.round(((product.originalPrice - product.price) / product.originalPrice) * 100)
    : 0;

  return {
    type: 'bubble',
    size: 'mega',
    header: discount > 0
      ? {
          type: 'box',
          layout: 'vertical',
          paddingAll: '8px 16px',
          contents: [{
            type: 'text',
            text: `🔥 ลด ${discount}%`,
            size: 'xs',
            color: '#FF5722',
            weight: 'bold',
          }],
        }
      : null,
    hero: {
      type: 'image',
      url: product.imageUrl || `https://via.placeholder.com/400x260/cccccc/ffffff?text=${encodeURIComponent(product.name)}`,
      size: 'full',
      aspectRatio: '20:13',
      aspectMode: 'cover',
      action: { type: 'uri', uri: product.url || 'https://example.com' },
    },
    body: {
      type: 'box',
      layout: 'vertical',
      paddingAll: '16px',
      spacing: 'sm',
      contents: [
        {
          type: 'text',
          text: product.name,
          weight: 'bold',
          size: 'lg',
          maxLines: 2,
          wrap: true,
        },
        {
          type: 'text',
          text: product.category,
          size: 'xs',
          color: '#999999',
        },
        {
          type: 'box',
          layout: 'baseline',
          spacing: 'xs',
          margin: 'xs',
          contents: [
            ...Array(Math.floor(product.rating)).fill({
              type: 'icon',
              url: 'https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png',
              size: 'xs',
            }),
            ...(product.rating % 1 >= 0.5 ? [{
              type: 'icon',
              url: 'https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gold_star_28.png',
              size: 'xs',
            }] : []),
            ...Array(5 - Math.ceil(product.rating)).fill({
              type: 'icon',
              url: 'https://scdn.line-apps.com/n/channel_devcenter/img/fx/review_gray_star_28.png',
              size: 'xs',
            }),
            {
              type: 'text',
              text: `${product.rating} (${product.reviewCount.toLocaleString()})`,
              size: 'xs',
              color: '#aaaaaa',
              margin: 'xs',
            },
          ],
        },
        {
          type: 'box',
          layout: 'horizontal',
          margin: 'md',
          alignItems: 'flex-end',
          contents: [
            {
              type: 'text',
              text: `฿${product.price.toLocaleString()}`,
              weight: 'bold',
              color: '#FF5722',
              size: 'xl',
              flex: 1,
            },
            ...(product.originalPrice ? [{
              type: 'text',
              text: `฿${product.originalPrice.toLocaleString()}`,
              color: '#aaaaaa',
              decoration: 'line-through',
              size: 'sm',
              flex: 0,
            }] : []),
          ],
        },
      ].filter(Boolean),
    },
    footer: {
      type: 'box',
      layout: 'horizontal',
      spacing: 'sm',
      paddingAll: '12px',
      contents: [
        {
          type: 'button',
          style: 'secondary',
          height: 'sm',
          flex: 1,
          action: {
            type: 'uri',
            label: 'ดูรายละเอียด',
            uri: product.url || 'https://example.com',
          },
        },
        {
          type: 'button',
          style: 'primary',
          height: 'sm',
          flex: 1,
          color: '#FF5722',
          action: {
            type: 'postback',
            label: '🛒 ซื้อเลย',
            data: `action=buy&id=${product.id}`,
            displayText: `เพิ่ม ${product.name} ลงตะกร้า`,
          },
        },
      ],
    },
    styles: {
      ...(discount > 0 ? { header: { backgroundColor: '#FFF3F0' } } : {}),
      footer: { separator: true, separatorColor: '#eeeeee' },
    },
  };
}

/**
 * สร้าง "View All" bubble
 */
function createViewAllBubble(category, totalCount) {
  return {
    type: 'bubble',
    size: 'mega',
    body: {
      type: 'box',
      layout: 'vertical',
      justifyContent: 'center',
      alignItems: 'center',
      paddingAll: '20px',
      spacing: 'md',
      contents: [
        {
          type: 'text',
          text: '→',
          size: '5xl',
          color: '#cccccc',
          align: 'center',
        },
        {
          type: 'text',
          text: `ดูสินค้าทั้งหมด`,
          size: 'md',
          color: '#888888',
          align: 'center',
          weight: 'bold',
        },
        {
          type: 'text',
          text: `${totalCount.toLocaleString()} รายการ`,
          size: 'sm',
          color: '#aaaaaa',
          align: 'center',
        },
      ],
    },
    footer: {
      type: 'box',
      layout: 'vertical',
      paddingAll: '12px',
      contents: [
        {
          type: 'button',
          style: 'secondary',
          action: {
            type: 'uri',
            label: `ดูทั้งหมด (${totalCount})`,
            uri: `https://example.com/products?category=${category}`,
          },
        },
      ],
    },
  };
}

/**
 * สร้าง Carousel จาก products array
 */
function createProductCarousel(products, options = {}) {
  const {
    showViewAll = true,
    totalCount = products.length,
    category = 'all',
    maxCards = 10,
  } = options;

  // จำกัดจำนวน product cards
  const productBubbles = products
    .slice(0, showViewAll ? maxCards - 1 : maxCards)
    .map((product, index) => createProductBubble(product, index));

  // เพิ่ม View All card
  const allBubbles = showViewAll && products.length >= maxCards
    ? [...productBubbles, createViewAllBubble(category, totalCount)]
    : productBubbles;

  return {
    type: 'flex',
    altText: `สินค้าแนะนำ ${products.length} รายการ`,
    contents: {
      type: 'carousel',
      contents: allBubbles,
    },
  };
}

// Express webhook handler
const express = require('express');
const app = express();

app.post('/webhook', line.middleware(config), async (req, res) => {
  await Promise.all(req.body.events.map(handleEvent));
  res.json({ success: true });
});

async function handleEvent(event) {
  if (event.type === 'message' && event.message.type === 'text') {
    return handleTextMessage(event);
  }
  if (event.type === 'postback') {
    return handlePostback(event);
  }
}

async function handleTextMessage(event) {
  const text = event.message.text.toLowerCase();

  if (text.includes('สินค้า') || text.includes('product')) {
    // Mock product data - ในการใช้งานจริงดึงจาก database
    const products = [
      {
        id: 'P001',
        name: 'Nike Air Max 90',
        category: 'รองเท้า',
        price: 3500,
        originalPrice: 4500,
        rating: 4.9,
        reviewCount: 2341,
        imageUrl: null,
        url: 'https://example.com/products/P001',
      },
      {
        id: 'P002',
        name: 'Adidas Ultraboost 22',
        category: 'รองเท้า',
        price: 6990,
        originalPrice: null,
        rating: 4.5,
        reviewCount: 1567,
        imageUrl: null,
        url: 'https://example.com/products/P002',
      },
      {
        id: 'P003',
        name: 'Puma RS-X3',
        category: 'รองเท้า',
        price: 2890,
        originalPrice: 3500,
        rating: 4.2,
        reviewCount: 890,
        imageUrl: null,
        url: 'https://example.com/products/P003',
      },
    ];

    const carousel = createProductCarousel(products, {
      showViewAll: true,
      totalCount: 25,
      category: 'shoes',
    });

    return client.replyMessage(event.replyToken, carousel);
  }
}

async function handlePostback(event) {
  const params = new URLSearchParams(event.postback.data);
  const action = params.get('action');
  const id = params.get('id');

  if (action === 'buy') {
    return client.replyMessage(event.replyToken, {
      type: 'text',
      text: `✅ เพิ่มสินค้า ${id} ลงตะกร้าแล้ว!\nพิมพ์ "ตะกร้า" เพื่อดูรายการสั่งซื้อ`,
    });
  }
}

app.listen(3000, () => console.log('Server running on port 3000'));
```

---

## 11. Complete Python Examples

### ตัวอย่างสมบูรณ์: Menu Carousel

```python
# menu_carousel.py
import os
from typing import List, Dict, Optional
from flask import Flask, request, abort
from linebot import LineBotApi, WebhookHandler
from linebot.models import (
    FlexSendMessage, TextSendMessage,
    MessageEvent, TextMessage, PostbackEvent
)

app = Flask(__name__)
line_bot_api = LineBotApi(os.environ.get('LINE_CHANNEL_ACCESS_TOKEN'))
handler = WebhookHandler(os.environ.get('LINE_CHANNEL_SECRET'))


def create_menu_bubble(item: Dict) -> Dict:
    """
    สร้าง menu bubble สำหรับแต่ละรายการอาหาร
    
    Args:
        item: dict ที่มี keys: id, name, category, description, price, 
              calories, prep_time, image_url, is_available, badge
    """
    
    badge_config = {
        'HOT': {'color': '#FF5722', 'text': '🔥 HOT'},
        'NEW': {'color': '#2196F3', 'text': '✨ NEW'},
        'BEST': {'color': '#FF9800', 'text': '⭐ BEST'},
        'VEGAN': {'color': '#4CAF50', 'text': '🌱 VEGAN'},
    }
    
    badge = item.get('badge')
    badge_conf = badge_config.get(badge)
    
    # สร้าง header ถ้ามี badge
    header = None
    if badge_conf:
        header = {
            "type": "box",
            "layout": "vertical",
            "paddingAll": "8px 16px",
            "contents": [{
                "type": "text",
                "text": badge_conf['text'],
                "size": "xs",
                "color": badge_conf['color'],
                "weight": "bold"
            }]
        }
    
    # Availability indicator
    availability_text = "✅ มีพร้อมเสิร์ฟ" if item.get('is_available', True) else "❌ หมดชั่วคราว"
    availability_color = "#4CAF50" if item.get('is_available', True) else "#F44336"
    
    bubble = {
        "type": "bubble",
        "size": "mega",
        "hero": {
            "type": "image",
            "url": item.get('image_url', f"https://via.placeholder.com/400x300/cccccc/ffffff?text={item['name']}"),
            "size": "full",
            "aspectRatio": "4:3",
            "aspectMode": "cover"
        },
        "body": {
            "type": "box",
            "layout": "vertical",
            "paddingAll": "16px",
            "spacing": "sm",
            "contents": [
                # ชื่อและ category
                {
                    "type": "text",
                    "text": item['name'],
                    "weight": "bold",
                    "size": "lg",
                    "maxLines": 1
                },
                {
                    "type": "text",
                    "text": item.get('category', 'เมนูทั่วไป'),
                    "size": "xs",
                    "color": "#999999"
                },
                # Description
                {
                    "type": "text",
                    "text": item.get('description', ''),
                    "size": "sm",
                    "color": "#666666",
                    "wrap": True,
                    "maxLines": 2
                },
                # Stats row
                {
                    "type": "box",
                    "layout": "horizontal",
                    "margin": "sm",
                    "contents": [
                        {
                            "type": "text",
                            "text": f"🔥 {item.get('calories', '-')} cal",
                            "size": "xs",
                            "color": "#888888",
                            "flex": 1
                        },
                        {
                            "type": "text",
                            "text": f"⏱️ {item.get('prep_time', '-')} นาที",
                            "size": "xs",
                            "color": "#888888",
                            "flex": 0
                        }
                    ]
                },
                # Availability
                {
                    "type": "text",
                    "text": availability_text,
                    "size": "xs",
                    "color": availability_color,
                    "margin": "xs"
                },
                # Price
                {
                    "type": "text",
                    "text": f"฿{item['price']:,}",
                    "weight": "bold",
                    "color": "#FF7043",
                    "size": "xl",
                    "margin": "sm"
                }
            ]
        },
        "footer": {
            "type": "box",
            "layout": "vertical",
            "paddingAll": "12px",
            "contents": [{
                "type": "button",
                "style": "primary" if item.get('is_available', True) else "secondary",
                "height": "sm",
                "color": "#FF7043" if item.get('is_available', True) else "#aaaaaa",
                "action": {
                    "type": "postback",
                    "label": "🛒 สั่งเลย" if item.get('is_available', True) else "❌ หมด",
                    "data": f"action=order&menuId={item['id']}" if item.get('is_available', True) else "action=outOfStock",
                    "displayText": f"สั่ง {item['name']} แล้ว" if item.get('is_available', True) else "รายการนี้หมดชั่วคราว"
                }
            }]
        }
    }
    
    if header:
        bubble['header'] = header
        # Add header style
        bubble['styles'] = {
            'header': {
                'backgroundColor': '#FFF8F6' if badge in ['HOT', 'BEST'] else '#F1F8E9'
            }
        }
    
    return bubble


def create_menu_carousel(menu_items: List[Dict]) -> FlexSendMessage:
    """
    สร้าง Carousel จากรายการเมนู
    
    Args:
        menu_items: list ของ menu item dicts
    
    Returns:
        FlexSendMessage
    """
    # จำกัดสูงสุด 12 items
    items_to_show = menu_items[:12]
    
    bubbles = [create_menu_bubble(item) for item in items_to_show]
    
    # นับ available items
    available_count = sum(1 for item in menu_items if item.get('is_available', True))
    
    return FlexSendMessage(
        alt_text=f"เมนูอาหาร {available_count}/{len(menu_items)} รายการพร้อมเสิร์ฟ",
        contents={
            "type": "carousel",
            "contents": bubbles
        }
    )


# Webhook
@app.route('/webhook', methods=['POST'])
def webhook():
    signature = request.headers.get('X-Line-Signature', '')
    body = request.get_data(as_text=True)
    
    try:
        handler.handle(body, signature)
    except Exception as e:
        print(f'Error: {e}')
        abort(400)
    
    return 'OK'


@handler.add(MessageEvent, message=TextMessage)
def handle_message(event):
    text = event.message.text.strip()
    
    if text in ['เมนู', 'menu', 'อาหาร']:
        # Mock menu data
        menu_items = [
            {
                'id': 'M001',
                'name': 'กาแฟลาเต้',
                'category': 'เครื่องดื่ม',
                'description': 'กาแฟเอสเปรสโซ่ผสมนมสด รสชาติเข้มข้นกลมกล่อม',
                'price': 65,
                'calories': 150,
                'prep_time': 5,
                'is_available': True,
                'badge': 'BEST',
                'image_url': None
            },
            {
                'id': 'M002',
                'name': 'สลัดผักสด',
                'category': 'อาหาร',
                'description': 'ผักสดหลากสี ซอส Caesar น้ำมันมะกอก Parmesan',
                'price': 120,
                'calories': 280,
                'prep_time': 3,
                'is_available': True,
                'badge': 'VEGAN',
                'image_url': None
            },
            {
                'id': 'M003',
                'name': 'เค้กช็อกโกแลต',
                'category': 'ขนม',
                'description': 'เค้กช็อกโกแลตเข้มข้น หน้าเนียน รสหวานพอดี',
                'price': 85,
                'calories': 320,
                'prep_time': 2,
                'is_available': False,
                'badge': None,
                'image_url': None
            },
            {
                'id': 'M004',
                'name': 'ชาเขียวมัทฉะ',
                'category': 'เครื่องดื่ม',
                'description': 'ชาเขียวมัทฉะจากญี่ปุ่น หอม นุ่ม กลมกล่อม',
                'price': 75,
                'calories': 120,
                'prep_time': 5,
                'is_available': True,
                'badge': 'NEW',
                'image_url': None
            }
        ]
        
        carousel = create_menu_carousel(menu_items)
        line_bot_api.reply_message(event.reply_token, carousel)
    else:
        line_bot_api.reply_message(
            event.reply_token,
            TextSendMessage(text='พิมพ์ "เมนู" เพื่อดูรายการอาหาร')
        )


@handler.add(PostbackEvent)
def handle_postback(event):
    data = dict(item.split('=') for item in event.postback.data.split('&'))
    action = data.get('action')
    
    if action == 'order':
        menu_id = data.get('menuId')
        line_bot_api.reply_message(
            event.reply_token,
            TextSendMessage(text=f'✅ รับออเดอร์ {menu_id} แล้ว!\nกรุณารอสักครู่...')
        )
    elif action == 'outOfStock':
        line_bot_api.reply_message(
            event.reply_token,
            TextSendMessage(text='😔 ขออภัย รายการนี้หมดชั่วคราว\nโปรดเลือกรายการอื่น')
        )


if __name__ == '__main__':
    app.run(port=3000, debug=True)
```

---

## 12. Best Practices

### Design Guidelines สำหรับ Carousel

```
✅ ใช้ขนาด "mega" สำหรับ product carousel
✅ ทุก card ควรมี hero image
✅ ทุก hero ควรใช้ aspectRatio เดียวกัน
✅ กำหนด maxLines สำหรับ title และ description
✅ footer ควรมีปุ่มที่ชัดเจน
✅ สูงสุด 6-8 cards สำหรับ UX ที่ดี (แม้ว่า max = 12)
✅ altText อธิบายเนื้อหาของ carousel ครบถ้วน
```

### Performance Tips

```
⚡ ใช้ CDN สำหรับรูปภาพทุกรูป
⚡ Lazy load ข้อมูล carousel (แค่แสดง 5-6 cards แรก)
⚡ Cache carousel JSON ที่สร้างแล้ว
⚡ ใช้ image placeholder ถ้า URL ยังไม่มี
```

### UX Guidelines

```
👆 Card ควรมี action เมื่อแตะที่ hero image
👆 ปุ่มต้องมี feedback (postback displayText)
👆 แสดงราคา/ข้อมูลสำคัญให้เห็นได้ทันที
👆 Card สุดท้ายเป็น "View All" สำหรับ list ที่ยาว
👆 Badge สีสดใส ช่วยดึงดูดความสนใจ
```

### Error Handling

```javascript
// จัดการกรณี products array ว่าง
function createProductCarousel(products) {
  if (!products || products.length === 0) {
    return {
      type: 'text',
      text: 'ขออภัย ไม่พบสินค้าที่ต้องการ กรุณาลองใหม่อีกครั้ง',
    };
  }
  
  // จำกัดสูงสุด 12 cards
  const validProducts = products.slice(0, 12);
  
  // ...
}
```

---

## 13. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Simple Carousel (ง่าย)

สร้าง Carousel แสดงข้อมูลทีมงาน 4 คน แต่ละ card มี:
- รูปโปรไฟล์ (1:1 ratio)
- ชื่อ, ตำแหน่ง, บริษัท
- Social media links

### แบบฝึกหัดที่ 2: Course Catalog (ปานกลาง)

สร้าง Carousel สำหรับ online course catalog ที่มี 5 courses:
- Thumbnail รูปคอร์ส
- ชื่อคอร์ส, instructor
- Rating, จำนวนนักเรียน
- ราคา (บาง course ฟรี)
- ปุ่ม "ลงทะเบียน" และ "ดูตัวอย่าง"

### แบบฝึกหัดที่ 3: Dynamic Category Filter (ยาก)

สร้าง system ที่:
1. รับ category จากผู้ใช้ (เช่น "อาหาร", "เครื่องดื่ม", "ขนม")
2. กรองข้อมูลตาม category
3. สร้าง Carousel และส่งกลับ
4. มีปุ่ม "เพิ่มเติม" ที่ postback ไปยัง handler เพื่อโหลดหน้าถัดไป

---

## สรุป

ใน Part นี้เราได้เรียนรู้ Carousel Container อย่างละเอียด:

1. **โครงสร้าง Carousel** และข้อจำกัดต่างๆ
2. **การสร้าง Multi-card Carousel** แบบ dynamic
3. **Card Navigation** ด้วย postback
4. **Consistent Card Sizing** เพื่อ UI ที่สวยงาม
5. **Mixed Content** เช่น Product + View All card
6. **Use Cases จริง**: Product, Menu, News, Events
7. **Node.js และ Python** examples แบบสมบูรณ์

ใน Part ต่อไปเราจะเรียนรู้การสร้าง Dynamic Flex Messages แบบ programmatic

---

## Resources

- [Flex Message Carousel Documentation](https://developers.line.biz/en/docs/messaging-api/flex-message-elements/#carousel)
- [Flex Message Simulator](https://developers.line.biz/flex-simulator/)
- [LINE Bot SDK Node.js GitHub](https://github.com/line/line-bot-sdk-nodejs)
- [LINE Bot SDK Python GitHub](https://github.com/line/line-bot-sdk-python)
