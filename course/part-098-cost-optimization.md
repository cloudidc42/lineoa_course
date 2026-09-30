# Part 98: Cost Optimization สำหรับ LINE Bot Systems

## บทนำ

การบริหารต้นทุนเป็นหนึ่งในความท้าทายสำคัญของการ run LINE Bot ใน production เพราะค่าใช้จ่ายอาจเพิ่มขึ้นอย่างรวดเร็วเมื่อมีผู้ใช้มากขึ้น บทนี้จะให้แนวทางครอบคลุมในการลดต้นทุนโดยไม่กระทบ performance และ reliability

---

## 1. LINE API Message Quota Optimization

### 1.1 ทำความเข้าใจ LINE Messaging API Pricing

```
LINE OA Pricing (2024):

Free Plan:
- Broadcast: 200 messages/เดือน
- Reply & Push: ไม่จำกัด (เฉพาะ reply token)

Light Plan: ฿399/เดือน
- Broadcast: 500 messages/เดือน
- Push: ฿0.17 ต่อ message หลังจาก 500

Standard Plan: ฿1,199/เดือน
- Broadcast: 3,000 messages/เดือน
- Push: ฿0.17 ต่อ message หลังจาก 3,000

Premium Plan: ฿1,999/เดือน
- Broadcast: ไม่จำกัด
- Push: ฿0.17 ต่อ message
```

### 1.2 Message Quota Monitoring

```typescript
// monitoring/quota-monitor.ts
import { LineClient } from '@line/bot-sdk';
import { Redis } from 'ioredis';
import { logger } from '../utils/logger';

interface QuotaStatus {
  totalUsage: number;
  monthlyLimit: number;
  remainingQuota: number;
  utilizationRate: number;
  projectedMonthEnd: number;
  alertLevel: 'normal' | 'warning' | 'critical';
}

export class LineQuotaMonitor {
  private lineClient: LineClient;
  private redis: Redis;
  private alertThresholds = {
    warning: 0.7,   // 70% used
    critical: 0.9,  // 90% used
  };

  constructor(lineClient: LineClient, redis: Redis) {
    this.lineClient = lineClient;
    this.redis = redis;
  }

  async getQuotaStatus(): Promise<QuotaStatus> {
    const [consumption, limit] = await Promise.all([
      this.lineClient.getMessageAPIQuotaConsumption(),
      this.lineClient.getMessageAPIQuota(),
    ]);

    const totalUsage = consumption.totalUsage;
    const monthlyLimit = limit.value || 999999;
    const remainingQuota = monthlyLimit - totalUsage;
    const utilizationRate = totalUsage / monthlyLimit;

    // คำนวณ projected usage
    const today = new Date().getDate();
    const daysInMonth = new Date(
      new Date().getFullYear(),
      new Date().getMonth() + 1,
      0
    ).getDate();
    const projectedMonthEnd = Math.ceil((totalUsage / today) * daysInMonth);

    let alertLevel: QuotaStatus['alertLevel'] = 'normal';
    if (utilizationRate >= this.alertThresholds.critical) {
      alertLevel = 'critical';
    } else if (utilizationRate >= this.alertThresholds.warning) {
      alertLevel = 'warning';
    }

    // Cache ใน Redis
    const status: QuotaStatus = {
      totalUsage,
      monthlyLimit,
      remainingQuota,
      utilizationRate,
      projectedMonthEnd,
      alertLevel,
    };

    await this.redis.setex(
      'line:quota:status',
      300, // cache 5 minutes
      JSON.stringify(status)
    );

    if (alertLevel !== 'normal') {
      await this.sendAlert(status);
    }

    return status;
  }

  private async sendAlert(status: QuotaStatus): Promise<void> {
    const message = `⚠️ LINE API Quota Alert
Level: ${status.alertLevel.toUpperCase()}
Used: ${status.totalUsage.toLocaleString()} / ${status.monthlyLimit.toLocaleString()}
Utilization: ${(status.utilizationRate * 100).toFixed(1)}%
Projected Month-end: ${status.projectedMonthEnd.toLocaleString()}`;

    // ส่งแจ้งเตือนผ่าน Slack, LINE, Email
    logger.warn('LINE Quota Alert', status);
  }
}
```

### 1.3 Smart Message Deduplication

```typescript
// optimization/message-dedup.ts
import { Redis } from 'ioredis';

export class MessageDeduplicator {
  constructor(
    private redis: Redis,
    private deduplicationWindow: number = 3600 // 1 hour
  ) {}

  async isDuplicate(
    userId: string,
    messageHash: string
  ): Promise<boolean> {
    const key = `msg:dedup:${userId}:${messageHash}`;
    const result = await this.redis.set(
      key,
      '1',
      'EX',
      this.deduplicationWindow,
      'NX' // Only set if not exists
    );
    
    return result === null; // null means key already existed = duplicate
  }

  hashMessage(message: any): string {
    const crypto = require('crypto');
    return crypto
      .createHash('md5')
      .update(JSON.stringify(message))
      .digest('hex');
  }
}

// ใช้ใน Push message service
export class OptimizedPushService {
  private dedup: MessageDeduplicator;
  private quotaMonitor: LineQuotaMonitor;

  async sendMessage(userId: string, message: any): Promise<boolean> {
    // ตรวจสอบ quota ก่อน
    const quota = await this.quotaMonitor.getQuotaStatus();
    if (quota.alertLevel === 'critical') {
      logger.warn(`Skipping push to ${userId}: quota critical`);
      return false;
    }

    // ตรวจสอบ duplicate
    const hash = this.dedup.hashMessage(message);
    if (await this.dedup.isDuplicate(userId, hash)) {
      logger.debug(`Skipping duplicate message to ${userId}`);
      return false;
    }

    // ส่ง message
    await this.lineClient.pushMessage(userId, message);
    return true;
  }
}
```

---

## 2. Message Content Optimization

### 2.1 Flex Message vs Text Message

```
ต้นทุน Flex Message:
- 1 Flex Message = 1 message quota
- แต่แสดงผลได้ดีกว่า Text หลาย messages
- ประหยัดกว่าการส่ง multiple text messages

ตัวอย่าง:
❌ แบบเปลืองเงิน: 5 push messages
"สถานะคำสั่งซื้อ: กำลังจัดส่ง"
"หมายเลขติดตาม: TH12345678"
"บริษัทขนส่ง: Kerry Express"
"วันที่จัดส่ง: 1 ม.ค. 2024"
"คาดว่าจะถึง: 3 ม.ค. 2024"

✅ แบบประหยัด: 1 flex message
[Flex ที่แสดงข้อมูลครบทุกอย่าง]
```

```typescript
// optimization/flex-optimizer.ts
export class FlexMessageOptimizer {
  /**
   * รวม notifications หลายอันเป็น Flex message เดียว
   */
  combineNotifications(notifications: Notification[]): FlexMessage {
    return {
      type: 'flex',
      altText: `คุณมี ${notifications.length} การแจ้งเตือนใหม่`,
      contents: {
        type: 'bubble',
        header: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'text',
            text: `🔔 ${notifications.length} การแจ้งเตือนใหม่`,
            weight: 'bold',
            size: 'lg',
          }],
        },
        body: {
          type: 'box',
          layout: 'vertical',
          spacing: 'sm',
          contents: notifications.map(n => ({
            type: 'box',
            layout: 'horizontal',
            contents: [
              {
                type: 'text',
                text: n.icon,
                size: 'sm',
                flex: 1,
              },
              {
                type: 'text',
                text: n.message,
                size: 'sm',
                flex: 9,
                wrap: true,
              },
            ],
          })),
        },
      },
    };
  }

  /**
   * Order summary ที่แสดงได้ใน 1 message
   */
  createOrderSummaryFlex(order: any): FlexMessage {
    const statusColor = this.getStatusColor(order.status);
    
    return {
      type: 'flex',
      altText: `สถานะคำสั่งซื้อ #${order.orderNumber}: ${order.status}`,
      contents: {
        type: 'bubble',
        styles: {
          header: { backgroundColor: statusColor },
        },
        header: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'text',
            text: `คำสั่งซื้อ #${order.orderNumber}`,
            color: '#ffffff',
            weight: 'bold',
          }],
        },
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            { type: 'text', text: `สถานะ: ${order.statusText}`, size: 'md' },
            { type: 'text', text: `ยอดรวม: ฿${order.total.toLocaleString()}`, size: 'sm' },
            order.trackingNumber ? {
              type: 'text',
              text: `เลขพัสดุ: ${order.trackingNumber}`,
              size: 'sm',
            } : null,
          ].filter(Boolean),
        },
        footer: {
          type: 'box',
          layout: 'horizontal',
          contents: [
            {
              type: 'button',
              action: {
                type: 'uri',
                label: 'ดูรายละเอียด',
                uri: `${process.env.LIFF_URL}?orderId=${order.id}`,
              },
              style: 'primary',
              height: 'sm',
            },
          ],
        },
      },
    };
  }

  private getStatusColor(status: string): string {
    const colors: Record<string, string> = {
      pending: '#FF9800',
      confirmed: '#2196F3',
      paid: '#4CAF50',
      shipped: '#9C27B0',
      delivered: '#4CAF50',
      cancelled: '#F44336',
    };
    return colors[status] || '#757575';
  }
}
```

### 2.2 Message Batching

```typescript
// optimization/message-batcher.ts
export class MessageBatcher {
  private queue: Map<string, any[]> = new Map();
  private flushInterval: NodeJS.Timeout;
  private batchWindow: number = 5000; // 5 seconds
  private maxBatchSize: number = 5;

  constructor(private lineClient: LineClient) {
    this.flushInterval = setInterval(() => {
      this.flushAll();
    }, this.batchWindow);
  }

  async addMessage(userId: string, message: any): Promise<void> {
    const userQueue = this.queue.get(userId) || [];
    userQueue.push(message);
    this.queue.set(userId, userQueue);

    // Auto-flush if batch is full
    if (userQueue.length >= this.maxBatchSize) {
      await this.flushUser(userId);
    }
  }

  private async flushUser(userId: string): Promise<void> {
    const messages = this.queue.get(userId);
    if (!messages || messages.length === 0) return;

    this.queue.delete(userId);

    try {
      if (messages.length === 1) {
        await this.lineClient.pushMessage(userId, messages[0]);
      } else {
        // รวมเป็น 1 push กับหลาย messages (ประหยัด quota)
        await this.lineClient.pushMessage(userId, messages.slice(0, 5));
      }
    } catch (error) {
      logger.error(`Failed to send batch to ${userId}`, error);
    }
  }

  private async flushAll(): Promise<void> {
    const users = Array.from(this.queue.keys());
    await Promise.allSettled(users.map(userId => this.flushUser(userId)));
  }

  destroy(): void {
    clearInterval(this.flushInterval);
  }
}
```

---

## 3. Targeting Optimization

### 3.1 ส่งหาคนที่ใช่เท่านั้น

```typescript
// targeting/smart-targeting.ts
export class SmartTargetingService {
  
  /**
   * กรองผู้ใช้ที่ควรได้รับ notification
   */
  async filterRelevantUsers(
    campaignType: string,
    allUserIds: string[]
  ): Promise<string[]> {
    const filters = this.getFiltersForCampaign(campaignType);
    
    let targetUsers = allUserIds;
    
    for (const filter of filters) {
      targetUsers = await this.applyFilter(targetUsers, filter);
    }
    
    return targetUsers;
  }

  private getFiltersForCampaign(type: string): UserFilter[] {
    const filterMap: Record<string, UserFilter[]> = {
      // ส่งโปรโมชั่นเฉพาะคนที่ active
      promotion: [
        { type: 'active_within_days', value: 30 },
        { type: 'has_not_purchased', value: 7 },
        { type: 'not_blocked', value: true },
      ],
      
      // แจ้งเตือน order เฉพาะคนที่มี pending orders
      order_update: [
        { type: 'has_pending_orders', value: true },
        { type: 'notifications_enabled', value: true },
      ],
      
      // Welcome message เฉพาะ new followers
      welcome: [
        { type: 'followed_within_hours', value: 1 },
        { type: 'is_new_user', value: true },
      ],
    };
    
    return filterMap[type] || [];
  }

  private async applyFilter(
    users: string[],
    filter: UserFilter
  ): Promise<string[]> {
    switch (filter.type) {
      case 'active_within_days':
        return this.filterActiveUsers(users, filter.value as number);
      
      case 'has_not_purchased':
        return this.filterNonPurchasers(users, filter.value as number);
      
      case 'not_blocked':
        return this.filterNonBlocked(users);
      
      default:
        return users;
    }
  }

  private async filterActiveUsers(
    users: string[],
    days: number
  ): Promise<string[]> {
    const cutoff = new Date();
    cutoff.setDate(cutoff.getDate() - days);
    
    const result = await db.query(
      `SELECT line_user_id FROM user_activities
       WHERE line_user_id = ANY($1)
       AND last_active_at >= $2`,
      [users, cutoff]
    );
    
    return result.rows.map(r => r.line_user_id);
  }

  private async filterNonPurchasers(
    users: string[],
    days: number
  ): Promise<string[]> {
    const cutoff = new Date();
    cutoff.setDate(cutoff.getDate() - days);
    
    const purchasers = await db.query(
      `SELECT DISTINCT line_user_id FROM orders
       WHERE line_user_id = ANY($1)
       AND created_at >= $2
       AND status NOT IN ('cancelled', 'refunded')`,
      [users, cutoff]
    );
    
    const purchaserSet = new Set(purchasers.rows.map((r: any) => r.line_user_id));
    return users.filter(id => !purchaserSet.has(id));
  }

  private async filterNonBlocked(users: string[]): Promise<string[]> {
    const blocked = await db.query(
      `SELECT line_user_id FROM users
       WHERE line_user_id = ANY($1)
       AND is_blocked = true`,
      [users]
    );
    
    const blockedSet = new Set(blocked.rows.map((r: any) => r.line_user_id));
    return users.filter(id => !blockedSet.has(id));
  }

  /**
   * คำนวณประสิทธิภาพของ targeting
   */
  async calculateTargetingEfficiency(
    campaignId: string
  ): Promise<TargetingMetrics> {
    const campaign = await db.query(
      'SELECT * FROM campaigns WHERE id = $1',
      [campaignId]
    );
    
    const metrics = await db.query(
      `SELECT 
        COUNT(*) as sent_count,
        COUNT(*) FILTER (WHERE opened) as open_count,
        COUNT(*) FILTER (WHERE clicked) as click_count,
        COUNT(*) FILTER (WHERE converted) as conversion_count
       FROM campaign_metrics
       WHERE campaign_id = $1`,
      [campaignId]
    );
    
    const m = metrics.rows[0];
    
    return {
      sentCount: parseInt(m.sent_count),
      openRate: m.sent_count > 0 ? m.open_count / m.sent_count : 0,
      clickRate: m.sent_count > 0 ? m.click_count / m.sent_count : 0,
      conversionRate: m.sent_count > 0 ? m.conversion_count / m.sent_count : 0,
      costPerConversion: campaign.rows[0]?.cost / (m.conversion_count || 1),
    };
  }
}

interface UserFilter {
  type: string;
  value: any;
}

interface TargetingMetrics {
  sentCount: number;
  openRate: number;
  clickRate: number;
  conversionRate: number;
  costPerConversion: number;
}
```

---

## 4. AWS Cost Breakdown และ Optimization

### 4.1 Typical LINE Bot AWS Architecture Costs

```
สถาปัตยกรรมทั่วไปสำหรับ LINE Bot ขนาดกลาง
(100,000 users, 50 req/sec average)

AWS Services Monthly Cost (USD):

1. Compute (EC2 / ECS)
   - Application: t3.medium x 2 = $67.20
   - Total Compute: ~$67

2. Database (RDS)
   - PostgreSQL: db.t3.medium Multi-AZ = $107.52
   - Read Replica: db.t3.medium = $53.76
   - Total DB: ~$161

3. Cache (ElastiCache)
   - Redis: cache.t3.micro = $24.48
   - Total Cache: ~$25

4. Load Balancer
   - ALB: $22.27 + $0.008/LCU
   - Total LB: ~$30

5. Storage
   - S3: 100GB = $2.30 + requests
   - EBS: 50GB gp3 = $4.00
   - Total Storage: ~$10

6. Data Transfer
   - Out to internet: 50GB = $4.50
   - Total Transfer: ~$5

7. Monitoring
   - CloudWatch: $5-20
   
Total Monthly: ~$300-400/month
```

### 4.2 AWS Cost Optimization Strategies

```bash
#!/bin/bash
# scripts/aws-cost-check.sh
# ตรวจสอบ AWS resources ที่ใช้งานน้อย

# ตรวจสอบ EC2 instances ที่มี CPU < 10%
aws ec2 describe-instances \
  --query 'Reservations[*].Instances[*].[InstanceId,InstanceType,State.Name]' \
  --output table

# ตรวจสอบ unused Elastic IPs
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==`null`].[PublicIp,AllocationId]'

# ตรวจสอบ unattached EBS volumes
aws ec2 describe-volumes \
  --filters 'Name=status,Values=available' \
  --query 'Volumes[*].[VolumeId,Size,CreateTime]'

# ตรวจสอบ old snapshots
aws ec2 describe-snapshots \
  --owner-ids self \
  --query 'Snapshots[?StartTime<=`2023-01-01`].[SnapshotId,VolumeSize,StartTime]'
```

```typescript
// optimization/aws-cost-optimizer.ts

export class AWSCostOptimizer {
  
  /**
   * ใช้ Spot Instances สำหรับ non-critical workloads
   */
  getSpotInstanceConfig(): SpotConfig {
    return {
      // สำหรับ consumer workers (สามารถ interrupt ได้)
      consumers: {
        instanceType: 't3.medium',
        spotPrice: '0.02',
        interruptionBehavior: 'terminate',
        onDemandFallback: true,
      },
      
      // สำหรับ batch processing
      batchJobs: {
        instanceType: 'c5.xlarge',
        spotPrice: '0.05',
        interruptionBehavior: 'stop',
      },
    };
  }

  /**
   * คำนวณ savings จาก Reserved Instances
   */
  calculateReservedInstanceSavings(
    instanceType: string,
    count: number
  ): ReservedInstanceSavings {
    // ราคาตัวอย่าง (USD/hour)
    const pricing: Record<string, { onDemand: number; reserved1yr: number; reserved3yr: number }> = {
      't3.medium': { onDemand: 0.0464, reserved1yr: 0.0278, reserved3yr: 0.0185 },
      't3.large': { onDemand: 0.0928, reserved1yr: 0.0556, reserved3yr: 0.037 },
      'r5.large': { onDemand: 0.126, reserved1yr: 0.0756, reserved3yr: 0.0504 },
    };

    const price = pricing[instanceType];
    if (!price) return { savings1yr: 0, savings3yr: 0 };

    const monthlyHours = 730;
    const onDemandMonthly = price.onDemand * monthlyHours * count;
    const reserved1yrMonthly = price.reserved1yr * monthlyHours * count;
    const reserved3yrMonthly = price.reserved3yr * monthlyHours * count;

    return {
      onDemandMonthly,
      reserved1yrMonthly,
      reserved3yrMonthly,
      savings1yr: onDemandMonthly - reserved1yrMonthly,
      savings1yrPercent: (1 - reserved1yrMonthly / onDemandMonthly) * 100,
      savings3yr: onDemandMonthly - reserved3yrMonthly,
      savings3yrPercent: (1 - reserved3yrMonthly / onDemandMonthly) * 100,
    };
  }

  /**
   * Auto-scaling เพื่อลดค่าใช้จ่ายในช่วง traffic ต่ำ
   */
  getAutoScalingPolicy(): AutoScalingPolicy {
    return {
      scaleOut: {
        metric: 'CPUUtilization',
        threshold: 70,
        cooldown: 300,
        adjustment: 2,
      },
      scaleIn: {
        metric: 'CPUUtilization',
        threshold: 30,
        cooldown: 600, // ลดช้ากว่า scale out
        adjustment: -1,
        protectFromScaleIn: {
          enabled: true,
          instances: 2, // ต้องมีอย่างน้อย 2 instances
        },
      },
      scheduledScaling: [
        // ลด capacity ตอนกลางคืน
        {
          name: 'scale-down-night',
          schedule: 'cron(0 22 * * ? *)', // 22:00 UTC = 05:00 Thai
          minCapacity: 1,
          maxCapacity: 2,
          desiredCapacity: 1,
        },
        // เพิ่ม capacity ช่วงเช้า
        {
          name: 'scale-up-morning',
          schedule: 'cron(0 1 * * ? *)', // 01:00 UTC = 08:00 Thai
          minCapacity: 2,
          maxCapacity: 10,
          desiredCapacity: 3,
        },
      ],
    };
  }
}

interface SpotConfig {
  consumers: any;
  batchJobs: any;
}

interface ReservedInstanceSavings {
  onDemandMonthly?: number;
  reserved1yrMonthly?: number;
  reserved3yrMonthly?: number;
  savings1yr: number;
  savings1yrPercent?: number;
  savings3yr: number;
  savings3yrPercent?: number;
}

interface AutoScalingPolicy {
  scaleOut: any;
  scaleIn: any;
  scheduledScaling: any[];
}
```

---

## 5. GCP Cost Breakdown

### 5.1 GCP Architecture สำหรับ LINE Bot

```
GCP Services Monthly Cost (USD)
สำหรับ LINE Bot ขนาดกลาง:

1. Cloud Run (แทน EC2)
   - 2 vCPU, 4GB RAM, 50 req/sec
   - ~$50-80/month (pay per request)
   - ประหยัดกว่า EC2 เพราะ scale to zero

2. Cloud SQL (PostgreSQL)
   - db-g1-small: $27.45/month
   - HA: $54.90/month
   
3. Cloud Memorystore (Redis)
   - 1GB: $45.76/month
   - Basic tier: $27.46/month

4. Cloud Storage
   - 100GB: $2.00/month

5. Load Balancing
   - ~$18-25/month

6. Firebase Hosting (LIFF)
   - Free tier ส่วนใหญ่

Total: ~$150-200/month
(ประหยัดกว่า AWS ~30-40%)
```

```yaml
# kubernetes/cloud-run-line-bot.yaml (GCP Cloud Run)
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
  name: line-bot
  namespace: default
spec:
  template:
    metadata:
      annotations:
        # ลด cold start
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "10"
        
        # Concurrency - รองรับ requests พร้อมกัน
        autoscaling.knative.dev/target: "80"
        
        # CPU ใช้เฉพาะเวลา processing
        run.googleapis.com/cpu-throttling: "true"
        
        # Scale to zero ตอน traffic ต่ำ
        autoscaling.knative.dev/scaleToZeroPodRetentionPeriod: "10m"
    spec:
      containerConcurrency: 100
      timeoutSeconds: 30
      containers:
      - image: gcr.io/project/line-bot:latest
        resources:
          limits:
            cpu: "2"
            memory: "512Mi"
          requests:
            cpu: "100m"
            memory: "128Mi"
        env:
        - name: NODE_ENV
          value: "production"
```

---

## 6. Database Cost Optimization

### 6.1 Connection Pooling

```typescript
// database/connection-pool.ts
import { Pool, PoolConfig } from 'pg';
import PgBouncer from 'pg-bouncer'; // หรือใช้ RDS Proxy

// ❌ แบบผิด - สร้าง connection ใหม่ทุกครั้ง
async function badExample() {
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });
  const result = await pool.query('SELECT 1');
  await pool.end(); // การปิด pool ทุกครั้งสิ้นเปลือง
}

// ✅ แบบถูก - ใช้ shared pool
const globalPool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,          // max connections
  min: 2,           // min idle connections
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Connection Pool Sizing Formula:
// max_connections = (number_of_workers * max_pool_size)
// ต้อง <= PostgreSQL max_connections (default 100)

// สำหรับ 3 pods, each with 5 pool size:
// total = 3 * 5 = 15 < 100 ✅
```

### 6.2 Query Optimization

```sql
-- ❌ N+1 Query problem (สิ้นเปลือง I/O)
SELECT * FROM orders WHERE user_id = $1;
-- สำหรับแต่ละ order:
SELECT * FROM order_items WHERE order_id = $2;

-- ✅ Optimized - 1 query
SELECT 
  o.*,
  json_agg(
    json_build_object(
      'id', oi.id,
      'productId', oi.product_id,
      'quantity', oi.quantity,
      'price', oi.unit_price
    )
  ) as items
FROM orders o
LEFT JOIN order_items oi ON o.id = oi.order_id
WHERE o.user_id = $1
GROUP BY o.id;

-- Partial Index - Index เฉพาะ rows ที่ query บ่อย
CREATE INDEX idx_orders_pending 
  ON orders (user_id, created_at DESC)
  WHERE status IN ('pending', 'confirmed');

-- Covering Index - Index รวม columns ที่ SELECT
CREATE INDEX idx_users_line_covering
  ON users (line_user_id)
  INCLUDE (display_name, loyalty_points, tier);
```

### 6.3 Data Archiving

```sql
-- Archive old data เพื่อลดขนาด table
-- สร้าง partition table สำหรับ orders
CREATE TABLE orders (
  id UUID NOT NULL,
  user_id UUID NOT NULL,
  status VARCHAR(50) NOT NULL,
  total_amount DECIMAL NOT NULL,
  created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- สร้าง partition ต่อเดือน
CREATE TABLE orders_2024_01 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Archive script (รันทุกเดือน)
-- ย้ายข้อมูลเก่ากว่า 1 ปีไปยัง cold storage
INSERT INTO orders_archive 
  SELECT * FROM orders 
  WHERE created_at < NOW() - INTERVAL '1 year';

DELETE FROM orders 
  WHERE created_at < NOW() - INTERVAL '1 year';
```

### 6.4 Read Replica Strategy

```typescript
// database/smart-routing.ts
export class DatabaseRouter {
  private writePool: Pool;
  private readPool: Pool;

  constructor(writeUrl: string, readUrl: string) {
    this.writePool = new Pool({ connectionString: writeUrl, max: 10 });
    this.readPool = new Pool({ connectionString: readUrl, max: 20 });
  }

  async query(sql: string, params?: any[], options?: { forceWrite?: boolean }) {
    const isWrite = this.isWriteQuery(sql) || options?.forceWrite;
    const pool = isWrite ? this.writePool : this.readPool;
    return pool.query(sql, params);
  }

  private isWriteQuery(sql: string): boolean {
    const normalized = sql.trim().toUpperCase();
    return normalized.startsWith('INSERT') ||
           normalized.startsWith('UPDATE') ||
           normalized.startsWith('DELETE') ||
           normalized.startsWith('CREATE') ||
           normalized.startsWith('DROP') ||
           normalized.startsWith('ALTER');
  }
}
```

---

## 7. CDN Optimization

### 7.1 CloudFront/Cloud CDN Setup

```typescript
// cdn/cdn-optimizer.ts

export class CDNOptimizer {
  
  /**
   * กำหนด Cache-Control headers
   */
  getCacheHeaders(resourceType: string): Record<string, string> {
    const strategies: Record<string, Record<string, string>> = {
      // Static assets - cache นาน
      'js-css': {
        'Cache-Control': 'public, max-age=31536000, immutable',
        'Vary': 'Accept-Encoding',
      },
      
      // Images - cache 1 เดือน
      'images': {
        'Cache-Control': 'public, max-age=2592000',
        'Vary': 'Accept-Encoding, Accept',
      },
      
      // API responses - cache สั้น
      'api-product': {
        'Cache-Control': 'public, max-age=300, stale-while-revalidate=60',
        'Vary': 'Accept, Accept-Language',
      },
      
      // User-specific - no cache
      'api-user': {
        'Cache-Control': 'private, no-cache, no-store',
        'Pragma': 'no-cache',
      },
      
      // Flex messages templates
      'flex-templates': {
        'Cache-Control': 'public, max-age=3600',
      },
    };

    return strategies[resourceType] || { 'Cache-Control': 'no-cache' };
  }

  /**
   * Optimize image delivery
   */
  getOptimizedImageUrl(originalUrl: string, options: ImageOptions): string {
    // ใช้ CloudFront + Lambda@Edge สำหรับ image optimization
    const params = new URLSearchParams();
    
    if (options.width) params.set('w', String(options.width));
    if (options.height) params.set('h', String(options.height));
    if (options.quality) params.set('q', String(options.quality));
    if (options.format) params.set('f', options.format);
    
    const cdnDomain = process.env.CDN_DOMAIN || 'cdn.yourapp.com';
    const path = new URL(originalUrl).pathname;
    
    return `https://${cdnDomain}${path}?${params.toString()}`;
  }

  /**
   * LINE Flex Message image optimization
   */
  optimizeFlexImages(flexContent: any): any {
    // เปลี่ยน image URLs ให้ผ่าน CDN
    const processContent = (content: any): any => {
      if (!content || typeof content !== 'object') return content;
      
      if (content.type === 'image' && content.url) {
        content.url = this.getOptimizedImageUrl(content.url, {
          width: 800,
          quality: 85,
          format: 'webp',
        });
      }
      
      if (Array.isArray(content.contents)) {
        content.contents = content.contents.map(processContent);
      }
      
      return content;
    };
    
    return processContent(flexContent);
  }
}

interface ImageOptions {
  width?: number;
  height?: number;
  quality?: number;
  format?: 'webp' | 'jpeg' | 'png';
}
```

---

## 8. FinOps Practices

### 8.1 Cost Allocation by Feature/Team

```typescript
// finops/cost-tagging.ts

// AWS Tags สำหรับ Cost Allocation
export const resourceTags = {
  // Resource tags
  getCommonTags(params: ResourceTagParams): Record<string, string> {
    return {
      'Environment': params.environment,
      'Service': params.service,
      'Team': params.team,
      'Feature': params.feature,
      'CostCenter': params.costCenter,
      'Project': 'line-bot',
      'ManagedBy': 'terraform',
    };
  },
};

// ตัวอย่าง Terraform resource tagging
/*
resource "aws_instance" "line_bot_app" {
  instance_type = "t3.medium"
  ami           = "ami-12345678"
  
  tags = merge(
    local.common_tags,
    {
      "Name"    = "line-bot-app"
      "Service" = "webhook-handler"
      "Team"    = "platform"
      "Feature" = "messaging"
    }
  )
}
*/

interface ResourceTagParams {
  environment: 'development' | 'staging' | 'production';
  service: string;
  team: string;
  feature: string;
  costCenter: string;
}
```

### 8.2 Cost Budget Alerts

```typescript
// finops/budget-manager.ts
export class CostBudgetManager {
  
  getBudgetConfig(): BudgetConfig {
    return {
      // Monthly budget
      monthly: {
        amount: 500, // USD
        alerts: [
          { threshold: 50, type: 'actual', action: 'notify' },
          { threshold: 80, type: 'actual', action: 'notify_escalate' },
          { threshold: 90, type: 'actual', action: 'auto_optimize' },
          { threshold: 100, type: 'forecasted', action: 'freeze_non_critical' },
        ],
      },
      
      // Per-service budgets
      services: {
        ec2: { monthly: 150, alertThreshold: 80 },
        rds: { monthly: 200, alertThreshold: 80 },
        elasticache: { monthly: 50, alertThreshold: 80 },
      },
    };
  }

  async checkBudgetStatus(): Promise<BudgetStatus[]> {
    // ดึงข้อมูลจาก AWS Cost Explorer
    const statuses: BudgetStatus[] = [];
    
    // Implementation ขึ้นอยู่กับ AWS SDK
    return statuses;
  }

  async autoOptimizeWhenOverBudget(): Promise<void> {
    const status = await this.checkBudgetStatus();
    
    for (const budget of status) {
      if (budget.utilizationPercent >= 90) {
        await this.applyCostSavingMeasures(budget.service);
      }
    }
  }

  private async applyCostSavingMeasures(service: string): Promise<void> {
    switch (service) {
      case 'ec2':
        // ลด desired capacity ของ Auto Scaling Group
        await this.reduceAutoScalingCapacity();
        break;
      case 'rds':
        // เปิด RDS auto pause (serverless)
        await this.enableRDSAutoPause();
        break;
    }
    
    logger.warn(`Applied cost saving measures for ${service}`);
  }

  private async reduceAutoScalingCapacity(): Promise<void> {
    // Implementation
  }

  private async enableRDSAutoPause(): Promise<void> {
    // Implementation
  }
}

interface BudgetConfig {
  monthly: any;
  services: Record<string, any>;
}

interface BudgetStatus {
  service: string;
  budgetAmount: number;
  actualSpend: number;
  forecastedSpend: number;
  utilizationPercent: number;
}
```

---

## 9. ROI Analysis Framework

### 9.1 LINE Bot ROI Calculator

```typescript
// analytics/roi-calculator.ts

export class ROICalculator {
  
  calculateLineOAROI(params: ROIParams): ROIAnalysis {
    const {
      monthlyInfrastructureCost,
      monthlyLinePlanCost,
      monthlyDevelopmentCost,
      monthlyRevenue,
      previousRevenue,
      customerSupportCostBefore,
      customerSupportCostAfter,
      conversionRateImprovement,
      averageOrderValue,
      newCustomersFromLINE,
    } = params;

    // คำนวณ Total Cost
    const totalMonthlyCost = 
      monthlyInfrastructureCost +
      monthlyLinePlanCost +
      monthlyDevelopmentCost;

    // คำนวณ Benefits
    const revenueIncrease = monthlyRevenue - previousRevenue;
    const customerSupportSavings = customerSupportCostBefore - customerSupportCostAfter;
    const newCustomerRevenue = newCustomersFromLINE * averageOrderValue;
    
    const totalMonthlyBenefit = 
      revenueIncrease +
      customerSupportSavings +
      newCustomerRevenue;

    // ROI Calculation
    const netMonthlyBenefit = totalMonthlyBenefit - totalMonthlyCost;
    const roi = (netMonthlyBenefit / totalMonthlyCost) * 100;
    const paybackPeriodMonths = totalMonthlyCost / Math.max(netMonthlyBenefit, 1);

    return {
      summary: {
        totalMonthlyCost,
        totalMonthlyBenefit,
        netMonthlyBenefit,
        roi: roi.toFixed(2) + '%',
        paybackPeriodMonths: Math.ceil(paybackPeriodMonths),
        isPositiveROI: netMonthlyBenefit > 0,
      },
      costBreakdown: {
        infrastructure: {
          amount: monthlyInfrastructureCost,
          percentage: (monthlyInfrastructureCost / totalMonthlyCost * 100).toFixed(1) + '%',
        },
        linePlan: {
          amount: monthlyLinePlanCost,
          percentage: (monthlyLinePlanCost / totalMonthlyCost * 100).toFixed(1) + '%',
        },
        development: {
          amount: monthlyDevelopmentCost,
          percentage: (monthlyDevelopmentCost / totalMonthlyCost * 100).toFixed(1) + '%',
        },
      },
      benefitBreakdown: {
        revenueIncrease,
        customerSupportSavings,
        newCustomerRevenue,
      },
      recommendations: this.generateRecommendations(params, roi),
    };
  }

  private generateRecommendations(params: ROIParams, roi: number): string[] {
    const recommendations: string[] = [];

    if (roi < 0) {
      recommendations.push('ROI ติดลบ - ควรทบทวนกลยุทธ์การใช้ LINE OA');
    }

    if (params.monthlyInfrastructureCost > params.totalMonthlyCost * 0.5) {
      recommendations.push('ค่า Infrastructure สูงเกิน 50% - พิจารณาใช้ Serverless');
    }

    if (params.conversionRateImprovement < 0.05) {
      recommendations.push('Conversion rate ต่ำ - ปรับปรุง user experience ใน LIFF');
    }

    if (params.newCustomersFromLINE < 100) {
      recommendations.push('ผู้ใช้ใหม่จาก LINE น้อย - เพิ่มงบ LINE Ads');
    }

    if (recommendations.length === 0) {
      recommendations.push('ROI ดี - ควรลงทุนเพิ่มในฟีเจอร์ใหม่');
    }

    return recommendations;
  }
}

interface ROIParams {
  monthlyInfrastructureCost: number;
  monthlyLinePlanCost: number;
  monthlyDevelopmentCost: number;
  totalMonthlyCost: number;
  monthlyRevenue: number;
  previousRevenue: number;
  customerSupportCostBefore: number;
  customerSupportCostAfter: number;
  conversionRateImprovement: number;
  averageOrderValue: number;
  newCustomersFromLINE: number;
}

interface ROIAnalysis {
  summary: any;
  costBreakdown: any;
  benefitBreakdown: any;
  recommendations: string[];
}

// ตัวอย่างการใช้งาน:
/*
const calculator = new ROICalculator();
const analysis = calculator.calculateLineOAROI({
  monthlyInfrastructureCost: 15000,  // ฿15,000
  monthlyLinePlanCost: 3000,         // ฿3,000 (Standard Plan)
  monthlyDevelopmentCost: 50000,     // ฿50,000 (developer salary)
  totalMonthlyCost: 68000,
  monthlyRevenue: 500000,            // ฿500,000
  previousRevenue: 400000,           // ฿400,000
  customerSupportCostBefore: 80000,  // ฿80,000
  customerSupportCostAfter: 30000,   // ฿30,000 (bot handles 60%)
  conversionRateImprovement: 0.15,   // 15% improvement
  averageOrderValue: 1500,           // ฿1,500
  newCustomersFromLINE: 200,         // 200 new customers
});

console.log(analysis);
// ROI: 170%
// Payback: < 1 month
*/
```

---

## 10. Cost Monitoring Dashboard

### 10.1 Prometheus Metrics สำหรับ Cost

```typescript
// monitoring/cost-metrics.ts
import { Gauge, Counter } from 'prom-client';

export class CostMetricsCollector {
  private lineQuotaUsage: Gauge;
  private lineQuotaRemaining: Gauge;
  private monthlyInfraSpend: Gauge;
  private pushMessagesCost: Counter;
  private broadcastMessagesCost: Counter;

  constructor() {
    this.lineQuotaUsage = new Gauge({
      name: 'line_api_quota_usage_total',
      help: 'Total LINE API quota used this month',
    });

    this.lineQuotaRemaining = new Gauge({
      name: 'line_api_quota_remaining',
      help: 'Remaining LINE API quota',
    });

    this.monthlyInfraSpend = new Gauge({
      name: 'infrastructure_monthly_spend_usd',
      help: 'Monthly infrastructure spend in USD',
      labelNames: ['service'],
    });

    this.pushMessagesCost = new Counter({
      name: 'line_push_messages_cost_total',
      help: 'Total cost of push messages in THB',
      labelNames: ['campaign'],
    });

    this.broadcastMessagesCost = new Counter({
      name: 'line_broadcast_cost_total',
      help: 'Total broadcast message cost',
    });
  }

  updateQuotaMetrics(usage: number, limit: number): void {
    this.lineQuotaUsage.set(usage);
    this.lineQuotaRemaining.set(limit - usage);
  }

  recordPushMessageCost(cost: number, campaign: string): void {
    this.pushMessagesCost.inc({ campaign }, cost);
  }

  updateInfraSpend(service: string, amount: number): void {
    this.monthlyInfraSpend.set({ service }, amount);
  }
}
```

### 10.2 Cost Dashboard HTML Template

```html
<!-- dashboard/cost-dashboard.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <title>LINE Bot Cost Dashboard</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    body { font-family: Arial, sans-serif; margin: 20px; background: #f5f5f5; }
    .grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; }
    .card { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
    .metric { font-size: 2em; font-weight: bold; color: #1a73e8; }
    .label { color: #666; font-size: 0.9em; }
    .alert { background: #fff3cd; border: 1px solid #ffc107; padding: 10px; border-radius: 4px; }
    .danger { background: #f8d7da; border: 1px solid #f5c6cb; }
  </style>
</head>
<body>
  <h1>💰 LINE Bot Cost Dashboard</h1>
  
  <div class="grid">
    <div class="card">
      <div class="label">LINE API Quota</div>
      <div class="metric" id="quotaUsed">-</div>
      <div class="label">จากทั้งหมด <span id="quotaLimit">-</span></div>
      <div id="quotaAlert"></div>
    </div>
    
    <div class="card">
      <div class="label">ค่าใช้จ่าย Infrastructure เดือนนี้</div>
      <div class="metric" id="infraCost">-</div>
      <div class="label">USD</div>
    </div>
    
    <div class="card">
      <div class="label">ROI เดือนนี้</div>
      <div class="metric" id="roi" style="color: green">-</div>
      <div class="label">%</div>
    </div>
  </div>

  <div style="margin-top: 20px;">
    <canvas id="costChart" height="100"></canvas>
  </div>

  <script>
    async function loadDashboard() {
      try {
        const response = await fetch('/api/metrics/cost');
        const data = await response.json();
        
        // Update metrics
        document.getElementById('quotaUsed').textContent = 
          data.quotaUsage.toLocaleString();
        document.getElementById('quotaLimit').textContent = 
          data.quotaLimit.toLocaleString();
        document.getElementById('infraCost').textContent = 
          '$' + data.infraCost.toFixed(2);
        document.getElementById('roi').textContent = 
          data.roi.toFixed(1) + '%';
        
        // Quota alert
        const utilizationRate = data.quotaUsage / data.quotaLimit;
        if (utilizationRate > 0.9) {
          document.getElementById('quotaAlert').innerHTML = 
            '<div class="alert danger">⚠️ Quota เกือบเต็ม!</div>';
        } else if (utilizationRate > 0.7) {
          document.getElementById('quotaAlert').innerHTML = 
            '<div class="alert">⚡ ใช้ไป 70% แล้ว</div>';
        }
        
        // Chart
        const ctx = document.getElementById('costChart').getContext('2d');
        new Chart(ctx, {
          type: 'line',
          data: {
            labels: data.dailyCosts.map(d => d.date),
            datasets: [{
              label: 'ค่าใช้จ่ายต่อวัน (USD)',
              data: data.dailyCosts.map(d => d.amount),
              borderColor: '#1a73e8',
              fill: true,
            }]
          },
          options: {
            responsive: true,
            scales: {
              y: { beginAtZero: true }
            }
          }
        });
      } catch (error) {
        console.error('Failed to load dashboard:', error);
      }
    }
    
    loadDashboard();
    setInterval(loadDashboard, 60000); // Refresh every minute
  </script>
</body>
</html>
```

---

## สรุป Cost Optimization Checklist

```
✅ LINE API Optimization
□ Monitor quota usage รายวัน
□ ใช้ Flex Messages แทน multiple text messages
□ Batch notifications ให้ผู้ใช้
□ Filter ผู้ใช้ที่ active เท่านั้น
□ Deduplication สำหรับ duplicate messages
□ ใช้ Narrowcast แทน Broadcast เมื่อทำได้

✅ Infrastructure Optimization
□ Right-size EC2/GCE instances
□ ใช้ Spot Instances สำหรับ batch jobs
□ Reserved Instances สำหรับ predictable workloads
□ Auto-scaling ตาม schedule (ลดช่วงกลางคืน)
□ ตรวจสอบ unused resources ทุกสัปดาห์
□ ใช้ Graviton/ARM instances (ถูกกว่า 20%)

✅ Database Optimization
□ Connection pooling ทุก service
□ Read replicas สำหรับ heavy reads
□ Data archiving สำหรับข้อมูลเก่า
□ Partition tables ต่อเดือน/ปี
□ Optimize slow queries
□ ใช้ Serverless Aurora สำหรับ dev/staging

✅ CDN & Storage
□ CloudFront/CDN สำหรับ static assets
□ Image optimization ก่อน serve
□ S3 Intelligent-Tiering สำหรับ old assets
□ Lifecycle policies สำหรับ logs

✅ FinOps
□ Tag ทุก resource ด้วย team/feature/env
□ Monthly budget alerts ตั้งไว้
□ Weekly cost review
□ ROI tracking per feature
□ Cost allocation report ต่อทีม

✅ Monitoring
□ Cost metrics ใน Prometheus
□ Dashboard สำหรับ stakeholders
□ Alert เมื่อ cost spike
□ Forecast month-end spend

คาดการณ์ประหยัดได้:
- Message deduplication: 10-15%
- Smart targeting: 20-30%
- Infrastructure right-sizing: 15-25%
- Spot instances: 60-70% สำหรับ batch
- Total savings: 30-50% ของค่าใช้จ่ายรวม
```

การบริหารต้นทุนที่ดีเริ่มจากการวัดค่าใช้จ่ายอย่างละเอียด จากนั้น optimize ทีละส่วน และ track ROI อย่างสม่ำเสมอ
