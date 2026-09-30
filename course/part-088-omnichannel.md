# Part 88: Omnichannel with LINE OA

## สารบัญ

1. [Omnichannel Overview](#overview)
2. [LINE + Facebook Messenger Integration](#facebook)
3. [LINE + WhatsApp](#whatsapp)
4. [LINE + Email Automation](#email)
5. [LINE + SMS](#sms)
6. [Unified Customer Profile](#unified-profile)
7. [Channel Routing Logic](#routing)
8. [Conversation Handoff Between Channels](#handoff)
9. [Unified Analytics](#analytics)
10. [Message Queue per Channel](#message-queue)
11. [Complete Omnichannel Hub](#complete-hub)

---

## 1. Omnichannel Overview {#overview}

### ภาพรวม Omnichannel Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Omnichannel Hub                                  │
│                                                                       │
│  Channels:                                                            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────┐  │
│  │  LINE OA │  │Facebook  │  │WhatsApp  │  │  Email   │  │ SMS  │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └──┬───┘  │
│       │             │             │             │           │        │
│  ┌────▼─────────────▼─────────────▼─────────────▼───────────▼────┐  │
│  │                    Message Gateway                              │  │
│  │  - Normalize message formats                                    │  │
│  │  - Route to appropriate handler                                 │  │
│  │  - Handle channel-specific features                             │  │
│  └────────────────────────┬────────────────────────────────────────┘  │
│                           │                                            │
│  ┌────────────────────────▼────────────────────────────────────────┐  │
│  │                  Core Services                                   │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │  │
│  │  │  Unified   │  │  Routing   │  │   AI/Bot   │  │  Agent   │  │  │
│  │  │  Profile   │  │  Engine    │  │  Service   │  │  Desk    │  │  │
│  │  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                           │                                            │
│  ┌────────────────────────▼────────────────────────────────────────┐  │
│  │                    Data Layer                                    │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐  ┌──────────┐  │  │
│  │  │ PostgreSQL │  │   Redis    │  │Elasticsearch│  │  S3/MinIO│  │  │
│  │  │  (Records) │  │  (Cache)   │  │  (Search)  │  │  (Media) │  │  │
│  │  └────────────┘  └────────────┘  └────────────┘  └──────────┘  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
```

### โครงสร้างโปรเจค

```
omnichannel-hub/
├── services/
│   ├── gateway/              # Message Gateway
│   │   ├── src/
│   │   │   ├── adapters/
│   │   │   │   ├── line.adapter.ts
│   │   │   │   ├── facebook.adapter.ts
│   │   │   │   ├── whatsapp.adapter.ts
│   │   │   │   ├── email.adapter.ts
│   │   │   │   └── sms.adapter.ts
│   │   │   ├── normalizer/
│   │   │   │   └── message.normalizer.ts
│   │   │   └── router/
│   │   │       └── channel.router.ts
│   ├── profile/              # Unified Profile Service
│   ├── routing/              # Routing Engine
│   ├── agent-desk/           # Human Agent Service
│   └── analytics/            # Unified Analytics
├── shared/
│   ├── types/
│   ├── utils/
│   └── constants/
├── infrastructure/
│   ├── docker-compose.yml
│   ├── kubernetes/
│   └── terraform/
└── docs/
```

---

## 2. LINE + Facebook Messenger Integration {#facebook}

### Message Normalization

```typescript
// services/gateway/src/normalizer/message.normalizer.ts
export interface NormalizedMessage {
  id: string
  channel: 'line' | 'facebook' | 'whatsapp' | 'email' | 'sms'
  userId: string           // Unified user ID
  channelUserId: string    // Channel-specific user ID
  conversationId: string
  type: 'text' | 'image' | 'file' | 'audio' | 'video' | 'location' | 'sticker' | 'template'
  content: MessageContent
  timestamp: Date
  metadata?: Record<string, unknown>
  replyTo?: string         // Original message ID for replies
}

interface MessageContent {
  text?: string
  imageUrl?: string
  fileUrl?: string
  audioUrl?: string
  videoUrl?: string
  location?: {
    latitude: number
    longitude: number
    address?: string
  }
  sticker?: {
    packageId: string
    stickerId: string
  }
}

// LINE Message Adapter
export class LineMessageAdapter {
  normalize(lineEvent: LineWebhookEvent): NormalizedMessage {
    const { type, message, source, timestamp, replyToken } = lineEvent

    const base = {
      channel: 'line' as const,
      channelUserId: source.userId ?? source.groupId ?? source.roomId ?? 'unknown',
      conversationId: this.getConversationId(source),
      timestamp: new Date(timestamp),
      metadata: { replyToken, sourceType: source.type }
    }

    switch (message?.type) {
      case 'text':
        return {
          ...base,
          id: message.id,
          userId: '', // จะถูก populate โดย unified profile service
          type: 'text',
          content: { text: message.text }
        }
      
      case 'image':
        return {
          ...base,
          id: message.id,
          userId: '',
          type: 'image',
          content: {
            imageUrl: `https://api-data.line.me/v2/bot/message/${message.id}/content`
          }
        }
      
      case 'sticker':
        return {
          ...base,
          id: message.id,
          userId: '',
          type: 'sticker',
          content: {
            sticker: {
              packageId: message.packageId,
              stickerId: message.stickerId
            }
          }
        }
      
      default:
        return {
          ...base,
          id: message?.id ?? `unknown_${Date.now()}`,
          userId: '',
          type: 'text',
          content: { text: `[Unsupported message type: ${message?.type}]` }
        }
    }
  }

  private getConversationId(source: LineEventSource): string {
    if (source.type === 'user') return `line_u_${source.userId}`
    if (source.type === 'group') return `line_g_${source.groupId}`
    if (source.type === 'room') return `line_r_${source.roomId}`
    return `line_unknown_${Date.now()}`
  }

  denormalize(message: NormalizedMessage): LineMessage {
    switch (message.type) {
      case 'text':
        return { type: 'text', text: message.content.text ?? '' }
      
      case 'image':
        return {
          type: 'image',
          originalContentUrl: message.content.imageUrl ?? '',
          previewImageUrl: message.content.imageUrl ?? ''
        }
      
      default:
        return { type: 'text', text: message.content.text ?? '[Message]' }
    }
  }
}

// Facebook Message Adapter
export class FacebookMessageAdapter {
  normalize(fbMessage: FacebookWebhookMessage): NormalizedMessage {
    const { sender, recipient, timestamp, message } = fbMessage

    const base = {
      channel: 'facebook' as const,
      channelUserId: sender.id,
      conversationId: `fb_${sender.id}`,
      timestamp: new Date(timestamp),
      metadata: { pageId: recipient.id }
    }

    if (message.text) {
      return {
        ...base,
        id: message.mid,
        userId: '',
        type: 'text',
        content: { text: message.text }
      }
    }

    if (message.attachments) {
      const attachment = message.attachments[0]
      
      if (attachment.type === 'image') {
        return {
          ...base,
          id: message.mid,
          userId: '',
          type: 'image',
          content: { imageUrl: attachment.payload.url }
        }
      }
      
      if (attachment.type === 'audio') {
        return {
          ...base,
          id: message.mid,
          userId: '',
          type: 'audio',
          content: { audioUrl: attachment.payload.url }
        }
      }
      
      if (attachment.type === 'location') {
        return {
          ...base,
          id: message.mid,
          userId: '',
          type: 'location',
          content: {
            location: {
              latitude: attachment.payload.coordinates.lat,
              longitude: attachment.payload.coordinates.long
            }
          }
        }
      }
    }

    return {
      ...base,
      id: message.mid,
      userId: '',
      type: 'text',
      content: { text: '[Unsupported Facebook message]' }
    }
  }

  denormalize(message: NormalizedMessage): FacebookMessage {
    switch (message.type) {
      case 'text':
        return {
          messaging_type: 'RESPONSE',
          message: { text: message.content.text ?? '' }
        }
      
      case 'image':
        return {
          messaging_type: 'RESPONSE',
          message: {
            attachment: {
              type: 'image',
              payload: { url: message.content.imageUrl, is_reusable: true }
            }
          }
        }
      
      default:
        return {
          messaging_type: 'RESPONSE',
          message: { text: message.content.text ?? '[Message]' }
        }
    }
  }
}
```

### Facebook Webhook Handler

```typescript
// services/gateway/src/adapters/facebook.adapter.ts
import axios from 'axios'
import { FacebookMessageAdapter } from '../normalizer/message.normalizer'
import { MessageBus } from '../bus/message.bus'

export class FacebookAdapter {
  private adapter = new FacebookMessageAdapter()
  private pageAccessToken: string
  private verifyToken: string

  constructor(config: { pageAccessToken: string; verifyToken: string }) {
    this.pageAccessToken = config.pageAccessToken
    this.verifyToken = config.verifyToken
  }

  // Webhook verification
  handleVerification(query: Record<string, string>): string | null {
    if (
      query['hub.mode'] === 'subscribe' &&
      query['hub.verify_token'] === this.verifyToken
    ) {
      return query['hub.challenge']
    }
    return null
  }

  // Receive webhook
  async handleWebhook(body: FacebookWebhookBody, bus: MessageBus): Promise<void> {
    if (body.object !== 'page') return

    for (const entry of body.entry) {
      for (const messaging of entry.messaging) {
        if (messaging.message && !messaging.message.is_echo) {
          const normalized = this.adapter.normalize(messaging)
          await bus.publish('message.received', normalized)
        } else if (messaging.postback) {
          await bus.publish('postback.received', {
            channel: 'facebook',
            channelUserId: messaging.sender.id,
            payload: messaging.postback.payload,
            title: messaging.postback.title
          })
        } else if (messaging.read) {
          await bus.publish('message.read', {
            channel: 'facebook',
            channelUserId: messaging.sender.id,
            watermark: messaging.read.watermark
          })
        }
      }
    }
  }

  // Send message
  async sendMessage(recipientId: string, message: NormalizedMessage): Promise<void> {
    const fbMessage = this.adapter.denormalize(message)
    
    await axios.post(
      `https://graph.facebook.com/v19.0/me/messages`,
      {
        recipient: { id: recipientId },
        ...fbMessage
      },
      {
        params: { access_token: this.pageAccessToken }
      }
    )
  }

  // ส่ง typing indicator
  async sendTyping(recipientId: string, isTyping: boolean): Promise<void> {
    await axios.post(
      `https://graph.facebook.com/v19.0/me/messages`,
      {
        recipient: { id: recipientId },
        sender_action: isTyping ? 'typing_on' : 'typing_off'
      },
      {
        params: { access_token: this.pageAccessToken }
      }
    )
  }

  // Get user profile
  async getUserProfile(userId: string): Promise<FacebookUserProfile> {
    const response = await axios.get(
      `https://graph.facebook.com/${userId}`,
      {
        params: {
          fields: 'name,first_name,last_name,profile_pic,locale,timezone,gender',
          access_token: this.pageAccessToken
        }
      }
    )
    return response.data
  }
}
```

---

## 3. LINE + WhatsApp {#whatsapp}

### WhatsApp Cloud API Integration

```typescript
// services/gateway/src/adapters/whatsapp.adapter.ts
import axios from 'axios'

interface WhatsAppConfig {
  phoneNumberId: string
  accessToken: string
  verifyToken: string
  webhookSecret: string
}

export class WhatsAppAdapter {
  private config: WhatsAppConfig
  private apiBase = 'https://graph.facebook.com/v19.0'

  constructor(config: WhatsAppConfig) {
    this.config = config
  }

  // Webhook verification
  handleVerification(query: Record<string, string>): string | null {
    if (
      query['hub.mode'] === 'subscribe' &&
      query['hub.verify_token'] === this.config.verifyToken
    ) {
      return query['hub.challenge']
    }
    return null
  }

  // Receive webhook
  async handleWebhook(body: WhatsAppWebhookBody, bus: MessageBus): Promise<void> {
    const { entry } = body
    
    for (const e of entry) {
      for (const change of e.changes) {
        if (change.field !== 'messages') continue
        
        const { messages, statuses, contacts } = change.value
        
        // Process messages
        if (messages) {
          for (const msg of messages) {
            const normalized = this.normalizeMessage(msg, contacts?.[0])
            await bus.publish('message.received', normalized)
            
            // Mark as read
            await this.markAsRead(msg.id)
          }
        }
        
        // Process status updates
        if (statuses) {
          for (const status of statuses) {
            await bus.publish('message.status', {
              channel: 'whatsapp',
              messageId: status.id,
              status: status.status, // sent, delivered, read, failed
              timestamp: new Date(parseInt(status.timestamp) * 1000)
            })
          }
        }
      }
    }
  }

  private normalizeMessage(
    msg: WhatsAppMessage,
    contact?: WhatsAppContact
  ): NormalizedMessage {
    const base = {
      channel: 'whatsapp' as const,
      channelUserId: msg.from,
      conversationId: `wa_${msg.from}`,
      timestamp: new Date(parseInt(msg.timestamp) * 1000),
      userId: '',
      metadata: {
        phoneNumber: msg.from,
        contactName: contact?.profile?.name
      }
    }

    switch (msg.type) {
      case 'text':
        return {
          ...base,
          id: msg.id,
          type: 'text',
          content: { text: msg.text?.body ?? '' }
        }
      
      case 'image':
        return {
          ...base,
          id: msg.id,
          type: 'image',
          content: { imageUrl: this.getMediaUrl(msg.image?.id ?? '') }
        }
      
      case 'audio':
        return {
          ...base,
          id: msg.id,
          type: 'audio',
          content: { audioUrl: this.getMediaUrl(msg.audio?.id ?? '') }
        }
      
      case 'document':
        return {
          ...base,
          id: msg.id,
          type: 'file',
          content: { fileUrl: this.getMediaUrl(msg.document?.id ?? '') }
        }
      
      case 'location':
        return {
          ...base,
          id: msg.id,
          type: 'location',
          content: {
            location: {
              latitude: msg.location?.latitude ?? 0,
              longitude: msg.location?.longitude ?? 0,
              address: msg.location?.address
            }
          }
        }
      
      default:
        return {
          ...base,
          id: msg.id,
          type: 'text',
          content: { text: `[${msg.type} message]` }
        }
    }
  }

  private getMediaUrl(mediaId: string): string {
    return `${this.apiBase}/${mediaId}?access_token=${this.config.accessToken}`
  }

  // Send text message
  async sendTextMessage(to: string, text: string): Promise<void> {
    await this.sendMessage({
      to,
      type: 'text',
      text: { body: text, preview_url: false }
    })
  }

  // Send template message
  async sendTemplate(
    to: string,
    templateName: string,
    languageCode: string,
    components?: WhatsAppTemplateComponent[]
  ): Promise<void> {
    await this.sendMessage({
      to,
      type: 'template',
      template: {
        name: templateName,
        language: { code: languageCode },
        components
      }
    })
  }

  // Send interactive message (buttons)
  async sendButtons(
    to: string,
    text: string,
    buttons: Array<{ id: string; title: string }>
  ): Promise<void> {
    await this.sendMessage({
      to,
      type: 'interactive',
      interactive: {
        type: 'button',
        body: { text },
        action: {
          buttons: buttons.map(btn => ({
            type: 'reply',
            reply: { id: btn.id, title: btn.title }
          }))
        }
      }
    })
  }

  // Send list message
  async sendList(
    to: string,
    headerText: string,
    bodyText: string,
    buttonText: string,
    sections: WhatsAppListSection[]
  ): Promise<void> {
    await this.sendMessage({
      to,
      type: 'interactive',
      interactive: {
        type: 'list',
        header: { type: 'text', text: headerText },
        body: { text: bodyText },
        action: {
          button: buttonText,
          sections
        }
      }
    })
  }

  private async sendMessage(message: WhatsAppSendMessage): Promise<void> {
    await axios.post(
      `${this.apiBase}/${this.config.phoneNumberId}/messages`,
      {
        messaging_product: 'whatsapp',
        ...message
      },
      {
        headers: {
          'Authorization': `Bearer ${this.config.accessToken}`,
          'Content-Type': 'application/json'
        }
      }
    )
  }

  private async markAsRead(messageId: string): Promise<void> {
    await axios.post(
      `${this.apiBase}/${this.config.phoneNumberId}/messages`,
      {
        messaging_product: 'whatsapp',
        status: 'read',
        message_id: messageId
      },
      {
        headers: { 'Authorization': `Bearer ${this.config.accessToken}` }
      }
    )
  }
}
```

---

## 4. LINE + Email Automation {#email}

### Email Adapter with Nodemailer

```typescript
// services/gateway/src/adapters/email.adapter.ts
import nodemailer from 'nodemailer'
import { simpleParser, ParsedMail } from 'mailparser'
import Imap from 'node-imap'

interface EmailConfig {
  smtp: {
    host: string
    port: number
    secure: boolean
    user: string
    password: string
  }
  imap: {
    host: string
    port: number
    tls: boolean
    user: string
    password: string
  }
}

export class EmailAdapter {
  private transporter: nodemailer.Transporter
  private imapClient: Imap
  private config: EmailConfig

  constructor(config: EmailConfig) {
    this.config = config
    
    this.transporter = nodemailer.createTransport({
      host: config.smtp.host,
      port: config.smtp.port,
      secure: config.smtp.secure,
      auth: {
        user: config.smtp.user,
        pass: config.smtp.password
      }
    })

    this.imapClient = new Imap({
      user: config.imap.user,
      password: config.imap.password,
      host: config.imap.host,
      port: config.imap.port,
      tls: config.imap.tls
    })
  }

  // ส่ง email
  async sendEmail(params: {
    to: string | string[]
    subject: string
    text?: string
    html?: string
    attachments?: nodemailer.Attachment[]
    replyTo?: string
  }): Promise<{ messageId: string }> {
    const result = await this.transporter.sendMail({
      from: `"LINE Mini CRM" <${this.config.smtp.user}>`,
      to: Array.isArray(params.to) ? params.to.join(', ') : params.to,
      subject: params.subject,
      text: params.text,
      html: params.html,
      attachments: params.attachments,
      replyTo: params.replyTo
    })

    return { messageId: result.messageId }
  }

  // รับ email ผ่าน IMAP
  startReceiving(onEmail: (message: NormalizedMessage) => void): void {
    this.imapClient.once('ready', () => {
      this.imapClient.openBox('INBOX', false, (err, box) => {
        if (err) throw err

        // รับ email ใหม่
        this.imapClient.on('mail', (numNew) => {
          const fetch = this.imapClient.seq.fetch(
            `${box.messages.total - numNew + 1}:*`,
            { bodies: '', struct: true }
          )

          fetch.on('message', (msg) => {
            let buffer = ''
            
            msg.on('body', (stream) => {
              stream.on('data', (chunk) => {
                buffer += chunk.toString('utf8')
              })
            })

            msg.once('end', async () => {
              const parsed = await simpleParser(buffer)
              const normalized = this.normalizeEmail(parsed)
              onEmail(normalized)
            })
          })
        })
      })
    })

    this.imapClient.connect()
  }

  private normalizeEmail(parsed: ParsedMail): NormalizedMessage {
    const fromAddress = parsed.from?.value[0]?.address ?? 'unknown'
    
    return {
      id: parsed.messageId ?? `email_${Date.now()}`,
      channel: 'email',
      channelUserId: fromAddress,
      userId: '',
      conversationId: `email_${fromAddress.replace('@', '_at_')}`,
      type: 'text',
      content: {
        text: parsed.text ?? parsed.subject ?? ''
      },
      timestamp: parsed.date ?? new Date(),
      metadata: {
        subject: parsed.subject,
        from: fromAddress,
        to: parsed.to?.value?.map(v => v.address).join(', '),
        htmlContent: parsed.html || undefined,
        attachments: parsed.attachments?.map(a => ({
          filename: a.filename,
          contentType: a.contentType,
          size: a.size
        }))
      }
    }
  }

  // ส่ง email template
  async sendTemplateEmail(params: {
    to: string
    template: 'welcome' | 'order_confirmation' | 'shipping_update' | 'support_reply'
    data: Record<string, unknown>
  }): Promise<void> {
    const templates: Record<string, { subject: string; html: (data: Record<string, unknown>) => string }> = {
      welcome: {
        subject: 'ยินดีต้อนรับสู่ระบบของเรา!',
        html: (data) => `
          <div style="font-family: sans-serif; max-width: 600px; margin: 0 auto;">
            <div style="background: #00B900; padding: 20px; text-align: center;">
              <img src="https://your-domain.com/logo.png" alt="Logo" height="40">
            </div>
            <div style="padding: 30px;">
              <h1>ยินดีต้อนรับ, ${data.name}!</h1>
              <p>ขอบคุณที่สมัครใช้งาน คุณสามารถเริ่มใช้งานได้ทันที</p>
              <a href="${data.loginUrl}" 
                 style="background: #00B900; color: white; padding: 12px 24px; 
                        text-decoration: none; border-radius: 6px; display: inline-block;">
                เริ่มใช้งาน
              </a>
            </div>
          </div>
        `
      },
      order_confirmation: {
        subject: `ยืนยันคำสั่งซื้อ #${(params.data as any).orderId}`,
        html: (data) => `
          <div style="font-family: sans-serif; max-width: 600px; margin: 0 auto;">
            <h2>ยืนยันคำสั่งซื้อ #${data.orderId}</h2>
            <p>เราได้รับคำสั่งซื้อของคุณแล้ว และกำลังดำเนินการ</p>
            <table style="width: 100%; border-collapse: collapse;">
              <tr style="background: #f5f5f5;">
                <th style="padding: 8px; text-align: left;">สินค้า</th>
                <th style="padding: 8px; text-align: right;">จำนวน</th>
                <th style="padding: 8px; text-align: right;">ราคา</th>
              </tr>
              ${(data.items as any[]).map(item => `
                <tr>
                  <td style="padding: 8px;">${item.name}</td>
                  <td style="padding: 8px; text-align: right;">${item.quantity}</td>
                  <td style="padding: 8px; text-align: right;">฿${item.price.toLocaleString()}</td>
                </tr>
              `).join('')}
              <tr style="font-weight: bold;">
                <td colspan="2" style="padding: 8px; text-align: right;">รวม:</td>
                <td style="padding: 8px; text-align: right;">฿${(data.total as number).toLocaleString()}</td>
              </tr>
            </table>
          </div>
        `
      },
      shipping_update: {
        subject: `อัปเดตการจัดส่ง - #${(params.data as any).orderId}`,
        html: (data) => `
          <div style="font-family: sans-serif; max-width: 600px; margin: 0 auto;">
            <h2>สถานะการจัดส่งอัปเดต</h2>
            <p>คำสั่งซื้อ #${data.orderId} ของคุณ:</p>
            <p style="font-size: 20px; color: #00B900; font-weight: bold;">
              ${data.status}
            </p>
            <p>หมายเลขพัสดุ: <strong>${data.trackingNumber}</strong></p>
            <a href="${data.trackingUrl}" 
               style="background: #0066CC; color: white; padding: 12px 24px; 
                      text-decoration: none; border-radius: 6px; display: inline-block;">
              ติดตามพัสดุ
            </a>
          </div>
        `
      },
      support_reply: {
        subject: `Re: ${(params.data as any).subject}`,
        html: (data) => `
          <div style="font-family: sans-serif; max-width: 600px; margin: 0 auto;">
            <h2>ตอบกลับจากทีมสนับสนุน</h2>
            <div style="background: #f9f9f9; padding: 15px; border-left: 4px solid #00B900; margin-bottom: 20px;">
              ${data.replyContent}
            </div>
            <p style="color: #666; font-size: 14px;">
              ทีมสนับสนุน LINE OA
            </p>
          </div>
        `
      }
    }

    const template = templates[params.template]
    await this.sendEmail({
      to: params.to,
      subject: template.subject,
      html: template.html(params.data)
    })
  }
}
```

---

## 5. LINE + SMS {#sms}

### SMS Integration ด้วย Twilio

```typescript
// services/gateway/src/adapters/sms.adapter.ts
import twilio from 'twilio'

interface SMSConfig {
  accountSid: string
  authToken: string
  fromNumber: string
}

export class SMSAdapter {
  private client: twilio.Twilio
  private fromNumber: string

  constructor(config: SMSConfig) {
    this.client = twilio(config.accountSid, config.authToken)
    this.fromNumber = config.fromNumber
  }

  // ส่ง SMS
  async sendSMS(to: string, message: string): Promise<{ messageSid: string }> {
    // ตรวจสอบรูปแบบเบอร์โทร
    const formattedNumber = this.formatPhoneNumber(to)

    const result = await this.client.messages.create({
      body: message,
      from: this.fromNumber,
      to: formattedNumber
    })

    return { messageSid: result.sid }
  }

  // ส่ง OTP
  async sendOTP(to: string, otp: string): Promise<void> {
    await this.sendSMS(to, `รหัส OTP ของคุณคือ: ${otp}\nรหัสนี้หมดอายุใน 5 นาที\nอย่าบอกรหัสนี้กับใคร`)
  }

  // ส่ง notification
  async sendNotification(to: string, type: string, data: Record<string, unknown>): Promise<void> {
    const messages: Record<string, (data: Record<string, unknown>) => string> = {
      order_confirmed: (d) => `คำสั่งซื้อ #${d.orderId} ได้รับการยืนยันแล้ว รวม ฿${d.total}`,
      order_shipped: (d) => `คำสั่งซื้อ #${d.orderId} กำลังจัดส่ง เลขพัสดุ: ${d.trackingNumber}`,
      order_delivered: (d) => `คำสั่งซื้อ #${d.orderId} จัดส่งสำเร็จแล้ว ขอบคุณที่ใช้บริการ`,
      appointment_reminder: (d) => `แจ้งเตือน: คุณมีนัด ${d.title} วันที่ ${d.date} เวลา ${d.time}`
    }

    const msgFn = messages[type]
    if (!msgFn) throw new Error(`Unknown SMS type: ${type}`)

    await this.sendSMS(to, msgFn(data))
  }

  // Handle incoming SMS webhook
  handleIncomingSMS(body: TwilioSMSWebhook): NormalizedMessage {
    return {
      id: body.MessageSid,
      channel: 'sms',
      channelUserId: body.From,
      userId: '',
      conversationId: `sms_${body.From.replace('+', '')}`,
      type: 'text',
      content: { text: body.Body },
      timestamp: new Date(),
      metadata: {
        from: body.From,
        to: body.To,
        country: body.FromCountry
      }
    }
  }

  private formatPhoneNumber(phone: string): string {
    // ลบ spaces, dashes
    let cleaned = phone.replace(/[\s\-()]/g, '')
    
    // เบอร์ไทย: แปลง 0xxxxxxxxx เป็น +66xxxxxxxxx
    if (cleaned.startsWith('0') && cleaned.length === 10) {
      cleaned = '+66' + cleaned.slice(1)
    }
    
    // ตรวจสอบว่ามี + prefix
    if (!cleaned.startsWith('+')) {
      cleaned = '+' + cleaned
    }
    
    return cleaned
  }
}
```

---

## 6. Unified Customer Profile {#unified-profile}

### Profile Merging Service

```typescript
// services/profile/src/profile.service.ts
import { prisma } from '@/lib/prisma'
import { redis } from '@/lib/redis'

export interface UnifiedProfile {
  id: string
  channels: {
    line?: { userId: string; displayName: string; pictureUrl?: string }
    facebook?: { psid: string; name: string; profilePic?: string }
    whatsapp?: { phoneNumber: string; name?: string }
    email?: { address: string; name?: string }
    sms?: { phoneNumber: string }
  }
  displayName: string
  email?: string
  phone?: string
  tags: string[]
  attributes: Record<string, unknown>
  conversationHistory: ConversationSummary[]
  createdAt: Date
  updatedAt: Date
  lastSeenAt: Date
  lastSeenChannel: string
}

interface ConversationSummary {
  id: string
  channel: string
  startedAt: Date
  endedAt?: Date
  messageCount: number
  lastMessage: string
  agent?: string
  resolved: boolean
}

export class ProfileService {
  private CACHE_TTL = 300 // 5 minutes

  // ค้นหาหรือสร้าง profile จาก channel identifier
  async findOrCreateProfile(params: {
    channel: string
    channelUserId: string
    initialData?: Partial<UnifiedProfile>
  }): Promise<UnifiedProfile> {
    // ตรวจสอบ cache ก่อน
    const cacheKey = `profile:${params.channel}:${params.channelUserId}`
    const cached = await redis.get(cacheKey)
    if (cached) return JSON.parse(cached)

    // ค้นหาใน database
    const channelIdentity = await prisma.channelIdentity.findUnique({
      where: {
        channel_channelUserId: {
          channel: params.channel,
          channelUserId: params.channelUserId
        }
      },
      include: { profile: true }
    })

    if (channelIdentity?.profile) {
      const profile = await this.buildUnifiedProfile(channelIdentity.profile.id)
      await redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(profile))
      return profile
    }

    // สร้าง profile ใหม่
    const profile = await this.createProfile(params)
    await redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(profile))
    return profile
  }

  // Merge profiles เมื่อพบว่าเป็นคนเดียวกัน
  async mergeProfiles(primaryId: string, secondaryId: string): Promise<UnifiedProfile> {
    const [primary, secondary] = await Promise.all([
      prisma.profile.findUniqueOrThrow({ where: { id: primaryId }, include: { channelIdentities: true } }),
      prisma.profile.findUniqueOrThrow({ where: { id: secondaryId }, include: { channelIdentities: true } })
    ])

    // Move all channel identities to primary
    await prisma.channelIdentity.updateMany({
      where: { profileId: secondaryId },
      data: { profileId: primaryId }
    })

    // Merge attributes
    const mergedAttributes = {
      ...(secondary.attributes as Record<string, unknown>),
      ...(primary.attributes as Record<string, unknown>)
    }

    // Update primary profile
    await prisma.profile.update({
      where: { id: primaryId },
      data: {
        email: primary.email ?? secondary.email,
        phone: primary.phone ?? secondary.phone,
        attributes: mergedAttributes,
        tags: [...new Set([...(primary.tags as string[]), ...(secondary.tags as string[])])]
      }
    })

    // Delete secondary
    await prisma.profile.delete({ where: { id: secondaryId } })

    // Invalidate cache
    await this.invalidateProfileCache(primaryId)

    return this.buildUnifiedProfile(primaryId)
  }

  // อัปเดต profile attributes
  async updateProfile(
    profileId: string,
    updates: Partial<Pick<UnifiedProfile, 'email' | 'phone' | 'tags' | 'attributes'>>
  ): Promise<void> {
    await prisma.profile.update({
      where: { id: profileId },
      data: {
        ...updates,
        updatedAt: new Date()
      }
    })

    await this.invalidateProfileCache(profileId)
  }

  // ค้นหา profile
  async searchProfiles(query: string, limit = 20): Promise<UnifiedProfile[]> {
    const profiles = await prisma.profile.findMany({
      where: {
        OR: [
          { displayName: { contains: query, mode: 'insensitive' } },
          { email: { contains: query, mode: 'insensitive' } },
          { phone: { contains: query } }
        ]
      },
      take: limit,
      orderBy: { lastSeenAt: 'desc' }
    })

    return Promise.all(profiles.map(p => this.buildUnifiedProfile(p.id)))
  }

  private async createProfile(params: {
    channel: string
    channelUserId: string
    initialData?: Partial<UnifiedProfile>
  }): Promise<UnifiedProfile> {
    const profile = await prisma.profile.create({
      data: {
        displayName: params.initialData?.displayName ?? `User_${params.channelUserId.slice(-6)}`,
        email: params.initialData?.email,
        phone: params.initialData?.phone,
        tags: params.initialData?.tags ?? [],
        attributes: params.initialData?.attributes ?? {},
        lastSeenAt: new Date(),
        lastSeenChannel: params.channel,
        channelIdentities: {
          create: {
            channel: params.channel,
            channelUserId: params.channelUserId
          }
        }
      }
    })

    return this.buildUnifiedProfile(profile.id)
  }

  private async buildUnifiedProfile(profileId: string): Promise<UnifiedProfile> {
    const profile = await prisma.profile.findUniqueOrThrow({
      where: { id: profileId },
      include: {
        channelIdentities: true,
        conversations: {
          orderBy: { startedAt: 'desc' },
          take: 10,
          include: { messages: { orderBy: { timestamp: 'desc' }, take: 1 } }
        }
      }
    })

    const channels: UnifiedProfile['channels'] = {}
    
    for (const identity of profile.channelIdentities) {
      const data = identity.data as Record<string, unknown>
      
      if (identity.channel === 'line') {
        channels.line = {
          userId: identity.channelUserId,
          displayName: data.displayName as string ?? '',
          pictureUrl: data.pictureUrl as string | undefined
        }
      } else if (identity.channel === 'facebook') {
        channels.facebook = {
          psid: identity.channelUserId,
          name: data.name as string ?? '',
          profilePic: data.profilePic as string | undefined
        }
      } else if (identity.channel === 'whatsapp') {
        channels.whatsapp = {
          phoneNumber: identity.channelUserId,
          name: data.name as string | undefined
        }
      } else if (identity.channel === 'email') {
        channels.email = {
          address: identity.channelUserId,
          name: data.name as string | undefined
        }
      } else if (identity.channel === 'sms') {
        channels.sms = {
          phoneNumber: identity.channelUserId
        }
      }
    }

    return {
      id: profile.id,
      channels,
      displayName: profile.displayName,
      email: profile.email ?? undefined,
      phone: profile.phone ?? undefined,
      tags: profile.tags as string[],
      attributes: profile.attributes as Record<string, unknown>,
      conversationHistory: profile.conversations.map(conv => ({
        id: conv.id,
        channel: conv.channel,
        startedAt: conv.startedAt,
        endedAt: conv.endedAt ?? undefined,
        messageCount: conv.messageCount,
        lastMessage: conv.messages[0]?.content as string ?? '',
        agent: conv.agentId ?? undefined,
        resolved: conv.resolved
      })),
      createdAt: profile.createdAt,
      updatedAt: profile.updatedAt,
      lastSeenAt: profile.lastSeenAt,
      lastSeenChannel: profile.lastSeenChannel
    }
  }

  private async invalidateProfileCache(profileId: string): Promise<void> {
    // ลบทุก cache key ที่เกี่ยวข้องกับ profile นี้
    const identities = await prisma.channelIdentity.findMany({
      where: { profileId }
    })

    const keys = identities.map(i => `profile:${i.channel}:${i.channelUserId}`)
    keys.push(`profile:id:${profileId}`)
    
    if (keys.length > 0) {
      await redis.del(...keys)
    }
  }
}
```

---

## 7. Channel Routing Logic {#routing}

### Intelligent Message Router

```typescript
// services/routing/src/router.service.ts
import { NormalizedMessage } from '../normalizer/message.normalizer'
import { ProfileService } from '../../profile/src/profile.service'

type RouteTarget = 'bot' | 'human_agent' | 'specific_agent' | 'queue'

interface RoutingDecision {
  target: RouteTarget
  agentId?: string
  queueId?: string
  reason: string
  priority: 'low' | 'normal' | 'high' | 'urgent'
}

interface RoutingRule {
  id: string
  name: string
  priority: number
  conditions: RoutingCondition[]
  action: RoutingAction
  isActive: boolean
}

interface RoutingCondition {
  field: string
  operator: 'equals' | 'contains' | 'startsWith' | 'endsWith' | 'greaterThan' | 'lessThan' | 'in'
  value: unknown
}

interface RoutingAction {
  target: RouteTarget
  agentId?: string
  queueId?: string
  priority: RoutingDecision['priority']
}

export class RoutingService {
  private profileService: ProfileService
  private rules: RoutingRule[] = []

  constructor(profileService: ProfileService) {
    this.profileService = profileService
    this.loadRules()
  }

  async route(message: NormalizedMessage): Promise<RoutingDecision> {
    // ดึง profile ของ user
    const profile = await this.profileService.findOrCreateProfile({
      channel: message.channel,
      channelUserId: message.channelUserId
    })

    // Context สำหรับ rule evaluation
    const context = {
      message,
      profile,
      channel: message.channel,
      userId: profile.id,
      text: message.content.text?.toLowerCase() ?? '',
      tags: profile.tags,
      attributes: profile.attributes,
      messageHistory: profile.conversationHistory
    }

    // ตรวจสอบ rules ตาม priority
    const sortedRules = [...this.rules]
      .filter(r => r.isActive)
      .sort((a, b) => b.priority - a.priority)

    for (const rule of sortedRules) {
      if (this.evaluateConditions(rule.conditions, context)) {
        return {
          target: rule.action.target,
          agentId: rule.action.agentId,
          queueId: rule.action.queueId,
          priority: rule.action.priority,
          reason: `Matched rule: ${rule.name}`
        }
      }
    }

    // Default routing
    return this.defaultRouting(context)
  }

  private evaluateConditions(
    conditions: RoutingCondition[],
    context: Record<string, unknown>
  ): boolean {
    return conditions.every(condition => {
      const value = this.getNestedValue(context, condition.field)
      
      switch (condition.operator) {
        case 'equals':
          return value === condition.value
        case 'contains':
          return typeof value === 'string' && 
                 value.includes(condition.value as string)
        case 'startsWith':
          return typeof value === 'string' && 
                 value.startsWith(condition.value as string)
        case 'endsWith':
          return typeof value === 'string' && 
                 value.endsWith(condition.value as string)
        case 'greaterThan':
          return typeof value === 'number' && 
                 value > (condition.value as number)
        case 'lessThan':
          return typeof value === 'number' && 
                 value < (condition.value as number)
        case 'in':
          return Array.isArray(condition.value) && 
                 condition.value.includes(value)
        default:
          return false
      }
    })
  }

  private getNestedValue(obj: Record<string, unknown>, path: string): unknown {
    return path.split('.').reduce((current: unknown, key) => {
      if (current && typeof current === 'object') {
        return (current as Record<string, unknown>)[key]
      }
      return undefined
    }, obj)
  }

  private defaultRouting(context: {
    message: NormalizedMessage
    profile: UnifiedProfile
  }): RoutingDecision {
    const { message, profile } = context

    // VIP customers -> high priority queue
    if (profile.tags.includes('vip')) {
      return {
        target: 'queue',
        queueId: 'vip-queue',
        priority: 'high',
        reason: 'VIP customer'
      }
    }

    // Keywords ที่ต้องการ human agent
    const urgentKeywords = ['ยกเลิก', 'คืนเงิน', 'ร้องเรียน', 'ด่วน', 'urgent', 'cancel', 'refund']
    const messageText = message.content.text?.toLowerCase() ?? ''
    
    if (urgentKeywords.some(kw => messageText.includes(kw))) {
      return {
        target: 'human_agent',
        priority: 'urgent',
        reason: 'Urgent keyword detected'
      }
    }

    // Default: route to bot
    return {
      target: 'bot',
      priority: 'normal',
      reason: 'Default routing'
    }
  }

  private async loadRules(): Promise<void> {
    // โหลด rules จาก database
    // ในตัวอย่างนี้ใช้ hardcoded rules
    this.rules = [
      {
        id: 'rule_1',
        name: 'VIP Fast Track',
        priority: 100,
        conditions: [
          { field: 'profile.tags', operator: 'contains', value: 'vip' }
        ],
        action: {
          target: 'specific_agent',
          agentId: 'vip-agent-team',
          priority: 'high'
        },
        isActive: true
      },
      {
        id: 'rule_2',
        name: 'Order Issues',
        priority: 90,
        conditions: [
          { field: 'text', operator: 'contains', value: 'คำสั่งซื้อ' }
        ],
        action: {
          target: 'queue',
          queueId: 'order-support',
          priority: 'normal'
        },
        isActive: true
      },
      {
        id: 'rule_3',
        name: 'Technical Support',
        priority: 80,
        conditions: [
          { field: 'text', operator: 'contains', value: 'ปัญหา' }
        ],
        action: {
          target: 'queue',
          queueId: 'technical-support',
          priority: 'normal'
        },
        isActive: true
      }
    ]
  }
}
```

---

## 8. Conversation Handoff Between Channels {#handoff}

### Handoff Service

```typescript
// services/routing/src/handoff.service.ts
export interface ConversationContext {
  profileId: string
  currentChannel: string
  conversationId: string
  messages: NormalizedMessage[]
  metadata: Record<string, unknown>
}

export class HandoffService {
  // ย้าย conversation จาก channel หนึ่งไปอีก channel
  async handoff(params: {
    fromChannel: string
    toChannel: string
    profileId: string
    reason: string
    includeHistory?: boolean
    targetChannelId?: string // ถ้ารู้ channel-specific ID
  }): Promise<{ success: boolean; newConversationId: string }> {
    // ดึง conversation history
    const history = params.includeHistory
      ? await this.getConversationHistory(params.profileId, params.fromChannel)
      : []

    // สร้าง conversation ใหม่ใน target channel
    const newConversation = await prisma.conversation.create({
      data: {
        profileId: params.profileId,
        channel: params.toChannel,
        status: 'open',
        metadata: {
          handoffFrom: params.fromChannel,
          handoffReason: params.reason,
          handoffAt: new Date().toISOString()
        }
      }
    })

    // ส่ง context message ไปยัง agent
    if (history.length > 0) {
      const summary = this.summarizeConversation(history)
      await this.notifyHandoff(params.toChannel, params.targetChannelId, {
        profileId: params.profileId,
        conversationId: newConversation.id,
        fromChannel: params.fromChannel,
        reason: params.reason,
        summary,
        history: history.slice(-5) // ส่ง 5 ข้อความล่าสุด
      })
    }

    return {
      success: true,
      newConversationId: newConversation.id
    }
  }

  private summarizeConversation(messages: NormalizedMessage[]): string {
    const texts = messages
      .filter(m => m.type === 'text')
      .map(m => m.content.text ?? '')
      .filter(Boolean)

    if (texts.length === 0) return 'ไม่มีข้อความ'
    if (texts.length <= 3) return texts.join(' | ')
    
    return `${texts.slice(0, 2).join(' | ')} ... (${texts.length} ข้อความ) ... ${texts[texts.length - 1]}`
  }

  private async notifyHandoff(
    channel: string,
    targetId: string | undefined,
    context: {
      profileId: string
      conversationId: string
      fromChannel: string
      reason: string
      summary: string
      history: NormalizedMessage[]
    }
  ): Promise<void> {
    // ส่งการแจ้งเตือนไปยัง agent desk
    await fetch('/api/agent-desk/handoff', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        channel,
        targetId,
        ...context
      })
    })
  }

  private async getConversationHistory(
    profileId: string,
    channel: string,
    limit = 20
  ): Promise<NormalizedMessage[]> {
    const messages = await prisma.message.findMany({
      where: {
        conversation: {
          profileId,
          channel
        }
      },
      orderBy: { timestamp: 'desc' },
      take: limit
    })

    return messages.reverse().map(m => ({
      id: m.id,
      channel: channel as NormalizedMessage['channel'],
      channelUserId: m.channelUserId,
      userId: profileId,
      conversationId: m.conversationId,
      type: m.type as NormalizedMessage['type'],
      content: m.content as NormalizedMessage['content'],
      timestamp: m.timestamp
    }))
  }
}
```

---

## 9. Unified Analytics {#analytics}

### Analytics Aggregator

```typescript
// services/analytics/src/analytics.service.ts
interface ChannelMetrics {
  channel: string
  totalMessages: number
  totalConversations: number
  avgResponseTime: number
  resolutionRate: number
  csat: number // Customer Satisfaction Score
  period: { from: Date; to: Date }
}

interface UnifiedDashboardData {
  overview: {
    totalChannels: number
    totalMessages: number
    totalConversations: number
    totalProfiles: number
    avgResponseTime: number
    overallResolutionRate: number
    overallCSAT: number
  }
  byChannel: ChannelMetrics[]
  topTopics: Array<{ topic: string; count: number; channels: string[] }>
  hourlyDistribution: Array<{ hour: number; count: number }>
  channelPreferences: Array<{ channel: string; percentage: number }>
}

export class UnifiedAnalyticsService {
  async getDashboard(period: { from: Date; to: Date }): Promise<UnifiedDashboardData> {
    const [overview, byChannel, topTopics, hourlyDist, channelPrefs] = await Promise.all([
      this.getOverview(period),
      this.getChannelMetrics(period),
      this.getTopTopics(period),
      this.getHourlyDistribution(period),
      this.getChannelPreferences(period)
    ])

    return { overview, byChannel, topTopics, hourlyDistribution: hourlyDist, channelPreferences: channelPrefs }
  }

  private async getOverview(period: { from: Date; to: Date }) {
    const [messages, conversations, profiles, responseTimes] = await Promise.all([
      prisma.message.count({ where: { timestamp: { gte: period.from, lte: period.to } } }),
      prisma.conversation.count({ where: { startedAt: { gte: period.from, lte: period.to } } }),
      prisma.profile.count({ where: { createdAt: { gte: period.from, lte: period.to } } }),
      prisma.conversation.aggregate({
        where: {
          startedAt: { gte: period.from, lte: period.to },
          firstResponseAt: { not: null }
        },
        _avg: { firstResponseTime: true }
      })
    ])

    return {
      totalChannels: 5,
      totalMessages: messages,
      totalConversations: conversations,
      totalProfiles: profiles,
      avgResponseTime: responseTimes._avg.firstResponseTime ?? 0,
      overallResolutionRate: await this.getResolutionRate(period),
      overallCSAT: await this.getCSAT(period)
    }
  }

  private async getChannelMetrics(period: { from: Date; to: Date }): Promise<ChannelMetrics[]> {
    const channels = ['line', 'facebook', 'whatsapp', 'email', 'sms']
    
    return Promise.all(channels.map(async channel => {
      const [messageCount, conversationCount] = await Promise.all([
        prisma.message.count({
          where: { conversation: { channel }, timestamp: { gte: period.from, lte: period.to } }
        }),
        prisma.conversation.count({
          where: { channel, startedAt: { gte: period.from, lte: period.to } }
        })
      ])

      return {
        channel,
        totalMessages: messageCount,
        totalConversations: conversationCount,
        avgResponseTime: await this.getChannelResponseTime(channel, period),
        resolutionRate: await this.getChannelResolutionRate(channel, period),
        csat: await this.getChannelCSAT(channel, period),
        period
      }
    }))
  }

  private async getResolutionRate(period: { from: Date; to: Date }): Promise<number> {
    const [total, resolved] = await Promise.all([
      prisma.conversation.count({ where: { startedAt: { gte: period.from, lte: period.to } } }),
      prisma.conversation.count({ where: { startedAt: { gte: period.from, lte: period.to }, resolved: true } })
    ])
    
    return total > 0 ? (resolved / total) * 100 : 0
  }

  private async getCSAT(period: { from: Date; to: Date }): Promise<number> {
    const result = await prisma.feedback.aggregate({
      where: { createdAt: { gte: period.from, lte: period.to } },
      _avg: { score: true }
    })
    return result._avg.score ?? 0
  }

  private async getChannelResponseTime(channel: string, period: { from: Date; to: Date }): Promise<number> {
    const result = await prisma.conversation.aggregate({
      where: { channel, startedAt: { gte: period.from, lte: period.to }, firstResponseAt: { not: null } },
      _avg: { firstResponseTime: true }
    })
    return result._avg.firstResponseTime ?? 0
  }

  private async getChannelResolutionRate(channel: string, period: { from: Date; to: Date }): Promise<number> {
    const [total, resolved] = await Promise.all([
      prisma.conversation.count({ where: { channel, startedAt: { gte: period.from, lte: period.to } } }),
      prisma.conversation.count({ where: { channel, startedAt: { gte: period.from, lte: period.to }, resolved: true } })
    ])
    return total > 0 ? (resolved / total) * 100 : 0
  }

  private async getChannelCSAT(channel: string, period: { from: Date; to: Date }): Promise<number> {
    const result = await prisma.feedback.aggregate({
      where: { conversation: { channel }, createdAt: { gte: period.from, lte: period.to } },
      _avg: { score: true }
    })
    return result._avg.score ?? 0
  }

  private async getTopTopics(period: { from: Date; to: Date }) {
    // ใช้ text analysis เพื่อหา topics
    const messages = await prisma.message.findMany({
      where: {
        type: 'text',
        timestamp: { gte: period.from, lte: period.to }
      },
      select: { content: true, conversation: { select: { channel: true } } },
      take: 1000
    })

    // Simple keyword extraction
    const topicCounts: Record<string, { count: number; channels: Set<string> }> = {}
    const keywords = ['สั่งซื้อ', 'ยกเลิก', 'คืนเงิน', 'จัดส่ง', 'ราคา', 'สอบถาม', 'ปัญหา', 'ขอบคุณ']

    for (const msg of messages) {
      const text = (msg.content as { text?: string }).text?.toLowerCase() ?? ''
      for (const keyword of keywords) {
        if (text.includes(keyword)) {
          if (!topicCounts[keyword]) {
            topicCounts[keyword] = { count: 0, channels: new Set() }
          }
          topicCounts[keyword].count++
          topicCounts[keyword].channels.add(msg.conversation.channel)
        }
      }
    }

    return Object.entries(topicCounts)
      .map(([topic, data]) => ({
        topic,
        count: data.count,
        channels: Array.from(data.channels)
      }))
      .sort((a, b) => b.count - a.count)
      .slice(0, 10)
  }

  private async getHourlyDistribution(period: { from: Date; to: Date }) {
    const messages = await prisma.message.groupBy({
      by: ['timestamp'],
      where: { timestamp: { gte: period.from, lte: period.to } },
      _count: true
    })

    const hourCounts = new Array(24).fill(0)
    for (const m of messages) {
      const hour = m.timestamp.getHours()
      hourCounts[hour] += m._count
    }

    return hourCounts.map((count, hour) => ({ hour, count }))
  }

  private async getChannelPreferences(period: { from: Date; to: Date }) {
    const channels = ['line', 'facebook', 'whatsapp', 'email', 'sms']
    const counts = await Promise.all(
      channels.map(ch => prisma.message.count({
        where: { conversation: { channel: ch }, timestamp: { gte: period.from, lte: period.to } }
      }))
    )
    
    const total = counts.reduce((sum, c) => sum + c, 0)
    
    return channels.map((channel, i) => ({
      channel,
      percentage: total > 0 ? (counts[i] / total) * 100 : 0
    }))
  }
}
```

---

## 10. Message Queue per Channel {#message-queue}

### BullMQ Message Queue Setup

```typescript
// services/gateway/src/queue/message.queue.ts
import { Queue, Worker, QueueEvents, Job } from 'bullmq'
import { redis } from '@/lib/redis'

const QUEUE_NAMES = {
  LINE: 'line-messages',
  FACEBOOK: 'facebook-messages',
  WHATSAPP: 'whatsapp-messages',
  EMAIL: 'email-messages',
  SMS: 'sms-messages'
} as const

// Queue configuration per channel
const QUEUE_CONFIG = {
  [QUEUE_NAMES.LINE]: {
    defaultJobOptions: {
      attempts: 3,
      backoff: { type: 'exponential', delay: 1000 },
      removeOnComplete: 100,
      removeOnFail: 500
    },
    concurrency: 10,
    rateLimiter: { max: 1000, duration: 1000 } // 1000 messages/sec
  },
  [QUEUE_NAMES.FACEBOOK]: {
    defaultJobOptions: {
      attempts: 3,
      backoff: { type: 'exponential', delay: 2000 },
      removeOnComplete: 100,
      removeOnFail: 500
    },
    concurrency: 5,
    rateLimiter: { max: 200, duration: 1000 } // 200 messages/sec
  },
  [QUEUE_NAMES.WHATSAPP]: {
    defaultJobOptions: {
      attempts: 3,
      backoff: { type: 'exponential', delay: 2000 },
      removeOnComplete: 100,
      removeOnFail: 500
    },
    concurrency: 5,
    rateLimiter: { max: 80, duration: 1000 } // 80 messages/sec (WhatsApp limit)
  },
  [QUEUE_NAMES.EMAIL]: {
    defaultJobOptions: {
      attempts: 5,
      backoff: { type: 'exponential', delay: 5000 },
      removeOnComplete: 50,
      removeOnFail: 200
    },
    concurrency: 3,
    rateLimiter: { max: 14, duration: 1000 } // 14 emails/sec (SES limit free tier)
  },
  [QUEUE_NAMES.SMS]: {
    defaultJobOptions: {
      attempts: 3,
      backoff: { type: 'fixed', delay: 5000 },
      removeOnComplete: 50,
      removeOnFail: 200
    },
    concurrency: 2,
    rateLimiter: { max: 1, duration: 1000 } // 1 SMS/sec (conservative)
  }
}

export class MessageQueueService {
  private queues: Map<string, Queue> = new Map()
  private workers: Map<string, Worker> = new Map()

  constructor() {
    this.initQueues()
  }

  private initQueues(): void {
    for (const [name, config] of Object.entries(QUEUE_CONFIG)) {
      const queue = new Queue(name, {
        connection: redis,
        defaultJobOptions: config.defaultJobOptions
      })

      this.queues.set(name, queue)
    }
  }

  async addToQueue(channel: string, message: NormalizedMessage, priority?: number): Promise<void> {
    const queueName = this.getQueueName(channel)
    const queue = this.queues.get(queueName)
    
    if (!queue) throw new Error(`Queue not found: ${queueName}`)

    await queue.add(
      'process-message',
      message,
      {
        priority: priority ?? 1,
        jobId: message.id // idempotency
      }
    )
  }

  startWorkers(processor: (job: Job) => Promise<void>): void {
    for (const [name, config] of Object.entries(QUEUE_CONFIG)) {
      const worker = new Worker(
        name,
        processor,
        {
          connection: redis,
          concurrency: config.concurrency,
          limiter: config.rateLimiter
        }
      )

      worker.on('completed', job => {
        console.log(`[${name}] Message processed: ${job.id}`)
      })

      worker.on('failed', (job, err) => {
        console.error(`[${name}] Message failed: ${job?.id}`, err.message)
      })

      worker.on('stalled', job => {
        console.warn(`[${name}] Message stalled: ${job}`)
      })

      this.workers.set(name, worker)
    }
  }

  private getQueueName(channel: string): string {
    const map: Record<string, string> = {
      line: QUEUE_NAMES.LINE,
      facebook: QUEUE_NAMES.FACEBOOK,
      whatsapp: QUEUE_NAMES.WHATSAPP,
      email: QUEUE_NAMES.EMAIL,
      sms: QUEUE_NAMES.SMS
    }
    return map[channel] ?? channel
  }

  async getQueueStats(): Promise<Record<string, QueueStats>> {
    const stats: Record<string, QueueStats> = {}
    
    for (const [name, queue] of this.queues) {
      const [waiting, active, completed, failed, delayed] = await Promise.all([
        queue.getWaitingCount(),
        queue.getActiveCount(),
        queue.getCompletedCount(),
        queue.getFailedCount(),
        queue.getDelayedCount()
      ])

      stats[name] = { waiting, active, completed, failed, delayed }
    }

    return stats
  }
}

interface QueueStats {
  waiting: number
  active: number
  completed: number
  failed: number
  delayed: number
}
```

---

## 11. Complete Omnichannel Hub {#complete-hub}

### Main Gateway Service

```typescript
// services/gateway/src/main.ts
import Fastify from 'fastify'
import { LineAdapter } from './adapters/line.adapter'
import { FacebookAdapter } from './adapters/facebook.adapter'
import { WhatsAppAdapter } from './adapters/whatsapp.adapter'
import { EmailAdapter } from './adapters/email.adapter'
import { SMSAdapter } from './adapters/sms.adapter'
import { ProfileService } from '../profile/src/profile.service'
import { RoutingService } from '../routing/src/router.service'
import { MessageQueueService } from './queue/message.queue'
import { MessageBus } from './bus/message.bus'

const app = Fastify({ logger: true })
const bus = new MessageBus()
const queue = new MessageQueueService()
const profileService = new ProfileService()
const routingService = new RoutingService(profileService)

const lineAdapter = new LineAdapter({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN!,
  channelSecret: process.env.LINE_CHANNEL_SECRET!
})

const fbAdapter = new FacebookAdapter({
  pageAccessToken: process.env.FB_PAGE_ACCESS_TOKEN!,
  verifyToken: process.env.FB_VERIFY_TOKEN!
})

const waAdapter = new WhatsAppAdapter({
  phoneNumberId: process.env.WA_PHONE_NUMBER_ID!,
  accessToken: process.env.WA_ACCESS_TOKEN!,
  verifyToken: process.env.WA_VERIFY_TOKEN!,
  webhookSecret: process.env.WA_WEBHOOK_SECRET!
})

// Register webhooks
app.post('/webhooks/line', async (req, reply) => {
  const signature = req.headers['x-line-signature'] as string
  if (!lineAdapter.verifySignature(JSON.stringify(req.body), signature)) {
    return reply.status(401).send({ error: 'Invalid signature' })
  }
  
  await lineAdapter.handleWebhook(req.body as any, bus)
  reply.send({ ok: true })
})

app.get('/webhooks/facebook', async (req, reply) => {
  const challenge = fbAdapter.handleVerification(req.query as any)
  if (challenge) reply.send(parseInt(challenge))
  else reply.status(403).send({ error: 'Invalid verification' })
})

app.post('/webhooks/facebook', async (req, reply) => {
  await fbAdapter.handleWebhook(req.body as any, bus)
  reply.send({ ok: true })
})

app.get('/webhooks/whatsapp', async (req, reply) => {
  const challenge = waAdapter.handleVerification(req.query as any)
  if (challenge) reply.send(parseInt(challenge))
  else reply.status(403).send({ error: 'Invalid verification' })
})

app.post('/webhooks/whatsapp', async (req, reply) => {
  await waAdapter.handleWebhook(req.body as any, bus)
  reply.send({ ok: true })
})

// Message Bus Handler
bus.subscribe('message.received', async (message: NormalizedMessage) => {
  // 1. Resolve unified profile
  const profile = await profileService.findOrCreateProfile({
    channel: message.channel,
    channelUserId: message.channelUserId
  })
  
  message.userId = profile.id

  // 2. Route message
  const routing = await routingService.route(message)

  // 3. Add to appropriate queue
  await queue.addToQueue(message.channel, {
    ...message,
    metadata: { ...message.metadata, routing }
  })
})

// Health check
app.get('/health', async () => {
  const queueStats = await queue.getQueueStats()
  return { status: 'ok', queues: queueStats }
})

app.listen({ port: 4000, host: '0.0.0.0' }, (err) => {
  if (err) {
    app.log.error(err)
    process.exit(1)
  }
})
```

### Docker Compose สำหรับ Omnichannel Hub

```yaml
# infrastructure/docker-compose.yml
version: '3.9'

services:
  gateway:
    build: ./services/gateway
    ports:
      - "4000:4000"
    environment:
      - LINE_CHANNEL_ACCESS_TOKEN=${LINE_CHANNEL_ACCESS_TOKEN}
      - LINE_CHANNEL_SECRET=${LINE_CHANNEL_SECRET}
      - FB_PAGE_ACCESS_TOKEN=${FB_PAGE_ACCESS_TOKEN}
      - FB_VERIFY_TOKEN=${FB_VERIFY_TOKEN}
      - WA_PHONE_NUMBER_ID=${WA_PHONE_NUMBER_ID}
      - WA_ACCESS_TOKEN=${WA_ACCESS_TOKEN}
      - REDIS_URL=redis://redis:6379
      - DATABASE_URL=postgresql://omni:omni_pass@postgres:5432/omnichannel
    depends_on: [postgres, redis]
    restart: unless-stopped

  profile-service:
    build: ./services/profile
    ports:
      - "4001:4001"
    environment:
      - DATABASE_URL=postgresql://omni:omni_pass@postgres:5432/omnichannel
      - REDIS_URL=redis://redis:6379
    depends_on: [postgres, redis]
    restart: unless-stopped

  routing-service:
    build: ./services/routing
    ports:
      - "4002:4002"
    environment:
      - DATABASE_URL=postgresql://omni:omni_pass@postgres:5432/omnichannel
      - REDIS_URL=redis://redis:6379
    depends_on: [postgres, redis]
    restart: unless-stopped

  agent-desk:
    build: ./services/agent-desk
    ports:
      - "4003:4003"
    environment:
      - DATABASE_URL=postgresql://omni:omni_pass@postgres:5432/omnichannel
      - REDIS_URL=redis://redis:6379
    depends_on: [postgres, redis]
    restart: unless-stopped

  analytics:
    build: ./services/analytics
    ports:
      - "4004:4004"
    environment:
      - DATABASE_URL=postgresql://omni:omni_pass@postgres:5432/omnichannel
    depends_on: [postgres]
    restart: unless-stopped

  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: omnichannel
      POSTGRES_USER: omni
      POSTGRES_PASSWORD: omni_pass
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    ports:
      - "6379:6379"
    restart: unless-stopped

  # Bull Board - Queue monitoring UI
  bull-board:
    image: deadly0/bull-board
    ports:
      - "3100:3000"
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on: [redis]

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on: [gateway]
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

---

### สรุป Omnichannel Best Practices

```
✅ Architecture:
   - ใช้ Message Bus (Pub/Sub) สำหรับ decoupling
   - Queue per channel สำหรับ rate limiting
   - Unified customer profile สำหรับ 360° view
   - Event-driven architecture

✅ Customer Experience:
   - Seamless handoff ระหว่าง channels
   - ไม่ต้อง repeat ข้อมูลเมื่อย้าย channel
   - Consistent tone และ messaging
   - Context preservation

✅ Operations:
   - Monitor queue depths
   - Set up alerts สำหรับ failed messages
   - Regular cleanup ของ old conversations
   - Capacity planning per channel

✅ Data:
   - GDPR/PDPA compliance
   - Data retention policies
   - Audit logs
   - Encryption at rest

💡 Cost Optimization:
   - ใช้ bot สำหรับ common inquiries (70-80%)
   - Escalate to human เฉพาะ complex issues
   - Batch non-urgent messages (email, SMS)
   - Cache frequently accessed profiles
```

---

*Part 88 จบแล้ว - ต่อไปใน Part 89: Data Pipeline for LINE OA Analytics*
