# ส่วนที่ 98: Cost Optimization สำหรับ LINE Bot Systems

---

## สารบัญ

1. [ภาพรวม Cost Optimization](#1-ภาพรวม-cost-optimization)
2. [LINE API Cost Analysis](#2-line-api-cost-analysis)
3. [Message Quota Optimization Strategies](#3-message-quota-optimization-strategies)
4. [Cloud Infrastructure Cost Reduction](#4-cloud-infrastructure-cost-reduction)
5. [Database Cost Optimization](#5-database-cost-optimization)
6. [CDN Cost Management](#6-cdn-cost-management)
7. [Monitoring Costs](#7-monitoring-costs)
8. [FinOps Practices](#8-finops-practices)
9. [Cost Allocation by Feature](#9-cost-allocation-by-feature)
10. [ROI Analysis Framework](#10-roi-analysis-framework)
11. [Complete Cost Dashboard](#11-complete-cost-dashboard)

---

## 1. ภาพรวม Cost Optimization

### 1.1 Cost Categories ของ LINE Bot System

```
LINE Bot System Total Cost Breakdown:

┌─────────────────────────────────────────────────────────────┐
│              Total Monthly Cost (ตัวอย่าง)                  │
│                                                             │
│  LINE Messaging API     ████████████  35%  ฿14,000         │
│  Cloud Compute (VMs)    ████████      25%  ฿10,000         │
│  Database (RDS/etc.)    ██████        20%  ฿8,000          │
│  CDN & Storage          ████          12%  ฿4,800          │
│  Monitoring & Logs      ██            5%   ฿2,000          │
│  Others                 █             3%   ฿1,200          │
│                                                             │
│  Total:                              100%  ฿40,000/month   │
└─────────────────────────────────────────────────────────────┘

หลังทำ Cost Optimization:
  LINE Messaging API     ████████      20%  ฿8,000   (-43%)
  Cloud Compute          ████          10%  ฿4,000   (-60%)
  Database               ████          11%  ฿4,400   (-45%)
  CDN & Storage          ███           8%   ฿3,200   (-33%)
  Monitoring             █             3%   ฿1,200   (-40%)
  Others                 █             3%   ฿1,200   (0%)

  New Total:                           55%  ฿22,000/month
  Monthly Savings:                     45%  ฿18,000/month
  Annual Savings:                           ฿216,000/year
```

### 1.2 Cost Optimization Framework

```
FrameWork: SMART Cost Optimization

S - Scrutinize (วิเคราะห์ค่าใช้จ่าย)
  ├── ดึงข้อมูล billing ทั้งหมด
  ├── จัดกลุ่มตาม service
  └── ระบุ cost drivers หลัก

M - Measure (วัดผลกระทบ)
  ├── คำนวณ cost per feature
  ├── วัด utilization rate
  └── หา waste/idle resources

A - Act (ดำเนินการ)
  ├── Right-size resources
  ├── Use Reserved/Spot instances
  └── Optimize code/queries

R - Review (ทบทวนผล)
  ├── เปรียบเทียบก่อน-หลัง
  ├── Monitor ต่อเนื่อง
  └── Fine-tune policies

T - Track (ติดตามต่อเนื่อง)
  ├── Budget alerts
  ├── Monthly reviews
  └── Anomaly detection
```

---

## 2. LINE API Cost Analysis

### 2.1 LINE Messaging API Pricing (2025)

```
LINE Messaging API Pricing Plans:

┌─────────────────────────────────────────────────────────────┐
│ Free Plan                                                    │
│  - Messages: 200 messages/month                             │
│  - Additional: ไม่สามารถซื้อเพิ่มได้                         │
│  - Features: Basic                                          │
│  - Price: ฿0/month                                          │
├─────────────────────────────────────────────────────────────┤
│ Light Plan                                                   │
│  - Messages: 5,000 messages/month                           │
│  - Additional: ฿2.20/message                               │
│  - Features: All                                            │
│  - Price: ฿399/month                                        │
├─────────────────────────────────────────────────────────────┤
│ Standard Plan                                               │
│  - Messages: 30,000 messages/month                          │
│  - Additional: ฿1.50/message                               │
│  - Features: All + Advanced API                             │
│  - Price: ฿1,199/month                                      │
└─────────────────────────────────────────────────────────────┘

หมายเหตุ: Reply messages ไม่นับ quota
```

### 2.2 Message Type Cost Analysis

```javascript
// src/cost/lineApiCostAnalyzer.js
class LINEApiCostAnalyzer {
  constructor(db, redis) {
    this.db = db;
    this.redis = redis;
    
    // ราคาต่อ message ตาม plan
    this.pricing = {
      light: { base: 399, included: 5000, additional: 2.20 },
      standard: { base: 1199, included: 30000, additional: 1.50 },
      premium: { base: 3999, included: 100000, additional: 1.20 },
    };
  }

  // คำนวณ cost จากการใช้งานจริง
  async calculateMonthlyApiCost(year, month) {
    const { rows } = await this.db.query(`
      SELECT 
        message_type,
        send_type,
        COUNT(*) as message_count,
        COUNT(DISTINCT user_id) as unique_recipients
      FROM bot_messages
      WHERE EXTRACT(YEAR FROM sent_at) = $1
        AND EXTRACT(MONTH FROM sent_at) = $2
      GROUP BY message_type, send_type
    `, [year, month]);

    const analysis = {
      byType: {},
      quotaUsage: 0,
      replyMessages: 0,
      pushMessages: 0,
      multicastMessages: 0,
      broadcastMessages: 0,
    };

    rows.forEach(row => {
      analysis.byType[`${row.message_type}_${row.send_type}`] = {
        count: row.message_count,
        uniqueRecipients: row.unique_recipients,
      };

      // Reply messages ไม่นับ quota
      if (row.send_type === 'reply') {
        analysis.replyMessages += row.message_count;
      } else {
        analysis.quotaUsage += row.message_count;
        
        if (row.send_type === 'push') analysis.pushMessages += row.message_count;
        if (row.send_type === 'multicast') analysis.multicastMessages += row.message_count;
        if (row.send_type === 'broadcast') analysis.broadcastMessages += row.message_count;
      }
    });

    // คำนวณ cost ตาม plan
    const costs = {};
    Object.entries(this.pricing).forEach(([plan, pricing]) => {
      const extraMessages = Math.max(0, analysis.quotaUsage - pricing.included);
      costs[plan] = {
        baseCost: pricing.base,
        extraMessageCost: extraMessages * pricing.additional,
        totalCost: pricing.base + (extraMessages * pricing.additional),
        extraMessages,
      };
    });

    // หา optimal plan
    const optimalPlan = Object.entries(costs).reduce((best, [plan, cost]) => {
      return cost.totalCost < best.totalCost ? { plan, ...cost } : best;
    }, { plan: 'none', totalCost: Infinity });

    return {
      quotaUsage: analysis.quotaUsage,
      replyMessages: analysis.replyMessages,
      breakdown: analysis.byType,
      costs,
      optimalPlan,
      estimatedSavings: costs.light?.totalCost - optimalPlan.totalCost || 0,
    };
  }

  // วิเคราะห์ broadcast efficiency
  async analyzeBroadcastEfficiency(campaignId) {
    const { rows } = await this.db.query(`
      SELECT 
        c.campaign_id,
        c.name,
        c.sent_count as total_sent,
        c.opened_count,
        c.clicked_count,
        c.converted_count,
        c.revenue_generated,
        COUNT(bm.message_id) as actual_messages_sent
      FROM campaigns c
      LEFT JOIN bot_messages bm ON c.campaign_id = bm.campaign_id
      WHERE c.campaign_id = $1
      GROUP BY c.campaign_id, c.name, c.sent_count, c.opened_count, c.clicked_count, c.converted_count, c.revenue_generated
    `, [campaignId]);

    if (rows.length === 0) return null;

    const campaign = rows[0];
    const messageCost = campaign.actual_messages_sent * 1.5; // Standard plan rate
    const revenuePerMessage = campaign.revenue_generated / campaign.actual_messages_sent;
    const roi = ((campaign.revenue_generated - messageCost) / messageCost) * 100;

    return {
      ...campaign,
      messageCost,
      revenuePerMessage,
      costPerOpen: campaign.opened_count > 0 ? messageCost / campaign.opened_count : null,
      costPerClick: campaign.clicked_count > 0 ? messageCost / campaign.clicked_count : null,
      costPerConversion: campaign.converted_count > 0 ? messageCost / campaign.converted_count : null,
      roi,
    };
  }
}

module.exports = LINEApiCostAnalyzer;
```

---

## 3. Message Quota Optimization Strategies

### 3.1 Smart Message Batching

```javascript
// src/cost/messageBatchOptimizer.js
class MessageBatchOptimizer {
  constructor(lineClient, redis) {
    this.lineClient = lineClient;
    this.redis = redis;
    this.batchQueue = new Map();
    this.batchTimeout = 5000; // 5 วินาที
    this.maxBatchSize = 500; // LINE multicast max
  }

  // เพิ่ม message เข้า queue
  async queueMessage(userId, message, priority = 'normal') {
    const key = `batch:${priority}`;
    
    await this.redis.rpush(key, JSON.stringify({ userId, message }));
    
    // Check ถ้า queue ใหญ่พอแล้ว
    const queueSize = await this.redis.llen(key);
    if (queueSize >= this.maxBatchSize) {
      await this.flushBatch(priority);
    }
  }

  // ส่ง batch
  async flushBatch(priority = 'normal') {
    const key = `batch:${priority}`;
    const batchSize = Math.min(await this.redis.llen(key), this.maxBatchSize);
    
    if (batchSize === 0) return;

    // ดึง messages จาก queue
    const messages = [];
    for (let i = 0; i < batchSize; i++) {
      const item = await this.redis.lpop(key);
      if (item) messages.push(JSON.parse(item));
    }

    // Group by message content (ใช้ multicast สำหรับ same message)
    const messageGroups = new Map();
    messages.forEach(({ userId, message }) => {
      const messageKey = JSON.stringify(message);
      if (!messageGroups.has(messageKey)) {
        messageGroups.set(messageKey, { message, userIds: [] });
      }
      messageGroups.get(messageKey).userIds.push(userId);
    });

    // ส่งด้วย multicast (ประหยัด quota กว่าส่งทีละคน)
    let sent = 0;
    for (const [, group] of messageGroups) {
      if (group.userIds.length === 1) {
        await this.lineClient.pushMessage(group.userIds[0], group.message);
        sent++;
      } else {
        // Multicast นับเป็น 1 message ต่อ user แต่ efficiency สูงกว่า
        const chunks = this.chunkArray(group.userIds, 500);
        for (const chunk of chunks) {
          await this.lineClient.multicast(chunk, [group.message]);
          sent += chunk.length;
        }
      }
    }

    return { sent, batches: messageGroups.size };
  }

  chunkArray(array, size) {
    const chunks = [];
    for (let i = 0; i < array.length; i += size) {
      chunks.push(array.slice(i, i + size));
    }
    return chunks;
  }

  // Schedule batch flush
  startBatchProcessor() {
    setInterval(async () => {
      await this.flushBatch('normal');
      await this.flushBatch('high');
    }, this.batchTimeout);
  }
}

module.exports = MessageBatchOptimizer;
```

### 3.2 Smart Segmentation สำหรับประหยัด Quota

```javascript
// src/cost/smartSegmentation.js
class SmartSegmentationService {
  constructor(db) {
    this.db = db;
  }

  // ส่งเฉพาะ users ที่มีโอกาส engage สูง
  async getHighEngagementUsers(campaignType, limit = null) {
    const query = `
      SELECT 
        u.user_id,
        u.display_name,
        -- Engagement score
        COALESCE(
          (
            -- Recent activity (40%)
            CASE 
              WHEN EXTRACT(DAYS FROM NOW() - MAX(e.timestamp)) <= 7 THEN 40
              WHEN EXTRACT(DAYS FROM NOW() - MAX(e.timestamp)) <= 14 THEN 25
              WHEN EXTRACT(DAYS FROM NOW() - MAX(e.timestamp)) <= 30 THEN 10
              ELSE 0
            END +
            -- Message open rate (30%)
            CASE 
              WHEN ROUND(SUM(CASE WHEN bm.is_opened THEN 1 ELSE 0 END)::NUMERIC / 
                        NULLIF(COUNT(bm.message_id), 0) * 100) >= 70 THEN 30
              WHEN ROUND(SUM(CASE WHEN bm.is_opened THEN 1 ELSE 0 END)::NUMERIC / 
                        NULLIF(COUNT(bm.message_id), 0) * 100) >= 40 THEN 20
              ELSE 5
            END +
            -- Purchase history (30%)
            CASE 
              WHEN COUNT(DISTINCT o.order_id) >= 3 THEN 30
              WHEN COUNT(DISTINCT o.order_id) >= 1 THEN 15
              ELSE 0
            END
          ), 0
        ) as engagement_score
      FROM line_users u
      LEFT JOIN line_events e ON u.user_id = e.user_id
      LEFT JOIN bot_messages bm ON u.user_id = bm.user_id 
                                AND bm.sent_at >= NOW() - INTERVAL '30 days'
      LEFT JOIN orders o ON u.user_id = o.user_id
      WHERE u.unfollowed_at IS NULL
        AND u.is_blocked = FALSE
      GROUP BY u.user_id, u.display_name
      HAVING COALESCE(
        CASE 
          WHEN EXTRACT(DAYS FROM NOW() - MAX(e.timestamp)) <= 30 THEN 1 ELSE 0
        END, 0
      ) = 1
      ORDER BY engagement_score DESC
      ${limit ? `LIMIT ${limit}` : ''}
    `;

    const { rows } = await this.db.query(query);
    return rows;
  }

  // คำนวณ optimal send time สำหรับแต่ละ user
  async getOptimalSendTime(userId) {
    const { rows } = await this.db.query(`
      SELECT 
        EXTRACT(HOUR FROM timestamp) as hour,
        EXTRACT(DOW FROM timestamp) as day_of_week,
        COUNT(*) as activity_count
      FROM line_events
      WHERE user_id = $1
        AND timestamp >= NOW() - INTERVAL '30 days'
      GROUP BY EXTRACT(HOUR FROM timestamp), EXTRACT(DOW FROM timestamp)
      ORDER BY activity_count DESC
      LIMIT 5
    `, [userId]);

    if (rows.length === 0) {
      return { hour: 10, dayOfWeek: 1, confidence: 'low' }; // Default: Monday 10am
    }

    return {
      hour: rows[0].hour,
      dayOfWeek: rows[0].day_of_week,
      confidence: rows[0].activity_count >= 5 ? 'high' : 'medium',
    };
  }

  // วิเคราะห์ unsubscribe risk ก่อนส่ง
  async assessUnsubscribeRisk(userId) {
    const { rows } = await this.db.query(`
      SELECT 
        COUNT(bm.message_id) as messages_received_30d,
        SUM(CASE WHEN bm.is_opened THEN 1 ELSE 0 END) as opened_30d,
        EXTRACT(DAYS FROM NOW() - u.followed_at) as account_age_days
      FROM line_users u
      LEFT JOIN bot_messages bm ON u.user_id = bm.user_id
                                AND bm.sent_at >= NOW() - INTERVAL '30 days'
      WHERE u.user_id = $1
      GROUP BY u.followed_at
    `, [userId]);

    if (rows.length === 0) return { risk: 'unknown' };

    const data = rows[0];
    const openRate = data.messages_received_30d > 0 
      ? data.opened_30d / data.messages_received_30d 
      : 0;

    // High frequency + low engagement = high unsubscribe risk
    if (data.messages_received_30d > 20 && openRate < 0.1) {
      return { risk: 'high', reason: 'High frequency, low engagement' };
    }
    if (data.messages_received_30d > 10 && openRate < 0.2) {
      return { risk: 'medium', reason: 'Above average frequency' };
    }

    return { risk: 'low' };
  }

  // คำนวณ quota savings จาก smart segmentation
  async calculateSegmentationSavings(campaignSize, targetEngagement = 0.5) {
    const totalFollowers = await this.db.query(`
      SELECT COUNT(*) as total FROM line_users WHERE unfollowed_at IS NULL
    `);
    const total = totalFollowers.rows[0].total;

    const targetAudience = Math.ceil(total * targetEngagement);
    const savedMessages = total - targetAudience;
    const savedCost = savedMessages * 1.5; // Standard plan rate

    return {
      totalFollowers: total,
      targetAudience,
      savedMessages,
      savedCost,
      costReductionPercent: Math.round(savedMessages / total * 100),
    };
  }
}

module.exports = SmartSegmentationService;
```

---

## 4. Cloud Infrastructure Cost Reduction

### 4.1 Right-sizing Analysis

```javascript
// src/cost/infrastructureCostOptimizer.js
const { CloudMonitoringClient } = require('@google-cloud/monitoring');
const { EC2 } = require('@aws-sdk/client-ec2');
const { CloudWatchClient } = require('@aws-sdk/client-cloudwatch');

class InfrastructureCostOptimizer {
  constructor(provider = 'aws') {
    this.provider = provider;
    
    if (provider === 'aws') {
      this.ec2 = new EC2({ region: process.env.AWS_REGION });
      this.cloudwatch = new CloudWatchClient({ region: process.env.AWS_REGION });
    }
  }

  // วิเคราะห์ CPU utilization
  async analyzeEC2Utilization(instanceId, days = 14) {
    const endTime = new Date();
    const startTime = new Date(Date.now() - days * 24 * 60 * 60 * 1000);

    const params = {
      MetricName: 'CPUUtilization',
      Namespace: 'AWS/EC2',
      Dimensions: [{ Name: 'InstanceId', Value: instanceId }],
      StartTime: startTime,
      EndTime: endTime,
      Period: 3600, // 1 hour
      Statistics: ['Average', 'Maximum', 'Minimum'],
    };

    const response = await this.cloudwatch.send(new GetMetricStatisticsCommand(params));
    const datapoints = response.Datapoints;

    const avgCPU = datapoints.reduce((sum, d) => sum + d.Average, 0) / datapoints.length;
    const maxCPU = Math.max(...datapoints.map(d => d.Maximum));

    // Recommendation based on utilization
    let recommendation = 'keep';
    let potentialSavings = 0;

    if (avgCPU < 5 && maxCPU < 20) {
      recommendation = 'terminate_or_downsize';
      potentialSavings = 90; // 90% savings
    } else if (avgCPU < 20 && maxCPU < 40) {
      recommendation = 'downsize';
      potentialSavings = 50; // 50% savings
    } else if (avgCPU < 40) {
      recommendation = 'consider_downsize';
      potentialSavings = 25;
    }

    return {
      instanceId,
      avgCPU: Math.round(avgCPU * 100) / 100,
      maxCPU: Math.round(maxCPU * 100) / 100,
      recommendation,
      potentialSavings,
    };
  }

  // สร้าง Auto Scaling Policy
  generateAutoScalingConfig() {
    return {
      // Scale up เมื่อ CPU > 70%
      scaleUpPolicy: {
        name: 'line-bot-scale-up',
        scalingAdjustment: 2,
        cooldown: 300,
        metricThreshold: 70,
        comparisonOperator: 'GreaterThanOrEqualToThreshold',
        evaluationPeriods: 2,
        period: 60,
        statistic: 'Average',
      },
      // Scale down เมื่อ CPU < 20%
      scaleDownPolicy: {
        name: 'line-bot-scale-down',
        scalingAdjustment: -1,
        cooldown: 600, // รอ 10 นาที ก่อน scale down
        metricThreshold: 20,
        comparisonOperator: 'LessThanOrEqualToThreshold',
        evaluationPeriods: 5, // ต้อง low 5 ช่วง ก่อน scale down
        period: 60,
        statistic: 'Average',
      },
      // Scheduled scaling สำหรับ peak hours
      scheduledActions: [
        {
          name: 'morning-peak',
          minSize: 3,
          maxSize: 10,
          desiredCapacity: 5,
          recurrence: '0 8 * * 1-5', // วันธรรมดา 8:00 น.
          timeZone: 'Asia/Bangkok',
        },
        {
          name: 'night-off-peak',
          minSize: 1,
          maxSize: 3,
          desiredCapacity: 1,
          recurrence: '0 23 * * *', // ทุกคืน 23:00 น.
          timeZone: 'Asia/Bangkok',
        },
        {
          name: 'weekend-peak',
          minSize: 2,
          maxSize: 8,
          desiredCapacity: 4,
          recurrence: '0 10 * * 6-7', // เสาร์-อาทิตย์ 10:00 น.
          timeZone: 'Asia/Bangkok',
        },
      ],
    };
  }

  // คำนวณ savings จาก Reserved Instances
  calculateReservedInstanceSavings(onDemandCost, months = 12) {
    const savings = {
      noUpfront1Year: {
        discount: 0.30,
        upfrontCost: 0,
        monthlyCost: onDemandCost * 0.70,
        totalCost: onDemandCost * 0.70 * 12,
        totalSavings: onDemandCost * 0.30 * 12,
      },
      partialUpfront1Year: {
        discount: 0.40,
        upfrontCost: onDemandCost * 0.20 * 12,
        monthlyCost: onDemandCost * 0.55,
        totalCost: onDemandCost * 0.20 * 12 + onDemandCost * 0.55 * 12,
        totalSavings: onDemandCost * 0.40 * 12 - onDemandCost * 0.05 * 12,
      },
      allUpfront1Year: {
        discount: 0.45,
        upfrontCost: onDemandCost * 0.55 * 12,
        monthlyCost: 0,
        totalCost: onDemandCost * 0.55 * 12,
        totalSavings: onDemandCost * 0.45 * 12,
      },
      allUpfront3Year: {
        discount: 0.60,
        upfrontCost: onDemandCost * 0.40 * 36,
        monthlyCost: 0,
        totalCost: onDemandCost * 0.40 * 36,
        totalSavings: onDemandCost * 0.60 * 36,
      },
    };

    return savings;
  }

  // Spot Instance strategy สำหรับ non-critical workloads
  getSpotInstanceStrategy() {
    return {
      recommendation: 'Use Spot Instances for batch jobs and analytics',
      configurations: [
        {
          useCase: 'Data Pipeline',
          spotSavings: '70-90%',
          implementation: `
            # AWS Batch with Spot Instances
            jobQueue:
              type: SPOT
              spotIamFleetRole: arn:aws:iam::xxx:role/AmazonEC2SpotFleetRole
              instanceTypes: [m5.large, m5.xlarge, m4.large]
              bidPercentage: 60  # 60% of on-demand price
          `,
        },
        {
          useCase: 'Analytics Processing',
          spotSavings: '60-80%',
          implementation: 'Use AWS EMR with Spot nodes for Spark jobs',
        },
        {
          useCase: 'Development/Testing',
          spotSavings: '70-90%',
          implementation: 'Run dev environments on Spot, handle interruptions',
        },
      ],
      handleInterruption: `
        // Graceful shutdown handler สำหรับ Spot interruption
        const handleSpotInterruption = async () => {
          // AWS gives 2-minute warning via instance metadata
          const response = await fetch('http://169.254.169.254/latest/meta-data/spot/instance-action');
          
          if (response.ok) {
            console.log('Spot interruption notice received');
            
            // Save state
            await saveCurrentState();
            
            // Drain current jobs
            await drainJobQueue();
            
            // Notify other services
            await notifyInterruption();
            
            console.log('Graceful shutdown complete');
          }
        };
        
        // Check every 5 seconds
        setInterval(handleSpotInterruption, 5000);
      `,
    };
  }
}

module.exports = InfrastructureCostOptimizer;
```

### 4.2 Kubernetes Cost Optimization

```yaml
# k8s/line-bot-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: line-bot
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: line-bot
  template:
    metadata:
      labels:
        app: line-bot
    spec:
      # ใช้ node selector เพื่อ schedule บน spot nodes
      nodeSelector:
        node.kubernetes.io/lifecycle: spot
      
      # Tolerate spot instance interruption
      tolerations:
      - key: "spot"
        operator: "Equal"
        value: "true"
        effect: "NoSchedule"
      
      containers:
      - name: line-bot
        image: your-registry/line-bot:latest
        resources:
          # ระบุ resource requests และ limits ให้แม่นยำ
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        
        # Liveness probe เพื่อ restart ถ้า unhealthy
        livenessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 30
          periodSeconds: 10
        
        # Readiness probe เพื่อไม่ส่ง traffic ถ้ายังไม่พร้อม
        readinessProbe:
          httpGet:
            path: /health
            port: 3000
          initialDelaySeconds: 5
          periodSeconds: 5

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: line-bot-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: line-bot
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 1
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 2
        periodSeconds: 30

---
# Vertical Pod Autoscaler (for right-sizing)
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: line-bot-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: line-bot
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: line-bot
      minAllowed:
        memory: "128Mi"
        cpu: "100m"
      maxAllowed:
        memory: "1Gi"
        cpu: "2"
```

---

## 5. Database Cost Optimization

### 5.1 Query Optimization

```sql
-- ตรวจสอบ slow queries
SELECT 
    query,
    calls,
    total_exec_time / 1000 as total_seconds,
    mean_exec_time / 1000 as avg_seconds,
    rows,
    100.0 * shared_blks_hit / nullif(shared_blks_hit + shared_blks_read, 0) AS hit_percent
FROM pg_stat_statements
WHERE mean_exec_time > 1000  -- queries ที่ใช้เวลา > 1 วินาที
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Index optimization
-- ตรวจสอบ unused indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as index_scans,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- indexes ที่ไม่ถูกใช้
  AND indexname NOT LIKE '%pkey%'
ORDER BY pg_relation_size(indexrelid::regclass) DESC;

-- สร้าง indexes ที่จำเป็น
-- 1. User ID + timestamp (พบบ่อยใน queries)
CREATE INDEX CONCURRENTLY idx_line_events_user_timestamp 
ON line_events(user_id, timestamp DESC);

-- 2. Event type + date (สำหรับ analytics)
CREATE INDEX CONCURRENTLY idx_line_events_type_date 
ON line_events(event_type, (timestamp::DATE));

-- 3. Order status + user
CREATE INDEX CONCURRENTLY idx_orders_user_status 
ON orders(user_id, status);

-- 4. JSONB index สำหรับ metadata
CREATE INDEX CONCURRENTLY idx_line_events_metadata 
ON line_events USING gin(metadata);

-- Partitioned tables สำหรับลด query cost
-- Automatic partition management
CREATE OR REPLACE FUNCTION create_monthly_partition()
RETURNS void AS $$
DECLARE
    partition_date DATE;
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    -- สร้าง partition สำหรับเดือนหน้า
    partition_date := DATE_TRUNC('month', NOW() + INTERVAL '1 month');
    partition_name := 'line_events_' || TO_CHAR(partition_date, 'YYYY_MM');
    start_date := partition_date;
    end_date := partition_date + INTERVAL '1 month';
    
    EXECUTE format('
        CREATE TABLE IF NOT EXISTS %I 
        PARTITION OF line_events 
        FOR VALUES FROM (%L) TO (%L)',
        partition_name, start_date, end_date
    );
    
    RAISE NOTICE 'Created partition: %', partition_name;
END;
$$ LANGUAGE plpgsql;

-- Schedule สร้าง partition ทุกเดือน
SELECT cron.schedule('create-partition', '0 0 1 * *', 'SELECT create_monthly_partition()');

-- Archive old partitions
CREATE OR REPLACE FUNCTION archive_old_partitions(months_to_keep INTEGER DEFAULT 12)
RETURNS void AS $$
DECLARE
    cutoff_date DATE;
    partition_name TEXT;
BEGIN
    cutoff_date := DATE_TRUNC('month', NOW() - (months_to_keep || ' months')::INTERVAL);
    
    FOR partition_name IN 
        SELECT tablename 
        FROM pg_tables 
        WHERE tablename LIKE 'line_events_%'
          AND tablename < 'line_events_' || TO_CHAR(cutoff_date, 'YYYY_MM')
    LOOP
        -- ย้ายไป archive storage หรือ delete
        EXECUTE format('DROP TABLE IF EXISTS %I', partition_name);
        RAISE NOTICE 'Archived partition: %', partition_name;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

### 5.2 Connection Pool Optimization

```javascript
// src/database/connectionPool.js
const { Pool } = require('pg');
const config = require('../config');

// Optimized pool configuration
const pool = new Pool({
  connectionString: config.database.url,
  
  // จำนวน connections ที่เหมาะสม
  // Rule of thumb: min(2 * CPUs, 10) for web servers
  min: parseInt(process.env.DB_POOL_MIN || 2),
  max: parseInt(process.env.DB_POOL_MAX || 10),
  
  // Connection timeout
  connectionTimeoutMillis: 5000,
  
  // Idle connection timeout (ปิด connection ที่ไม่ใช้)
  idleTimeoutMillis: 30000,
  
  // Statement timeout (ป้องกัน runaway queries)
  statement_timeout: 30000,
});

// Monitor pool health
pool.on('connect', (client) => {
  // Set session-level optimizations
  client.query('SET statement_timeout = 30000');
  client.query('SET lock_timeout = 5000');
});

pool.on('error', (err) => {
  console.error('Pool error:', err);
});

// Pool metrics
const getPoolMetrics = () => ({
  totalCount: pool.totalCount,
  idleCount: pool.idleCount,
  waitingCount: pool.waitingCount,
});

module.exports = { pool, getPoolMetrics };
```

### 5.3 Caching Strategy

```javascript
// src/database/cacheManager.js
const Redis = require('ioredis');
const config = require('../config');

class DatabaseCacheManager {
  constructor() {
    this.redis = new Redis(config.redis);
    this.defaultTTL = 300; // 5 นาที
  }

  // Cache query result
  async cacheQuery(key, queryFn, ttl = this.defaultTTL) {
    // Try cache first
    const cached = await this.redis.get(key);
    if (cached) {
      return JSON.parse(cached);
    }

    // Execute query
    const result = await queryFn();

    // Cache result
    await this.redis.set(key, JSON.stringify(result), 'EX', ttl);

    return result;
  }

  // Cache with tags สำหรับ bulk invalidation
  async cacheWithTags(key, queryFn, tags = [], ttl = this.defaultTTL) {
    const cached = await this.redis.get(key);
    if (cached) {
      return JSON.parse(cached);
    }

    const result = await queryFn();

    const pipeline = this.redis.pipeline();
    pipeline.set(key, JSON.stringify(result), 'EX', ttl);
    
    // เพิ่ม key ใน tag sets
    for (const tag of tags) {
      pipeline.sadd(`tag:${tag}`, key);
      pipeline.expire(`tag:${tag}`, ttl * 2);
    }
    
    await pipeline.exec();

    return result;
  }

  // Invalidate cache by tags
  async invalidateByTags(tags) {
    for (const tag of tags) {
      const keys = await this.redis.smembers(`tag:${tag}`);
      
      if (keys.length > 0) {
        const pipeline = this.redis.pipeline();
        keys.forEach(key => pipeline.del(key));
        pipeline.del(`tag:${tag}`);
        await pipeline.exec();
      }
    }
  }

  // ใช้งานจริง
  async getUserProfile(userId) {
    return this.cacheWithTags(
      `user:profile:${userId}`,
      () => db.query('SELECT * FROM line_users WHERE user_id = $1', [userId]),
      [`user:${userId}`, 'users'],
      3600 // 1 ชั่วโมง
    );
  }

  async getOrderHistory(userId) {
    return this.cacheWithTags(
      `orders:user:${userId}`,
      () => db.query('SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT 20', [userId]),
      [`user:${userId}`, `orders:user:${userId}`],
      300 // 5 นาที
    );
  }

  // วิเคราะห์ cache efficiency
  async getCacheStats() {
    const info = await this.redis.info('stats');
    const hits = parseInt(info.match(/keyspace_hits:(\d+)/)?.[1] || 0);
    const misses = parseInt(info.match(/keyspace_misses:(\d+)/)?.[1] || 0);
    const total = hits + misses;
    
    return {
      hits,
      misses,
      total,
      hitRate: total > 0 ? Math.round(hits / total * 100) : 0,
    };
  }
}

module.exports = new DatabaseCacheManager();
```

---

## 6. CDN Cost Management

### 6.1 Asset Optimization Strategy

```javascript
// src/cost/cdnOptimizer.js
const sharp = require('sharp');
const path = require('path');
const AWS = require('@aws-sdk/client-s3');
const { CloudFront } = require('@aws-sdk/client-cloudfront');

class CDNOptimizer {
  constructor() {
    this.s3 = new AWS.S3Client({ region: process.env.AWS_REGION });
    this.cloudfront = new CloudFront({ region: process.env.AWS_REGION });
  }

  // Optimize images ก่อน upload
  async optimizeAndUploadImage(imageBuffer, filename, options = {}) {
    const {
      maxWidth = 1200,
      maxHeight = 1200,
      quality = 80,
      format = 'webp',
    } = options;

    // Optimize ด้วย sharp
    const optimized = await sharp(imageBuffer)
      .resize(maxWidth, maxHeight, {
        fit: 'inside',
        withoutEnlargement: true,
      })
      .webp({ quality })
      .toBuffer();

    const originalSize = imageBuffer.length;
    const optimizedSize = optimized.length;
    const savings = Math.round((1 - optimizedSize / originalSize) * 100);

    // Upload to S3
    const key = `images/${Date.now()}-${path.basename(filename, path.extname(filename))}.webp`;
    
    await this.s3.send(new AWS.PutObjectCommand({
      Bucket: process.env.S3_BUCKET,
      Key: key,
      Body: optimized,
      ContentType: 'image/webp',
      CacheControl: 'max-age=31536000', // 1 ปี
      Metadata: {
        originalSize: originalSize.toString(),
        optimizedSize: optimizedSize.toString(),
        savings: `${savings}%`,
      },
    }));

    const url = `https://${process.env.CLOUDFRONT_DOMAIN}/${key}`;

    return {
      url,
      originalSize,
      optimizedSize,
      savings,
    };
  }

  // ตั้งค่า CloudFront cache behaviors
  getCacheBehaviors() {
    return [
      {
        PathPattern: '/images/*',
        CachePolicyId: 'Managed-CachingOptimized',
        TTL: 31536000, // 1 ปี สำหรับ images
        Compress: true,
      },
      {
        PathPattern: '/static/*',
        CachePolicyId: 'Managed-CachingOptimized',
        TTL: 2592000, // 30 วัน สำหรับ static assets
        Compress: true,
      },
      {
        PathPattern: '/api/*',
        CachePolicyId: 'Managed-CachingDisabled',
        TTL: 0, // ไม่ cache API responses
      },
      {
        PathPattern: '/liff/*',
        CachePolicyId: 'Managed-CachingOptimized',
        TTL: 3600, // 1 ชั่วโมง สำหรับ LIFF pages
        Compress: true,
      },
    ];
  }

  // วิเคราะห์ CloudFront cost
  async analyzeCloudFrontCost(distributionId) {
    // ดึง usage metrics จาก CloudWatch
    const metrics = await this.getDistributionMetrics(distributionId);
    
    // CloudFront pricing (approximate)
    const pricing = {
      httpRequests: 0.0075 / 10000, // per 10,000 requests
      dataTransfer: {
        first10TB: 0.085, // per GB
        next40TB: 0.080,
        next100TB: 0.060,
        next350TB: 0.040,
        over500TB: 0.030,
      },
    };

    const requestCost = metrics.requests * pricing.httpRequests;
    
    let transferCost = 0;
    const gbTransferred = metrics.bytesDownloaded / (1024 * 1024 * 1024);
    
    if (gbTransferred <= 10240) {
      transferCost = gbTransferred * pricing.dataTransfer.first10TB;
    } else if (gbTransferred <= 51200) {
      transferCost = 10240 * pricing.dataTransfer.first10TB + 
                     (gbTransferred - 10240) * pricing.dataTransfer.next40TB;
    }

    return {
      requests: metrics.requests,
      gbTransferred,
      requestCost,
      transferCost,
      totalCost: requestCost + transferCost,
      cacheHitRate: metrics.cacheHitRate,
      estimatedSavingsFromCaching: transferCost * (metrics.cacheHitRate / 100),
    };
  }

  async getDistributionMetrics(distributionId) {
    // Simplified - ในจริงควรดึงจาก CloudWatch
    return {
      requests: 1000000,
      bytesDownloaded: 50 * 1024 * 1024 * 1024, // 50GB
      cacheHitRate: 85,
    };
  }
}

module.exports = CDNOptimizer;
```

---

## 7. Monitoring Costs

### 7.1 Cost-effective Monitoring Setup

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 60s  # เพิ่มจาก default 15s เพื่อลด storage
  evaluation_interval: 60s

# Rules สำหรับ alerting
rule_files:
  - "cost_alerts.yml"
  - "performance_alerts.yml"

scrape_configs:
  - job_name: 'line-bot'
    scrape_interval: 30s  # ลดความถี่
    static_configs:
      - targets: ['line-bot:3000']
    
  - job_name: 'node-exporter'
    scrape_interval: 60s
    static_configs:
      - targets: ['node-exporter:9100']

# Retention settings (ลด storage cost)
storage:
  tsdb:
    retention.time: 30d   # เก็บแค่ 30 วัน (default 15d)
    retention.size: 10GB  # จำกัด size
```

```yaml
# prometheus/cost_alerts.yml
groups:
  - name: cost_alerts
    rules:
    
    # แจ้งเตือนเมื่อ message quota ใกล้หมด
    - alert: HighMessageQuotaUsage
      expr: line_bot_messages_sent_today / line_bot_monthly_quota > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "LINE message quota at {{ $value | humanizePercentage }}"
        description: "Consider optimizing message sending to avoid extra charges"

    # แจ้งเตือนเมื่อ infrastructure cost สูง
    - alert: HighInfrastructureCost
      expr: aws_estimated_charges > 50000  # 50,000 บาท
      for: 1h
      labels:
        severity: critical
      annotations:
        summary: "AWS cost exceeded ฿50,000 this month"
        
    # แจ้งเตือนเมื่อ DB connections สูง
    - alert: HighDatabaseConnections
      expr: pg_stat_activity_count > 80
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High database connection count: {{ $value }}"
```

### 7.2 Log Cost Optimization

```javascript
// src/logging/costOptimizedLogger.js
const winston = require('winston');
const { ElasticsearchTransport } = require('winston-elasticsearch');

class CostOptimizedLogger {
  constructor() {
    // Log levels ที่ส่งไป external storage
    const externalLogLevels = process.env.NODE_ENV === 'production' 
      ? ['error', 'warn']  // Production: เก็บแค่ error และ warn
      : ['error', 'warn', 'info', 'debug'];

    this.logger = winston.createLogger({
      level: process.env.LOG_LEVEL || 'info',
      format: winston.format.combine(
        winston.format.timestamp(),
        winston.format.json(),
      ),
      transports: [
        // Console (local - ไม่มีค่าใช้จ่าย)
        new winston.transports.Console({
          format: winston.format.simple(),
        }),
        
        // File transport (local - ประหยัดกว่า cloud)
        new winston.transports.File({
          filename: 'logs/error.log',
          level: 'error',
          maxsize: 10 * 1024 * 1024, // 10MB
          maxFiles: 5,
          tailable: true,
        }),
        
        // Elasticsearch สำหรับ production logs (เฉพาะ important logs)
        ...(process.env.ELASTICSEARCH_URL ? [
          new ElasticsearchTransport({
            level: 'warn',
            clientOpts: { node: process.env.ELASTICSEARCH_URL },
            index: 'line-bot-logs',
            // ILM policy สำหรับ lifecycle
            dataStream: true,
          })
        ] : []),
      ],
    });
  }

  // Sampling สำหรับ high-volume logs
  infoSampled(message, meta = {}, sampleRate = 0.1) {
    if (Math.random() < sampleRate) {
      this.logger.info(message, { ...meta, sampled: true, sampleRate });
    }
  }

  error(message, meta = {}) { this.logger.error(message, meta); }
  warn(message, meta = {}) { this.logger.warn(message, meta); }
  info(message, meta = {}) { this.logger.info(message, meta); }
  debug(message, meta = {}) { this.logger.debug(message, meta); }
}

module.exports = new CostOptimizedLogger();
```

---

## 8. FinOps Practices

### 8.1 Cost Tagging Strategy

```javascript
// src/cost/costTagManager.js
class CostTagManager {
  constructor() {
    this.mandatoryTags = ['Project', 'Environment', 'Team', 'Feature', 'CostCenter'];
  }

  // สร้าง tag set สำหรับ resources
  generateResourceTags(config) {
    const tags = {
      Project: 'LINE-OA-Bot',
      Environment: config.environment || 'production',
      Team: config.team || 'backend',
      Feature: config.feature || 'general',
      CostCenter: config.costCenter || 'IT-001',
      ManagedBy: 'terraform',
      CreatedAt: new Date().toISOString().split('T')[0],
      Owner: config.owner || 'team@company.com',
    };

    // ตรวจสอบว่ามี mandatory tags ครบ
    const missingTags = this.mandatoryTags.filter(t => !tags[t]);
    if (missingTags.length > 0) {
      throw new Error(`Missing mandatory tags: ${missingTags.join(', ')}`);
    }

    return tags;
  }

  // วิเคราะห์ cost by tag
  async analyzeCostByTag(tagKey, tagValue, startDate, endDate) {
    // AWS Cost Explorer API
    const params = {
      TimePeriod: { Start: startDate, End: endDate },
      Granularity: 'MONTHLY',
      Filter: {
        Tags: {
          Key: tagKey,
          Values: [tagValue],
        },
      },
      GroupBy: [{ Type: 'DIMENSION', Key: 'SERVICE' }],
      Metrics: ['BlendedCost'],
    };

    // ในจริงใช้ AWS Cost Explorer SDK
    // const response = await costExplorer.getCostAndUsage(params);
    
    // Simplified return
    return {
      tagKey,
      tagValue,
      period: { start: startDate, end: endDate },
      services: [
        { name: 'EC2', cost: 8500 },
        { name: 'RDS', cost: 4200 },
        { name: 'ElastiCache', cost: 1800 },
      ],
      total: 14500,
    };
  }
}

module.exports = new CostTagManager();
```

### 8.2 Budget Management

```javascript
// src/cost/budgetManager.js
class BudgetManager {
  constructor(db, notificationService) {
    this.db = db;
    this.notificationService = notificationService;
    
    // Budget thresholds
    this.budgets = {
      lineApi: {
        monthly: 15000, // ฿15,000/เดือน
        alerts: [50, 75, 90, 100],
      },
      infrastructure: {
        monthly: 20000,
        alerts: [60, 80, 95, 100],
      },
      database: {
        monthly: 8000,
        alerts: [70, 90, 100],
      },
      total: {
        monthly: 50000,
        alerts: [50, 70, 85, 100],
      },
    };
  }

  // ตรวจสอบ budget ทุกวัน
  async checkBudgets() {
    const currentMonth = new Date().getMonth() + 1;
    const currentYear = new Date().getFullYear();
    const daysInMonth = new Date(currentYear, currentMonth, 0).getDate();
    const currentDay = new Date().getDate();
    const monthProgress = currentDay / daysInMonth;

    const currentSpending = await this.getCurrentMonthSpending();

    const alerts = [];

    for (const [category, budget] of Object.entries(this.budgets)) {
      const spent = currentSpending[category] || 0;
      const spentPercent = Math.round(spent / budget.monthly * 100);
      const projectedMonthly = Math.round(spent / monthProgress);
      const projectedPercent = Math.round(projectedMonthly / budget.monthly * 100);

      for (const threshold of budget.alerts) {
        const alertKey = `budget:alert:${category}:${threshold}:${currentYear}-${currentMonth}`;
        const alreadyAlerted = await this.redis.exists(alertKey);

        if (!alreadyAlerted) {
          if (spentPercent >= threshold || projectedPercent >= threshold) {
            alerts.push({
              category,
              threshold,
              currentSpending: spent,
              budget: budget.monthly,
              spentPercent,
              projectedMonthly,
              projectedPercent,
            });

            // Mark as alerted
            await this.redis.set(alertKey, '1', 'EX', 86400 * 35);
          }
        }
      }
    }

    if (alerts.length > 0) {
      await this.sendBudgetAlerts(alerts);
    }

    return alerts;
  }

  async getCurrentMonthSpending() {
    // ดึงข้อมูลจาก billing API หรือ database
    const { rows } = await this.db.query(`
      SELECT 
        category,
        SUM(amount) as total
      FROM cost_records
      WHERE EXTRACT(YEAR FROM recorded_at) = EXTRACT(YEAR FROM NOW())
        AND EXTRACT(MONTH FROM recorded_at) = EXTRACT(MONTH FROM NOW())
      GROUP BY category
    `);

    const spending = {};
    rows.forEach(row => {
      spending[row.category] = parseFloat(row.total);
    });

    spending.total = Object.values(spending).reduce((sum, v) => sum + v, 0);

    return spending;
  }

  async sendBudgetAlerts(alerts) {
    for (const alert of alerts) {
      const isOverBudget = alert.spentPercent >= 100;
      const message = isOverBudget
        ? `🚨 Budget Exceeded: ${alert.category}\nใช้ไปแล้ว ฿${alert.currentSpending.toLocaleString()} / ฿${alert.budget.toLocaleString()} (${alert.spentPercent}%)`
        : `⚠️ Budget Alert: ${alert.category}\nถึง ${alert.threshold}% แล้ว\nใช้ไปแล้ว ฿${alert.currentSpending.toLocaleString()}\nประมาณการทั้งเดือน: ฿${alert.projectedMonthly.toLocaleString()}`;

      // ส่งแจ้งเตือน
      await this.notificationService.sendToAdmin(message);
    }
  }
}

module.exports = BudgetManager;
```

---

## 9. Cost Allocation by Feature

### 9.1 Feature Cost Tracking

```javascript
// src/cost/featureCostTracker.js
class FeatureCostTracker {
  constructor(db) {
    this.db = db;
    
    // Feature definitions
    this.features = {
      CHAT_SUPPORT: 'chat_support',
      ORDER_TRACKING: 'order_tracking',
      BROADCAST: 'broadcast',
      LIFF_PAGES: 'liff_pages',
      AI_CHATBOT: 'ai_chatbot',
      ANALYTICS: 'analytics',
      WEBHOOK_PROCESSING: 'webhook_processing',
    };
  }

  // บันทึก API call cost ตาม feature
  async trackApiCost(feature, apiType, count = 1, cost = null) {
    const apiCosts = {
      push_message: 1.5,
      multicast_message: 1.5,
      broadcast_message: 1.5,
      reply_message: 0,
      get_profile: 0,
      get_group_info: 0,
    };

    const actualCost = cost ?? (apiCosts[apiType] || 0) * count;

    await this.db.query(`
      INSERT INTO feature_costs (feature, api_type, call_count, cost, recorded_at)
      VALUES ($1, $2, $3, $4, NOW())
      ON CONFLICT (feature, api_type, DATE(recorded_at))
      DO UPDATE SET 
        call_count = feature_costs.call_count + $3,
        cost = feature_costs.cost + $4
    `, [feature, apiType, count, actualCost]);
  }

  // ดึง cost breakdown by feature
  async getFeatureCostBreakdown(startDate, endDate) {
    const { rows } = await this.db.query(`
      SELECT 
        feature,
        SUM(call_count) as total_calls,
        SUM(cost) as total_cost,
        SUM(cost) / NULLIF(SUM(call_count), 0) as cost_per_call
      FROM feature_costs
      WHERE recorded_at BETWEEN $1 AND $2
      GROUP BY feature
      ORDER BY total_cost DESC
    `, [startDate, endDate]);

    const total = rows.reduce((sum, r) => sum + parseFloat(r.total_cost), 0);

    return rows.map(row => ({
      ...row,
      costPercent: Math.round(parseFloat(row.total_cost) / total * 100),
      total,
    }));
  }

  // คำนวณ cost per user per feature
  async getCostPerUser(feature, period = 'monthly') {
    const interval = period === 'monthly' ? '1 month' : '1 week';
    
    const { rows } = await this.db.query(`
      SELECT 
        fc.feature,
        SUM(fc.cost) as total_cost,
        COUNT(DISTINCT lu.user_id) as active_users,
        SUM(fc.cost) / NULLIF(COUNT(DISTINCT lu.user_id), 0) as cost_per_user
      FROM feature_costs fc
      CROSS JOIN (
        SELECT DISTINCT user_id 
        FROM line_events 
        WHERE timestamp >= NOW() - INTERVAL '${interval}'
      ) lu
      WHERE fc.feature = $1
        AND fc.recorded_at >= NOW() - INTERVAL '${interval}'
      GROUP BY fc.feature
    `, [feature]);

    return rows[0];
  }
}

module.exports = FeatureCostTracker;
```

---

## 10. ROI Analysis Framework

### 10.1 ROI Calculator

```javascript
// src/cost/roiCalculator.js
class ROICalculator {
  constructor(db) {
    this.db = db;
  }

  // คำนวณ ROI ของ LINE OA
  async calculateLINEOAROI(startDate, endDate) {
    // Costs
    const costs = await this.getTotalCosts(startDate, endDate);
    
    // Revenue
    const revenue = await this.getTotalRevenue(startDate, endDate);
    
    // Intangible benefits (ประมาณการ)
    const savedCustomerServiceCost = await this.calculateSavedCSCost(startDate, endDate);
    const increasedRetentionValue = await this.calculateRetentionValue(startDate, endDate);
    
    const totalBenefits = revenue.direct + savedCustomerServiceCost + increasedRetentionValue;
    const netBenefits = totalBenefits - costs.total;
    const roi = costs.total > 0 ? (netBenefits / costs.total) * 100 : 0;
    const paybackMonths = costs.total > 0 
      ? Math.ceil(costs.total / (totalBenefits / this.getMonthCount(startDate, endDate)))
      : 0;

    return {
      period: { startDate, endDate },
      costs: {
        ...costs,
        breakdown: costs.breakdown,
      },
      benefits: {
        directRevenue: revenue.direct,
        savedCSCost: savedCustomerServiceCost,
        retentionValue: increasedRetentionValue,
        total: totalBenefits,
      },
      roi: Math.round(roi * 100) / 100,
      netBenefits,
      paybackPeriodMonths: paybackMonths,
      isPositiveROI: roi > 0,
    };
  }

  async getTotalCosts(startDate, endDate) {
    const { rows } = await this.db.query(`
      SELECT 
        category,
        SUM(amount) as total
      FROM cost_records
      WHERE recorded_at BETWEEN $1 AND $2
      GROUP BY category
    `, [startDate, endDate]);

    const breakdown = {};
    rows.forEach(r => { breakdown[r.category] = parseFloat(r.total); });
    const total = Object.values(breakdown).reduce((sum, v) => sum + v, 0);

    return { total, breakdown };
  }

  async getTotalRevenue(startDate, endDate) {
    const { rows } = await this.db.query(`
      SELECT 
        SUM(o.total) as direct_revenue,
        SUM(o.total * 0.3) as attributed_revenue  -- 30% margin
      FROM orders o
      WHERE o.status = 'completed'
        AND o.created_at BETWEEN $1 AND $2
        AND o.source IN ('direct_message', 'broadcast', 'liff', 'rich_menu')
    `, [startDate, endDate]);

    return {
      direct: parseFloat(rows[0].direct_revenue || 0),
      attributed: parseFloat(rows[0].attributed_revenue || 0),
    };
  }

  async calculateSavedCSCost(startDate, endDate) {
    const { rows } = await this.db.query(`
      SELECT COUNT(*) as automated_resolutions
      FROM user_sessions
      WHERE start_time BETWEEN $1 AND $2
        AND outcome = 'resolved'
        AND agent_id IS NULL  -- Resolved by bot, no human agent
    `, [startDate, endDate]);

    const costPerAgentResolution = 50; // ฿50 ต่อการแก้ปัญหาโดย agent
    return parseInt(rows[0].automated_resolutions) * costPerAgentResolution;
  }

  async calculateRetentionValue(startDate, endDate) {
    const { rows } = await this.db.query(`
      WITH retention_impact AS (
        SELECT 
          COUNT(DISTINCT u.user_id) as retained_users,
          AVG(COALESCE(o_total.spending, 0)) as avg_user_value
        FROM line_users u
        LEFT JOIN (
          SELECT user_id, SUM(total) as spending
          FROM orders
          WHERE status = 'completed'
          GROUP BY user_id
        ) o_total ON u.user_id = o_total.user_id
        WHERE u.followed_at < $1
          AND u.unfollowed_at IS NULL
      )
      SELECT 
        retained_users,
        avg_user_value,
        retained_users * avg_user_value * 0.1 as retention_value_impact  -- 10% contribution
      FROM retention_impact
    `, [startDate, endDate]);

    return parseFloat(rows[0].retention_value_impact || 0);
  }

  getMonthCount(startDate, endDate) {
    const start = new Date(startDate);
    const end = new Date(endDate);
    return Math.max(1, (end.getFullYear() - start.getFullYear()) * 12 + 
                       (end.getMonth() - start.getMonth()));
  }
}

module.exports = ROICalculator;
```

---

## 11. Complete Cost Dashboard

### 11.1 Cost Dashboard API

```javascript
// src/cost/costDashboardApi.js
const express = require('express');
const router = express.Router();
const LINEApiCostAnalyzer = require('./lineApiCostAnalyzer');
const InfrastructureCostOptimizer = require('./infrastructureCostOptimizer');
const FeatureCostTracker = require('./featureCostTracker');
const ROICalculator = require('./roiCalculator');
const BudgetManager = require('./budgetManager');

// Overview endpoint
router.get('/overview', async (req, res) => {
  try {
    const { year, month } = req.query;
    const now = new Date();
    const targetYear = parseInt(year || now.getFullYear());
    const targetMonth = parseInt(month || now.getMonth() + 1);

    const [lineApiCost, featureCosts, budgetStatus] = await Promise.all([
      new LINEApiCostAnalyzer(req.db, req.redis)
        .calculateMonthlyApiCost(targetYear, targetMonth),
      new FeatureCostTracker(req.db)
        .getFeatureCostBreakdown(
          new Date(targetYear, targetMonth - 1, 1),
          new Date(targetYear, targetMonth, 0)
        ),
      new BudgetManager(req.db).getCurrentMonthSpending(),
    ]);

    res.json({
      period: { year: targetYear, month: targetMonth },
      lineApi: lineApiCost,
      byFeature: featureCosts,
      budget: budgetStatus,
      generatedAt: new Date().toISOString(),
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// ROI Analysis
router.get('/roi', async (req, res) => {
  try {
    const { startDate, endDate } = req.query;
    
    const roi = await new ROICalculator(req.db).calculateLINEOAROI(
      startDate || new Date(Date.now() - 90 * 24 * 60 * 60 * 1000).toISOString(),
      endDate || new Date().toISOString()
    );
    
    res.json(roi);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Optimization Recommendations
router.get('/recommendations', async (req, res) => {
  try {
    const recommendations = [];
    
    // LINE API optimization
    const now = new Date();
    const lineAnalysis = await new LINEApiCostAnalyzer(req.db, req.redis)
      .calculateMonthlyApiCost(now.getFullYear(), now.getMonth() + 1);
    
    if (lineAnalysis.optimalPlan.plan !== 'standard') {
      recommendations.push({
        category: 'LINE API',
        type: 'plan_optimization',
        priority: 'high',
        currentCost: lineAnalysis.costs.standard?.totalCost,
        optimizedCost: lineAnalysis.optimalPlan.totalCost,
        savings: lineAnalysis.estimatedSavings,
        action: `เปลี่ยนไปใช้ ${lineAnalysis.optimalPlan.plan} plan`,
      });
    }

    if (lineAnalysis.quotaUsage > lineAnalysis.costs.standard?.included * 0.8) {
      recommendations.push({
        category: 'LINE API',
        type: 'smart_segmentation',
        priority: 'medium',
        description: 'ใช้ smart segmentation ส่ง message เฉพาะ highly engaged users',
        estimatedSavings: lineAnalysis.quotaUsage * 0.3 * 1.5,
        action: 'Implement engagement-based targeting',
      });
    }

    res.json({
      recommendations,
      totalPotentialSavings: recommendations.reduce((sum, r) => sum + (r.savings || r.estimatedSavings || 0), 0),
      generatedAt: new Date().toISOString(),
    });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

module.exports = router;
```

### 11.2 Cost Dashboard Frontend

```html
<!-- public/admin/cost-dashboard.html -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>Cost Management Dashboard</title>
  <script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Sarabun', sans-serif; background: #f5f5f5; padding: 20px; }
    
    .dashboard-header {
      background: linear-gradient(135deg, #1a237e, #283593);
      color: white;
      padding: 20px 30px;
      border-radius: 12px;
      margin-bottom: 24px;
    }
    
    .kpi-grid {
      display: grid;
      grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
      gap: 16px;
      margin-bottom: 24px;
    }
    
    .kpi-card {
      background: white;
      border-radius: 12px;
      padding: 20px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.1);
      border-left: 4px solid;
    }
    
    .kpi-card.warning { border-left-color: #FF9800; }
    .kpi-card.success { border-left-color: #4CAF50; }
    .kpi-card.danger { border-left-color: #F44336; }
    .kpi-card.info { border-left-color: #2196F3; }
    
    .kpi-label { font-size: 12px; color: #666; margin-bottom: 8px; }
    .kpi-value { font-size: 28px; font-weight: bold; }
    .kpi-sub { font-size: 12px; color: #999; margin-top: 4px; }
    
    .chart-grid {
      display: grid;
      grid-template-columns: 1fr 1fr;
      gap: 20px;
      margin-bottom: 24px;
    }
    
    .chart-card {
      background: white;
      border-radius: 12px;
      padding: 20px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    }
    
    .chart-card h3 {
      font-size: 16px;
      margin-bottom: 16px;
      color: #333;
    }
    
    .recommendations {
      background: white;
      border-radius: 12px;
      padding: 20px;
      box-shadow: 0 2px 8px rgba(0,0,0,0.1);
    }
    
    .recommendation-item {
      display: flex;
      align-items: flex-start;
      padding: 12px;
      border-bottom: 1px solid #f0f0f0;
      gap: 12px;
    }
    
    .priority-badge {
      padding: 4px 8px;
      border-radius: 4px;
      font-size: 11px;
      font-weight: bold;
      white-space: nowrap;
    }
    
    .priority-badge.high { background: #ffebee; color: #c62828; }
    .priority-badge.medium { background: #fff3e0; color: #e65100; }
    .priority-badge.low { background: #e8f5e9; color: #2e7d32; }
    
    .savings-amount { color: #4CAF50; font-weight: bold; }
    
    @media (max-width: 768px) {
      .chart-grid { grid-template-columns: 1fr; }
    }
  </style>
</head>
<body>
  <div class="dashboard-header">
    <h1>💰 Cost Management Dashboard</h1>
    <p>LINE OA Bot - วิเคราะห์และจัดการค่าใช้จ่าย</p>
  </div>

  <div class="kpi-grid">
    <div class="kpi-card info">
      <div class="kpi-label">ค่าใช้จ่ายรวมเดือนนี้</div>
      <div class="kpi-value" id="total-cost">-</div>
      <div class="kpi-sub" id="budget-percent">- % ของ budget</div>
    </div>
    <div class="kpi-card warning">
      <div class="kpi-label">LINE API Cost</div>
      <div class="kpi-value" id="line-api-cost">-</div>
      <div class="kpi-sub" id="quota-used">Quota: - messages</div>
    </div>
    <div class="kpi-card success">
      <div class="kpi-label">ROI เดือนนี้</div>
      <div class="kpi-value" id="roi">-</div>
      <div class="kpi-sub">ผลตอบแทนจากการลงทุน</div>
    </div>
    <div class="kpi-card danger">
      <div class="kpi-label">Potential Savings</div>
      <div class="kpi-value" id="potential-savings">-</div>
      <div class="kpi-sub">จากการ optimize</div>
    </div>
  </div>

  <div class="chart-grid">
    <div class="chart-card">
      <h3>📊 Cost by Category</h3>
      <canvas id="costBreakdownChart"></canvas>
    </div>
    <div class="chart-card">
      <h3>📈 Cost Trend (6 เดือน)</h3>
      <canvas id="costTrendChart"></canvas>
    </div>
    <div class="chart-card">
      <h3>🔧 Cost by Feature</h3>
      <canvas id="featureCostChart"></canvas>
    </div>
    <div class="chart-card">
      <h3>💡 ROI Analysis</h3>
      <canvas id="roiChart"></canvas>
    </div>
  </div>

  <div class="recommendations">
    <h3 style="margin-bottom: 16px">🎯 Optimization Recommendations</h3>
    <div id="recommendations-list">
      <div style="text-align: center; padding: 20px; color: #999">กำลังโหลด...</div>
    </div>
  </div>

  <script>
    // โหลดข้อมูล
    async function loadDashboard() {
      try {
        const [overview, roi, recommendations] = await Promise.all([
          fetch('/api/cost/overview').then(r => r.json()),
          fetch('/api/cost/roi').then(r => r.json()),
          fetch('/api/cost/recommendations').then(r => r.json()),
        ]);

        updateKPIs(overview, roi, recommendations);
        renderCharts(overview, roi);
        renderRecommendations(recommendations);
      } catch (error) {
        console.error('Error loading dashboard:', error);
      }
    }

    function updateKPIs(overview, roi, recommendations) {
      const totalCost = Object.values(overview.budget).reduce((sum, v) => sum + v, 0);
      
      document.getElementById('total-cost').textContent = 
        `฿${totalCost.toLocaleString('th-TH')}`;
      document.getElementById('budget-percent').textContent = 
        `${Math.round(totalCost / 50000 * 100)}% ของ budget ฿50,000`;
      
      document.getElementById('line-api-cost').textContent = 
        `฿${(overview.lineApi?.optimalPlan?.totalCost || 0).toLocaleString('th-TH')}`;
      document.getElementById('quota-used').textContent = 
        `Quota: ${(overview.lineApi?.quotaUsage || 0).toLocaleString()} messages`;
      
      document.getElementById('roi').textContent = 
        `${roi.roi || 0}%`;
      
      document.getElementById('potential-savings').textContent = 
        `฿${(recommendations.totalPotentialSavings || 0).toLocaleString('th-TH')}`;
    }

    function renderCharts(overview, roi) {
      // Cost breakdown pie chart
      const costData = overview.budget || {};
      const costLabels = Object.keys(costData).map(k => k.replace('_', ' ').toUpperCase());
      const costValues = Object.values(costData);

      new Chart(document.getElementById('costBreakdownChart'), {
        type: 'doughnut',
        data: {
          labels: costLabels,
          datasets: [{
            data: costValues,
            backgroundColor: ['#2196F3', '#4CAF50', '#FF9800', '#F44336', '#9C27B0'],
          }],
        },
        options: {
          responsive: true,
          plugins: {
            legend: { position: 'bottom' },
          },
        },
      });

      // ROI bar chart
      const roiData = roi.benefits || {};
      new Chart(document.getElementById('roiChart'), {
        type: 'bar',
        data: {
          labels: ['Direct Revenue', 'Saved CS Cost', 'Retention Value', 'Total Costs'],
          datasets: [{
            label: '฿',
            data: [
              roiData.directRevenue || 0,
              roiData.savedCSCost || 0,
              roiData.retentionValue || 0,
              -(roi.costs?.total || 0),
            ],
            backgroundColor: (ctx) => ctx.raw >= 0 ? '#4CAF50' : '#F44336',
          }],
        },
        options: {
          responsive: true,
          plugins: {
            legend: { display: false },
          },
          scales: {
            y: {
              beginAtZero: false,
              ticks: {
                callback: (v) => `฿${v.toLocaleString('th-TH')}`,
              },
            },
          },
        },
      });
    }

    function renderRecommendations(data) {
      const container = document.getElementById('recommendations-list');
      
      if (!data.recommendations || data.recommendations.length === 0) {
        container.innerHTML = '<div style="text-align: center; padding: 20px; color: #666">ไม่มีคำแนะนำในขณะนี้ 🎉</div>';
        return;
      }

      container.innerHTML = data.recommendations.map(rec => `
        <div class="recommendation-item">
          <span class="priority-badge ${rec.priority}">${rec.priority.toUpperCase()}</span>
          <div style="flex: 1">
            <div style="font-weight: bold; margin-bottom: 4px">${rec.category}: ${rec.type.replace('_', ' ')}</div>
            <div style="font-size: 14px; color: #666">${rec.description || rec.action}</div>
          </div>
          <div style="text-align: right">
            <div class="savings-amount">
              ประหยัด ฿${((rec.savings || rec.estimatedSavings || 0)).toLocaleString('th-TH')}
            </div>
            <div style="font-size: 12px; color: #999">/เดือน</div>
          </div>
        </div>
      `).join('');
    }

    loadDashboard();
    // Refresh ทุก 5 นาที
    setInterval(loadDashboard, 300000);
  </script>
</body>
</html>
```

---

## สรุปบทที่ 98

### Cost Optimization Checklist

- [ ] วิเคราะห์ LINE API quota usage และเลือก plan ที่เหมาะสม
- [ ] Implement smart segmentation ลด broadcast ที่ไม่จำเป็น
- [ ] ใช้ message batching และ multicast แทน push ทีละคน
- [ ] Right-size EC2/GKE instances ตาม actual usage
- [ ] ตั้งค่า Auto Scaling policies
- [ ] ใช้ Reserved Instances สำหรับ production workloads
- [ ] ใช้ Spot/Preemptible instances สำหรับ batch jobs
- [ ] Optimize database queries และ add indexes
- [ ] Setup connection pooling
- [ ] Implement caching layer
- [ ] Optimize CDN cache TTL
- [ ] Compress images ก่อน upload
- [ ] ตั้งค่า log retention policy
- [ ] ใช้ sampling สำหรับ high-volume logs
- [ ] Tag ทุก resource ตาม cost allocation
- [ ] ตั้งค่า budget alerts
- [ ] Review costs ทุกเดือน
- [ ] คำนวณ ROI ทุก quarter
- [ ] สร้าง cost dashboard สำหรับ team

### เป้าหมาย Cost Reduction

| Category | Current | Target | Expected Savings |
|----------|---------|--------|-----------------|
| LINE API | ฿14,000 | ฿8,000 | ฿6,000/month |
| Infrastructure | ฿10,000 | ฿4,000 | ฿6,000/month |
| Database | ฿8,000 | ฿4,400 | ฿3,600/month |
| CDN | ฿4,800 | ฿3,200 | ฿1,600/month |
| Monitoring | ฿2,000 | ฿1,200 | ฿800/month |
| **Total** | **฿40,000** | **฿22,000** | **฿18,000/month** |

**Annual Savings: ฿216,000/year (45% reduction)**
