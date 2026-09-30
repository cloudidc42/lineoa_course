# ตอนที่ 69: การผสานระบบ CRM กับ LINE OA

## บทนำ

การผสานระบบ CRM (Customer Relationship Management) กับ LINE OA ช่วยให้ธุรกิจสามารถจัดการความสัมพันธ์กับลูกค้าได้อย่างมีประสิทธิภาพ ตั้งแต่การรวบรวมข้อมูล Lead การติดตามการสนทนา ไปจนถึงการวิเคราะห์ Customer Lifetime Value

## สถาปัตยกรรมระบบ CRM + LINE OA

```
┌─────────────────────────────────────────────────────────────┐
│                    LINE OA Ecosystem                         │
│                                                              │
│   ┌──────────┐    ┌──────────┐    ┌──────────────────────┐  │
│   │  LINE    │───▶│ Webhook  │───▶│   CRM Integration    │  │
│   │  Users   │    │ Handler  │    │       Layer          │  │
│   └──────────┘    └──────────┘    └──────────────────────┘  │
│                                            │                 │
│                         ┌──────────────────┼──────────────┐ │
│                         ▼                  ▼              ▼ │
│                   ┌──────────┐    ┌──────────┐   ┌──────┐  │
│                   │ HubSpot  │    │Salesforce│   │Custom│  │
│                   │   CRM    │    │   CRM    │   │ CRM  │  │
│                   └──────────┘    └──────────┘   └──────┘  │
│                                                              │
│   ┌─────────────────────────────────────────────────────┐   │
│   │              Data Flow                               │   │
│   │                                                      │   │
│   │  LINE Chat → Lead Capture → Contact Enrichment       │   │
│   │       → Pipeline Management → Follow-up Automation  │   │
│   └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## 1. Customer Data Model

### โครงสร้างข้อมูลลูกค้า

```sql
-- สร้าง Database Schema สำหรับ CRM
CREATE DATABASE line_crm;
USE line_crm;

-- ตาราง contacts หลัก
CREATE TABLE contacts (
    id VARCHAR(36) PRIMARY KEY DEFAULT (UUID()),
    line_user_id VARCHAR(100) UNIQUE,
    display_name VARCHAR(255),
    picture_url VARCHAR(500),
    email VARCHAR(255),
    phone VARCHAR(20),
    company VARCHAR(255),
    job_title VARCHAR(255),
    industry VARCHAR(100),
    location VARCHAR(255),
    language VARCHAR(10) DEFAULT 'th',
    
    -- ข้อมูล CRM
    lead_score INT DEFAULT 0,
    lifecycle_stage ENUM('subscriber', 'lead', 'marketing_qualified', 'sales_qualified', 'opportunity', 'customer', 'evangelist', 'other') DEFAULT 'subscriber',
    lead_status ENUM('new', 'open', 'in_progress', 'open_deal', 'unqualified', 'attempted_to_contact', 'connected', 'bad_timing') DEFAULT 'new',
    
    -- ข้อมูลทางการตลาด
    source VARCHAR(100),
    source_detail VARCHAR(255),
    utm_source VARCHAR(100),
    utm_medium VARCHAR(100),
    utm_campaign VARCHAR(100),
    
    -- การติดตาม
    first_seen_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_seen_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    last_message_at TIMESTAMP,
    total_messages INT DEFAULT 0,
    
    -- ข้อมูลเพิ่มเติม
    tags JSON,
    custom_fields JSON,
    notes TEXT,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_line_user_id (line_user_id),
    INDEX idx_lifecycle_stage (lifecycle_stage),
    INDEX idx_lead_score (lead_score),
    INDEX idx_last_seen_at (last_seen_at)
);

-- ตาราง deals/opportunities
CREATE TABLE deals (
    id VARCHAR(36) PRIMARY KEY DEFAULT (UUID()),
    contact_id VARCHAR(36) NOT NULL,
    title VARCHAR(255) NOT NULL,
    amount DECIMAL(15, 2) DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'THB',
    stage ENUM('appointment_scheduled', 'qualified_to_buy', 'presentation_scheduled', 'decision_maker_bought_in', 'contract_sent', 'closed_won', 'closed_lost') DEFAULT 'appointment_scheduled',
    probability INT DEFAULT 0,
    close_date DATE,
    owner_id VARCHAR(36),
    pipeline_id VARCHAR(36),
    
    -- แหล่งที่มา
    source VARCHAR(100),
    
    -- ข้อมูลเพิ่มเติม
    description TEXT,
    tags JSON,
    custom_fields JSON,
    
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    closed_at TIMESTAMP,
    
    FOREIGN KEY (contact_id) REFERENCES contacts(id) ON DELETE CASCADE,
    INDEX idx_contact_id (contact_id),
    INDEX idx_stage (stage),
    INDEX idx_close_date (close_date)
);

-- ตาราง activities/interactions
CREATE TABLE activities (
    id VARCHAR(36) PRIMARY KEY DEFAULT (UUID()),
    contact_id VARCHAR(36) NOT NULL,
    deal_id VARCHAR(36),
    type ENUM('line_message', 'email', 'call', 'meeting', 'note', 'task', 'form_submission') NOT NULL,
    direction ENUM('inbound', 'outbound') DEFAULT 'inbound',
    subject VARCHAR(255),
    body TEXT,
    
    -- LINE specific
    line_message_type VARCHAR(50),
    line_message_id VARCHAR(100),
    
    -- สถานะ
    status ENUM('pending', 'completed', 'cancelled') DEFAULT 'completed',
    outcome VARCHAR(255),
    
    -- เวลา
    occurred_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    due_at TIMESTAMP,
    completed_at TIMESTAMP,
    
    owner_id VARCHAR(36),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (contact_id) REFERENCES contacts(id) ON DELETE CASCADE,
    FOREIGN KEY (deal_id) REFERENCES deals(id) ON DELETE SET NULL,
    INDEX idx_contact_id (contact_id),
    INDEX idx_type (type),
    INDEX idx_occurred_at (occurred_at)
);

-- ตาราง pipelines
CREATE TABLE pipelines (
    id VARCHAR(36) PRIMARY KEY DEFAULT (UUID()),
    name VARCHAR(255) NOT NULL,
    stages JSON NOT NULL,
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ตาราง email campaigns
CREATE TABLE campaigns (
    id VARCHAR(36) PRIMARY KEY DEFAULT (UUID()),
    name VARCHAR(255) NOT NULL,
    type ENUM('line_broadcast', 'email', 'multi_channel') DEFAULT 'line_broadcast',
    status ENUM('draft', 'scheduled', 'sending', 'sent', 'cancelled') DEFAULT 'draft',
    
    -- เนื้อหา
    subject VARCHAR(255),
    content JSON,
    
    -- เป้าหมาย
    target_segment JSON,
    target_count INT DEFAULT 0,
    
    -- ผลลัพธ์
    sent_count INT DEFAULT 0,
    delivered_count INT DEFAULT 0,
    opened_count INT DEFAULT 0,
    clicked_count INT DEFAULT 0,
    converted_count INT DEFAULT 0,
    
    scheduled_at TIMESTAMP,
    sent_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ตาราง customer_lifetime_value
CREATE TABLE customer_lifetime_value (
    id VARCHAR(36) PRIMARY KEY DEFAULT (UUID()),
    contact_id VARCHAR(36) NOT NULL UNIQUE,
    
    -- การซื้อ
    total_purchases INT DEFAULT 0,
    total_revenue DECIMAL(15, 2) DEFAULT 0,
    average_order_value DECIMAL(15, 2) DEFAULT 0,
    first_purchase_date DATE,
    last_purchase_date DATE,
    
    -- การคำนวณ CLV
    predicted_clv DECIMAL(15, 2) DEFAULT 0,
    clv_segment ENUM('high_value', 'mid_value', 'low_value', 'at_risk', 'lost') DEFAULT 'low_value',
    
    -- ข้อมูลพฤติกรรม
    engagement_score DECIMAL(5, 2) DEFAULT 0,
    churn_probability DECIMAL(5, 2) DEFAULT 0,
    
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    FOREIGN KEY (contact_id) REFERENCES contacts(id) ON DELETE CASCADE
);
```

## 2. Lead Management จาก LINE Conversations

### Node.js Lead Manager

```javascript
// src/crm/lead-manager.js
const db = require('../database/mysql');
const lineClient = require('../line/client');
const { v4: uuidv4 } = require('uuid');

class LeadManager {
    constructor() {
        this.leadScoreRules = {
            // กิจกรรมที่เพิ่มคะแนน
            message_sent: 1,
            question_asked: 3,
            product_inquiry: 5,
            price_inquiry: 8,
            demo_request: 15,
            form_submitted: 10,
            visited_website: 2,
            opened_email: 2,
            clicked_link: 5,
            
            // โปรไฟล์ที่เพิ่มคะแนน
            has_phone: 10,
            has_email: 10,
            has_company: 15,
            has_job_title: 5,
            
            // พฤติกรรมที่ลดคะแนน
            inactive_30_days: -5,
            unsubscribed: -50,
            bounce_email: -10
        };
    }

    // สร้างหรืออัพเดท Contact จาก LINE User
    async upsertContact(lineUserId, additionalData = {}) {
        try {
            // ดึงข้อมูลจาก LINE
            const profile = await lineClient.getProfile(lineUserId);
            
            // ตรวจสอบว่ามี Contact อยู่แล้วหรือไม่
            const existing = await db.query(
                'SELECT * FROM contacts WHERE line_user_id = ?',
                [lineUserId]
            );

            if (existing.length > 0) {
                // อัพเดทข้อมูล
                await db.query(
                    `UPDATE contacts SET 
                        display_name = ?,
                        picture_url = ?,
                        last_seen_at = NOW(),
                        total_messages = total_messages + 1
                        ${additionalData.email ? ', email = ?' : ''}
                        ${additionalData.phone ? ', phone = ?' : ''}
                    WHERE line_user_id = ?`,
                    [
                        profile.displayName,
                        profile.pictureUrl,
                        ...(additionalData.email ? [additionalData.email] : []),
                        ...(additionalData.phone ? [additionalData.phone] : []),
                        lineUserId
                    ]
                );
                
                return existing[0];
            } else {
                // สร้าง Contact ใหม่
                const contactId = uuidv4();
                await db.query(
                    `INSERT INTO contacts (
                        id, line_user_id, display_name, picture_url,
                        email, phone, source, lifecycle_stage
                    ) VALUES (?, ?, ?, ?, ?, ?, 'line_oa', 'subscriber')`,
                    [
                        contactId,
                        lineUserId,
                        profile.displayName,
                        profile.pictureUrl,
                        additionalData.email || null,
                        additionalData.phone || null
                    ]
                );
                
                // คำนวณ lead score เริ่มต้น
                await this.updateLeadScore(contactId);
                
                return { id: contactId, line_user_id: lineUserId, ...profile };
            }
        } catch (error) {
            console.error('Error upserting contact:', error);
            throw error;
        }
    }

    // วิเคราะห์ข้อความเพื่อจับ Intent
    analyzeMessageIntent(messageText) {
        const intents = [];
        const lowerText = messageText.toLowerCase();
        
        // ตรวจจับความตั้งใจซื้อ
        const buyingKeywords = ['ราคา', 'price', 'ซื้อ', 'buy', 'order', 'สั่ง', 'จอง', 'book', 'เท่าไหร่', 'how much'];
        if (buyingKeywords.some(kw => lowerText.includes(kw))) {
            intents.push({ type: 'price_inquiry', score: 8 });
        }
        
        // ตรวจจับการสอบถามสินค้า
        const productKeywords = ['สินค้า', 'product', 'บริการ', 'service', 'ข้อมูล', 'info', 'detail', 'รายละเอียด'];
        if (productKeywords.some(kw => lowerText.includes(kw))) {
            intents.push({ type: 'product_inquiry', score: 5 });
        }
        
        // ตรวจจับการขอ Demo
        const demoKeywords = ['demo', 'ทดลอง', 'trial', 'ทดสอบ', 'test', 'ตัวอย่าง', 'sample'];
        if (demoKeywords.some(kw => lowerText.includes(kw))) {
            intents.push({ type: 'demo_request', score: 15 });
        }
        
        // ตรวจจับการร้องเรียน
        const complaintKeywords = ['ปัญหา', 'problem', 'error', 'bug', 'ไม่ได้', 'broken', 'เสีย'];
        if (complaintKeywords.some(kw => lowerText.includes(kw))) {
            intents.push({ type: 'support_request', score: 0 });
        }
        
        return intents;
    }

    // อัพเดท Lead Score
    async updateLeadScore(contactId) {
        try {
            const contact = await db.query(
                'SELECT * FROM contacts WHERE id = ?',
                [contactId]
            );
            
            if (!contact.length) return;
            
            let score = 0;
            const c = contact[0];
            
            // คะแนนจากข้อมูลโปรไฟล์
            if (c.email) score += this.leadScoreRules.has_email;
            if (c.phone) score += this.leadScoreRules.has_phone;
            if (c.company) score += this.leadScoreRules.has_company;
            if (c.job_title) score += this.leadScoreRules.has_job_title;
            
            // คะแนนจากกิจกรรม
            const activities = await db.query(
                'SELECT type, COUNT(*) as count FROM activities WHERE contact_id = ? GROUP BY type',
                [contactId]
            );
            
            activities.forEach(activity => {
                const scoreKey = activity.type.replace('-', '_');
                if (this.leadScoreRules[scoreKey]) {
                    score += this.leadScoreRules[scoreKey] * Math.min(activity.count, 5);
                }
            });
            
            // ตรวจสอบการไม่ active
            const daysSinceLastSeen = Math.floor(
                (Date.now() - new Date(c.last_seen_at).getTime()) / (1000 * 60 * 60 * 24)
            );
            
            if (daysSinceLastSeen > 30) {
                score += this.leadScoreRules.inactive_30_days;
            }
            
            // อัพเดทคะแนนและ lifecycle stage
            score = Math.max(0, Math.min(100, score));
            const lifecycleStage = this.scoreToLifecycleStage(score);
            
            await db.query(
                'UPDATE contacts SET lead_score = ?, lifecycle_stage = ? WHERE id = ?',
                [score, lifecycleStage, contactId]
            );
            
            return score;
        } catch (error) {
            console.error('Error updating lead score:', error);
            throw error;
        }
    }

    // แปลงคะแนนเป็น lifecycle stage
    scoreToLifecycleStage(score) {
        if (score >= 80) return 'sales_qualified';
        if (score >= 60) return 'marketing_qualified';
        if (score >= 40) return 'lead';
        if (score >= 20) return 'subscriber';
        return 'subscriber';
    }

    // บันทึก Activity
    async logActivity(contactId, activityData) {
        try {
            const activityId = uuidv4();
            
            await db.query(
                `INSERT INTO activities (
                    id, contact_id, deal_id, type, direction,
                    subject, body, line_message_type, line_message_id,
                    occurred_at
                ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, NOW())`,
                [
                    activityId,
                    contactId,
                    activityData.dealId || null,
                    activityData.type,
                    activityData.direction || 'inbound',
                    activityData.subject || null,
                    activityData.body || null,
                    activityData.lineMessageType || null,
                    activityData.lineMessageId || null
                ]
            );
            
            // อัพเดท lead score หลังบันทึก activity
            await this.updateLeadScore(contactId);
            
            return activityId;
        } catch (error) {
            console.error('Error logging activity:', error);
            throw error;
        }
    }

    // ดึง Lead ที่ต้องการ Follow-up
    async getLeadsForFollowup() {
        try {
            const leads = await db.query(`
                SELECT 
                    c.*,
                    MAX(a.occurred_at) as last_activity_at,
                    COUNT(d.id) as deal_count,
                    SUM(d.amount) as total_deal_value
                FROM contacts c
                LEFT JOIN activities a ON c.id = a.contact_id
                LEFT JOIN deals d ON c.id = d.contact_id AND d.stage NOT IN ('closed_won', 'closed_lost')
                WHERE 
                    c.lifecycle_stage IN ('lead', 'marketing_qualified', 'sales_qualified')
                    AND (
                        MAX(a.occurred_at) < DATE_SUB(NOW(), INTERVAL 3 DAY)
                        OR MAX(a.occurred_at) IS NULL
                    )
                GROUP BY c.id
                ORDER BY c.lead_score DESC
                LIMIT 50
            `);
            
            return leads;
        } catch (error) {
            console.error('Error getting leads for followup:', error);
            throw error;
        }
    }
}

module.exports = new LeadManager();
```

## 3. Contact Enrichment Strategies

```javascript
// src/crm/contact-enrichment.js
const axios = require('axios');
const db = require('../database/mysql');

class ContactEnrichmentService {
    constructor() {
        this.enrichmentSources = ['clearbit', 'hunter', 'fullcontact'];
    }

    // Enrich ข้อมูลจาก Email
    async enrichFromEmail(contactId, email) {
        try {
            const enrichedData = {};
            
            // ลอง Clearbit ก่อน
            try {
                const clearbitData = await this.fetchFromClearbit(email);
                if (clearbitData) {
                    Object.assign(enrichedData, clearbitData);
                }
            } catch (e) {
                console.log('Clearbit enrichment failed:', e.message);
            }
            
            // อัพเดทข้อมูลใน Database
            if (Object.keys(enrichedData).length > 0) {
                await this.updateContactWithEnrichedData(contactId, enrichedData);
            }
            
            return enrichedData;
        } catch (error) {
            console.error('Error enriching contact:', error);
            throw error;
        }
    }

    // ดึงข้อมูลจาก Clearbit
    async fetchFromClearbit(email) {
        const response = await axios.get(
            `https://person.clearbit.com/v2/combined/find?email=${email}`,
            {
                headers: {
                    'Authorization': `Bearer ${process.env.CLEARBIT_API_KEY}`
                }
            }
        );
        
        if (!response.data) return null;
        
        const data = response.data;
        return {
            first_name: data.person?.name?.givenName,
            last_name: data.person?.name?.familyName,
            company: data.company?.name,
            job_title: data.person?.employment?.title,
            industry: data.company?.category?.industry,
            location: data.person?.location,
            linkedin_url: data.person?.linkedin?.handle,
            twitter_handle: data.person?.twitter?.handle,
            company_size: data.company?.metrics?.employees,
            company_domain: data.company?.domain
        };
    }

    // อัพเดทข้อมูลที่ Enrich แล้ว
    async updateContactWithEnrichedData(contactId, data) {
        const allowedFields = [
            'first_name', 'last_name', 'company', 'job_title',
            'industry', 'location', 'linkedin_url', 'twitter_handle'
        ];
        
        const updateFields = [];
        const values = [];
        
        allowedFields.forEach(field => {
            if (data[field] !== undefined && data[field] !== null) {
                updateFields.push(`${field} = ?`);
                values.push(data[field]);
            }
        });
        
        // บันทึกข้อมูลเพิ่มเติมใน custom_fields
        if (data.company_size || data.linkedin_url) {
            updateFields.push('custom_fields = JSON_MERGE_PATCH(COALESCE(custom_fields, "{}"), ?)');
            values.push(JSON.stringify({
                company_size: data.company_size,
                linkedin_url: data.linkedin_url,
                enriched_at: new Date().toISOString(),
                enrichment_source: 'clearbit'
            }));
        }
        
        if (updateFields.length > 0) {
            values.push(contactId);
            await db.query(
                `UPDATE contacts SET ${updateFields.join(', ')} WHERE id = ?`,
                values
            );
        }
    }

    // วิเคราะห์ Keyword จากการสนทนา
    async extractDataFromConversation(contactId, messageHistory) {
        const extractedData = {};
        
        // Regular expressions สำหรับดึงข้อมูล
        const patterns = {
            email: /[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/g,
            phone: /(?:\+66|0)[0-9]{8,9}/g,
            company: /(?:บริษัท|company|corp|ltd|จำกัด)\s+([^\s,]+)/gi,
            budget: /(?:งบประมาณ|budget|ราคา|price)[^\d]*(\d+(?:,\d+)*)/gi
        };
        
        const allText = messageHistory
            .filter(m => m.direction === 'inbound')
            .map(m => m.body)
            .join(' ');
        
        // ดึง Email
        const emails = allText.match(patterns.email);
        if (emails && emails.length > 0) {
            extractedData.email = emails[0];
        }
        
        // ดึง Phone
        const phones = allText.match(patterns.phone);
        if (phones && phones.length > 0) {
            extractedData.phone = phones[0];
        }
        
        // ดึง Budget
        const budgets = patterns.budget.exec(allText);
        if (budgets) {
            extractedData.estimated_budget = parseInt(budgets[1].replace(/,/g, ''));
        }
        
        // อัพเดทข้อมูล
        if (Object.keys(extractedData).length > 0) {
            await this.updateContactWithEnrichedData(contactId, extractedData);
        }
        
        return extractedData;
    }
}

module.exports = new ContactEnrichmentService();
```

## 4. HubSpot CRM Integration กับ Webhooks

```javascript
// src/integrations/hubspot.js
const axios = require('axios');
const crypto = require('crypto');
const db = require('../database/mysql');

class HubSpotIntegration {
    constructor() {
        this.baseUrl = 'https://api.hubapi.com';
        this.apiKey = process.env.HUBSPOT_API_KEY;
        this.portalId = process.env.HUBSPOT_PORTAL_ID;
        
        this.client = axios.create({
            baseURL: this.baseUrl,
            headers: {
                'Authorization': `Bearer ${this.apiKey}`,
                'Content-Type': 'application/json'
            }
        });
    }

    // สร้างหรืออัพเดท Contact ใน HubSpot
    async upsertContact(lineUserId, contactData) {
        try {
            const properties = {
                email: contactData.email,
                firstname: contactData.firstName || contactData.displayName,
                lastname: contactData.lastName || '',
                phone: contactData.phone,
                company: contactData.company,
                jobtitle: contactData.jobTitle,
                // Custom properties สำหรับ LINE
                line_user_id: lineUserId,
                line_display_name: contactData.displayName,
                line_picture_url: contactData.pictureUrl,
                hs_lead_status: this.mapLeadStatus(contactData.leadStatus),
                lifecyclestage: this.mapLifecycleStage(contactData.lifecycleStage)
            };

            // ลองดึง Contact จาก email ก่อน
            if (contactData.email) {
                try {
                    const existing = await this.client.get(
                        `/crm/v3/objects/contacts/${contactData.email}?idProperty=email`
                    );
                    
                    // อัพเดท Contact ที่มีอยู่
                    await this.client.patch(
                        `/crm/v3/objects/contacts/${existing.data.id}`,
                        { properties }
                    );
                    
                    return existing.data.id;
                } catch (e) {
                    if (e.response?.status !== 404) throw e;
                }
            }

            // สร้าง Contact ใหม่
            const response = await this.client.post(
                '/crm/v3/objects/contacts',
                { properties }
            );
            
            return response.data.id;
        } catch (error) {
            console.error('HubSpot upsert contact error:', error.response?.data || error.message);
            throw error;
        }
    }

    // สร้าง Deal ใน HubSpot
    async createDeal(contactId, dealData) {
        try {
            const response = await this.client.post('/crm/v3/objects/deals', {
                properties: {
                    dealname: dealData.title,
                    amount: dealData.amount,
                    dealstage: this.mapDealStage(dealData.stage),
                    pipeline: 'default',
                    closedate: dealData.closeDate,
                    description: dealData.description,
                    hs_deal_source: 'LINE OA'
                }
            });
            
            const dealId = response.data.id;
            
            // เชื่อม Deal กับ Contact
            await this.client.put(
                `/crm/v3/objects/deals/${dealId}/associations/contacts/${contactId}/3`
            );
            
            return dealId;
        } catch (error) {
            console.error('HubSpot create deal error:', error.response?.data || error.message);
            throw error;
        }
    }

    // บันทึก Note/Activity ใน HubSpot
    async logActivity(contactId, activity) {
        try {
            const engagement = {
                type: this.mapActivityType(activity.type),
                timestamp: new Date(activity.occurredAt).getTime(),
                associations: {
                    contactIds: [contactId]
                },
                metadata: {
                    body: activity.body,
                    subject: activity.subject || 'LINE OA Interaction'
                }
            };
            
            const response = await this.client.post(
                '/engagements/v1/engagements',
                engagement
            );
            
            return response.data.engagement.id;
        } catch (error) {
            console.error('HubSpot log activity error:', error.response?.data || error.message);
            throw error;
        }
    }

    // รับ Webhook จาก HubSpot
    async handleWebhook(payload, signature) {
        // ตรวจสอบ signature
        const expectedSig = crypto
            .createHash('sha256')
            .update(process.env.HUBSPOT_WEBHOOK_SECRET + payload)
            .digest('hex');
            
        if (signature !== expectedSig) {
            throw new Error('Invalid webhook signature');
        }
        
        const events = JSON.parse(payload);
        
        for (const event of events) {
            await this.processWebhookEvent(event);
        }
    }

    // ประมวลผล Webhook Event
    async processWebhookEvent(event) {
        try {
            switch (event.subscriptionType) {
                case 'contact.propertyChange':
                    await this.handleContactPropertyChange(event);
                    break;
                case 'deal.stageChange':
                    await this.handleDealStageChange(event);
                    break;
                case 'contact.creation':
                    await this.handleNewContact(event);
                    break;
            }
        } catch (error) {
            console.error('Error processing HubSpot webhook:', error);
        }
    }

    // จัดการเมื่อ Deal Stage เปลี่ยน
    async handleDealStageChange(event) {
        const hubspotDealId = event.objectId;
        
        // ดึงข้อมูล Deal จาก HubSpot
        const deal = await this.client.get(
            `/crm/v3/objects/deals/${hubspotDealId}?properties=dealname,amount,dealstage,associations`
        );
        
        // ดึง LINE User IDs ที่เชื่อมกับ Deal
        const contactAssocs = await this.client.get(
            `/crm/v3/objects/deals/${hubspotDealId}/associations/contacts`
        );
        
        for (const assoc of contactAssocs.data.results) {
            const contact = await this.client.get(
                `/crm/v3/objects/contacts/${assoc.id}?properties=line_user_id,email`
            );
            
            const lineUserId = contact.data.properties.line_user_id;
            if (!lineUserId) continue;
            
            // ส่งแจ้งเตือนผ่าน LINE
            await this.sendDealUpdateNotification(lineUserId, deal.data, event.propertyValue);
        }
    }

    // ส่งแจ้งเตือนการอัพเดท Deal ผ่าน LINE
    async sendDealUpdateNotification(lineUserId, deal, newStage) {
        const lineClient = require('../line/client');
        
        const stageMessages = {
            closedwon: '🎉 ยินดีด้วย! ดีลของคุณปิดสำเร็จแล้ว',
            closedlost: 'ขออภัย ดีลครั้งนี้ไม่ได้ดำเนินต่อ แต่เรายังพร้อมช่วยเหลือคุณ',
            contractsent: '📄 สัญญาถูกส่งให้คุณแล้ว กรุณาตรวจสอบ',
            presentationscheduled: '📅 นัดหมายการนำเสนอถูกยืนยันแล้ว'
        };
        
        const message = stageMessages[newStage] || `สถานะดีล "${deal.properties.dealname}" ได้อัพเดทแล้ว`;
        
        await lineClient.pushMessage(lineUserId, {
            type: 'text',
            text: message
        });
    }

    // Map functions
    mapLeadStatus(status) {
        const mapping = {
            'new': 'NEW',
            'open': 'OPEN',
            'in_progress': 'IN_PROGRESS',
            'connected': 'CONNECTED',
            'unqualified': 'UNQUALIFIED'
        };
        return mapping[status] || 'NEW';
    }

    mapLifecycleStage(stage) {
        const mapping = {
            'subscriber': 'subscriber',
            'lead': 'lead',
            'marketing_qualified': 'marketingqualifiedlead',
            'sales_qualified': 'salesqualifiedlead',
            'opportunity': 'opportunity',
            'customer': 'customer',
            'evangelist': 'evangelist'
        };
        return mapping[stage] || 'subscriber';
    }

    mapDealStage(stage) {
        const mapping = {
            'appointment_scheduled': 'appointmentscheduled',
            'qualified_to_buy': 'qualifiedtobuy',
            'presentation_scheduled': 'presentationscheduled',
            'decision_maker_bought_in': 'decisionmakerboughtin',
            'contract_sent': 'contractsent',
            'closed_won': 'closedwon',
            'closed_lost': 'closedlost'
        };
        return mapping[stage] || 'appointmentscheduled';
    }

    mapActivityType(type) {
        const mapping = {
            'line_message': 'NOTE',
            'email': 'EMAIL',
            'call': 'CALL',
            'meeting': 'MEETING',
            'note': 'NOTE',
            'task': 'TASK'
        };
        return mapping[type] || 'NOTE';
    }
}

module.exports = new HubSpotIntegration();
```

## 5. Salesforce LINE Connector

```javascript
// src/integrations/salesforce.js
const jsforce = require('jsforce');
const db = require('../database/mysql');

class SalesforceIntegration {
    constructor() {
        this.conn = new jsforce.Connection({
            loginUrl: process.env.SALESFORCE_LOGIN_URL || 'https://login.salesforce.com'
        });
        this.isConnected = false;
    }

    // เชื่อมต่อ Salesforce
    async connect() {
        if (this.isConnected) return;
        
        await this.conn.login(
            process.env.SALESFORCE_USERNAME,
            process.env.SALESFORCE_PASSWORD + process.env.SALESFORCE_SECURITY_TOKEN
        );
        
        this.isConnected = true;
        console.log('Connected to Salesforce');
    }

    // สร้าง Lead ใน Salesforce
    async createLead(contactData) {
        await this.connect();
        
        try {
            const result = await this.conn.sobject('Lead').create({
                FirstName: contactData.firstName || contactData.displayName,
                LastName: contactData.lastName || 'LINE User',
                Company: contactData.company || 'Unknown',
                Email: contactData.email,
                Phone: contactData.phone,
                LeadSource: 'LINE OA',
                Status: 'New',
                Description: `LINE User ID: ${contactData.lineUserId}`,
                
                // Custom Fields (ต้องสร้างใน Salesforce ก่อน)
                LINE_User_ID__c: contactData.lineUserId,
                LINE_Display_Name__c: contactData.displayName,
                LINE_Picture_URL__c: contactData.pictureUrl
            });
            
            return result.id;
        } catch (error) {
            console.error('Salesforce create lead error:', error);
            throw error;
        }
    }

    // อัพเดท Lead ใน Salesforce
    async updateLead(leadId, updateData) {
        await this.connect();
        
        try {
            await this.conn.sobject('Lead').update({
                Id: leadId,
                ...updateData
            });
        } catch (error) {
            console.error('Salesforce update lead error:', error);
            throw error;
        }
    }

    // บันทึก Task/Activity ใน Salesforce
    async logTask(leadId, taskData) {
        await this.connect();
        
        try {
            const result = await this.conn.sobject('Task').create({
                WhoId: leadId,
                Subject: taskData.subject || 'LINE OA Interaction',
                Description: taskData.body,
                ActivityDate: new Date().toISOString().split('T')[0],
                Status: 'Completed',
                Priority: 'Normal',
                Type: 'Call',
                
                // Custom field
                Interaction_Channel__c: 'LINE OA'
            });
            
            return result.id;
        } catch (error) {
            console.error('Salesforce log task error:', error);
            throw error;
        }
    }

    // ค้นหา Lead จาก LINE User ID
    async findLeadByLineUserId(lineUserId) {
        await this.connect();
        
        try {
            const result = await this.conn.query(
                `SELECT Id, FirstName, LastName, Email, Phone, Status, LeadSource 
                 FROM Lead 
                 WHERE LINE_User_ID__c = '${lineUserId}'
                 LIMIT 1`
            );
            
            return result.records.length > 0 ? result.records[0] : null;
        } catch (error) {
            console.error('Salesforce find lead error:', error);
            throw error;
        }
    }

    // Sync ข้อมูลจาก Salesforce ไป Local DB
    async syncFromSalesforce() {
        await this.connect();
        
        try {
            const result = await this.conn.query(`
                SELECT Id, FirstName, LastName, Email, Phone, Company,
                       Status, LeadSource, LINE_User_ID__c, LINE_Display_Name__c,
                       LastModifiedDate
                FROM Lead
                WHERE LINE_User_ID__c != null
                AND LastModifiedDate = LAST_N_DAYS:1
            `);
            
            for (const lead of result.records) {
                if (!lead.LINE_User_ID__c) continue;
                
                await db.query(`
                    UPDATE contacts 
                    SET 
                        email = COALESCE(?, email),
                        phone = COALESCE(?, phone),
                        company = COALESCE(?, company),
                        custom_fields = JSON_SET(
                            COALESCE(custom_fields, '{}'),
                            '$.salesforce_id', ?,
                            '$.salesforce_synced_at', NOW()
                        )
                    WHERE line_user_id = ?
                `, [
                    lead.Email,
                    lead.Phone,
                    lead.Company,
                    lead.Id,
                    lead.LINE_User_ID__c
                ]);
            }
            
            console.log(`Synced ${result.records.length} leads from Salesforce`);
        } catch (error) {
            console.error('Salesforce sync error:', error);
            throw error;
        }
    }
}

module.exports = new SalesforceIntegration();
```

## 6. Custom CRM ด้วย Node.js + MySQL

```javascript
// src/crm/pipeline-manager.js
const db = require('../database/mysql');
const { v4: uuidv4 } = require('uuid');

class PipelineManager {
    // สร้าง Deal ใหม่
    async createDeal(contactId, dealData) {
        const dealId = uuidv4();
        
        await db.query(`
            INSERT INTO deals (
                id, contact_id, title, amount, currency,
                stage, probability, close_date, source, description
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        `, [
            dealId,
            contactId,
            dealData.title,
            dealData.amount || 0,
            dealData.currency || 'THB',
            dealData.stage || 'appointment_scheduled',
            dealData.probability || this.stageToProbability(dealData.stage),
            dealData.closeDate || null,
            dealData.source || 'line_oa',
            dealData.description || null
        ]);
        
        return dealId;
    }

    // อัพเดท Stage ของ Deal
    async updateDealStage(dealId, newStage, notes = null) {
        const probability = this.stageToProbability(newStage);
        
        await db.query(`
            UPDATE deals 
            SET stage = ?, probability = ?,
                ${newStage === 'closed_won' || newStage === 'closed_lost' ? 'closed_at = NOW(),' : ''}
                updated_at = NOW()
            WHERE id = ?
        `, [newStage, probability, dealId]);
        
        // บันทึก History
        if (notes) {
            const deal = await db.query('SELECT contact_id FROM deals WHERE id = ?', [dealId]);
            if (deal.length > 0) {
                await db.query(`
                    INSERT INTO activities (id, contact_id, deal_id, type, subject, body)
                    VALUES (?, ?, ?, 'note', ?, ?)
                `, [
                    uuidv4(),
                    deal[0].contact_id,
                    dealId,
                    `Deal Stage Changed to ${newStage}`,
                    notes
                ]);
            }
        }
    }

    // ดึง Pipeline Overview
    async getPipelineOverview(pipelineId = null) {
        const stages = [
            'appointment_scheduled',
            'qualified_to_buy',
            'presentation_scheduled',
            'decision_maker_bought_in',
            'contract_sent',
            'closed_won',
            'closed_lost'
        ];
        
        const results = await db.query(`
            SELECT 
                stage,
                COUNT(*) as deal_count,
                SUM(amount) as total_value,
                AVG(amount) as avg_value
            FROM deals
            WHERE closed_at IS NULL OR stage IN ('closed_won', 'closed_lost')
            GROUP BY stage
        `);
        
        return stages.map(stage => {
            const data = results.find(r => r.stage === stage) || {};
            return {
                stage,
                dealCount: data.deal_count || 0,
                totalValue: data.total_value || 0,
                avgValue: data.avg_value || 0,
                probability: this.stageToProbability(stage)
            };
        });
    }

    // คำนวณ Weighted Pipeline Value
    async getWeightedPipelineValue() {
        const deals = await db.query(`
            SELECT amount, stage, probability
            FROM deals
            WHERE stage NOT IN ('closed_won', 'closed_lost')
        `);
        
        const weightedValue = deals.reduce((total, deal) => {
            return total + (deal.amount * (deal.probability / 100));
        }, 0);
        
        return weightedValue;
    }

    stageToProbability(stage) {
        const mapping = {
            'appointment_scheduled': 20,
            'qualified_to_buy': 40,
            'presentation_scheduled': 60,
            'decision_maker_bought_in': 80,
            'contract_sent': 90,
            'closed_won': 100,
            'closed_lost': 0
        };
        return mapping[stage] || 0;
    }
}

module.exports = new PipelineManager();
```

## 7. Customer Journey Stages

```javascript
// src/crm/journey-manager.js
const db = require('../database/mysql');
const lineClient = require('../line/client');

class CustomerJourneyManager {
    constructor() {
        this.journeyStages = {
            awareness: {
                triggers: ['first_message', 'followed_account'],
                actions: ['send_welcome', 'add_to_nurture_list'],
                nextStage: 'consideration'
            },
            consideration: {
                triggers: ['product_inquiry', 'price_check', 'viewed_catalog'],
                actions: ['send_product_info', 'assign_sales_rep'],
                nextStage: 'decision'
            },
            decision: {
                triggers: ['demo_request', 'quote_request', 'comparison_check'],
                actions: ['schedule_demo', 'send_proposal', 'offer_trial'],
                nextStage: 'purchase'
            },
            purchase: {
                triggers: ['order_placed', 'payment_received'],
                actions: ['send_confirmation', 'assign_account_manager'],
                nextStage: 'retention'
            },
            retention: {
                triggers: ['repeat_purchase', 'referral_made'],
                actions: ['send_loyalty_reward', 'request_review'],
                nextStage: 'advocacy'
            },
            advocacy: {
                triggers: ['review_posted', 'referred_friend'],
                actions: ['send_ambassador_invite', 'give_special_discount'],
                nextStage: null
            }
        };
    }

    // ตรวจจับและอัพเดท Journey Stage
    async processJourneyEvent(lineUserId, eventType, eventData = {}) {
        try {
            const contact = await db.query(
                'SELECT * FROM contacts WHERE line_user_id = ?',
                [lineUserId]
            );
            
            if (!contact.length) return;
            
            const c = contact[0];
            const currentStage = c.lifecycle_stage;
            
            // ตรวจสอบว่า Event นี้ trigger อะไร
            for (const [stageName, stageConfig] of Object.entries(this.journeyStages)) {
                if (stageConfig.triggers.includes(eventType)) {
                    // ดำเนิน Actions
                    await this.executeJourneyActions(
                        lineUserId,
                        c.id,
                        stageConfig.actions,
                        eventData
                    );
                    
                    // อัพเดท Stage ถ้าจำเป็น
                    if (this.shouldAdvanceStage(currentStage, stageName)) {
                        await this.advanceJourneyStage(c.id, stageName);
                    }
                    
                    break;
                }
            }
        } catch (error) {
            console.error('Error processing journey event:', error);
        }
    }

    // ตรวจสอบว่าควร Advance Stage หรือไม่
    shouldAdvanceStage(currentStage, targetStage) {
        const stageOrder = [
            'subscriber', 'lead', 'marketing_qualified',
            'sales_qualified', 'opportunity', 'customer', 'evangelist'
        ];
        
        const currentIndex = stageOrder.indexOf(currentStage);
        
        const stageMapping = {
            'awareness': 'subscriber',
            'consideration': 'lead',
            'decision': 'marketing_qualified',
            'purchase': 'customer',
            'retention': 'customer',
            'advocacy': 'evangelist'
        };
        
        const targetIndex = stageOrder.indexOf(stageMapping[targetStage] || currentStage);
        
        return targetIndex > currentIndex;
    }

    // อัพเดท Journey Stage
    async advanceJourneyStage(contactId, newJourneyStage) {
        const stageMapping = {
            'awareness': 'subscriber',
            'consideration': 'lead',
            'decision': 'marketing_qualified',
            'purchase': 'customer',
            'retention': 'customer',
            'advocacy': 'evangelist'
        };
        
        const lifecycleStage = stageMapping[newJourneyStage];
        if (lifecycleStage) {
            await db.query(
                'UPDATE contacts SET lifecycle_stage = ? WHERE id = ?',
                [lifecycleStage, contactId]
            );
        }
    }

    // ดำเนิน Journey Actions
    async executeJourneyActions(lineUserId, contactId, actions, eventData) {
        for (const action of actions) {
            try {
                switch (action) {
                    case 'send_welcome':
                        await this.sendWelcomeMessage(lineUserId);
                        break;
                    case 'send_product_info':
                        await this.sendProductInfo(lineUserId, eventData.productId);
                        break;
                    case 'schedule_demo':
                        await this.scheduleDemo(lineUserId, contactId);
                        break;
                    case 'send_proposal':
                        await this.sendProposal(lineUserId, contactId, eventData);
                        break;
                    case 'send_loyalty_reward':
                        await this.sendLoyaltyReward(lineUserId, contactId);
                        break;
                    case 'assign_sales_rep':
                        await this.assignSalesRep(contactId);
                        break;
                }
            } catch (error) {
                console.error(`Error executing action ${action}:`, error);
            }
        }
    }

    // ส่งข้อความต้อนรับ
    async sendWelcomeMessage(lineUserId) {
        await lineClient.pushMessage(lineUserId, [
            {
                type: 'text',
                text: 'ยินดีต้อนรับสู่ LINE OA ของเรา! 🎉\n\nเราพร้อมให้บริการคุณตลอด 24 ชั่วโมง\n\nคุณสามารถ:\n📦 ดูสินค้าของเรา\n💬 พูดคุยกับทีมงาน\n❓ ถามคำถาม'
            },
            {
                type: 'template',
                altText: 'เริ่มต้นใช้งาน',
                template: {
                    type: 'buttons',
                    text: 'คุณต้องการทำอะไร?',
                    actions: [
                        { type: 'message', label: '🛍️ ดูสินค้า', text: 'ดูสินค้า' },
                        { type: 'message', label: '💬 ติดต่อทีมงาน', text: 'ติดต่อทีมงาน' },
                        { type: 'message', label: '❓ คำถามที่พบบ่อย', text: 'FAQ' }
                    ]
                }
            }
        ]);
    }
}

module.exports = new CustomerJourneyManager();
```

## 8. Follow-up Automation Sequences

```javascript
// src/crm/followup-automation.js
const db = require('../database/mysql');
const lineClient = require('../line/client');
const cron = require('node-cron');

class FollowupAutomation {
    constructor() {
        this.sequences = {
            // Sequence สำหรับ Lead ใหม่
            new_lead: [
                { delay: 0, action: 'send_welcome', template: 'welcome_message' },
                { delay: 1440, action: 'send_product_info', template: 'product_overview' },  // 24 ชั่วโมง
                { delay: 4320, action: 'send_case_study', template: 'success_stories' },     // 3 วัน
                { delay: 10080, action: 'send_offer', template: 'limited_offer' }             // 7 วัน
            ],
            
            // Sequence สำหรับ Cart Abandonment
            cart_abandoned: [
                { delay: 60, action: 'cart_reminder', template: 'cart_reminder_1h' },        // 1 ชั่วโมง
                { delay: 1440, action: 'cart_reminder', template: 'cart_reminder_24h' },      // 24 ชั่วโมง
                { delay: 4320, action: 'cart_discount', template: 'cart_discount_offer' }     // 3 วัน
            ],
            
            // Sequence หลังการซื้อ
            post_purchase: [
                { delay: 0, action: 'order_confirmation', template: 'order_confirm' },
                { delay: 2880, action: 'shipping_update', template: 'shipping_info' },        // 2 วัน
                { delay: 10080, action: 'satisfaction_survey', template: 'survey_request' },  // 7 วัน
                { delay: 30240, action: 'upsell', template: 'upsell_offer' }                 // 21 วัน
            ],
            
            // Sequence สำหรับ Re-engagement
            reengagement: [
                { delay: 0, action: 'miss_you', template: 'reengagement_1' },
                { delay: 4320, action: 'special_offer', template: 'reengagement_offer' },    // 3 วัน
                { delay: 10080, action: 'last_chance', template: 'final_offer' }              // 7 วัน
            ]
        };
    }

    // เริ่ม Sequence
    async startSequence(lineUserId, sequenceName, data = {}) {
        try {
            const contact = await db.query(
                'SELECT id FROM contacts WHERE line_user_id = ?',
                [lineUserId]
            );
            
            if (!contact.length) return;
            
            const sequence = this.sequences[sequenceName];
            if (!sequence) {
                console.error(`Sequence ${sequenceName} not found`);
                return;
            }
            
            // บันทึก sequence ที่กำลัง run
            await db.query(`
                INSERT INTO automation_sequences (
                    id, contact_id, sequence_name, status, data, started_at
                ) VALUES (UUID(), ?, ?, 'running', ?, NOW())
                ON DUPLICATE KEY UPDATE status = 'running', started_at = NOW()
            `, [contact[0].id, sequenceName, JSON.stringify(data)]);
            
            // Schedule แต่ละ Step
            for (const step of sequence) {
                await this.scheduleStep(lineUserId, contact[0].id, sequenceName, step, data);
            }
        } catch (error) {
            console.error('Error starting sequence:', error);
        }
    }

    // Schedule Step
    async scheduleStep(lineUserId, contactId, sequenceName, step, data) {
        const runAt = new Date(Date.now() + step.delay * 60 * 1000);
        
        await db.query(`
            INSERT INTO scheduled_messages (
                id, contact_id, line_user_id, sequence_name, step_action,
                template_name, data, run_at, status
            ) VALUES (UUID(), ?, ?, ?, ?, ?, ?, ?, 'pending')
        `, [
            contactId,
            lineUserId,
            sequenceName,
            step.action,
            step.template,
            JSON.stringify(data),
            runAt
        ]);
    }

    // รัน Scheduler (ทุก 1 นาที)
    startScheduler() {
        cron.schedule('* * * * *', async () => {
            await this.processScheduledMessages();
        });
        
        console.log('Follow-up automation scheduler started');
    }

    // ประมวลผล Scheduled Messages
    async processScheduledMessages() {
        try {
            const pendingMessages = await db.query(`
                SELECT * FROM scheduled_messages
                WHERE status = 'pending' AND run_at <= NOW()
                LIMIT 50
            `);
            
            for (const msg of pendingMessages) {
                try {
                    // ตรวจสอบว่า Sequence ยัง active อยู่ไหม
                    const seqStatus = await db.query(`
                        SELECT status FROM automation_sequences
                        WHERE contact_id = ? AND sequence_name = ?
                    `, [msg.contact_id, msg.sequence_name]);
                    
                    if (!seqStatus.length || seqStatus[0].status !== 'running') {
                        // ยกเลิก Message
                        await db.query(
                            "UPDATE scheduled_messages SET status = 'cancelled' WHERE id = ?",
                            [msg.id]
                        );
                        continue;
                    }
                    
                    // ส่ง Message
                    await this.sendTemplateMessage(
                        msg.line_user_id,
                        msg.template_name,
                        JSON.parse(msg.data || '{}')
                    );
                    
                    // อัพเดทสถานะ
                    await db.query(
                        "UPDATE scheduled_messages SET status = 'sent', sent_at = NOW() WHERE id = ?",
                        [msg.id]
                    );
                } catch (error) {
                    console.error(`Error processing message ${msg.id}:`, error);
                    
                    await db.query(
                        "UPDATE scheduled_messages SET status = 'failed', error = ? WHERE id = ?",
                        [error.message, msg.id]
                    );
                }
            }
        } catch (error) {
            console.error('Error processing scheduled messages:', error);
        }
    }

    // หยุด Sequence
    async stopSequence(lineUserId, sequenceName) {
        const contact = await db.query(
            'SELECT id FROM contacts WHERE line_user_id = ?',
            [lineUserId]
        );
        
        if (!contact.length) return;
        
        await db.query(`
            UPDATE automation_sequences 
            SET status = 'stopped', stopped_at = NOW()
            WHERE contact_id = ? AND sequence_name = ?
        `, [contact[0].id, sequenceName]);
        
        // ยกเลิก Pending Messages
        await db.query(`
            UPDATE scheduled_messages 
            SET status = 'cancelled'
            WHERE contact_id = ? AND sequence_name = ? AND status = 'pending'
        `, [contact[0].id, sequenceName]);
    }

    // ส่ง Template Message
    async sendTemplateMessage(lineUserId, templateName, data) {
        const templates = {
            welcome_message: {
                type: 'text',
                text: `สวัสดีครับ/ค่ะ ${data.name || ''}! ยินดีต้อนรับสู่ครอบครัวของเรา 🎉`
            },
            cart_reminder_1h: {
                type: 'template',
                altText: 'คุณยังมีสินค้าในตะกร้า',
                template: {
                    type: 'buttons',
                    text: `คุณยังมี ${data.itemCount || 0} รายการในตะกร้า มูลค่า ${data.totalAmount || 0} บาท`,
                    actions: [
                        { type: 'uri', label: '🛒 ดูตะกร้าสินค้า', uri: data.cartUrl || '#' },
                        { type: 'message', label: '❌ ลบตะกร้า', text: 'ลบตะกร้าสินค้า' }
                    ]
                }
            }
        };
        
        const template = templates[templateName];
        if (!template) {
            console.error(`Template ${templateName} not found`);
            return;
        }
        
        await lineClient.pushMessage(lineUserId, template);
    }
}

module.exports = new FollowupAutomation();
```

## 9. Email + LINE Multi-channel Campaigns

```javascript
// src/crm/multichannel-campaign.js
const nodemailer = require('nodemailer');
const lineClient = require('../line/client');
const db = require('../database/mysql');

class MultichannelCampaignManager {
    constructor() {
        this.emailTransporter = nodemailer.createTransport({
            host: process.env.SMTP_HOST,
            port: process.env.SMTP_PORT || 587,
            secure: false,
            auth: {
                user: process.env.SMTP_USER,
                pass: process.env.SMTP_PASS
            }
        });
    }

    // สร้างและรัน Campaign
    async runCampaign(campaignId) {
        const campaign = await db.query(
            'SELECT * FROM campaigns WHERE id = ?',
            [campaignId]
        );
        
        if (!campaign.length) throw new Error('Campaign not found');
        
        const c = campaign[0];
        const targetContacts = await this.getTargetContacts(c.target_segment);
        
        // อัพเดทสถานะ Campaign
        await db.query(
            "UPDATE campaigns SET status = 'sending', target_count = ? WHERE id = ?",
            [targetContacts.length, campaignId]
        );
        
        let sentCount = 0;
        let failedCount = 0;
        
        for (const contact of targetContacts) {
            try {
                const content = JSON.parse(c.content);
                
                // ส่งตาม channel
                if (c.type === 'line_broadcast' || c.type === 'multi_channel') {
                    if (contact.line_user_id) {
                        await this.sendLineMessage(contact.line_user_id, content.line);
                        sentCount++;
                    }
                }
                
                if (c.type === 'email' || c.type === 'multi_channel') {
                    if (contact.email) {
                        await this.sendEmail(contact.email, content.email, contact);
                        sentCount++;
                    }
                }
                
                // บันทึก Activity
                await db.query(`
                    INSERT INTO activities (id, contact_id, type, subject, body, occurred_at)
                    VALUES (UUID(), ?, 'campaign_sent', ?, ?, NOW())
                `, [contact.id, `Campaign: ${c.name}`, `Campaign ID: ${campaignId}`]);
                
            } catch (error) {
                console.error(`Error sending to contact ${contact.id}:`, error);
                failedCount++;
            }
        }
        
        // อัพเดทผลลัพธ์
        await db.query(`
            UPDATE campaigns 
            SET status = 'sent', sent_count = ?, sent_at = NOW()
            WHERE id = ?
        `, [sentCount, campaignId]);
        
        return { sentCount, failedCount, total: targetContacts.length };
    }

    // ดึง Target Contacts ตาม Segment
    async getTargetContacts(segmentConfig) {
        const config = typeof segmentConfig === 'string' 
            ? JSON.parse(segmentConfig) 
            : segmentConfig;
        
        let query = 'SELECT * FROM contacts WHERE 1=1';
        const params = [];
        
        if (config.lifecycle_stage) {
            query += ' AND lifecycle_stage IN (?)';
            params.push(config.lifecycle_stage);
        }
        
        if (config.min_lead_score) {
            query += ' AND lead_score >= ?';
            params.push(config.min_lead_score);
        }
        
        if (config.tags && config.tags.length > 0) {
            const tagConditions = config.tags.map(() => 'JSON_CONTAINS(tags, ?)').join(' OR ');
            query += ` AND (${tagConditions})`;
            config.tags.forEach(tag => params.push(JSON.stringify(tag)));
        }
        
        if (config.last_seen_days) {
            query += ` AND last_seen_at >= DATE_SUB(NOW(), INTERVAL ? DAY)`;
            params.push(config.last_seen_days);
        }
        
        return await db.query(query, params);
    }

    // ส่ง LINE Message
    async sendLineMessage(lineUserId, content) {
        if (!content) return;
        
        const messages = Array.isArray(content) ? content : [content];
        await lineClient.pushMessage(lineUserId, messages);
    }

    // ส่ง Email
    async sendEmail(email, content, contact) {
        if (!content) return;
        
        const personalizedHtml = this.personalizeContent(content.html, contact);
        
        await this.emailTransporter.sendMail({
            from: `"${process.env.EMAIL_FROM_NAME}" <${process.env.EMAIL_FROM}>`,
            to: email,
            subject: this.personalizeContent(content.subject, contact),
            html: personalizedHtml,
            text: content.text || ''
        });
    }

    // Personalize เนื้อหา
    personalizeContent(template, contact) {
        if (!template) return '';
        
        return template
            .replace(/\{\{name\}\}/g, contact.display_name || 'ลูกค้า')
            .replace(/\{\{email\}\}/g, contact.email || '')
            .replace(/\{\{phone\}\}/g, contact.phone || '')
            .replace(/\{\{company\}\}/g, contact.company || '');
    }
}

module.exports = new MultichannelCampaignManager();
```

## 10. Customer Lifetime Value Tracking

```javascript
// src/crm/clv-calculator.js
const db = require('../database/mysql');

class CLVCalculator {
    // คำนวณ CLV สำหรับ Contact
    async calculateCLV(contactId) {
        try {
            // ดึงข้อมูลการซื้อ
            const purchases = await db.query(`
                SELECT 
                    COUNT(*) as total_purchases,
                    SUM(amount) as total_revenue,
                    AVG(amount) as avg_order_value,
                    MIN(created_at) as first_purchase,
                    MAX(created_at) as last_purchase,
                    DATEDIFF(MAX(created_at), MIN(created_at)) as customer_lifespan_days
                FROM deals
                WHERE contact_id = ? AND stage = 'closed_won'
            `, [contactId]);
            
            const p = purchases[0];
            
            if (!p.total_purchases) {
                return {
                    contactId,
                    clv: 0,
                    segment: 'low_value',
                    churnProbability: 0.8
                };
            }
            
            // คำนวณ Purchase Frequency (ครั้ง/เดือน)
            const lifespanMonths = Math.max(1, p.customer_lifespan_days / 30);
            const purchaseFrequency = p.total_purchases / lifespanMonths;
            
            // คำนวณ Customer Value ต่อเดือน
            const monthlyValue = p.avg_order_value * purchaseFrequency;
            
            // คำนวณ Average Customer Lifespan (คาดการณ์ 24 เดือน)
            const avgLifespan = Math.min(24, Math.max(3, lifespanMonths * 1.5));
            
            // CLV = Monthly Value × Average Customer Lifespan
            const clv = monthlyValue * avgLifespan;
            
            // คำนวณ Churn Probability
            const daysSinceLastPurchase = p.last_purchase 
                ? Math.floor((Date.now() - new Date(p.last_purchase).getTime()) / (1000 * 60 * 60 * 24))
                : 365;
            
            const churnProbability = Math.min(0.95, daysSinceLastPurchase / (lifespanMonths * 30 * 2));
            
            // กำหนด Segment
            const segment = this.determineSegment(clv, p.total_purchases);
            
            // อัพเดท Database
            await db.query(`
                INSERT INTO customer_lifetime_value (
                    id, contact_id, total_purchases, total_revenue, 
                    average_order_value, first_purchase_date, last_purchase_date,
                    predicted_clv, clv_segment, churn_probability, updated_at
                ) VALUES (UUID(), ?, ?, ?, ?, ?, ?, ?, ?, ?, NOW())
                ON DUPLICATE KEY UPDATE
                    total_purchases = VALUES(total_purchases),
                    total_revenue = VALUES(total_revenue),
                    average_order_value = VALUES(average_order_value),
                    last_purchase_date = VALUES(last_purchase_date),
                    predicted_clv = VALUES(predicted_clv),
                    clv_segment = VALUES(clv_segment),
                    churn_probability = VALUES(churn_probability),
                    updated_at = NOW()
            `, [
                contactId,
                p.total_purchases,
                p.total_revenue,
                p.avg_order_value,
                p.first_purchase,
                p.last_purchase,
                clv,
                segment,
                churnProbability
            ]);
            
            return {
                contactId,
                clv: Math.round(clv),
                segment,
                totalRevenue: p.total_revenue,
                avgOrderValue: p.avg_order_value,
                purchaseFrequency,
                churnProbability: Math.round(churnProbability * 100) / 100
            };
        } catch (error) {
            console.error('Error calculating CLV:', error);
            throw error;
        }
    }

    // กำหนด CLV Segment
    determineSegment(clv, totalPurchases) {
        if (clv >= 100000 || totalPurchases >= 10) return 'high_value';
        if (clv >= 30000 || totalPurchases >= 5) return 'mid_value';
        if (totalPurchases >= 1) return 'low_value';
        return 'potential';
    }

    // คำนวณ CLV ทั้งหมด (Batch Job)
    async calculateAllCLV() {
        const contacts = await db.query(
            "SELECT DISTINCT contact_id FROM deals WHERE stage = 'closed_won'"
        );
        
        const results = [];
        for (const c of contacts) {
            const result = await this.calculateCLV(c.contact_id);
            results.push(result);
        }
        
        console.log(`Calculated CLV for ${results.length} customers`);
        return results;
    }

    // ดึง High Value Customers
    async getHighValueCustomers(limit = 100) {
        return await db.query(`
            SELECT 
                c.id, c.display_name, c.email, c.phone,
                c.line_user_id, clv.predicted_clv, clv.clv_segment,
                clv.total_purchases, clv.total_revenue, clv.churn_probability
            FROM contacts c
            JOIN customer_lifetime_value clv ON c.id = clv.contact_id
            WHERE clv.clv_segment = 'high_value'
            ORDER BY clv.predicted_clv DESC
            LIMIT ?
        `, [limit]);
    }

    // ดึง At-Risk Customers (ใกล้จะ Churn)
    async getAtRiskCustomers() {
        return await db.query(`
            SELECT 
                c.id, c.display_name, c.line_user_id,
                clv.predicted_clv, clv.churn_probability,
                clv.last_purchase_date
            FROM contacts c
            JOIN customer_lifetime_value clv ON c.id = clv.contact_id
            WHERE 
                clv.churn_probability >= 0.7
                AND clv.clv_segment IN ('high_value', 'mid_value')
            ORDER BY clv.predicted_clv DESC
        `);
    }
}

module.exports = new CLVCalculator();
```

## 11. Complete CRM Pipeline Implementation

```javascript
// src/crm/crm-webhook-handler.js
const express = require('express');
const router = express.Router();
const leadManager = require('./lead-manager');
const journeyManager = require('./journey-manager');
const followupAutomation = require('./followup-automation');
const clvCalculator = require('./clv-calculator');
const contactEnrichment = require('./contact-enrichment');
const hubspot = require('../integrations/hubspot');
const db = require('../database/mysql');

// LINE Webhook Handler พร้อม CRM Integration
router.post('/line/webhook', async (req, res) => {
    res.sendStatus(200);
    
    const events = req.body.events;
    
    for (const event of events) {
        try {
            await processLineEvent(event);
        } catch (error) {
            console.error('Error processing LINE event:', error);
        }
    }
});

async function processLineEvent(event) {
    const lineUserId = event.source.userId;
    
    switch (event.type) {
        case 'follow':
            await handleFollow(lineUserId);
            break;
            
        case 'unfollow':
            await handleUnfollow(lineUserId);
            break;
            
        case 'message':
            await handleMessage(lineUserId, event.message, event.replyToken);
            break;
            
        case 'postback':
            await handlePostback(lineUserId, event.postback.data, event.replyToken);
            break;
    }
}

async function handleFollow(lineUserId) {
    // สร้าง/อัพเดท Contact
    const contact = await leadManager.upsertContact(lineUserId);
    
    // เริ่ม Welcome Sequence
    await followupAutomation.startSequence(lineUserId, 'new_lead', {
        name: contact.display_name
    });
    
    // บันทึก Activity
    await leadManager.logActivity(contact.id, {
        type: 'line_message',
        direction: 'inbound',
        subject: 'User Followed LINE OA',
        body: 'ผู้ใช้ติดตาม LINE OA'
    });
    
    // Sync ไป HubSpot
    if (process.env.HUBSPOT_API_KEY) {
        try {
            await hubspot.upsertContact(lineUserId, {
                displayName: contact.display_name,
                pictureUrl: contact.picture_url,
                lifecycleStage: 'subscriber'
            });
        } catch (e) {
            console.error('HubSpot sync error:', e.message);
        }
    }
    
    // Process Journey
    await journeyManager.processJourneyEvent(lineUserId, 'followed_account');
}

async function handleUnfollow(lineUserId) {
    // อัพเดทสถานะ
    await db.query(
        "UPDATE contacts SET lifecycle_stage = 'other', lead_status = 'unqualified' WHERE line_user_id = ?",
        [lineUserId]
    );
    
    // หยุด Sequences ทั้งหมด
    const sequences = ['new_lead', 'cart_abandoned', 'reengagement'];
    for (const seq of sequences) {
        await followupAutomation.stopSequence(lineUserId, seq);
    }
}

async function handleMessage(lineUserId, message, replyToken) {
    // Upsert Contact
    const contact = await leadManager.upsertContact(lineUserId);
    
    // วิเคราะห์ Intent
    let intents = [];
    if (message.type === 'text') {
        intents = leadManager.analyzeMessageIntent(message.text);
    }
    
    // บันทึก Activity
    await leadManager.logActivity(contact.id, {
        type: 'line_message',
        direction: 'inbound',
        subject: `LINE Message: ${message.type}`,
        body: message.type === 'text' ? message.text : `[${message.type}]`,
        lineMessageType: message.type,
        lineMessageId: message.id
    });
    
    // Process Intents
    for (const intent of intents) {
        await journeyManager.processJourneyEvent(lineUserId, intent.type, {
            message: message.text,
            score: intent.score
        });
        
        // ถ้ามี High Intent ให้หยุด Nurture Sequence
        if (intent.type === 'demo_request' || intent.type === 'price_inquiry') {
            await followupAutomation.stopSequence(lineUserId, 'new_lead');
        }
    }
    
    // Extract ข้อมูลจากการสนทนา
    const recentMessages = await db.query(`
        SELECT body, direction FROM activities
        WHERE contact_id = ? AND type = 'line_message'
        ORDER BY occurred_at DESC
        LIMIT 20
    `, [contact.id]);
    
    const extractedData = await contactEnrichment.extractDataFromConversation(
        contact.id,
        recentMessages
    );
    
    // ถ้าได้ Email ใหม่ ให้ Enrich
    if (extractedData.email && !contact.email) {
        await contactEnrichment.enrichFromEmail(contact.id, extractedData.email);
    }
}

async function handlePostback(lineUserId, postbackData, replyToken) {
    const data = new URLSearchParams(postbackData);
    const action = data.get('action');
    
    const contact = await db.query(
        'SELECT id FROM contacts WHERE line_user_id = ?',
        [lineUserId]
    );
    
    if (!contact.length) return;
    
    switch (action) {
        case 'view_product':
            await journeyManager.processJourneyEvent(lineUserId, 'product_inquiry', {
                productId: data.get('productId')
            });
            break;
            
        case 'request_demo':
            await journeyManager.processJourneyEvent(lineUserId, 'demo_request');
            await followupAutomation.stopSequence(lineUserId, 'new_lead');
            break;
            
        case 'purchase_complete':
            await journeyManager.processJourneyEvent(lineUserId, 'order_placed', {
                orderId: data.get('orderId'),
                amount: data.get('amount')
            });
            await followupAutomation.startSequence(lineUserId, 'post_purchase');
            await clvCalculator.calculateCLV(contact[0].id);
            break;
    }
}

// API Endpoints สำหรับ CRM Dashboard
router.get('/api/crm/contacts', async (req, res) => {
    try {
        const { stage, page = 1, limit = 20 } = req.query;
        const offset = (page - 1) * limit;
        
        let query = 'SELECT * FROM contacts WHERE 1=1';
        const params = [];
        
        if (stage) {
            query += ' AND lifecycle_stage = ?';
            params.push(stage);
        }
        
        query += ' ORDER BY lead_score DESC LIMIT ? OFFSET ?';
        params.push(parseInt(limit), parseInt(offset));
        
        const contacts = await db.query(query, params);
        const total = await db.query('SELECT COUNT(*) as count FROM contacts');
        
        res.json({
            data: contacts,
            total: total[0].count,
            page: parseInt(page),
            limit: parseInt(limit)
        });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

router.get('/api/crm/pipeline', async (req, res) => {
    try {
        const pipelineManager = require('./pipeline-manager');
        const overview = await pipelineManager.getPipelineOverview();
        const weightedValue = await pipelineManager.getWeightedPipelineValue();
        
        res.json({ stages: overview, weightedValue });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

router.get('/api/crm/clv/high-value', async (req, res) => {
    try {
        const customers = await clvCalculator.getHighValueCustomers(50);
        res.json(customers);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

router.get('/api/crm/clv/at-risk', async (req, res) => {
    try {
        const customers = await clvCalculator.getAtRiskCustomers();
        res.json(customers);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

module.exports = router;
```

## 12. ตาราง Database เพิ่มเติม

```sql
-- ตาราง automation_sequences
CREATE TABLE automation_sequences (
    id VARCHAR(36) PRIMARY KEY,
    contact_id VARCHAR(36) NOT NULL,
    sequence_name VARCHAR(100) NOT NULL,
    status ENUM('running', 'completed', 'stopped', 'paused') DEFAULT 'running',
    data JSON,
    started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    stopped_at TIMESTAMP,
    completed_at TIMESTAMP,
    
    UNIQUE KEY unique_contact_sequence (contact_id, sequence_name),
    FOREIGN KEY (contact_id) REFERENCES contacts(id) ON DELETE CASCADE
);

-- ตาราง scheduled_messages
CREATE TABLE scheduled_messages (
    id VARCHAR(36) PRIMARY KEY,
    contact_id VARCHAR(36) NOT NULL,
    line_user_id VARCHAR(100) NOT NULL,
    sequence_name VARCHAR(100),
    step_action VARCHAR(100),
    template_name VARCHAR(100),
    data JSON,
    run_at TIMESTAMP NOT NULL,
    status ENUM('pending', 'sent', 'failed', 'cancelled') DEFAULT 'pending',
    sent_at TIMESTAMP,
    error TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    FOREIGN KEY (contact_id) REFERENCES contacts(id) ON DELETE CASCADE,
    INDEX idx_run_at_status (run_at, status)
);
```

## 13. Environment Configuration

```env
# .env
# Database
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASS=your_password
DB_NAME=line_crm

# LINE OA
LINE_CHANNEL_ACCESS_TOKEN=your_line_token
LINE_CHANNEL_SECRET=your_channel_secret

# HubSpot
HUBSPOT_API_KEY=your_hubspot_api_key
HUBSPOT_PORTAL_ID=your_portal_id
HUBSPOT_WEBHOOK_SECRET=your_webhook_secret

# Salesforce
SALESFORCE_USERNAME=your_username
SALESFORCE_PASSWORD=your_password
SALESFORCE_SECURITY_TOKEN=your_token
SALESFORCE_LOGIN_URL=https://login.salesforce.com

# Clearbit
CLEARBIT_API_KEY=your_clearbit_key

# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your_email@gmail.com
SMTP_PASS=your_app_password
EMAIL_FROM=your_email@gmail.com
EMAIL_FROM_NAME=Your Company Name
```

## 14. Package.json

```json
{
    "name": "line-oa-crm",
    "version": "1.0.0",
    "scripts": {
        "start": "node src/index.js",
        "dev": "nodemon src/index.js",
        "migrate": "node src/database/migrate.js",
        "seed": "node src/database/seed.js"
    },
    "dependencies": {
        "@line/bot-sdk": "^9.0.0",
        "axios": "^1.6.0",
        "express": "^4.18.2",
        "jsforce": "^1.11.1",
        "mysql2": "^3.6.0",
        "node-cron": "^3.0.3",
        "nodemailer": "^6.9.7",
        "uuid": "^9.0.0"
    },
    "devDependencies": {
        "nodemon": "^3.0.1"
    }
}
```

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Customer Data Model** - โครงสร้าง Database สำหรับจัดการข้อมูลลูกค้า
2. **Lead Management** - การจัดการ Lead จาก LINE Conversations ด้วย Lead Scoring
3. **Contact Enrichment** - การเติมเต็มข้อมูลลูกค้าจากแหล่งข้อมูลต่างๆ
4. **HubSpot Integration** - การผสานกับ HubSpot CRM ผ่าน API และ Webhooks
5. **Salesforce Integration** - การเชื่อมต่อ Salesforce CRM
6. **Customer Journey** - การจัดการ Customer Journey Stages
7. **Follow-up Automation** - ระบบ Automation สำหรับ Follow-up อัตโนมัติ
8. **Multi-channel Campaigns** - การรัน Campaign ผ่าน LINE + Email
9. **CLV Tracking** - การติดตาม Customer Lifetime Value
10. **Complete Pipeline** - การรวมทุกส่วนเข้าด้วยกัน
