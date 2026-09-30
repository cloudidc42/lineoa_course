# Part 99: Team Structure และ Process สำหรับ LINE Bot Development

## บทนำ

การพัฒนา LINE Bot ในระดับ Production ต้องการ team ที่มีโครงสร้างชัดเจน process ที่ดี และ culture ที่เหมาะสม บทนี้ครอบคลุมทุกด้านของการบริหาร team LINE Bot ตั้งแต่การสรรหาบุคลากรจนถึง incident management

---

## 1. Team Structure สำหรับ LINE Bot

### 1.1 โครงสร้าง Team ตามขนาดธุรกิจ

#### Startup (1-3 คน)
```
Tech Lead / Full-Stack Developer
├── Backend + LINE Bot API
├── LIFF Frontend
├── DevOps (basic)
└── Analytics (basic)

ความรับผิดชอบ:
- พัฒนาและ maintain ทุกส่วน
- ตอบสนอง incidents
- วางแผน roadmap
```

#### SME (5-10 คน)
```
Engineering Manager
├── Backend Team (2-3 คน)
│   ├── Senior Backend Developer
│   └── Backend Developer x 1-2
├── Frontend/LIFF Team (1-2 คน)
│   └── Frontend Developer
├── DevOps/Platform (1 คน)
│   └── DevOps Engineer
└── QA (1 คน)
    └── QA Engineer
```

#### Enterprise (10+ คน)
```
Head of Engineering
├── Platform Team (3-4 คน)
│   ├── Platform Lead
│   ├── DevOps Engineer x 2
│   └── Security Engineer
├── LINE Bot Team (4-5 คน)
│   ├── Tech Lead
│   ├── Senior Backend Developer x 2
│   └── Backend Developer
├── LIFF/Frontend Team (2-3 คน)
│   ├── Frontend Lead
│   └── Frontend Developer x 1-2
├── Data Team (2-3 คน)
│   ├── Data Engineer
│   └── Data Analyst
└── QA Team (2 คน)
    ├── QA Lead
    └── QA Engineer
```

### 1.2 Roles และ Responsibilities

#### LINE Bot Tech Lead
```
ความรับผิดชอบหลัก:
□ Architecture decisions สำหรับ LINE Bot system
□ Code review และ technical standards
□ Mentoring junior developers
□ Performance optimization
□ Incident escalation และ resolution
□ Technical roadmap

ทักษะที่ต้องการ:
- LINE Messaging API expertise
- Node.js / Python backend
- Database design (PostgreSQL, Redis)
- Message broker (Kafka/RabbitMQ)
- Docker / Kubernetes
- Monitoring (Prometheus, Grafana)

KPIs:
- System uptime > 99.9%
- Mean Time to Recovery < 30 minutes
- Code coverage > 80%
- Sprint velocity improvement 10%/quarter
```

#### Backend Developer (LINE Bot)
```
ความรับผิดชอบหลัก:
□ พัฒนา webhook handlers
□ LINE API integration (Push, Reply, Broadcast)
□ LIFF backend APIs
□ Database operations
□ Unit tests

ทักษะที่ต้องการ:
- Node.js หรือ Python
- REST API / GraphQL
- LINE SDK
- PostgreSQL / MySQL
- Redis
- Git

Level ที่เหมาะสม:
- Junior: 0-2 ปี experience
- Mid: 2-5 ปี
- Senior: 5+ ปี
```

#### LIFF Frontend Developer
```
ความรับผิดชอบหลัก:
□ พัฒนา LIFF applications
□ LINE Login integration
□ Flex Message design
□ Rich Menu design
□ UI/UX optimization สำหรับ mobile

ทักษะที่ต้องการ:
- React.js / Vue.js
- LIFF SDK
- TypeScript
- CSS / Tailwind
- Design tools (Figma)
```

#### DevOps Engineer
```
ความรับผิดชอบหลัก:
□ CI/CD pipeline management
□ Kubernetes cluster management
□ Monitoring และ alerting setup
□ Security patching
□ Performance tuning
□ On-call rotation

ทักษะที่ต้องการ:
- Docker / Kubernetes
- Terraform / Ansible
- AWS / GCP
- CI/CD (GitHub Actions, GitLab CI)
- Monitoring (Prometheus, Grafana, ELK)
- Security (WAF, SAST, DAST)
```

---

## 2. Development Workflow

### 2.1 Git Flow สำหรับ LINE Bot

```
Branch Strategy:

main (production)
  └── release/v1.2.0
        └── develop
              ├── feature/new-order-flex-message
              ├── feature/line-pay-integration
              └── bugfix/webhook-timeout-fix

Branch Naming Convention:
- feature/[ticket-id]-[short-description]
- bugfix/[ticket-id]-[short-description]
- hotfix/[ticket-id]-[short-description]
- release/v[major].[minor].[patch]
- chore/[description]

ตัวอย่าง:
- feature/LINE-123-add-order-tracking
- bugfix/LINE-456-fix-reply-token-expired
- hotfix/LINE-789-critical-webhook-failure
```

```bash
#!/bin/bash
# scripts/git-workflow.sh

# สร้าง feature branch
git_start_feature() {
  TICKET=$1
  DESCRIPTION=$2
  
  if [ -z "$TICKET" ] || [ -z "$DESCRIPTION" ]; then
    echo "Usage: git_start_feature LINE-123 my-feature-name"
    return 1
  fi
  
  git checkout develop
  git pull origin develop
  git checkout -b "feature/${TICKET}-${DESCRIPTION}"
  echo "✅ Feature branch created: feature/${TICKET}-${DESCRIPTION}"
}

# Finish feature (merge back to develop)
git_finish_feature() {
  BRANCH=$(git branch --show-current)
  
  # Run tests before merge
  npm test
  if [ $? -ne 0 ]; then
    echo "❌ Tests failed! Fix before merging."
    return 1
  fi
  
  git checkout develop
  git pull origin develop
  git merge --no-ff "$BRANCH" -m "Merge ${BRANCH} into develop"
  git push origin develop
  git branch -d "$BRANCH"
  echo "✅ Feature merged to develop"
}

# สร้าง release
git_start_release() {
  VERSION=$1
  
  git checkout develop
  git pull origin develop
  git checkout -b "release/v${VERSION}"
  
  # อัพเดท version
  npm version $VERSION --no-git-tag-version
  git add package.json package-lock.json
  git commit -m "chore: bump version to ${VERSION}"
  
  echo "✅ Release branch created: release/v${VERSION}"
  echo "ทำ final testing แล้ว merge ไปยัง main"
}
```

### 2.2 Trunk Based Development

```
สำหรับ team ที่ต้องการ deploy บ่อย:

main (production-ready always)
  ├── feature/xyz (short-lived, max 1-2 days)
  └── feature/abc

Rules:
1. Branch อายุไม่เกิน 2 วัน
2. Merge ต้อง pass CI/CD ทั้งหมด
3. ใช้ Feature Flags สำหรับ incomplete features
4. Deploy to production อย่างน้อยทุกวัน

Feature Flags:
```

```typescript
// feature-flags/flags.ts
export const FeatureFlags = {
  NEW_ORDER_FLOW: process.env.FF_NEW_ORDER_FLOW === 'true',
  LINE_PAY_V2: process.env.FF_LINE_PAY_V2 === 'true',
  AI_CHATBOT: process.env.FF_AI_CHATBOT === 'true',
  SUBSCRIPTION_MESSAGES: process.env.FF_SUBSCRIPTION_MESSAGES === 'true',
};

// ใน code
if (FeatureFlags.NEW_ORDER_FLOW) {
  return handleNewOrderFlow(event);
} else {
  return handleLegacyOrderFlow(event);
}
```

---

## 3. Sprint Planning สำหรับ LINE Features

### 3.1 Sprint Planning Template

```markdown
# Sprint [N] Planning - LINE OA Team
Date: [DD/MM/YYYY]
Sprint Duration: 2 weeks ([Start] - [End])

## Sprint Goal
"[เป้าหมายหลักของ sprint นี้ใน 1-2 ประโยค]"
ตัวอย่าง: "Deploy ระบบ Order Tracking ผ่าน LINE ให้ผู้ใช้ติดตาม status ได้ใน real-time"

## Team Capacity
| Member | Days Available | Notes |
|--------|---------------|-------|
| สมชาย (TL) | 10 | - |
| สมหญิง (Dev) | 9 | วันหยุด 1 วัน |
| สมศักดิ์ (LIFF) | 10 | - |
| สมพร (DevOps) | 8 | ประชุม 2 วัน |
| สมใจ (QA) | 10 | - |

Total Story Points Capacity: 35-40 points

## Selected Stories

### Must Have (This Sprint)
| Story | Points | Owner | Dependencies |
|-------|--------|-------|-------------|
| LINE-201: Webhook handler for order events | 5 | สมชาย | - |
| LINE-202: Push notification on status change | 3 | สมชาย | LINE-201 |
| LINE-203: Order tracking LIFF page | 8 | สมศักดิ์ | LINE-201 |
| LINE-204: Status change Flex message design | 5 | สมศักดิ์ | - |
| LINE-205: Integration tests | 5 | สมใจ | LINE-201,202,203 |

Subtotal: 26 points

### Nice to Have
| Story | Points | Owner |
|-------|--------|-------|
| LINE-206: Delivery map integration | 8 | สมชาย |
| LINE-207: SMS fallback notification | 5 | สมหญิง |

### Sprint Risks
1. LINE API rate limiting during load testing
2. สมหญิงลางาน 1 วัน
3. Deploy window ต้องรอ approval จาก Infra

## Definition of Done
□ Unit tests ผ่าน (coverage > 80%)
□ Integration tests ผ่าน
□ Code review โดย 2 คน
□ Documentation อัพเดท
□ QA sign-off
□ Deploy to staging ผ่าน
□ Performance test ผ่าน (P95 < 500ms)
```

### 3.2 Story Point Guidelines

```
Story Points สำหรับ LINE Bot Features:

1 point: แก้ไข text/copy ใน message template
2 points: เพิ่ม button ใน Flex message
3 points: เพิ่ม auto-reply rule ใหม่
5 points: สร้าง webhook handler ใหม่
8 points: พัฒนา LIFF page ใหม่
13 points: Integration กับ external service ใหม่
21 points: Feature ขนาดใหญ่ (ควรแบ่งย่อย)

Poker Planning Sessions:
- ประชุมทุก Sprint Planning
- ใช้ Fibonacci sequence: 1, 2, 3, 5, 8, 13, 21
- ถ้าต่างกันมาก ต้องอธิบายก่อน estimate ใหม่
```

---

## 4. Code Review Checklist

### 4.1 LINE Bot Code Review Checklist

```markdown
## Code Review Checklist - LINE Bot

### General
- [ ] Code อ่านเข้าใจง่าย มี comments ที่จำเป็น
- [ ] ไม่มี hardcoded values (ใช้ env vars หรือ config)
- [ ] Error handling ครอบคลุม
- [ ] Logging เหมาะสม (ไม่มาก/น้อยเกินไป)
- [ ] ไม่มี console.log ที่ไม่จำเป็น

### Security
- [ ] ไม่มี secrets/credentials ใน code
- [ ] Validate LINE signature ก่อนประมวลผล
- [ ] Input validation และ sanitization
- [ ] ไม่มี SQL injection vulnerabilities
- [ ] Sensitive data ไม่ถูก log

### LINE API Specific
- [ ] ตรวจสอบ replyToken expiry (30 วินาที)
- [ ] Handle rate limits ด้วย retry logic
- [ ] Message format ถูกต้องตาม LINE spec
- [ ] Webhook response ส่งภายใน 200ms
- [ ] Error messages เหมาะสมสำหรับ users

### Performance
- [ ] ไม่มี N+1 query problems
- [ ] Database queries ใช้ indexes
- [ ] Async operations ใช้ Promise.all() เมื่อทำได้
- [ ] Cache ใช้ที่เหมาะสม
- [ ] Memory leaks ไม่มี

### Testing
- [ ] Unit tests ครอบคลุม business logic
- [ ] Integration tests สำหรับ LINE API calls
- [ ] Edge cases ถูก test
- [ ] Mock LINE API ใน tests อย่างถูกต้อง

### LIFF Specific
- [ ] LIFF SDK initialized ก่อนใช้งาน
- [ ] Handle LIFF errors gracefully
- [ ] Responsive design ทำงานบน mobile
- [ ] LINE Login flow ถูกต้อง

### Documentation
- [ ] README อัพเดทถ้าจำเป็น
- [ ] API documentation อัพเดท
- [ ] CHANGELOG เพิ่ม entry
```

### 4.2 PR Template

```markdown
<!-- .github/pull_request_template.md -->
## สรุปการเปลี่ยนแปลง
[อธิบายสั้นๆ ว่าทำอะไร]

## ประเภทการเปลี่ยนแปลง
- [ ] 🐛 Bug fix
- [ ] ✨ New feature
- [ ] 🔨 Refactoring
- [ ] 📝 Documentation
- [ ] 🔧 Configuration change
- [ ] 🚀 Performance improvement

## Ticket Reference
LINE-[ticket number]: [link]

## การทดสอบ
- [ ] Unit tests ผ่าน
- [ ] Integration tests ผ่าน
- [ ] Manual testing ใน LINE App
- [ ] Tested on iOS
- [ ] Tested on Android

## Screenshots (ถ้ามี UI changes)
| Before | After |
|--------|-------|
| [screenshot] | [screenshot] |

## Checklist
- [ ] Self-review code แล้ว
- [ ] อัพเดท documentation
- [ ] ไม่มี breaking changes (หรือระบุ migration guide)
- [ ] ไม่มี sensitive data ใน code

## Notes for Reviewers
[สิ่งที่อยากให้ reviewer ดูเป็นพิเศษ]
```

---

## 5. Documentation Standards

### 5.1 LINE Bot Documentation Structure

```
docs/
├── README.md               - Overview และ Quick Start
├── architecture/
│   ├── overview.md         - Architecture diagram และ description
│   ├── database.md         - Database schema
│   └── kafka-topics.md     - Event topics design
├── api/
│   ├── webhook-events.md   - Webhook event types
│   ├── line-api.md         - LINE API endpoints ที่ใช้
│   └── internal-api.md     - Internal REST/GraphQL API
├── development/
│   ├── setup.md            - Local development setup
│   ├── testing.md          - Testing guide
│   └── debugging.md        - Debugging tips
├── deployment/
│   ├── staging.md          - Deploy to staging
│   ├── production.md       - Deploy to production
│   └── rollback.md         - Rollback procedures
├── operations/
│   ├── monitoring.md       - Monitoring setup
│   ├── alerts.md           - Alert playbooks
│   └── runbook.md          - Operational runbook
└── line-specific/
    ├── flex-messages.md    - Flex message templates
    ├── rich-menus.md       - Rich menu management
    └── liff-apps.md        - LIFF app documentation
```

### 5.2 Webhook Handler Documentation Template

```typescript
/**
 * Handles incoming LINE follow events
 * 
 * @description
 * เมื่อผู้ใช้ follow LINE OA นี้ handler นี้จะ:
 * 1. สร้าง/อัพเดท user profile ใน database
 * 2. ส่ง welcome message
 * 3. บันทึก analytics event
 * 4. Trigger onboarding sequence ถ้าเป็น new user
 * 
 * @param event - LINE FollowEvent object
 * @param context - Request context (userId, correlationId)
 * 
 * @example
 * // LINE webhook payload:
 * // {
 * //   "type": "follow",
 * //   "source": { "type": "user", "userId": "U..." },
 * //   "timestamp": 1234567890
 * // }
 * 
 * @throws {DatabaseError} เมื่อไม่สามารถบันทึกข้อมูลได้
 * @throws {LineApiError} เมื่อส่ง welcome message ไม่ได้
 * 
 * @see {@link https://developers.line.biz/en/reference/messaging-api/#follow-event}
 */
export async function handleFollowEvent(
  event: FollowEvent,
  context: RequestContext
): Promise<void> {
  // Implementation
}
```

### 5.3 Architecture Decision Records (ADR)

```markdown
# ADR-001: ใช้ Apache Kafka สำหรับ LINE Event Processing

Date: 2024-01-15
Status: Accepted
Deciders: Tech Lead, Engineering Manager

## Context
LINE Bot ของเราต้องการประมวลผล events จาก webhook อย่างน้อย 100 events/second
และต้องการส่งต่อให้ services อื่นๆ (Analytics, CRM, Notification)

## Decision
ใช้ Apache Kafka เป็น event broker

## Alternatives Considered

### Option 1: Direct API Calls (Rejected)
- ❌ Tight coupling ระหว่าง services
- ❌ ถ้า service หนึ่งล่ม จะกระทบทั้งหมด
- ❌ ยาก scale

### Option 2: RabbitMQ (Rejected)
- ❌ ไม่มี event replay capability
- ❌ ไม่เหมาะกับ high-throughput
- ✅ ง่ายกว่า Kafka

### Option 3: AWS SQS/SNS
- ✅ Managed service
- ❌ Vendor lock-in
- ❌ ต้นทุนสูงกว่า

### Option 4: Apache Kafka (Chosen)
- ✅ High throughput (millions/day)
- ✅ Event replay สำหรับ debugging
- ✅ Multiple consumer groups
- ✅ Durable message storage
- ❌ ซับซ้อนกว่า
- ❌ ต้องการ expertise

## Consequences
- ต้องการ Kafka cluster (3 brokers)
- Team ต้องเรียน Kafka
- Latency เพิ่มขึ้นเล็กน้อย (~5ms)
- ได้ event sourcing capability

## Status Update
2024-03-01: Implemented และ stable ใน production
```

---

## 6. On-Call Runbook

### 6.1 On-Call Responsibilities

```
On-Call Engineer Responsibilities:

Primary On-Call:
- ตอบสนอง P1 alerts ภายใน 5 นาที
- ตอบสนอง P2 alerts ภายใน 15 นาที
- ตอบสนอง P3 alerts ภายใน 1 ชั่วโมง
- Update status page เมื่อมี incident
- Escalate ถ้าไม่สามารถแก้ไขได้ใน 30 นาที

Secondary On-Call:
- Backup สำหรับ Primary
- ช่วย escalation paths
- Review postmortems

On-Call Schedule:
- หมุนเวียนทุกสัปดาห์
- Handoff ทุกวันจันทร์ 09:00
- ห้ามมี on-call นานเกิน 2 สัปดาห์ติดต่อกัน
```

### 6.2 Alert Priority Levels

```yaml
# alerts/priority-definitions.yml

P1 - Critical (ตอบภายใน 5 นาที):
  - LINE Webhook endpoint down
  - Database connection failed
  - Payment processing failed
  - Error rate > 10%
  - Data loss detected
  actions:
    - Wake up on-call immediately
    - Page manager
    - Start incident bridge

P2 - High (ตอบภายใน 15 นาที):
  - Response time > 2 seconds (P95)
  - Error rate 5-10%
  - Kafka consumer lag > 10,000
  - Disk usage > 90%
  - LINE API quota > 90%
  actions:
    - Notify on-call
    - Start investigation
    - Post in #incidents

P3 - Medium (ตอบภายใน 1 ชั่วโมง):
  - Response time > 1 second (P95)
  - Error rate 1-5%
  - Memory usage > 80%
  - Kafka consumer lag > 1,000
  actions:
    - Notify on-call during business hours
    - Create ticket

P4 - Low (ตอบใน next business day):
  - Non-critical service degradation
  - Warning thresholds hit
  - Non-impacting anomalies
```

### 6.3 Runbook Templates

```markdown
# Runbook: LINE Webhook Down

## Symptoms
- Webhook health check failing
- LINE messages ไม่ได้รับคำตอบ
- Error rate spike ใน Grafana

## Impact
- Users ไม่สามารถใช้ LINE Bot ได้
- Auto-reply ไม่ทำงาน
- Revenue impact (ถ้า chatbot-driven sales)

## Investigation Steps

### Step 1: Check Application Status
```bash
# ตรวจสอบ pods
kubectl get pods -n line-bot

# ดู logs
kubectl logs -n line-bot -l app=webhook-handler --tail=100

# ตรวจสอบ errors
kubectl logs -n line-bot -l app=webhook-handler --tail=100 | grep ERROR
```

### Step 2: Check Health Endpoints
```bash
# Health check
curl https://api.yourapp.com/health

# LINE webhook endpoint
curl -X POST https://api.yourapp.com/webhook \
  -H "Content-Type: application/json" \
  -d '{"events":[]}'
```

### Step 3: Check Database
```bash
# Connection test
psql $DATABASE_URL -c "SELECT 1"

# Check connections
psql $DATABASE_URL -c "SELECT count(*) FROM pg_stat_activity"
```

### Step 4: Check Infrastructure
```bash
# Load balancer health
aws elbv2 describe-target-health --target-group-arn $TG_ARN

# ECS service
aws ecs describe-services --cluster line-bot --services webhook-handler
```

## Resolution Steps

### If Application Crash
1. Restart pods: `kubectl rollout restart deployment/webhook-handler -n line-bot`
2. ถ้ายังไม่ดี: Rollback `kubectl rollout undo deployment/webhook-handler -n line-bot`

### If Database Issue
1. Check RDS status ใน AWS Console
2. ถ้า connection pool เต็ม: restart application
3. ถ้า RDS down: failover ไป Read Replica

### If High Memory
1. Scale up: `kubectl scale deployment/webhook-handler --replicas=5 -n line-bot`
2. ดู memory leak patterns ใน logs

## Escalation
- ไม่สามารถแก้ไขได้ใน 30 นาที → Escalate to Tech Lead
- Database issue → Escalate to DBA
- Infrastructure issue → Escalate to DevOps

## Post-Incident
- สร้าง Postmortem ภายใน 48 ชั่วโมง
- Update runbook ถ้าขั้นตอนต้องแก้ไข
```

---

## 7. Incident Management

### 7.1 Incident Response Procedure

```
Incident Response Process:

1. Detection (0-5 min)
   - Alert triggered โดย monitoring
   - User report ผ่าน support channel
   - Team member ค้นพบ

2. Triage (5-15 min)
   - On-call ประเมิน impact
   - Assign severity level (P1-P4)
   - สร้าง incident ticket
   - Notify stakeholders

3. Investigation (15-60 min)
   - รวบรวม evidence (logs, metrics)
   - ระบุ root cause
   - หา workaround/fix

4. Mitigation (30-120 min)
   - Deploy fix หรือ workaround
   - Verify resolution
   - Monitor for recurrence

5. Resolution
   - Confirm service restored
   - Update status page
   - Notify stakeholders

6. Postmortem (within 48h)
   - Timeline of events
   - Root cause analysis
   - Action items ป้องกันในอนาคต
```

### 7.2 Communication Templates

```
P1 Incident - Initial Notification:
"🚨 [P1 INCIDENT] LINE Bot webhook down
Time: [HH:MM]
Impact: Users ไม่สามารถใช้ LINE Bot ได้
IC (Incident Commander): [ชื่อ]
Status: Investigating
Next update: +15 minutes
#incidents"

P1 Incident - Update:
"🔄 [P1 UPDATE] LINE Bot webhook
Time: [HH:MM]
Root Cause: [สาเหตุที่พบ]
Action: [สิ่งที่กำลังทำ]
ETA: [เวลาที่คาดว่าจะแก้ได้]
Next update: +15 minutes"

P1 Incident - Resolution:
"✅ [P1 RESOLVED] LINE Bot webhook
Time: [HH:MM]
Resolution: [วิธีแก้ไข]
Duration: [X] minutes
Users affected: [จำนวนโดยประมาณ]
Postmortem: Coming within 48 hours"
```

### 7.3 Postmortem Template

```markdown
# Postmortem: [ชื่อ Incident]
Date: [DD/MM/YYYY]
Severity: P[N]
Duration: [X] minutes
Author: [ชื่อ]
Reviewers: [ชื่อ]

## Executive Summary
[สรุปสั้นๆ 2-3 ประโยค สำหรับผู้บริหาร]

## Impact
- ผู้ใช้ที่ได้รับผลกระทบ: ~[จำนวน] คน
- Duration: [X] นาที
- Revenue impact: ~฿[จำนวน] (ถ้าประมาณได้)
- Features ที่ไม่ทำงาน: [รายการ]

## Timeline
| Time | Event |
|------|-------|
| HH:MM | Alert triggered |
| HH:MM | On-call notified |
| HH:MM | Investigation started |
| HH:MM | Root cause identified |
| HH:MM | Fix deployed |
| HH:MM | Service restored |
| HH:MM | Incident closed |

## Root Cause Analysis

### What happened?
[อธิบาย technical details]

### Why did it happen?
[5 Whys analysis]
Why 1: [...]
Why 2: [...]
Why 3: [...]
Why 4: [...]
Why 5: Root cause

## Contributing Factors
1. [Factor 1]
2. [Factor 2]

## What Went Well
- [สิ่งที่ทำได้ดี]
- Detection time รวดเร็ว

## What Could Be Improved
- [สิ่งที่ควรปรับปรุง]

## Action Items
| Action | Owner | Due Date | Priority |
|--------|-------|----------|----------|
| เพิ่ม alert สำหรับ X | สมชาย | 2024-02-01 | P1 |
| Update runbook | สมหญิง | 2024-02-07 | P2 |
| Implement circuit breaker | Tech Lead | 2024-02-14 | P2 |

## Lessons Learned
[Key takeaways สำหรับ team]
```

---

## 8. KPIs สำหรับ LINE OA Team

### 8.1 Technical KPIs

```typescript
// kpis/technical-kpis.ts

export interface TechnicalKPIs {
  // Reliability
  uptime: {
    target: 99.9; // %
    measurement: 'monthly';
    current?: number;
  };
  
  // Performance
  p95ResponseTime: {
    target: 500; // ms
    measurement: 'webhook handler';
    current?: number;
  };
  
  // Quality
  codeCoverage: {
    target: 80; // %
    measurement: 'unit tests';
    current?: number;
  };
  
  deploymentFrequency: {
    target: 'daily';
    measurement: 'production deploys';
    current?: string;
  };
  
  leadTimeForChanges: {
    target: 2; // days from commit to production
    measurement: 'calendar days';
    current?: number;
  };
  
  mttr: {
    target: 30; // minutes - Mean Time to Recovery
    measurement: 'per incident';
    current?: number;
  };
  
  changeFailureRate: {
    target: 5; // % of deploys causing incident
    measurement: 'monthly';
    current?: number;
  };
}

// KPI Dashboard
export async function calculateTeamKPIs(
  period: { from: Date; to: Date }
): Promise<TechnicalKPIs> {
  const [
    uptime,
    p95Response,
    deployFrequency,
    incidents,
    deploys,
  ] = await Promise.all([
    calculateUptime(period),
    calculateP95Response(period),
    countDeploys(period),
    getIncidents(period),
    getDeploys(period),
  ]);

  const failedDeploys = deploys.filter(d => d.caused_incident);
  
  return {
    uptime: {
      target: 99.9,
      measurement: 'monthly',
      current: uptime,
    },
    p95ResponseTime: {
      target: 500,
      measurement: 'webhook handler',
      current: p95Response,
    },
    codeCoverage: {
      target: 80,
      measurement: 'unit tests',
      current: await getLatestCoverage(),
    },
    deploymentFrequency: {
      target: 'daily',
      measurement: 'production deploys',
      current: `${deploys.length} deploys`,
    },
    leadTimeForChanges: {
      target: 2,
      measurement: 'calendar days',
      current: await calculateLeadTime(deploys),
    },
    mttr: {
      target: 30,
      measurement: 'per incident',
      current: incidents.length > 0
        ? incidents.reduce((sum, i) => sum + i.duration, 0) / incidents.length
        : 0,
    },
    changeFailureRate: {
      target: 5,
      measurement: 'monthly',
      current: deploys.length > 0
        ? (failedDeploys.length / deploys.length) * 100
        : 0,
    },
  };
}
```

### 8.2 Business KPIs สำหรับ LINE Bot

```
Business KPIs:

ด้าน Engagement:
□ Monthly Active Users (MAU) - เป้า: เพิ่ม 10%/เดือน
□ Message Open Rate - เป้า: > 60%
□ Click-Through Rate (CTR) - เป้า: > 15%
□ User Retention (30-day) - เป้า: > 40%
□ Follow Rate - เป้า: 5% ของ impressions

ด้าน Business:
□ Orders from LINE - เป้า: 30% ของ total orders
□ Revenue from LINE - เป้า: ฿500,000/เดือน
□ Customer Support Tickets Deflected - เป้า: 60%
□ Average Order Value (AOV) via LINE - เป้า: > AOV จาก channels อื่น
□ Customer Lifetime Value (CLV) via LINE - เป้า: 20% สูงกว่า average

ด้าน Operations:
□ Bot Resolution Rate - เป้า: > 70% (แก้ได้โดยไม่ต้องโอนให้ agent)
□ Average Handling Time - เป้า: < 2 นาที
□ Customer Satisfaction Score (CSAT) - เป้า: > 4.5/5
□ First Contact Resolution - เป้า: > 80%
```

---

## 9. New Member Onboarding Guide

### 9.1 Onboarding Checklist

```markdown
# LINE Bot Team Onboarding Checklist

## Day 1: Administrative
- [ ] IT setup: laptop, accounts, access
- [ ] เพิ่มเข้า Slack channels (#line-bot-team, #incidents, #deployments)
- [ ] เพิ่มเข้า GitHub organization
- [ ] เพิ่มเข้า JIRA/Linear project
- [ ] เพิ่มเข้า AWS/GCP accounts (read-only ก่อน)
- [ ] Meeting กับ manager และ buddy
- [ ] ทัวร์ office / แนะนำทีม

## Week 1: Environment Setup
- [ ] Clone repository ทุกตัว
- [ ] Setup local development environment
- [ ] Run LINE Bot ใน local development
- [ ] Setup LINE Developer Account ส่วนตัวสำหรับ testing
- [ ] อ่าน Architecture documentation
- [ ] ทำ LINE API tutorial
- [ ] Review code ของ key features

## Week 2-4: Ramp Up
- [ ] Pair programming กับ buddy
- [ ] Pick up first ticket (P4 หรือ chore)
- [ ] Review PRs ของ team members
- [ ] เข้าใจ deployment process
- [ ] Shadow on-call engineer
- [ ] ทำ LINE Bot Certified Professional exam (optional)

## Month 2-3: Independent Work
- [ ] Own feature จาก planning ถึง production
- [ ] Lead on-call rotation (กับ buddy)
- [ ] Contribute to architecture discussions
- [ ] ปรับปรุง documentation

## Buddy System
- แต่ละ new hire มี buddy ที่ experience > 6 เดือน
- Buddy meet ทุกวันในสัปดาห์แรก
- Weekly check-in ต่อไปอีก 3 เดือน
```

### 9.2 Local Development Setup Guide

```bash
#!/bin/bash
# scripts/setup-dev.sh

echo "🚀 Setting up LINE Bot development environment..."

# Prerequisites check
check_prerequisites() {
  echo "Checking prerequisites..."
  
  commands=("node" "npm" "docker" "git" "kubectl")
  for cmd in "${commands[@]}"; do
    if command -v "$cmd" &> /dev/null; then
      echo "  ✅ $cmd: $(${cmd} --version 2>&1 | head -1)"
    else
      echo "  ❌ $cmd: not found"
      echo "  Please install $cmd first"
      exit 1
    fi
  done
}

# Clone repositories
clone_repositories() {
  echo "Cloning repositories..."
  
  REPOS=(
    "line-bot-backend"
    "line-bot-liff"
    "line-bot-infrastructure"
  )
  
  for repo in "${REPOS[@]}"; do
    if [ ! -d "$repo" ]; then
      git clone "https://github.com/yourcompany/${repo}.git"
      echo "  ✅ Cloned $repo"
    else
      echo "  ⏭️  $repo already exists"
    fi
  done
}

# Setup environment files
setup_env() {
  echo "Setting up environment files..."
  
  if [ ! -f "line-bot-backend/.env" ]; then
    cp line-bot-backend/.env.example line-bot-backend/.env
    echo "  ✅ Created .env from example"
    echo "  ⚠️  Please update .env with your values:"
    echo "     - LINE_CHANNEL_ACCESS_TOKEN"
    echo "     - LINE_CHANNEL_SECRET"
    echo "     - DATABASE_URL"
  fi
}

# Start local services
start_services() {
  echo "Starting local services (Docker)..."
  docker-compose -f docker-compose.dev.yml up -d
  
  echo "Waiting for services to be ready..."
  sleep 10
  
  # Check services
  if docker-compose -f docker-compose.dev.yml ps | grep -q "Up"; then
    echo "  ✅ Services started successfully"
  else
    echo "  ❌ Some services failed to start"
    docker-compose -f docker-compose.dev.yml ps
  fi
}

# Install dependencies
install_dependencies() {
  echo "Installing dependencies..."
  
  cd line-bot-backend
  npm install
  echo "  ✅ Backend dependencies installed"
  
  cd ../line-bot-liff
  npm install
  echo "  ✅ LIFF dependencies installed"
  
  cd ..
}

# Setup ngrok for webhook testing
setup_ngrok() {
  echo "Setting up ngrok for webhook testing..."
  
  if ! command -v ngrok &> /dev/null; then
    echo "  ⚠️  ngrok not found. Install from https://ngrok.com/download"
  else
    echo "  ✅ ngrok is installed"
    echo "  Run: ngrok http 3000"
    echo "  Then update LINE Webhook URL in LINE Developers Console"
  fi
}

# Main
check_prerequisites
clone_repositories
setup_env
start_services
install_dependencies
setup_ngrok

echo ""
echo "✅ Development environment setup complete!"
echo ""
echo "Next steps:"
echo "1. Update .env with your credentials"
echo "2. Run: cd line-bot-backend && npm run dev"
echo "3. Run: ngrok http 3000"
echo "4. Update Webhook URL in LINE Developers Console"
echo "5. Test your LINE Bot!"
```

---

## 10. Training Curriculum

### 10.1 LINE Bot Developer Training Plan

```
Training Curriculum สำหรับ LINE Bot Developer

Month 1: Foundation (80 ชั่วโมง)

Week 1-2: LINE Platform Basics (20h)
□ LINE OA Dashboard (2h)
□ LINE Messaging API Overview (4h)
□ Webhook events deep dive (4h)
□ Reply vs Push vs Broadcast (3h)
□ Hands-on: สร้าง basic chatbot (7h)

Week 3-4: Message Types (20h)
□ Text, Sticker, Image messages (3h)
□ Template messages (4h)
□ Flex Messages (8h)
□ Quick Reply (2h)
□ Hands-on: สร้าง product catalog bot (3h)

Week 5-6: Advanced Features (20h)
□ Rich Menu API (4h)
□ Postback events (3h)
□ LIFF Introduction (5h)
□ LINE Login (3h)
□ Hands-on: LIFF order form (5h)

Week 7-8: Backend Development (20h)
□ Node.js/Express webhook server (4h)
□ Database design สำหรับ LINE bots (4h)
□ State management with Redis (3h)
□ Error handling & logging (3h)
□ Testing LINE bots (6h)

Month 2: Intermediate (80 ชั่วโมง)

Week 9-10: Production Readiness (20h)
□ Security best practices (5h)
□ Rate limiting (3h)
□ Monitoring setup (5h)
□ CI/CD for LINE bots (7h)

Week 11-12: Advanced Architecture (20h)
□ Event-driven architecture (8h)
□ Microservices patterns (5h)
□ Scaling strategies (7h)

Week 13-14: LINE Commerce (20h)
□ LINE Pay integration (8h)
□ E-commerce bot development (8h)
□ Analytics setup (4h)

Week 15-16: Project Work (20h)
□ Capstone project (20h)
   - Design และ implement complete LINE Bot system
   - Code review by senior
   - Demo to team

Certifications:
□ LINE Front-end Framework (LIFF) certification
□ LINE Messaging API certification
□ Node.js certification (optional)

Resources:
- LINE Developers Documentation
- LINE Bot examples repo (internal)
- Team wiki
- YouTube: LINE Developers Channel
```

---

## 11. Complete Team Handbook Template

### 11.1 Team Values และ Principles

```markdown
# LINE Bot Team Handbook

## Our Mission
"สร้าง LINE Bot ที่ทำให้ชีวิตลูกค้าง่ายขึ้น และช่วยให้ธุรกิจเติบโต"

## Team Values

### 1. User First
- ทุก decision ต้องถามว่า "นี่ดีสำหรับ user หรือไม่?"
- Performance ไม่ดีคือ bug ชนิดหนึ่ง
- UX ที่ดีสร้าง trust และ engagement

### 2. Reliability Matters
- "It works on my machine" ไม่พอ
- ทุก deploy ต้องผ่าน automated tests
- Monitor production อย่างสม่ำเสมอ

### 3. Team Over Individual
- Code review ช่วย team บรรลุเป้าหมาย ไม่ใช่วิจารณ์คนเขียน
- Knowledge sharing เป็นส่วนสำคัญของงาน
- Celebrate team wins, not individual wins

### 4. Continuous Improvement
- Retrospectives ช่วย improve processes
- Blameless postmortems ช่วย improve systems
- Learning > Blame

### 5. Transparency
- Communicate openly about challenges
- ถ้ามีปัญหา บอก team ให้เร็วที่สุด
- Share knowledge อย่างเปิดเผย
```

### 11.2 Meeting Cadence

```
Meeting Schedule:

Daily Standup: 09:30-09:45 (15 min)
- What did I do yesterday?
- What will I do today?
- Any blockers?

Weekly Team Meeting: จันทร์ 10:00-11:00
- Sprint progress review
- Discuss blockers
- Architecture discussions

Sprint Planning: ทุก 2 สัปดาห์ (3 ชั่วโมง)
- Review upcoming tickets
- Estimate story points
- Commit to sprint goals

Sprint Review/Demo: ทุก 2 สัปดาห์ (1 ชั่วโมง)
- Demo new features
- Stakeholder feedback

Sprint Retrospective: ทุก 2 สัปดาห์ (1 ชั่วโมง)
- What went well?
- What could be improved?
- Action items

Monthly Tech Talk: เดือนละครั้ง (1 ชั่วโมง)
- Share learning, new technologies
- สมาชิก team สลับกัน present

Meeting Rules:
1. เริ่มตรงเวลา
2. มี agenda ล่วงหน้า
3. Note taker หมุนเวียน
4. Action items มี owner และ due date
5. ไม่มีโทรศัพท์/laptop ถ้าไม่จำเป็น
```

### 11.3 Communication Channels

```
Communication Tools:

Slack:
#line-bot-team    - ทั่วไป
#line-incidents   - Incidents เท่านั้น
#line-deployments - Deploy notifications
#line-alerts      - Automated alerts
#line-reviews     - Code reviews
#line-random      - สนุกสนาน

Email:
- สำหรับ formal communication
- External stakeholders
- ไม่ใช่ urgent items

JIRA/Linear:
- Project management
- Bug tracking
- Sprint planning

Confluence/Notion:
- Documentation
- Runbooks
- Architecture decisions

PagerDuty:
- Incident management
- On-call scheduling
- Alert routing

Response Time Expectations:
- Slack (business hours): ภายใน 2 ชั่วโมง
- Email (business hours): ภายใน 24 ชั่วโมง
- Slack (urgent/production): ทันที
- PagerDuty P1: 5 นาที
```

---

## สรุป

บทนี้ครอบคลุม:

1. **Team Structure** สำหรับขนาดธุรกิจต่างๆ
2. **Roles & Responsibilities** ที่ชัดเจน
3. **Git Flow** สำหรับ LINE Bot development
4. **Sprint Planning** templates
5. **Code Review** checklist ครอบคลุม
6. **Documentation Standards** สำหรับ LINE Bot
7. **On-Call Runbook** พร้อมใช้งาน
8. **Incident Management** process
9. **KPIs** ทั้ง technical และ business
10. **Onboarding Guide** สำหรับ new members
11. **Training Curriculum** 2 เดือน

Team ที่ดีมี process ที่ชัดเจน สื่อสารเปิดเผย และ focus ที่การพัฒนาอย่างต่อเนื่อง นั่นคือสูตรสำเร็จสำหรับ LINE Bot team ที่ประสบความสำเร็จ
