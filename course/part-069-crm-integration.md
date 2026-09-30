# Part 69: CRM Integration บน LINE OA

## บทนำ

ในบทนี้เราจะเชื่อมต่อ LINE OA กับระบบ CRM ทั้ง HubSpot, Salesforce และสร้าง Custom CRM ด้วย MySQL รองรับการติดตาม Customer Journey, Lead Management และ Follow-up Automation

## สิ่งที่จะได้เรียนรู้

- Customer Data Enrichment
- Lead Tracking จาก LINE
- HubSpot CRM Integration
- Salesforce Integration
- Custom CRM ด้วย MySQL
- Customer Journey Tracking
- Funnel Analysis
- Follow-up Automation
- Multi-channel (LINE + Email)
- CRM Pipeline

---

## 1. Custom CRM Database Schema

```sql
CREATE DATABASE IF NOT EXISTS lineoa_crm CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE lineoa_crm;

-- Contacts / Leads
CREATE TABLE contacts (
  id INT PRIMARY KEY AUTO_INCREMENT,
  line_user_id VARCHAR(100) UNIQUE,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  email VARCHAR(200),
  phone VARCHAR(20),
  company VARCHAR(200),
  job_title VARCHAR(200),
  picture_url TEXT,
  lifecycle_stage ENUM('subscriber','lead','mql','sql','opportunity','customer','evangelist') DEFAULT 'subscriber',
  lead_source VARCHAR(100),
  lead_score INT DEFAULT 0,
  assigned_to INT,
  tags JSON DEFAULT '[]',
  properties JSON DEFAULT '{}',
  hubspot_id VARCHAR(100),
  salesforce_id VARCHAR(100),
  do_not_contact TINYINT(1) DEFAULT 0,
  last_activity_at TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  INDEX idx_line_user_id (line_user_id),
  INDEX idx_lifecycle_stage (lifecycle_stage),
  INDEX idx_lead_score (lead_score)
);

-- Sales Team
CREATE TABLE sales_reps (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  email VARCHAR(200) UNIQUE,
  phone VARCHAR(20),
  line_user_id VARCHAR(100),
  role VARCHAR(100),
  is_active TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Deals / Opportunities
CREATE TABLE deals (
  id INT PRIMARY KEY AUTO_INCREMENT,
  contact_id INT NOT NULL,
  assigned_to INT,
  title VARCHAR(300) NOT NULL,
  value DECIMAL(15,2) DEFAULT 0,
  currency VARCHAR(10) DEFAULT 'THB',
  stage ENUM('prospecting','qualification','proposal','negotiation','closed_won','closed_lost') DEFAULT 'prospecting',
  probability INT DEFAULT 10,
  expected_close_date DATE,
  actual_close_date DATE,
  lost_reason TEXT,
  notes TEXT,
  source VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (contact_id) REFERENCES contacts(id),
  INDEX idx_stage (stage),
  INDEX idx_contact_id (contact_id)
);

-- Activities / Interactions
CREATE TABLE activities (
  id INT PRIMARY KEY AUTO_INCREMENT,
  contact_id INT NOT NULL,
  deal_id INT,
  type ENUM('message','call','email','meeting','note','task','line_message') NOT NULL,
  direction ENUM('inbound','outbound') DEFAULT 'inbound',
  subject VARCHAR(300),
  body TEXT,
  status ENUM('open','completed','cancelled') DEFAULT 'completed',
  scheduled_at TIMESTAMP NULL,
  completed_at TIMESTAMP NULL,
  created_by INT,
  metadata JSON,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (contact_id) REFERENCES contacts(id),
  FOREIGN KEY (deal_id) REFERENCES deals(id) ON DELETE SET NULL,
  INDEX idx_contact_id (contact_id),
  INDEX idx_type (type),
  INDEX idx_created_at (created_at)
);

-- Customer Journey Events
CREATE TABLE journey_events (
  id INT PRIMARY KEY AUTO_INCREMENT,
  contact_id INT NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  event_name VARCHAR(200),
  properties JSON,
  session_id VARCHAR(100),
  source VARCHAR(100) DEFAULT 'line',
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (contact_id) REFERENCES contacts(id),
  INDEX idx_contact_id (contact_id),
  INDEX idx_event_type (event_type),
  INDEX idx_created_at (created_at)
);

-- Lead Scoring Rules
CREATE TABLE scoring_rules (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  event_type VARCHAR(100) NOT NULL,
  condition_field VARCHAR(100),
  condition_value VARCHAR(200),
  points INT NOT NULL,
  is_active TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Follow-up Tasks
CREATE TABLE tasks (
  id INT PRIMARY KEY AUTO_INCREMENT,
  contact_id INT NOT NULL,
  deal_id INT,
  assigned_to INT,
  type ENUM('call','email','line_message','meeting','follow_up') NOT NULL,
  title VARCHAR(300) NOT NULL,
  description TEXT,
  due_date TIMESTAMP,
  priority ENUM('low','medium','high','urgent') DEFAULT 'medium',
  status ENUM('pending','in_progress','completed','overdue') DEFAULT 'pending',
  completed_at TIMESTAMP NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  FOREIGN KEY (contact_id) REFERENCES contacts(id),
  INDEX idx_contact_id (contact_id),
  INDEX idx_status (status),
  INDEX idx_due_date (due_date)
);

-- Email Templates
CREATE TABLE email_templates (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  subject VARCHAR(300) NOT NULL,
  body_html TEXT NOT NULL,
  body_text TEXT,
  variables JSON,
  is_active TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- LINE Message Templates
CREATE TABLE line_templates (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  type ENUM('text','flex','template') NOT NULL,
  content JSON NOT NULL,
  variables JSON,
  trigger_event VARCHAR(100),
  delay_minutes INT DEFAULT 0,
  is_active TINYINT(1) DEFAULT 1,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Automation Workflows
CREATE TABLE workflows (
  id INT PRIMARY KEY AUTO_INCREMENT,
  name VARCHAR(200) NOT NULL,
  trigger_type VARCHAR(100) NOT NULL,
  trigger_conditions JSON,
  actions JSON,
  is_active TINYINT(1) DEFAULT 1,
  run_count INT DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Workflow Executions
CREATE TABLE workflow_executions (
  id INT PRIMARY KEY AUTO_INCREMENT,
  workflow_id INT NOT NULL,
  contact_id INT NOT NULL,
  status ENUM('running','completed','failed') DEFAULT 'running',
  current_step INT DEFAULT 0,
  context JSON,
  started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  completed_at TIMESTAMP NULL,
  FOREIGN KEY (workflow_id) REFERENCES workflows(id),
  FOREIGN KEY (contact_id) REFERENCES contacts(id)
);

-- Lead Scoring Rules ตัวอย่าง
INSERT INTO scoring_rules (name, event_type, points) VALUES
('Follow LINE OA', 'follow', 10),
('ส่งข้อความ', 'message', 5),
('คลิก Rich Menu', 'postback', 3),
('ดูสินค้า', 'view_product', 8),
('เพิ่มลงตะกร้า', 'add_to_cart', 15),
('ซื้อสินค้า', 'purchase', 50),
('ให้คะแนน', 'review', 10),
('Share', 'share', 20);
```

## 2. Contact Service

```javascript
// src/services/contactService.js
const db = require('../config/database');
const hubspotService = require('./hubspotService');
const salesforceService = require('./salesforceService');

class ContactService {
  // สร้างหรืออัปเดต contact จาก LINE
  async upsertFromLine(lineUserId, profileData = null) {
    const [[existing]] = await db.query(
      'SELECT * FROM contacts WHERE line_user_id = ?',
      [lineUserId]
    );

    if (existing) {
      // อัปเดต last activity
      await db.query(
        'UPDATE contacts SET last_activity_at = NOW() WHERE id = ?',
        [existing.id]
      );
      return existing;
    }

    // สร้างใหม่
    const [result] = await db.query(
      `INSERT INTO contacts 
       (line_user_id, first_name, picture_url, lifecycle_stage, lead_source, last_activity_at)
       VALUES (?, ?, ?, 'subscriber', 'line', NOW())`,
      [lineUserId, profileData?.displayName || '', profileData?.pictureUrl || '']
    );

    const [[contact]] = await db.query('SELECT * FROM contacts WHERE id = ?', [result.insertId]);

    // Sync กับ HubSpot
    if (process.env.HUBSPOT_ACCESS_TOKEN) {
      try {
        const hsContact = await hubspotService.createContact({
          properties: {
            firstname: contact.first_name,
            line_user_id: lineUserId,
            lead_source: 'LINE'
          }
        });
        await db.query(
          'UPDATE contacts SET hubspot_id = ? WHERE id = ?',
          [hsContact.id, contact.id]
        );
      } catch (err) {
        console.error('HubSpot sync error:', err.message);
      }
    }

    return contact;
  }

  // อัปเดตข้อมูล contact
  async updateContact(contactId, data) {
    const allowed = ['first_name', 'last_name', 'email', 'phone', 'company', 'job_title',
                     'lifecycle_stage', 'lead_score', 'tags', 'properties'];

    const updates = {};
    for (const key of allowed) {
      if (data[key] !== undefined) {
        updates[key] = ['tags', 'properties'].includes(key) ? JSON.stringify(data[key]) : data[key];
      }
    }

    if (Object.keys(updates).length === 0) return;

    const setClause = Object.keys(updates).map(k => `${k} = ?`).join(', ');
    await db.query(
      `UPDATE contacts SET ${setClause} WHERE id = ?`,
      [...Object.values(updates), contactId]
    );

    // Sync กับ external CRM
    const [[contact]] = await db.query('SELECT * FROM contacts WHERE id = ?', [contactId]);

    if (contact.hubspot_id && process.env.HUBSPOT_ACCESS_TOKEN) {
      try {
        await hubspotService.updateContact(contact.hubspot_id, {
          properties: {
            email: data.email,
            phone: data.phone,
            company: data.company
          }
        });
      } catch (err) {
        console.error('HubSpot update error:', err.message);
      }
    }
  }

  // เพิ่ม activity
  async logActivity(contactId, type, direction, subject, body, metadata = {}) {
    const [result] = await db.query(
      `INSERT INTO activities (contact_id, type, direction, subject, body, metadata, completed_at)
       VALUES (?, ?, ?, ?, ?, ?, NOW())`,
      [contactId, type, direction, subject, body, JSON.stringify(metadata)]
    );

    await db.query(
      'UPDATE contacts SET last_activity_at = NOW() WHERE id = ?',
      [contactId]
    );

    return result.insertId;
  }

  // บันทึก Journey Event
  async trackEvent(contactId, eventType, eventName, properties = {}) {
    await db.query(
      `INSERT INTO journey_events (contact_id, event_type, event_name, properties)
       VALUES (?, ?, ?, ?)`,
      [contactId, eventType, eventName, JSON.stringify(properties)]
    );

    // คำนวณ lead score
    await this.updateLeadScore(contactId, eventType, properties);

    // trigger workflows
    await this.triggerWorkflows(contactId, eventType, properties);
  }

  // คำนวณ Lead Score
  async updateLeadScore(contactId, eventType, properties = {}) {
    const [rules] = await db.query(
      'SELECT * FROM scoring_rules WHERE event_type = ? AND is_active = 1',
      [eventType]
    );

    let totalPoints = 0;
    for (const rule of rules) {
      if (rule.condition_field && rule.condition_value) {
        if (properties[rule.condition_field] === rule.condition_value) {
          totalPoints += rule.points;
        }
      } else {
        totalPoints += rule.points;
      }
    }

    if (totalPoints !== 0) {
      await db.query(
        'UPDATE contacts SET lead_score = lead_score + ? WHERE id = ?',
        [totalPoints, contactId]
      );

      // อัปเดต lifecycle stage ตาม score
      const [[contact]] = await db.query('SELECT lead_score FROM contacts WHERE id = ?', [contactId]);
      const newStage = this.getLifecycleStage(contact.lead_score);
      await db.query(
        'UPDATE contacts SET lifecycle_stage = ? WHERE id = ? AND lifecycle_stage != ?',
        [newStage, contactId, newStage]
      );
    }
  }

  getLifecycleStage(score) {
    if (score >= 100) return 'opportunity';
    if (score >= 60) return 'sql';
    if (score >= 30) return 'mql';
    if (score >= 15) return 'lead';
    return 'subscriber';
  }

  // ดึง contacts ตาม filter
  async getContacts(filters = {}, page = 1, limit = 20) {
    const offset = (page - 1) * limit;
    let where = '1=1';
    const params = [];

    if (filters.lifecycle_stage) {
      where += ' AND lifecycle_stage = ?';
      params.push(filters.lifecycle_stage);
    }
    if (filters.min_score) {
      where += ' AND lead_score >= ?';
      params.push(filters.min_score);
    }
    if (filters.search) {
      where += ' AND (first_name LIKE ? OR last_name LIKE ? OR email LIKE ? OR phone LIKE ?)';
      const s = `%${filters.search}%`;
      params.push(s, s, s, s);
    }

    const [contacts] = await db.query(
      `SELECT * FROM contacts WHERE ${where} ORDER BY lead_score DESC, last_activity_at DESC LIMIT ? OFFSET ?`,
      [...params, limit, offset]
    );

    const [[{ total }]] = await db.query(
      `SELECT COUNT(*) as total FROM contacts WHERE ${where}`,
      params
    );

    return { contacts, total, page, limit };
  }

  // Trigger Automation Workflows
  async triggerWorkflows(contactId, eventType, properties = {}) {
    const [workflows] = await db.query(
      'SELECT * FROM workflows WHERE is_active = 1 AND trigger_type = ?',
      [eventType]
    );

    for (const wf of workflows) {
      try {
        const conditions = JSON.parse(wf.trigger_conditions || '{}');
        const conditionsMet = this.checkConditions(conditions, properties);

        if (conditionsMet) {
          // ตรวจสอบว่าเคย run workflow นี้แล้วหรือยัง
          const [[existing]] = await db.query(
            "SELECT id FROM workflow_executions WHERE workflow_id = ? AND contact_id = ? AND status = 'completed'",
            [wf.id, contactId]
          );

          if (!existing) {
            await this.executeWorkflow(wf, contactId, properties);
          }
        }
      } catch (err) {
        console.error(`Workflow ${wf.id} trigger error:`, err.message);
      }
    }
  }

  checkConditions(conditions, data) {
    if (!conditions || Object.keys(conditions).length === 0) return true;
    return Object.entries(conditions).every(([key, value]) => data[key] === value);
  }

  async executeWorkflow(workflow, contactId, context) {
    const [result] = await db.query(
      `INSERT INTO workflow_executions (workflow_id, contact_id, status, context)
       VALUES (?, ?, 'running', ?)`,
      [workflow.id, contactId, JSON.stringify(context)]
    );
    const executionId = result.insertId;

    const actions = JSON.parse(workflow.actions || '[]');

    for (let i = 0; i < actions.length; i++) {
      const action = actions[i];
      try {
        await this.executeAction(action, contactId, context);
        await db.query(
          'UPDATE workflow_executions SET current_step = ? WHERE id = ?',
          [i + 1, executionId]
        );

        // รอถ้ามี delay
        if (action.delay_minutes) {
          await new Promise(resolve => setTimeout(resolve, action.delay_minutes * 60 * 1000));
        }
      } catch (err) {
        console.error(`Action ${action.type} error:`, err.message);
      }
    }

    await db.query(
      "UPDATE workflow_executions SET status = 'completed', completed_at = NOW() WHERE id = ?",
      [executionId]
    );

    await db.query(
      'UPDATE workflows SET run_count = run_count + 1 WHERE id = ?',
      [workflow.id]
    );
  }

  async executeAction(action, contactId, context) {
    const [[contact]] = await db.query('SELECT * FROM contacts WHERE id = ?', [contactId]);
    if (!contact) return;

    switch (action.type) {
      case 'send_line_message':
        await this.sendLineMessage(contact.line_user_id, action.message);
        await this.logActivity(contactId, 'line_message', 'outbound',
          'Auto LINE message', JSON.stringify(action.message));
        break;

      case 'send_email':
        if (contact.email) {
          await this.sendEmail(contact.email, action.template_id, context);
          await this.logActivity(contactId, 'email', 'outbound', 'Auto email', action.subject);
        }
        break;

      case 'create_task':
        await db.query(
          `INSERT INTO tasks (contact_id, type, title, description, due_date, priority)
           VALUES (?, ?, ?, ?, DATE_ADD(NOW(), INTERVAL ? DAY), ?)`,
          [contactId, action.task_type, action.title, action.description,
           action.due_in_days || 1, action.priority || 'medium']
        );
        break;

      case 'update_lifecycle':
        await db.query(
          'UPDATE contacts SET lifecycle_stage = ? WHERE id = ?',
          [action.stage, contactId]
        );
        break;

      case 'add_tag':
        const tags = JSON.parse(contact.tags || '[]');
        if (!tags.includes(action.tag)) {
          tags.push(action.tag);
          await db.query('UPDATE contacts SET tags = ? WHERE id = ?', [JSON.stringify(tags), contactId]);
        }
        break;
    }
  }

  async sendLineMessage(lineUserId, message) {
    if (!lineUserId) return;
    const line = require('@line/bot-sdk');
    const client = new line.Client({ channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN });
    await client.pushMessage(lineUserId, message);
  }

  async sendEmail(to, templateId, variables) {
    const nodemailer = require('nodemailer');
    const [[template]] = await db.query(
      'SELECT * FROM email_templates WHERE id = ?', [templateId]
    );
    if (!template) return;

    let subject = template.subject;
    let html = template.body_html;

    // แทนที่ variables
    if (variables) {
      Object.entries(variables).forEach(([k, v]) => {
        subject = subject.replace(new RegExp(`{{${k}}}`, 'g'), v);
        html = html.replace(new RegExp(`{{${k}}}`, 'g'), v);
      });
    }

    const transporter = nodemailer.createTransporter({
      host: process.env.SMTP_HOST,
      port: process.env.SMTP_PORT || 587,
      auth: { user: process.env.SMTP_USER, pass: process.env.SMTP_PASS }
    });

    await transporter.sendMail({
      from: process.env.SMTP_FROM,
      to,
      subject,
      html
    });
  }
}

module.exports = new ContactService();
```

## 3. HubSpot Integration

```javascript
// src/services/hubspotService.js
const axios = require('axios');

const HUBSPOT_BASE = 'https://api.hubapi.com';

class HubSpotService {
  constructor() {
    this.accessToken = process.env.HUBSPOT_ACCESS_TOKEN;
  }

  getHeaders() {
    return {
      'Authorization': `Bearer ${this.accessToken}`,
      'Content-Type': 'application/json'
    };
  }

  // สร้าง contact
  async createContact(data) {
    const response = await axios.post(
      `${HUBSPOT_BASE}/crm/v3/objects/contacts`,
      data,
      { headers: this.getHeaders() }
    );
    return response.data;
  }

  // อัปเดต contact
  async updateContact(contactId, data) {
    const response = await axios.patch(
      `${HUBSPOT_BASE}/crm/v3/objects/contacts/${contactId}`,
      data,
      { headers: this.getHeaders() }
    );
    return response.data;
  }

  // ค้นหา contact
  async searchContact(email = null, lineUserId = null) {
    const filters = [];

    if (email) {
      filters.push({ propertyName: 'email', operator: 'EQ', value: email });
    }
    if (lineUserId) {
      filters.push({ propertyName: 'line_user_id', operator: 'EQ', value: lineUserId });
    }

    if (filters.length === 0) return null;

    const response = await axios.post(
      `${HUBSPOT_BASE}/crm/v3/objects/contacts/search`,
      {
        filterGroups: [{ filters }],
        properties: ['email', 'firstname', 'lastname', 'phone', 'line_user_id']
      },
      { headers: this.getHeaders() }
    );

    return response.data.results[0] || null;
  }

  // สร้าง Deal
  async createDeal(contactId, dealData) {
    const deal = await axios.post(
      `${HUBSPOT_BASE}/crm/v3/objects/deals`,
      {
        properties: {
          dealname: dealData.title,
          amount: dealData.value,
          dealstage: this.mapDealStage(dealData.stage),
          closedate: dealData.expected_close_date
        }
      },
      { headers: this.getHeaders() }
    );

    // เชื่อม deal กับ contact
    await axios.put(
      `${HUBSPOT_BASE}/crm/v3/objects/deals/${deal.data.id}/associations/contacts/${contactId}/deal_to_contact`,
      {},
      { headers: this.getHeaders() }
    );

    return deal.data;
  }

  mapDealStage(stage) {
    const mapping = {
      prospecting: 'appointmentscheduled',
      qualification: 'qualifiedtobuy',
      proposal: 'presentationscheduled',
      negotiation: 'decisionmakerboughtin',
      closed_won: 'closedwon',
      closed_lost: 'closedlost'
    };
    return mapping[stage] || 'appointmentscheduled';
  }

  // บันทึก activity
  async logEngagement(contactId, type, data) {
    const engagementTypes = {
      email: 'EMAIL',
      call: 'CALL',
      note: 'NOTE',
      meeting: 'MEETING'
    };

    await axios.post(
      `${HUBSPOT_BASE}/engagements/v1/engagements`,
      {
        engagement: {
          active: true,
          type: engagementTypes[type] || 'NOTE',
          timestamp: Date.now()
        },
        associations: { contactIds: [contactId] },
        metadata: data
      },
      { headers: this.getHeaders() }
    );
  }

  // ดึง Contact Properties
  async getContactProperties() {
    const response = await axios.get(
      `${HUBSPOT_BASE}/crm/v3/properties/contacts`,
      { headers: this.getHeaders() }
    );
    return response.data.results;
  }

  // Webhook handler
  async handleWebhook(events) {
    const db = require('../config/database');
    const contactService = require('./contactService');

    for (const evt of events) {
      if (evt.subscriptionType === 'contact.creation') {
        // HubSpot contact ใหม่ -> อาจ sync กลับ
        console.log('New HubSpot contact:', evt.objectId);
      } else if (evt.subscriptionType === 'deal.stageChange') {
        // Deal stage เปลี่ยน
        console.log('Deal stage changed:', evt.objectId, evt.propertyValue);

        // อัปเดต Deal ใน local DB
        await db.query(
          'UPDATE deals SET stage = ? WHERE salesforce_id = ? OR hubspot_id = ?',
          [evt.propertyValue, evt.objectId, evt.objectId]
        );
      }
    }
  }
}

module.exports = new HubSpotService();
```

## 4. Salesforce Integration

```javascript
// src/services/salesforceService.js
const jsforce = require('jsforce');

class SalesforceService {
  constructor() {
    this.conn = new jsforce.Connection({
      loginUrl: process.env.SALESFORCE_LOGIN_URL || 'https://login.salesforce.com'
    });
    this.isConnected = false;
  }

  async connect() {
    if (this.isConnected) return;
    await this.conn.login(
      process.env.SALESFORCE_USERNAME,
      process.env.SALESFORCE_PASSWORD + process.env.SALESFORCE_TOKEN
    );
    this.isConnected = true;
  }

  // สร้าง Lead ใน Salesforce
  async createLead(contactData) {
    await this.connect();

    const lead = await this.conn.sobject('Lead').create({
      FirstName: contactData.first_name,
      LastName: contactData.last_name || 'Unknown',
      Email: contactData.email,
      Phone: contactData.phone,
      Company: contactData.company || 'N/A',
      Title: contactData.job_title,
      LeadSource: 'LINE',
      Description: `LINE User ID: ${contactData.line_user_id}`
    });

    return lead;
  }

  // แปลง Lead เป็น Contact + Account + Opportunity
  async convertLead(leadId, convertData) {
    await this.connect();

    const result = await this.conn.apex.post('/lead/convert', {
      leadId,
      doCreateOpportunity: convertData.createOpportunity,
      opportunityName: convertData.opportunityName,
      convertedStatus: 'Converted'
    });

    return result;
  }

  // สร้าง Opportunity
  async createOpportunity(contactId, data) {
    await this.connect();

    const opp = await this.conn.sobject('Opportunity').create({
      Name: data.title,
      AccountId: data.accountId,
      Amount: data.value,
      CloseDate: data.expected_close_date,
      StageName: this.mapStage(data.stage),
      Description: data.notes
    });

    return opp;
  }

  mapStage(stage) {
    const mapping = {
      prospecting: 'Prospecting',
      qualification: 'Qualification',
      proposal: 'Proposal/Price Quote',
      negotiation: 'Negotiation/Review',
      closed_won: 'Closed Won',
      closed_lost: 'Closed Lost'
    };
    return mapping[stage] || 'Prospecting';
  }

  // Query SOQL
  async query(soql) {
    await this.connect();
    const result = await this.conn.query(soql);
    return result.records;
  }

  // ดึง Contact จาก LINE User ID
  async getContactByLineId(lineUserId) {
    await this.connect();
    const records = await this.query(
      `SELECT Id, FirstName, LastName, Email, Phone FROM Contact WHERE LINE_User_ID__c = '${lineUserId}'`
    );
    return records[0] || null;
  }

  // Log Activity
  async logTask(contactId, subject, description, type = 'Other') {
    await this.connect();
    await this.conn.sobject('Task').create({
      WhoId: contactId,
      Subject: subject,
      Description: description,
      Type: type,
      Status: 'Completed',
      ActivityDate: new Date().toISOString().split('T')[0]
    });
  }
}

module.exports = new SalesforceService();
```

## 5. Funnel Analysis Service

```javascript
// src/services/funnelService.js
const db = require('../config/database');

class FunnelService {
  // คำนวณ Funnel Stats
  async getFunnelStats(startDate, endDate) {
    const params = [startDate, endDate];

    const [[stats]] = await db.query(
      `SELECT
        COUNT(*) as total_contacts,
        SUM(lifecycle_stage = 'subscriber') as subscribers,
        SUM(lifecycle_stage = 'lead') as leads,
        SUM(lifecycle_stage = 'mql') as mqls,
        SUM(lifecycle_stage = 'sql') as sqls,
        SUM(lifecycle_stage = 'opportunity') as opportunities,
        SUM(lifecycle_stage = 'customer') as customers,
        AVG(lead_score) as avg_lead_score
       FROM contacts
       WHERE created_at BETWEEN ? AND ?`,
      params
    );

    // Conversion rates
    const conversionRates = {
      subscriber_to_lead: stats.leads > 0 ? (stats.leads / stats.subscribers * 100).toFixed(1) : 0,
      lead_to_mql: stats.mqls > 0 ? (stats.mqls / stats.leads * 100).toFixed(1) : 0,
      mql_to_sql: stats.sqls > 0 ? (stats.sqls / stats.mqls * 100).toFixed(1) : 0,
      sql_to_opportunity: stats.opportunities > 0 ? (stats.opportunities / stats.sqls * 100).toFixed(1) : 0,
      opportunity_to_customer: stats.customers > 0 ? (stats.customers / stats.opportunities * 100).toFixed(1) : 0
    };

    return { ...stats, conversionRates };
  }

  // วิเคราะห์ customer journey
  async getJourneyAnalysis(days = 30) {
    const [topEvents] = await db.query(
      `SELECT event_type, COUNT(*) as count, COUNT(DISTINCT contact_id) as unique_contacts
       FROM journey_events
       WHERE created_at >= DATE_SUB(NOW(), INTERVAL ? DAY)
       GROUP BY event_type
       ORDER BY count DESC
       LIMIT 20`,
      [days]
    );

    // Average time to convert
    const [[conversionTime]] = await db.query(
      `SELECT AVG(TIMESTAMPDIFF(HOUR, c.created_at, MAX(je.created_at))) as avg_hours_to_convert
       FROM contacts c
       JOIN journey_events je ON c.id = je.contact_id AND je.event_type = 'purchase'
       WHERE c.lifecycle_stage = 'customer'
       GROUP BY c.id`
    );

    return { topEvents, avgHoursToConvert: conversionTime?.avg_hours_to_convert || 0 };
  }

  // Cohort Analysis
  async getCohortData(year, month) {
    const [cohort] = await db.query(
      `SELECT
        DATE_FORMAT(created_at, '%Y-%m') as cohort_month,
        COUNT(*) as total_users,
        SUM(lifecycle_stage = 'customer') as converted_customers,
        ROUND(SUM(lifecycle_stage = 'customer') / COUNT(*) * 100, 1) as conversion_rate
       FROM contacts
       WHERE YEAR(created_at) = ? AND MONTH(created_at) = ?
       GROUP BY cohort_month`,
      [year, month]
    );

    return cohort;
  }

  // Top Lead Sources
  async getLeadSources(days = 30) {
    const [sources] = await db.query(
      `SELECT lead_source, COUNT(*) as count,
              AVG(lead_score) as avg_score,
              SUM(lifecycle_stage = 'customer') as customers
       FROM contacts
       WHERE created_at >= DATE_SUB(NOW(), INTERVAL ? DAY)
       GROUP BY lead_source
       ORDER BY count DESC`,
      [days]
    );

    return sources;
  }

  // Sales Pipeline Value
  async getPipelineValue() {
    const [pipeline] = await db.query(
      `SELECT stage,
              COUNT(*) as count,
              SUM(value) as total_value,
              AVG(probability) as avg_probability,
              SUM(value * probability / 100) as weighted_value
       FROM deals
       WHERE stage NOT IN ('closed_won', 'closed_lost')
       GROUP BY stage
       ORDER BY FIELD(stage, 'prospecting','qualification','proposal','negotiation')`
    );

    return pipeline;
  }

  // Follow-up Due Tasks
  async getDueTasks(salesRepId = null) {
    let where = "status IN ('pending', 'in_progress') AND due_date <= DATE_ADD(NOW(), INTERVAL 1 DAY)";
    const params = [];

    if (salesRepId) {
      where += ' AND assigned_to = ?';
      params.push(salesRepId);
    }

    const [tasks] = await db.query(
      `SELECT t.*, c.first_name, c.last_name, c.line_user_id, c.email, c.phone
       FROM tasks t
       JOIN contacts c ON t.contact_id = c.id
       WHERE ${where}
       ORDER BY t.due_date ASC
       LIMIT 50`,
      params
    );

    return tasks;
  }
}

module.exports = new FunnelService();
```

## 6. Follow-up Automation

```javascript
// src/services/followUpService.js
const db = require('../config/database');
const line = require('@line/bot-sdk');
const nodemailer = require('nodemailer');

const lineClient = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN
});

class FollowUpService {
  // ส่ง follow-up LINE message
  async sendLineFollowUp(contactId, templateId, variables = {}) {
    const [[contact]] = await db.query('SELECT * FROM contacts WHERE id = ?', [contactId]);
    if (!contact?.line_user_id || contact.do_not_contact) return;

    const [[template]] = await db.query(
      'SELECT * FROM line_templates WHERE id = ? AND is_active = 1',
      [templateId]
    );
    if (!template) return;

    let content = JSON.parse(JSON.stringify(template.content));

    // แทนที่ variables ใน template
    const contentStr = JSON.stringify(content);
    let processedContent = contentStr;
    Object.entries(variables).forEach(([k, v]) => {
      processedContent = processedContent.replace(new RegExp(`{{${k}}}`, 'g'), v);
    });
    content = JSON.parse(processedContent);

    await lineClient.pushMessage(contact.line_user_id, content);

    // บันทึก activity
    await db.query(
      `INSERT INTO activities (contact_id, type, direction, subject, body, completed_at)
       VALUES (?, 'line_message', 'outbound', 'Follow-up LINE message', ?, NOW())`,
      [contactId, template.name]
    );
  }

  // เรียก follow-up แบบ batch
  async runScheduledFollowUps() {
    const now = new Date();

    // ดึง tasks ที่ถึงกำหนด
    const [tasks] = await db.query(
      `SELECT t.*, c.line_user_id, c.email, c.first_name
       FROM tasks t
       JOIN contacts c ON t.contact_id = c.id
       WHERE t.status = 'pending'
         AND t.due_date <= ?
         AND c.do_not_contact = 0`,
      [now]
    );

    console.log(`Found ${tasks.length} follow-up tasks due`);

    for (const task of tasks) {
      try {
        if (task.type === 'line_message' && task.line_user_id) {
          await lineClient.pushMessage(task.line_user_id, {
            type: 'text',
            text: task.description || task.title
          });
        } else if (task.type === 'email' && task.email) {
          await this.sendFollowUpEmail(task.email, task.title, task.description);
        }

        await db.query(
          "UPDATE tasks SET status = 'completed', completed_at = NOW() WHERE id = ?",
          [task.id]
        );

        await db.query(
          `INSERT INTO activities (contact_id, type, direction, subject, completed_at)
           VALUES (?, ?, 'outbound', ?, NOW())`,
          [task.contact_id, task.type, task.title]
        );
      } catch (err) {
        console.error(`Follow-up task ${task.id} error:`, err.message);
        await db.query(
          "UPDATE tasks SET status = 'overdue' WHERE id = ?",
          [task.id]
        );
      }
    }

    return tasks.length;
  }

  async sendFollowUpEmail(to, subject, body) {
    const transporter = nodemailer.createTransporter({
      host: process.env.SMTP_HOST,
      port: parseInt(process.env.SMTP_PORT) || 587,
      secure: false,
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASS
      }
    });

    await transporter.sendMail({
      from: `"${process.env.COMPANY_NAME}" <${process.env.SMTP_FROM}>`,
      to,
      subject,
      text: body,
      html: `<div style="font-family: Arial, sans-serif;">${body.replace(/\n/g, '<br>')}</div>`
    });
  }

  // สร้าง follow-up tasks อัตโนมัติหลังซื้อสินค้า
  async createPostPurchaseTasks(contactId, orderId) {
    const followUpSchedule = [
      { days: 1, title: 'ตรวจสอบความพึงพอใจ', type: 'line_message' },
      { days: 7, title: 'ส่ง upsell offer', type: 'line_message' },
      { days: 30, title: 'ขอรีวิว', type: 'line_message' },
      { days: 90, title: 'ส่ง loyalty reward', type: 'line_message' }
    ];

    for (const schedule of followUpSchedule) {
      const dueDate = new Date();
      dueDate.setDate(dueDate.getDate() + schedule.days);

      await db.query(
        `INSERT INTO tasks (contact_id, type, title, description, due_date, priority)
         VALUES (?, ?, ?, ?, ?, 'medium')`,
        [contactId, schedule.type, schedule.title,
         `Follow up for order ${orderId}`, dueDate]
      );
    }
  }
}

module.exports = new FollowUpService();
```

## 7. CRM Webhook Handler (LINE Events)

```javascript
// src/handlers/crmHandler.js
const contactService = require('../services/contactService');
const funnelService = require('../services/funnelService');
const followUpService = require('../services/followUpService');

async function handleCRMEvent(event, client) {
  const userId = event.source.userId;

  // Upsert contact
  let profile = null;
  try {
    profile = await client.getProfile(userId);
  } catch (err) {
    console.error('Get profile error:', err.message);
  }

  const contact = await contactService.upsertFromLine(userId, profile);

  // Track events
  if (event.type === 'follow') {
    await contactService.trackEvent(contact.id, 'follow', 'Followed LINE OA', {
      source: 'line'
    });
  } else if (event.type === 'unfollow') {
    await contactService.trackEvent(contact.id, 'unfollow', 'Unfollowed LINE OA');
    await contactService.updateContact(contact.id, { lifecycle_stage: 'subscriber' });
  } else if (event.type === 'message') {
    await contactService.trackEvent(contact.id, 'message', 'Sent message', {
      message_type: event.message.type,
      text: event.message.type === 'text' ? event.message.text?.slice(0, 100) : ''
    });

    await contactService.logActivity(
      contact.id,
      'line_message',
      'inbound',
      'LINE Message',
      event.message.type === 'text' ? event.message.text : `[${event.message.type}]`,
      { event_id: event.webhookEventId }
    );
  } else if (event.type === 'postback') {
    const params = new URLSearchParams(event.postback.data);
    const action = params.get('action');

    await contactService.trackEvent(contact.id, 'postback', action, {
      postback_data: event.postback.data
    });

    // Track specific events
    if (action === 'view_product') {
      await contactService.trackEvent(contact.id, 'view_product', 'Viewed product', {
        product_id: params.get('product_id')
      });
    } else if (action === 'add_cart') {
      await contactService.trackEvent(contact.id, 'add_to_cart', 'Added to cart', {
        product_id: params.get('product_id')
      });
    }
  }

  return contact;
}

// Track purchase event
async function trackPurchase(userId, orderId, amount) {
  const [[contact]] = await require('../config/database').query(
    'SELECT * FROM contacts WHERE line_user_id = ?', [userId]
  );
  if (!contact) return;

  await contactService.trackEvent(contact.id, 'purchase', 'Made purchase', {
    order_id: orderId,
    amount
  });

  // อัปเดต lifecycle stage
  await contactService.updateContact(contact.id, { lifecycle_stage: 'customer' });

  // สร้าง post-purchase follow-up tasks
  await followUpService.createPostPurchaseTasks(contact.id, orderId);
}

module.exports = { handleCRMEvent, trackPurchase };
```

## 8. Admin CRM API

```javascript
// src/routes/admin.js
const express = require('express');
const db = require('../config/database');
const contactService = require('../services/contactService');
const funnelService = require('../services/funnelService');
const followUpService = require('../services/followUpService');

const router = express.Router();

function adminAuth(req, res, next) {
  if (req.headers['x-admin-token'] !== process.env.ADMIN_TOKEN) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  next();
}

router.use(adminAuth);

// Funnel Overview
router.get('/funnel', async (req, res) => {
  const { start = '2024-01-01', end = new Date().toISOString().split('T')[0] } = req.query;
  const stats = await funnelService.getFunnelStats(start, end);
  res.json(stats);
});

// Contacts
router.get('/contacts', async (req, res) => {
  const { lifecycle_stage, min_score, search, page = 1, limit = 20 } = req.query;
  const result = await contactService.getContacts({ lifecycle_stage, min_score: parseInt(min_score), search }, parseInt(page), parseInt(limit));
  res.json(result);
});

router.get('/contacts/:id', async (req, res) => {
  const [[contact]] = await db.query('SELECT * FROM contacts WHERE id = ?', [req.params.id]);
  if (!contact) return res.status(404).json({ error: 'Not found' });

  const [activities] = await db.query(
    'SELECT * FROM activities WHERE contact_id = ? ORDER BY created_at DESC LIMIT 20',
    [req.params.id]
  );

  const [journeyEvents] = await db.query(
    'SELECT * FROM journey_events WHERE contact_id = ? ORDER BY created_at DESC LIMIT 50',
    [req.params.id]
  );

  const [deals] = await db.query(
    'SELECT * FROM deals WHERE contact_id = ? ORDER BY created_at DESC',
    [req.params.id]
  );

  const [tasks] = await db.query(
    "SELECT * FROM tasks WHERE contact_id = ? AND status != 'completed' ORDER BY due_date",
    [req.params.id]
  );

  res.json({ contact, activities, journeyEvents, deals, tasks });
});

router.put('/contacts/:id', async (req, res) => {
  await contactService.updateContact(parseInt(req.params.id), req.body);
  res.json({ ok: true });
});

// Deals
router.get('/deals', async (req, res) => {
  const [deals] = await db.query(
    `SELECT d.*, c.first_name, c.last_name, c.email, c.line_user_id
     FROM deals d
     JOIN contacts c ON d.contact_id = c.id
     ORDER BY d.created_at DESC LIMIT 50`
  );
  res.json({ deals });
});

router.post('/deals', async (req, res) => {
  const { contact_id, title, value, stage, expected_close_date, notes } = req.body;
  const [result] = await db.query(
    `INSERT INTO deals (contact_id, title, value, stage, expected_close_date, notes)
     VALUES (?, ?, ?, ?, ?, ?)`,
    [contact_id, title, value, stage || 'prospecting', expected_close_date, notes]
  );
  res.json({ id: result.insertId });
});

router.put('/deals/:id', async (req, res) => {
  const { stage, value, notes, probability } = req.body;
  await db.query(
    'UPDATE deals SET stage = COALESCE(?, stage), value = COALESCE(?, value), notes = COALESCE(?, notes), probability = COALESCE(?, probability) WHERE id = ?',
    [stage, value, notes, probability, req.params.id]
  );
  res.json({ ok: true });
});

// Pipeline
router.get('/pipeline', async (req, res) => {
  const pipeline = await funnelService.getPipelineValue();
  res.json({ pipeline });
});

// Tasks
router.get('/tasks', async (req, res) => {
  const { rep_id } = req.query;
  const tasks = await funnelService.getDueTasks(rep_id);
  res.json({ tasks });
});

router.post('/tasks', async (req, res) => {
  const { contact_id, type, title, description, due_date, priority, assigned_to } = req.body;
  const [result] = await db.query(
    `INSERT INTO tasks (contact_id, type, title, description, due_date, priority, assigned_to)
     VALUES (?, ?, ?, ?, ?, ?, ?)`,
    [contact_id, type, title, description, due_date, priority || 'medium', assigned_to]
  );
  res.json({ id: result.insertId });
});

// Send LINE message to contact
router.post('/contacts/:id/send-line', async (req, res) => {
  const [[contact]] = await db.query('SELECT * FROM contacts WHERE id = ?', [req.params.id]);
  if (!contact?.line_user_id) return res.status(400).json({ error: 'No LINE user ID' });

  const { message } = req.body;

  const line = require('@line/bot-sdk');
  const client = new line.Client({ channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN });
  await client.pushMessage(contact.line_user_id, { type: 'text', text: message });

  await contactService.logActivity(contact.id, 'line_message', 'outbound', 'Manual LINE message', message);
  res.json({ ok: true });
});

// Journey Analysis
router.get('/journey', async (req, res) => {
  const { days = 30 } = req.query;
  const analysis = await funnelService.getJourneyAnalysis(parseInt(days));
  res.json(analysis);
});

// Lead Sources
router.get('/lead-sources', async (req, res) => {
  const { days = 30 } = req.query;
  const sources = await funnelService.getLeadSources(parseInt(days));
  res.json({ sources });
});

// Workflows
router.get('/workflows', async (req, res) => {
  const [workflows] = await db.query('SELECT * FROM workflows ORDER BY created_at DESC');
  res.json({ workflows });
});

router.post('/workflows', async (req, res) => {
  const { name, trigger_type, trigger_conditions, actions } = req.body;
  const [result] = await db.query(
    'INSERT INTO workflows (name, trigger_type, trigger_conditions, actions) VALUES (?, ?, ?, ?)',
    [name, trigger_type, JSON.stringify(trigger_conditions || {}), JSON.stringify(actions || [])]
  );
  res.json({ id: result.insertId });
});

// Run follow-up scheduler
router.post('/run-followups', async (req, res) => {
  const count = await followUpService.runScheduledFollowUps();
  res.json({ processed: count });
});

module.exports = router;
```

## 9. index.js

```javascript
// index.js
require('dotenv').config();
const express = require('express');
const line = require('@line/bot-sdk');
const { handleCRMEvent } = require('./src/handlers/crmHandler');
const cron = require('node-cron');
const followUpService = require('./src/services/followUpService');

const app = express();
const lineConfig = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};
const client = new line.Client(lineConfig);

app.use('/webhook', line.middleware(lineConfig), async (req, res) => {
  await Promise.all(req.body.events.map(async event => {
    try {
      await handleCRMEvent(event, client);
    } catch (err) {
      console.error('CRM event error:', err.message);
    }
  }));
  res.json({ ok: true });
});

app.use(express.json());
app.use('/admin', require('./src/routes/admin'));

// Cron: รันทุก 30 นาที
cron.schedule('*/30 * * * *', async () => {
  try {
    await followUpService.runScheduledFollowUps();
  } catch (err) {
    console.error('Follow-up cron error:', err.message);
  }
});

const PORT = process.env.PORT || 3000;
app.listen(PORT, () => console.log(`CRM Bot on port ${PORT}`));
```

## 10. package.json

```json
{
  "name": "lineoa-crm",
  "version": "1.0.0",
  "dependencies": {
    "@line/bot-sdk": "^8.0.0",
    "axios": "^1.6.0",
    "dotenv": "^16.0.0",
    "express": "^4.18.0",
    "jsforce": "^1.11.0",
    "mysql2": "^3.6.0",
    "node-cron": "^3.0.0",
    "nodemailer": "^6.9.0"
  }
}
```

## สรุป

CRM Integration สำหรับ LINE OA ที่สร้างในบทนี้:

1. **Custom CRM** - Database schema ครบถ้วน contacts/deals/activities
2. **Lead Scoring** - คะแนนอัตโนมัติจาก LINE interactions
3. **Journey Tracking** - ติดตาม customer journey ทุก touchpoint
4. **HubSpot Integration** - Sync contacts และ deals
5. **Salesforce Integration** - Create leads และ opportunities
6. **Funnel Analysis** - วิเคราะห์ conversion rates
7. **Workflow Automation** - Trigger actions จาก events
8. **Follow-up System** - Tasks และ scheduled messages
9. **Multi-channel** - LINE + Email
10. **Admin API** - จัดการ CRM จาก dashboard
