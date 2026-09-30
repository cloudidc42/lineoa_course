# Part 99: Team Structure & Development Process สำหรับ LINE Bot

## บทนำ: ทำไมต้องมี Process ที่ดี?

การพัฒนา LINE Bot ในระดับ Production ไม่ใช่แค่การเขียนโค้ดคนเดียว แต่ต้องการทีมงานที่มีความเชี่ยวชาญหลากหลาย กระบวนการทำงานที่ชัดเจน และวัฒนธรรมองค์กรที่เอื้อต่อการพัฒนา บทนี้จะครอบคลุมทุกแง่มุมของการสร้างทีมและกระบวนการทำงานที่มีประสิทธิภาพ

```
                    LINE OA Team Structure
    ┌─────────────────────────────────────────────────┐
    │                  Product Owner                   │
    │              (LINE OA Manager)                   │
    └─────────────────────┬───────────────────────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
   ┌─────────┐      ┌──────────┐      ┌──────────┐
   │  Tech   │      │  Design  │      │  Data    │
   │  Lead   │      │  Lead    │      │  Lead    │
   └────┬────┘      └────┬─────┘      └────┬─────┘
        │                │                 │
   ┌────┴────┐      ┌────┴────┐      ┌────┴─────┐
   │Backend  │      │UX/UI    │      │Analytics │
   │Frontend │      │Designer │      │Engineer  │
   │DevOps   │      │         │      │          │
   │QA       │      │         │      │          │
   └─────────┘      └─────────┘      └──────────┘
```

---

## 1. โครงสร้างทีมและบทบาทหน้าที่

### 1.1 LINE OA Manager (ผู้จัดการ LINE OA)

LINE OA Manager คือสะพานเชื่อมระหว่างธุรกิจและเทคโนโลยี มีหน้าที่กำหนดทิศทางและวัตถุประสงค์ของ LINE OA

**ความรับผิดชอบหลัก:**

```markdown
## LINE OA Manager - Job Description

### หน้าที่และความรับผิดชอบ

1. **กลยุทธ์และวิสัยทัศน์**
   - กำหนด Product Roadmap ของ LINE OA
   - วิเคราะห์ตลาดและคู่แข่ง
   - กำหนด KPI และ OKR สำหรับทีม
   - ประสานงานกับผู้บริหารระดับสูง

2. **การจัดการ Content**
   - วางแผน Content Calendar
   - อนุมัติ Rich Menu และ Broadcast
   - ดูแลโทนและ Brand Voice
   - วิเคราะห์ประสิทธิภาพ Content

3. **การบริหารความสัมพันธ์**
   - จัดการความสัมพันธ์กับ LINE Thailand
   - ประสานงานกับทีม Marketing
   - ดูแลการร้องเรียนจากลูกค้าในระดับ Escalation

4. **Budget Management**
   - จัดสรรงบประมาณสำหรับ LINE Messaging API
   - ติดตาม ROI ของ LINE OA
   - วางแผนการลงทุนในเทคโนโลยีใหม่

### ทักษะที่จำเป็น
- เข้าใจ LINE Ecosystem อย่างลึกซึ้ง
- มีประสบการณ์ด้าน Digital Marketing
- สามารถวิเคราะห์ข้อมูลได้
- มีทักษะการสื่อสารที่ดี
- เข้าใจ Technical concepts พื้นฐาน

### KPI
- จำนวนผู้ติดตาม LINE OA (Monthly Growth Rate)
- อัตราการเปิดอ่าน Broadcast
- Customer Satisfaction Score (CSAT)
- Revenue Generated from LINE OA
- Bot Automation Rate
```

### 1.2 Bot Developer (นักพัฒนา Bot)

```markdown
## Bot Developer Roles

### 1. Backend Bot Developer

**ความรับผิดชอบ:**
- พัฒนา Webhook Handler
- ออกแบบและพัฒนา Business Logic
- ผสานรวมกับ LINE Messaging API
- พัฒนา API endpoints
- ดูแล Database design
- Performance optimization

**Tech Stack ที่ต้องรู้:**
- Node.js / Python / Java / Go
- Express / Fastify / FastAPI
- PostgreSQL / MySQL / MongoDB
- Redis / Memcached
- Docker / Kubernetes
- AWS / GCP / Azure

**ตัวอย่าง Code ที่ต้องเขียนได้:**
```typescript
// Webhook Handler - ต้องเข้าใจและเขียนได้
import { Client, WebhookEvent, MessageEvent } from '@line/bot-sdk';
import { injectable, inject } from 'inversify';
import { MessageProcessor } from './processors/MessageProcessor';
import { EventRouter } from './routers/EventRouter';
import { Logger } from './utils/Logger';

@injectable()
export class WebhookController {
  constructor(
    @inject('LineClient') private client: Client,
    @inject('MessageProcessor') private messageProcessor: MessageProcessor,
    @inject('EventRouter') private eventRouter: EventRouter,
    @inject('Logger') private logger: Logger
  ) {}

  async handleEvents(events: WebhookEvent[]): Promise<void> {
    const promises = events.map(event => this.handleEvent(event));
    await Promise.allSettled(promises);
  }

  private async handleEvent(event: WebhookEvent): Promise<void> {
    try {
      await this.eventRouter.route(event);
    } catch (error) {
      this.logger.error('Event processing failed', {
        eventType: event.type,
        error: error instanceof Error ? error.message : 'Unknown error',
        userId: 'source' in event ? event.source.userId : undefined
      });
      
      // ส่ง error message กลับถ้าเป็น message event
      if (event.type === 'message' && 'replyToken' in event) {
        await this.client.replyMessage(event.replyToken, {
          type: 'text',
          text: 'ขออภัย เกิดข้อผิดพลาด กรุณาลองใหม่อีกครั้ง'
        });
      }
    }
  }
}
```

### 2. Frontend / LIFF Developer

**ความรับผิดชอบ:**
- พัฒนา LIFF Applications
- ออกแบบ Rich Menu
- สร้าง Flex Message Templates
- Responsive Design
- Performance optimization

**Tech Stack ที่ต้องรู้:**
- React / Vue.js / Next.js
- LIFF SDK
- TypeScript
- TailwindCSS / Material-UI
- Webpack / Vite
- Jest / Testing Library
```

### 1.3 UX/UI Designer

```markdown
## UX/UI Designer สำหรับ LINE Bot

### บทบาทพิเศษในระบบ LINE

LINE Bot มีข้อจำกัดด้าน UI ที่แตกต่างจาก Web App ทั่วไป Designer ต้องเข้าใจ:

1. **LINE UI Components**
   - Flex Message layouts
   - Quick Reply buttons
   - Rich Menu design
   - Carousel และ Image Map
   - LIFF page design

2. **Conversational UX**
   - การออกแบบ Conversation Flow
   - Persona ของ Bot
   - Error messages ที่เป็นมิตร
   - Onboarding experience

3. **Design System สำหรับ LINE**

```json
// Design Tokens สำหรับ Flex Message
{
  "colors": {
    "primary": "#06C755",
    "primaryLight": "#E8FFF1",
    "secondary": "#00B900",
    "text": {
      "primary": "#111111",
      "secondary": "#555555",
      "disabled": "#AAAAAA"
    },
    "background": {
      "primary": "#FFFFFF",
      "secondary": "#F7F7F7"
    },
    "semantic": {
      "success": "#06C755",
      "warning": "#FF9500",
      "error": "#FF3B30",
      "info": "#007AFF"
    }
  },
  "typography": {
    "sizes": {
      "xxl": "20px",
      "xl": "18px",
      "lg": "16px",
      "md": "14px",
      "sm": "12px",
      "xs": "10px"
    },
    "weights": {
      "bold": "bold",
      "regular": "regular"
    }
  },
  "spacing": {
    "xs": "4px",
    "sm": "8px",
    "md": "12px",
    "lg": "16px",
    "xl": "24px"
  }
}
```

**Deliverables:**
- Conversation Flow Diagrams
- Wireframes สำหรับ LIFF pages
- Flex Message designs (Figma)
- Rich Menu layouts
- Design System documentation
- Usability test results
```

### 1.4 Data Analyst

```python
## Data Analyst บน LINE OA

### ความรับผิดชอบ
# วิเคราะห์ข้อมูลเพื่อปรับปรุง LINE OA performance

import pandas as pd
import numpy as np
from datetime import datetime, timedelta
import matplotlib.pyplot as plt
import seaborn as sns

class LineOAAnalytics:
    """
    Class สำหรับวิเคราะห์ข้อมูล LINE OA
    """
    
    def __init__(self, db_connection):
        self.db = db_connection
        
    def calculate_retention_rate(self, cohort_date: datetime) -> pd.DataFrame:
        """
        คำนวณ Retention Rate ของผู้ใช้
        """
        query = """
            WITH cohort AS (
                SELECT 
                    user_id,
                    DATE(first_interaction) as cohort_date,
                    DATE(interaction_date) as activity_date,
                    DATEDIFF(DATE(interaction_date), DATE(first_interaction)) as days_since_first
                FROM (
                    SELECT 
                        user_id,
                        interaction_date,
                        MIN(interaction_date) OVER (PARTITION BY user_id) as first_interaction
                    FROM user_interactions
                ) t
                WHERE DATE(first_interaction) = %s
            )
            SELECT 
                days_since_first,
                COUNT(DISTINCT user_id) as users,
                COUNT(DISTINCT user_id) * 100.0 / MAX(COUNT(DISTINCT user_id)) OVER () as retention_rate
            FROM cohort
            GROUP BY days_since_first
            ORDER BY days_since_first
        """
        
        df = pd.read_sql(query, self.db, params=[cohort_date])
        return df
    
    def analyze_message_performance(self, 
                                     start_date: datetime,
                                     end_date: datetime) -> dict:
        """
        วิเคราะห์ประสิทธิภาพของ Bot Messages
        """
        query = """
            SELECT 
                message_type,
                COUNT(*) as total_sent,
                SUM(CASE WHEN clicked = 1 THEN 1 ELSE 0 END) as clicked,
                SUM(CASE WHEN converted = 1 THEN 1 ELSE 0 END) as converted,
                AVG(response_time_ms) as avg_response_time,
                SUM(CASE WHEN user_left_after = 1 THEN 1 ELSE 0 END) as drop_offs
            FROM bot_messages
            WHERE created_at BETWEEN %s AND %s
            GROUP BY message_type
        """
        
        df = pd.read_sql(query, self.db, params=[start_date, end_date])
        
        # คำนวณ Click-Through Rate และ Conversion Rate
        df['ctr'] = (df['clicked'] / df['total_sent'] * 100).round(2)
        df['conversion_rate'] = (df['converted'] / df['total_sent'] * 100).round(2)
        df['drop_off_rate'] = (df['drop_offs'] / df['total_sent'] * 100).round(2)
        
        return {
            'summary': df.to_dict(orient='records'),
            'best_message_type': df.loc[df['conversion_rate'].idxmax(), 'message_type'],
            'worst_drop_off': df.loc[df['drop_off_rate'].idxmax(), 'message_type']
        }
    
    def segment_users(self) -> pd.DataFrame:
        """
        แบ่ง Segment ผู้ใช้ตาม RFM Model
        """
        query = """
            SELECT 
                user_id,
                DATEDIFF(NOW(), MAX(interaction_date)) as recency,
                COUNT(*) as frequency,
                SUM(order_amount) as monetary
            FROM user_interactions
            LEFT JOIN orders USING(user_id)
            WHERE interaction_date >= DATE_SUB(NOW(), INTERVAL 90 DAY)
            GROUP BY user_id
        """
        
        df = pd.read_sql(query, self.db)
        
        # สร้าง RFM Score
        df['R_Score'] = pd.qcut(df['recency'], 5, labels=[5,4,3,2,1]).astype(int)
        df['F_Score'] = pd.qcut(df['frequency'].rank(method='first'), 5, labels=[1,2,3,4,5]).astype(int)
        df['M_Score'] = pd.qcut(df['monetary'].rank(method='first'), 5, labels=[1,2,3,4,5]).astype(int)
        
        df['RFM_Score'] = df['R_Score'].astype(str) + df['F_Score'].astype(str) + df['M_Score'].astype(str)
        
        # กำหนด Segment
        def assign_segment(row):
            rfm = row['RFM_Score']
            r, f, m = int(rfm[0]), int(rfm[1]), int(rfm[2])
            
            if r >= 4 and f >= 4 and m >= 4:
                return 'Champions'
            elif r >= 3 and f >= 3:
                return 'Loyal Customers'
            elif r >= 4 and f <= 2:
                return 'New Customers'
            elif r <= 2 and f >= 3:
                return 'At Risk'
            elif r <= 2 and f <= 2:
                return 'Lost Customers'
            else:
                return 'Potential'
        
        df['Segment'] = df.apply(assign_segment, axis=1)
        
        return df

### KPI ที่ Data Analyst ต้องติดตาม
# - DAU/MAU (Daily/Monthly Active Users)
# - Message Open Rate
# - Click-Through Rate (CTR)
# - Conversion Rate
# - Customer Acquisition Cost (CAC)
# - Customer Lifetime Value (CLV)
# - Churn Rate
# - Net Promoter Score (NPS)
```

### 1.5 DevOps Engineer

```yaml
## DevOps Engineer สำหรับ LINE Bot

# CI/CD Pipeline Configuration
name: LINE Bot CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  # Stage 1: Code Quality
  quality-check:
    name: Code Quality Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: Run TypeScript check
        run: npm run type-check
      
      - name: Run Tests
        run: npm run test:coverage
      
      - name: SonarQube Scan
        uses: SonarSource/sonarcloud-github-action@master
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}

  # Stage 2: Security Scan
  security-scan:
    name: Security Vulnerability Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run npm audit
        run: npm audit --audit-level=high
      
      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
      
      - name: Trivy container scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'line-bot:latest'
          format: 'sarif'
          output: 'trivy-results.sarif'

  # Stage 3: Build
  build:
    name: Build Docker Image
    needs: [quality-check, security-scan]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ secrets.REGISTRY_URL }}
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
      
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.REGISTRY_URL }}/line-bot:latest
            ${{ secrets.REGISTRY_URL }}/line-bot:${{ github.sha }}
          cache-from: type=registry,ref=${{ secrets.REGISTRY_URL }}/line-bot:buildcache
          cache-to: type=registry,ref=${{ secrets.REGISTRY_URL }}/line-bot:buildcache,mode=max

  # Stage 4: Deploy to Staging
  deploy-staging:
    name: Deploy to Staging
    needs: build
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - name: Deploy to Kubernetes (Staging)
        uses: azure/k8s-deploy@v1
        with:
          namespace: line-bot-staging
          manifests: |
            k8s/staging/deployment.yaml
            k8s/staging/service.yaml
          images: |
            ${{ secrets.REGISTRY_URL }}/line-bot:${{ github.sha }}
      
      - name: Run Smoke Tests
        run: |
          sleep 30
          npm run test:smoke -- --env=staging

  # Stage 5: Deploy to Production (manual approval)
  deploy-production:
    name: Deploy to Production
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://api.example.com
    steps:
      - name: Deploy to Kubernetes (Production)
        uses: azure/k8s-deploy@v1
        with:
          namespace: line-bot-production
          manifests: |
            k8s/production/deployment.yaml
            k8s/production/service.yaml
          images: |
            ${{ secrets.REGISTRY_URL }}/line-bot:${{ github.sha }}
          strategy: canary
          percentage: 10
```

### 1.6 QA Engineer

```typescript
## QA Engineer สำหรับ LINE Bot

### Testing Strategy

// Test Pyramid สำหรับ LINE Bot
/*
         /\
        /  \
       / E2E \        <- น้อยที่สุด (5-10%)
      /________\      Playwright / Cypress
     /          \
    / Integration \   <- ปานกลาง (20-30%)
   /______________\   Supertest / Jest
  /                \
 /   Unit Tests     \ <- มากที่สุด (60-70%)
/____________________\ Jest / Mocha
*/

// ตัวอย่าง Unit Test
import { describe, it, expect, beforeEach, jest } from '@jest/globals';
import { IntentClassifier } from '../src/services/IntentClassifier';
import { UserContext } from '../src/types';

describe('IntentClassifier', () => {
  let classifier: IntentClassifier;
  
  beforeEach(() => {
    classifier = new IntentClassifier();
  });
  
  describe('classify', () => {
    it('should classify greeting intent correctly', async () => {
      const messages = ['สวัสดี', 'หวัดดี', 'ดีครับ', 'hello', 'hi'];
      
      for (const message of messages) {
        const result = await classifier.classify(message);
        expect(result.intent).toBe('greeting');
        expect(result.confidence).toBeGreaterThan(0.8);
      }
    });
    
    it('should classify order intent with product name', async () => {
      const result = await classifier.classify('ขอสั่งน้ำแดง 2 แก้ว');
      
      expect(result.intent).toBe('order');
      expect(result.entities).toMatchObject({
        product: 'น้ำแดง',
        quantity: 2
      });
    });
    
    it('should return fallback for unknown intent', async () => {
      const result = await classifier.classify('xxxxxxx gibberish text xxxxx');
      
      expect(result.intent).toBe('unknown');
      expect(result.confidence).toBeLessThan(0.5);
    });
  });
});

// ตัวอย่าง Integration Test
describe('Webhook Handler Integration', () => {
  let app: Express;
  let lineClientMock: jest.MockedObject<Client>;
  
  beforeEach(() => {
    lineClientMock = createMockLineClient();
    app = createApp({ lineClient: lineClientMock });
  });
  
  it('should handle text message and reply', async () => {
    const event = createMockMessageEvent({
      type: 'text',
      text: 'สั่งกาแฟ'
    });
    
    const response = await request(app)
      .post('/webhook')
      .set('x-line-signature', generateSignature(event))
      .send({ events: [event] });
    
    expect(response.status).toBe(200);
    expect(lineClientMock.replyMessage).toHaveBeenCalledWith(
      event.replyToken,
      expect.objectContaining({
        type: 'text',
        text: expect.stringContaining('กาแฟ')
      })
    );
  });
});

// ตัวอย่าง E2E Test
describe('LINE Bot E2E Tests', () => {
  it('should complete order flow from start to finish', async () => {
    const testUser = await createTestLineUser();
    
    // Step 1: Send greeting
    await testUser.sendMessage('สวัสดี');
    const greeting = await testUser.waitForResponse();
    expect(greeting.text).toContain('สวัสดี');
    
    // Step 2: View menu
    await testUser.tapButton('ดูเมนู');
    const menu = await testUser.waitForResponse();
    expect(menu.type).toBe('flex');
    
    // Step 3: Add to cart
    await testUser.tapButton('เพิ่มใส่ตะกร้า');
    const cartConfirm = await testUser.waitForResponse();
    expect(cartConfirm.text).toContain('เพิ่มแล้ว');
    
    // Step 4: Checkout
    await testUser.sendMessage('ชำระเงิน');
    const payment = await testUser.waitForResponse();
    expect(payment.type).toBe('flex');
    expect(payment.altText).toContain('ชำระเงิน');
    
    // Verify order in database
    const order = await db.findLatestOrderByUserId(testUser.userId);
    expect(order.status).toBe('pending_payment');
  });
});
```

---

## 2. Development Workflow

### 2.1 Git Flow vs Trunk-Based Development

```
Git Flow Model:
═══════════════

main ──────────────────────────────────────────────────► (Production)
       │                          ▲
       │                          │ merge (release)
       ▼                          │
develop ──────────────────────────────────────────────►
       │          ▲       │                ▲
       │ branch   │merge  │ branch         │ merge
       ▼          │       ▼                │
  feature/xxx ────┘   release/1.2.0 ───────┘
                       │
                    hotfix ──► main (emergency fix)


Trunk-Based Development Model:
════════════════════════════════

main/trunk ────────────────────────────────────────────► (Always deployable)
              │   ▲   │    ▲    │   ▲
   short-lived│   │   │    │    │   │
   branches   ▼   │   ▼    │    ▼   │
         feat-a ──┘  feat-b ──┘ feat-c ──┘
         (< 2 days)
```

**เมื่อไหร่ใช้อะไร:**

| เงื่อนไข | แนะนำ |
|---------|-------|
| ทีมขนาดเล็ก (<5 คน) | Trunk-Based |
| Deployment บ่อย (หลายครั้ง/วัน) | Trunk-Based |
| ทีมใหญ่ หลาย Feature พร้อมกัน | Git Flow |
| มี Release Cycle ที่ชัดเจน | Git Flow |
| ต้องการ Hotfix ฉุกเฉิน | Git Flow |

### 2.2 Branch Naming Conventions

```bash
# Convention: <type>/<ticket-id>/<short-description>
# ตัวอย่าง Branch Names

# Feature branches
git checkout -b feature/LINE-123/add-flex-message-template
git checkout -b feature/LINE-456/implement-payment-flow
git checkout -b feature/LINE-789/user-segmentation

# Bug fix branches
git checkout -b fix/LINE-234/webhook-signature-validation
git checkout -b fix/LINE-567/message-encoding-thai-chars
git checkout -b hotfix/LINE-890/production-crash-null-pointer

# Refactoring
git checkout -b refactor/LINE-111/extract-message-processor
git checkout -b refactor/LINE-222/optimize-database-queries

# Documentation
git checkout -b docs/LINE-333/update-api-documentation

# Chore/Infrastructure
git checkout -b chore/LINE-444/upgrade-dependencies
git checkout -b chore/LINE-555/update-docker-base-image

# Release
git checkout -b release/v2.3.0
git checkout -b release/v2.3.1-hotfix

# Validation Script
validate_branch_name() {
  local branch_name="$1"
  local pattern="^(feature|fix|hotfix|refactor|docs|chore|release)\/[A-Z]+-[0-9]+\/[a-z0-9-]+$|^release\/v[0-9]+\.[0-9]+\.[0-9]+(-[a-z]+)?$"
  
  if [[ $branch_name =~ $pattern ]]; then
    echo "✓ Branch name is valid: $branch_name"
    return 0
  else
    echo "✗ Invalid branch name: $branch_name"
    echo "Format: <type>/<TICKET-ID>/<short-description>"
    echo "Types: feature, fix, hotfix, refactor, docs, chore, release"
    return 1
  fi
}
```

### 2.3 Commit Message Convention

```bash
# Conventional Commits Format
# <type>(<scope>): <subject>
#
# [optional body]
#
# [optional footer]

# ตัวอย่าง Good Commit Messages

git commit -m "feat(webhook): add signature validation middleware

Implement HMAC-SHA256 validation for incoming LINE webhooks
to prevent unauthorized requests.

- Add validateSignature() function
- Add middleware for Express
- Write unit tests for edge cases

Closes: LINE-123
Breaking-change: None"

git commit -m "fix(message): handle Thai character encoding in Flex Message

Thai characters were being corrupted when building Flex Message
JSON payload. Fixed by explicitly setting UTF-8 encoding.

Fixes: LINE-456"

git commit -m "perf(database): add index for user_id lookup

User profile queries were doing full table scans.
Added composite index on (user_id, created_at) reducing
query time from 2000ms to 15ms.

Ticket: LINE-789"

# Commit Types
# feat:     New feature
# fix:      Bug fix
# docs:     Documentation changes
# style:    Code style (formatting, no logic change)
# refactor: Code refactoring
# perf:     Performance improvement
# test:     Adding/updating tests
# chore:    Build process, tooling
# ci:       CI/CD configuration
# revert:   Reverting previous commit

# Git Hook สำหรับ validate commit message
cat > .git/hooks/commit-msg << 'EOF'
#!/bin/bash
commit_msg=$(cat "$1")
pattern="^(feat|fix|docs|style|refactor|perf|test|chore|ci|revert)(\(.+\))?: .{1,100}$"

if ! echo "$commit_msg" | grep -qE "$pattern"; then
  echo "ERROR: Commit message does not follow Conventional Commits format"
  echo "Format: <type>(<scope>): <subject>"
  echo "Example: feat(webhook): add signature validation"
  exit 1
fi
EOF
chmod +x .git/hooks/commit-msg
```

### 2.4 Pull Request Template

```markdown
# สร้างไฟล์ .github/pull_request_template.md

## Summary
<!-- อธิบายสั้นๆ ว่า PR นี้ทำอะไร -->

## Motivation & Context
<!-- ทำไมต้องทำ PR นี้? มี Ticket/Issue ไหนที่เกี่ยวข้อง? -->
Closes #<!-- Ticket number -->

## Type of Change
- [ ] Bug fix (non-breaking change that fixes an issue)
- [ ] New feature (non-breaking change that adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update
- [ ] Refactoring (no functional changes)
- [ ] Performance improvement

## Screenshots / Demo
<!-- สำหรับ UI changes โปรดแนบ screenshot -->
| Before | After |
|--------|-------|
| <!-- screenshot before --> | <!-- screenshot after --> |

## LINE Bot Specific Checks
- [ ] Tested on LINE mobile app (iOS)
- [ ] Tested on LINE mobile app (Android)
- [ ] Tested with Thai language input
- [ ] Tested Flex Message renders correctly
- [ ] Tested Rich Menu if modified
- [ ] Tested LIFF if modified

## Testing
- [ ] Unit tests pass (`npm run test`)
- [ ] Integration tests pass (`npm run test:integration`)
- [ ] Added new tests for new functionality
- [ ] No regression in existing tests

## Code Quality
- [ ] Code follows project style guide
- [ ] Self-review completed
- [ ] Comments added for complex logic
- [ ] No console.log() left in production code
- [ ] No hardcoded secrets or credentials

## Documentation
- [ ] API documentation updated (if applicable)
- [ ] README updated (if applicable)
- [ ] CHANGELOG updated

## Deployment Notes
<!-- มี migration, environment variable, หรือ configuration ที่ต้อง update ไหม? -->

## Checklist for Reviewer
- [ ] Logic is correct
- [ ] Edge cases handled
- [ ] Error handling is appropriate
- [ ] Performance considerations reviewed
- [ ] Security implications reviewed
```

---

## 3. Sprint Planning สำหรับ LINE Features

### 3.1 Sprint Structure (2 สัปดาห์)

```
Sprint Timeline (2 weeks / 10 working days)
═══════════════════════════════════════════

Day 1 (Monday)
├── Sprint Planning (3 hours)
│   ├── Review Sprint Goal
│   ├── Estimate User Stories (Planning Poker)
│   ├── Create Sprint Backlog
│   └── Assign Tasks

Day 2-9 (Daily)
├── Daily Standup (15 min)
│   ├── What did you do yesterday?
│   ├── What will you do today?
│   └── Any blockers?
├── Development work
└── Code Reviews

Day 8 (Thursday of week 2)
└── Code Freeze / QA Testing begins

Day 9 (Friday of week 2)
└── Bug fixing from QA

Day 10 (Friday)
├── Sprint Review / Demo (1 hour)
├── Retrospective (1 hour)
└── Sprint Deployment
```

### 3.2 Story Point Estimation

```markdown
## Story Point Guide สำหรับ LINE Bot Features

### 1 Point - Simple (1-2 ชั่วโมง)
- เปลี่ยนข้อความ Static
- เพิ่ม Quick Reply button ใหม่
- แก้ไข typo ใน message template
- อัพเดท Rich Menu text

### 2 Points - Small (half day)
- เพิ่ม Flex Message template ใหม่
- เพิ่ม handler สำหรับ keyword ใหม่
- อัพเดท API endpoint ง่ายๆ
- เพิ่ม unit test สำหรับ existing code

### 3 Points - Medium (1 วัน)
- Feature ใหม่ที่ต้องการ database query
- Integrate กับ external API
- สร้าง LIFF หน้าใหม่ simple
- Implement caching layer

### 5 Points - Large (2-3 วัน)
- Feature ซับซ้อนที่ต้องการหลาย components
- Payment integration
- User authentication flow
- Database schema changes

### 8 Points - Extra Large (3-5 วัน)
- ระบบ Chatbot ใหม่ทั้งหมด
- Migration ข้อมูลขนาดใหญ่
- Refactor major component
- New microservice

### 13 Points - Too Big → Split!
```

### 3.3 Feature Backlog Template

```markdown
## User Story Template

**Title:** [Feature Name]

**As a** [type of user],
**I want** [some goal],
**So that** [some reason/value]

---
**Story:** As a ลูกค้า,
I want สั่งอาหารผ่าน LINE Bot,
So that ฉันสามารถสั่งอาหารได้สะดวกโดยไม่ต้องโทรหา

**Acceptance Criteria:**
Given ฉันเปิด LINE OA ของร้านอาหาร
When ฉันพิมพ์ "สั่งอาหาร" หรือ กด Quick Reply
Then ฉันเห็นเมนูอาหารในรูปแบบ Carousel

Given ฉันกดเลือกเมนูอาหาร
When ฉันกด "เพิ่มใส่ตะกร้า"  
Then ระบบบันทึกรายการและแสดงจำนวนในตะกร้า

Given ฉันต้องการชำระเงิน
When ฉันกด "ชำระเงิน"
Then ฉันเห็น Flex Message สรุปรายการและราคา พร้อม PromptPay QR Code

**Technical Notes:**
- ใช้ Redis เก็บ shopping cart (TTL 24 ชั่วโมง)
- Integrate กับ PromptPay API
- ส่ง notification ให้ ร้านอาหารเมื่อมี order ใหม่

**Story Points:** 8
**Priority:** High
**Sprint:** Sprint 5
**Assigned To:** Backend: @john, Frontend: @jane
**Dependencies:** Payment gateway API docs

**Definition of Done:**
- [ ] Unit tests written and passing
- [ ] Integration tests passing
- [ ] Tested on LINE mobile
- [ ] Code reviewed
- [ ] Documentation updated
- [ ] QA sign-off received
- [ ] Deployed to staging
```

---

## 4. Code Review Checklist

```markdown
# Code Review Checklist - LINE Bot Development

## ✅ General Code Quality

### Correctness
- [ ] Logic ถูกต้องตาม requirements
- [ ] Edge cases ได้รับการ handle
- [ ] Error handling เหมาะสม
- [ ] ไม่มี potential infinite loop หรือ deadlock

### Readability
- [ ] ชื่อ variable/function สื่อความหมายชัดเจน
- [ ] มี comment สำหรับ complex logic
- [ ] Function มีขนาดเหมาะสม (< 50 lines)
- [ ] ไม่มี magic numbers

### Performance
- [ ] ไม่มี N+1 query
- [ ] ใช้ async/await ถูกต้อง
- [ ] มีการ cache ข้อมูลที่ควร cache
- [ ] ไม่มี memory leak

## ✅ LINE Bot Specific

### Webhook
- [ ] Validate LINE signature
- [ ] Handle ทุก event type ที่อาจเกิดขึ้น
- [ ] Reply token ถูกใช้ภายใน 60 วินาที
- [ ] ส่ง 200 OK ก่อนประมวลผล (async)

### Messages
- [ ] Flex Message JSON valid
- [ ] Alt text สำหรับ Flex Message
- [ ] ข้อความภาษาไทยถูกต้อง
- [ ] ไม่เกิน rate limit (500 messages/second)
- [ ] ขนาด Flex Message ไม่เกิน limit

### LIFF
- [ ] liff.init() เรียกก่อนใช้งาน
- [ ] Handle กรณีเปิดนอก LINE
- [ ] ข้อมูล user ได้จาก LIFF ไม่ใช่ client input
- [ ] HTTPS เสมอ

## ✅ Security

- [ ] ไม่มี hardcoded secrets
- [ ] Input validation ครบถ้วน
- [ ] SQL injection prevention (parameterized queries)
- [ ] ไม่ log ข้อมูล sensitive
- [ ] CORS configured ถูกต้อง

## ✅ Testing

- [ ] Unit tests ครอบคลุม happy path
- [ ] Unit tests ครอบคลุม error cases
- [ ] Test coverage ≥ 80%
- [ ] Mock ถูกต้อง (ไม่ call external services ใน unit test)

## ✅ Database

- [ ] Migration script ถูกต้อง
- [ ] Index เหมาะสม
- [ ] Transaction ใช้ถูกที่
- [ ] ไม่มี sensitive data ที่ไม่ encrypt

## Review Sign-off

| Reviewer | Status | Comment |
|----------|--------|---------|
| Tech Lead | ⬜ Pending | |
| Senior Dev | ⬜ Pending | |
| QA | ⬜ Pending | |
```

---

## 5. Documentation Standards

### 5.1 API Documentation Template

```typescript
/**
 * @api {post} /api/messages/broadcast Send Broadcast Message
 * @apiName BroadcastMessage
 * @apiGroup Messages
 * @apiVersion 2.0.0
 *
 * @apiDescription ส่ง Broadcast message ไปยัง followers ทั้งหมด
 * หรือ segment ที่กำหนด
 *
 * @apiHeader {String} Authorization Bearer <access_token>
 * @apiHeader {String} Content-Type application/json
 *
 * @apiParam {Object} message Message object
 * @apiParam {String} message.type Message type (text|flex|image)
 * @apiParam {String} [message.text] Text content (required if type=text)
 * @apiParam {Object} [message.contents] Flex Message contents
 * @apiParam {String[]} [targetSegmentIds] Segment IDs to target
 * @apiParam {Object} [schedule] Scheduling options
 * @apiParam {String} [schedule.sendAt] ISO 8601 datetime to send
 *
 * @apiExample {json} Request Example:
 * {
 *   "message": {
 *     "type": "text",
 *     "text": "สวัสดีครับ! มีโปรโมชั่นพิเศษวันนี้"
 *   },
 *   "targetSegmentIds": ["segment-vip", "segment-active"],
 *   "schedule": {
 *     "sendAt": "2024-12-25T09:00:00+07:00"
 *   }
 * }
 *
 * @apiSuccess {String} broadcastId Broadcast ID
 * @apiSuccess {Number} targetCount Number of recipients
 * @apiSuccess {String} status Status (scheduled|sent)
 * @apiSuccess {String} scheduledAt Scheduled time
 *
 * @apiSuccessExample {json} Success Response:
 * {
 *   "broadcastId": "bc_01234567890",
 *   "targetCount": 5420,
 *   "status": "scheduled",
 *   "scheduledAt": "2024-12-25T09:00:00+07:00"
 * }
 *
 * @apiError (400) InvalidMessage Message format is invalid
 * @apiError (401) Unauthorized Invalid or expired token
 * @apiError (429) RateLimitExceeded Too many requests
 *
 * @apiErrorExample {json} Error Response:
 * {
 *   "error": "INVALID_MESSAGE",
 *   "message": "Message type 'flex' requires 'contents' field",
 *   "code": 400
 * }
 */
async function broadcastMessage(req: Request, res: Response): Promise<void> {
  // implementation
}
```

### 5.2 ADR (Architecture Decision Record)

```markdown
# ADR-001: เลือกใช้ Redis สำหรับ Session Management

**Date:** 2024-01-15
**Status:** Accepted
**Deciders:** @tech-lead, @backend-dev-1, @backend-dev-2

## Context

LINE Bot ของเราต้องการ manage conversation state ของผู้ใช้ระหว่าง webhook calls เนื่องจาก HTTP เป็น stateless protocol เราจึงต้องการ external storage

## Decision

เลือกใช้ Redis เป็น session store

## Options Considered

### Option 1: Redis
- **Pros:** In-memory (เร็วมาก), support TTL, data structures หลายแบบ, highly available
- **Cons:** ต้องจ่ายค่า infrastructure, data loss if not configured properly

### Option 2: PostgreSQL
- **Pros:** ACID compliant, ไม่ต้องการ infrastructure เพิ่ม
- **Cons:** ช้ากว่า Redis 10-100x สำหรับ read/write บ่อยๆ

### Option 3: In-memory (Process)
- **Pros:** ง่ายที่สุด, เร็วที่สุด
- **Cons:** ไม่รองรับ horizontal scaling, data หายเมื่อ restart

## Reasoning

LINE Bot ต้องการ response time < 100ms เพื่อ reply ก่อน token หมดอายุ
Redis ให้ latency < 1ms ซึ่งเหมาะสมที่สุด

## Consequences

**Positive:**
- Session lookup เร็วมาก
- รองรับ horizontal scaling
- TTL management ง่าย

**Negative:**  
- ต้องจัดการ Redis availability
- เพิ่ม operational complexity
- ต้องมี backup strategy

## Implementation Notes

```typescript
const redisConfig = {
  host: process.env.REDIS_HOST,
  port: 6379,
  password: process.env.REDIS_PASSWORD,
  keyPrefix: 'linebot:session:',
  ttl: 3600 // 1 hour
};
```

**Reviewed By:** @team-lead
**Review Date:** 2024-01-16
```

---

## 6. On-Call Runbook Template

```markdown
# LINE Bot On-Call Runbook v2.0

## ข้อมูลสำคัญ

**Service:** LINE OA Bot - [ชื่อธุรกิจ]
**Production URL:** https://api.example.com
**Dashboard:** https://grafana.example.com
**Logs:** https://logs.example.com
**Runbook Updated:** 2024-01-15
**Owner:** Platform Team

---

## On-Call Schedule

| วันที่ | Primary | Secondary |
|-------|---------|-----------|
| จ-พ   | @john   | @jane     |
| พฤ-ศ  | @bob    | @alice    |
| เสาร์-อาทิตย์ | @charlie | @diana |

## การติดต่อฉุกเฉิน

| Level | ช่องทาง | Response Time |
|-------|---------|---------------|
| P1 - Critical | โทรศัพท์ | 15 นาที |
| P2 - High | LINE Group | 30 นาที |
| P3 - Medium | Slack | 2 ชั่วโมง |
| P4 - Low | Jira | Next business day |

---

## Incident Response Procedures

### Incident Severity Levels

```
P1 - Critical (ระดับวิกฤต)
────────────────────────────
ลักษณะ: Service down ทั้งหมด, ไม่สามารถ receive webhook ได้
ผลกระทบ: ลูกค้าทั้งหมดใช้ไม่ได้
Response Time: 15 นาที
Action: โทรหา On-Call ทันที, escalate ถึง CTO

P2 - High (ระดับสูง)
──────────────────────
ลักษณะ: Feature หลัก broken (Payment, Order)
ผลกระทบ: ลูกค้าส่วนใหญ่ได้รับผลกระทบ
Response Time: 30 นาที
Action: Notify on-call via LINE Group

P3 - Medium (ระดับกลาง)
──────────────────────────
ลักษณะ: Feature รองทำงานผิดปกติ
ผลกระทบ: ลูกค้าส่วนน้อยได้รับผลกระทบ
Response Time: 2 ชั่วโมง
Action: Create Jira ticket, fix in next sprint

P4 - Low (ระดับต่ำ)
──────────────────────
ลักษณะ: UI bugs, minor issues
ผลกระทบ: Cosmetic, ไม่กระทบ business
Response Time: Next business day
Action: Create Jira ticket
```

### Incident Response Flow

```
Alert Triggered
     │
     ▼
Acknowledge Alert (< 5 min)
     │
     ▼
Assess Severity (P1/P2/P3/P4)
     │
     ├── P1: ─► Wake CTO + All team + Start War Room
     │
     ├── P2: ─► Notify team lead + Start investigation
     │
     └── P3/P4: ─► Create ticket + Normal flow
     
     │
     ▼
Diagnose Problem
     │
     ├── Check recent deployments
     ├── Check error logs
     ├── Check metrics/dashboards
     └── Check external dependencies
     
     │
     ▼
Implement Fix or Rollback
     │
     ▼
Verify Resolution
     │
     ▼
Post-Incident Review (P1/P2)
     │
     ▼
Update Runbook if needed
```

---

## Common Incidents & Resolutions

### 1. Webhook ไม่ได้รับ Events

**Symptoms:**
- LINE events ไม่เข้าระบบ
- User ส่งข้อความแต่ Bot ไม่ตอบ

**Diagnostic Steps:**
```bash
# Check webhook endpoint health
curl -X GET https://api.example.com/health

# Check recent logs
kubectl logs -n line-bot -l app=webhook-handler --tail=100

# Check error rate
kubectl top pods -n line-bot

# Verify LINE webhook URL
# ไปที่ LINE Developer Console และ Verify Webhook
```

**Resolution:**
```bash
# ถ้า Pod crash
kubectl rollout restart deployment/line-bot -n line-bot

# ถ้า Certificate expired
kubectl describe certificate line-bot-tls -n line-bot
certbot renew --nginx

# ถ้า Memory/CPU limit
kubectl describe hpa line-bot-hpa -n line-bot
kubectl patch hpa line-bot-hpa -n line-bot --patch '{"spec":{"maxReplicas":20}}'
```

### 2. ตอบช้า / Timeout

**Symptoms:**
- LINE แสดง "ข้อความส่งไม่ได้"
- Response time > 3 วินาที

**Diagnostic Steps:**
```bash
# Check database connections
psql -U admin -d linebot -c "SELECT count(*) FROM pg_stat_activity"

# Check Redis latency
redis-cli -h redis.example.com ping
redis-cli -h redis.example.com latency history event

# Check slow queries
psql -U admin -d linebot -c "
  SELECT query, mean_exec_time, calls 
  FROM pg_stat_statements 
  ORDER BY mean_exec_time DESC 
  LIMIT 10"
```

### 3. Broadcast ล้มเหลว

**Symptoms:**
- Broadcast status stuck ที่ "processing"
- ส่งได้แค่บางส่วน

**Diagnostic Steps:**
```bash
# Check broadcast worker
kubectl logs -n line-bot -l app=broadcast-worker --tail=100

# Check LINE API quota
curl -H "Authorization: Bearer $CHANNEL_ACCESS_TOKEN" \
  https://api.line.me/v2/bot/info

# Check queue depth
redis-cli -h redis.example.com llen broadcast:queue
```

---

## Runbook: Manual Operations

### Restart Services
```bash
# Restart all LINE Bot pods
kubectl rollout restart deployment/line-bot-webhook -n line-bot
kubectl rollout restart deployment/line-bot-worker -n line-bot

# ตรวจสอบ
kubectl get pods -n line-bot -w
```

### Database Backup & Restore
```bash
# Manual backup
pg_dump -U admin -h db.example.com linebot | gzip > backup_$(date +%Y%m%d_%H%M%S).sql.gz

# Restore (ระวัง! จะ overwrite ข้อมูล)
gunzip -c backup_20240115_120000.sql.gz | psql -U admin -h db.example.com linebot
```

### Clear Cache
```bash
# Clear session cache (ระวัง! users จะต้อง re-authenticate)
redis-cli -h redis.example.com --scan --pattern "linebot:session:*" | xargs redis-cli -h redis.example.com del

# Clear specific user cache
redis-cli -h redis.example.com del "linebot:user:Uf1234567890abcdef"
```

### Feature Flags
```bash
# Disable feature temporarily
redis-cli -h redis.example.com set "feature:payment:enabled" "false"

# Re-enable
redis-cli -h redis.example.com del "feature:payment:enabled"
```
```

---

## 7. KPIs และ OKRs สำหรับ LINE OA Team

### 7.1 OKR Framework

```markdown
# Q1 2024 OKRs - LINE OA Team

## Objective 1: เพิ่ม User Engagement
**Key Results:**
- KR1.1: เพิ่ม Monthly Active Users จาก 50,000 เป็น 75,000 (+50%)
- KR1.2: เพิ่ม Message Read Rate จาก 35% เป็น 50%
- KR1.3: ลด Unblock Rate ให้ต่ำกว่า 2% ต่อเดือน

**Initiatives:**
- Personalization engine (สร้าง content ที่ตรงกับ interest)
- A/B test message formats
- Improve conversation UX

---

## Objective 2: เพิ่ม Revenue จาก LINE OA
**Key Results:**
- KR2.1: เพิ่ม Conversion Rate จาก 2.3% เป็น 4%
- KR2.2: เพิ่ม Average Order Value จาก 850 บาท เป็น 1,200 บาท
- KR2.3: ลด Cart Abandonment Rate จาก 65% เป็น 45%

**Initiatives:**
- Implement Cart Recovery messages
- Add upsell/cross-sell in checkout flow
- PromptPay payment integration

---

## Objective 3: ปรับปรุง Technical Reliability
**Key Results:**
- KR3.1: เพิ่ม Uptime จาก 99.5% เป็น 99.9%
- KR3.2: ลด Mean Time to Recovery (MTTR) จาก 45 นาที เป็น 15 นาที
- KR3.3: เพิ่ม Test Coverage จาก 65% เป็น 85%

**Initiatives:**
- Implement Circuit Breaker
- Automate rollback procedures
- Increase test coverage

---

## Weekly OKR Check-in Template
| KR | Target | Current | Status | Action Needed |
|----|--------|---------|--------|---------------|
| MAU growth | 75,000 | 62,000 | 🟡 On Track | - |
| Message Read Rate | 50% | 38% | 🔴 Behind | A/B test this week |
| Conversion Rate | 4% | 2.8% | 🟡 On Track | - |
| Uptime | 99.9% | 99.7% | 🟡 On Track | Review incidents |
```

### 7.2 Engineering KPIs Dashboard

```typescript
// Engineering KPIs ที่ต้องติดตาม

interface EngineeringKPIs {
  // Velocity & Productivity
  velocityMetrics: {
    storyPointsPerSprint: number;    // Target: > 40 SP
    deploymentFrequency: string;      // Target: หลายครั้ง/วัน
    leadTime: number;                 // Target: < 2 days (code → production)
    bugEscapeRate: number;            // Target: < 5% bugs reach production
  };
  
  // Quality
  qualityMetrics: {
    testCoverage: number;             // Target: > 80%
    codeReviewTime: number;           // Target: < 24 hours
    technicalDebtRatio: number;       // Target: < 10%
    criticalBugsOpen: number;         // Target: 0
  };
  
  // Operations
  operationalMetrics: {
    uptime: number;                   // Target: > 99.9%
    mttr: number;                     // Target: < 15 minutes
    mtbf: number;                     // Target: > 30 days
    errorRate: number;                // Target: < 0.1%
    p95ResponseTime: number;          // Target: < 200ms
  };
  
  // Team Health
  teamMetrics: {
    teamSatisfaction: number;         // Target: > 4/5 (survey)
    onboardingTime: number;           // Target: < 2 weeks to first PR
    documentationCoverage: number;    // Target: > 90%
    knowledgeSilos: number;           // Target: 0 (bus factor ≥ 2)
  };
}

// Weekly Engineering Review Template
const weeklyReview = {
  date: '2024-01-15',
  period: 'Week 3, Q1 2024',
  
  highlights: [
    'Deploy payment feature สำเร็จ ใน 0 incidents',
    'Test coverage เพิ่มขึ้นจาก 72% เป็น 78%'
  ],
  
  concerns: [
    'P95 response time เพิ่มขึ้น (300ms → 450ms) - investigate DB indexes',
    'PR review time เฉลี่ย 36 ชั่วโมง เกิน target'
  ],
  
  metrics: {
    deployments: 12,
    incidentCount: 0,
    p95ResponseTime: 450,
    testCoverage: 78,
    openPRs: 8,
    avgPRReviewTime: 36
  },
  
  nextWeekPriorities: [
    'Optimize database queries สำหรับ user lookup',
    'Reduce PR review backlog - dedicated review time 10-11am'
  ]
};
```

---

## 8. Onboarding Checklist สำหรับสมาชิกใหม่

```markdown
# New Team Member Onboarding Checklist

**ชื่อ:** ___________________
**ตำแหน่ง:** ___________________
**วันเริ่มงาน:** ___________________
**Buddy:** ___________________

## Week 1: Orientation & Setup

### Day 1 (วันแรก)
- [ ] รับ Laptop และ access cards
- [ ] Setup LINE account (work)
- [ ] เพิ่มเข้า LINE Groups ทั้งหมด
- [ ] Setup corporate email
- [ ] อ่าน Team Handbook
- [ ] Meet the team (1-on-1 สั้นๆ กับแต่ละคน)
- [ ] รับทราบ Emergency contacts

### Day 2-3 (Development Setup)
- [ ] Clone repositories
  ```bash
  git clone https://github.com/company/line-bot-backend.git
  git clone https://github.com/company/line-bot-frontend.git
  git clone https://github.com/company/line-bot-shared.git
  ```
- [ ] Install dependencies และ run development environment
- [ ] Access ได้แก่:
  - [ ] GitHub organization
  - [ ] AWS/GCP console (read-only)
  - [ ] Jira / Linear
  - [ ] Confluence / Notion
  - [ ] Grafana / DataDog
  - [ ] Sentry
  - [ ] LINE Developer Console (staging)
- [ ] ทำ Hello World PR แรก

### Day 4-5 (Codebase Exploration)
- [ ] อ่าน Architecture Overview document
- [ ] ทำความเข้าใจ data flow ใน LINE Bot
- [ ] อ่าน ADRs (Architecture Decision Records) สำคัญ
- [ ] Shadow on-call engineer 1 วัน
- [ ] อ่าน Post-mortem ของ incidents ใหญ่ๆ ที่ผ่านมา

## Week 2: Deep Dive

### Technical Deep Dives (เลือกตามตำแหน่ง)

**Backend Developer:**
- [ ] LINE Messaging API Fundamentals (2 ชั่วโมง)
- [ ] Webhook Handler Architecture (1 ชั่วโมง)
- [ ] Database Schema & Migrations
- [ ] Redis Usage & Patterns
- [ ] Testing Strategy & Writing Tests

**Frontend/LIFF Developer:**
- [ ] LIFF SDK Overview
- [ ] Flex Message Builder deep dive
- [ ] Component Library workshop
- [ ] Performance optimization techniques

**DevOps Engineer:**
- [ ] Infrastructure overview (AWS/GCP)
- [ ] Kubernetes cluster walkthrough
- [ ] CI/CD pipeline walkthrough
- [ ] Monitoring & Alerting setup

### Process Deep Dives
- [ ] เข้า Sprint Planning (observe)
- [ ] เข้า Daily Standup (observe)
- [ ] อ่าน Git workflow & conventions
- [ ] อ่าน Code review guidelines
- [ ] เข้า 1-on-1 กับ manager

## Week 3-4: First Contribution

- [ ] รับ Starter task (P4 ticket)
- [ ] นำเสนอ Solution approach ก่อน code
- [ ] Submit PR แรก
- [ ] ผ่าน Code Review กับ mentor
- [ ] Deploy to staging
- [ ] Present work ใน Sprint Review

## Month 2: Growing Independence

- [ ] รับ P3/P2 tickets อิสระ
- [ ] ทำ PR review ให้ผู้อื่น
- [ ] เป็น on-call secondary
- [ ] Present ใน Team Knowledge Share
- [ ] มี 30-60-90 day review กับ manager

## Month 3: Full Team Member

- [ ] รับ Feature ownership
- [ ] Lead sprint ใด sprint หนึ่ง
- [ ] เป็น on-call primary
- [ ] Onboard สมาชิกใหม่คนต่อไป
- [ ] Complete 90-day performance review

---

## สิ่งที่ต้องเรียนรู้ (Self-Study Checklist)

**LINE Platform:**
- [ ] [LINE Developers Documentation](https://developers.line.biz/)
- [ ] LINE Messaging API Cookbook
- [ ] LIFF v2 Documentation

**Technical:**
- [ ] TypeScript Handbook
- [ ] Node.js Best Practices
- [ ] Docker & Kubernetes basics
- [ ] PostgreSQL Performance

**Soft Skills:**
- [ ] Agile/Scrum basics
- [ ] Technical Writing
- [ ] Incident Management
```

---

## 9. Training Curriculum (12-Week Plan)

```markdown
# 12-Week LINE Bot Developer Training Curriculum

## Module 1: Foundation (Week 1-2)

### Week 1: LINE Platform Basics
**วัตถุประสงค์:** เข้าใจ LINE Ecosystem และ API พื้นฐาน

| วัน | หัวข้อ | ระยะเวลา | รูปแบบ |
|----|--------|---------|--------|
| จันทร์ | LINE Platform Overview & Ecosystem | 3 ชม. | Lecture + Demo |
| อังคาร | LINE Messaging API - Text & Sticker | 4 ชม. | Hands-on Lab |
| พุธ | Flex Message Builder | 4 ชม. | Workshop |
| พฤหัส | Rich Menu & Image Map | 3 ชม. | Project |
| ศุกร์ | Mini Project: สร้าง Info Bot | 4 ชม. | Project |

**Lab: สร้าง Customer Service Bot พื้นฐาน**
```typescript
// Lab Exercise: Basic Echo Bot
const express = require('express');
const line = require('@line/bot-sdk');

const config = {
  channelAccessToken: process.env.LINE_CHANNEL_ACCESS_TOKEN,
  channelSecret: process.env.LINE_CHANNEL_SECRET
};

const client = new line.Client(config);
const app = express();

app.post('/webhook', line.middleware(config), (req, res) => {
  Promise.all(req.body.events.map(handleEvent))
    .then(result => res.json(result));
});

async function handleEvent(event) {
  if (event.type !== 'message' || event.message.type !== 'text') {
    return null;
  }
  
  // TODO: นักเรียนต้องเพิ่ม:
  // 1. ทักทายถ้าพิมพ์ "สวัสดี"
  // 2. ตอบ FAQ 5 ข้อ
  // 3. ส่ง Flex Message สำหรับ "เมนู"
  
  return client.replyMessage(event.replyToken, {
    type: 'text',
    text: `Echo: ${event.message.text}`
  });
}
```

### Week 2: State Management & Conversation Flow
**วัตถุประสงค์:** สร้าง Multi-step conversation

| วัน | หัวข้อ | รูปแบบ |
|----|--------|--------|
| จันทร์ | State Machine สำหรับ Conversation | Lecture |
| อังคาร | Redis สำหรับ Session Storage | Hands-on |
| พุธ | LIFF - Getting Started | Workshop |
| พฤหัส | LIFF - Advanced Features | Workshop |
| ศุกร์ | Mini Project: Order Bot with State | Project |

---

## Module 2: Backend Development (Week 3-5)

### Week 3: Database Design & API
**Topics:**
- Database schema สำหรับ LINE Bot
- RESTful API design
- Authentication & Authorization
- Error handling patterns

### Week 4: Advanced Features
**Topics:**
- Broadcast & Audience Segmentation
- Chatbot NLP Integration
- Payment Integration (PromptPay)
- File upload & Media handling

### Week 5: Performance & Testing
**Topics:**
- Load testing with Artillery
- Unit & Integration testing
- Performance optimization
- Caching strategies

---

## Module 3: Operations (Week 6-8)

### Week 6: DevOps Basics
**Topics:**
- Docker containerization
- CI/CD with GitHub Actions
- Kubernetes fundamentals
- Environment management

### Week 7: Monitoring & Observability
**Topics:**
- Grafana dashboards
- Log management (ELK Stack)
- Alerting rules
- Distributed tracing

### Week 8: Security
**Topics:**
- OWASP Top 10 สำหรับ LINE Bot
- Secrets management
- Penetration testing basics
- Compliance (PDPA)

---

## Module 4: Advanced Topics (Week 9-10)

### Week 9: Scalability
**Topics:**
- Horizontal scaling
- Message queue patterns
- Microservices architecture
- Database optimization

### Week 10: AI/ML Integration
**Topics:**
- Natural Language Understanding
- Recommendation systems
- Predictive analytics
- Computer vision สำหรับ LINE

---

## Module 5: Capstone Project (Week 11-12)

### Week 11-12: Build Complete System
**Project:** สร้าง E-commerce LINE Bot ที่มีครบทุก feature

**Deliverables:**
1. ✅ Architecture document
2. ✅ Working LINE Bot (staging)
3. ✅ Test coverage > 80%
4. ✅ API documentation
5. ✅ Monitoring dashboard
6. ✅ Deployment runbook
7. ✅ Presentation (15 นาที)

**Grading Criteria:**
| หัวข้อ | คะแนน |
|-------|-------|
| Functionality (ทำงานได้ตาม requirements) | 30% |
| Code Quality (clean, tested, documented) | 25% |
| Architecture (scalable, maintainable) | 20% |
| Operations (monitoring, deployment) | 15% |
| Presentation | 10% |
```

---

## 10. Team Handbook Template

```markdown
# LINE OA Team Handbook
**Version:** 2.0
**Last Updated:** January 2024
**Owner:** Team Lead

---

## 1. ยินดีต้อนรับ

ยินดีต้อนรับสู่ LINE OA Team! 

เราเป็นทีมที่สร้างประสบการณ์ที่ดีที่สุดให้กับลูกค้าของ [ชื่อบริษัท] ผ่าน LINE Official Account เรามีความมุ่งมั่นที่จะ...

### วิสัยทัศน์ (Vision)
"เป็น LINE Bot ที่ดีที่สุดในอุตสาหกรรม [X] ของไทย"

### พันธกิจ (Mission)
"สร้าง conversational experience ที่ seamless และ personalized ให้กับลูกค้าทุกคน"

### ค่านิยม (Values)
1. **Customer First** - ตัดสินใจโดยคำนึงถึงลูกค้าเป็นอันดับแรก
2. **Data Driven** - ใช้ข้อมูลในการตัดสินใจ ไม่ใช่ความรู้สึก
3. **Ship Fast, Learn Faster** - Deploy บ่อย, เรียนรู้จาก production
4. **No Blame Culture** - เมื่อเกิดปัญหา แก้ไขก่อน หาสาเหตุทีหลัง
5. **Radical Transparency** - share ข้อมูลอย่างเปิดเผย

---

## 2. วิธีการทำงาน

### Working Hours
- Core Hours: 10:00 - 16:00 (ต้องออนไลน์/พร้อมตอบสนอง)
- Flexible Hours: 08:00 - 19:00 (work ช่วงไหนก็ได้)
- Remote Work: ได้ทุกวัน (เข้า office อย่างน้อย 2 วัน/สัปดาห์)

### Communication Channels
| ช่องทาง | ใช้สำหรับ | Expected Response Time |
|--------|---------|----------------------|
| LINE Group (Work) | Quick questions, updates | 30 นาที (core hours) |
| Email | Formal communication | 24 ชั่วโมง |
| Jira/Linear | Task tracking | ดูตาม SLA ของ ticket |
| GitHub PR | Code review | 24 ชั่วโมง |
| Video Call | Complex discussions | ตามนัด |

### Meeting Norms
- เริ่มตรงเวลาเสมอ
- มี Agenda ก่อนทุก meeting
- ทุกคน on-camera ใน video call
- สรุป action items หลัง meeting ทุกครั้ง

---

## 3. Engineering Guidelines

### Tech Stack
**Backend:** Node.js 20 LTS, TypeScript 5.x, Express.js / Fastify
**Database:** PostgreSQL 15, Redis 7
**Frontend/LIFF:** React 18, Next.js 14, TypeScript
**Infrastructure:** AWS (EKS, RDS, ElastiCache), Terraform
**CI/CD:** GitHub Actions
**Monitoring:** Grafana, Prometheus, Loki
**Error Tracking:** Sentry

### Definition of Ready (DoR)
User story พร้อม develop เมื่อ:
- [ ] มี acceptance criteria ที่ชัดเจน
- [ ] Story point ถูก estimate แล้ว
- [ ] Dependencies ถูก identify
- [ ] Design/wireframe พร้อม (ถ้าต้องการ)

### Definition of Done (DoD)
Feature เสร็จเมื่อ:
- [ ] Code pass review
- [ ] Unit tests > 80% coverage
- [ ] Integration tests pass
- [ ] QA sign-off
- [ ] Documentation updated
- [ ] Deployed to staging
- [ ] No critical bugs

---

## 4. Career Ladder

### Junior Developer (L1)
- 0-2 ปีประสบการณ์
- ทำ well-defined tasks ได้
- ต้องการ guidance บ้าง
- Focus: เรียนรู้ codebase และ domain

### Developer (L2)
- 2-4 ปีประสบการณ์
- ทำงานอิสระได้
- Mentor junior developers
- Focus: Deliver features with quality

### Senior Developer (L3)
- 4-7 ปีประสบการณ์
- Lead feature development
- Drive technical decisions
- Focus: Team productivity & quality

### Staff Developer (L4)
- 7+ ปีประสบการณ์
- Cross-team impact
- Define technical direction
- Focus: Organization-wide impact

### Promotion Criteria
| Level | Technical | Leadership | Business Impact |
|-------|-----------|------------|----------------|
| L1→L2 | ทำงานอิสระได้ | Collaborate well | Deliver features on time |
| L2→L3 | Expert in 2+ areas | Mentor 1-2 juniors | Drive OKRs |
| L3→L4 | Technical vision | Lead team | Org-wide impact |

---

## 5. Recognition & Rewards

### Peer Recognition
ทุก Sprint retrospective มี "Shoutout" ให้กันและกัน

### Engineering Awards (รายไตรมาส)
- **Most Impactful Feature** - Feature ที่ส่งผลต่อ KPI มากที่สุด
- **Quality Champion** - คนที่ prevent bugs มากที่สุด
- **Team Player** - คนที่ช่วย team มากที่สุด
- **Innovation Award** - ไอเดียใหม่ที่นำไปใช้ได้จริง

---

## 6. Health & Wellbeing

### Work-Life Balance
- ไม่ส่ง message คาดหวังการตอบ after 18:00 (ยกเว้น P1 incidents)
- ลาพักร้อนอย่างน้อย 12 วัน/ปี (บังคับ!)
- Mental health days: ลาได้โดยไม่ต้องอธิบาย

### Learning Budget
- 15,000 บาท/คน/ปี สำหรับ courses, conferences, books
- 2 วัน/เดือน สำหรับ self-learning

### On-Call Compensation
- On-call ช่วง weekend: +2,000 บาท/วัน
- Incident ที่กระทบเวลาส่วนตัว: Comp-off 2:1
```

---

## 11. Stakeholder Communication Templates

### 11.1 Weekly Status Report

```markdown
# LINE OA Weekly Report
**Week:** [Week Number] - [Date Range]
**Prepared by:** [Name]
**Recipients:** [Manager, Stakeholders]

---

## 🟢 Executive Summary
[2-3 ประโยคสรุปภาพรวมสัปดาห์นี้]

## 📊 Key Metrics

| Metric | This Week | Last Week | Target | Status |
|--------|-----------|-----------|--------|--------|
| Active Users | 45,230 | 43,100 | 50,000 | 🟡 |
| Messages Sent | 1.2M | 1.1M | 1.5M | 🟡 |
| Broadcast Open Rate | 42% | 39% | 50% | 🟡 |
| Bot Response Time | 145ms | 162ms | <200ms | 🟢 |
| Uptime | 99.95% | 99.87% | 99.9% | 🟢 |
| Conversion Rate | 3.2% | 2.8% | 4% | 🟡 |

## ✅ Accomplishments This Week
1. Deploy Payment Feature (LINE Pay integration) - สำเร็จ
2. ปรับปรุง Rich Menu ใหม่ - Engagement เพิ่ม 23%
3. Fix critical bug: Thai character encoding

## 🔨 In Progress
1. User Segmentation feature (70% complete, ETA: Friday)
2. Performance optimization (DB indexes)
3. LIFF 2.0 upgrade

## ⚠️ Blockers & Risks
1. LINE Pay API sandbox ยัง unstable - รอ LINE Thailand resolve
2. Design resource ขาดเนื่องจากลา - อาจกระทบ Sprint goal

## 📅 Next Week Plan
1. Complete User Segmentation feature
2. Start Broadcast Scheduler feature
3. Security audit ตาม schedule

## 💡 Notes for Stakeholders
- Broadcast วัน 15 มกราคม ready - รอ content อนุมัติ
- Budget request สำหรับ LINE Notification ส่งให้ทาง Finance แล้ว
```

### 11.2 Incident Communication Template

```markdown
# Incident Communication Template

## Initial Alert (ส่งใน 15 นาที)
Subject: [P1/P2] LINE Bot Incident - [Brief Description]

เรียนทีม,

เกิด incident กับ LINE Bot ณ เวลา [TIME]

**ผลกระทบ:** [อธิบายผลกระทบต่อ users]
**สถานะ:** กำลังสืบสวนหาสาเหตุ
**ผู้รับผิดชอบ:** [ชื่อ]

จะ update ทุก 30 นาที

---

## Progress Update (ทุก 30 นาที)
เวลา [TIME] - Update #[N]

**สถานะ:** [Investigating / Identified cause / Implementing fix]
**สาเหตุ:** [อธิบายถ้าทราบแล้ว]
**Actions taken:** [รายการสิ่งที่ทำไปแล้ว]
**ETA to resolve:** [เวลาคาด]

---

## Resolution Notice
Subject: [RESOLVED] LINE Bot Incident - [Brief Description]

Incident ได้รับการแก้ไขแล้วเมื่อเวลา [TIME]

**Timeline:**
- [TIME]: Incident detected
- [TIME]: Investigation started  
- [TIME]: Root cause identified
- [TIME]: Fix deployed
- [TIME]: Verified resolved

**Root Cause:** [อธิบาย]
**Impact:** [ผู้ใช้กี่คน, นานแค่ไหน]
**Fix:** [อธิบาย solution]
**Prevention:** [จะป้องกันอย่างไรในอนาคต]

Post-mortem meeting: [วันเวลา]
```

---

## สรุป

การสร้างทีมและกระบวนการทำงานที่ดีเป็นรากฐานของ LINE Bot ที่ประสบความสำเร็จ สิ่งสำคัญที่ต้องจำไว้:

1. **คน** คือทรัพยากรที่สำคัญที่สุด - ลงทุนในการพัฒนาทีม
2. **Process** ต้องเหมาะกับขนาดทีมและบริบทธุรกิจ
3. **Communication** ที่ชัดเจนช่วยลด misalignment
4. **Measurement** ทำให้เห็นว่าต้องปรับปรุงที่ไหน
5. **Culture** ที่ดีช่วยรักษาคนเก่งให้อยู่กับทีม

ทีมที่ดีไม่ได้สร้างข้ามคืน แต่ต้องลงทุนอย่างต่อเนื่องในระยะยาว

---

*Part 99 จาก 100 - ใกล้ถึงบทสุดท้ายแล้ว!*
