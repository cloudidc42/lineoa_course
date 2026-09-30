# Part 100: Production Best Practices และ Capstone Project

## บทนำ - บทสุดท้ายของหลักสูตร

ยินดีด้วยที่เดินทางมาถึงบทที่ 100! บทนี้เป็นบทสรุปของทุกสิ่งที่เรียนมาตลอดหลักสูตร เราจะรวบรวม best practices ทั้งหมดสำหรับการ deploy LINE Bot สู่ production พร้อม capstone project ที่จะนำทุกทักษะมาใช้งานจริง

---

## 1. Pre-Launch Checklist (100+ รายการ)

### 1.1 Architecture Checklist

```
✅ Architecture Review

High Availability:
□ Application ทำงานได้ multi-instance (stateless)
□ Database มี replication (primary + replica)
□ Cache layer มี replication
□ Load balancer configured ถูกต้อง
□ Health checks ทุก service ทำงาน
□ Circuit breakers implement แล้ว
□ Graceful shutdown handles แล้ว

Scalability:
□ Horizontal scaling ทำงานได้
□ Auto-scaling rules configured
□ Database connection pooling ตั้งค่าถูก
□ Cache ลด database load อย่างน้อย 50%
□ CDN สำหรับ static assets
□ Async processing สำหรับ heavy operations

Resilience:
□ Retry logic สำหรับ external API calls
□ Timeout configured ทุก external call
□ Fallback behavior เมื่อ dependencies down
□ Data backups ทำงานและ tested
□ Disaster recovery procedure documented
□ RTO/RPO defined และ achievable
```

### 1.2 Security Checklist

```
✅ Security Review

LINE Webhook Security:
□ Validate LINE signature ทุก request
□ HTTPS only (TLS 1.2+)
□ Certificate valid และ auto-renew
□ Webhook endpoint ไม่ expose internal info

Authentication & Authorization:
□ LIFF authentication ทำงานถูกต้อง
□ LINE Login configured
□ JWT secrets ไม่ hardcoded
□ Token expiry configured (access: 1h, refresh: 30d)
□ Role-based access control implement

Data Security:
□ Sensitive data encrypted at rest
□ Sensitive data encrypted in transit
□ PII data ไม่ถูก log
□ Database credentials ใน secrets manager
□ API keys ใน environment variables ไม่ใช่ code
□ .env files อยู่ใน .gitignore

Infrastructure Security:
□ Security groups/firewall rules ตั้งค่าถูก
□ Principle of least privilege สำหรับ IAM
□ VPC/network segmentation
□ WAF enabled (ถ้าจำเป็น)
□ DDoS protection
□ Regular security patches

Compliance:
□ PDPA compliance สำหรับ Thai users
□ Privacy policy updated
□ Cookie consent (ถ้ามี web component)
□ Data retention policy documented
□ User data deletion capability
```

### 1.3 Performance Checklist

```
✅ Performance Review

Response Times:
□ Webhook handler P95 < 500ms
□ LIFF pages load < 3 seconds on mobile
□ Database queries P95 < 100ms
□ Cache hit rate > 80%
□ LINE API calls P95 < 1 second

Load Testing Results:
□ Load test ผ่านที่ expected peak traffic
□ Stress test ระบุ breaking point
□ Soak test (24h) ไม่มี memory leaks
□ Latency stable ภายใต้ load

Optimization:
□ Database indexes ครอบคลุม queries หลัก
□ N+1 queries แก้ไขแล้ว
□ Images optimized และผ่าน CDN
□ Caching headers ตั้งค่าถูก
□ Gzip compression enabled
□ Connection keep-alive enabled
```

### 1.4 Monitoring Checklist

```
✅ Monitoring Review

Metrics:
□ Application metrics (request rate, error rate, latency)
□ Business metrics (orders, revenue, active users)
□ LINE API quota monitoring
□ Infrastructure metrics (CPU, memory, disk)
□ Database metrics (connections, query time, replication lag)
□ Kafka consumer lag

Alerting:
□ P1 alerts สำหรับ critical issues
□ P2 alerts สำหรับ important issues
□ Alert fatigue ไม่มี (ไม่มี noise alerts)
□ On-call rotation configured
□ Escalation policy defined

Logging:
□ Structured logging (JSON format)
□ Log levels appropriate (no debug in production)
□ Correlation IDs ใน logs
□ PII ไม่อยู่ใน logs
□ Log retention policy (30 days minimum)
□ Log search ทำได้รวดเร็ว

Dashboards:
□ Real-time status dashboard
□ Business KPI dashboard
□ Infrastructure health dashboard
□ Error tracking dashboard (Sentry)
□ Accessible to stakeholders
```

### 1.5 Testing Checklist

```
✅ Testing Review

Unit Tests:
□ Coverage > 80%
□ Critical paths 100% coverage
□ LINE message handlers tested
□ Business logic tested
□ Error scenarios tested

Integration Tests:
□ LINE webhook integration tests
□ Database integration tests
□ External API mocks configured
□ End-to-end flows tested

Performance Tests:
□ Load test results documented
□ Stress test results documented
□ Soak test completed (min 4 hours)

User Acceptance:
□ QA sign-off
□ Stakeholder demo completed
□ LINE OA account tested on real device
□ iOS testing
□ Android testing
□ LIFF tested in LINE app
```

---

## 2. Go-Live Procedure

### 2.1 Deployment Plan Template

```markdown
# Production Deployment Plan
Service: LINE Bot v[X.Y.Z]
Date: [DD/MM/YYYY]
Time: [HH:MM] ICT
Duration: ~[N] hours

## Deployment Team
- Deployment Lead: [ชื่อ]
- QA: [ชื่อ]
- On-Call: [ชื่อ]
- Rollback Contact: [ชื่อ]

## Pre-Deployment Steps (2 hours before)

1. [ ] Final code review completed
2. [ ] Staging deployment verified
3. [ ] Database migration tested on staging
4. [ ] Rollback plan reviewed
5. [ ] Team notified in #line-deployments
6. [ ] Monitoring dashboards ready

## Deployment Steps

### Phase 1: Database Migration (T-30 min)
1. [ ] Backup production database
2. [ ] Run migration scripts
3. [ ] Verify migration successful
4. [ ] Test backward compatibility

### Phase 2: Application Deployment (T=0)
1. [ ] Update Docker image tags
2. [ ] Deploy to 1 instance (canary)
3. [ ] Monitor error rate for 5 minutes
4. [ ] If healthy: deploy remaining instances
5. [ ] Verify health checks pass
6. [ ] Verify LINE webhook responding

### Phase 3: Verification (T+15 min)
1. [ ] Test follow event
2. [ ] Test message reply
3. [ ] Test LIFF app
4. [ ] Check metrics are normal
5. [ ] QA sign-off

## Go/No-Go Decision Criteria
✅ GO if:
- Error rate < 1%
- P95 latency < 500ms
- All health checks passing
- QA verified

❌ NO-GO if:
- Error rate > 5%
- P95 latency > 2 seconds
- Any health check failing
- Critical bugs found

## Rollback Procedure
If No-Go:
1. kubectl rollout undo deployment/line-bot
2. Verify rollback successful
3. Notify team
4. Schedule post-mortem

## Post-Deployment Monitoring (2 hours)
- [ ] Error rate < 1%
- [ ] Response time normal
- [ ] LINE API quota normal
- [ ] No alerts firing
```

### 2.2 Zero-Downtime Deployment Script

```bash
#!/bin/bash
# scripts/deploy-production.sh

set -e
set -u

NAMESPACE="line-bot"
DEPLOYMENT="webhook-handler"
IMAGE_TAG=$1

if [ -z "$IMAGE_TAG" ]; then
  echo "Usage: ./deploy-production.sh <image-tag>"
  exit 1
fi

echo "🚀 Starting production deployment..."
echo "Image: ${IMAGE_TAG}"

# Pre-deployment checks
pre_deployment_checks() {
  echo "Running pre-deployment checks..."
  
  # Check cluster connection
  kubectl cluster-info > /dev/null 2>&1 || {
    echo "❌ Cannot connect to cluster"
    exit 1
  }
  
  # Check current deployment health
  READY=$(kubectl get deployment/${DEPLOYMENT} -n ${NAMESPACE} \
    -o jsonpath='{.status.readyReplicas}')
  DESIRED=$(kubectl get deployment/${DEPLOYMENT} -n ${NAMESPACE} \
    -o jsonpath='{.spec.replicas}')
  
  if [ "$READY" != "$DESIRED" ]; then
    echo "❌ Current deployment not healthy: ${READY}/${DESIRED}"
    exit 1
  fi
  
  echo "✅ Pre-deployment checks passed"
}

# Deploy with rolling update
rolling_deploy() {
  echo "Updating image to ${IMAGE_TAG}..."
  
  kubectl set image deployment/${DEPLOYMENT} \
    ${DEPLOYMENT}=gcr.io/project/line-bot:${IMAGE_TAG} \
    -n ${NAMESPACE}
  
  # Wait for rollout
  kubectl rollout status deployment/${DEPLOYMENT} -n ${NAMESPACE} \
    --timeout=5m
  
  echo "✅ Deployment successful"
}

# Verify deployment
verify_deployment() {
  echo "Verifying deployment..."
  sleep 30  # Wait for pods to stabilize
  
  # Check health endpoint
  HEALTH=$(curl -s -o /dev/null -w "%{http_code}" \
    https://api.yourapp.com/health)
  
  if [ "$HEALTH" != "200" ]; then
    echo "❌ Health check failed: HTTP ${HEALTH}"
    return 1
  fi
  
  # Check error rate (using metrics API)
  ERROR_RATE=$(curl -s "http://prometheus:9090/api/v1/query?query=\
    rate(http_requests_total{status=~'5..'}[5m])" | \
    python3 -c "import sys,json; d=json.load(sys.stdin); \
    print(d['data']['result'][0]['value'][1] if d['data']['result'] else '0')")
  
  if (( $(echo "$ERROR_RATE > 0.05" | bc -l) )); then
    echo "❌ Error rate too high: ${ERROR_RATE}"
    return 1
  fi
  
  echo "✅ Deployment verified successfully"
}

# Rollback if needed
rollback() {
  echo "⚠️  Rolling back deployment..."
  kubectl rollout undo deployment/${DEPLOYMENT} -n ${NAMESPACE}
  kubectl rollout status deployment/${DEPLOYMENT} -n ${NAMESPACE}
  echo "✅ Rollback completed"
  
  # Notify team
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H 'Content-type: application/json' \
    -d "{\"text\":\"⚠️ LINE Bot deployment ROLLED BACK to previous version\"}"
}

# Main deployment flow
main() {
  pre_deployment_checks
  rolling_deploy
  
  if ! verify_deployment; then
    echo "Deployment verification failed!"
    rollback
    exit 1
  fi
  
  # Notify success
  curl -s -X POST "$SLACK_WEBHOOK" \
    -H 'Content-type: application/json' \
    -d "{\"text\":\"✅ LINE Bot deployed successfully: ${IMAGE_TAG}\"}"
  
  echo "🎉 Production deployment complete!"
}

main
```

---

## 3. Post-Launch Monitoring

### 3.1 Real-time Monitoring Setup

```typescript
// monitoring/production-monitor.ts
import * as Prometheus from 'prom-client';
import { logger } from '../utils/logger';

// Custom metrics สำหรับ LINE Bot
const metrics = {
  // Webhook metrics
  webhookEventsTotal: new Prometheus.Counter({
    name: 'line_webhook_events_total',
    help: 'Total LINE webhook events received',
    labelNames: ['event_type', 'source_type'],
  }),
  
  webhookProcessingDuration: new Prometheus.Histogram({
    name: 'line_webhook_processing_duration_seconds',
    help: 'Webhook event processing duration',
    labelNames: ['event_type'],
    buckets: [0.01, 0.05, 0.1, 0.3, 0.5, 1, 2, 5],
  }),
  
  // LINE API metrics
  lineApiCallsTotal: new Prometheus.Counter({
    name: 'line_api_calls_total',
    help: 'Total LINE API calls made',
    labelNames: ['method', 'status'],
  }),
  
  lineApiDuration: new Prometheus.Histogram({
    name: 'line_api_duration_seconds',
    help: 'LINE API call duration',
    labelNames: ['method'],
    buckets: [0.1, 0.3, 0.5, 1, 2, 5, 10],
  }),
  
  // Business metrics
  ordersCreated: new Prometheus.Counter({
    name: 'business_orders_created_total',
    help: 'Total orders created via LINE',
    labelNames: ['payment_method'],
  }),
  
  activeUsers: new Prometheus.Gauge({
    name: 'business_active_users_current',
    help: 'Current active users',
  }),
  
  messagesSent: new Prometheus.Counter({
    name: 'line_messages_sent_total',
    help: 'Total messages sent via LINE',
    labelNames: ['message_type', 'send_type'],
  }),
};

// Middleware สำหรับ tracking webhooks
export function trackWebhookEvent(eventType: string, sourceType: string) {
  return (target: any, key: string, descriptor: PropertyDescriptor) => {
    const original = descriptor.value;
    
    descriptor.value = async function (...args: any[]) {
      const end = metrics.webhookProcessingDuration
        .startTimer({ event_type: eventType });
      
      metrics.webhookEventsTotal.inc({
        event_type: eventType,
        source_type: sourceType,
      });
      
      try {
        const result = await original.apply(this, args);
        return result;
      } finally {
        end();
      }
    };
    
    return descriptor;
  };
}

// Health check endpoint
export function setupHealthCheck(app: any) {
  app.get('/health', async (req: any, res: any) => {
    const checks = await runHealthChecks();
    const isHealthy = Object.values(checks).every(c => c.status === 'healthy');
    
    res.status(isHealthy ? 200 : 503).json({
      status: isHealthy ? 'healthy' : 'unhealthy',
      timestamp: new Date().toISOString(),
      version: process.env.APP_VERSION,
      checks,
    });
  });
  
  app.get('/ready', async (req: any, res: any) => {
    const ready = await checkReadiness();
    res.status(ready ? 200 : 503).json({ ready });
  });
  
  app.get('/metrics', async (req: any, res: any) => {
    res.set('Content-Type', Prometheus.register.contentType);
    res.end(await Prometheus.register.metrics());
  });
}

async function runHealthChecks(): Promise<Record<string, any>> {
  const results: Record<string, any> = {};
  
  // Database check
  try {
    await db.query('SELECT 1');
    results.database = { status: 'healthy' };
  } catch (error: any) {
    results.database = { status: 'unhealthy', error: error.message };
  }
  
  // Redis check
  try {
    await redis.ping();
    results.redis = { status: 'healthy' };
  } catch (error: any) {
    results.redis = { status: 'unhealthy', error: error.message };
  }
  
  // Kafka check
  try {
    const kafkaConnected = await checkKafkaConnection();
    results.kafka = { status: kafkaConnected ? 'healthy' : 'unhealthy' };
  } catch (error: any) {
    results.kafka = { status: 'unhealthy', error: error.message };
  }
  
  return results;
}
```

### 3.2 SLA Monitoring

```typescript
// monitoring/sla-monitor.ts

export class SLAMonitor {
  private slos = {
    availability: 99.9,          // 99.9% uptime
    webhookLatency: 500,          // P95 < 500ms
    errorRate: 1,                 // < 1% errors
    lineApiSuccessRate: 99,       // 99% LINE API success
  };

  async calculateCurrentSLO(window: string = '30d'): Promise<SLOReport> {
    const [
      availability,
      webhookLatencyP95,
      errorRate,
      lineApiSuccessRate,
    ] = await Promise.all([
      this.calculateAvailability(window),
      this.calculateWebhookLatencyP95(window),
      this.calculateErrorRate(window),
      this.calculateLineApiSuccessRate(window),
    ]);

    const report: SLOReport = {
      period: window,
      calculatedAt: new Date().toISOString(),
      slos: {
        availability: {
          target: this.slos.availability,
          current: availability,
          meeting: availability >= this.slos.availability,
          errorBudgetRemaining: this.calculateErrorBudget(
            availability,
            this.slos.availability,
            window
          ),
        },
        webhookLatency: {
          target: this.slos.webhookLatency,
          current: webhookLatencyP95,
          meeting: webhookLatencyP95 <= this.slos.webhookLatency,
        },
        errorRate: {
          target: this.slos.errorRate,
          current: errorRate,
          meeting: errorRate <= this.slos.errorRate,
        },
        lineApiSuccessRate: {
          target: this.slos.lineApiSuccessRate,
          current: lineApiSuccessRate,
          meeting: lineApiSuccessRate >= this.slos.lineApiSuccessRate,
        },
      },
    };

    return report;
  }

  private calculateErrorBudget(
    current: number,
    target: number,
    window: string
  ): ErrorBudget {
    const windowHours = this.parseWindowToHours(window);
    const totalMinutes = windowHours * 60;
    const allowedDowntimeMinutes = totalMinutes * (1 - target / 100);
    const actualDowntimeMinutes = totalMinutes * (1 - current / 100);
    const remainingMinutes = allowedDowntimeMinutes - actualDowntimeMinutes;

    return {
      totalMinutes: allowedDowntimeMinutes,
      usedMinutes: actualDowntimeMinutes,
      remainingMinutes,
      percentageUsed: (actualDowntimeMinutes / allowedDowntimeMinutes) * 100,
    };
  }

  private parseWindowToHours(window: string): number {
    const match = window.match(/(\d+)([dhm])/);
    if (!match) return 720; // default 30 days
    
    const value = parseInt(match[1]);
    switch (match[2]) {
      case 'd': return value * 24;
      case 'h': return value;
      case 'm': return value / 60;
      default: return 720;
    }
  }
}
```

---

## 4. Scaling Playbook

### 4.1 When to Scale

```
Scaling Decision Matrix:

Metric                    | Action
--------------------------|------------------------------------------
CPU > 70% sustained       | Scale out (add instances)
Memory > 80%              | Scale out or increase instance size
DB connections > 80% max  | Scale out or use connection proxy
Kafka lag > 10,000        | Scale out consumers
LINE API approaching limit | Optimize message sending or upgrade plan
Error rate > 5%           | Investigate before scaling
Response time P95 > 1s    | Profile and optimize first

Scaling Rules:
1. ลอง vertical scale ก่อน (cheaper for databases)
2. Horizontal scale สำหรับ stateless services
3. ทำ load test ก่อน scaling decision
4. Monitor 24h หลัง scaling
```

### 4.2 Kubernetes Scaling Configuration

```yaml
# k8s/hpa-webhook-handler.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webhook-handler-hpa
  namespace: line-bot
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webhook-handler
  
  minReplicas: 2
  maxReplicas: 20
  
  metrics:
  # Scale based on CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  
  # Scale based on memory
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  
  # Scale based on custom metric (requests per second)
  - type: External
    external:
      metric:
        name: line_webhook_events_per_second
        selector:
          matchLabels:
            service: webhook-handler
      target:
        type: AverageValue
        averageValue: "50"
  
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 4       # เพิ่มทีละ 4 pods
        periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300  # รอ 5 นาทีก่อนลด
      policies:
      - type: Pods
        value: 2       # ลดทีละ 2 pods
        periodSeconds: 120
```

---

## 5. Disaster Recovery

### 5.1 RTO/RPO Targets

```
Recovery Objectives:

P1 Services (LINE Webhook, LIFF API):
- RTO: 30 minutes (restore within 30 min)
- RPO: 5 minutes (max 5 min data loss)

P2 Services (Analytics, CRM):
- RTO: 4 hours
- RPO: 1 hour

P3 Services (Reports, Batch jobs):
- RTO: 24 hours
- RPO: 24 hours
```

### 5.2 DR Procedures

```markdown
# Disaster Recovery Runbook

## Scenario 1: Complete Region Failure

### Detection
- Multiple P1 alerts firing
- Region health check failing
- Status page showing region issue

### Recovery Steps

1. **Confirm region failure (5 min)**
   - Check AWS/GCP status page
   - Verify in multiple tools
   - Do NOT assume: verify first

2. **Activate DR region (10 min)**
   ```bash
   # Update DNS to DR region
   aws route53 change-resource-record-sets \
     --hosted-zone-id $ZONE_ID \
     --change-batch file://dr-dns-change.json
   
   # Verify DNS propagation
   dig api.yourapp.com +short
   ```

3. **Start DR services (15 min)**
   ```bash
   # Scale up DR services
   kubectl scale deployment --all -n line-bot \
     --replicas=3 \
     --context=dr-cluster
   
   # Verify pods running
   kubectl get pods -n line-bot --context=dr-cluster
   ```

4. **Restore latest backup (15 min)**
   ```bash
   # ถ้าจำเป็น restore database
   ./scripts/restore-db.sh --snapshot=latest --region=dr
   ```

5. **Update LINE Webhook URL (5 min)**
   - LINE Developers Console → Messaging API
   - Update webhook URL to DR endpoint
   - Verify webhook

6. **Verify service health (10 min)**
   ```bash
   # Test webhook
   curl https://dr-api.yourapp.com/health
   
   # Test LINE message
   ./scripts/send-test-message.sh --env=dr
   ```

7. **Communicate (ongoing)**
   - Update status page
   - Notify stakeholders
   - Provide ETA for primary region recovery

Total DR Time: ~60 minutes
```

---

## 6. Capstone Project: Build Complete LINE OA System

### 6.1 Project Overview: "FreshMart" - ระบบ E-Commerce สำหรับร้านค้าปลีก

```
Project: FreshMart LINE OA System
ประเภทธุรกิจ: ร้านขายของชำออนไลน์
เป้าหมาย: สร้าง LINE Bot ที่ช่วยให้ลูกค้าสั่งของชำผ่าน LINE

Key Features:
1. สินค้า catalog ด้วย Flex Messages
2. ระบบ shopping cart
3. สั่งซื้อและชำระเงินผ่าน LINE Pay
4. ติดตามสถานะ delivery
5. ระบบ loyalty points
6. Customer support chatbot
7. Analytics dashboard
8. Admin LIFF app
```

### 6.2 Architecture Planning

```
FreshMart Architecture:

LINE App
   ↓
LINE Platform (Webhook + LIFF)
   ↓
API Gateway (nginx/Kong)
   ↓
┌──────────────────────────────────────┐
│          Application Services         │
│                                      │
│  Webhook Handler  │  LIFF API        │
│  (Node.js)        │  (Node.js)       │
│                   │                  │
│  Order Service    │  Product Service  │
│  (Node.js)        │  (Node.js)       │
│                   │                  │
│  Payment Service  │  Notification    │
│  (Node.js)        │  Service (Node)  │
└──────────────────────────────────────┘
         ↓                    ↓
   ┌──────────┐         ┌──────────┐
   │  Kafka   │         │   Redis  │
   │  Cluster │         │  Cache   │
   └──────────┘         └──────────┘
         ↓
   ┌──────────┐
   │PostgreSQL│
   │  (RDS)   │
   └──────────┘
         ↓
   ┌──────────┐
   │Analytics │
   │(ClickHse)│
   └──────────┘
```

### 6.3 Database Schema

```sql
-- database/schema.sql

-- Users
CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  line_user_id VARCHAR(255) UNIQUE NOT NULL,
  display_name VARCHAR(255),
  picture_url TEXT,
  email VARCHAR(255),
  phone VARCHAR(20),
  loyalty_points INT DEFAULT 0,
  tier VARCHAR(20) DEFAULT 'bronze',
  followed_at TIMESTAMPTZ DEFAULT NOW(),
  last_active_at TIMESTAMPTZ,
  is_blocked BOOLEAN DEFAULT false,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_users_line_id ON users(line_user_id);
CREATE INDEX idx_users_tier ON users(tier);

-- Products
CREATE TABLE products (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  sku VARCHAR(100) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  name_en VARCHAR(255),
  description TEXT,
  category_id UUID REFERENCES categories(id),
  price DECIMAL(10,2) NOT NULL,
  compare_at_price DECIMAL(10,2),
  unit VARCHAR(50),
  stock_quantity INT DEFAULT 0,
  is_available BOOLEAN DEFAULT true,
  weight_grams INT,
  images JSONB DEFAULT '[]',
  tags TEXT[] DEFAULT '{}',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_sku ON products(sku);
CREATE INDEX idx_products_available ON products(is_available) WHERE is_available = true;

-- Orders
CREATE TABLE orders (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_number VARCHAR(50) UNIQUE NOT NULL,
  user_id UUID REFERENCES users(id) NOT NULL,
  status VARCHAR(50) NOT NULL DEFAULT 'pending',
  payment_status VARCHAR(50) DEFAULT 'unpaid',
  payment_method VARCHAR(50),
  subtotal DECIMAL(10,2) NOT NULL,
  shipping_fee DECIMAL(10,2) DEFAULT 0,
  discount DECIMAL(10,2) DEFAULT 0,
  total_amount DECIMAL(10,2) NOT NULL,
  shipping_address JSONB,
  notes TEXT,
  coupon_code VARCHAR(50),
  delivery_slot TIMESTAMPTZ,
  delivered_at TIMESTAMPTZ,
  cancelled_at TIMESTAMPTZ,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Order Items
CREATE TABLE order_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id UUID REFERENCES orders(id) NOT NULL,
  product_id UUID REFERENCES products(id) NOT NULL,
  product_snapshot JSONB NOT NULL,
  quantity INT NOT NULL,
  unit_price DECIMAL(10,2) NOT NULL,
  total_price DECIMAL(10,2) NOT NULL,
  notes TEXT
);

CREATE INDEX idx_order_items_order ON order_items(order_id);

-- Cart
CREATE TABLE carts (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) UNIQUE NOT NULL,
  expires_at TIMESTAMPTZ DEFAULT NOW() + INTERVAL '7 days',
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE cart_items (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  cart_id UUID REFERENCES carts(id) NOT NULL,
  product_id UUID REFERENCES products(id) NOT NULL,
  quantity INT NOT NULL DEFAULT 1,
  added_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(cart_id, product_id)
);
```

### 6.4 Development Phases

```
Phase 1: Foundation (สัปดาห์ 1-2)

1. Project setup
   □ สร้าง GitHub repository
   □ Setup Docker Compose for local development
   □ Setup CI/CD pipeline
   □ Create LINE OA account สำหรับ development
   □ Configure LINE channel

2. Basic webhook
   □ Express server setup
   □ Webhook endpoint
   □ Signature validation
   □ Basic echo bot (ทดสอบ connection)
   □ Logging setup

3. Database
   □ Design schema
   □ Setup PostgreSQL
   □ Run migrations
   □ Seed test data

Phase 2: Core Features (สัปดาห์ 3-5)

4. Product Catalog
   □ Product model และ CRUD API
   □ Browse products ผ่าน LINE (Flex Carousel)
   □ Search products
   □ Product detail page (LIFF)
   □ Category navigation ด้วย Rich Menu

5. Shopping Cart
   □ Add/remove items
   □ Cart persistence (Redis)
   □ Cart summary Flex message
   □ Quantity update

6. Order System
   □ Checkout flow
   □ Address selection
   □ Order confirmation message
   □ Order summary Flex message

Phase 3: Payments & Fulfillment (สัปดาห์ 6-7)

7. LINE Pay Integration
   □ Payment request
   □ Payment confirmation webhook
   □ Refund handling
   □ Payment receipt

8. Delivery Tracking
   □ Order status updates
   □ Push notifications ทุก status change
   □ Tracking link

Phase 4: Engagement (สัปดาห์ 8-9)

9. Loyalty System
   □ Points calculation
   □ Points history
   □ Tier system
   □ Rewards redemption

10. Customer Support
    □ Basic FAQ bot
    □ Escalation to human agent
    □ Support ticket tracking

Phase 5: Analytics & Optimization (สัปดาห์ 10)

11. Analytics
    □ Event tracking
    □ Dashboard setup
    □ KPI monitoring
    □ Kafka event pipeline

12. Performance Optimization
    □ Caching strategy
    □ Load testing
    □ Query optimization

Phase 6: Production Launch (สัปดาห์ 11-12)

13. Pre-launch
    □ Security review
    □ Performance testing
    □ UAT
    □ Documentation

14. Launch
    □ Production deployment
    □ Monitoring setup
    □ On-call rotation
    □ Post-launch monitoring
```

### 6.5 Key Implementation: Product Catalog Bot

```typescript
// src/handlers/catalog.handler.ts
import { MessageEvent, FlexMessage, FlexCarousel } from '@line/bot-sdk';
import { ProductService } from '../services/product.service';
import { LineMessageBuilder } from '../utils/message-builder';
import { CartService } from '../services/cart.service';

export class CatalogHandler {
  constructor(
    private productService: ProductService,
    private cartService: CartService,
    private lineClient: any
  ) {}

  async handleCategoryBrowse(
    event: MessageEvent,
    categoryId: string
  ): Promise<void> {
    const userId = event.source.userId!;
    const products = await this.productService.getProductsByCategory(
      categoryId,
      { limit: 10, inStockOnly: true }
    );

    if (products.length === 0) {
      await this.lineClient.replyMessage(event.replyToken, {
        type: 'text',
        text: 'ขออภัย ไม่พบสินค้าในหมวดนี้ในขณะนี้',
      });
      return;
    }

    // สร้าง Flex Carousel จาก products
    const carousel = this.buildProductCarousel(products);

    await this.lineClient.replyMessage(event.replyToken, [
      {
        type: 'text',
        text: `🛒 สินค้าในหมวด "${products[0].category?.name}" (${products.length} รายการ)`,
      },
      carousel,
    ]);
  }

  private buildProductCarousel(products: any[]): FlexMessage {
    const bubbles = products.map(product => ({
      type: 'bubble' as const,
      size: 'micro',
      hero: {
        type: 'image' as const,
        url: product.images[0]?.url || 'https://via.placeholder.com/300',
        size: 'full',
        aspectRatio: '1:1',
        aspectMode: 'cover' as const,
      },
      body: {
        type: 'box' as const,
        layout: 'vertical' as const,
        paddingMd: '17px',
        paddingTop: '15px',
        contents: [
          {
            type: 'text' as const,
            text: product.name,
            weight: 'bold' as const,
            size: 'md',
            wrap: true,
          },
          {
            type: 'box' as const,
            layout: 'baseline' as const,
            contents: [
              {
                type: 'text' as const,
                text: `฿${product.price.toLocaleString()}`,
                size: 'xl',
                color: '#0D8186',
                flex: 0,
              },
              {
                type: 'text' as const,
                text: `/${product.unit}`,
                size: 'sm',
                color: '#888888',
                flex: 0,
              },
            ],
          },
          product.stockQuantity < 10 ? {
            type: 'text' as const,
            text: `เหลือ ${product.stockQuantity} ${product.unit}`,
            size: 'xs',
            color: '#FF4444',
          } : null,
        ].filter(Boolean),
      },
      footer: {
        type: 'box' as const,
        layout: 'vertical' as const,
        gap: 'xs',
        contents: [
          {
            type: 'button' as const,
            action: {
              type: 'postback' as const,
              label: '🛒 เพิ่มลงตะกร้า',
              data: `action=add_to_cart&productId=${product.id}`,
              displayText: `เพิ่ม "${product.name}" ลงตะกร้า`,
            },
            style: 'primary' as const,
            height: 'sm',
            color: '#0D8186',
          },
          {
            type: 'button' as const,
            action: {
              type: 'uri' as const,
              label: 'ดูรายละเอียด',
              uri: `${process.env.LIFF_URL}/product/${product.id}`,
            },
            style: 'link' as const,
            height: 'sm',
          },
        ],
      },
    }));

    return {
      type: 'flex',
      altText: `สินค้า ${products.length} รายการ`,
      contents: {
        type: 'carousel',
        contents: bubbles,
      } as FlexCarousel,
    };
  }

  async handleAddToCart(event: any, productId: string): Promise<void> {
    const userId = event.source.userId!;

    try {
      const [product, cart] = await Promise.all([
        this.productService.getById(productId),
        this.cartService.getOrCreateCart(userId),
      ]);

      if (!product) {
        await this.lineClient.replyMessage(event.replyToken, {
          type: 'text',
          text: 'ขออภัย ไม่พบสินค้านี้',
        });
        return;
      }

      if (product.stockQuantity === 0) {
        await this.lineClient.replyMessage(event.replyToken, {
          type: 'text',
          text: `ขออภัย ${product.name} หมดสต๊อกแล้ว`,
        });
        return;
      }

      await this.cartService.addItem(cart.id, productId, 1);
      const updatedCart = await this.cartService.getCartWithItems(cart.id);

      const message = this.buildCartAddedMessage(product, updatedCart);
      await this.lineClient.replyMessage(event.replyToken, message);
    } catch (error) {
      logger.error('Error adding to cart', { userId, productId, error });
      await this.lineClient.replyMessage(event.replyToken, {
        type: 'text',
        text: 'เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง',
      });
    }
  }

  private buildCartAddedMessage(product: any, cart: any): FlexMessage {
    const cartTotal = cart.items.reduce(
      (sum: number, item: any) => sum + item.product.price * item.quantity,
      0
    );

    return {
      type: 'flex',
      altText: `เพิ่ม ${product.name} ลงตะกร้าแล้ว`,
      contents: {
        type: 'bubble',
        styles: { header: { backgroundColor: '#0D8186' } },
        header: {
          type: 'box',
          layout: 'vertical',
          contents: [{
            type: 'text',
            text: '✅ เพิ่มลงตะกร้าแล้ว!',
            color: '#ffffff',
            weight: 'bold',
            size: 'lg',
          }],
        },
        body: {
          type: 'box',
          layout: 'vertical',
          contents: [
            {
              type: 'text',
              text: product.name,
              weight: 'bold',
              size: 'md',
            },
            {
              type: 'separator',
              margin: 'md',
            },
            {
              type: 'box',
              layout: 'horizontal',
              margin: 'md',
              contents: [
                { type: 'text', text: `รายการในตะกร้า: ${cart.items.length}`, flex: 5 },
                { type: 'text', text: `฿${cartTotal.toLocaleString()}`, flex: 5, align: 'end', color: '#0D8186', weight: 'bold' },
              ],
            },
          ],
        },
        footer: {
          type: 'box',
          layout: 'horizontal',
          gap: 'sm',
          contents: [
            {
              type: 'button',
              action: {
                type: 'postback',
                label: 'ดูตะกร้า',
                data: 'action=view_cart',
              },
              style: 'primary',
              height: 'sm',
              flex: 1,
            },
            {
              type: 'button',
              action: {
                type: 'postback',
                label: 'ซื้อต่อ',
                data: 'action=browse_products',
              },
              style: 'secondary',
              height: 'sm',
              flex: 1,
            },
          ],
        },
      },
    };
  }
}
```

### 6.6 Testing Strategy

```typescript
// tests/catalog.handler.test.ts
import { describe, it, expect, beforeEach, jest } from '@jest/globals';
import { CatalogHandler } from '../src/handlers/catalog.handler';

describe('CatalogHandler', () => {
  let handler: CatalogHandler;
  let mockProductService: any;
  let mockCartService: any;
  let mockLineClient: any;

  beforeEach(() => {
    mockProductService = {
      getProductsByCategory: jest.fn(),
      getById: jest.fn(),
    };
    
    mockCartService = {
      getOrCreateCart: jest.fn(),
      addItem: jest.fn(),
      getCartWithItems: jest.fn(),
    };
    
    mockLineClient = {
      replyMessage: jest.fn().mockResolvedValue(undefined),
    };
    
    handler = new CatalogHandler(
      mockProductService,
      mockCartService,
      mockLineClient
    );
  });

  describe('handleCategoryBrowse', () => {
    it('should send product carousel when products found', async () => {
      const mockProducts = [
        {
          id: 'prod-1',
          name: 'ข้าวสารหอมมะลิ',
          price: 180,
          unit: 'kg',
          stockQuantity: 50,
          images: [{ url: 'https://example.com/rice.jpg' }],
          category: { name: 'ข้าวสาร' },
        },
      ];

      mockProductService.getProductsByCategory.mockResolvedValue(mockProducts);

      const event = createMockMessageEvent('user-123');
      await handler.handleCategoryBrowse(event, 'cat-rice');

      expect(mockLineClient.replyMessage).toHaveBeenCalledWith(
        event.replyToken,
        expect.arrayContaining([
          expect.objectContaining({ type: 'text' }),
          expect.objectContaining({ type: 'flex' }),
        ])
      );
    });

    it('should send "not found" message when no products', async () => {
      mockProductService.getProductsByCategory.mockResolvedValue([]);

      const event = createMockMessageEvent('user-123');
      await handler.handleCategoryBrowse(event, 'cat-empty');

      expect(mockLineClient.replyMessage).toHaveBeenCalledWith(
        event.replyToken,
        expect.objectContaining({
          type: 'text',
          text: expect.stringContaining('ขออภัย'),
        })
      );
    });
  });

  describe('handleAddToCart', () => {
    it('should add product to cart successfully', async () => {
      const mockProduct = {
        id: 'prod-1',
        name: 'ข้าวสารหอมมะลิ',
        price: 180,
        stockQuantity: 50,
      };

      const mockCart = {
        id: 'cart-1',
        items: [{ product: mockProduct, quantity: 1 }],
      };

      mockProductService.getById.mockResolvedValue(mockProduct);
      mockCartService.getOrCreateCart.mockResolvedValue({ id: 'cart-1' });
      mockCartService.addItem.mockResolvedValue(undefined);
      mockCartService.getCartWithItems.mockResolvedValue(mockCart);

      const event = createMockPostbackEvent('user-123');
      await handler.handleAddToCart(event, 'prod-1');

      expect(mockCartService.addItem).toHaveBeenCalledWith('cart-1', 'prod-1', 1);
      expect(mockLineClient.replyMessage).toHaveBeenCalled();
    });

    it('should notify when product out of stock', async () => {
      mockProductService.getById.mockResolvedValue({
        id: 'prod-1',
        name: 'ข้าวสารหอมมะลิ',
        stockQuantity: 0,
      });

      const event = createMockPostbackEvent('user-123');
      await handler.handleAddToCart(event, 'prod-1');

      expect(mockCartService.addItem).not.toHaveBeenCalled();
      expect(mockLineClient.replyMessage).toHaveBeenCalledWith(
        event.replyToken,
        expect.objectContaining({
          text: expect.stringContaining('หมดสต๊อก'),
        })
      );
    });
  });
});

function createMockMessageEvent(userId: string): any {
  return {
    type: 'message',
    replyToken: 'mock-reply-token',
    source: { type: 'user', userId },
    timestamp: Date.now(),
    message: { type: 'text', text: 'test' },
  };
}

function createMockPostbackEvent(userId: string): any {
  return {
    type: 'postback',
    replyToken: 'mock-reply-token',
    source: { type: 'user', userId },
    timestamp: Date.now(),
    postback: { data: 'action=test' },
  };
}
```

---

## 7. Course Completion Summary

### 7.1 สิ่งที่เรียนรู้ตลอดหลักสูตร 100 บท

```
บทเรียนที่ 1-20: LINE OA Fundamentals
✅ LINE OA account setup และ dashboard
✅ Messaging plans และ quota
✅ Broadcast และ Auto-reply
✅ Rich Menu และ Chat features
✅ Analytics เบื้องต้น

บทเรียนที่ 21-50: LINE Messaging API
✅ Webhook events ทุกประเภท
✅ Reply, Push, Multicast, Broadcast
✅ Message types ทุกชนิด
✅ Flex Messages (Bubble, Carousel)
✅ LIFF (LINE Front-end Framework)
✅ LINE Login integration
✅ Database design และ state management

บทเรียนที่ 51-70: Advanced Features
✅ Rich Menu API
✅ Narrowcast และ Audience
✅ LINE Pay integration
✅ AI/NLP integration
✅ Real-world bots (E-commerce, Booking, Food Order)

บทเรียนที่ 71-90: Enterprise Architecture
✅ Microservices
✅ Queue management
✅ Docker และ Kubernetes
✅ CI/CD pipeline
✅ Multi-region deployment
✅ Security hardening
✅ PDPA compliance

บทเรียนที่ 91-100: Advanced Topics
✅ API Gateway
✅ Event-Driven Architecture (Kafka)
✅ GraphQL
✅ WebSocket real-time
✅ Advanced Analytics
✅ Cost Optimization
✅ Team Process
✅ Production Best Practices
```

### 7.2 Career Paths หลังจบหลักสูตร

```
Career Options:

1. LINE Bot Developer (Junior/Mid)
   ทักษะหลัก: Node.js, LINE SDK, LIFF, PostgreSQL
   เงินเดือนเฉลี่ย (Thailand): ฿30,000-60,000/month
   
   Next Steps:
   - Build portfolio (2-3 real bots)
   - LINE Certified Developer exam
   - Contribute to open source

2. Full-Stack Developer (with LINE expertise)
   ทักษะหลัก: React, Node.js, LINE APIs, GraphQL
   เงินเดือนเฉลี่ย: ฿50,000-100,000/month
   
   Next Steps:
   - เรียน React/Vue เพิ่มเติม
   - GraphQL certification
   - Build LIFF apps

3. Solutions Architect (LINE Ecosystem)
   ทักษะหลัก: Architecture, AWS/GCP, Kafka, Security
   เงินเดือนเฉลี่ย: ฿100,000-200,000/month
   
   Next Steps:
   - AWS/GCP certifications
   - System design practice
   - Lead architecture reviews

4. Tech Lead / Engineering Manager
   ทักษะหลัก: Technical + People management
   เงินเดือนเฉลี่ย: ฿150,000-300,000/month
   
   Next Steps:
   - Mentor junior developers
   - Lead team projects
   - Management skills

5. Entrepreneur / Consultant
   ทักษะ: All of the above + Business
   
   Next Steps:
   - Build your own LINE Bot SaaS
   - Offer LINE Bot consulting services
   - Create LINE Bot courses/content
```

---

## 8. Next Steps หลังจบหลักสูตร

### 8.1 สิ่งที่ควรทำทันที

```
Week 1-2: สร้าง Portfolio Project
□ เลือก idea ที่คุณสนใจจริงๆ
□ วางแผน architecture
□ เริ่ม development
□ Deploy ไปยัง production (ฟรี: Railway, Render, Vercel)

Week 3-4: LINE Certification
□ สมัคร LINE Certified Developer Program
□ ศึกษา study materials
□ ทำ practice exams
□ เข้าสอบ

Month 2-3: Community Engagement
□ Join LINE Developer Community Thailand (Facebook Group)
□ เข้าร่วม LINE Developer Meetup
□ Share งานใน LinkedIn
□ Write blog post เกี่ยวกับสิ่งที่เรียนรู้

Month 3-6: Advanced Learning
□ เลือก specialization (AI, Analytics, Enterprise)
□ เรียนทักษะเพิ่มเติมที่เกี่ยวข้อง
□ Build 2nd project ที่ complex กว่า
□ Contribute to LINE SDK open source
```

### 8.2 Resources ที่แนะนำ

```
Official Resources:
- LINE Developers: developers.line.biz
- LINE API Documentation: developers.line.biz/en/reference/messaging-api/
- LINE LIFF: developers.line.biz/en/docs/liff/overview/
- LINE YouTube: youtube.com/@LINEDevelopersTH

Communities:
- LINE Developers Thailand (Facebook Group)
- LINE Developers Slack
- GitHub: line/line-bot-sdk-nodejs
- GitHub: line/line-bot-sdk-python

Learning Platforms:
- LINE Developer Certificate Program
- Udemy (Node.js, TypeScript courses)
- freeCodeCamp
- The Odin Project

Books:
- "Designing Data-Intensive Applications" - Martin Kleppmann
- "Clean Architecture" - Robert C. Martin
- "Site Reliability Engineering" - Google
- "The Pragmatic Programmer" - Hunt & Thomas

Thai Communities:
- Creatorsgarten (Bangkok tech meetups)
- Bkk.js (JavaScript community)
- CloudNative Thailand
- DevBangkok
```

---

## 9. Final Words

```
ยินดีด้วย! 🎉

คุณได้เรียนรู้ครบทุกด้านของ LINE OA Development ตั้งแต่:
- การตั้งค่า LINE OA เบื้องต้น
- การพัฒนา LINE Bot ด้วย Node.js และ Python
- การสร้าง LIFF applications ด้วย React
- การออกแบบ architecture ระดับ Enterprise
- การจัดการ team และ process

แต่การเรียนรู้ไม่ได้จบแค่นี้

เทคโนโลยีเปลี่ยนแปลงอยู่เสมอ LINE ออก features ใหม่ทุกปี
JavaScript ecosystem พัฒนาอย่างต่อเนื่อง Cloud platforms
เพิ่ม capabilities ใหม่ทุกเดือน

สิ่งที่สำคัญที่สุดคือ:
1. Keep learning - อย่าหยุดเรียนรู้
2. Build things - ทักษะมาจากการลงมือทำ
3. Share knowledge - สิ่งที่คุณรู้อาจเปลี่ยนชีวิตคนอื่น
4. Be curious - ตั้งคำถามและหาคำตอบ
5. Have fun - สนุกกับสิ่งที่ทำ

LINE Bot ไม่ใช่แค่ technology มันคือ bridge ระหว่าง
ธุรกิจกับผู้ใช้ คุณมีพลังในการสร้างประสบการณ์ที่ดี
ให้กับผู้ใช้นับล้านคนในประเทศไทย

ขอให้โชคดีในการสร้าง LINE Bot ที่ยอดเยี่ยม! 🚀

- Team LINE OA Course
```

---

## Final Checklist: Pre-Launch สำหรับ Production

```markdown
# Final Pre-Launch Checklist

## 1. Code Quality
□ Code review completed by 2+ engineers
□ All tests passing (unit, integration, e2e)
□ Code coverage > 80%
□ No critical vulnerabilities (npm audit, snyk)
□ Dependencies up to date
□ No console.logs in production code

## 2. Security
□ LINE signature validation working
□ HTTPS certificates valid
□ Environment variables in secrets manager
□ No sensitive data in logs
□ PDPA compliance verified
□ Penetration test completed (for large systems)

## 3. Performance
□ Load test passed (2x expected traffic)
□ P95 latency < 500ms for webhooks
□ LIFF page loads < 3 seconds on 4G
□ Database queries optimized
□ Caching implemented

## 4. Monitoring
□ Health checks configured
□ Alerting rules set up
□ Dashboard accessible
□ Log aggregation working
□ Error tracking (Sentry) setup

## 5. Operations
□ Runbooks written
□ On-call rotation configured
□ Escalation paths documented
□ Rollback plan ready
□ Backup/restore tested

## 6. Business
□ LINE OA profile complete
□ Privacy policy updated
□ Terms of service updated
□ Welcome message tested
□ Rich Menu deployed
□ Customer support process ready

## 7. Team
□ All team members briefed
□ Emergency contacts list ready
□ Stakeholders notified
□ Launch communication plan ready

Sign-off Required:
□ Tech Lead: ________________
□ QA Lead: ________________
□ Security: ________________
□ Product Manager: ________________
□ Engineering Manager: ________________

Launch Date: ________________
Launch Time: ________________

🚀 ขอให้การ launch ประสบความสำเร็จ!
```

---

*ขอบคุณที่เรียนหลักสูตร LINE OA Development ครบ 100 บท ไม่ว่าคุณจะนำความรู้นี้ไปใช้ในอาชีพ สร้างธุรกิจ หรือช่วยเหลือผู้อื่น ขอให้ประสบความสำเร็จในทุกเส้นทาง 🙏*
