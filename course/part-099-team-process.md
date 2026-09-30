# ส่วนที่ 99: Team Structure และ Development Process สำหรับ LINE Bot

---

## สารบัญ

1. [Team Structure สำหรับ LINE Bot Development](#1-team-structure-สำหรับ-line-bot-development)
2. [Roles and Responsibilities](#2-roles-and-responsibilities)
3. [Development Workflow](#3-development-workflow)
4. [Git Branching Strategy](#4-git-branching-strategy)
5. [Code Review Process](#5-code-review-process)
6. [Documentation Standards](#6-documentation-standards)
7. [On-call Rotation](#7-on-call-rotation)
8. [Incident Management](#8-incident-management)
9. [Sprint Planning สำหรับ LINE Bot Features](#9-sprint-planning-สำหรับ-line-bot-features)
10. [KPIs สำหรับ LINE OA Team](#10-kpis-สำหรับ-line-oa-team)
11. [Training Plan สำหรับ New Team Members](#11-training-plan-สำหรับ-new-team-members)
12. [Complete Team Handbook Template](#12-complete-team-handbook-template)

---

## 1. Team Structure สำหรับ LINE Bot Development

### 1.1 ขนาด Team ตาม Scale ของธุรกิจ

```
Team Structure Recommendations:

Startup (< 10,000 followers):
├── 1 Full-stack Developer
├── 1 UX Designer (part-time)
└── 1 Product Manager

Small Business (10,000 - 100,000 followers):
├── 1 LINE OA Manager
├── 2 Bot Developers (Frontend + Backend)
├── 1 UX Designer
├── 1 Content Creator
└── 1 DevOps (part-time)

Medium Business (100,000 - 1,000,000 followers):
├── 1 Head of LINE OA
├── 1 LINE OA Manager
├── 3 Bot Developers (FE, BE, Full-stack)
├── 1 UX Designer
├── 1 Data Analyst
├── 1 DevOps Engineer
├── 2 Content Creators
└── 2 Customer Service Agents

Enterprise (> 1,000,000 followers):
├── 1 Director of Digital
├── 1 Head of LINE OA
├── 2 LINE OA Managers
├── 5+ Bot Developers
├── 2 UX Designers
├── 2 Data Analysts
├── 2 DevOps Engineers
├── 3 Content Creators
└── 5+ Customer Service Agents
```

### 1.2 Organization Chart

```
LINE OA Team Organization:

                    ┌─────────────────────┐
                    │   Product Director   │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼──────┐ ┌───────▼───────┐ ┌────▼────────┐
    │  LINE OA       │ │   Tech Lead   │ │  Data Lead  │
    │  Manager       │ │               │ │             │
    └─────────┬──────┘ └───────┬───────┘ └────┬────────┘
              │                │               │
    ┌─────────▼────┐  ┌────────▼────┐  ┌─────▼──────┐
    │  Content     │  │  Developers │  │  Analysts  │
    │  Team        │  │  + DevOps   │  │            │
    └──────────────┘  └─────────────┘  └────────────┘
```

---

## 2. Roles and Responsibilities

### 2.1 LINE OA Manager

```
Role: LINE OA Manager
═════════════════════

ความรับผิดชอบหลัก:
├── กำหนดกลยุทธ์ LINE OA ให้สอดคล้องกับ business goals
├── วางแผน content calendar ประจำเดือน
├── ติดตาม KPIs และรายงานต่อผู้บริหาร
├── บริหารจัดการ message quota และ budget
├── ประสานงานระหว่าง technical team และ business team
└── อนุมัติ campaigns ก่อน publish

ทักษะที่จำเป็น:
├── เข้าใจ LINE OA platform ทุกส่วน
├── ทักษะการวิเคราะห์ข้อมูล
├── ทักษะการสื่อสารและ presentation
├── ความเข้าใจ digital marketing
└── ทักษะการบริหารโครงการ

KPIs ส่วนตัว:
├── Follower Growth Rate
├── Message Open Rate > 50%
├── Campaign ROI > 200%
├── Customer Satisfaction Score > 4.0/5.0
└── Content Calendar Adherence > 90%

Deliverables รายเดือน:
├── Monthly performance report
├── Content calendar สำหรับเดือนถัดไป
├── Campaign briefs
└── Budget reconciliation report
```

### 2.2 Bot Developer

```
Role: Bot Developer
═══════════════════

Frontend Bot Developer:
├── ความรับผิดชอบ:
│   ├── พัฒนา LIFF applications
│   ├── ออกแบบ Rich Messages (Flex Message)
│   ├── พัฒนา Rich Menu
│   └── Optimize performance ของ LIFF
│
├── Tech Stack:
│   ├── JavaScript/TypeScript
│   ├── React.js หรือ Vue.js
│   ├── LINE LIFF SDK
│   ├── CSS/SCSS
│   └── LINE Flex Message Builder

Backend Bot Developer:
├── ความรับผิดชอบ:
│   ├── พัฒนา Webhook handlers
│   ├── Integrate LINE Messaging API
│   ├── ออกแบบ database schema
│   ├── สร้าง APIs สำหรับ LIFF
│   └── Implement business logic
│
├── Tech Stack:
│   ├── Node.js / Python
│   ├── LINE Messaging API SDK
│   ├── PostgreSQL / MySQL
│   ├── Redis
│   └── Express.js / FastAPI

DevOps Engineer (สำหรับ LINE Bot):
├── ความรับผิดชอบ:
│   ├── CI/CD pipeline
│   ├── Kubernetes / Docker deployment
│   ├── Monitoring และ alerting
│   ├── Infrastructure as Code
│   └── Security hardening
│
├── Tech Stack:
│   ├── Docker / Kubernetes
│   ├── Terraform / Pulumi
│   ├── GitHub Actions / GitLab CI
│   ├── Prometheus / Grafana
│   └── AWS / GCP / Azure

Code Review Standards สำหรับ Bot Developers:
├── ตรวจสอบ LINE API error handling
├── ตรวจสอบ message quota usage
├── ตรวจสอบ webhook security (signature verification)
├── ตรวจสอบ async/await patterns
├── ตรวจสอบ sensitive data handling
└── ตรวจสอบ rate limiting implementation
```

### 2.3 UX Designer สำหรับ LINE Bot

```
Role: UX Designer (LINE OA Specialist)
═══════════════════════════════════════

ความรับผิดชอบหลัก:
├── ออกแบบ Flex Message layouts
├── ออกแบบ Rich Menu (ทั้ง desktop และ mobile)
├── ออกแบบ user flow สำหรับ bot conversations
├── สร้าง LIFF page wireframes และ prototypes
└── วิเคราะห์ user behavior จาก analytics

LINE-specific Design Skills:
├── Flex Message JSON structure
├── LINE Design System
│   ├── Typography: Sarabun (TH), Noto Sans (EN)
│   ├── Colors: LINE Green (#00B900), White, Gray
│   └── Icons: LINE Icon Set
├── Rich Menu best practices
│   ├── ขนาด: 2500x1686px (6 panels), 2500x843px (3 panels)
│   ├── File format: PNG, JPEG
│   └── Max file size: 1MB
├── Conversation UI patterns
└── Mobile-first design

Design Process:
1. Research: วิเคราะห์ user needs จาก LINE OA insights
2. Ideation: สร้าง conversation flows และ wireframes
3. Prototype: ทำ interactive prototype ด้วย Figma
4. Test: A/B testing บน LINE
5. Iterate: ปรับปรุงตาม data

Design Tools:
├── Figma (หลัก)
├── LINE Flex Message Simulator
├── Adobe XD
└── Zeplin (สำหรับ handoff)
```

### 2.4 Data Analyst

```
Role: Data Analyst (LINE OA)
════════════════════════════

ความรับผิดชอบหลัก:
├── วิเคราะห์ LINE OA insights data
├── สร้าง performance dashboards
├── วิเคราะห์ campaign performance
├── ทำ cohort analysis และ user segmentation
├── สร้าง automated reports
└── ให้ข้อมูลสนับสนุนการตัดสินใจ

Technical Skills:
├── SQL (PostgreSQL เป็นหลัก)
├── Python (pandas, matplotlib, seaborn)
├── Google Analytics / Mixpanel
├── Apache Superset หรือ Metabase
├── LINE OA Insights API
└── BigQuery หรือ Redshift

Weekly Deliverables:
├── สรุปยอด followers และ engagement
├── Top performing content analysis
├── Conversion funnel report
└── Upcoming campaign predictions

Monthly Deliverables:
├── Full performance report
├── Cohort retention analysis
├── Revenue attribution report
├── Churn risk analysis
└── Next month forecasting
```

### 2.5 Customer Service Agent (LINE)

```
Role: Customer Service Agent (LINE OA)
═══════════════════════════════════════

ความรับผิดชอบหลัก:
├── ตอบคำถามลูกค้าผ่าน LINE chat
├── ยกระดับปัญหาที่ Bot ไม่สามารถแก้ได้
├── Manage assigned conversations
├── ติดตาม open tickets จนปิดสำเร็จ
└── Collect customer feedback

Service Level Targets:
├── First Response Time: < 3 นาที (business hours)
├── Resolution Time: < 24 ชั่วโมง
├── CSAT Score: > 4.2/5.0
├── First Contact Resolution Rate: > 75%
└── Messages Handled Per Day: 50-80

Escalation Guidelines:
├── Level 1: CS Agent (ทั่วไป)
├── Level 2: Senior CS (ปัญหาซับซ้อน)
├── Level 3: Technical Team (ปัญหา technical)
└── Level 4: Management (ปัญหา VIP/sensitive)

Tools:
├── LINE Official Account Manager
├── Admin Dashboard (custom)
├── Ticketing System (Freshdesk/Zendesk)
├── CRM System
└── Knowledge Base
```

---

## 3. Development Workflow

### 3.1 Feature Development Process

```
Feature Development Flow:

1. DISCOVERY (1-2 สัปดาห์)
   ├── Product Brief จาก LINE OA Manager
   ├── User Research (LINE OA analytics)
   ├── Technical Feasibility Study
   └── Effort Estimation

2. DESIGN (1-2 สัปดาห์)
   ├── UX Designer สร้าง wireframes
   ├── Design Review กับ team
   ├── Flex Message prototypes
   └── Technical Architecture Design

3. DEVELOPMENT (2-4 สัปดาห์)
   ├── Backend development
   ├── Frontend/LIFF development
   ├── Integration testing
   └── Code review

4. QA & TESTING (1 สัปดาห์)
   ├── Unit tests
   ├── Integration tests
   ├── LINE Bot manual testing
   └── User Acceptance Testing

5. DEPLOYMENT (1-2 วัน)
   ├── Staging deployment
   ├── Smoke testing
   ├── Production deployment
   └── Post-deployment monitoring

6. MONITORING (ต่อเนื่อง)
   ├── Analytics monitoring
   ├── Error tracking
   ├── Performance monitoring
   └── User feedback collection
```

### 3.2 Development Environment Setup

```bash
#!/bin/bash
# scripts/setup-dev-environment.sh

echo "Setting up LINE Bot Development Environment..."

# 1. Check requirements
check_requirements() {
  echo "Checking requirements..."
  
  commands=("node" "npm" "git" "docker" "docker-compose")
  
  for cmd in "${commands[@]}"; do
    if ! command -v "$cmd" &> /dev/null; then
      echo "ERROR: $cmd is not installed"
      exit 1
    fi
    echo "✓ $cmd: $(${cmd} --version 2>&1 | head -1)"
  done
}

# 2. Setup project
setup_project() {
  echo "Setting up project..."
  
  # Clone or check existing
  if [ ! -d ".git" ]; then
    echo "ERROR: Not in a git repository"
    exit 1
  fi
  
  # Install dependencies
  npm install
  
  # Setup environment
  if [ ! -f ".env" ]; then
    cp .env.example .env
    echo "✓ Created .env from .env.example"
    echo "⚠ Please fill in the required values in .env"
  fi
}

# 3. Start local services
start_services() {
  echo "Starting local services..."
  
  docker-compose -f docker-compose.dev.yml up -d
  
  echo "Waiting for services to start..."
  sleep 5
  
  # Check services
  services=("postgres" "redis")
  for service in "${services[@]}"; do
    if docker-compose -f docker-compose.dev.yml ps | grep "$service" | grep "Up" > /dev/null; then
      echo "✓ $service is running"
    else
      echo "ERROR: $service failed to start"
    fi
  done
}

# 4. Setup LINE webhook for development
setup_ngrok() {
  if command -v ngrok &> /dev/null; then
    echo "Starting ngrok..."
    ngrok http 3000 &
    sleep 2
    
    NGROK_URL=$(curl -s localhost:4040/api/tunnels | python3 -c "import sys,json; print(json.load(sys.stdin)['tunnels'][0]['public_url'])")
    echo "✓ ngrok URL: $NGROK_URL"
    echo "  Set this as your LINE Webhook URL: ${NGROK_URL}/webhook"
  else
    echo "⚠ ngrok not found. Install it for LINE webhook testing."
  fi
}

# 5. Run database migrations
run_migrations() {
  echo "Running database migrations..."
  npm run db:migrate
  npm run db:seed
  echo "✓ Database ready"
}

check_requirements
setup_project
start_services
run_migrations
setup_ngrok

echo ""
echo "✅ Development environment ready!"
echo "   Start the bot: npm run dev"
echo "   Dashboard: http://localhost:3000/admin"
echo "   API Docs: http://localhost:3000/api-docs"
```

---

## 4. Git Branching Strategy

### 4.1 GitFlow for LINE Bot Projects

```
Git Branching Strategy (Modified GitFlow):

main ──────────────────────────────────────────────────────>
  │                                                         
  ├── develop ─────────────────────────────────────────────>
  │       │                                                 
  │       ├── feature/line-oa-101-rich-menu-v2 ──────────>
  │       │                            merge ──────────────>
  │       │                                                 
  │       ├── feature/line-oa-102-ai-chatbot ─────────────>
  │       │                         merge ─────────────────>
  │       │                                                 
  │       ├── release/v2.5.0 ──────────────────────────────>
  │       │         merge to main ──────────────────────────>
  │       │                                                 
  │       └── hotfix/fix-webhook-timeout ──────────────────>
  │                   merge to main + develop ─────────────>
  │                                                         
```

### 4.2 Branch Naming Convention

```
Branch Naming Standards:

Feature branches:
  feature/{ticket-number}-{short-description}
  ตัวอย่าง:
  - feature/LINE-101-rich-menu-v2
  - feature/LINE-102-ai-chatbot-integration
  - feature/LINE-103-payment-liff

Bug fix branches:
  fix/{ticket-number}-{short-description}
  ตัวอย่าง:
  - fix/LINE-201-webhook-timeout
  - fix/LINE-202-flex-message-rendering

Hotfix branches (production issues):
  hotfix/{ticket-number}-{short-description}
  ตัวอย่าง:
  - hotfix/LINE-301-critical-webhook-error

Release branches:
  release/v{major}.{minor}.{patch}
  ตัวอย่าง:
  - release/v2.5.0
  - release/v2.5.1

Chore/Improvement:
  chore/{description}
  ตัวอย่าง:
  - chore/update-line-sdk-v10
  - chore/add-redis-caching
```

### 4.3 Commit Message Convention

```
Commit Message Format:
<type>(<scope>): <short description>

[Optional body]

[Optional footer]

Types:
  feat:     New feature
  fix:      Bug fix
  docs:     Documentation only
  style:    Formatting (no logic change)
  refactor: Code refactoring
  test:     Adding/updating tests
  chore:    Maintenance tasks
  perf:     Performance improvement

Scopes (LINE Bot specific):
  webhook:    Webhook handlers
  liff:       LIFF application
  messaging:  Message sending
  ai:         AI/chatbot features
  payment:    Payment integration
  analytics:  Analytics features
  admin:      Admin dashboard
  infra:      Infrastructure

ตัวอย่าง commit messages:
  feat(liff): add real-time order tracking page
  fix(webhook): handle LINE signature verification error
  docs(api): update webhook endpoint documentation
  chore(deps): upgrade @line/bot-sdk to v9.3.0
  perf(messaging): implement message batching for broadcasts
  test(webhook): add unit tests for message handler
```

### 4.4 Git Hooks สำหรับ Quality Control

```javascript
// .husky/pre-commit
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

echo "Running pre-commit checks..."

# Lint
npm run lint
if [ $? -ne 0 ]; then
  echo "ERROR: Lint failed. Please fix lint errors before committing."
  exit 1
fi

# Type check (ถ้าใช้ TypeScript)
npm run type-check 2>/dev/null
if [ $? -ne 0 ]; then
  echo "ERROR: TypeScript type check failed."
  exit 1
fi

# Test
npm run test:unit
if [ $? -ne 0 ]; then
  echo "ERROR: Tests failed. Please fix tests before committing."
  exit 1
fi

echo "✓ Pre-commit checks passed"
```

```javascript
// .husky/commit-msg
#!/bin/sh
. "$(dirname "$0")/_/husky.sh"

# ตรวจสอบ commit message format
npx --no -- commitlint --edit $1
```

```javascript
// commitlint.config.js
module.exports = {
  extends: ['@commitlint/config-conventional'],
  rules: {
    'type-enum': [2, 'always', [
      'feat', 'fix', 'docs', 'style', 'refactor', 
      'test', 'chore', 'perf', 'ci', 'revert'
    ]],
    'scope-enum': [2, 'always', [
      'webhook', 'liff', 'messaging', 'ai', 'payment',
      'analytics', 'admin', 'infra', 'auth', 'queue',
      'auction', 'crm', 'deps', 'config'
    ]],
    'subject-max-length': [2, 'always', 100],
    'body-max-line-length': [2, 'always', 150],
  },
};
```

---

## 5. Code Review Process

### 5.1 Code Review Checklist

```
LINE Bot Code Review Checklist:
════════════════════════════════

SECURITY (Must Check):
□ LINE Channel Secret ไม่ถูก hardcode
□ Webhook signature verification มีอยู่และทำงานถูกต้อง
□ User input ถูก sanitize ก่อนใช้
□ SQL queries ใช้ parameterized queries
□ Sensitive data ไม่ถูก log
□ JWT tokens ถูก validate อย่างถูกต้อง
□ Rate limiting มีอยู่สำหรับ public endpoints

LINE API BEST PRACTICES:
□ Reply tokens ถูกใช้ภายใน 30 วินาที
□ Message quota ถูกพิจารณาก่อนส่ง broadcast
□ Retry logic มีอยู่สำหรับ API calls
□ Error responses จาก LINE API ถูก handle
□ Push messages ใช้ multicast เมื่อส่งหลายคน
□ Message types ถูกต้องตาม LINE specs

CODE QUALITY:
□ ไม่มี console.log ใน production code
□ Error handling ครอบคลุมทุก async operations
□ Functions มีขนาดเหมาะสม (< 50 lines)
□ DRY principle ถูกปฏิบัติตาม
□ Comments อธิบาย "why" ไม่ใช่ "what"
□ TypeScript types ถูกกำหนดอย่างชัดเจน

TESTING:
□ Unit tests ครอบคลุม critical paths
□ Integration tests สำหรับ LINE API calls
□ Edge cases ถูก test
□ Test coverage > 80% สำหรับ new code

PERFORMANCE:
□ Database queries ใช้ indexes
□ N+1 query problem ไม่มี
□ Async operations ทำงานแบบ parallel เมื่อเป็นไปได้
□ Large datasets ถูก paginate
□ Redis caching ถูกใช้อย่างเหมาะสม

DOCUMENTATION:
□ JSDoc/docstrings สำหรับ public functions
□ README อัพเดทถ้า setup steps เปลี่ยน
□ API endpoints มี documentation
□ Environment variables มีใน .env.example
```

### 5.2 Pull Request Template

```markdown
<!-- .github/pull_request_template.md -->
## 📋 สรุปการเปลี่ยนแปลง

<!-- อธิบายว่า PR นี้ทำอะไร และทำไมถึงต้องทำ -->

## 🔗 Ticket Reference

Closes #[ticket-number]

## 📝 ประเภทการเปลี่ยนแปลง

- [ ] 🆕 Feature ใหม่
- [ ] 🐛 Bug fix
- [ ] ♻️ Refactoring
- [ ] 📚 Documentation
- [ ] 🔧 Configuration

## 🧪 วิธีการ Test

1. <!-- ขั้นตอนที่ 1 -->
2. <!-- ขั้นตอนที่ 2 -->

## 📸 Screenshots (ถ้ามี UI changes)

| Before | After |
|--------|-------|
| <!-- screenshot --> | <!-- screenshot --> |

## ✅ Checklist

### Security
- [ ] ไม่มี credentials ใน code
- [ ] Webhook signature verification ทำงานถูกต้อง
- [ ] Input validation ครบถ้วน

### LINE API
- [ ] ทดสอบกับ LINE Official Account จริง
- [ ] Message quota ถูกพิจารณา
- [ ] Error cases ถูก handle

### Code Quality
- [ ] Tests ผ่านทั้งหมด
- [ ] Lint ไม่มี errors
- [ ] Self-review เสร็จแล้ว

### Documentation
- [ ] Code comments เหมาะสม
- [ ] README อัพเดทแล้ว (ถ้าจำเป็น)

## 🚨 Breaking Changes

<!-- ระบุถ้ามี breaking changes -->

## 🗒️ Additional Notes

<!-- ข้อมูลเพิ่มเติมสำหรับ reviewers -->
```

### 5.3 Code Review Guidelines

```javascript
// src/utils/reviewGuidelines.js
/**
 * LINE Bot Code Review Guidelines
 * 
 * Example ของ code ที่ดีและไม่ดีสำหรับการ review
 */

// ❌ BAD: Webhook handler ที่ไม่ดี
app.post('/webhook', (req, res) => {
  // ไม่ verify signature
  // ไม่ handle errors
  // ไม่ respond ทันที
  
  const events = req.body.events;
  events.forEach(event => {
    if (event.type === 'message') {
      // Long synchronous operation
      const result = heavyComputation(event.message.text);
      client.replyMessage(event.replyToken, {
        type: 'text',
        text: result
      });
    }
  });
  
  res.sendStatus(200);
});

// ✅ GOOD: Webhook handler ที่ถูกต้อง
app.post('/webhook',
  line.middleware({ channelSecret: config.line.channelSecret }),
  async (req, res) => {
    // ตอบ LINE ทันที เพื่อไม่ให้ timeout
    res.sendStatus(200);
    
    try {
      // Process events แบบ async
      await Promise.all(
        req.body.events.map(event => processEvent(event))
      );
    } catch (error) {
      logger.error('Webhook processing error:', {
        error: error.message,
        stack: error.stack,
      });
      // ไม่ re-throw เพราะ LINE รับแล้ว
    }
  }
);

// ❌ BAD: Hardcoded credentials
const client = new line.Client({
  channelAccessToken: 'abc123xyz...',  // Never!
  channelSecret: 'secret123...',       // Never!
});

// ✅ GOOD: Environment variables
const client = new line.Client({
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET,
});

// ❌ BAD: N+1 query
async function getUsersWithOrders(userIds) {
  const users = await User.findAll({ where: { id: userIds } });
  
  // N+1 problem: query สำหรับแต่ละ user
  for (const user of users) {
    user.orders = await Order.findAll({ where: { userId: user.id } });
  }
  
  return users;
}

// ✅ GOOD: Efficient query
async function getUsersWithOrders(userIds) {
  return User.findAll({
    where: { id: userIds },
    include: [{
      model: Order,
      required: false,
    }],
  });
}
```

---

## 6. Documentation Standards

### 6.1 API Documentation

```javascript
/**
 * @api {post} /api/messages/send Send Message to LINE User
 * @apiName SendMessage
 * @apiGroup Messages
 * @apiVersion 1.0.0
 *
 * @apiHeader {String} Authorization Bearer token
 * @apiHeader {String} Content-Type application/json
 *
 * @apiBody {String} userId LINE User ID
 * @apiBody {String} messageType Message type (text|flex|image)
 * @apiBody {Object} content Message content
 *
 * @apiSuccess {Boolean} success Operation status
 * @apiSuccess {String} messageId LINE Message ID
 * @apiSuccess {Number} remainingQuota Remaining message quota
 *
 * @apiError {String} error Error message
 * @apiError {String} code Error code
 *
 * @apiExample {json} Request Example:
 *   {
 *     "userId": "U1234567890abcdef",
 *     "messageType": "text",
 *     "content": {
 *       "text": "Hello from API!"
 *     }
 *   }
 *
 * @apiExample {json} Success Response:
 *   {
 *     "success": true,
 *     "messageId": "8iXXXXXXXXXXXXXXX",
 *     "remainingQuota": 4985
 *   }
 */

// src/routes/messageRoutes.js
router.post('/send', authenticate, async (req, res) => {
  try {
    const { userId, messageType, content } = req.body;
    
    // Validate
    if (!userId || !messageType || !content) {
      return res.status(400).json({
        error: 'Missing required fields',
        code: 'VALIDATION_ERROR',
        required: ['userId', 'messageType', 'content'],
      });
    }
    
    const result = await messageService.sendMessage(userId, messageType, content);
    
    res.json({
      success: true,
      messageId: result.messageId,
      remainingQuota: result.remainingQuota,
    });
  } catch (error) {
    logger.error('Send message error:', error);
    
    if (error.code === 'QUOTA_EXCEEDED') {
      return res.status(429).json({
        error: 'Message quota exceeded',
        code: 'QUOTA_EXCEEDED',
        resetAt: error.resetAt,
      });
    }
    
    res.status(500).json({
      error: 'Internal server error',
      code: 'INTERNAL_ERROR',
    });
  }
});
```

### 6.2 README Template สำหรับ LINE Bot Project

```markdown
# [Project Name] LINE Bot

![Version](https://img.shields.io/badge/version-2.0.0-blue)
![Node](https://img.shields.io/badge/node-18.x-green)
![Tests](https://img.shields.io/badge/tests-passing-brightgreen)

## สารบัญ

- [ภาพรวม](#ภาพรวม)
- [Requirements](#requirements)
- [การติดตั้ง](#การติดตั้ง)
- [Configuration](#configuration)
- [การรัน](#การรัน)
- [API Documentation](#api-documentation)
- [Testing](#testing)
- [Deployment](#deployment)
- [Troubleshooting](#troubleshooting)

## ภาพรวม

[อธิบาย LINE Bot นี้ทำอะไร, ฟีเจอร์หลัก, สถาปัตยกรรม]

## Requirements

- Node.js >= 18.x
- PostgreSQL >= 14
- Redis >= 7
- Docker (optional)
- LINE Developer Account

## การติดตั้ง

```bash
# Clone repository
git clone https://github.com/your-org/line-bot.git
cd line-bot

# Install dependencies
npm install

# Setup environment
cp .env.example .env
# แก้ไข .env ตาม configuration ของคุณ

# Setup database
npm run db:migrate
npm run db:seed
```

## Configuration

สร้าง `.env` file จาก `.env.example`:

| Variable | Required | Description |
|----------|----------|-------------|
| `LINE_CHANNEL_ACCESS_TOKEN` | ✅ | LINE Channel Access Token |
| `LINE_CHANNEL_SECRET` | ✅ | LINE Channel Secret |
| `DATABASE_URL` | ✅ | PostgreSQL connection string |
| `REDIS_URL` | ✅ | Redis connection URL |
| `JWT_SECRET` | ✅ | JWT signing secret |
| `LIFF_ID` | ✅ | LINE LIFF Application ID |

## Architecture

```
[Architecture diagram here]
```

## Testing

```bash
# Unit tests
npm test

# Integration tests
npm run test:integration

# Coverage
npm run test:coverage
```
```

---

## 7. On-call Rotation

### 7.1 On-call Schedule

```
On-call Rotation Policy:
═════════════════════════

ช่วงเวลา:
├── Business Hours: 09:00 - 18:00 (Mon-Fri)
│   Primary: On-duty developer
│   
├── After Hours: 18:00 - 09:00 (Mon-Fri) + Weekends
│   Primary: On-call developer (หมุนเวียนรายสัปดาห์)
│   Secondary: Team Lead (backup)
│
└── Holidays:
    Primary: Volunteer + Compensation days
    Secondary: Team Lead

Rotation Schedule (4-person team):
Week 1: Developer A (Primary), Developer B (Secondary)
Week 2: Developer B (Primary), Developer C (Secondary)
Week 3: Developer C (Primary), Developer D (Secondary)
Week 4: Developer D (Primary), Developer A (Secondary)

On-call Responsibilities:
├── ตอบ PagerDuty/OpsGenie alerts
├── Investigate และ resolve P1/P2 incidents
├── Escalate ถ้าจำเป็น
├── Post incident report หลัง resolve
└── Update runbook ถ้าพบวิธีใหม่

Compensation:
├── On-call allowance: ฿2,000/week
├── Incident response bonus: ฿500/P1 resolved
└── Comp time: 1 day ต่อ major incident
```

### 7.2 Alert Configuration

```yaml
# alertmanager/config.yml
global:
  resolve_timeout: 5m
  slack_api_url: ${SLACK_WEBHOOK_URL}

route:
  receiver: 'default'
  group_by: ['alertname', 'severity']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  
  routes:
  - match:
      severity: critical
    receiver: pagerduty-critical
    continue: true
    
  - match:
      severity: critical
    receiver: slack-alerts
    
  - match:
      severity: warning
    receiver: slack-alerts

receivers:
- name: 'default'
  slack_configs:
  - channel: '#line-bot-alerts'
    title: '{{ .GroupLabels.alertname }}'
    text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

- name: 'pagerduty-critical'
  pagerduty_configs:
  - service_key: ${PAGERDUTY_SERVICE_KEY}
    description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'

- name: 'slack-alerts'
  slack_configs:
  - channel: '#line-bot-oncall'
    title: '[{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
    text: |
      *Summary:* {{ .CommonAnnotations.summary }}
      *Description:* {{ .CommonAnnotations.description }}
      *Runbook:* {{ .CommonAnnotations.runbook_url }}
    color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
```

---

## 8. Incident Management

### 8.1 Incident Severity Levels

```
Incident Severity Classification:
══════════════════════════════════

P1 - Critical (ตอบสนองใน 15 นาที):
├── LINE webhook ไม่ทำงาน (ลูกค้าส่งข้อความไม่ได้รับการตอบ)
├── Payment integration ล้มเหลว (ทำให้สูญเสียรายได้)
├── Database down (ระบบทั้งหมดไม่ทำงาน)
├── Security breach
└── Data loss

P2 - High (ตอบสนองใน 1 ชั่วโมง):
├── Bot ตอบช้า (> 30 วินาที)
├── LIFF pages ไม่โหลด
├── Broadcast campaign ส่งไม่ออก
├── Admin dashboard ไม่ทำงาน
└── Analytics ไม่อัพเดท

P3 - Medium (ตอบสนองใน 4 ชั่วโมง, business hours):
├── Rich Menu ไม่แสดง
├── บาง features ทำงานผิดปกติ
├── Performance degradation (ช้าลง 50%)
└── Non-critical integrations ล้มเหลว

P4 - Low (ตอบสนองใน 1-2 วัน, business hours):
├── Minor UI bugs
├── Translation errors
├── Low-priority feature requests
└── Documentation issues
```

### 8.2 Incident Response Runbook

```markdown
# Incident Response Runbook

## เมื่อ LINE Webhook ไม่ทำงาน

### Symptoms
- ไม่มี logs ใน webhook endpoint
- LINE ส่ง webhook retry notifications
- Users ร้องเรียนว่า bot ไม่ตอบ

### Diagnosis Steps

1. ตรวจสอบ server status:
   ```bash
   # ตรวจสอบ process
   pm2 status
   docker ps
   
   # ตรวจสอบ logs
   pm2 logs line-bot --lines 100
   docker logs line-bot --tail 100
   ```

2. ตรวจสอบ webhook endpoint:
   ```bash
   # Test webhook endpoint
   curl -X POST https://your-domain.com/webhook \
     -H "Content-Type: application/json" \
     -d '{"events":[]}'
   ```

3. ตรวจสอบ LINE Channel settings:
   - เข้า LINE Developer Console
   - ตรวจสอบ Webhook URL ถูกต้อง
   - ตรวจสอบ Webhook ถูก Enable
   - Click "Verify" ดู status

4. ตรวจสอบ SSL certificate:
   ```bash
   curl -I https://your-domain.com/webhook
   openssl s_client -connect your-domain.com:443 -servername your-domain.com
   ```

5. ตรวจสอบ database connection:
   ```bash
   psql $DATABASE_URL -c "SELECT 1;"
   redis-cli -u $REDIS_URL ping
   ```

### Resolution Steps

หาก server down:
1. Restart service: `pm2 restart line-bot` หรือ `docker restart line-bot`
2. ถ้าไม่แก้ไข deploy ใหม่: `./scripts/deploy.sh`

หาก SSL issue:
1. Renew certificate: `certbot renew`
2. Reload nginx: `nginx -s reload`

หาก database issue:
1. Restart PostgreSQL: `systemctl restart postgresql`
2. ตรวจสอบ disk space: `df -h`

### Post-Incident
1. บันทึก RCA (Root Cause Analysis)
2. อัพเดท monitoring alerts
3. Update runbook
```

### 8.3 Post-Incident Report Template

```markdown
# Post-Incident Report

## Incident Summary

| Field | Value |
|-------|-------|
| Incident ID | INC-2025-001 |
| Severity | P1 |
| Start Time | 2025-01-15 14:30 ICT |
| End Time | 2025-01-15 15:45 ICT |
| Duration | 75 minutes |
| Affected Service | LINE Webhook |
| Impact | Users ไม่สามารถส่งข้อความได้ |

## Timeline

| เวลา | เหตุการณ์ |
|------|---------|
| 14:30 | Alert triggered: Webhook response rate = 0% |
| 14:32 | On-call engineer ได้รับ PagerDuty alert |
| 14:35 | On-call เริ่ม investigate |
| 14:45 | Root cause identified: SSL certificate expired |
| 14:50 | Started certificate renewal process |
| 15:40 | Certificate renewed, webhook restored |
| 15:45 | Incident resolved, monitoring confirmed |

## Root Cause

SSL certificate expired เนื่องจาก auto-renewal cron job ล้มเหลวเมื่อ 2 สัปดาห์ก่อน

## Impact

- ผู้ใช้ที่ได้รับผลกระทบ: ~1,200 users (active users ช่วง 14:30-15:45)
- Messages lost: 347 messages ที่ LINE retry ไม่สำเร็จ
- Revenue impact: ฿2,500 (estimated lost orders)

## Resolution

1. Renewed SSL certificate manually
2. Fixed auto-renewal cron job
3. Added monitoring for certificate expiry

## Action Items

| Action | Owner | Due Date | Priority |
|--------|-------|----------|----------|
| Add SSL cert expiry monitoring | DevOps | 2025-01-20 | P1 |
| Fix certbot cron job | DevOps | 2025-01-17 | P1 |
| Add LINE webhook health check | Backend | 2025-01-22 | P2 |
| Document SSL renewal process | DevOps | 2025-01-25 | P3 |

## Lessons Learned

- ควรมี monitoring สำหรับ SSL certificate expiry
- Auto-renewal failures ควร alert ทันที
- ควรมี backup process สำหรับ manual renewal
```

---

## 9. Sprint Planning สำหรับ LINE Bot Features

### 9.1 Sprint Setup

```
Sprint Configuration:
═════════════════════

Sprint Duration: 2 สัปดาห์
Sprint Capacity:
  - Developer (Senior): 8 story points/sprint
  - Developer (Mid): 6 story points/sprint
  - UX Designer: 5 story points/sprint
  - Total Team: ~25-30 points/sprint

Story Point Scale:
  1 point  = < 2 ชั่วโมง (trivial)
  2 points = 2-4 ชั่วโมง (small)
  3 points = 4-8 ชั่วโมง (medium)
  5 points = 1-2 วัน (large)
  8 points = 2-3 วัน (extra large)
  13 points = 1 สัปดาห์ (epic, ควรแตก)

Sprint Ceremonies:
├── Sprint Planning: ทุก 2 สัปดาห์, เช้าวันจันทร์ (2 ชั่วโมง)
├── Daily Standup: ทุกวัน 09:30 (15 นาที)
├── Sprint Review: ทุก 2 สัปดาห์, บ่ายวันศุกร์ (1 ชั่วโมง)
└── Sprint Retrospective: หลัง Sprint Review (45 นาที)
```

### 9.2 User Story Template สำหรับ LINE Bot

```markdown
## User Story

**Title:** Rich Menu ที่ปรับตาม User Segment

**As a** ลูกค้าที่เป็น Premium member,
**I want** เห็น Rich Menu ที่มีตัวเลือก exclusive สำหรับ Premium,
**So that** ฉันสามารถเข้าถึง features พิเศษได้ง่ายขึ้น

## Acceptance Criteria

- [ ] ผู้ใช้ที่มี tag "premium" เห็น Rich Menu Version B ที่มีปุ่ม "VIP Lounge"
- [ ] ผู้ใช้ทั่วไปเห็น Rich Menu Version A แบบปกติ
- [ ] การเปลี่ยน Rich Menu เกิดขึ้นภายใน 5 วินาที หลัง user ได้รับ tag
- [ ] Rich Menu ทำงานได้ทั้ง iOS และ Android
- [ ] มี analytics tracking สำหรับ Rich Menu interactions

## Technical Notes

- ใช้ LINE Per-user Rich Menu API
- Cache user segment ใน Redis (TTL 30 นาที)
- Webhook สำหรับ segment change event

## Definition of Done

- [ ] Code ผ่าน code review
- [ ] Unit tests เขียนครอบคลุม
- [ ] ทดสอบกับ LINE app จริง (iOS + Android)
- [ ] Analytics tracking ทำงาน
- [ ] Documentation อัพเดทแล้ว
- [ ] Product Owner approved

## Story Points: 5

## Sprint: Sprint 23
```

### 9.3 Sprint Velocity Tracking

```javascript
// src/project/velocityTracker.js
class SprintVelocityTracker {
  constructor(db) {
    this.db = db;
  }

  async recordSprintCompletion(sprintData) {
    await this.db.query(`
      INSERT INTO sprints (
        sprint_number,
        start_date,
        end_date,
        planned_points,
        completed_points,
        stories_planned,
        stories_completed,
        bugs_fixed,
        team_size
      ) VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9)
    `, [
      sprintData.number,
      sprintData.startDate,
      sprintData.endDate,
      sprintData.plannedPoints,
      sprintData.completedPoints,
      sprintData.storiesPlanned,
      sprintData.storiesCompleted,
      sprintData.bugsFixed,
      sprintData.teamSize,
    ]);
  }

  async getVelocityTrend(lastNSprints = 6) {
    const { rows } = await this.db.query(`
      SELECT 
        sprint_number,
        planned_points,
        completed_points,
        ROUND(completed_points::NUMERIC / planned_points * 100, 2) as completion_rate,
        stories_completed,
        bugs_fixed,
        team_size
      FROM sprints
      ORDER BY sprint_number DESC
      LIMIT $1
    `, [lastNSprints]);

    const avgVelocity = rows.reduce((sum, r) => sum + r.completed_points, 0) / rows.length;
    const trend = rows.length >= 3 
      ? rows[0].completed_points - rows[rows.length - 1].completed_points > 0 
        ? 'improving' 
        : 'declining'
      : 'unknown';

    return {
      sprints: rows,
      avgVelocity: Math.round(avgVelocity),
      trend,
      recommendation: Math.round(avgVelocity * 0.9), // Plan at 90% of avg
    };
  }
}

module.exports = SprintVelocityTracker;
```

---

## 10. KPIs สำหรับ LINE OA Team

### 10.1 Team KPI Dashboard

```javascript
// src/metrics/teamKPIs.js
const TEAM_KPIS = {
  // Product KPIs
  product: {
    followerGrowthRate: {
      target: 10,  // % per month
      unit: '%',
      frequency: 'monthly',
      owner: 'LINE OA Manager',
      calculation: 'NEW_FOLLOWERS / PREV_MONTH_FOLLOWERS * 100',
    },
    messageOpenRate: {
      target: 50,  // %
      unit: '%',
      frequency: 'per campaign',
      owner: 'Content Team',
      calculation: 'OPENED / SENT * 100',
    },
    campaignROI: {
      target: 200,  // %
      unit: '%',
      frequency: 'per campaign',
      owner: 'LINE OA Manager',
      calculation: '(REVENUE - COST) / COST * 100',
    },
    customerSatisfactionScore: {
      target: 4.0,  // out of 5
      unit: '/5',
      frequency: 'weekly',
      owner: 'CS Team',
      calculation: 'AVG(satisfaction_score)',
    },
  },
  
  // Technical KPIs
  technical: {
    systemUptime: {
      target: 99.9,  // %
      unit: '%',
      frequency: 'monthly',
      owner: 'DevOps',
      calculation: 'UPTIME_MINUTES / TOTAL_MINUTES * 100',
    },
    webhookResponseTime: {
      target: 200,  // ms
      unit: 'ms',
      frequency: 'daily',
      owner: 'Backend Team',
      calculation: 'P95 response time',
    },
    deploymentFrequency: {
      target: 4,  // per month
      unit: 'deployments/month',
      frequency: 'monthly',
      owner: 'Tech Lead',
      calculation: 'COUNT(deployments)',
    },
    mttr: {  // Mean Time To Recovery
      target: 30,  // minutes
      unit: 'minutes',
      frequency: 'per incident',
      owner: 'DevOps',
      calculation: 'AVG(resolution_time)',
    },
    codeReviewTurnaround: {
      target: 24,  // hours
      unit: 'hours',
      frequency: 'weekly',
      owner: 'Tech Lead',
      calculation: 'AVG(time from PR open to merge)',
    },
  },
  
  // Service KPIs
  service: {
    firstResponseTime: {
      target: 3,  // minutes
      unit: 'minutes',
      frequency: 'daily',
      owner: 'CS Manager',
      calculation: 'AVG(time_to_first_reply)',
    },
    resolutionTime: {
      target: 24,  // hours
      unit: 'hours',
      frequency: 'daily',
      owner: 'CS Manager',
      calculation: 'AVG(time_to_resolve)',
    },
    firstContactResolutionRate: {
      target: 75,  // %
      unit: '%',
      frequency: 'weekly',
      owner: 'CS Manager',
      calculation: 'RESOLVED_IN_ONE_CONTACT / TOTAL * 100',
    },
    botResolutionRate: {
      target: 60,  // % resolved by bot without human
      unit: '%',
      frequency: 'weekly',
      owner: 'Bot Developer',
      calculation: 'BOT_RESOLVED / TOTAL * 100',
    },
  },
  
  // Business KPIs
  business: {
    monthlyActiveUsers: {
      target_growth: 15,  // % growth
      unit: 'users',
      frequency: 'monthly',
      owner: 'Product Manager',
    },
    revenueFromLINE: {
      unit: '฿',
      frequency: 'monthly',
      owner: 'Business',
      calculation: 'SUM(orders from LINE source)',
    },
    costPerAcquisition: {
      target_max: 500,  // ฿ per new customer
      unit: '฿',
      frequency: 'monthly',
      owner: 'Marketing',
      calculation: 'TOTAL_LINE_COST / NEW_CUSTOMERS',
    },
    customerLifetimeValue: {
      target: 3000,  // ฿
      unit: '฿',
      frequency: 'quarterly',
      owner: 'Analytics',
      calculation: 'AVG(customer total spending)',
    },
  },
};

module.exports = TEAM_KPIS;
```

---

## 11. Training Plan สำหรับ New Team Members

### 11.1 Onboarding Checklist

```
New Team Member Onboarding Checklist:
═══════════════════════════════════════

Week 1: Environment Setup & Fundamentals
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Day 1:
□ Welcome meeting กับ team
□ รับ equipment และ accounts
   □ GitHub account + access
   □ LINE Developer Account
□ ติดตั้ง development environment
□ อ่าน Company/Team handbook
□ Setup password manager

Day 2:
□ อ่าน LINE Messaging API documentation
□ สร้าง LINE OA test account
□ Run project locally
□ ทำความเข้าใจ architecture overview

Day 3-4:
□ LINE API Fundamentals:
   □ Webhook concepts
   □ Message types
   □ Reply vs Push vs Multicast
   □ Rich Message formats
□ ทำ Hello World bot
□ ทำความเข้าใจ codebase structure

Day 5:
□ Code review shadows (ดู 2-3 PRs)
□ รู้จัก deployment process
□ รู้จัก monitoring tools

Week 2: Deep Dive
━━━━━━━━━━━━━━━━━

□ LIFF development fundamentals
□ Database schema และ ORM
□ Testing practices
□ CI/CD pipeline
□ Incident response procedures
□ Complete first small task

Week 3-4: Hands-on Project
━━━━━━━━━━━━━━━━━━━━━━━━━━

□ Own และ complete first feature
□ Submit first PR for review
□ Participate in sprint ceremonies
□ Shadow on-call rotation
□ Present work ใน Sprint Review

Month 2: Independence
━━━━━━━━━━━━━━━━━━━

□ Lead first feature development
□ Mentor newer team member
□ Contribute to documentation
□ First on-call rotation
□ 30-day check-in กับ manager
```

### 11.2 Learning Resources

```
LINE Bot Development Learning Path:
════════════════════════════════════

ขั้นพื้นฐาน (สัปดาห์ 1-2):
┌────────────────────────────────────────────┐
│ 1. LINE Official Account Introduction     │
│    - https://developers.line.biz/en/      │
│    - เวลา: 4-6 ชั่วโมง                   │
│                                           │
│ 2. LINE Messaging API Reference           │
│    - Message types ทั้งหมด               │
│    - Webhook events                       │
│    - เวลา: 8-10 ชั่วโมง                  │
│                                           │
│ 3. LINE SDK (Node.js)                     │
│    - npm: @line/bot-sdk                   │
│    - GitHub examples                      │
│    - เวลา: 4-6 ชั่วโมง                   │
└────────────────────────────────────────────┘

ขั้นกลาง (สัปดาห์ 3-4):
┌────────────────────────────────────────────┐
│ 1. Flex Message                           │
│    - Flex Message Simulator               │
│    - Advanced layouts                     │
│    - เวลา: 6-8 ชั่วโมง                   │
│                                           │
│ 2. LIFF (LINE Front-end Framework)        │
│    - LIFF documentation                   │
│    - React + LIFF integration             │
│    - เวลา: 8-10 ชั่วโมง                  │
│                                           │
│ 3. LINE Login                             │
│    - OAuth 2.0 flow                       │
│    - User profile access                  │
│    - เวลา: 4-6 ชั่วโมง                   │
└────────────────────────────────────────────┘

ขั้นสูง (เดือน 2):
┌────────────────────────────────────────────┐
│ 1. LINE Pay Integration                   │
│    - Payment flow                         │
│    - Sandbox testing                      │
│                                           │
│ 2. Advanced Analytics                     │
│    - LINE OA Insights API                 │
│    - Custom analytics                     │
│                                           │
│ 3. AI Integration                         │
│    - OpenAI / Claude API                  │
│    - Prompt engineering                   │
│                                           │
│ 4. Production Best Practices              │
│    - Scaling                              │
│    - Security                             │
│    - Cost optimization                    │
└────────────────────────────────────────────┘

Internal Training Materials:
├── /docs/architecture.md
├── /docs/development-guide.md
├── /docs/deployment.md
├── /docs/incident-runbook.md
└── /docs/coding-standards.md
```

---

## 12. Complete Team Handbook Template

### 12.1 Team Handbook

```markdown
# LINE OA Development Team Handbook

## ยินดีต้อนรับเข้าสู่ทีม! 🎉

handbook นี้เป็นคู่มือสำหรับสมาชิกทุกคนในทีม LINE OA Development

---

## ค่านิยมของทีม

1. **Customer First** - ทุกการตัดสินใจต้องพิจารณาผลกระทบต่อลูกค้า
2. **Quality over Speed** - ดีกว่าเร็วแต่มีปัญหา
3. **Collaboration** - แชร์ความรู้, ช่วยเหลือกัน
4. **Continuous Improvement** - เรียนรู้จาก mistakes
5. **Data-Driven** - ตัดสินใจด้วยข้อมูล

---

## Working Hours & Communication

### เวลาทำงาน
- Core Hours: 10:00 - 17:00 (Mon-Fri)
- Flexible: 09:00 - 09:59 หรือ 17:01 - 20:00
- Remote: ได้ทุกวัน แต่ต้อง Available ใน core hours

### Communication Channels
| ช่องทาง | ใช้สำหรับ | SLA |
|---------|----------|-----|
| Slack #general | Team announcements | Read within 4h |
| Slack #dev | Technical discussions | Response within 2h |
| Slack #urgent | P1/P2 issues | Response within 15min |
| Email | Formal communications | Response within 24h |
| Jira | Task tracking | Update daily |

### Meeting Norms
- ทุก meeting มี agenda ล่วงหน้า 24 ชั่วโมง
- เริ่มตรงเวลาเสมอ
- ถ้าไม่จำเป็น ลด meeting ลง
- บันทึก action items ทุก meeting
- Async first - ใช้ written communication ก่อน

---

## Development Practices

### Code Standards
- Language: TypeScript (preferred), JavaScript
- Formatter: Prettier (config ใน .prettierrc)
- Linter: ESLint (config ใน .eslintrc)
- Commit format: Conventional Commits

### Branching
- Main: production code
- Develop: integration branch
- Feature: feature/{ticket-num}-{description}
- Hotfix: hotfix/{ticket-num}-{description}

### Code Review
- ทุก PR ต้องมี 2 reviewers
- Review ภายใน 24 business hours
- ใช้ PR template
- ห้าม merge ถ้า CI ล้มเหลว

### Testing
- Unit tests ต้องมีสำหรับทุก business logic
- Coverage target: 80%
- Integration tests สำหรับ API endpoints
- E2E tests สำหรับ critical user flows

---

## Deployments

### Environments
| Environment | Branch | URL | Auto-deploy |
|-------------|--------|-----|-------------|
| Development | develop | dev.linebot.com | ✅ On merge |
| Staging | release/* | staging.linebot.com | ✅ On merge |
| Production | main | linebot.com | ❌ Manual |

### Production Deployment Process
1. Create release branch จาก develop
2. Final testing บน staging
3. Product Owner approval
4. Create PR: release → main
5. Code review (2 approvals required)
6. Merge PR
7. Manual trigger production deployment
8. Post-deployment monitoring (30 นาที)
9. Update CHANGELOG.md

### Rollback Procedure
1. Identify issue
2. Notify team ใน Slack
3. Execute: `./scripts/rollback.sh <version>`
4. Verify rollback สำเร็จ
5. Post incident report

---

## On-Call

### Rotation
- ทุกคนหมุน on-call
- หนึ่งคนต่อสัปดาห์
- เริ่ม Monday 09:00, จบ Monday 09:00 สัปดาห์ถัดไป

### Responsibilities
- ตอบ PagerDuty alerts ภายใน 15 นาที
- ทำ incident runbook
- Post incident report ภายใน 24 ชั่วโมง

### Compensation
- Allowance: ฿2,000/week
- Incident resolved (P1): ฿500 bonus
- Comp day: 1 วัน ต่อ major incident

---

## Growth & Development

### Annual Review Process
1. Self-assessment (เดือน 11)
2. Peer feedback
3. Manager review meeting
4. Career development plan
5. Compensation review

### Learning Budget
- ทุกคนได้รับ ฿20,000/ปี สำหรับ:
  - Online courses (Udemy, Coursera, etc.)
  - Conference tickets
  - Books
  - Certifications

### Tech Talks
- ทุก 2 สัปดาห์: ใครก็ได้ present 20 นาที
- Topics: new tech, project retrospective, tutorials
- บันทึก video ไว้สำหรับทีมที่ไม่ได้เข้า

---

## Benefits & Perks

### Leave Policy
- Annual Leave: 15 วัน/ปี
- Sick Leave: ไม่จำกัด (ต้องมี doctor's note > 3 วัน)
- Remote Work: ไม่จำกัด (core hours required)
- Parental Leave: 90 วัน (primary), 14 วัน (secondary)
- Study Leave: 5 วัน/ปี

### Equipment
- Laptop: MacBook Pro 16" (หรือเทียบเท่า)
- Monitor: 27" 4K
- Peripherals: keyboard, mouse, headset

### Workstation Allowance
- Home office setup: ฿15,000 (ครั้งแรก)
- Annual refresh: ฿5,000

---

## Contact Directory

| Role | ชื่อ | Email | LINE |
|------|------|-------|------|
| CTO | [Name] | cto@company.com | @cto |
| Tech Lead | [Name] | techlead@company.com | @lead |
| LINE OA Manager | [Name] | lineoa@company.com | @manager |
| DevOps Lead | [Name] | devops@company.com | @devops |
| On-call (current) | Rotates | oncall@company.com | @oncall |

---

*Handbook version 3.2 | Updated: 2025-01*
*ส่ง feedback ได้ที่ #handbook-feedback บน Slack*
```

---

## สรุปบทที่ 99

ในบทนี้เราได้เรียนรู้การสร้างและบริหารทีม LINE Bot Development ที่มีประสิทธิภาพ:

1. **Team Structure** - จัดทีมตาม scale ของธุรกิจ
2. **Roles** - กำหนด responsibility ชัดเจนสำหรับทุก role
3. **Development Workflow** - กระบวนการพัฒนาแบบ structured
4. **Git Strategy** - GitFlow สำหรับ LINE Bot projects
5. **Code Review** - Standards และ checklist สำหรับ review
6. **Documentation** - Template และมาตรฐาน
7. **On-call** - Rotation และ compensation
8. **Incident Management** - Severity levels และ runbooks
9. **Sprint Planning** - Agile สำหรับ LINE Bot features
10. **KPIs** - วัดผลทีมในทุกมิติ
11. **Training** - แผนการพัฒนาสมาชิกใหม่
12. **Team Handbook** - เอกสารครบสำหรับทั้งทีม

### Checklist การสร้างทีม

- [ ] กำหนด team structure ตาม business size
- [ ] เขียน job descriptions สำหรับทุก role
- [ ] สร้าง development workflow
- [ ] Setup Git branching strategy
- [ ] สร้าง PR template และ code review checklist
- [ ] เขียน documentation standards
- [ ] Setup on-call rotation
- [ ] สร้าง incident runbooks
- [ ] กำหนด KPIs สำหรับทุก role
- [ ] สร้าง onboarding checklist
- [ ] เขียน team handbook
- [ ] Setup training materials
