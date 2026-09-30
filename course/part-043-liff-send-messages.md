# Part 43: LIFF Send Messages - การส่งข้อความจาก LIFF

## สารบัญ

1. [liff.sendMessages() API](#liff-sendmessages-api)
2. [ประเภทข้อความที่ส่งได้](#ประเภทข้อความที่ส่งได้)
3. [การส่งข้อความ Text](#การส่งขอความ-text)
4. [การส่ง Flex Messages จาก LIFF](#การสง-flex-messages)
5. [liff.shareTargetPicker()](#liff-sharetargetpicker)
6. [การเลือก Share Targets](#การเลือก-share-targets)
7. [After Send Callbacks](#after-send-callbacks)
8. [Error Handling](#error-handling)
9. [Use Cases](#use-cases)
10. [React Component Examples](#react-component-examples)
11. [แบบฝึกหัด](#แบบฝึกหัด)

---

## 1. liff.sendMessages() API

### 1.1 ภาพรวม

`liff.sendMessages()` ช่วยให้ LIFF App สามารถส่งข้อความในนามผู้ใช้ไปยัง Chat ที่เปิด LIFF อยู่ได้ โดยไม่ต้องให้ผู้ใช้พิมพ์เอง

```
LIFF App (เปิดจาก Chat กับ Bot)
    ↓
User กดปุ่มใน LIFF
    ↓
liff.sendMessages([{ type: 'text', text: 'Hello!' }])
    ↓
ข้อความปรากฏใน Chat (ในนาม User)
    ↓
Bot รับข้อความและ Process ต่อ
```

### 1.2 Constraints ที่ต้องรู้

```javascript
const constraints = {
    maxMessages: 5,        // ส่งได้สูงสุด 5 ข้อความต่อครั้ง
    requireScope: "chat_message.write",  // ต้องมี Scope นี้
    requireInClient: true, // ต้องเปิดจาก LINE App เท่านั้น
    requireLogin: true,    // ต้อง Login แล้ว
    requireContext: true   // ต้องเปิดจาก Chat (ไม่ใช่ Direct URL)
};
```

### 1.3 เงื่อนไขการใช้งาน

```javascript
async function canSendMessages() {
    // 1. ต้องอยู่ใน LINE App
    if (!liff.isInClient()) {
        return { can: false, reason: 'ต้องเปิดจาก LINE App' };
    }
    
    // 2. ต้อง Login แล้ว
    if (!liff.isLoggedIn()) {
        return { can: false, reason: 'ต้อง Login ก่อน' };
    }
    
    // 3. ต้องมี Context (เปิดจาก Chat)
    const context = liff.getContext();
    if (!context || context.type === 'none') {
        return { can: false, reason: 'ต้องเปิดจาก Chat' };
    }
    
    return { can: true };
}
```

### 1.4 Signature

```typescript
// TypeScript Signature
liff.sendMessages(messages: Message[]): Promise<void>

// Message Types
type Message = 
    | TextMessage
    | ImageMessage
    | VideoMessage
    | AudioMessage
    | LocationMessage
    | StickerMessage
    | FlexMessage
    | TemplateMessage;
```

---

## 2. ประเภทข้อความที่ส่งได้

### 2.1 ตารางประเภทข้อความ

| ประเภท | type | รายละเอียด |
|--------|------|-----------|
| Text | `text` | ข้อความธรรมดา |
| Image | `image` | รูปภาพ |
| Video | `video` | วิดีโอ |
| Audio | `audio` | ไฟล์เสียง |
| Location | `location` | ตำแหน่งพิกัด |
| Sticker | `sticker` | สติกเกอร์ LINE |
| Flex | `flex` | Flex Message |
| Template | `template` | Template Message |

### 2.2 Text Message

```javascript
{
    type: 'text',
    text: 'ข้อความของคุณ',
    emojis: [  // Optional: ใส่ Emoji ของ LINE
        {
            index: 0,         // ตำแหน่งใน text (0-based)
            productId: '5ac1bfd5040ab15980c9b435',
            emojiId: '001'
        }
    ]
}
```

### 2.3 Image Message

```javascript
{
    type: 'image',
    originalContentUrl: 'https://example.com/image.jpg',  // ไม่เกิน 10MB
    previewImageUrl: 'https://example.com/preview.jpg'    // ไม่เกิน 1MB
}
```

### 2.4 Sticker Message

```javascript
{
    type: 'sticker',
    packageId: '446',
    stickerId: '1988'
    // ดู Sticker List: https://developers.line.biz/en/docs/messaging-api/sticker-list/
}
```

### 2.5 Location Message

```javascript
{
    type: 'location',
    title: 'บ้านของฉัน',
    address: '123 ถ.สุขุมวิท กรุงเทพ 10110',
    latitude: 13.7563,
    longitude: 100.5018
}
```

---

## 3. การส่งข้อความ Text

### 3.1 Basic Text Message

```javascript
async function sendTextMessage(text) {
    if (!liff.isInClient()) {
        alert('ฟีเจอร์นี้ใช้ได้เฉพาะใน LINE App');
        return;
    }
    
    try {
        await liff.sendMessages([{
            type: 'text',
            text: text
        }]);
        
        console.log('Message sent successfully!');
        
        // ปิด LIFF หลังส่ง (Optional)
        liff.closeWindow();
        
    } catch (error) {
        console.error('Send message error:', error);
        
        if (error.code === 'SEND_MESSAGES_FAILED') {
            alert('ไม่สามารถส่งข้อความได้');
        } else {
            alert('เกิดข้อผิดพลาด: ' + error.message);
        }
    }
}

// ส่งข้อความหลายอัน (สูงสุด 5)
async function sendMultipleMessages() {
    await liff.sendMessages([
        { type: 'text', text: 'ข้อความที่ 1' },
        { type: 'text', text: 'ข้อความที่ 2' },
        { type: 'sticker', packageId: '446', stickerId: '1988' }
    ]);
}
```

### 3.2 Form ที่ส่งข้อความหลัง Submit

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ฟอร์มสั่งอาหาร</title>
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
            padding: 0 0 100px;
        }
        
        .header {
            background: #06c755;
            color: white;
            padding: 20px 16px;
            font-size: 18px;
            font-weight: 700;
        }
        
        .section {
            background: white;
            margin: 12px 16px;
            border-radius: 12px;
            padding: 16px;
        }
        
        .section h3 {
            font-size: 14px;
            color: #666;
            text-transform: uppercase;
            margin-bottom: 12px;
        }
        
        .menu-item {
            display: flex;
            align-items: center;
            padding: 12px 0;
            border-bottom: 1px solid #f0f0f0;
            gap: 12px;
        }
        
        .menu-item:last-child { border-bottom: none; }
        
        .menu-icon {
            font-size: 32px;
            flex-shrink: 0;
        }
        
        .menu-info {
            flex: 1;
        }
        
        .menu-name {
            font-weight: 600;
            font-size: 15px;
        }
        
        .menu-price {
            color: #06c755;
            font-size: 13px;
            margin-top: 2px;
        }
        
        .quantity-control {
            display: flex;
            align-items: center;
            gap: 8px;
        }
        
        .qty-btn {
            width: 32px;
            height: 32px;
            border-radius: 50%;
            border: 1.5px solid #06c755;
            background: white;
            color: #06c755;
            font-size: 18px;
            cursor: pointer;
            display: flex;
            align-items: center;
            justify-content: center;
            line-height: 1;
        }
        
        .qty-display {
            width: 32px;
            text-align: center;
            font-weight: 700;
            font-size: 16px;
        }
        
        .form-group {
            margin-bottom: 16px;
        }
        
        .form-group label {
            display: block;
            font-size: 13px;
            font-weight: 600;
            color: #374151;
            margin-bottom: 6px;
        }
        
        .form-control {
            width: 100%;
            padding: 12px;
            border: 1.5px solid #e5e7eb;
            border-radius: 8px;
            font-size: 15px;
            outline: none;
        }
        
        .form-control:focus {
            border-color: #06c755;
        }
        
        textarea.form-control {
            height: 80px;
            resize: none;
        }
        
        .order-summary {
            background: #f0fdf4;
            border: 1.5px solid #86efac;
            border-radius: 12px;
            padding: 16px;
        }
        
        .summary-row {
            display: flex;
            justify-content: space-between;
            margin-bottom: 8px;
            font-size: 14px;
        }
        
        .summary-total {
            font-size: 18px;
            font-weight: 700;
            color: #06c755;
            border-top: 1.5px solid #86efac;
            padding-top: 8px;
            margin-top: 8px;
        }
        
        .submit-bar {
            position: fixed;
            bottom: 0;
            left: 0;
            right: 0;
            background: white;
            border-top: 1px solid #e5e7eb;
            padding: 12px 16px;
            padding-bottom: max(12px, env(safe-area-inset-bottom));
        }
        
        .submit-btn {
            width: 100%;
            padding: 16px;
            background: #06c755;
            color: white;
            border: none;
            border-radius: 12px;
            font-size: 16px;
            font-weight: 700;
            cursor: pointer;
            transition: opacity 0.2s;
        }
        
        .submit-btn:disabled {
            opacity: 0.5;
            cursor: not-allowed;
        }
        
        .submit-btn:active:not(:disabled) {
            opacity: 0.85;
        }
    </style>
</head>
<body>

<div id="app" class="app" style="display:none">
    <div class="header">🍱 สั่งอาหาร</div>
    
    <!-- เมนูอาหาร -->
    <div class="section">
        <h3>เมนู</h3>
        <div id="menuList">
            <!-- จะ Generate ด้วย JavaScript -->
        </div>
    </div>
    
    <!-- ข้อมูลลูกค้า -->
    <div class="section">
        <h3>ข้อมูลการส่ง</h3>
        <div class="form-group">
            <label>ชื่อ</label>
            <input type="text" id="customerName" class="form-control" 
                   placeholder="ชื่อของคุณ">
        </div>
        <div class="form-group">
            <label>ที่อยู่จัดส่ง</label>
            <textarea id="deliveryAddress" class="form-control"
                      placeholder="ระบุที่อยู่จัดส่ง..."></textarea>
        </div>
        <div class="form-group">
            <label>หมายเหตุ</label>
            <input type="text" id="orderNote" class="form-control"
                   placeholder="เช่น ไม่เผ็ด, ไม่ใส่ผัก">
        </div>
    </div>
    
    <!-- สรุปคำสั่งซื้อ -->
    <div class="section">
        <h3>สรุปคำสั่งซื้อ</h3>
        <div class="order-summary">
            <div id="orderSummaryItems"></div>
            <div class="summary-row summary-total">
                <span>รวมทั้งหมด</span>
                <span id="totalAmount">฿0</span>
            </div>
        </div>
    </div>
</div>

<div class="submit-bar">
    <button class="submit-btn" id="submitBtn" onclick="submitOrder()">
        🛒 ยืนยันคำสั่งซื้อ
    </button>
</div>

<div id="loading" style="display:flex;align-items:center;justify-content:center;
                          min-height:100vh;flex-direction:column;gap:16px">
    <div style="width:40px;height:40px;border:3px solid #e5e7eb;
                border-top-color:#06c755;border-radius:50%;
                animation:spin 0.7s linear infinite"></div>
    <p style="color:#666">กำลังโหลด...</p>
    <style>@keyframes spin{to{transform:rotate(360deg)}}</style>
</div>

<script>
    const LIFF_ID = 'YOUR_LIFF_ID';
    
    // เมนูอาหาร
    const menuItems = [
        { id: 'M001', icon: '🍛', name: 'ข้าวผัดกระเพรา', price: 65 },
        { id: 'M002', icon: '🍜', name: 'ผัดไทย', price: 70 },
        { id: 'M003', icon: '🥘', name: 'ต้มยำกุ้ง', price: 120 },
        { id: 'M004', icon: '🍱', name: 'ข้าวมันไก่', price: 60 },
        { id: 'M005', icon: '🥗', name: 'ยำวุ้นเส้น', price: 75 }
    ];
    
    // State
    let quantities = {};
    menuItems.forEach(item => { quantities[item.id] = 0; });
    
    // Render Menu
    function renderMenu() {
        const menuList = document.getElementById('menuList');
        menuList.innerHTML = menuItems.map(item => `
            <div class="menu-item">
                <div class="menu-icon">${item.icon}</div>
                <div class="menu-info">
                    <div class="menu-name">${item.name}</div>
                    <div class="menu-price">฿${item.price}</div>
                </div>
                <div class="quantity-control">
                    <button class="qty-btn" onclick="changeQty('${item.id}', -1)">−</button>
                    <span class="qty-display" id="qty_${item.id}">0</span>
                    <button class="qty-btn" onclick="changeQty('${item.id}', 1)">+</button>
                </div>
            </div>
        `).join('');
    }
    
    // เปลี่ยนจำนวน
    function changeQty(itemId, delta) {
        quantities[itemId] = Math.max(0, (quantities[itemId] || 0) + delta);
        document.getElementById(`qty_${itemId}`).textContent = quantities[itemId];
        updateSummary();
    }
    
    // อัพเดตสรุป
    function updateSummary() {
        const orderedItems = menuItems.filter(item => quantities[item.id] > 0);
        const total = orderedItems.reduce((sum, item) => 
            sum + (item.price * quantities[item.id]), 0
        );
        
        const summaryEl = document.getElementById('orderSummaryItems');
        if (orderedItems.length === 0) {
            summaryEl.innerHTML = '<p style="color:#9ca3af;font-size:14px">ยังไม่ได้เลือกเมนู</p>';
        } else {
            summaryEl.innerHTML = orderedItems.map(item => `
                <div class="summary-row">
                    <span>${item.icon} ${item.name} ×${quantities[item.id]}</span>
                    <span>฿${(item.price * quantities[item.id]).toLocaleString()}</span>
                </div>
            `).join('');
        }
        
        document.getElementById('totalAmount').textContent = `฿${total.toLocaleString()}`;
        document.getElementById('submitBtn').disabled = total === 0;
    }
    
    // Submit Order
    async function submitOrder() {
        const name = document.getElementById('customerName').value.trim();
        const address = document.getElementById('deliveryAddress').value.trim();
        const note = document.getElementById('orderNote').value.trim();
        
        if (!name) {
            alert('กรุณาระบุชื่อ');
            document.getElementById('customerName').focus();
            return;
        }
        
        if (!address) {
            alert('กรุณาระบุที่อยู่จัดส่ง');
            document.getElementById('deliveryAddress').focus();
            return;
        }
        
        // สร้างข้อความสรุป
        const orderedItems = menuItems.filter(item => quantities[item.id] > 0);
        const total = orderedItems.reduce((sum, item) => 
            sum + (item.price * quantities[item.id]), 0
        );
        
        if (orderedItems.length === 0) {
            alert('กรุณาเลือกเมนูก่อน');
            return;
        }
        
        // สร้าง Order Text
        const orderTime = new Date().toLocaleString('th-TH', {
            timeZone: 'Asia/Bangkok',
            year: 'numeric', month: 'short', day: 'numeric',
            hour: '2-digit', minute: '2-digit'
        });
        
        const orderText = [
            `📋 คำสั่งซื้ออาหาร`,
            `━━━━━━━━━━━━━━`,
            `👤 ชื่อ: ${name}`,
            `📍 ที่อยู่: ${address}`,
            note ? `📝 หมายเหตุ: ${note}` : '',
            ``,
            `🍱 รายการ:`,
            ...orderedItems.map(item => 
                `  ${item.icon} ${item.name} ×${quantities[item.id]} = ฿${(item.price * quantities[item.id]).toLocaleString()}`
            ),
            ``,
            `💰 รวม: ฿${total.toLocaleString()}`,
            `🕐 เวลา: ${orderTime}`
        ].filter(Boolean).join('\n');
        
        const submitBtn = document.getElementById('submitBtn');
        submitBtn.disabled = true;
        submitBtn.textContent = '⏳ กำลังส่ง...';
        
        try {
            if (!liff.isInClient()) {
                // Desktop mode: แสดงข้อความแทน
                alert('คำสั่งซื้อ:\n\n' + orderText);
                return;
            }
            
            await liff.sendMessages([
                {
                    type: 'text',
                    text: orderText
                }
            ]);
            
            // ส่งสำเร็จ
            alert('✅ ส่งคำสั่งซื้อเรียบร้อยแล้ว!\nเราจะจัดส่งให้ในไม่ช้า');
            liff.closeWindow();
            
        } catch (error) {
            console.error('Send error:', error);
            
            let errorMsg = 'ไม่สามารถส่งคำสั่งซื้อได้';
            if (error.message?.includes('Permission')) {
                errorMsg = 'ไม่ได้รับอนุญาตให้ส่งข้อความ\nกรุณาตรวจสอบ LIFF Scope';
            }
            
            alert(`❌ ${errorMsg}\n\n${error.message || ''}`);
            submitBtn.disabled = false;
            submitBtn.textContent = '🛒 ยืนยันคำสั่งซื้อ';
        }
    }
    
    // Init
    liff.init({ liffId: LIFF_ID })
        .then(async () => {
            if (!liff.isLoggedIn()) {
                liff.login({ redirectUri: window.location.href });
                return;
            }
            
            // แสดง App
            document.getElementById('loading').style.display = 'none';
            document.getElementById('app').style.display = 'block';
            
            // Load Profile เพื่อตั้งชื่ออัตโนมัติ
            try {
                const profile = await liff.getProfile();
                document.getElementById('customerName').value = profile.displayName;
            } catch (e) {}
            
            renderMenu();
            updateSummary();
        })
        .catch(err => {
            document.getElementById('loading').innerHTML = 
                `<p style="color:red">Error: ${err.message}</p>`;
        });
</script>
</body>
</html>
```

---

## 4. การส่ง Flex Messages จาก LIFF

### 4.1 Flex Message Structure

```javascript
const flexMessage = {
    type: 'flex',
    altText: 'ข้อความแทนสำหรับ Notification',
    contents: {
        // Bubble หรือ Carousel
        type: 'bubble',
        // ...
    }
};

await liff.sendMessages([flexMessage]);
```

### 4.2 Order Summary Flex Message

```javascript
// สร้าง Order Summary Flex Message
function createOrderSummaryFlex(orderData) {
    const { orderId, items, total, customerName, address, estimatedTime } = orderData;
    
    return {
        type: 'flex',
        altText: `สรุปคำสั่งซื้อ #${orderId}`,
        contents: {
            type: 'bubble',
            size: 'mega',
            
            header: {
                type: 'box',
                layout: 'horizontal',
                backgroundColor: '#06c755',
                paddingAll: '16px',
                contents: [
                    {
                        type: 'box',
                        layout: 'vertical',
                        contents: [
                            {
                                type: 'text',
                                text: '✅ ยืนยันคำสั่งซื้อ',
                                color: '#ffffff',
                                weight: 'bold',
                                size: 'lg'
                            },
                            {
                                type: 'text',
                                text: `Order #${orderId}`,
                                color: '#a8f5c3',
                                size: 'sm',
                                margin: 'xs'
                            }
                        ]
                    }
                ]
            },
            
            body: {
                type: 'box',
                layout: 'vertical',
                paddingAll: '16px',
                spacing: 'md',
                contents: [
                    // Customer Info
                    {
                        type: 'box',
                        layout: 'vertical',
                        backgroundColor: '#f8f9fa',
                        cornerRadius: '8px',
                        padding: '12px',
                        contents: [
                            createInfoRow('👤', 'ชื่อ', customerName),
                            createInfoRow('📍', 'ที่อยู่', address)
                        ]
                    },
                    
                    // Separator
                    { type: 'separator', color: '#e5e7eb' },
                    
                    // Items
                    {
                        type: 'text',
                        text: '🍱 รายการสินค้า',
                        weight: 'bold',
                        color: '#374151'
                    },
                    ...items.map(item => createItemRow(item)),
                    
                    // Separator
                    { type: 'separator', color: '#e5e7eb' },
                    
                    // Total
                    {
                        type: 'box',
                        layout: 'horizontal',
                        contents: [
                            {
                                type: 'text',
                                text: 'รวมทั้งหมด',
                                weight: 'bold',
                                size: 'lg',
                                color: '#111'
                            },
                            {
                                type: 'text',
                                text: `฿${total.toLocaleString()}`,
                                weight: 'bold',
                                size: 'xl',
                                color: '#06c755',
                                align: 'end'
                            }
                        ]
                    },
                    
                    // Estimated Time
                    {
                        type: 'text',
                        text: `⏰ เวลาจัดส่งโดยประมาณ: ${estimatedTime}`,
                        color: '#6b7280',
                        size: 'sm',
                        wrap: true
                    }
                ]
            },
            
            footer: {
                type: 'box',
                layout: 'vertical',
                padding: '16px',
                contents: [
                    {
                        type: 'button',
                        action: {
                            type: 'uri',
                            label: '📦 ติดตามคำสั่งซื้อ',
                            uri: `https://liff.line.me/YOUR_LIFF_ID/order/${orderId}`
                        },
                        style: 'primary',
                        color: '#06c755',
                        height: 'md'
                    }
                ]
            }
        }
    };
}

function createInfoRow(icon, label, value) {
    return {
        type: 'box',
        layout: 'horizontal',
        margin: 'xs',
        contents: [
            {
                type: 'text',
                text: `${icon} ${label}:`,
                color: '#6b7280',
                size: 'sm',
                flex: 2
            },
            {
                type: 'text',
                text: value,
                color: '#111',
                size: 'sm',
                flex: 3,
                wrap: true,
                align: 'end'
            }
        ]
    };
}

function createItemRow(item) {
    return {
        type: 'box',
        layout: 'horizontal',
        margin: 'sm',
        contents: [
            {
                type: 'text',
                text: `${item.name} ×${item.quantity}`,
                color: '#374151',
                size: 'sm',
                flex: 3,
                wrap: true
            },
            {
                type: 'text',
                text: `฿${(item.price * item.quantity).toLocaleString()}`,
                color: '#374151',
                size: 'sm',
                align: 'end',
                flex: 2
            }
        ]
    };
}

// ใช้งาน
async function sendOrderConfirmation(orderData) {
    const flexMsg = createOrderSummaryFlex(orderData);
    
    await liff.sendMessages([flexMsg]);
    
    console.log('Order confirmation sent!');
}
```

---

## 5. liff.shareTargetPicker()

### 5.1 ความแตกต่างจาก sendMessages

```
liff.sendMessages()               liff.shareTargetPicker()
─────────────────────             ────────────────────────────
ส่งใน Current Chat                ส่งไปยัง Chat ที่เลือก
ไม่เลือก Target                  ผู้ใช้เลือก Target เอง
ส่งอัตโนมัติ                     แสดง Target Picker UI
ต้องมี Context (Chat)             ไม่ต้องมี Context
เหมาะกับ "ยืนยันใน Chat นี้"     เหมาะกับ "แชร์ให้เพื่อน"
```

### 5.2 Basic Usage

```javascript
// Share ข้อความไปยัง Target ที่ผู้ใช้เลือก
async function shareMessage() {
    if (!liff.isInClient()) {
        alert('ต้องเปิดจาก LINE App');
        return;
    }
    
    try {
        const result = await liff.shareTargetPicker(
            [
                {
                    type: 'text',
                    text: 'ลองดูสินค้านี้สิ! 🎉\nhttps://example.com/product'
                }
            ],
            {
                isMultiple: true // อนุญาตให้เลือกหลาย Target
            }
        );
        
        if (result) {
            // ผู้ใช้เลือก Target แล้ว
            console.log('Shared successfully!', result);
        } else {
            // ผู้ใช้กด Cancel
            console.log('Share cancelled');
        }
        
    } catch (error) {
        console.error('Share error:', error);
        
        if (error.code === 'CANCEL') {
            // ผู้ใช้ยกเลิก (ไม่ใช่ Error จริง)
        } else {
            alert('เกิดข้อผิดพลาด: ' + error.message);
        }
    }
}
```

### 5.3 Share Flex Message

```javascript
async function shareProductCard(product) {
    const flexMessage = {
        type: 'flex',
        altText: `${product.name} - ฿${product.price}`,
        contents: {
            type: 'bubble',
            hero: {
                type: 'image',
                url: product.imageUrl,
                size: 'full',
                aspectRatio: '3:2',
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
                        size: 'xl',
                        wrap: true
                    },
                    {
                        type: 'box',
                        layout: 'baseline',
                        margin: 'md',
                        contents: [
                            {
                                type: 'text',
                                text: `฿${product.price.toLocaleString()}`,
                                size: 'xl',
                                color: '#06c755',
                                weight: 'bold'
                            }
                        ]
                    },
                    {
                        type: 'text',
                        text: product.description,
                        size: 'sm',
                        color: '#666',
                        wrap: true,
                        margin: 'md'
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
                            label: 'ดูสินค้า',
                            uri: `https://liff.line.me/YOUR_LIFF_ID/product/${product.id}`
                        },
                        style: 'primary',
                        color: '#06c755'
                    }
                ]
            }
        }
    };
    
    try {
        const result = await liff.shareTargetPicker(
            [flexMessage],
            { isMultiple: false } // เลือกได้ 1 Chat
        );
        
        return result !== null; // true = แชร์สำเร็จ
    } catch (error) {
        if (error.code !== 'CANCEL') {
            throw error;
        }
        return false;
    }
}
```

---

## 6. การเลือก Share Targets

### 6.1 isMultiple Option

```javascript
// isMultiple: true - แชร์ไปหลาย Chat พร้อมกัน
await liff.shareTargetPicker(messages, { isMultiple: true });

// isMultiple: false - แชร์ไป Chat เดียว (Default)
await liff.shareTargetPicker(messages, { isMultiple: false });
await liff.shareTargetPicker(messages); // Same as above
```

### 6.2 Target Picker UI

```
เมื่อเรียก shareTargetPicker():

┌────────────────────────────────┐
│    เลือกผู้รับข้อความ           │
├────────────────────────────────┤
│ 🔍 ค้นหา                       │
├────────────────────────────────┤
│ 👥 กลุ่ม                       │
│  ○ กลุ่มครอบครัว                │
│  ○ กลุ่มเพื่อน 2024            │
├────────────────────────────────┤
│ 💬 แชท 1:1                     │
│  ○ สมชาย                       │
│  ○ มาลี                        │
│  ○ วีระ                        │
├────────────────────────────────┤
│          [ส่ง]  [ยกเลิก]       │
└────────────────────────────────┘
```

### 6.3 Checking Share Result

```javascript
async function shareWithResult(messages) {
    try {
        const result = await liff.shareTargetPicker(messages, {
            isMultiple: true
        });
        
        if (result) {
            // result.status: 'success'
            console.log('Share status:', result.status);
            return { 
                success: true, 
                message: 'แชร์สำเร็จ!' 
            };
        } else {
            // null = ผู้ใช้กด Cancel
            return { 
                success: false, 
                message: 'ยกเลิกการแชร์' 
            };
        }
        
    } catch (error) {
        console.error('Share error:', error);
        
        const errorMessages = {
            'SEND_MESSAGES_FORBIDDEN': 'ไม่ได้รับอนุญาต',
            'INVALID_ARGUMENT': 'ข้อความไม่ถูกต้อง',
            'NETWORK_ERROR': 'ปัญหาการเชื่อมต่อ'
        };
        
        return { 
            success: false, 
            message: errorMessages[error.code] || error.message 
        };
    }
}
```

---

## 7. After Send Callbacks

### 7.1 Actions หลัง Send สำเร็จ

```javascript
async function sendAndTakeAction(messages, action) {
    try {
        await liff.sendMessages(messages);
        
        // Actions หลัง Send สำเร็จ
        switch (action) {
            case 'close':
                liff.closeWindow();
                break;
                
            case 'redirect':
                window.location.href = '/thank-you';
                break;
                
            case 'show-success':
                showSuccessScreen();
                break;
                
            case 'none':
            default:
                // ทำอะไร
                break;
        }
        
    } catch (error) {
        handleSendError(error);
    }
}

// Success Screen
function showSuccessScreen() {
    document.body.innerHTML = `
        <div style="display:flex;flex-direction:column;align-items:center;
                    justify-content:center;min-height:100vh;text-align:center;
                    padding:32px;gap:16px">
            <div style="font-size:80px">✅</div>
            <h2 style="font-size:22px;font-weight:700">ส่งสำเร็จ!</h2>
            <p style="color:#666">ข้อความถูกส่งไปยัง Chat เรียบร้อยแล้ว</p>
            <button onclick="liff.closeWindow()"
                    style="padding:14px 32px;background:#06c755;color:white;
                           border:none;border-radius:10px;font-size:16px;
                           font-weight:700;cursor:pointer;margin-top:8px">
                ปิดหน้าต่าง
            </button>
        </div>
    `;
}
```

### 7.2 Optimistic UI Pattern

```javascript
// Optimistic UI: แสดงผลลัพธ์ก่อน แล้วค่อย Send
async function optimisticSend(formData) {
    // 1. แสดงหน้า Loading ทันที
    showLoadingOverlay();
    
    try {
        // 2. ส่งข้อมูลไป Backend (Optional)
        const orderResponse = await fetch('/api/orders', {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify(formData)
        });
        const order = await orderResponse.json();
        
        // 3. ส่งข้อความ LIFF
        await liff.sendMessages([
            createOrderFlex(order)
        ]);
        
        // 4. แสดงผลสำเร็จ
        hideLoadingOverlay();
        showSuccessMessage(order.id);
        
        // 5. ปิด LIFF หลัง 2 วินาที
        setTimeout(() => liff.closeWindow(), 2000);
        
    } catch (error) {
        // 5. แสดง Error
        hideLoadingOverlay();
        showErrorMessage(error);
    }
}
```

---

## 8. Error Handling

### 8.1 Error Types

```javascript
// Error codes ที่อาจเกิดขึ้น
const messageErrors = {
    // sendMessages errors
    'SEND_MESSAGES_FAILED': 'ส่งข้อความไม่สำเร็จ',
    'SEND_MESSAGES_FORBIDDEN': 'ไม่มีสิทธิ์ส่งข้อความ (ขาด Scope)',
    'INVALID_ARGUMENT': 'รูปแบบข้อความไม่ถูกต้อง',
    
    // shareTargetPicker errors  
    'CANCEL': 'ผู้ใช้ยกเลิก',
    'TARGET_PICKER_FAILED': 'Target Picker ล้มเหลว',
    
    // Network errors
    'NETWORK_ERROR': 'ปัญหาการเชื่อมต่อ Internet',
    
    // Context errors
    'NOT_IN_CLIENT': 'ต้องเปิดจาก LINE App',
    'NOT_LOGGED_IN': 'ต้อง Login ก่อน'
};
```

### 8.2 Comprehensive Error Handler

```javascript
class MessageSender {
    constructor() {
        this.retryCount = 0;
        this.maxRetries = 2;
    }
    
    async send(messages, options = {}) {
        // Pre-checks
        const checks = this.preCheck(options);
        if (!checks.ok) {
            throw new Error(checks.error);
        }
        
        // Validate messages
        this.validateMessages(messages);
        
        // Send with retry
        return await this.sendWithRetry(messages);
    }
    
    preCheck(options) {
        if (!liff.isInClient() && !options.allowExternal) {
            return { ok: false, error: 'ต้องเปิดจาก LINE App' };
        }
        
        if (!liff.isLoggedIn()) {
            return { ok: false, error: 'ต้อง Login ก่อน' };
        }
        
        const context = liff.getContext();
        if (!context && !options.useSharePicker) {
            return { ok: false, error: 'ต้องเปิดจาก Chat' };
        }
        
        return { ok: true };
    }
    
    validateMessages(messages) {
        if (!Array.isArray(messages) || messages.length === 0) {
            throw new Error('Messages array is empty');
        }
        
        if (messages.length > 5) {
            throw new Error('Maximum 5 messages per send');
        }
        
        messages.forEach((msg, index) => {
            if (!msg.type) {
                throw new Error(`Message at index ${index} missing type`);
            }
        });
    }
    
    async sendWithRetry(messages) {
        try {
            await liff.sendMessages(messages);
            this.retryCount = 0;
            return { success: true };
            
        } catch (error) {
            // ไม่ Retry ถ้าเป็น User Error
            const noRetryErrors = ['CANCEL', 'SEND_MESSAGES_FORBIDDEN', 'INVALID_ARGUMENT'];
            
            if (noRetryErrors.includes(error.code)) {
                throw error;
            }
            
            // Retry สำหรับ Network errors
            if (this.retryCount < this.maxRetries) {
                this.retryCount++;
                console.log(`Retrying... (${this.retryCount}/${this.maxRetries})`);
                
                await new Promise(resolve => setTimeout(resolve, 1000 * this.retryCount));
                return this.sendWithRetry(messages);
            }
            
            throw error;
        }
    }
}

// ใช้งาน
const sender = new MessageSender();

try {
    await sender.send([
        { type: 'text', text: 'Hello!' }
    ]);
    console.log('Sent!');
} catch (error) {
    console.error('Failed:', error.message);
    
    // UI Feedback
    const userFriendlyMessages = {
        'SEND_MESSAGES_FORBIDDEN': 'ไม่ได้รับอนุญาต กรุณาเปิดแอปใหม่',
        'NETWORK_ERROR': 'ไม่มี Internet กรุณาลองใหม่',
        'NOT_IN_CLIENT': 'กรุณาเปิดจาก LINE App',
        'NOT_LOGGED_IN': 'กรุณา Login ก่อน'
    };
    
    const msg = userFriendlyMessages[error.code] || 
                'เกิดข้อผิดพลาด กรุณาลองใหม่';
    showErrorToast(msg);
}
```

---

## 9. Use Cases

### 9.1 Form Submission Confirmation

```javascript
// Use Case: ผู้ใช้กรอกฟอร์ม → ส่ง Confirmation ไปใน Chat

async function handleFormSubmit(formData) {
    // 1. Validate
    if (!validateForm(formData)) return;
    
    // 2. สร้างข้อความยืนยัน
    const confirmText = formatConfirmation(formData);
    
    // 3. ส่งข้อความ
    try {
        await liff.sendMessages([{
            type: 'text',
            text: confirmText
        }]);
        
        // 4. ปิด LIFF
        liff.closeWindow();
    } catch (error) {
        handleError(error);
    }
}

function formatConfirmation(data) {
    const lines = [
        `📝 ข้อมูลที่ลงทะเบียน`,
        `━━━━━━━━━━━━━━━━━━`,
        `👤 ชื่อ-นามสกุล: ${data.fullName}`,
        `📞 โทรศัพท์: ${data.phone}`,
        `📧 อีเมล: ${data.email}`,
        `🏠 ที่อยู่: ${data.address}`,
        `━━━━━━━━━━━━━━━━━━`,
        `กรุณาตรวจสอบข้อมูลก่อนยืนยัน`
    ];
    return lines.join('\n');
}
```

### 9.2 Event Invitation

```javascript
// Use Case: แชร์คำเชิญงาน Event

async function shareEventInvitation(event) {
    const inviteFlex = {
        type: 'flex',
        altText: `เชิญชวน: ${event.name}`,
        contents: {
            type: 'bubble',
            header: {
                type: 'box',
                layout: 'vertical',
                backgroundColor: event.color || '#6366f1',
                paddingAll: '20px',
                contents: [
                    { type: 'text', text: event.emoji || '🎉', size: '3xl' },
                    {
                        type: 'text',
                        text: event.name,
                        color: '#fff',
                        size: 'xl',
                        weight: 'bold',
                        margin: 'sm',
                        wrap: true
                    }
                ]
            },
            body: {
                type: 'box',
                layout: 'vertical',
                paddingAll: '16px',
                spacing: 'md',
                contents: [
                    createEventRow('📅', 'วันที่', event.date),
                    createEventRow('🕐', 'เวลา', event.time),
                    createEventRow('📍', 'สถานที่', event.venue),
                    {
                        type: 'separator',
                        margin: 'md',
                        color: '#e5e7eb'
                    },
                    {
                        type: 'text',
                        text: event.description,
                        size: 'sm',
                        color: '#666',
                        wrap: true,
                        margin: 'md'
                    }
                ]
            },
            footer: {
                type: 'box',
                layout: 'horizontal',
                paddingAll: '12px',
                spacing: 'sm',
                contents: [
                    {
                        type: 'button',
                        action: {
                            type: 'uri',
                            label: '✅ ตอบรับ',
                            uri: `${event.rsvpUrl}?action=accept`
                        },
                        style: 'primary',
                        color: '#06c755',
                        flex: 1
                    },
                    {
                        type: 'button',
                        action: {
                            type: 'uri',
                            label: 'ดูรายละเอียด',
                            uri: event.detailUrl
                        },
                        style: 'secondary',
                        flex: 1
                    }
                ]
            }
        }
    };
    
    try {
        const result = await liff.shareTargetPicker(
            [inviteFlex],
            { isMultiple: true }
        );
        
        if (result) {
            showToast('แชร์คำเชิญสำเร็จ! 🎉');
        }
    } catch (error) {
        if (error.code !== 'CANCEL') {
            console.error('Share error:', error);
        }
    }
}

function createEventRow(icon, label, value) {
    return {
        type: 'box',
        layout: 'horizontal',
        contents: [
            { type: 'text', text: `${icon} ${label}`, color: '#666', size: 'sm', flex: 2 },
            { type: 'text', text: value, color: '#111', size: 'sm', flex: 3, wrap: true, align: 'end' }
        ]
    };
}
```

---

## 10. React Component Examples

### 10.1 MessageSender Component

```jsx
// components/MessageSender.jsx
import { useState, useCallback } from 'react';
import liff from '@line/liff';

const MessageSender = ({ messages, onSuccess, onError }) => {
    const [sending, setSending] = useState(false);
    const [sent, setSent] = useState(false);
    
    const handleSend = useCallback(async () => {
        if (!liff.isInClient()) {
            onError?.('ต้องเปิดจาก LINE App');
            return;
        }
        
        setSending(true);
        
        try {
            await liff.sendMessages(messages);
            setSent(true);
            onSuccess?.();
        } catch (error) {
            console.error('Send error:', error);
            onError?.(error.message || 'เกิดข้อผิดพลาด');
        } finally {
            setSending(false);
        }
    }, [messages, onSuccess, onError]);
    
    if (sent) {
        return (
            <div style={{
                display: 'flex',
                flexDirection: 'column',
                alignItems: 'center',
                gap: 12,
                padding: 24,
                textAlign: 'center'
            }}>
                <span style={{ fontSize: 48 }}>✅</span>
                <h3 style={{ margin: 0, color: '#06c755' }}>ส่งสำเร็จ!</h3>
                <button
                    onClick={() => liff.closeWindow()}
                    style={{
                        padding: '10px 24px',
                        background: '#06c755',
                        color: 'white',
                        border: 'none',
                        borderRadius: 8,
                        fontSize: 14,
                        cursor: 'pointer'
                    }}
                >
                    ปิดหน้าต่าง
                </button>
            </div>
        );
    }
    
    return (
        <button
            onClick={handleSend}
            disabled={sending}
            style={{
                width: '100%',
                padding: '14px 24px',
                background: sending ? '#9ca3af' : '#06c755',
                color: 'white',
                border: 'none',
                borderRadius: 12,
                fontSize: 16,
                fontWeight: 700,
                cursor: sending ? 'not-allowed' : 'pointer',
                display: 'flex',
                alignItems: 'center',
                justifyContent: 'center',
                gap: 8
            }}
        >
            {sending ? (
                <>
                    <span style={{
                        width: 16,
                        height: 16,
                        border: '2px solid rgba(255,255,255,0.3)',
                        borderTopColor: 'white',
                        borderRadius: '50%',
                        animation: 'spin 0.6s linear infinite'
                    }} />
                    กำลังส่ง...
                </>
            ) : (
                '💬 ส่งข้อความ'
            )}
        </button>
    );
};

export default MessageSender;
```

### 10.2 ShareButton Component

```jsx
// components/ShareButton.jsx
import { useState } from 'react';
import liff from '@line/liff';

const ShareButton = ({ 
    messages, 
    isMultiple = false, 
    label = 'แชร์',
    onShared,
    onCancel 
}) => {
    const [status, setStatus] = useState('idle'); // idle, sharing, success, error
    const [errorMsg, setErrorMsg] = useState('');
    
    const handleShare = async () => {
        if (!liff.isInClient()) {
            alert('ต้องเปิดจาก LINE App');
            return;
        }
        
        setStatus('sharing');
        
        try {
            const result = await liff.shareTargetPicker(messages, { isMultiple });
            
            if (result) {
                setStatus('success');
                onShared?.(result);
                setTimeout(() => setStatus('idle'), 2000);
            } else {
                setStatus('idle');
                onCancel?.();
            }
        } catch (error) {
            if (error.code === 'CANCEL') {
                setStatus('idle');
                onCancel?.();
            } else {
                setStatus('error');
                setErrorMsg(error.message);
                setTimeout(() => setStatus('idle'), 3000);
            }
        }
    };
    
    const styles = {
        idle: { background: '#06c755', color: 'white' },
        sharing: { background: '#9ca3af', color: 'white' },
        success: { background: '#059669', color: 'white' },
        error: { background: '#dc2626', color: 'white' }
    };
    
    const labels = {
        idle: label,
        sharing: 'กำลังแชร์...',
        success: '✓ แชร์สำเร็จ!',
        error: `❌ ${errorMsg || 'เกิดข้อผิดพลาด'}`
    };
    
    return (
        <button
            onClick={handleShare}
            disabled={status === 'sharing'}
            style={{
                padding: '12px 24px',
                borderRadius: 10,
                border: 'none',
                fontSize: 15,
                fontWeight: 600,
                cursor: status === 'sharing' ? 'not-allowed' : 'pointer',
                transition: 'all 0.2s',
                ...styles[status]
            }}
        >
            {labels[status]}
        </button>
    );
};

export default ShareButton;
```

### 10.3 Complete Order Summary Component

```jsx
// components/OrderSummary.jsx
import { useState } from 'react';
import liff from '@line/liff';

const OrderSummary = ({ order, onConfirm, onCancel }) => {
    const [sending, setSending] = useState(false);
    
    const handleConfirm = async () => {
        setSending(true);
        
        const summaryText = [
            `✅ ยืนยันคำสั่งซื้อ #${order.id}`,
            `────────────────────`,
            ...order.items.map(item => 
                `• ${item.name} ×${item.qty} = ฿${(item.price * item.qty).toLocaleString()}`
            ),
            `────────────────────`,
            `💰 รวม: ฿${order.total.toLocaleString()}`,
            `📍 ที่อยู่: ${order.address}`
        ].join('\n');
        
        try {
            if (liff.isInClient()) {
                await liff.sendMessages([{ type: 'text', text: summaryText }]);
            }
            onConfirm?.(order);
        } catch (error) {
            console.error(error);
            alert('เกิดข้อผิดพลาดในการส่งข้อความ');
        } finally {
            setSending(false);
        }
    };
    
    return (
        <div style={{ padding: '16px' }}>
            <h2 style={{ marginBottom: 16 }}>สรุปคำสั่งซื้อ</h2>
            
            {order.items.map((item, i) => (
                <div key={i} style={{
                    display: 'flex',
                    justifyContent: 'space-between',
                    padding: '8px 0',
                    borderBottom: '1px solid #f0f0f0'
                }}>
                    <span>{item.name} ×{item.qty}</span>
                    <span>฿{(item.price * item.qty).toLocaleString()}</span>
                </div>
            ))}
            
            <div style={{
                display: 'flex',
                justifyContent: 'space-between',
                padding: '12px 0',
                fontWeight: 700,
                fontSize: 18,
                color: '#06c755',
                borderTop: '2px solid #06c755',
                marginTop: 8
            }}>
                <span>รวม</span>
                <span>฿{order.total.toLocaleString()}</span>
            </div>
            
            <div style={{ display: 'flex', gap: 12, marginTop: 16 }}>
                <button
                    onClick={onCancel}
                    style={{
                        flex: 1,
                        padding: '14px',
                        background: 'white',
                        border: '1.5px solid #d1d5db',
                        borderRadius: 10,
                        fontSize: 15,
                        cursor: 'pointer'
                    }}
                >
                    ยกเลิก
                </button>
                <button
                    onClick={handleConfirm}
                    disabled={sending}
                    style={{
                        flex: 2,
                        padding: '14px',
                        background: sending ? '#9ca3af' : '#06c755',
                        color: 'white',
                        border: 'none',
                        borderRadius: 10,
                        fontSize: 15,
                        fontWeight: 700,
                        cursor: sending ? 'not-allowed' : 'pointer'
                    }}
                >
                    {sending ? 'กำลังยืนยัน...' : 'ยืนยันคำสั่งซื้อ'}
                </button>
            </div>
        </div>
    );
};

export default OrderSummary;
```

---

## 11. แบบฝึกหัด

### แบบฝึกหัดที่ 1: Rating Form

สร้าง LIFF App ที่:
1. แสดง Rating Stars (1-5)
2. มี Text Area สำหรับ Comment
3. เมื่อ Submit ส่ง Rating + Comment ไปใน Chat
4. ปิด LIFF หลังส่ง

```javascript
// Hint: Message Format
const ratingMessage = {
    type: 'text',
    text: `⭐ Rating: ${'⭐'.repeat(rating)}\n💬 ${comment}`
};
```

### แบบฝึกหัดที่ 2: Product Share Card

สร้าง LIFF App ที่:
1. แสดงรายการสินค้า
2. มีปุ่ม "แชร์" ในแต่ละสินค้า
3. เมื่อกด แสดง shareTargetPicker พร้อม Flex Message ของสินค้านั้น
4. แสดงสถานะ "แชร์สำเร็จ" หลังส่ง

### แบบฝึกหัดที่ 3: Event RSVP

สร้างระบบตอบรับงาน Event:
1. แสดงรายละเอียด Event (Flex Message)
2. มีปุ่ม "ยืนยันเข้าร่วม" และ "ขอโทษ ไม่ได้"
3. เมื่อกด ส่งข้อความยืนยัน/ปฏิเสธไปใน Chat
4. Bot รับข้อความและบันทึก RSVP

---

## สรุป

| API | ใช้เมื่อ |
|-----|---------|
| `liff.sendMessages()` | ส่งข้อความใน Current Chat ที่เปิด LIFF |
| `liff.shareTargetPicker()` | แชร์ไปยัง Chat ที่ผู้ใช้เลือกเอง |

**Key Points:**
- `sendMessages()` ต้องมี Context (เปิดจาก Chat)
- `shareTargetPicker()` ใช้ได้แม้ไม่มี Context
- ส่งได้สูงสุด 5 ข้อความต่อครั้ง
- ต้องอยู่ใน LINE App เท่านั้น
- Handle `error.code === 'CANCEL'` แยกจาก Error จริง

### ขั้นตอนต่อไป
- **Part 44**: LIFF Advanced - QR Scanner, Bluetooth, liff.getContext()
