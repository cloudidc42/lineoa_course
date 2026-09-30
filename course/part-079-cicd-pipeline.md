# Part 79: CI/CD Pipeline สำหรับ LINE Bot

## บทนำ

CI/CD (Continuous Integration / Continuous Deployment) ช่วยให้คุณ deploy LINE Bot ได้อย่างรวดเร็ว ปลอดภัย และมั่นใจว่าไม่มี regression เมื่อ code เปลี่ยนแปลง

บทนี้จะครอบคลุม:
- GitHub Actions workflow ครบวงจร
- การทดสอบอัตโนมัติ
- Staging และ Production environments
- Zero-downtime deployment
- Rollback strategy

```
CI/CD Pipeline Overview:
┌──────────────────────────────────────────────────────────────────┐
│                                                                    │
│  Developer → Push Code → GitHub                                    │
│                              │                                     │
│                    ┌─────────▼──────────┐                         │
│                    │   GitHub Actions    │                         │
│                    └──┬──────────────┬──┘                         │
│                       │              │                             │
│              ┌────────▼───┐  ┌───────▼────────┐                  │
│              │  CI Tests   │  │  Build & Push  │                  │
│              │  - Unit     │  │  Docker Image  │                  │
│              │  - Integration│ └───────┬────────┘                 │
│              │  - Security │          │                            │
│              └────────┬───┘  ┌────────▼───────┐                  │
│                       │      │ Deploy Staging  │                  │
│                       │      │ (auto on PR)    │                  │
│                       │      └────────┬────────┘                  │
│                       │               │                            │
│                       │      ┌────────▼────────┐                  │
│                       │      │ Manual Approval  │                  │
│                       │      │ for Production   │                  │
│                       │      └────────┬─────────┘                 │
│                       │               │                            │
│                       └───────────────┼────────────────┐          │
│                                       │                │           │
│                              ┌────────▼─────────┐     │           │
│                              │ Deploy Production │     │           │
│                              │ (zero-downtime)  │     │           │
│                              └──────────────────┘     │           │
│                                                        │           │
│                              ┌─────────────────────────┘          │
│                              │ Rollback if needed                  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. Repository Structure

### 1.1 โครงสร้าง Project

```
linebot-project/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml              # CI tests
│   │   ├── deploy-staging.yml  # Deploy to staging
│   │   ├── deploy-production.yml # Deploy to production
│   │   └── security-scan.yml   # Security scanning
│   ├── CODEOWNERS
│   └── pull_request_template.md
├── src/
│   ├── app.js
│   ├── handlers/
│   ├── services/
│   ├── models/
│   └── middleware/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── docker/
│   ├── Dockerfile
│   ├── Dockerfile.dev
│   └── nginx.conf
├── k8s/
│   ├── staging/
│   └── production/
├── scripts/
│   ├── deploy.sh
│   ├── rollback.sh
│   └── health-check.sh
├── docker-compose.yml
├── docker-compose.test.yml
├── .env.example
└── package.json
```

---

## 2. GitHub Actions - CI Workflow

### 2.1 Main CI Workflow

```yaml
# .github/workflows/ci.yml
name: CI Tests

on:
  push:
    branches:
      - main
      - develop
      - 'feature/**'
      - 'hotfix/**'
  pull_request:
    branches:
      - main
      - develop

env:
  NODE_VERSION: '20'
  
jobs:
  # ============================
  # Job 1: Lint & Type Check
  # ============================
  lint:
    name: Lint & Type Check
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: Check formatting (Prettier)
        run: npm run format:check
      
      - name: TypeScript type check
        run: npm run type-check
        if: ${{ hashFiles('tsconfig.json') != '' }}

  # ============================
  # Job 2: Unit Tests
  # ============================
  unit-tests:
    name: Unit Tests
    runs-on: ubuntu-latest
    needs: lint
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
        env:
          LINE_CHANNEL_SECRET: test-channel-secret
          LINE_CHANNEL_ACCESS_TOKEN: test-access-token
          JWT_SECRET: test-jwt-secret-for-testing-only
          CSRF_SECRET: test-csrf-secret-for-testing
      
      - name: Upload coverage report
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage/lcov.info
          flags: unit
          fail_ci_if_error: false
      
      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/coverage-summary.json | jq '.total.lines.pct')
          echo "Coverage: $COVERAGE%"
          if (( $(echo "$COVERAGE < 70" | bc -l) )); then
            echo "Coverage $COVERAGE% is below threshold 70%"
            exit 1
          fi

  # ============================
  # Job 3: Integration Tests
  # ============================
  integration-tests:
    name: Integration Tests
    runs-on: ubuntu-latest
    needs: lint
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: testpassword
          MYSQL_DATABASE: linebot_test
          MYSQL_USER: linebot
          MYSQL_PASSWORD: testpassword
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd="redis-cli ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=5
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run database migrations
        run: npm run db:migrate:test
        env:
          DB_HOST: localhost
          DB_PORT: 3306
          DB_NAME: linebot_test
          DB_USER: linebot
          DB_PASSWORD: testpassword
      
      - name: Run integration tests
        run: npm run test:integration
        env:
          NODE_ENV: test
          DB_HOST: localhost
          DB_PORT: 3306
          DB_NAME: linebot_test
          DB_USER: linebot
          DB_PASSWORD: testpassword
          REDIS_HOST: localhost
          REDIS_PORT: 6379
          LINE_CHANNEL_SECRET: test-channel-secret
          LINE_CHANNEL_ACCESS_TOKEN: test-access-token

  # ============================
  # Job 4: Security Scan
  # ============================
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: lint
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run npm audit
        run: npm audit --production --audit-level=high
      
      - name: Run Snyk scan
        uses: snyk/actions/node@master
        continue-on-error: true
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high --sarif-file-output=snyk.sarif
      
      - name: Upload Snyk results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: snyk.sarif

  # ============================
  # Job 5: Build Docker Image
  # ============================
  build:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests, security-scan]
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=sha,prefix=sha-
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}
      
      - name: Build and push
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VERSION=${{ github.sha }}
```

---

## 3. Staging Deployment Workflow

### 3.1 Auto Deploy to Staging

```yaml
# .github/workflows/deploy-staging.yml
name: Deploy to Staging

on:
  push:
    branches:
      - develop
  pull_request:
    branches:
      - main
    types: [opened, synchronize, reopened]

concurrency:
  group: staging-${{ github.ref }}
  cancel-in-progress: true

jobs:
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging-bot.your-company.com
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push staging image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:staging-${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.28.0'
      
      - name: Configure kubectl for staging
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.STAGING_KUBECONFIG }}
      
      - name: Deploy to staging
        run: |
          # อัปเดต image tag
          kubectl set image deployment/linebot \
            linebot=ghcr.io/${{ github.repository }}:staging-${{ github.sha }} \
            -n linebot-staging
          
          # รอให้ deployment เสร็จ
          kubectl rollout status deployment/linebot \
            -n linebot-staging \
            --timeout=5m
      
      - name: Update LINE Webhook URL (Staging)
        run: |
          # อัปเดต webhook URL ใน LINE Developers Console
          curl -X PUT https://api.line.me/v2/bot/channel/webhook/endpoint \
            -H "Authorization: Bearer ${{ secrets.LINE_STAGING_ACCESS_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"webhookEndpointUrl": "https://staging-bot.your-company.com/webhook"}'
      
      - name: Run smoke tests
        run: |
          # รอให้ service พร้อม
          sleep 30
          
          # Test health endpoint
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://staging-bot.your-company.com/health)
          if [ "$STATUS" != "200" ]; then
            echo "Health check failed: $STATUS"
            exit 1
          fi
          
          echo "Staging deployment successful!"
      
      - name: Comment PR with staging URL
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `
              🚀 **Staging Deployment Successful!**
              
              | Info | Details |
              |------|---------|
              | URL | https://staging-bot.your-company.com |
              | Image | \`staging-${{ github.sha }}\` |
              | Deployed | ${new Date().toISOString()} |
              
              ✅ Health check passed
              `
            })
```

---

## 4. Production Deployment

### 4.1 Production Deployment with Approval

```yaml
# .github/workflows/deploy-production.yml
name: Deploy to Production

on:
  push:
    branches:
      - main
  workflow_dispatch:
    inputs:
      version:
        description: 'Version to deploy (default: latest main)'
        required: false
        type: string
      skip_tests:
        description: 'Skip integration tests (for hotfixes only)'
        required: false
        type: boolean
        default: false

jobs:
  # Pre-deployment checks
  pre-deploy-checks:
    name: Pre-deployment Checks
    runs-on: ubuntu-latest
    
    outputs:
      image-tag: ${{ steps.get-tag.outputs.tag }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Get image tag
        id: get-tag
        run: |
          if [ -n "${{ inputs.version }}" ]; then
            echo "tag=${{ inputs.version }}" >> $GITHUB_OUTPUT
          else
            echo "tag=sha-${{ github.sha }}" >> $GITHUB_OUTPUT
          fi
      
      - name: Verify image exists
        run: |
          docker manifest inspect ghcr.io/${{ github.repository }}:${{ steps.get-tag.outputs.tag }} || \
          docker manifest inspect ghcr.io/${{ github.repository }}:${{ github.sha }}
      
      - name: Check staging deployment status
        run: |
          # ตรวจสอบว่า staging ผ่านการทดสอบแล้ว
          curl -f https://staging-bot.your-company.com/health || \
          echo "Warning: Staging health check failed"

  # Production deployment (requires approval)
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: pre-deploy-checks
    environment:
      name: production
      url: https://bot.your-company.com
    
    # ต้องมีการ approve จาก team lead
    # ตั้งค่าใน GitHub → Settings → Environments → production → Required reviewers
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup kubectl
        uses: azure/setup-kubectl@v3
      
      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.PRODUCTION_KUBECONFIG }}
      
      - name: Create deployment snapshot (for rollback)
        run: |
          # บันทึก current deployment
          kubectl get deployment linebot -n linebot-production -o yaml > /tmp/current-deployment.yaml
          
          # บันทึก image ปัจจุบัน
          CURRENT_IMAGE=$(kubectl get deployment linebot -n linebot-production \
            -o jsonpath='{.spec.template.spec.containers[0].image}')
          echo "Current image: $CURRENT_IMAGE"
          echo "ROLLBACK_IMAGE=$CURRENT_IMAGE" >> $GITHUB_ENV
      
      - name: Zero-downtime deployment
        run: |
          IMAGE_TAG="${{ needs.pre-deploy-checks.outputs.image-tag }}"
          
          echo "Deploying: ghcr.io/${{ github.repository }}:${IMAGE_TAG}"
          
          # Strategy: Rolling Update
          kubectl set image deployment/linebot \
            linebot=ghcr.io/${{ github.repository }}:${IMAGE_TAG} \
            -n linebot-production
          
          # รอ Rolling Update เสร็จ
          kubectl rollout status deployment/linebot \
            -n linebot-production \
            --timeout=10m
      
      - name: Run post-deployment health checks
        run: |
          # รอให้ service stable
          sleep 60
          
          # Health check หลาย endpoint
          echo "Testing health endpoint..."
          curl -f https://bot.your-company.com/health
          
          echo "Testing webhook endpoint..."
          STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
            -X POST https://bot.your-company.com/webhook \
            -H "Content-Type: application/json" \
            -H "X-Line-Signature: test" \
            -d '{}')
          
          # ควรได้ 401 (signature invalid) ไม่ใช่ 500
          if [ "$STATUS" = "500" ] || [ "$STATUS" = "502" ] || [ "$STATUS" = "503" ]; then
            echo "Post-deployment health check failed: $STATUS"
            exit 1
          fi
          
          echo "✅ All health checks passed!"
      
      - name: Update LINE Webhook URL
        run: |
          curl -X PUT https://api.line.me/v2/bot/channel/webhook/endpoint \
            -H "Authorization: Bearer ${{ secrets.LINE_PRODUCTION_ACCESS_TOKEN }}" \
            -H "Content-Type: application/json" \
            -d '{"webhookEndpointUrl": "https://bot.your-company.com/webhook"}'
      
      - name: Notify deployment success
        if: success()
        uses: 8398a7/action-slack@v3
        with:
          status: success
          text: |
            ✅ Production deployment successful!
            Version: ${{ needs.pre-deploy-checks.outputs.image-tag }}
            URL: https://bot.your-company.com
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
      
      - name: Rollback on failure
        if: failure()
        run: |
          echo "Deployment failed! Rolling back to: $ROLLBACK_IMAGE"
          
          kubectl set image deployment/linebot \
            linebot=${{ env.ROLLBACK_IMAGE }} \
            -n linebot-production
          
          kubectl rollout status deployment/linebot \
            -n linebot-production \
            --timeout=5m
          
          echo "Rollback completed"
      
      - name: Notify rollback
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          text: |
            🔄 Production deployment FAILED and was rolled back!
            Failed version: ${{ needs.pre-deploy-checks.outputs.image-tag }}
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 5. Environment Management

### 5.1 Environment Configuration

```javascript
// config/environment.js

const environments = {
  development: {
    name: 'development',
    webhookUrl: 'https://xxxx.ngrok.io/webhook',
    lineChannelId: process.env.LINE_CHANNEL_ID,
    database: {
      host: 'localhost',
      name: 'linebot_dev'
    },
    features: {
      debugMode: true,
      mockLineApi: true,
      rateLimiting: false
    }
  },
  
  staging: {
    name: 'staging',
    webhookUrl: 'https://staging-bot.your-company.com/webhook',
    lineChannelId: process.env.LINE_STAGING_CHANNEL_ID,
    database: {
      host: process.env.DB_HOST,
      name: 'linebot_staging'
    },
    features: {
      debugMode: true,
      mockLineApi: false,
      rateLimiting: true
    }
  },
  
  production: {
    name: 'production',
    webhookUrl: 'https://bot.your-company.com/webhook',
    lineChannelId: process.env.LINE_CHANNEL_ID,
    database: {
      host: process.env.DB_HOST,
      name: 'linebot_production'
    },
    features: {
      debugMode: false,
      mockLineApi: false,
      rateLimiting: true
    }
  }
};

const currentEnv = process.env.NODE_ENV || 'development';
module.exports = environments[currentEnv];
```

### 5.2 Feature Flags

```javascript
// services/featureFlags.js
const redis = require('../config/redis');

class FeatureFlags {
  /**
   * ตรวจสอบว่า feature เปิดหรือไม่
   */
  async isEnabled(feature, userId = null) {
    // ตรวจสอบ global flag ก่อน
    const globalFlag = await redis.get(`feature:${feature}`);
    
    if (globalFlag === 'disabled') return false;
    if (globalFlag === 'enabled') return true;
    
    // ตรวจสอบ percentage rollout
    const rolloutPercent = await redis.get(`feature:${feature}:rollout`);
    if (rolloutPercent && userId) {
      // Consistent rollout ตาม user ID
      const hash = parseInt(userId.substring(0, 8), 16);
      return (hash % 100) < parseInt(rolloutPercent);
    }
    
    // Default: disabled
    return false;
  }
  
  /**
   * เปิด/ปิด feature
   */
  async setFlag(feature, enabled, rolloutPercent = null) {
    if (rolloutPercent) {
      await redis.set(`feature:${feature}`, 'partial');
      await redis.set(`feature:${feature}:rollout`, rolloutPercent);
    } else {
      await redis.set(`feature:${feature}`, enabled ? 'enabled' : 'disabled');
    }
  }
}

// ตัวอย่างการใช้งาน
const featureFlags = new FeatureFlags();

async function handleNewFeature(event) {
  const userId = event.source.userId;
  
  // ตรวจสอบว่า new feature เปิดหรือไม่
  if (await featureFlags.isEnabled('new_menu_design', userId)) {
    return await showNewMenuDesign(event);
  } else {
    return await showOldMenuDesign(event);
  }
}
```

---

## 6. Secrets ใน CI/CD

### 6.1 GitHub Secrets Configuration

```yaml
# .github/workflows/manage-secrets.yml
# วิธีตั้งค่า secrets ใน GitHub

# Secrets ที่ต้องตั้งใน GitHub Repository Settings:
# Settings → Secrets and variables → Actions

# ====================================
# Secrets สำหรับ Development/Testing
# ====================================
# LINE_CHANNEL_SECRET_TEST
# LINE_CHANNEL_ACCESS_TOKEN_TEST
# DB_PASSWORD_TEST

# ====================================
# Secrets สำหรับ Staging
# ====================================
# LINE_STAGING_CHANNEL_SECRET
# LINE_STAGING_ACCESS_TOKEN
# STAGING_DB_PASSWORD
# STAGING_KUBECONFIG

# ====================================
# Secrets สำหรับ Production
# ====================================
# LINE_CHANNEL_SECRET
# LINE_CHANNEL_ACCESS_TOKEN
# PRODUCTION_DB_PASSWORD
# PRODUCTION_KUBECONFIG
# SNYK_TOKEN
# SLACK_WEBHOOK_URL
```

### 6.2 Using External Secrets Manager

```yaml
# .github/workflows/deploy-with-vault.yml
name: Deploy with Vault Secrets

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Import Secrets from Vault
        uses: hashicorp/vault-action@v2
        with:
          url: ${{ secrets.VAULT_ADDR }}
          method: jwt
          role: github-actions
          secrets: |
            secret/data/linebot/production line_channel_secret | LINE_CHANNEL_SECRET ;
            secret/data/linebot/production line_access_token | LINE_CHANNEL_ACCESS_TOKEN ;
            secret/data/linebot/production db_password | DB_PASSWORD
      
      - name: Use secrets
        run: |
          echo "Secrets loaded successfully"
          # LINE_CHANNEL_SECRET, LINE_CHANNEL_ACCESS_TOKEN, DB_PASSWORD
          # ถูก set เป็น environment variables โดยอัตโนมัติ
```

---

## 7. Zero-Downtime Deployment

### 7.1 Rolling Update Strategy

```yaml
# k8s/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linebot
  namespace: linebot-production
spec:
  replicas: 3
  
  # Rolling Update Strategy
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1          # สร้าง pod ใหม่ได้สูงสุด 1 ตัวเกินจำนวนที่กำหนด
      maxUnavailable: 0    # ไม่อนุญาตให้ pod ใดล่มระหว่าง update
  
  selector:
    matchLabels:
      app: linebot
  
  template:
    metadata:
      labels:
        app: linebot
    spec:
      containers:
        - name: linebot
          image: ghcr.io/your-org/linebot:latest
          
          # Health checks
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          
          # Graceful shutdown
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]
          
          # Resources
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
      
      # Grace period สำหรับ graceful shutdown
      terminationGracePeriodSeconds: 60
```

### 7.2 Graceful Shutdown Implementation

```javascript
// src/server.js
const http = require('http');
const app = require('./app');

const server = http.createServer(app);

let isShuttingDown = false;

// Middleware ตรวจสอบ shutdown state
app.use((req, res, next) => {
  if (isShuttingDown) {
    res.setHeader('Connection', 'close');
    return res.status(503).json({ 
      error: 'Server is shutting down',
      retryAfter: 30
    });
  }
  next();
});

// Graceful shutdown handler
async function gracefulShutdown(signal) {
  console.log(`Received ${signal}. Starting graceful shutdown...`);
  
  isShuttingDown = true;
  
  // หยุดรับ request ใหม่
  server.close(async () => {
    console.log('HTTP server closed');
    
    try {
      // รอให้ requests ที่กำลังทำอยู่เสร็จ
      await waitForActiveRequests();
      
      // ปิด database connections
      await closeDatabaseConnections();
      
      // ปิด Redis connections
      await closeRedisConnection();
      
      // Flush logs
      await flushLogs();
      
      console.log('Graceful shutdown complete');
      process.exit(0);
    } catch (err) {
      console.error('Error during shutdown:', err);
      process.exit(1);
    }
  });
  
  // Force exit หลัง 30 วินาที
  setTimeout(() => {
    console.error('Forced shutdown after timeout');
    process.exit(1);
  }, 30000);
}

// Track active requests
let activeRequests = 0;

app.use((req, res, next) => {
  activeRequests++;
  res.on('finish', () => activeRequests--);
  next();
});

function waitForActiveRequests() {
  return new Promise((resolve) => {
    const check = () => {
      if (activeRequests === 0) {
        resolve();
      } else {
        console.log(`Waiting for ${activeRequests} active requests...`);
        setTimeout(check, 1000);
      }
    };
    check();
  });
}

// ลงทะเบียน signal handlers
process.on('SIGTERM', () => gracefulShutdown('SIGTERM'));
process.on('SIGINT', () => gracefulShutdown('SIGINT'));

server.listen(process.env.PORT || 3000, () => {
  console.log(`LINE Bot server listening on port ${process.env.PORT || 3000}`);
});
```

---

## 8. Rollback Strategy

### 8.1 Automated Rollback Script

```bash
#!/bin/bash
# scripts/rollback.sh

set -euo pipefail

NAMESPACE="${NAMESPACE:-linebot-production}"
DEPLOYMENT="linebot"
TIMEOUT="5m"

echo "==================================="
echo "LINE Bot Rollback Script"
echo "==================================="

# แสดง deployment history
echo "Deployment History:"
kubectl rollout history deployment/$DEPLOYMENT -n $NAMESPACE

# รับ revision ที่ต้องการ rollback
if [ -n "${TARGET_REVISION:-}" ]; then
  REVISION=$TARGET_REVISION
  echo "Rolling back to revision: $REVISION"
  kubectl rollout undo deployment/$DEPLOYMENT \
    --to-revision=$REVISION \
    -n $NAMESPACE
else
  echo "Rolling back to previous version..."
  kubectl rollout undo deployment/$DEPLOYMENT -n $NAMESPACE
fi

# รอให้ rollback เสร็จ
echo "Waiting for rollback to complete..."
kubectl rollout status deployment/$DEPLOYMENT \
  -n $NAMESPACE \
  --timeout=$TIMEOUT

# ตรวจสอบ health
echo "Running health check..."
sleep 30

HEALTH_STATUS=$(curl -s -o /dev/null -w "%{http_code}" \
  https://bot.your-company.com/health)

if [ "$HEALTH_STATUS" = "200" ]; then
  echo "✅ Rollback successful! Health check passed."
  
  # แจ้ง Slack
  if [ -n "${SLACK_WEBHOOK_URL:-}" ]; then
    curl -X POST "$SLACK_WEBHOOK_URL" \
      -H "Content-Type: application/json" \
      -d "{\"text\":\"🔄 LINE Bot rolled back successfully to previous version\"}"
  fi
else
  echo "❌ Health check failed after rollback: $HEALTH_STATUS"
  exit 1
fi
```

### 8.2 GitHub Actions Rollback Workflow

```yaml
# .github/workflows/rollback.yml
name: Emergency Rollback

on:
  workflow_dispatch:
    inputs:
      revision:
        description: 'K8s deployment revision number (blank = previous)'
        required: false
        type: string
      reason:
        description: 'Reason for rollback'
        required: true
        type: string

jobs:
  rollback:
    name: Rollback Production
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.PRODUCTION_KUBECONFIG }}
      
      - name: Execute rollback
        run: |
          if [ -n "${{ inputs.revision }}" ]; then
            kubectl rollout undo deployment/linebot \
              --to-revision=${{ inputs.revision }} \
              -n linebot-production
          else
            kubectl rollout undo deployment/linebot \
              -n linebot-production
          fi
          
          kubectl rollout status deployment/linebot \
            -n linebot-production \
            --timeout=5m
      
      - name: Verify rollback
        run: |
          sleep 30
          curl -f https://bot.your-company.com/health
      
      - name: Record rollback
        uses: actions/github-script@v7
        with:
          script: |
            github.rest.issues.create({
              owner: context.repo.owner,
              repo: context.repo.repo,
              title: `[Rollback] Production rollback - ${{ inputs.reason }}`,
              body: `
              ## Production Rollback
              
              **Reason:** ${{ inputs.reason }}
              **Rolled back by:** ${{ github.actor }}
              **Time:** ${new Date().toISOString()}
              **Target revision:** ${{ inputs.revision || 'previous' }}
              
              Please investigate and fix the issue before re-deploying.
              `,
              labels: ['incident', 'rollback']
            })
```

---

## 9. Webhook URL Management

### 9.1 Dynamic Webhook URL per Environment

```javascript
// scripts/updateWebhookUrl.js
const axios = require('axios');

async function updateLineWebhookUrl(accessToken, webhookUrl) {
  console.log(`Updating LINE webhook URL to: ${webhookUrl}`);
  
  try {
    // ตรวจสอบ URL ก่อน
    const verifyResponse = await axios.post(
      'https://api.line.me/v2/bot/channel/webhook/test',
      { webhookEndpointUrl: webhookUrl },
      {
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    if (!verifyResponse.data.success) {
      throw new Error(`Webhook URL test failed: ${JSON.stringify(verifyResponse.data)}`);
    }
    
    // อัปเดต webhook URL
    await axios.put(
      'https://api.line.me/v2/bot/channel/webhook/endpoint',
      { webhookEndpointUrl: webhookUrl },
      {
        headers: {
          'Authorization': `Bearer ${accessToken}`,
          'Content-Type': 'application/json'
        }
      }
    );
    
    console.log(`✅ Webhook URL updated successfully to: ${webhookUrl}`);
    
  } catch (err) {
    console.error('Failed to update webhook URL:', err.response?.data || err.message);
    process.exit(1);
  }
}

// ใช้งาน
const accessToken = process.env.LINE_CHANNEL_ACCESS_TOKEN;
const webhookUrl = process.env.WEBHOOK_URL;

if (!accessToken || !webhookUrl) {
  console.error('Missing LINE_CHANNEL_ACCESS_TOKEN or WEBHOOK_URL');
  process.exit(1);
}

updateLineWebhookUrl(accessToken, webhookUrl);
```

### 9.2 Ngrok สำหรับ Development

```javascript
// scripts/dev-start.js
const ngrok = require('@ngrok/ngrok');
const axios = require('axios');

async function startDevEnvironment() {
  // เริ่ม ngrok tunnel
  const listener = await ngrok.connect({
    addr: 3000,
    authtoken: process.env.NGROK_AUTH_TOKEN,
    domain: process.env.NGROK_DOMAIN // static domain (paid)
  });
  
  const publicUrl = listener.url();
  const webhookUrl = `${publicUrl}/webhook`;
  
  console.log(`🚇 ngrok URL: ${publicUrl}`);
  console.log(`🔗 Webhook URL: ${webhookUrl}`);
  
  // อัปเดต LINE webhook URL อัตโนมัติ
  await axios.put(
    'https://api.line.me/v2/bot/channel/webhook/endpoint',
    { webhookEndpointUrl: webhookUrl },
    {
      headers: {
        'Authorization': `Bearer ${process.env.LINE_CHANNEL_ACCESS_TOKEN}`,
        'Content-Type': 'application/json'
      }
    }
  );
  
  console.log('✅ LINE webhook URL updated');
  
  // เริ่ม app
  require('../src/app');
  
  // Cleanup เมื่อปิด
  process.on('SIGINT', async () => {
    await ngrok.disconnect();
    process.exit(0);
  });
}

startDevEnvironment();
```

---

## 10. Test Automation

### 10.1 Unit Test ตัวอย่าง

```javascript
// tests/unit/handlers/messageHandler.test.js
const { handleTextMessage } = require('../../../src/handlers/messageHandler');
const { createMockEvent } = require('../../helpers/mockFactory');

// Mock LINE Client
jest.mock('../../../src/config/lineClient', () => ({
  replyMessage: jest.fn().mockResolvedValue({}),
  pushMessage: jest.fn().mockResolvedValue({})
}));

describe('Message Handler', () => {
  let lineClient;
  
  beforeEach(() => {
    lineClient = require('../../../src/config/lineClient');
    jest.clearAllMocks();
  });
  
  describe('handleTextMessage', () => {
    test('ตอบกลับ "สวัสดี" เมื่อได้รับ greeting', async () => {
      const event = createMockEvent({
        type: 'message',
        message: { type: 'text', text: 'สวัสดี' },
        source: { type: 'user', userId: 'U1234567890abcdef1234567890abcdef' }
      });
      
      await handleTextMessage(event);
      
      expect(lineClient.replyMessage).toHaveBeenCalledWith(
        event.replyToken,
        expect.objectContaining({
          type: 'text',
          text: expect.stringContaining('สวัสดี')
        })
      );
    });
    
    test('ไม่ตอบกลับเมื่อ userId ไม่ถูกต้อง', async () => {
      const event = createMockEvent({
        type: 'message',
        message: { type: 'text', text: 'สวัสดี' },
        source: { type: 'user', userId: 'invalid-id' }
      });
      
      await handleTextMessage(event);
      
      expect(lineClient.replyMessage).not.toHaveBeenCalled();
    });
  });
});

// tests/helpers/mockFactory.js
function createMockEvent(overrides = {}) {
  return {
    type: 'message',
    mode: 'active',
    timestamp: Date.now(),
    replyToken: 'test-reply-token-' + Math.random(),
    source: {
      type: 'user',
      userId: 'Uxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
    },
    message: {
      id: '12345678901234',
      type: 'text',
      text: 'test message'
    },
    ...overrides
  };
}

module.exports = { createMockEvent };
```

### 10.2 Integration Test ตัวอย่าง

```javascript
// tests/integration/webhook.test.js
const request = require('supertest');
const crypto = require('crypto');
const app = require('../../src/app');
const db = require('../../src/config/database');

describe('Webhook Integration Tests', () => {
  beforeAll(async () => {
    await db.migrate.latest();
    await db.seed.run();
  });
  
  afterAll(async () => {
    await db.destroy();
  });
  
  function createSignature(body) {
    return crypto
      .createHmac('SHA256', process.env.LINE_CHANNEL_SECRET)
      .update(typeof body === 'string' ? body : JSON.stringify(body))
      .digest('base64');
  }
  
  test('บันทึก user ลง database เมื่อรับ follow event', async () => {
    const body = {
      events: [{
        type: 'follow',
        source: {
          type: 'user',
          userId: 'Utest123456789012345678901234567'
        },
        timestamp: Date.now(),
        replyToken: 'test-reply-token'
      }]
    };
    
    const bodyStr = JSON.stringify(body);
    const signature = createSignature(bodyStr);
    
    await request(app)
      .post('/webhook')
      .set('X-Line-Signature', signature)
      .set('Content-Type', 'application/json')
      .send(bodyStr)
      .expect(200);
    
    // ตรวจสอบ database
    const user = await db('users')
      .where({ line_user_id: 'Utest123456789012345678901234567' })
      .first();
    
    expect(user).toBeDefined();
    expect(user.is_following).toBe(true);
  });
  
  test('ลบ user เมื่อรับ unfollow event', async () => {
    // เพิ่ม user ก่อน
    await db('users').insert({
      line_user_id: 'Utest_unfollow_123456789012345',
      is_following: true
    });
    
    const body = {
      events: [{
        type: 'unfollow',
        source: {
          type: 'user',
          userId: 'Utest_unfollow_123456789012345'
        },
        timestamp: Date.now()
      }]
    };
    
    const bodyStr = JSON.stringify(body);
    const signature = createSignature(bodyStr);
    
    await request(app)
      .post('/webhook')
      .set('X-Line-Signature', signature)
      .set('Content-Type', 'application/json')
      .send(bodyStr)
      .expect(200);
    
    const user = await db('users')
      .where({ line_user_id: 'Utest_unfollow_123456789012345' })
      .first();
    
    expect(user.is_following).toBe(false);
  });
});
```

---

## 11. Feature Branch Deployment

### 11.1 Preview Environments

```yaml
# .github/workflows/preview-environment.yml
name: Feature Preview Environment

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  create-preview:
    name: Create Preview Environment
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout
        uses: actions/checkout@v4
      
      - name: Get PR number
        id: pr
        run: echo "number=${{ github.event.pull_request.number }}" >> $GITHUB_OUTPUT
      
      - name: Build preview image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ghcr.io/${{ github.repository }}:pr-${{ steps.pr.outputs.number }}
      
      - name: Deploy preview environment
        run: |
          PR_NUM="${{ steps.pr.outputs.number }}"
          NAMESPACE="linebot-preview-pr${PR_NUM}"
          
          # สร้าง namespace ถ้ายังไม่มี
          kubectl create namespace $NAMESPACE --dry-run=client -o yaml | kubectl apply -f -
          
          # Deploy application
          cat <<EOF | kubectl apply -f -
          apiVersion: apps/v1
          kind: Deployment
          metadata:
            name: linebot-preview
            namespace: $NAMESPACE
          spec:
            replicas: 1
            selector:
              matchLabels:
                app: linebot-preview
            template:
              metadata:
                labels:
                  app: linebot-preview
              spec:
                containers:
                  - name: linebot
                    image: ghcr.io/${{ github.repository }}:pr-${PR_NUM}
                    env:
                      - name: NODE_ENV
                        value: staging
                      - name: LINE_CHANNEL_SECRET
                        valueFrom:
                          secretKeyRef:
                            name: line-secrets
                            key: channel-secret-staging
          EOF
          
          echo "Preview URL: https://pr${PR_NUM}.preview.your-company.com"
      
      - name: Comment PR with preview URL
        uses: actions/github-script@v7
        with:
          script: |
            const prNum = ${{ steps.pr.outputs.number }};
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: `
              🔍 **Preview Environment Ready!**
              
              | URL | https://pr${prNum}.preview.your-company.com |
              |-----|---|
              | Status | ✅ Running |
              | Image | \`pr-${prNum}\` |
              
              Preview will be deleted when PR is closed.
              `
            })
  
  cleanup-preview:
    name: Cleanup Preview Environment
    runs-on: ubuntu-latest
    if: github.event_name == 'pull_request' && github.event.action == 'closed'
    
    steps:
      - name: Delete preview namespace
        run: |
          PR_NUM="${{ github.event.pull_request.number }}"
          kubectl delete namespace linebot-preview-pr${PR_NUM} --ignore-not-found
```

---

## 12. Package.json Scripts

### 12.1 Complete npm Scripts

```json
{
  "name": "linebot",
  "version": "1.0.0",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "dev:tunnel": "node scripts/dev-start.js",
    
    "test": "jest",
    "test:unit": "jest tests/unit --testPathPattern=unit",
    "test:integration": "jest tests/integration --testPathPattern=integration",
    "test:e2e": "jest tests/e2e --testPathPattern=e2e",
    "test:coverage": "jest --coverage",
    "test:watch": "jest --watch",
    
    "lint": "eslint src tests --ext .js,.ts",
    "lint:fix": "eslint src tests --ext .js,.ts --fix",
    "format": "prettier --write 'src/**/*.js' 'tests/**/*.js'",
    "format:check": "prettier --check 'src/**/*.js' 'tests/**/*.js'",
    "type-check": "tsc --noEmit",
    
    "db:migrate": "knex migrate:latest",
    "db:migrate:rollback": "knex migrate:rollback",
    "db:migrate:test": "NODE_ENV=test knex migrate:latest",
    "db:seed": "knex seed:run",
    "db:seed:test": "NODE_ENV=test knex seed:run",
    
    "build": "npm run lint && npm test && docker build -t linebot .",
    "docker:build": "docker build -t linebot:latest .",
    "docker:run": "docker-compose up",
    "docker:stop": "docker-compose down",
    
    "deploy:staging": "bash scripts/deploy.sh staging",
    "deploy:production": "bash scripts/deploy.sh production",
    "rollback": "bash scripts/rollback.sh",
    
    "security:audit": "npm audit --production",
    "security:fix": "npm audit fix",
    "security:scan": "snyk test",
    
    "update-webhook": "node scripts/updateWebhookUrl.js"
  },
  "jest": {
    "testEnvironment": "node",
    "coverageThreshold": {
      "global": {
        "branches": 70,
        "functions": 70,
        "lines": 70,
        "statements": 70
      }
    },
    "collectCoverageFrom": [
      "src/**/*.js",
      "!src/config/**",
      "!src/migrations/**"
    ]
  }
}
```

---

## สรุปบทที่ 79

### CI/CD Best Practices

```
CI/CD Pipeline Best Practices:
┌──────────────────────────────────────────────────────────────┐
│                                                               │
│  ✅ ทดสอบทุก push ด้วย GitHub Actions                        │
│  ✅ Build Docker image สำหรับทุก environment                 │
│  ✅ Deploy staging อัตโนมัติ (no manual step)                │
│  ✅ ต้องมี approval ก่อน deploy production                   │
│  ✅ Zero-downtime deployment ด้วย Rolling Update             │
│  ✅ Rollback อัตโนมัติเมื่อ health check ล้มเหลว            │
│  ✅ Secrets ใน GitHub Secrets / Vault                        │
│  ✅ Preview environments สำหรับ Pull Requests               │
│  ✅ Security scanning ทุก commit                             │
│  ✅ บันทึก audit trail สำหรับทุก deployment                 │
│                                                               │
└──────────────────────────────────────────────────────────────┘
```

**ต่อไป**: [Part 80 - Docker & Kubernetes](./part-080-docker-kubernetes.md)
