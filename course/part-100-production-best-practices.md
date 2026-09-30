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

---

## ภาคผนวก A: Advanced Production Patterns

### A.1 Zero-Downtime Deployment

```bash
#!/bin/bash
# zero-downtime-deploy.sh
# Blue-Green Deployment สำหรับ LINE Bot Production

set -e

NEW_VERSION="$1"
NAMESPACE="line-bot-production"
APP_NAME="line-bot-webhook"
REGISTRY="registry.example.com"

if [ -z "$NEW_VERSION" ]; then
  echo "Usage: $0 <version>"
  exit 1
fi

echo "=== Zero-Downtime Deployment: $NEW_VERSION ==="

# ตรวจสอบ active slot
CURRENT_ACTIVE=$(kubectl get service "$APP_NAME" \
  -n "$NAMESPACE" \
  -o jsonpath='{.spec.selector.slot}' 2>/dev/null || echo "blue")

DEPLOY_SLOT="green"
ROLLBACK_SLOT="blue"
if [ "$CURRENT_ACTIVE" == "green" ]; then
  DEPLOY_SLOT="blue"
  ROLLBACK_SLOT="green"
fi

echo "Active: $CURRENT_ACTIVE → Deploying to: $DEPLOY_SLOT"

# Deploy ไปยัง inactive slot
kubectl set image "deployment/$APP_NAME-$DEPLOY_SLOT" \
  "$APP_NAME=$REGISTRY/$APP_NAME:$NEW_VERSION" \
  -n "$NAMESPACE"

kubectl rollout status "deployment/$APP_NAME-$DEPLOY_SLOT" \
  -n "$NAMESPACE" --timeout=5m

# Health check
for i in {1..5}; do
  RESPONSE=$(curl -sf "http://$APP_NAME-$DEPLOY_SLOT.$NAMESPACE.svc.cluster.local/health" \
    -o /dev/null -w "%{http_code}")
  [ "$RESPONSE" == "200" ] && echo "✅ Health check $i/5" || { echo "❌ ABORT"; exit 1; }
  sleep 3
done

# Traffic switch
kubectl patch service "$APP_NAME" -n "$NAMESPACE" \
  --type merge -p "{\"spec\":{\"selector\":{\"slot\":\"$DEPLOY_SLOT\"}}}"

echo "✅ Traffic switched. Monitoring errors..."
sleep 30

# Monitor errors post-switch
ERROR_COUNT=$(kubectl logs -n "$NAMESPACE" -l "slot=$DEPLOY_SLOT" \
  --since=30s 2>/dev/null | grep -c "ERROR" || true)

if [ "$ERROR_COUNT" -gt 20 ]; then
  echo "❌ High error count ($ERROR_COUNT). Rolling back..."
  kubectl patch service "$APP_NAME" -n "$NAMESPACE" \
    --type merge -p "{\"spec\":{\"selector\":{\"slot\":\"$ROLLBACK_SLOT\"}}}"
  exit 1
fi

echo "🎉 Deployment complete! Active: $DEPLOY_SLOT, Version: $NEW_VERSION"
```

### A.2 Feature Flag System

```typescript
// Feature Flags สำหรับ Safe Deployment

import Redis from 'ioredis';

interface FeatureFlag {
  name: string;
  enabled: boolean;
  rolloutPercentage: number;
  allowlist?: string[];
  blocklist?: string[];
}

export class FeatureFlagService {
  private cache = new Map<string, { value: FeatureFlag; expiry: number }>();
  
  constructor(private redis: Redis) {}
  
  async isEnabled(flagName: string, userId?: string): Promise<boolean> {
    const flag = await this.getFlag(flagName);
    if (!flag || !flag.enabled) return false;
    
    if (userId && flag.allowlist?.includes(userId)) return true;
    if (userId && flag.blocklist?.includes(userId)) return false;
    
    if (flag.rolloutPercentage >= 100) return true;
    if (flag.rolloutPercentage <= 0) return false;
    
    if (userId) {
      const hash = this.hashString(`${userId}:${flagName}`);
      return (hash % 100) < flag.rolloutPercentage;
    }
    
    return Math.random() * 100 < flag.rolloutPercentage;
  }
  
  private async getFlag(name: string): Promise<FeatureFlag | null> {
    const cached = this.cache.get(name);
    if (cached && cached.expiry > Date.now()) return cached.value;
    
    const data = await this.redis.get(`feature:${name}`);
    if (!data) return null;
    
    const flag = JSON.parse(data) as FeatureFlag;
    this.cache.set(name, { value: flag, expiry: Date.now() + 60000 });
    return flag;
  }
  
  async setFlag(flag: FeatureFlag): Promise<void> {
    await this.redis.setex(`feature:${flag.name}`, 2592000, JSON.stringify(flag));
    this.cache.delete(flag.name);
  }
  
  private hashString(str: string): number {
    let hash = 5381;
    for (let i = 0; i < str.length; i++) {
      hash = ((hash << 5) + hash) + str.charCodeAt(i);
      hash = hash & hash;
    }
    return Math.abs(hash);
  }
}

// ใช้งานใน Webhook Handler
async function handleTextMessage(event: MessageEvent): Promise<void> {
  const userId = event.source.userId!;
  const useAIChatbot = await featureFlags.isEnabled('ai-chatbot-v2', userId);
  
  if (useAIChatbot) {
    await handleWithAI(event);
  } else {
    await handleWithRuleBased(event);
  }
}
```

### A.3 A/B Testing Framework

```typescript
// A/B Testing สำหรับ LINE Bot
// ทดสอบ message formats, CTAs, flows

interface Experiment {
  id: string;
  name: string;
  variants: Array<{
    id: string;
    name: string;
    weight: number;
    isControl: boolean;
  }>;
  status: 'running' | 'paused' | 'completed';
}

export class ABTestingService {
  constructor(private redis: Redis, private analytics: AnalyticsService) {}
  
  async getVariant(experimentId: string, userId: string): Promise<string> {
    // Check existing assignment (deterministic)
    const key = `ab:${experimentId}:${userId}`;
    const existing = await this.redis.get(key);
    if (existing) return existing;
    
    const experiment = await this.getExperiment(experimentId);
    if (!experiment || experiment.status !== 'running') return 'control';
    
    // Assign variant based on consistent hash
    const hash = this.hash(`${userId}:${experimentId}`);
    let cumulative = 0;
    let assignedVariant = experiment.variants[0].id;
    
    for (const variant of experiment.variants) {
      cumulative += variant.weight;
      if (hash % 100 < cumulative) {
        assignedVariant = variant.id;
        break;
      }
    }
    
    await this.redis.setex(key, 86400 * 30, assignedVariant);
    await this.analytics.track('ab_assignment', { userId, experimentId, variant: assignedVariant });
    
    return assignedVariant;
  }
  
  async trackConversion(
    experimentId: string,
    userId: string,
    metric: string
  ): Promise<void> {
    const variant = await this.redis.get(`ab:${experimentId}:${userId}`);
    if (!variant) return;
    
    await this.analytics.track('ab_conversion', {
      userId, experimentId, variant, metric,
      timestamp: new Date().toISOString()
    });
  }
  
  private hash(str: string): number {
    let h = 0;
    for (let i = 0; i < str.length; i++) {
      h = Math.imul(31, h) + str.charCodeAt(i) | 0;
    }
    return Math.abs(h);
  }
  
  private async getExperiment(id: string): Promise<Experiment | null> {
    const data = await this.redis.get(`experiment:${id}`);
    return data ? JSON.parse(data) : null;
  }
}

// ตัวอย่าง: ทดสอบ Welcome Message
const welcomeExperiment: Experiment = {
  id: 'welcome-msg-2024-q1',
  name: 'Welcome Message Format Test',
  status: 'running',
  variants: [
    { id: 'control', name: 'Text Only', weight: 50, isControl: true },
    { id: 'flex', name: 'Flex Message with Image', weight: 50, isControl: false }
  ]
};
```

---

## ภาคผนวก B: PDPA Compliance

### B.1 Personal Data Protection Implementation

```typescript
// PDPA (พ.ร.บ. คุ้มครองข้อมูลส่วนบุคคล) สำหรับ LINE Bot

interface ConsentRecord {
  userId: string;
  type: 'marketing' | 'analytics' | 'personalization';
  granted: boolean;
  timestamp: Date;
  policyVersion: string;
  ipAddress?: string;
}

export class PDPAService {
  constructor(private db: Pool) {}
  
  // บันทึกความยินยอม
  async recordConsent(records: ConsentRecord[]): Promise<void> {
    const client = await this.db.connect();
    try {
      await client.query('BEGIN');
      
      for (const record of records) {
        await client.query(
          `INSERT INTO consent_records 
           (user_id, consent_type, granted, granted_at, policy_version, ip_address)
           VALUES ($1, $2, $3, $4, $5, $6)
           ON CONFLICT (user_id, consent_type)
           DO UPDATE SET
             granted = EXCLUDED.granted,
             granted_at = NOW(),
             policy_version = EXCLUDED.policy_version`,
          [record.userId, record.type, record.granted,
           record.timestamp, record.policyVersion, record.ipAddress]
        );
      }
      
      await client.query('COMMIT');
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }
  
  // ตรวจสอบความยินยอม
  async hasConsent(userId: string, type: ConsentRecord['type']): Promise<boolean> {
    const result = await this.db.query(
      `SELECT granted FROM consent_records
       WHERE user_id = $1 AND consent_type = $2
       ORDER BY granted_at DESC LIMIT 1`,
      [userId, type]
    );
    return result.rows.length > 0 && result.rows[0].granted;
  }
  
  // ส่งออกข้อมูลผู้ใช้ (Right of Access - มาตรา 30)
  async exportUserData(lineUserId: string): Promise<Record<string, unknown>> {
    const [profile, orders, consents] = await Promise.all([
      this.db.query(
        'SELECT id, display_name, email, phone, created_at FROM line_users WHERE line_user_id = $1',
        [lineUserId]
      ),
      this.db.query(
        `SELECT o.order_number, o.total_amount, o.status, o.created_at
         FROM orders o JOIN line_users u ON u.id = o.user_id
         WHERE u.line_user_id = $1`,
        [lineUserId]
      ),
      this.db.query(
        'SELECT consent_type, granted, granted_at FROM consent_records WHERE user_id = $1',
        [lineUserId]
      )
    ]);
    
    return {
      exportedAt: new Date().toISOString(),
      requestedBy: lineUserId,
      profile: profile.rows[0] || null,
      orderHistory: orders.rows,
      consentHistory: consents.rows,
      retentionPolicy: 'Data retained for 3 years from last activity'
    };
  }
  
  // ลบ/Anonymize ข้อมูล (Right to Erasure - มาตรา 33)
  async deleteUserData(lineUserId: string): Promise<void> {
    const client = await this.db.connect();
    try {
      await client.query('BEGIN');
      
      // Anonymize แทนการลบเพื่อรักษา business records
      await client.query(
        `UPDATE line_users SET
           display_name = 'Deleted User',
           picture_url = NULL,
           phone = NULL,
           email = NULL,
           line_user_id = 'deleted_' || id
         WHERE line_user_id = $1`,
        [lineUserId]
      );
      
      await client.query(
        'DELETE FROM consent_records WHERE user_id = $1',
        [lineUserId]
      );
      
      // บันทึก audit log
      await client.query(
        `INSERT INTO data_deletion_audit (original_line_user_id, deleted_at, reason)
         VALUES ($1, NOW(), 'User requested deletion per PDPA')`,
        [lineUserId]
      );
      
      await client.query('COMMIT');
    } catch (err) {
      await client.query('ROLLBACK');
      throw err;
    } finally {
      client.release();
    }
  }
}
```

---

## ภาคผนวก C: Career Paths (เส้นทางอาชีพ)

### C.1 LINE Bot Developer Career Ladder

```
Career Path: LINE Bot Development
══════════════════════════════════

Junior LINE Bot Developer (0-2 ปี)
├── เงินเดือน: 25,000 - 45,000 THB/เดือน
├── ทำ well-defined tasks
├── ต้องการ guidance
└── Focus: เรียนรู้ LINE API + codebase

LINE Bot Developer (2-4 ปี)
├── เงินเดือน: 45,000 - 80,000 THB/เดือน
├── ทำงานอิสระได้
├── Mentor juniors
└── Focus: Deliver quality features

Senior LINE Bot Developer (4-7 ปี)
├── เงินเดือน: 80,000 - 150,000 THB/เดือน
├── Lead feature development
├── Drive technical decisions
└── Focus: Team productivity

Tech Lead / Staff Engineer (7+ ปี)
├── เงินเดือน: 150,000 - 250,000+ THB/เดือน
├── Cross-team impact
├── Technical strategy
└── Focus: Org-wide impact

Freelance / Consultant
├── เงินเดือน: 150,000 - 500,000+ THB/เดือน
├── หลาย clients พร้อมกัน
└── Focus: Business value delivery
```

### C.2 Portfolio Projects ที่ควรมี

```markdown
## Portfolio สำหรับ LINE Bot Developer

### Project 1: Customer Service Bot (Basic)
- Multi-step conversation flow
- FAQ automation
- Escalation to human agent
- Rich Menu navigation
**Tech:** Node.js, LINE Messaging API, Redis

### Project 2: E-Commerce Bot (Intermediate)
- Product catalog (Flex Message Carousel)
- Shopping cart
- Payment (PromptPay)
- Order tracking
**Tech:** TypeScript, PostgreSQL, React LIFF

### Project 3: Enterprise System (Advanced)
- Multi-store support
- Real-time notifications
- Analytics dashboard
- Kubernetes deployment
**Tech:** Microservices, Kubernetes, Grafana

## GitHub Portfolio Best Practices
1. README ที่ดี: demo GIF, architecture diagram, setup guide
2. Live demo link
3. Test coverage badge (Codecov)
4. CI/CD status badge
5. Clear commit history

## สิ่งที่ Recruiter มองหา
- ✅ Working demo บน LINE
- ✅ Clean, readable code
- ✅ Tests (unit + integration)
- ✅ Documentation
- ✅ Production deployment experience
```

### C.3 Certification Roadmap

```markdown
## Certification Paths ปี 2024-2025

### Q1 (Year 1): Foundation
- LINE Certified Business Manager
  ลงทะเบียน: businessmanager.line.biz
  
- AWS Cloud Practitioner (ฟรี prep materials)
  เวลาเรียน: 40-60 ชั่วโมง
  Cost: $100

### Q2 (Year 1): Development
- Node.js Application Developer (OpenJS)
  เวลาเรียน: 60-80 ชั่วโมง
  Cost: $300
  
- MongoDB Developer Associate (ฟรี!)
  university.mongodb.com

### Q3-Q4 (Year 1): Infrastructure
- Kubernetes (CKA)
  เวลาเรียน: 3-4 เดือน
  Cost: $395
  
- AWS Solutions Architect Associate
  เวลาเรียน: 3-6 เดือน
  Cost: $300

### Year 2: Specialization
- Google Cloud Professional Data Engineer
- AWS DevOps Engineer Professional
- Security certifications (CEH, OSCP)
```

---

## ภาคผนวก D: Community & Resources

### D.1 Thai Developer Communities

```markdown
## LINE Developer Communities

### Official
- LINE Developer Community TH:
  community.line.me/th
  
- LINE Developers Blog (EN):
  developers.line.biz/en/blog/
  
- LINE API Status:
  linestatus.page.link/api-status

### Facebook Groups
- LINE BOT Thailand: ~15,000+ members
  facebook.com/groups/linebotthailand
  
- LINE OA Thailand: ~25,000+ members
  facebook.com/groups/lineofficialthai

### Developer Events
- LINE Developer Meetup (รายไตรมาส)
  meetup.com/line-developers-bangkok
  
- BKK.JS (JavaScript community)
  bkkjs.github.io
  
- Thailand Tech Events
  eventpop.me (ค้นหา "developer")

### Online Learning
- LINE Developers YouTube Channel
- Udemy: "LINE Bot" courses (ตรวจสอบ rating)
- GitHub: github.com/line (official repos + examples)

## Stack Overflow Tags
- [line-messaging-api] - 2,000+ questions
- [liff] - 500+ questions
- [line-bot-sdk] - 1,500+ questions
```

### D.2 Open Source Contributions

```markdown
## เริ่มต้น Contribute ให้ LINE SDK

### Step-by-Step Guide

1. **เลือก Issue**
   github.com/line/line-bot-sdk-nodejs/issues
   Filter: label:"good first issue"

2. **Fork & Clone**
   ```bash
   git fork https://github.com/line/line-bot-sdk-nodejs
   git clone https://github.com/YOUR_NAME/line-bot-sdk-nodejs
   cd line-bot-sdk-nodejs
   npm install
   ```

3. **สร้าง Branch**
   ```bash
   git checkout -b fix/issue-123-description
   ```

4. **เขียน Fix + Test**
   - Fix the bug
   - Add/update unit test
   - Run: npm test
   - Ensure coverage doesn't drop

5. **Submit PR**
   - Clear description
   - Reference issue: "Fixes #123"
   - Follow contribution guidelines

### ประโยชน์
- GitHub profile แสดง green squares
- Build reputation ใน community
- เรียนรู้ production-grade code
- อาจได้ job offer!

### Other Projects to Contribute
- awesome-line-bot (curated resources)
- line-flex-message-designer
- Community chatbot templates
```

---

## ภาคผนวก E: สรุปทักษะทั้ง 100 บท

```markdown
# Skills Matrix - LINE OA Development (100 Parts)

## Tier 1: Core LINE Skills
| ทักษะ | บท | ระดับหลังจบ |
|------|---|------------|
| LINE Messaging API | 1-10 | Expert |
| Flex Message | 8-9, 86 | Expert |
| LIFF v2 | 18-19, 86 | Advanced |
| Rich Menu | 10, 33 | Advanced |
| LINE Login | 34 | Intermediate |
| LINE Pay | 29 | Intermediate |
| LINE Notify | 39 | Basic |
| LINE Mini App | 87 | Basic |

## Tier 2: Backend Skills
| ทักษะ | บท | ระดับหลังจบ |
|------|---|------------|
| Node.js/TypeScript | ตลอด | Advanced |
| PostgreSQL | 26, 60 | Advanced |
| Redis | 21, 53 | Advanced |
| REST API Design | 56 | Advanced |
| GraphQL | 95 | Intermediate |
| Message Queue | 52 | Intermediate |
| WebSocket | 96 | Intermediate |

## Tier 3: DevOps Skills
| ทักษะ | บท | ระดับหลังจบ |
|------|---|------------|
| Docker | 80 | Advanced |
| Kubernetes | 80, 82 | Intermediate |
| CI/CD | 79 | Advanced |
| Monitoring | 83, 97 | Intermediate |
| Security | 57, 91 | Intermediate |
| Cloud (AWS/GCP) | 82 | Intermediate |

## Tier 4: Architecture
| ทักษะ | บท | ระดับหลังจบ |
|------|---|------------|
| Microservices | 51, 81 | Intermediate |
| Event-Driven | 94 | Intermediate |
| High Availability | 90 | Intermediate |
| Multi-Tenant | 92 | Basic |
| API Gateway | 93 | Intermediate |

## Tier 5: Business & Soft Skills
| ทักษะ | บท | ระดับหลังจบ |
|------|---|------------|
| Analytics & Data | 83, 97 | Intermediate |
| Team Management | 99 | Basic |
| Production Ops | 100 | Advanced |
| PDPA Compliance | 100 | Basic |
| Cost Optimization | 98 | Basic |
```

---

## สรุป: จาก 0 ถึง Production-Ready

ตลอด 100 บทที่ผ่านมา คุณได้เดินทางจาก:

```
บทที่ 1:  "LINE OA คืออะไร?"
    ↓
บทที่ 10: "สร้าง Flex Message ได้แล้ว"
    ↓
บทที่ 30: "ระบบ Broadcast และ Segmentation"
    ↓
บทที่ 60: "Performance Optimization"
    ↓
บทที่ 80: "Kubernetes & CI/CD"
    ↓
บทที่ 90: "High Availability Design"
    ↓
บทที่ 99: "Team & Process"
    ↓
บทที่ 100: "Production Best Practices"
           "YOU ARE HERE - READY TO BUILD!"
```

**คุณตอนนี้:**
- เขียน LINE Bot ระดับ Production ได้
- ออกแบบ Scalable Architecture ได้
- Deploy บน Kubernetes ได้
- Monitor และ Respond to Incidents ได้
- สร้างและ Lead ทีมได้
- Comply กับ PDPA ได้

**คุณพร้อมแล้ว ออกไปสร้างสิ่งที่ยิ่งใหญ่!** 🚀

---

## ภาคผนวก F: Capstone Project - Restaurant Ordering System

### F.1 Project Overview

```
Multi-Store Restaurant Ordering System บน LINE OA
══════════════════════════════════════════════════

Features:
├── 🍜 Menu browsing (Flex Message Carousel)
├── 🛒 Shopping cart (Redis)
├── 💳 Payment (PromptPay QR / LINE Pay)
├── 📱 Order status real-time
├── 👨‍🍳 Kitchen display (LIFF)
├── 📊 Analytics dashboard (LIFF)
├── 🏪 Multi-store support
└── 📣 Broadcast promotions

Architecture:
     LINE App
       │
  ┌────▼────┐
  │  LIFF   │◄──── React + TypeScript
  └────┬────┘
       │ HTTPS
  ┌────▼────────────┐
  │   API Gateway   │
  └────┬────────────┘
       │
  ┌────▼────────────────────────┐
  │      Microservices           │
  │  ┌────────┐  ┌────────────┐ │
  │  │Webhook │  │Order Svc   │ │
  │  └────────┘  └────────────┘ │
  │  ┌────────┐  ┌────────────┐ │
  │  │Menu Svc│  │Payment Svc │ │
  │  └────────┘  └────────────┘ │
  │  ┌────────┐  ┌────────────┐ │
  │  │Kitchen │  │Analytics   │ │
  │  │Display │  │Service     │ │
  │  └────────┘  └────────────┘ │
  └─────────────────────────────┘
       │
  ┌────▼────────────────────────┐
  │         Data Layer           │
  │  PostgreSQL  Redis  S3       │
  └─────────────────────────────┘
```

### F.2 Database Schema (Core Tables)

```sql
-- Restaurant System Core Schema

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Stores
CREATE TABLE stores (
  id           UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  name         VARCHAR(100) NOT NULL,
  slug         VARCHAR(50) UNIQUE NOT NULL,
  line_oa_id   VARCHAR(50) UNIQUE,
  address      TEXT,
  phone        VARCHAR(20),
  is_open      BOOLEAN DEFAULT true,
  open_time    TIME DEFAULT '09:00',
  close_time   TIME DEFAULT '22:00',
  timezone     VARCHAR(50) DEFAULT 'Asia/Bangkok',
  logo_url     VARCHAR(500),
  created_at   TIMESTAMPTZ DEFAULT NOW()
);

-- Menu Categories
CREATE TABLE categories (
  id         UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  store_id   UUID REFERENCES stores(id) ON DELETE CASCADE,
  name       VARCHAR(100) NOT NULL,
  icon_emoji VARCHAR(10),
  sort_order SMALLINT DEFAULT 0,
  is_active  BOOLEAN DEFAULT true
);

-- Menu Items
CREATE TABLE menu_items (
  id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  store_id         UUID REFERENCES stores(id) ON DELETE CASCADE,
  category_id      UUID REFERENCES categories(id),
  name             VARCHAR(100) NOT NULL,
  name_en          VARCHAR(100),
  description      TEXT,
  price            NUMERIC(10,2) NOT NULL CHECK (price >= 0),
  image_url        VARCHAR(500),
  is_available     BOOLEAN DEFAULT true,
  is_recommended   BOOLEAN DEFAULT false,
  prep_time_min    SMALLINT DEFAULT 15,
  calories         SMALLINT,
  is_vegetarian    BOOLEAN DEFAULT false,
  allergens        TEXT[],
  sort_order       SMALLINT DEFAULT 0,
  created_at       TIMESTAMPTZ DEFAULT NOW(),
  updated_at       TIMESTAMPTZ DEFAULT NOW()
);

-- Line Users
CREATE TABLE line_users (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  line_user_id  VARCHAR(50) UNIQUE NOT NULL,
  display_name  VARCHAR(200),
  picture_url   VARCHAR(500),
  phone         VARCHAR(20),
  email         VARCHAR(200),
  tier          VARCHAR(20) DEFAULT 'standard',
  points        INTEGER DEFAULT 0,
  is_blocked    BOOLEAN DEFAULT false,
  created_at    TIMESTAMPTZ DEFAULT NOW(),
  last_active   TIMESTAMPTZ DEFAULT NOW()
);

-- Orders
CREATE TABLE orders (
  id               UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  store_id         UUID REFERENCES stores(id),
  user_id          UUID REFERENCES line_users(id),
  order_number     VARCHAR(20) UNIQUE NOT NULL,
  status           VARCHAR(30) NOT NULL DEFAULT 'pending_payment',
  delivery_type    VARCHAR(20) NOT NULL CHECK (delivery_type IN ('dine_in','takeaway','delivery')),
  table_number     VARCHAR(10),
  subtotal         NUMERIC(10,2) NOT NULL,
  delivery_fee     NUMERIC(10,2) DEFAULT 0,
  discount         NUMERIC(10,2) DEFAULT 0,
  total            NUMERIC(10,2) NOT NULL,
  notes            TEXT,
  est_ready_min    SMALLINT,
  confirmed_at     TIMESTAMPTZ,
  ready_at         TIMESTAMPTZ,
  completed_at     TIMESTAMPTZ,
  cancelled_at     TIMESTAMPTZ,
  cancel_reason    TEXT,
  created_at       TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_orders_store_created  ON orders(store_id, created_at DESC);
CREATE INDEX idx_orders_user           ON orders(user_id, created_at DESC);
CREATE INDEX idx_orders_status_store   ON orders(status, store_id)
  WHERE status NOT IN ('completed','cancelled');

-- Order Items
CREATE TABLE order_items (
  id            UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  order_id      UUID REFERENCES orders(id) ON DELETE CASCADE,
  menu_item_id  UUID REFERENCES menu_items(id),
  item_name     VARCHAR(100) NOT NULL,  -- snapshot at order time
  quantity      SMALLINT NOT NULL CHECK (quantity > 0),
  unit_price    NUMERIC(10,2) NOT NULL,
  total_price   NUMERIC(10,2) NOT NULL,
  notes         TEXT
);

-- Payments
CREATE TABLE payments (
  id              UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  order_id        UUID REFERENCES orders(id) UNIQUE,
  method          VARCHAR(30) NOT NULL,
  status          VARCHAR(20) NOT NULL DEFAULT 'pending',
  amount          NUMERIC(10,2) NOT NULL,
  transaction_ref VARCHAR(100),
  qr_code_url     VARCHAR(500),
  paid_at         TIMESTAMPTZ,
  created_at      TIMESTAMPTZ DEFAULT NOW()
);

-- Loyalty Points Log
CREATE TABLE points_log (
  id          UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  user_id     UUID REFERENCES line_users(id),
  order_id    UUID REFERENCES orders(id),
  delta       INTEGER NOT NULL,  -- positive = earn, negative = redeem
  balance     INTEGER NOT NULL,
  reason      VARCHAR(100),
  created_at  TIMESTAMPTZ DEFAULT NOW()
);
```

### F.3 Backend Service Implementation

```typescript
// services/MenuService.ts
import { Pool } from 'pg';
import Redis from 'ioredis';

export class MenuService {
  private readonly CACHE_TTL = 300; // 5 minutes
  
  constructor(
    private db: Pool,
    private cache: Redis
  ) {}
  
  async getStoreMenu(storeId: string): Promise<StoreMenu> {
    const cacheKey = `menu:${storeId}`;
    
    // Try cache first
    const cached = await this.cache.get(cacheKey);
    if (cached) return JSON.parse(cached);
    
    // Fetch from DB
    const [categoriesResult, itemsResult] = await Promise.all([
      this.db.query(
        `SELECT id, name, icon_emoji, sort_order
         FROM categories
         WHERE store_id = $1 AND is_active = true
         ORDER BY sort_order, name`,
        [storeId]
      ),
      this.db.query(
        `SELECT mi.id, mi.category_id, mi.name, mi.description,
                mi.price, mi.image_url, mi.is_recommended,
                mi.prep_time_min, mi.is_vegetarian, mi.allergens
         FROM menu_items mi
         WHERE mi.store_id = $1 AND mi.is_available = true
         ORDER BY mi.sort_order, mi.name`,
        [storeId]
      )
    ]);
    
    const menu: StoreMenu = {
      categories: categoriesResult.rows,
      items: itemsResult.rows,
      cachedAt: new Date().toISOString()
    };
    
    // Cache for 5 minutes
    await this.cache.setex(cacheKey, this.CACHE_TTL, JSON.stringify(menu));
    
    return menu;
  }
  
  // Invalidate cache when menu changes
  async invalidateMenuCache(storeId: string): Promise<void> {
    await this.cache.del(`menu:${storeId}`);
  }
  
  // Build Flex Message for menu carousel
  buildMenuCarousel(items: MenuItem[], storeId: string): FlexMessage {
    const bubbles = items.slice(0, 10).map(item => ({
      type: 'bubble',
      size: 'kilo',
      hero: item.image_url ? {
        type: 'image',
        url: item.image_url,
        size: 'full',
        aspectRatio: '20:13',
        aspectMode: 'cover'
      } : undefined,
      body: {
        type: 'box',
        layout: 'vertical',
        spacing: 'sm',
        contents: [
          {
            type: 'text',
            text: item.name,
            weight: 'bold',
            size: 'md',
            wrap: true
          },
          {
            type: 'text',
            text: item.description || '',
            size: 'sm',
            color: '#888888',
            wrap: true,
            maxLines: 2
          },
          {
            type: 'box',
            layout: 'baseline',
            contents: [
              {
                type: 'text',
                text: `฿${item.price.toFixed(0)}`,
                size: 'xl',
                weight: 'bold',
                color: '#06C755'
              }
            ]
          }
        ]
      },
      footer: {
        type: 'box',
        layout: 'vertical',
        contents: [{
          type: 'button',
          style: 'primary',
          color: '#06C755',
          height: 'sm',
          action: {
            type: 'postback',
            label: 'เพิ่มใส่ตะกร้า',
            data: `action=add_to_cart&itemId=${item.id}&storeId=${storeId}`
          }
        }]
      }
    }));
    
    return {
      type: 'flex',
      altText: 'เมนูอาหาร',
      contents: {
        type: 'carousel',
        contents: bubbles
      }
    };
  }
}

// services/CartService.ts
interface CartItem {
  menuItemId: string;
  name: string;
  price: number;
  quantity: number;
  notes?: string;
}

interface Cart {
  storeId: string;
  items: CartItem[];
  updatedAt: string;
}

export class CartService {
  private readonly CART_TTL = 86400; // 24 hours
  
  constructor(private cache: Redis) {}
  
  private cartKey(userId: string, storeId: string): string {
    return `cart:${userId}:${storeId}`;
  }
  
  async getCart(userId: string, storeId: string): Promise<Cart | null> {
    const data = await this.cache.get(this.cartKey(userId, storeId));
    return data ? JSON.parse(data) : null;
  }
  
  async addItem(userId: string, storeId: string, item: CartItem): Promise<Cart> {
    let cart = await this.getCart(userId, storeId) ?? {
      storeId,
      items: [],
      updatedAt: new Date().toISOString()
    };
    
    const idx = cart.items.findIndex(i => i.menuItemId === item.menuItemId);
    if (idx >= 0) {
      cart.items[idx].quantity += item.quantity;
    } else {
      cart.items.push(item);
    }
    
    cart.updatedAt = new Date().toISOString();
    await this.cache.setex(
      this.cartKey(userId, storeId),
      this.CART_TTL,
      JSON.stringify(cart)
    );
    
    return cart;
  }
  
  async removeItem(userId: string, storeId: string, menuItemId: string): Promise<Cart> {
    const cart = await this.getCart(userId, storeId);
    if (!cart) throw new Error('Cart not found');
    
    cart.items = cart.items.filter(i => i.menuItemId !== menuItemId);
    cart.updatedAt = new Date().toISOString();
    
    await this.cache.setex(
      this.cartKey(userId, storeId),
      this.CART_TTL,
      JSON.stringify(cart)
    );
    
    return cart;
  }
  
  async clearCart(userId: string, storeId: string): Promise<void> {
    await this.cache.del(this.cartKey(userId, storeId));
  }
  
  calculateTotal(cart: Cart): { subtotal: number; itemCount: number } {
    const subtotal = cart.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
    const itemCount = cart.items.reduce((sum, item) => sum + item.quantity, 0);
    return { subtotal, itemCount };
  }
  
  buildCartSummaryFlex(cart: Cart, userId: string): FlexMessage {
    const { subtotal, itemCount } = this.calculateTotal(cart);
    
    const itemRows = cart.items.map(item => ({
      type: 'box',
      layout: 'horizontal',
      contents: [
        { type: 'text', text: `${item.quantity}x ${item.name}`, flex: 3, size: 'sm', wrap: true },
        { type: 'text', text: `฿${(item.price * item.quantity).toFixed(0)}`, flex: 1, size: 'sm', align: 'end' }
      ]
    }));
    
    return {
      type: 'flex',
      altText: `ตะกร้า ${itemCount} รายการ - ฿${subtotal.toFixed(0)}`,
      contents: {
        type: 'bubble',
        header: {
          type: 'box', layout: 'vertical',
          backgroundColor: '#06C755',
          contents: [{
            type: 'text',
            text: `🛒 ตะกร้าสินค้า (${itemCount} รายการ)`,
            color: '#FFFFFF',
            weight: 'bold'
          }]
        },
        body: {
          type: 'box',
          layout: 'vertical',
          spacing: 'sm',
          contents: [
            ...itemRows,
            { type: 'separator', margin: 'md' },
            {
              type: 'box',
              layout: 'horizontal',
              margin: 'md',
              contents: [
                { type: 'text', text: 'รวมทั้งหมด', weight: 'bold', flex: 2 },
                { type: 'text', text: `฿${subtotal.toFixed(0)}`, weight: 'bold', color: '#06C755', flex: 1, align: 'end' }
              ]
            }
          ]
        },
        footer: {
          type: 'box',
          layout: 'vertical',
          spacing: 'sm',
          contents: [
            {
              type: 'button',
              style: 'primary',
              color: '#06C755',
              action: {
                type: 'postback',
                label: 'สั่งอาหาร',
                data: `action=checkout&storeId=${cart.storeId}`
              }
            },
            {
              type: 'button',
              style: 'secondary',
              action: {
                type: 'postback',
                label: 'ดูเมนูเพิ่ม',
                data: `action=view_menu&storeId=${cart.storeId}`
              }
            }
          ]
        }
      }
    };
  }
}
```

### F.4 LIFF Kitchen Display

```typescript
// liff/KitchenDisplay.tsx
// หน้าจอสำหรับ Kitchen Staff ดูออเดอร์

import React, { useEffect, useState, useCallback } from 'react';
import liff from '@line/liff';

interface Order {
  id: string;
  orderNumber: string;
  status: 'confirmed' | 'preparing' | 'ready';
  items: Array<{ name: string; quantity: number; notes?: string }>;
  deliveryType: string;
  tableNumber?: string;
  createdAt: string;
  estimatedMinutes: number;
}

const STATUS_COLORS = {
  confirmed: 'bg-yellow-100 border-yellow-400',
  preparing: 'bg-blue-100 border-blue-400',
  ready: 'bg-green-100 border-green-400'
} as const;

const STATUS_LABELS = {
  confirmed: 'รอเตรียม',
  preparing: 'กำลังทำ',
  ready: 'พร้อมเสิร์ฟ'
} as const;

export default function KitchenDisplay() {
  const [orders, setOrders] = useState<Order[]>([]);
  const [storeId, setStoreId] = useState('');
  const [loading, setLoading] = useState(true);
  const [lastUpdate, setLastUpdate] = useState(new Date());
  
  useEffect(() => {
    initKitchenDisplay();
  }, []);
  
  async function initKitchenDisplay() {
    await liff.init({ liffId: import.meta.env.VITE_KITCHEN_LIFF_ID });
    
    const params = new URLSearchParams(window.location.search);
    const sid = params.get('storeId') || '';
    setStoreId(sid);
    
    await loadOrders(sid);
    setLoading(false);
    
    // Auto-refresh every 30 seconds
    const interval = setInterval(() => loadOrders(sid), 30000);
    return () => clearInterval(interval);
  }
  
  const loadOrders = useCallback(async (sid: string) => {
    const token = await liff.getAccessToken();
    const res = await fetch(`/api/kitchen/orders?storeId=${sid}`, {
      headers: { Authorization: `Bearer ${token}` }
    });
    const data = await res.json();
    setOrders(data.orders || []);
    setLastUpdate(new Date());
  }, []);
  
  async function updateOrderStatus(orderId: string, newStatus: string) {
    const token = await liff.getAccessToken();
    await fetch(`/api/kitchen/orders/${orderId}/status`, {
      method: 'PUT',
      headers: {
        'Content-Type': 'application/json',
        Authorization: `Bearer ${token}`
      },
      body: JSON.stringify({ status: newStatus })
    });
    await loadOrders(storeId);
  }
  
  if (loading) return (
    <div className="flex items-center justify-center h-screen bg-gray-900">
      <div className="text-white text-xl">Loading Kitchen Display...</div>
    </div>
  );
  
  const columns = {
    confirmed: orders.filter(o => o.status === 'confirmed'),
    preparing: orders.filter(o => o.status === 'preparing'),
    ready: orders.filter(o => o.status === 'ready')
  };
  
  return (
    <div className="min-h-screen bg-gray-900 p-4">
      <div className="flex justify-between items-center mb-4">
        <h1 className="text-white text-2xl font-bold">Kitchen Display</h1>
        <span className="text-gray-400 text-sm">
          Updated: {lastUpdate.toLocaleTimeString('th-TH')}
        </span>
      </div>
      
      <div className="grid grid-cols-3 gap-4">
        {(Object.keys(columns) as Array<keyof typeof columns>).map(status => (
          <div key={status} className="space-y-3">
            <div className="text-center">
              <span className={`inline-block px-3 py-1 rounded-full text-sm font-bold
                ${status === 'confirmed' ? 'bg-yellow-500 text-black' :
                  status === 'preparing' ? 'bg-blue-500 text-white' : 'bg-green-500 text-white'}`}>
                {STATUS_LABELS[status]} ({columns[status].length})
              </span>
            </div>
            
            {columns[status].map(order => (
              <div key={order.id}
                   className={`rounded-lg border-2 p-3 ${STATUS_COLORS[status]}`}>
                <div className="flex justify-between items-start mb-2">
                  <span className="font-bold text-lg"># {order.orderNumber}</span>
                  {order.tableNumber && (
                    <span className="bg-gray-800 text-white px-2 py-0.5 rounded text-sm">
                      โต๊ะ {order.tableNumber}
                    </span>
                  )}
                </div>
                
                <ul className="space-y-1 mb-3">
                  {order.items.map((item, i) => (
                    <li key={i} className="text-sm">
                      <span className="font-bold">{item.quantity}x</span> {item.name}
                      {item.notes && <span className="text-gray-600 ml-1">({item.notes})</span>}
                    </li>
                  ))}
                </ul>
                
                {status === 'confirmed' && (
                  <button
                    onClick={() => updateOrderStatus(order.id, 'preparing')}
                    className="w-full bg-blue-500 text-white py-2 rounded font-bold">
                    เริ่มทำ
                  </button>
                )}
                {status === 'preparing' && (
                  <button
                    onClick={() => updateOrderStatus(order.id, 'ready')}
                    className="w-full bg-green-500 text-white py-2 rounded font-bold">
                    พร้อมแล้ว
                  </button>
                )}
                {status === 'ready' && (
                  <button
                    onClick={() => updateOrderStatus(order.id, 'delivered')}
                    className="w-full bg-gray-500 text-white py-2 rounded font-bold">
                    ส่งแล้ว
                  </button>
                )}
              </div>
            ))}
          </div>
        ))}
      </div>
    </div>
  );
}
```

### F.5 Deployment สำหรับ Capstone Project

```yaml
# docker-compose.yml - Local Development

version: '3.9'

services:
  webhook:
    build:
      context: ./services/webhook
      dockerfile: Dockerfile
    ports:
      - "3001:3000"
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://admin:password@postgres:5432/restaurant
      - REDIS_URL=redis://redis:6379
      - LINE_CHANNEL_ACCESS_TOKEN=${LINE_CHANNEL_ACCESS_TOKEN}
      - LINE_CHANNEL_SECRET=${LINE_CHANNEL_SECRET}
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - ./services/webhook/src:/app/src
    command: npm run dev

  api:
    build:
      context: ./services/api
    ports:
      - "3002:3000"
    environment:
      - DATABASE_URL=postgresql://admin:password@postgres:5432/restaurant
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=${JWT_SECRET}
    depends_on:
      - postgres
      - redis
    volumes:
      - ./services/api/src:/app/src
    command: npm run dev

  liff:
    build:
      context: ./liff
    ports:
      - "3000:3000"
    environment:
      - VITE_API_URL=http://localhost:3002
      - VITE_LIFF_ID=${LIFF_ID}
      - VITE_KITCHEN_LIFF_ID=${KITCHEN_LIFF_ID}
    volumes:
      - ./liff/src:/app/src
    command: npm run dev

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=restaurant
      - POSTGRES_USER=admin
      - POSTGRES_PASSWORD=password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin -d restaurant"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - webhook
      - api
      - liff

volumes:
  postgres_data:
  redis_data:
```

```dockerfile
# Dockerfile สำหรับ Production

FROM node:20-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm ci --only=production

# Copy source
COPY src ./src
COPY tsconfig.json ./

# Build
RUN npm run build

# Production stage
FROM node:20-alpine AS production

RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy built files
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
COPY package.json ./

# Security: run as non-root
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=30s --retries=3 \
  CMD wget -qO- http://localhost:3000/health || exit 1

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

---

## ภาคผนวก G: Production Readiness Final Scorecard

```markdown
# Final Production Readiness Assessment

ใช้ scorecard นี้ก่อน launch ทุกครั้ง
คะแนนแต่ละข้อ: 0=ยังไม่ทำ, 1=ทำบางส่วน, 2=ทำครบ

## Domain 1: Architecture (max 16 points)
□ Horizontal scaling ready       __/2
□ No single point of failure     __/2
□ Circuit breakers implemented   __/2
□ Health check endpoints ready   __/2
□ Graceful shutdown handled      __/2
□ Auto-scaling configured        __/2
□ Backup & restore tested        __/2
□ DR procedure documented        __/2
                    Subtotal: __/16

## Domain 2: Security (max 20 points)
□ LINE signature validation      __/2
□ Input validation all endpoints __/2
□ SQL injection prevented        __/2
□ Secrets in vault/env only      __/2
□ TLS 1.2+ enforced             __/2
□ Security headers set           __/2
□ Rate limiting enabled          __/2
□ Container scan passed          __/2
□ PDPA consent implemented       __/2
□ Penetration test done          __/2
                    Subtotal: __/20

## Domain 3: Performance (max 12 points)
□ P95 response time < 200ms     __/2
□ Load test 3x traffic passed    __/2
□ Database queries optimized     __/2
□ Caching implemented            __/2
□ CDN for static assets          __/2
□ No N+1 query problems          __/2
                    Subtotal: __/12

## Domain 4: Monitoring (max 12 points)
□ Metrics exposed & scraped      __/2
□ Dashboards created             __/2
□ P1 alerts → phone call         __/2
□ P2 alerts → messaging          __/2
□ Centralized logging            __/2
□ Error tracking (Sentry)        __/2
                    Subtotal: __/12

## Domain 5: Operations (max 10 points)
□ Runbook documented             __/2
□ On-call rotation set           __/2
□ Rollback tested                __/2
□ Incident process defined       __/2
□ Post-mortem template ready     __/2
                    Subtotal: __/10

## Domain 6: Testing (max 10 points)
□ Unit coverage > 80%            __/2
□ Integration tests pass         __/2
□ E2E critical paths covered     __/2
□ Load testing passed            __/2
□ Regression suite automated     __/2
                    Subtotal: __/10

## Total Score: __/80

## Scoring Guide
- 72-80 (90%+)    🟢 LAUNCH READY
- 60-71 (75-89%)  🟡 MOSTLY READY - fix gaps first
- 40-59 (50-74%)  🔴 NOT READY - significant work needed
- < 40  (< 50%)   ⛔ DO NOT LAUNCH

## Non-Negotiables (must be 2/2)
These 5 items must be perfect before any launch:
1. LINE Webhook Signature Validation
2. Input Validation on all endpoints
3. Secrets not in source code
4. Health check endpoints working
5. Rollback procedure tested and documented
```

---

*🎉 จบแล้ว! Course LINE OA Development - 100 Parts Complete!*

*"The best time to plant a tree was 20 years ago. The second best time is now."*

*"Code is like humor. When you have to explain it, it's bad." - Cory House*

*"Programs must be written for people to read, and only incidentally for machines to execute." - Harold Abelson*

*ออกไปสร้าง LINE Bot ที่ดีที่สุดในประเทศไทยได้เลย! 🇹🇭*

*— Team LINE OA Course*
