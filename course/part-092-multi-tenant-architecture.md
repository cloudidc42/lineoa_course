# Part 92: Multi-Tenant LINE Bot Platform

## บทนำ

Multi-tenant architecture คือรูปแบบการออกแบบที่ให้ instance เดียวของ application รองรับ "ผู้เช่า" (tenant) หลายรายได้พร้อมกัน โดยแต่ละ tenant จะมีข้อมูลและการตั้งค่าที่แยกออกจากกัน บทนี้จะครอบคลุมการสร้าง multi-tenant LINE Bot platform ที่สมบูรณ์

---

## 1. สถาปัตยกรรม Multi-Tenant Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     MULTI-TENANT LINE BOT PLATFORM                       │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                        Admin Portal                               │    │
│  │  Tenant A Dashboard  │  Tenant B Dashboard  │  Tenant C Dashboard │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                    │                                      │
│  ┌─────────────────────────────────▼─────────────────────────────────┐  │
│  │                    Tenant Router / Resolver                         │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │  │
│  │  │Subdomain      │  │ Custom Domain│  │ LINE Channel ID Mapping  │ │  │
│  │  │tenantA.bot.me │  │ api.tenantA  │  │ channel_xxx → tenantA   │ │  │
│  │  └──────────────┘  └──────────────┘  └──────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                    │                                      │
│  ┌─────────────────────────────────▼─────────────────────────────────┐  │
│  │                    Shared Application Layer                         │  │
│  │  ┌────────────┐  ┌────────────┐  ┌──────────┐  ┌───────────────┐ │  │
│  │  │ Message    │  │ User Mgmt  │  │ Webhook  │  │ Analytics     │ │  │
│  │  │ Handler    │  │ Service    │  │ Service  │  │ Service       │ │  │
│  │  └────────────┘  └────────────┘  └──────────┘  └───────────────┘ │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                    │                                      │
│  ┌─────────────────────────────────▼─────────────────────────────────┐  │
│  │                      Data Isolation Layer                           │  │
│  │                                                                     │  │
│  │  Strategy 1: Shared DB    Strategy 2: Schema/DB   Strategy 3: DB  │  │
│  │  ┌─────────────────┐      ┌────────────────┐      ┌────────────┐  │  │
│  │  │ shared_db        │      │ tenant_a_schema│      │ tenant_a   │  │  │
│  │  │  tenant_id FK    │      │ tenant_b_schema│      │ tenant_b   │  │  │
│  │  └─────────────────┘      └────────────────┘      └────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Tenant Isolation Strategies

### 2.1 Strategy 1: Shared Database with Tenant ID

```typescript
// models/TenantAwareModel.ts
import { Pool } from 'pg';
import { AsyncLocalStorage } from 'async_hooks';

// Context สำหรับเก็บ tenant ปัจจุบัน
export const tenantContext = new AsyncLocalStorage<string>();

export class TenantAwareRepository<T> {
  constructor(
    private pool: Pool,
    private tableName: string,
    private tenantColumn: string = 'tenant_id'
  ) {}

  private getCurrentTenantId(): string {
    const tenantId = tenantContext.getStore();
    if (!tenantId) {
      throw new Error('No tenant context found. Use TenantContext.run() first.');
    }
    return tenantId;
  }

  async findAll(where: Partial<T> = {}): Promise<T[]> {
    const tenantId = this.getCurrentTenantId();
    const conditions: string[] = [`${this.tenantColumn} = $1`];
    const values: any[] = [tenantId];
    let paramIndex = 2;

    for (const [key, value] of Object.entries(where)) {
      conditions.push(`${key} = $${paramIndex}`);
      values.push(value);
      paramIndex++;
    }

    const query = `
      SELECT * FROM ${this.tableName}
      WHERE ${conditions.join(' AND ')}
    `;

    const result = await this.pool.query(query, values);
    return result.rows as T[];
  }

  async findById(id: string): Promise<T | null> {
    const tenantId = this.getCurrentTenantId();
    const result = await this.pool.query(
      `SELECT * FROM ${this.tableName} WHERE id = $1 AND ${this.tenantColumn} = $2`,
      [id, tenantId]
    );
    return (result.rows[0] as T) || null;
  }

  async create(data: Partial<T>): Promise<T> {
    const tenantId = this.getCurrentTenantId();
    const dataWithTenant = { ...data, [this.tenantColumn]: tenantId };
    
    const keys = Object.keys(dataWithTenant);
    const values = Object.values(dataWithTenant);
    const placeholders = keys.map((_, i) => `$${i + 1}`);

    const result = await this.pool.query(
      `INSERT INTO ${this.tableName} (${keys.join(', ')}) VALUES (${placeholders.join(', ')}) RETURNING *`,
      values
    );

    return result.rows[0] as T;
  }

  async update(id: string, data: Partial<T>): Promise<T | null> {
    const tenantId = this.getCurrentTenantId();
    const updates = Object.keys(data).map((key, i) => `${key} = $${i + 3}`);
    
    const result = await this.pool.query(
      `UPDATE ${this.tableName} 
       SET ${updates.join(', ')}, updated_at = NOW()
       WHERE id = $1 AND ${this.tenantColumn} = $2
       RETURNING *`,
      [id, tenantId, ...Object.values(data)]
    );

    return (result.rows[0] as T) || null;
  }

  async delete(id: string): Promise<boolean> {
    const tenantId = this.getCurrentTenantId();
    const result = await this.pool.query(
      `DELETE FROM ${this.tableName} WHERE id = $1 AND ${this.tenantColumn} = $2`,
      [id, tenantId]
    );
    return (result.rowCount || 0) > 0;
  }

  async count(where: Partial<T> = {}): Promise<number> {
    const tenantId = this.getCurrentTenantId();
    const conditions: string[] = [`${this.tenantColumn} = $1`];
    const values: any[] = [tenantId];
    let paramIndex = 2;

    for (const [key, value] of Object.entries(where)) {
      conditions.push(`${key} = $${paramIndex}`);
      values.push(value);
      paramIndex++;
    }

    const result = await this.pool.query(
      `SELECT COUNT(*) FROM ${this.tableName} WHERE ${conditions.join(' AND ')}`,
      values
    );

    return parseInt(result.rows[0].count);
  }
}

// Row Level Security Policy สำหรับ PostgreSQL
export const TENANT_RLS_SETUP = `
-- เปิดใช้งาน RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;
ALTER TABLE messages ENABLE ROW LEVEL SECURITY;
ALTER TABLE conversations ENABLE ROW LEVEL SECURITY;

-- Policy สำหรับ application user
CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

CREATE POLICY tenant_isolation ON messages
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

CREATE POLICY tenant_isolation ON conversations
    USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Function สำหรับ set tenant context
CREATE OR REPLACE FUNCTION set_tenant_context(p_tenant_id uuid)
RETURNS void AS $$
BEGIN
    PERFORM set_config('app.tenant_id', p_tenant_id::text, true);
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
`;
```

### 2.2 Strategy 2: Schema Per Tenant

```typescript
// database/SchemaManager.ts
import { Pool, PoolClient } from 'pg';

export class SchemaManager {
  private pool: Pool;

  constructor(pool: Pool) {
    this.pool = pool;
  }

  async createTenantSchema(tenantId: string): Promise<void> {
    const schemaName = this.sanitizeSchemaName(tenantId);
    
    const client = await this.pool.connect();
    try {
      await client.query('BEGIN');

      // สร้าง Schema
      await client.query(`CREATE SCHEMA IF NOT EXISTS ${schemaName}`);

      // สร้าง Tables ใน Schema
      await this.createTenantTables(client, schemaName);

      // สร้าง Indexes
      await this.createTenantIndexes(client, schemaName);

      // Grant permissions
      await client.query(`
        GRANT USAGE ON SCHEMA ${schemaName} TO app_user;
        GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA ${schemaName} TO app_user;
        GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA ${schemaName} TO app_user;
      `);

      await client.query('COMMIT');
    } catch (error) {
      await client.query('ROLLBACK');
      throw error;
    } finally {
      client.release();
    }
  }

  private async createTenantTables(client: PoolClient, schema: string): Promise<void> {
    await client.query(`
      -- Users table
      CREATE TABLE IF NOT EXISTS ${schema}.users (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        line_user_id VARCHAR(100) UNIQUE NOT NULL,
        display_name VARCHAR(255),
        picture_url TEXT,
        language VARCHAR(10) DEFAULT 'th',
        metadata JSONB DEFAULT '{}',
        created_at TIMESTAMPTZ DEFAULT NOW(),
        updated_at TIMESTAMPTZ DEFAULT NOW(),
        last_seen_at TIMESTAMPTZ
      );

      -- Conversations table
      CREATE TABLE IF NOT EXISTS ${schema}.conversations (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        user_id UUID REFERENCES ${schema}.users(id),
        context JSONB DEFAULT '{}',
        state VARCHAR(50) DEFAULT 'active',
        started_at TIMESTAMPTZ DEFAULT NOW(),
        ended_at TIMESTAMPTZ,
        metadata JSONB DEFAULT '{}'
      );

      -- Messages table
      CREATE TABLE IF NOT EXISTS ${schema}.messages (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        conversation_id UUID REFERENCES ${schema}.conversations(id),
        user_id UUID REFERENCES ${schema}.users(id),
        direction VARCHAR(10) CHECK (direction IN ('inbound', 'outbound')),
        message_type VARCHAR(50) NOT NULL,
        content JSONB NOT NULL,
        line_message_id VARCHAR(100),
        sent_at TIMESTAMPTZ DEFAULT NOW(),
        delivered_at TIMESTAMPTZ,
        read_at TIMESTAMPTZ,
        metadata JSONB DEFAULT '{}'
      );

      -- Rich Menus
      CREATE TABLE IF NOT EXISTS ${schema}.rich_menus (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        line_rich_menu_id VARCHAR(100),
        name VARCHAR(255) NOT NULL,
        config JSONB NOT NULL,
        is_default BOOLEAN DEFAULT false,
        target_audience JSONB DEFAULT '{}',
        created_at TIMESTAMPTZ DEFAULT NOW(),
        updated_at TIMESTAMPTZ DEFAULT NOW()
      );

      -- Bot Configurations
      CREATE TABLE IF NOT EXISTS ${schema}.bot_configs (
        key VARCHAR(255) PRIMARY KEY,
        value JSONB NOT NULL,
        description TEXT,
        updated_at TIMESTAMPTZ DEFAULT NOW()
      );

      -- Broadcasts
      CREATE TABLE IF NOT EXISTS ${schema}.broadcasts (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        name VARCHAR(255) NOT NULL,
        message_content JSONB NOT NULL,
        audience_filter JSONB DEFAULT '{}',
        scheduled_at TIMESTAMPTZ,
        sent_at TIMESTAMPTZ,
        status VARCHAR(50) DEFAULT 'draft',
        statistics JSONB DEFAULT '{}',
        created_at TIMESTAMPTZ DEFAULT NOW()
      );

      -- Analytics Events
      CREATE TABLE IF NOT EXISTS ${schema}.analytics_events (
        id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
        user_id UUID REFERENCES ${schema}.users(id),
        event_type VARCHAR(100) NOT NULL,
        event_data JSONB DEFAULT '{}',
        session_id VARCHAR(100),
        timestamp TIMESTAMPTZ DEFAULT NOW()
      ) PARTITION BY RANGE (timestamp);

      -- Partitions for analytics (by month)
      CREATE TABLE IF NOT EXISTS ${schema}.analytics_events_2024_01 
        PARTITION OF ${schema}.analytics_events
        FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
    `);
  }

  private async createTenantIndexes(client: PoolClient, schema: string): Promise<void> {
    await client.query(`
      CREATE INDEX IF NOT EXISTS idx_${schema}_users_line_user_id 
        ON ${schema}.users(line_user_id);
      
      CREATE INDEX IF NOT EXISTS idx_${schema}_messages_conversation 
        ON ${schema}.messages(conversation_id, sent_at DESC);
      
      CREATE INDEX IF NOT EXISTS idx_${schema}_analytics_user_time 
        ON ${schema}.analytics_events(user_id, timestamp DESC);
      
      CREATE INDEX IF NOT EXISTS idx_${schema}_analytics_event_type 
        ON ${schema}.analytics_events(event_type, timestamp DESC);
    `);
  }

  async deleteTenantSchema(tenantId: string, backup: boolean = true): Promise<void> {
    const schemaName = this.sanitizeSchemaName(tenantId);
    
    if (backup) {
      await this.backupTenantSchema(schemaName);
    }

    await this.pool.query(`DROP SCHEMA IF EXISTS ${schemaName} CASCADE`);
  }

  private async backupTenantSchema(schemaName: string): Promise<void> {
    const backupSchema = `${schemaName}_backup_${Date.now()}`;
    // การ backup schema โดยการ rename
    await this.pool.query(`ALTER SCHEMA ${schemaName} RENAME TO ${backupSchema}`);
  }

  private sanitizeSchemaName(tenantId: string): string {
    // ทำให้ schema name ปลอดภัย
    return 'tenant_' + tenantId.replace(/[^a-z0-9_]/gi, '_').toLowerCase();
  }

  async getTenantConnection(tenantId: string): Promise<TenantConnection> {
    const schemaName = this.sanitizeSchemaName(tenantId);
    return new TenantConnection(this.pool, schemaName);
  }
}

class TenantConnection {
  constructor(
    private pool: Pool,
    private schema: string
  ) {}

  async query(sql: string, values?: any[]): Promise<any> {
    const client = await this.pool.connect();
    try {
      await client.query(`SET search_path TO ${this.schema}, public`);
      return await client.query(sql, values);
    } finally {
      client.release();
    }
  }
}
```

### 2.3 Strategy 3: Database Per Tenant

```typescript
// database/TenantDatabaseManager.ts
import { createPool, Pool } from 'slonik';
import { createConnection } from 'mysql2/promise';

interface TenantDatabase {
  tenantId: string;
  connectionString: string;
  pool: Pool;
  createdAt: Date;
}

export class TenantDatabaseManager {
  private tenantDatabases: Map<string, TenantDatabase> = new Map();
  private masterPool: Pool;

  constructor(masterConnectionString: string) {
    this.masterPool = createPool(masterConnectionString);
  }

  async provisionTenantDatabase(tenantId: string): Promise<void> {
    const dbName = `linebot_tenant_${tenantId.replace(/-/g, '_')}`;
    const dbUser = `tenant_${tenantId.replace(/-/g, '_')}`;
    const dbPassword = this.generateSecurePassword();

    // สร้าง Database
    await this.masterPool.query(`
      CREATE DATABASE ${dbName}
      WITH ENCODING 'UTF8'
      LC_COLLATE = 'th_TH.UTF-8'
      LC_CTYPE = 'th_TH.UTF-8'
      TEMPLATE template0
    `);

    // สร้าง User สำหรับ tenant
    await this.masterPool.query(`
      CREATE USER ${dbUser} WITH PASSWORD '${dbPassword}';
      GRANT ALL PRIVILEGES ON DATABASE ${dbName} TO ${dbUser};
    `);

    // เชื่อมต่อกับ tenant database และสร้าง tables
    const tenantConnectionString = `postgresql://${dbUser}:${dbPassword}@${process.env.DB_HOST}/${dbName}`;
    const tenantPool = createPool(tenantConnectionString);
    
    await this.runMigrations(tenantPool);

    // เก็บ connection info
    this.tenantDatabases.set(tenantId, {
      tenantId,
      connectionString: tenantConnectionString,
      pool: tenantPool,
      createdAt: new Date()
    });

    // เก็บ connection info ใน master database (encrypted)
    await this.saveTenantDbConfig(tenantId, {
      dbName,
      dbUser,
      connectionString: this.encryptConnectionString(tenantConnectionString)
    });
  }

  async getTenantPool(tenantId: string): Promise<Pool> {
    if (this.tenantDatabases.has(tenantId)) {
      return this.tenantDatabases.get(tenantId)!.pool;
    }

    // Load from config store
    const config = await this.loadTenantDbConfig(tenantId);
    const connectionString = this.decryptConnectionString(config.connectionString);
    const pool = createPool(connectionString);
    
    this.tenantDatabases.set(tenantId, {
      tenantId,
      connectionString,
      pool,
      createdAt: new Date()
    });

    return pool;
  }

  private async runMigrations(pool: Pool): Promise<void> {
    await pool.query(`
      CREATE TABLE IF NOT EXISTS migrations (
        id SERIAL PRIMARY KEY,
        version VARCHAR(50) UNIQUE NOT NULL,
        applied_at TIMESTAMPTZ DEFAULT NOW()
      );
    `);
    // Run all migration scripts
  }

  private generateSecurePassword(): string {
    const crypto = require('crypto');
    return crypto.randomBytes(32).toString('base64url');
  }

  private encryptConnectionString(connectionString: string): string {
    // Encrypt with KMS
    return connectionString; // Placeholder
  }

  private decryptConnectionString(encrypted: string): string {
    // Decrypt with KMS
    return encrypted; // Placeholder
  }

  private async saveTenantDbConfig(tenantId: string, config: any): Promise<void> {
    await this.masterPool.query(
      `INSERT INTO tenant_databases (tenant_id, config, created_at) VALUES ($1, $2, NOW())`,
      [tenantId, JSON.stringify(config)]
    );
  }

  private async loadTenantDbConfig(tenantId: string): Promise<any> {
    const result = await this.masterPool.query(
      `SELECT config FROM tenant_databases WHERE tenant_id = $1`,
      [tenantId]
    );
    return result.rows[0]?.config;
  }
}
```

---

## 3. Tenant Provisioning Automation

### 3.1 Tenant Provisioner Service

```typescript
// provisioning/TenantProvisioner.ts
import { EventEmitter } from 'events';
import { TenantRepository } from '../repositories/TenantRepository';
import { SchemaManager } from '../database/SchemaManager';
import { LINEChannelManager } from '../line/LINEChannelManager';
import { BillingService } from '../billing/BillingService';
import { NotificationService } from '../notifications/NotificationService';

interface ProvisioningRequest {
  companyName: string;
  adminEmail: string;
  adminName: string;
  plan: 'starter' | 'business' | 'enterprise';
  lineChannelId?: string;
  lineChannelSecret?: string;
  lineChannelAccessToken?: string;
  customDomain?: string;
  settings?: Partial<TenantSettings>;
}

interface TenantSettings {
  language: string;
  timezone: string;
  webhookUrl: string;
  richMenuEnabled: boolean;
  broadcastEnabled: boolean;
  analyticsEnabled: boolean;
  maxUsers: number;
  maxMessages: number;
  customFeatures: string[];
}

interface Tenant {
  id: string;
  companyName: string;
  adminEmail: string;
  plan: string;
  status: 'provisioning' | 'active' | 'suspended' | 'terminated';
  settings: TenantSettings;
  lineChannels: LineChannel[];
  createdAt: Date;
  metadata: Record<string, any>;
}

interface LineChannel {
  channelId: string;
  name: string;
  isActive: boolean;
  webhookUrl: string;
}

export class TenantProvisioner extends EventEmitter {
  private tenantRepo: TenantRepository;
  private schemaManager: SchemaManager;
  private lineManager: LINEChannelManager;
  private billing: BillingService;
  private notifications: NotificationService;

  constructor() {
    super();
    this.tenantRepo = new TenantRepository();
    this.schemaManager = new SchemaManager();
    this.lineManager = new LINEChannelManager();
    this.billing = new BillingService();
    this.notifications = new NotificationService();
  }

  async provisionTenant(request: ProvisioningRequest): Promise<Tenant> {
    const tenantId = this.generateTenantId(request.companyName);
    
    console.log(`Starting provisioning for tenant: ${tenantId}`);
    this.emit('provisioning:started', { tenantId, request });

    try {
      // Step 1: Create tenant record
      const tenant = await this.createTenantRecord(tenantId, request);
      this.emit('provisioning:step', { tenantId, step: 'record_created' });

      // Step 2: Setup database
      await this.setupDatabase(tenantId);
      this.emit('provisioning:step', { tenantId, step: 'database_ready' });

      // Step 3: Configure LINE Channel
      if (request.lineChannelId) {
        await this.configureLINEChannel(tenantId, request);
        this.emit('provisioning:step', { tenantId, step: 'line_configured' });
      }

      // Step 4: Setup billing
      await this.setupBilling(tenantId, request.plan, request.adminEmail);
      this.emit('provisioning:step', { tenantId, step: 'billing_setup' });

      // Step 5: Create admin account
      const adminToken = await this.createAdminAccount(tenantId, request);
      this.emit('provisioning:step', { tenantId, step: 'admin_created' });

      // Step 6: Configure custom domain
      if (request.customDomain) {
        await this.setupCustomDomain(tenantId, request.customDomain);
        this.emit('provisioning:step', { tenantId, step: 'domain_configured' });
      }

      // Step 7: Apply default configurations
      await this.applyDefaultConfigurations(tenantId, request.plan);
      this.emit('provisioning:step', { tenantId, step: 'configs_applied' });

      // Step 8: Activate tenant
      await this.tenantRepo.updateStatus(tenantId, 'active');

      // Step 9: Send welcome email
      await this.notifications.sendWelcomeEmail({
        email: request.adminEmail,
        name: request.adminName,
        tenantId,
        loginUrl: `https://${tenantId}.linebot-platform.com/admin`,
        tempPassword: adminToken
      });

      const finalTenant = await this.tenantRepo.findById(tenantId);
      this.emit('provisioning:completed', { tenantId, tenant: finalTenant });
      
      return finalTenant!;

    } catch (error) {
      this.emit('provisioning:failed', { tenantId, error });
      
      // Rollback
      await this.rollbackProvisioning(tenantId);
      throw error;
    }
  }

  private async createTenantRecord(tenantId: string, request: ProvisioningRequest): Promise<Tenant> {
    const defaultSettings = this.getDefaultSettings(request.plan);
    
    return await this.tenantRepo.create({
      id: tenantId,
      companyName: request.companyName,
      adminEmail: request.adminEmail,
      plan: request.plan,
      status: 'provisioning',
      settings: {
        ...defaultSettings,
        ...request.settings
      },
      lineChannels: [],
      createdAt: new Date(),
      metadata: {}
    });
  }

  private getDefaultSettings(plan: string): TenantSettings {
    const planSettings: Record<string, TenantSettings> = {
      starter: {
        language: 'th',
        timezone: 'Asia/Bangkok',
        webhookUrl: '',
        richMenuEnabled: true,
        broadcastEnabled: false,
        analyticsEnabled: true,
        maxUsers: 1000,
        maxMessages: 10000,
        customFeatures: []
      },
      business: {
        language: 'th',
        timezone: 'Asia/Bangkok',
        webhookUrl: '',
        richMenuEnabled: true,
        broadcastEnabled: true,
        analyticsEnabled: true,
        maxUsers: 10000,
        maxMessages: 100000,
        customFeatures: ['custom_rich_menu', 'advanced_analytics']
      },
      enterprise: {
        language: 'th',
        timezone: 'Asia/Bangkok',
        webhookUrl: '',
        richMenuEnabled: true,
        broadcastEnabled: true,
        analyticsEnabled: true,
        maxUsers: -1, // Unlimited
        maxMessages: -1, // Unlimited
        customFeatures: ['custom_rich_menu', 'advanced_analytics', 'white_label', 'api_access', 'dedicated_support']
      }
    };

    return planSettings[plan] || planSettings.starter;
  }

  private async setupDatabase(tenantId: string): Promise<void> {
    await this.schemaManager.createTenantSchema(tenantId);
    
    // เพิ่ม default data
    await this.seedDefaultData(tenantId);
  }

  private async seedDefaultData(tenantId: string): Promise<void> {
    const conn = await this.schemaManager.getTenantConnection(tenantId);
    
    // Default bot configurations
    await conn.query(`
      INSERT INTO bot_configs (key, value, description) VALUES
      ('welcome_message', '{"text": "สวัสดีครับ! ยินดีต้อนรับ"}', 'Welcome message for new users'),
      ('error_message', '{"text": "ขออภัย ไม่สามารถดำเนินการได้ในขณะนี้"}', 'Error fallback message'),
      ('quick_replies', '[]', 'Default quick reply options'),
      ('business_hours', '{"enabled": false, "timezone": "Asia/Bangkok"}', 'Business hours settings')
      ON CONFLICT (key) DO NOTHING
    `);
  }

  private async configureLINEChannel(tenantId: string, request: ProvisioningRequest): Promise<void> {
    const webhookUrl = `https://api.linebot-platform.com/webhook/${tenantId}`;
    
    await this.lineManager.configureChannel({
      tenantId,
      channelId: request.lineChannelId!,
      channelSecret: request.lineChannelSecret!,
      accessToken: request.lineChannelAccessToken!,
      webhookUrl
    });

    // Verify webhook
    await this.lineManager.verifyWebhook(request.lineChannelAccessToken!, webhookUrl);
  }

  private async setupBilling(tenantId: string, plan: string, email: string): Promise<void> {
    const planPricing = {
      starter: { amount: 990, currency: 'THB', interval: 'month' },
      business: { amount: 2990, currency: 'THB', interval: 'month' },
      enterprise: { amount: 9900, currency: 'THB', interval: 'month' }
    };

    await this.billing.createSubscription({
      tenantId,
      email,
      plan,
      pricing: planPricing[plan as keyof typeof planPricing],
      trialDays: 30
    });
  }

  private async createAdminAccount(tenantId: string, request: ProvisioningRequest): Promise<string> {
    const tempPassword = this.generateTempPassword();
    
    // Create admin user in tenant schema
    const conn = await this.schemaManager.getTenantConnection(tenantId);
    // Implementation...
    
    return tempPassword;
  }

  private async setupCustomDomain(tenantId: string, domain: string): Promise<void> {
    // Configure DNS (Cloudflare API)
    // Setup SSL certificate
    // Configure nginx/load balancer
  }

  private async applyDefaultConfigurations(tenantId: string, plan: string): Promise<void> {
    // Apply plan-specific configurations
  }

  private async rollbackProvisioning(tenantId: string): Promise<void> {
    try {
      await this.schemaManager.deleteTenantSchema(tenantId, false);
      await this.tenantRepo.delete(tenantId);
    } catch (rollbackError) {
      console.error('Rollback failed:', rollbackError);
    }
  }

  private generateTenantId(companyName: string): string {
    const slug = companyName
      .toLowerCase()
      .replace(/[^a-z0-9]/g, '-')
      .replace(/-+/g, '-')
      .substring(0, 30);
    
    const random = Math.random().toString(36).substring(2, 6);
    return `${slug}-${random}`;
  }

  private generateTempPassword(): string {
    const crypto = require('crypto');
    return crypto.randomBytes(16).toString('base64url');
  }
}
```

---

## 4. Billing Per Tenant

### 4.1 Metered Billing Service

```typescript
// billing/BillingService.ts
import Stripe from 'stripe';
import { Redis } from 'ioredis';
import { TenantRepository } from '../repositories/TenantRepository';

interface UsageRecord {
  tenantId: string;
  metric: 'messages' | 'api_calls' | 'users' | 'storage_gb';
  quantity: number;
  timestamp: Date;
}

interface BillingPlan {
  id: string;
  name: string;
  basePrice: number;
  currency: string;
  interval: 'month' | 'year';
  included: Record<string, number>;
  overage: Record<string, number>;
  features: string[];
}

export class BillingService {
  private stripe: Stripe;
  private redis: Redis;
  private tenantRepo: TenantRepository;

  private readonly PLANS: Record<string, BillingPlan> = {
    starter: {
      id: 'starter',
      name: 'Starter',
      basePrice: 990,
      currency: 'thb',
      interval: 'month',
      included: {
        messages: 10000,
        api_calls: 50000,
        users: 1000,
        storage_gb: 5
      },
      overage: {
        messages: 0.1,    // ฿0.10 ต่อ message
        api_calls: 0.001,
        users: 0.5,
        storage_gb: 10
      },
      features: ['basic_analytics', 'webhook', 'rich_menu']
    },
    business: {
      id: 'business',
      name: 'Business',
      basePrice: 2990,
      currency: 'thb',
      interval: 'month',
      included: {
        messages: 100000,
        api_calls: 500000,
        users: 10000,
        storage_gb: 50
      },
      overage: {
        messages: 0.05,
        api_calls: 0.0005,
        users: 0.3,
        storage_gb: 8
      },
      features: ['advanced_analytics', 'broadcast', 'webhook', 'rich_menu', 'api_access']
    }
  };

  constructor() {
    this.stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
    this.redis = new Redis(process.env.REDIS_URL!);
    this.tenantRepo = new TenantRepository();
  }

  async createSubscription(config: {
    tenantId: string;
    email: string;
    plan: string;
    pricing: any;
    trialDays: number;
  }): Promise<Stripe.Subscription> {
    // สร้าง Stripe Customer
    const customer = await this.stripe.customers.create({
      email: config.email,
      metadata: { tenantId: config.tenantId }
    });

    // สร้าง Subscription
    const subscription = await this.stripe.subscriptions.create({
      customer: customer.id,
      items: [
        { price: await this.getStripePriceId(config.plan) }
      ],
      trial_period_days: config.trialDays,
      metadata: { tenantId: config.tenantId },
      expand: ['latest_invoice.payment_intent']
    });

    // เก็บ customer ID
    await this.tenantRepo.updateBillingInfo(config.tenantId, {
      stripeCustomerId: customer.id,
      stripeSubscriptionId: subscription.id,
      plan: config.plan,
      trialEndsAt: new Date(Date.now() + config.trialDays * 24 * 60 * 60 * 1000)
    });

    return subscription;
  }

  async recordUsage(record: UsageRecord): Promise<void> {
    const key = `usage:${record.tenantId}:${record.metric}:${this.getCurrentBillingPeriod()}`;
    await this.redis.incrby(key, record.quantity);
    
    // Set expiry for 2 billing periods
    await this.redis.expire(key, 60 * 24 * 60 * 60); // 60 วัน

    // ตรวจสอบ quota
    await this.checkQuota(record.tenantId, record.metric);
  }

  async getCurrentUsage(tenantId: string): Promise<Record<string, number>> {
    const period = this.getCurrentBillingPeriod();
    const metrics = ['messages', 'api_calls', 'users', 'storage_gb'];
    
    const usage: Record<string, number> = {};
    for (const metric of metrics) {
      const key = `usage:${tenantId}:${metric}:${period}`;
      const value = await this.redis.get(key);
      usage[metric] = parseInt(value || '0');
    }

    return usage;
  }

  async calculateOverageCharges(tenantId: string): Promise<number> {
    const tenant = await this.tenantRepo.findById(tenantId);
    if (!tenant) return 0;

    const plan = this.PLANS[tenant.plan];
    const usage = await this.getCurrentUsage(tenantId);
    
    let totalOverage = 0;

    for (const [metric, used] of Object.entries(usage)) {
      const included = plan.included[metric] || 0;
      const overageRate = plan.overage[metric] || 0;
      
      if (used > included) {
        totalOverage += (used - included) * overageRate;
      }
    }

    return totalOverage;
  }

  async generateInvoice(tenantId: string, month: string): Promise<any> {
    const tenant = await this.tenantRepo.findById(tenantId);
    if (!tenant) throw new Error('Tenant not found');

    const plan = this.PLANS[tenant.plan];
    const usage = await this.getUsageForPeriod(tenantId, month);
    const overage = await this.calculateOverageForPeriod(tenantId, month, usage);

    return {
      tenantId,
      period: month,
      items: [
        {
          description: `${plan.name} Plan - Base subscription`,
          quantity: 1,
          unitPrice: plan.basePrice,
          total: plan.basePrice
        },
        ...this.generateOverageLineItems(usage, plan),
      ],
      subtotal: plan.basePrice + overage,
      tax: (plan.basePrice + overage) * 0.07, // VAT 7%
      total: (plan.basePrice + overage) * 1.07,
      currency: 'THB'
    };
  }

  private generateOverageLineItems(usage: Record<string, number>, plan: BillingPlan): any[] {
    return Object.entries(usage)
      .filter(([metric, used]) => used > (plan.included[metric] || 0))
      .map(([metric, used]) => {
        const excess = used - plan.included[metric];
        const rate = plan.overage[metric];
        return {
          description: `${metric} overage (${excess} units @ ฿${rate})`,
          quantity: excess,
          unitPrice: rate,
          total: excess * rate
        };
      });
  }

  private async checkQuota(tenantId: string, metric: string): Promise<void> {
    const tenant = await this.tenantRepo.findById(tenantId);
    if (!tenant) return;

    const plan = this.PLANS[tenant.plan];
    const limit = plan.included[metric];
    
    if (limit === -1) return; // Unlimited

    const usage = await this.getCurrentUsage(tenantId);
    const used = usage[metric] || 0;

    // แจ้งเตือนเมื่อใช้ถึง 80%
    if (used >= limit * 0.8 && used < limit) {
      await this.sendQuotaWarning(tenantId, metric, used, limit);
    }

    // Throttle เมื่อถึง limit (ยกเว้น enterprise ที่จ่าย overage)
    if (used >= limit && tenant.plan === 'starter') {
      throw new Error(`Quota exceeded for ${metric}. Please upgrade your plan.`);
    }
  }

  private getCurrentBillingPeriod(): string {
    const now = new Date();
    return `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}`;
  }

  private async getStripePriceId(plan: string): Promise<string> {
    const priceIds: Record<string, string> = {
      starter: process.env.STRIPE_STARTER_PRICE_ID!,
      business: process.env.STRIPE_BUSINESS_PRICE_ID!,
      enterprise: process.env.STRIPE_ENTERPRISE_PRICE_ID!
    };
    return priceIds[plan];
  }

  private async sendQuotaWarning(tenantId: string, metric: string, used: number, limit: number): Promise<void> {
    // Send email/LINE notification
  }

  private async getUsageForPeriod(tenantId: string, period: string): Promise<Record<string, number>> {
    const metrics = ['messages', 'api_calls', 'users', 'storage_gb'];
    const usage: Record<string, number> = {};
    
    for (const metric of metrics) {
      const key = `usage:${tenantId}:${metric}:${period}`;
      const value = await this.redis.get(key);
      usage[metric] = parseInt(value || '0');
    }
    
    return usage;
  }

  private async calculateOverageForPeriod(tenantId: string, period: string, usage: Record<string, number>): Promise<number> {
    const tenant = await this.tenantRepo.findById(tenantId);
    if (!tenant) return 0;

    const plan = this.PLANS[tenant.plan];
    let total = 0;

    for (const [metric, used] of Object.entries(usage)) {
      const included = plan.included[metric] || 0;
      const rate = plan.overage[metric] || 0;
      if (used > included) {
        total += (used - included) * rate;
      }
    }

    return total;
  }
}
```

---

## 5. Custom Configuration Per Tenant

### 5.1 Tenant Configuration Service

```typescript
// config/TenantConfigService.ts
import { Redis } from 'ioredis';
import { SchemaManager } from '../database/SchemaManager';

interface TenantConfig {
  // Bot Behavior
  welcomeMessage: string;
  fallbackMessage: string;
  businessHours?: {
    enabled: boolean;
    timezone: string;
    schedule: Record<string, { open: string; close: string }>;
    outOfHoursMessage: string;
  };
  
  // LINE Settings
  richMenus: RichMenuConfig[];
  quickReplies: QuickReplyConfig[];
  
  // Integrations
  webhookUrl?: string;
  crmIntegration?: CRMConfig;
  aiConfig?: AIConfig;
  
  // Branding
  brandName: string;
  brandColor?: string;
  brandLogoUrl?: string;
  
  // Features
  features: Record<string, boolean>;
  
  // Rate Limits
  rateLimits: Record<string, number>;
}

interface RichMenuConfig {
  id: string;
  name: string;
  lineRichMenuId: string;
  isDefault: boolean;
  audiences?: AudienceConfig[];
}

interface QuickReplyConfig {
  label: string;
  text: string;
  action: 'message' | 'uri' | 'postback';
  value: string;
}

interface CRMConfig {
  type: 'salesforce' | 'hubspot' | 'custom';
  apiKey: string;
  syncFields: string[];
}

interface AIConfig {
  provider: 'openai' | 'anthropic' | 'google';
  model: string;
  systemPrompt: string;
  temperature: number;
}

interface AudienceConfig {
  type: 'tag' | 'user_list' | 'all';
  value?: string[];
}

export class TenantConfigService {
  private redis: Redis;
  private schemaManager: SchemaManager;
  
  private readonly CACHE_TTL = 300; // 5 นาที

  constructor() {
    this.redis = new Redis(process.env.REDIS_URL!);
    this.schemaManager = new SchemaManager();
  }

  async getConfig(tenantId: string, key?: string): Promise<any> {
    const cacheKey = `config:${tenantId}${key ? ':' + key : ''}`;
    
    // ลอง cache ก่อน
    const cached = await this.redis.get(cacheKey);
    if (cached) return JSON.parse(cached);

    // ดึงจาก database
    const conn = await this.schemaManager.getTenantConnection(tenantId);
    
    if (key) {
      const result = await conn.query(
        'SELECT value FROM bot_configs WHERE key = $1',
        [key]
      );
      const value = result.rows[0]?.value;
      await this.redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(value));
      return value;
    } else {
      const result = await conn.query('SELECT key, value FROM bot_configs');
      const config = Object.fromEntries(result.rows.map((r: any) => [r.key, r.value]));
      await this.redis.setex(cacheKey, this.CACHE_TTL, JSON.stringify(config));
      return config;
    }
  }

  async updateConfig(tenantId: string, key: string, value: any): Promise<void> {
    const conn = await this.schemaManager.getTenantConnection(tenantId);
    
    await conn.query(
      `INSERT INTO bot_configs (key, value, updated_at) VALUES ($1, $2, NOW())
       ON CONFLICT (key) DO UPDATE SET value = $2, updated_at = NOW()`,
      [key, JSON.stringify(value)]
    );

    // Invalidate cache
    await this.redis.del(`config:${tenantId}:${key}`);
    await this.redis.del(`config:${tenantId}`);

    // Publish config change event
    await this.redis.publish(
      `config_updated:${tenantId}`,
      JSON.stringify({ key, value, updatedAt: new Date() })
    );
  }

  async bulkUpdateConfig(tenantId: string, configs: Record<string, any>): Promise<void> {
    const conn = await this.schemaManager.getTenantConnection(tenantId);
    
    for (const [key, value] of Object.entries(configs)) {
      await conn.query(
        `INSERT INTO bot_configs (key, value, updated_at) VALUES ($1, $2, NOW())
         ON CONFLICT (key) DO UPDATE SET value = $2, updated_at = NOW()`,
        [key, JSON.stringify(value)]
      );
    }

    // Invalidate entire tenant cache
    const keys = await this.redis.keys(`config:${tenantId}:*`);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }

  async watchConfigChanges(tenantId: string, callback: (key: string, value: any) => void): Promise<void> {
    const subscriber = this.redis.duplicate();
    await subscriber.subscribe(`config_updated:${tenantId}`);
    
    subscriber.on('message', (channel, message) => {
      const { key, value } = JSON.parse(message);
      callback(key, value);
    });
  }
}
```

---

## 6. White-Label LINE Bots

### 6.1 White-Label Configuration

```typescript
// whitelabel/WhiteLabelService.ts
import { LINEAPIClient } from '../line/LINEAPIClient';
import { AssetService } from './AssetService';

interface WhiteLabelConfig {
  tenantId: string;
  botName: string;
  botIconUrl: string;
  botDescription: string;
  welcomeMessage: string;
  primaryColor: string;
  secondaryColor: string;
  logoUrl: string;
  customDomain?: string;
  customTermsUrl?: string;
  customPrivacyUrl?: string;
  socialLinks?: Record<string, string>;
}

export class WhiteLabelService {
  private lineClient: LINEAPIClient;
  private assetService: AssetService;

  constructor(tenantId: string) {
    const channelAccessToken = this.getTenantAccessToken(tenantId);
    this.lineClient = new LINEAPIClient(channelAccessToken);
    this.assetService = new AssetService();
  }

  async applyWhiteLabel(config: WhiteLabelConfig): Promise<void> {
    // 1. Upload assets
    const processedLogo = await this.assetService.processLogo(config.logoUrl);
    const processedIcon = await this.assetService.processIcon(config.botIconUrl);

    // 2. Generate themed Rich Menu
    await this.generateThemedRichMenu(config);

    // 3. Update LIFF apps branding
    await this.updateLIFFBranding(config);

    // 4. Apply custom CSS for LIFF
    await this.generateCustomCSS(config);

    // 5. Update welcome message
    await this.updateWelcomeContent(config);
  }

  async generateThemedRichMenu(config: WhiteLabelConfig): Promise<void> {
    // สร้าง Rich Menu image ด้วย Dynamic generation
    const richMenuImage = await this.generateRichMenuImage({
      primaryColor: config.primaryColor,
      secondaryColor: config.secondaryColor,
      logoUrl: config.logoUrl,
      buttons: [
        { label: 'หน้าหลัก', iconUrl: '' },
        { label: 'ติดต่อเรา', iconUrl: '' },
        { label: 'สินค้า', iconUrl: '' }
      ]
    });

    const richMenu = await this.lineClient.createRichMenu({
      size: { width: 2500, height: 843 },
      selected: true,
      name: `${config.botName} Main Menu`,
      chatBarText: 'เมนู',
      areas: [
        {
          bounds: { x: 0, y: 0, width: 833, height: 843 },
          action: { type: 'message', text: 'หน้าหลัก' }
        },
        {
          bounds: { x: 833, y: 0, width: 834, height: 843 },
          action: { type: 'message', text: 'ติดต่อเรา' }
        },
        {
          bounds: { x: 1667, y: 0, width: 833, height: 843 },
          action: { type: 'message', text: 'สินค้า' }
        }
      ]
    });

    await this.lineClient.uploadRichMenuImage(richMenu.richMenuId, richMenuImage);
    await this.lineClient.setDefaultRichMenu(richMenu.richMenuId);
  }

  private async generateRichMenuImage(config: any): Promise<Buffer> {
    const { createCanvas, loadImage } = require('canvas');
    const canvas = createCanvas(2500, 843);
    const ctx = canvas.getContext('2d');

    // วาด background
    ctx.fillStyle = config.primaryColor;
    ctx.fillRect(0, 0, 2500, 843);

    // วาด dividers
    ctx.strokeStyle = config.secondaryColor;
    ctx.lineWidth = 4;
    ctx.beginPath();
    ctx.moveTo(833, 0);
    ctx.lineTo(833, 843);
    ctx.moveTo(1667, 0);
    ctx.lineTo(1667, 843);
    ctx.stroke();

    // เพิ่ม text สำหรับแต่ละ button
    ctx.fillStyle = '#FFFFFF';
    ctx.font = 'bold 60px Arial';
    ctx.textAlign = 'center';
    ctx.textBaseline = 'middle';

    config.buttons.forEach((btn: any, i: number) => {
      const x = (i * 833) + 416;
      ctx.fillText(btn.label, x, 421);
    });

    return canvas.toBuffer('image/png');
  }

  private async generateCustomCSS(config: WhiteLabelConfig): Promise<string> {
    return `
      :root {
        --primary-color: ${config.primaryColor};
        --secondary-color: ${config.secondaryColor};
        --brand-name: "${config.botName}";
      }

      .header-logo {
        background-image: url('${config.logoUrl}');
        background-size: contain;
        background-repeat: no-repeat;
      }

      .btn-primary {
        background-color: var(--primary-color);
        border-color: var(--primary-color);
      }

      .btn-primary:hover {
        background-color: var(--secondary-color);
        border-color: var(--secondary-color);
      }

      .brand-title::before {
        content: var(--brand-name);
      }
    `;
  }

  private async updateLIFFBranding(config: WhiteLabelConfig): Promise<void> {
    // Update LIFF app settings
  }

  private async updateWelcomeContent(config: WhiteLabelConfig): Promise<void> {
    // Update welcome flex message with brand colors
  }

  private getTenantAccessToken(tenantId: string): string {
    // Load from secure storage
    return '';
  }
}
```

---

## 7. Admin Portal for Tenant Management

### 7.1 Admin API

```typescript
// admin/AdminController.ts
import { Router, Request, Response } from 'express';
import { TenantRepository } from '../repositories/TenantRepository';
import { TenantProvisioner } from '../provisioning/TenantProvisioner';
import { BillingService } from '../billing/BillingService';
import { AdminAuthMiddleware } from '../auth/AdminAuthMiddleware';

export function createAdminRouter(): Router {
  const router = Router();
  const tenantRepo = new TenantRepository();
  const provisioner = new TenantProvisioner();
  const billing = new BillingService();
  const adminAuth = new AdminAuthMiddleware();

  // Apply admin authentication
  router.use(adminAuth.requireSuperAdmin());

  // List all tenants
  router.get('/tenants', async (req: Request, res: Response) => {
    try {
      const { page = 1, limit = 20, status, plan, search } = req.query;
      
      const tenants = await tenantRepo.findAll({
        page: Number(page),
        limit: Number(limit),
        filters: {
          status: status as string,
          plan: plan as string,
          search: search as string
        }
      });

      res.json({
        data: tenants.items,
        pagination: {
          page: Number(page),
          limit: Number(limit),
          total: tenants.total,
          pages: Math.ceil(tenants.total / Number(limit))
        }
      });
    } catch (error) {
      res.status(500).json({ error: (error as Error).message });
    }
  });

  // Get tenant details
  router.get('/tenants/:tenantId', async (req: Request, res: Response) => {
    try {
      const tenant = await tenantRepo.findById(req.params.tenantId);
      if (!tenant) {
        res.status(404).json({ error: 'Tenant not found' });
        return;
      }

      const usage = await billing.getCurrentUsage(req.params.tenantId);
      const invoice = await billing.generateInvoice(
        req.params.tenantId,
        new Date().toISOString().substring(0, 7)
      );

      res.json({
        tenant,
        usage,
        currentBill: invoice
      });
    } catch (error) {
      res.status(500).json({ error: (error as Error).message });
    }
  });

  // Create new tenant
  router.post('/tenants', async (req: Request, res: Response) => {
    try {
      const tenant = await provisioner.provisionTenant(req.body);
      res.status(201).json({ tenant });
    } catch (error) {
      res.status(400).json({ error: (error as Error).message });
    }
  });

  // Update tenant status
  router.patch('/tenants/:tenantId/status', async (req: Request, res: Response) => {
    try {
      const { status, reason } = req.body;
      
      await tenantRepo.updateStatus(req.params.tenantId, status);
      
      // Audit log
      await this.auditLog({
        action: 'TENANT_STATUS_CHANGED',
        tenantId: req.params.tenantId,
        oldStatus: 'unknown',
        newStatus: status,
        reason,
        adminId: (req as any).admin.id
      });

      res.json({ success: true });
    } catch (error) {
      res.status(400).json({ error: (error as Error).message });
    }
  });

  // Upgrade/Downgrade plan
  router.post('/tenants/:tenantId/plan', async (req: Request, res: Response) => {
    try {
      const { newPlan, effectiveDate } = req.body;
      
      await billing.changePlan(req.params.tenantId, newPlan, effectiveDate);
      
      res.json({ success: true, message: `Plan changed to ${newPlan}` });
    } catch (error) {
      res.status(400).json({ error: (error as Error).message });
    }
  });

  // Get tenant usage metrics
  router.get('/tenants/:tenantId/metrics', async (req: Request, res: Response) => {
    try {
      const { period = 'month' } = req.query;
      
      const metrics = await this.getDetailedMetrics(req.params.tenantId, period as string);
      res.json(metrics);
    } catch (error) {
      res.status(500).json({ error: (error as Error).message });
    }
  });

  // Force logout all sessions for tenant
  router.post('/tenants/:tenantId/force-logout', async (req: Request, res: Response) => {
    try {
      await this.invalidateAllTenantSessions(req.params.tenantId);
      res.json({ success: true });
    } catch (error) {
      res.status(500).json({ error: (error as Error).message });
    }
  });

  // Get platform-wide statistics
  router.get('/statistics', async (req: Request, res: Response) => {
    try {
      const stats = await this.getPlatformStatistics();
      res.json(stats);
    } catch (error) {
      res.status(500).json({ error: (error as Error).message });
    }
  });

  return router;
}
```

---

## 8. Tenant Onboarding Flow

### 8.1 Step-by-Step Onboarding

```typescript
// onboarding/OnboardingService.ts
import { Redis } from 'ioredis';
import { TenantConfigService } from '../config/TenantConfigService';

interface OnboardingStep {
  id: string;
  title: string;
  description: string;
  isRequired: boolean;
  isCompleted: boolean;
  completedAt?: Date;
  order: number;
}

interface OnboardingProgress {
  tenantId: string;
  currentStep: number;
  steps: OnboardingStep[];
  completedAt?: Date;
  startedAt: Date;
}

export class OnboardingService {
  private redis: Redis;
  private configService: TenantConfigService;

  private readonly ONBOARDING_STEPS: Omit<OnboardingStep, 'isCompleted' | 'completedAt'>[] = [
    {
      id: 'line_channel',
      title: 'เชื่อมต่อ LINE Official Account',
      description: 'กรอก Channel ID, Channel Secret และ Channel Access Token',
      isRequired: true,
      order: 1
    },
    {
      id: 'webhook_setup',
      title: 'ตั้งค่า Webhook',
      description: 'กำหนด Webhook URL และทดสอบการเชื่อมต่อ',
      isRequired: true,
      order: 2
    },
    {
      id: 'welcome_message',
      title: 'ตั้งค่าข้อความต้อนรับ',
      description: 'กำหนดข้อความที่จะส่งเมื่อมี user ใหม่ follow',
      isRequired: false,
      order: 3
    },
    {
      id: 'rich_menu',
      title: 'สร้าง Rich Menu',
      description: 'ออกแบบ Rich Menu สำหรับ bot ของคุณ',
      isRequired: false,
      order: 4
    },
    {
      id: 'test_bot',
      title: 'ทดสอบ Bot',
      description: 'ส่งข้อความทดสอบและตรวจสอบการทำงาน',
      isRequired: true,
      order: 5
    },
    {
      id: 'billing',
      title: 'ตั้งค่าการชำระเงิน',
      description: 'เพิ่มข้อมูลการชำระเงินเพื่อใช้งานต่อหลังหมด Trial',
      isRequired: false,
      order: 6
    }
  ];

  constructor() {
    this.redis = new Redis(process.env.REDIS_URL!);
    this.configService = new TenantConfigService();
  }

  async getProgress(tenantId: string): Promise<OnboardingProgress> {
    const key = `onboarding:${tenantId}`;
    const stored = await this.redis.get(key);
    
    if (stored) return JSON.parse(stored);

    // สร้าง progress ใหม่
    const progress: OnboardingProgress = {
      tenantId,
      currentStep: 1,
      startedAt: new Date(),
      steps: this.ONBOARDING_STEPS.map(step => ({
        ...step,
        isCompleted: false
      }))
    };

    await this.saveProgress(tenantId, progress);
    return progress;
  }

  async completeStep(tenantId: string, stepId: string, data?: any): Promise<OnboardingProgress> {
    const progress = await this.getProgress(tenantId);
    
    const step = progress.steps.find(s => s.id === stepId);
    if (!step) throw new Error(`Step ${stepId} not found`);

    // Validate step completion
    await this.validateStepCompletion(tenantId, stepId, data);

    step.isCompleted = true;
    step.completedAt = new Date();

    // Process step-specific actions
    await this.processStepCompletion(tenantId, stepId, data);

    // Update current step
    const nextStep = progress.steps.find(s => !s.isCompleted && s.order > step.order);
    if (nextStep) {
      progress.currentStep = nextStep.order;
    }

    // Check if onboarding is complete
    const requiredSteps = progress.steps.filter(s => s.isRequired);
    if (requiredSteps.every(s => s.isCompleted)) {
      progress.completedAt = new Date();
      await this.handleOnboardingComplete(tenantId);
    }

    await this.saveProgress(tenantId, progress);
    return progress;
  }

  private async validateStepCompletion(tenantId: string, stepId: string, data: any): Promise<void> {
    switch (stepId) {
      case 'line_channel':
        if (!data?.channelId || !data?.channelSecret || !data?.accessToken) {
          throw new Error('Missing LINE channel credentials');
        }
        // Verify credentials with LINE API
        await this.verifyLINECredentials(data);
        break;
      
      case 'webhook_setup':
        if (!data?.webhookUrl) {
          throw new Error('Missing webhook URL');
        }
        // Test webhook connectivity
        await this.testWebhookConnectivity(data.webhookUrl);
        break;
      
      case 'test_bot':
        // Check if test message was sent and received
        const testResult = await this.checkTestMessageResult(tenantId);
        if (!testResult.success) {
          throw new Error('Bot test failed: ' + testResult.message);
        }
        break;
    }
  }

  private async processStepCompletion(tenantId: string, stepId: string, data: any): Promise<void> {
    switch (stepId) {
      case 'line_channel':
        await this.configService.updateConfig(tenantId, 'line_channel', {
          channelId: data.channelId,
          channelSecret: data.channelSecret,
          accessToken: data.accessToken
        });
        break;
      
      case 'welcome_message':
        await this.configService.updateConfig(tenantId, 'welcome_message', data.message);
        break;
      
      case 'rich_menu':
        await this.configService.updateConfig(tenantId, 'default_rich_menu', data.richMenuId);
        break;
    }
  }

  private async handleOnboardingComplete(tenantId: string): Promise<void> {
    await this.configService.updateConfig(tenantId, 'onboarding_completed', true);
    
    // Track completion
    console.log(`Tenant ${tenantId} completed onboarding`);
  }

  private async saveProgress(tenantId: string, progress: OnboardingProgress): Promise<void> {
    const key = `onboarding:${tenantId}`;
    await this.redis.set(key, JSON.stringify(progress));
  }

  private async verifyLINECredentials(data: any): Promise<void> {
    // Verify with LINE API
  }

  private async testWebhookConnectivity(webhookUrl: string): Promise<void> {
    // Test webhook
  }

  private async checkTestMessageResult(tenantId: string): Promise<{ success: boolean; message?: string }> {
    return { success: true };
  }
}
```

---

## 9. Data Migration Between Tenants

### 9.1 Tenant Data Migration Service

```typescript
// migration/TenantMigrationService.ts
import { SchemaManager } from '../database/SchemaManager';
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { createGzip, createGunzip } from 'zlib';
import { pipeline } from 'stream/promises';

interface MigrationConfig {
  sourceTenantId: string;
  targetTenantId: string;
  tables: string[];
  includeMedia: boolean;
  deleteSource: boolean;
}

interface MigrationResult {
  id: string;
  status: 'success' | 'partial' | 'failed';
  recordsMigrated: Record<string, number>;
  errors: string[];
  startedAt: Date;
  completedAt?: Date;
  exportUrl?: string;
}

export class TenantMigrationService {
  private schemaManager: SchemaManager;
  private s3Client: S3Client;

  constructor() {
    this.schemaManager = new SchemaManager();
    this.s3Client = new S3Client({ region: process.env.AWS_REGION });
  }

  async migrateTenant(config: MigrationConfig): Promise<MigrationResult> {
    const migrationId = `mig_${Date.now()}`;
    const result: MigrationResult = {
      id: migrationId,
      status: 'failed',
      recordsMigrated: {},
      errors: [],
      startedAt: new Date()
    };

    try {
      // 1. Validate both tenants exist
      await this.validateTenants(config.sourceTenantId, config.targetTenantId);

      // 2. Create export
      const exportData = await this.exportTenantData(config.sourceTenantId, config.tables);

      // 3. Transform data for target tenant
      const transformedData = await this.transformData(exportData, config);

      // 4. Import into target tenant
      for (const [table, records] of Object.entries(transformedData)) {
        await this.importTableData(config.targetTenantId, table, records as any[]);
        result.recordsMigrated[table] = (records as any[]).length;
      }

      // 5. Migrate media files if requested
      if (config.includeMedia) {
        await this.migrateMediaFiles(config.sourceTenantId, config.targetTenantId);
      }

      // 6. Verify migration
      const verificationResult = await this.verifyMigration(config, result.recordsMigrated);
      if (!verificationResult.success) {
        result.errors.push(...verificationResult.errors);
        result.status = 'partial';
      } else {
        result.status = 'success';
      }

      // 7. Archive source if requested
      if (config.deleteSource && result.status === 'success') {
        const exportUrl = await this.archiveTenantData(config.sourceTenantId);
        result.exportUrl = exportUrl;
        await this.schemaManager.deleteTenantSchema(config.sourceTenantId, false);
      }

    } catch (error) {
      result.errors.push((error as Error).message);
      result.status = 'failed';
    }

    result.completedAt = new Date();
    return result;
  }

  private async exportTenantData(tenantId: string, tables: string[]): Promise<Record<string, any[]>> {
    const conn = await this.schemaManager.getTenantConnection(tenantId);
    const data: Record<string, any[]> = {};

    for (const table of tables) {
      const result = await conn.query(`SELECT * FROM ${table} ORDER BY created_at ASC`);
      data[table] = result.rows;
    }

    return data;
  }

  private async transformData(data: Record<string, any[]>, config: MigrationConfig): Promise<Record<string, any[]>> {
    const transformed: Record<string, any[]> = {};
    
    for (const [table, records] of Object.entries(data)) {
      transformed[table] = records.map(record => ({
        ...record,
        // ID mapping จะต้องทำตาม business logic
        id: require('crypto').randomUUID(),
        migrated_from: config.sourceTenantId,
        migrated_at: new Date()
      }));
    }

    return transformed;
  }

  private async importTableData(tenantId: string, table: string, records: any[]): Promise<void> {
    if (records.length === 0) return;

    const conn = await this.schemaManager.getTenantConnection(tenantId);
    const columns = Object.keys(records[0]);
    const placeholders = records.map((_, rowIndex) =>
      `(${columns.map((_, colIndex) => `$${rowIndex * columns.length + colIndex + 1}`).join(', ')})`
    ).join(', ');

    const values = records.flatMap(record => columns.map(col => record[col]));

    await conn.query(
      `INSERT INTO ${table} (${columns.join(', ')}) VALUES ${placeholders} ON CONFLICT (id) DO NOTHING`,
      values
    );
  }

  private async migrateMediaFiles(sourceTenantId: string, targetTenantId: string): Promise<void> {
    // Copy S3 objects from source to target prefix
    const sourcePrefix = `tenants/${sourceTenantId}/`;
    const targetPrefix = `tenants/${targetTenantId}/`;
    
    // List and copy all objects
    console.log(`Migrating media from ${sourcePrefix} to ${targetPrefix}`);
  }

  private async verifyMigration(config: MigrationConfig, recordsMigrated: Record<string, number>): Promise<{
    success: boolean;
    errors: string[];
  }> {
    const errors: string[] = [];

    for (const [table, count] of Object.entries(recordsMigrated)) {
      const sourceConn = await this.schemaManager.getTenantConnection(config.sourceTenantId);
      const targetConn = await this.schemaManager.getTenantConnection(config.targetTenantId);

      const [sourceResult, targetResult] = await Promise.all([
        sourceConn.query(`SELECT COUNT(*) FROM ${table}`),
        targetConn.query(`SELECT COUNT(*) FROM ${table}`)
      ]);

      const sourceCount = parseInt(sourceResult.rows[0].count);
      const targetCount = parseInt(targetResult.rows[0].count);

      if (targetCount < count) {
        errors.push(`Table ${table}: Expected ${count} records, got ${targetCount}`);
      }
    }

    return {
      success: errors.length === 0,
      errors
    };
  }

  private async archiveTenantData(tenantId: string): Promise<string> {
    const conn = await this.schemaManager.getTenantConnection(tenantId);
    const tables = ['users', 'conversations', 'messages', 'analytics_events'];
    const exportData: Record<string, any[]> = {};

    for (const table of tables) {
      const result = await conn.query(`SELECT * FROM ${table}`);
      exportData[table] = result.rows;
    }

    const json = JSON.stringify(exportData);
    const key = `archives/${tenantId}/${Date.now()}.json.gz`;

    // Upload compressed archive to S3
    const gzipped = require('zlib').gzipSync(Buffer.from(json));
    
    await this.s3Client.send(new PutObjectCommand({
      Bucket: process.env.ARCHIVE_BUCKET!,
      Key: key,
      Body: gzipped,
      ContentEncoding: 'gzip',
      ContentType: 'application/json'
    }));

    return `s3://${process.env.ARCHIVE_BUCKET}/${key}`;
  }

  private async validateTenants(...tenantIds: string[]): Promise<void> {
    for (const tenantId of tenantIds) {
      const exists = await this.schemaManager.tenantExists(tenantId);
      if (!exists) throw new Error(`Tenant ${tenantId} not found`);
    }
  }
}
```

---

## สรุป

การสร้าง Multi-Tenant LINE Bot Platform ต้องคำนึงถึง:

1. **Data Isolation** - เลือก strategy ที่เหมาะสมกับ use case (Shared DB, Schema, หรือ Database)
2. **Tenant Provisioning** - ทำให้กระบวนการสร้าง tenant อัตโนมัติและเชื่อถือได้
3. **Billing** - ระบบ billing ที่ยุติธรรมและโปร่งใส
4. **Custom Configuration** - ให้ tenant ปรับแต่งได้ตามต้องการ
5. **White-Label** - รองรับ custom branding
6. **Admin Portal** - เครื่องมือสำหรับจัดการ tenants
7. **Onboarding** - กระบวนการ onboarding ที่ smooth
8. **Data Migration** - เครื่องมือย้ายข้อมูลที่ปลอดภัย

Platform ที่ดีต้องสมดุลระหว่าง **isolation** (ความปลอดภัย/ความเป็นส่วนตัว) และ **efficiency** (ต้นทุน/ประสิทธิภาพ) ตามความต้องการของธุรกิจ
