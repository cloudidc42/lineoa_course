# Part 80: Docker & Kubernetes สำหรับ LINE Bot

## บทนำ

Docker และ Kubernetes เป็นเครื่องมือสำคัญสำหรับ deploy LINE Bot ใน production อย่างมีประสิทธิภาพ

- **Docker**: Package application และ dependencies ให้รันได้ทุกที่
- **Kubernetes (K8s)**: จัดการ container หลาย ๆ ตัว scaling, self-healing, load balancing

```
Docker & Kubernetes Architecture:
┌────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                           │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                   Ingress Controller                     │  │
│  │         (LINE Webhook → /webhook)                       │  │
│  └──────────────────────┬──────────────────────────────────┘  │
│                          │                                      │
│  ┌──────────────────────▼──────────────────────────────────┐  │
│  │                   LINE Bot Service                       │  │
│  │                                                          │  │
│  │   ┌──────────┐  ┌──────────┐  ┌──────────┐            │  │
│  │   │  Pod 1   │  │  Pod 2   │  │  Pod 3   │            │  │
│  │   │ linebot  │  │ linebot  │  │ linebot  │            │  │
│  │   └──────────┘  └──────────┘  └──────────┘            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │  MySQL Pod │  │  Redis Pod   │  │  Config/Secrets       │  │
│  └────────────┘  └──────────────┘  └───────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

---

## 1. Dockerfile สำหรับ Node.js LINE Bot

### 1.1 Production Dockerfile (Multi-stage)

```dockerfile
# docker/Dockerfile
# =============================================
# Stage 1: Builder
# =============================================
FROM node:20-alpine AS builder

# ติดตั้ง dependencies สำหรับ build
RUN apk add --no-cache python3 make g++

WORKDIR /app

# Copy package files ก่อน (เพื่อ cache)
COPY package*.json ./

# ติดตั้ง dependencies รวม devDependencies (สำหรับ build)
RUN npm ci

# Copy source code
COPY . .

# Build (ถ้า TypeScript)
RUN npm run build 2>/dev/null || true

# =============================================
# Stage 2: Production
# =============================================
FROM node:20-alpine AS production

# Security: อย่ารันเป็น root
RUN addgroup -g 1001 -S nodejs && \
    adduser -S linebot -u 1001 -G nodejs

# ติดตั้ง dumb-init สำหรับ proper signal handling
RUN apk add --no-cache dumb-init

WORKDIR /app

# Copy package files
COPY package*.json ./

# ติดตั้งเฉพาะ production dependencies
RUN npm ci --production && \
    npm cache clean --force

# Copy built code จาก builder stage
COPY --from=builder --chown=linebot:nodejs /app/src ./src
COPY --from=builder --chown=linebot:nodejs /app/dist ./dist 2>/dev/null || true

# สร้าง log directory
RUN mkdir -p /var/log/linebot && \
    chown -R linebot:nodejs /var/log/linebot

# เปลี่ยนเป็น non-root user
USER linebot

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', (r) => r.statusCode === 200 ? process.exit(0) : process.exit(1))"

EXPOSE 3000

# ใช้ dumb-init เพื่อ handle signals ถูกต้อง
ENTRYPOINT ["dumb-init", "--"]
CMD ["node", "src/server.js"]
```

### 1.2 Development Dockerfile

```dockerfile
# docker/Dockerfile.dev
FROM node:20-alpine

# ติดตั้ง tools สำหรับ development
RUN apk add --no-cache \
    python3 \
    make \
    g++ \
    curl \
    git

# ติดตั้ง nodemon globally
RUN npm install -g nodemon

WORKDIR /app

# Copy package files
COPY package*.json ./
RUN npm install

# ไม่ copy source code (ใช้ volume mount แทน)

EXPOSE 3000
EXPOSE 9229  # Node.js debugger port

# Watch mode with debugging
CMD ["nodemon", "--inspect=0.0.0.0:9229", "src/server.js"]
```

### 1.3 Build และ Run

```bash
# Build production image
docker build -f docker/Dockerfile -t linebot:latest .

# Build development image
docker build -f docker/Dockerfile.dev -t linebot:dev .

# Run production
docker run -d \
  --name linebot \
  -p 3000:3000 \
  -e LINE_CHANNEL_SECRET=your-secret \
  -e LINE_CHANNEL_ACCESS_TOKEN=your-token \
  -e DB_HOST=host.docker.internal \
  linebot:latest

# ดู logs
docker logs -f linebot

# เข้า container
docker exec -it linebot sh
```

---

## 2. Dockerfile สำหรับ Python LINE Bot

### 2.1 Python Production Dockerfile

```dockerfile
# docker/Dockerfile.python
# =============================================
# Stage 1: Builder
# =============================================
FROM python:3.11-slim AS builder

WORKDIR /app

# ติดตั้ง build dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
    build-essential \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .

# ติดตั้ง dependencies ใน virtual env
RUN python -m venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"
RUN pip install --no-cache-dir -r requirements.txt

# =============================================
# Stage 2: Production
# =============================================
FROM python:3.11-slim AS production

# Security: สร้าง non-root user
RUN groupadd -r linebot && useradd -r -g linebot linebot

# Copy virtual env จาก builder
COPY --from=builder /opt/venv /opt/venv
ENV PATH="/opt/venv/bin:$PATH"

WORKDIR /app

# Copy application
COPY --chown=linebot:linebot . .

# สร้าง log directory
RUN mkdir -p /var/log/linebot && \
    chown -R linebot:linebot /var/log/linebot

USER linebot

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=30s --retries=3 \
    CMD python -c "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"

EXPOSE 8000

# ใช้ gunicorn สำหรับ production
CMD ["gunicorn", \
     "--workers", "4", \
     "--worker-class", "uvicorn.workers.UvicornWorker", \
     "--bind", "0.0.0.0:8000", \
     "--timeout", "60", \
     "--graceful-timeout", "30", \
     "--access-logfile", "-", \
     "--error-logfile", "-", \
     "main:app"]
```

```text
# requirements.txt
line-bot-sdk==3.11.0
fastapi==0.104.1
uvicorn==0.24.0
gunicorn==21.2.0
sqlalchemy==2.0.23
alembic==1.12.1
redis==5.0.1
python-dotenv==1.0.0
pydantic==2.5.2
pydantic-settings==2.1.0
cryptography==41.0.7
prometheus-client==0.19.0
structlog==23.2.0
```

---

## 3. Docker Compose สำหรับ Development

### 3.1 Development docker-compose.yml

```yaml
# docker-compose.yml
version: '3.8'

services:
  # LINE Bot Application
  app:
    build:
      context: .
      dockerfile: docker/Dockerfile.dev
    ports:
      - "3000:3000"
      - "9229:9229"  # Debugger
    volumes:
      - .:/app
      - /app/node_modules  # ใช้ node_modules ใน container
      - ./logs:/var/log/linebot
    environment:
      - NODE_ENV=development
      - DB_HOST=db
      - DB_PORT=3306
      - DB_NAME=linebot_dev
      - DB_USER=linebot
      - DB_PASSWORD=devpassword
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    env_file:
      - .env.development
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - linebot-network
    restart: unless-stopped

  # MySQL Database
  db:
    image: mysql:8.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: linebot_dev
      MYSQL_USER: linebot
      MYSQL_PASSWORD: devpassword
    volumes:
      - mysql_data:/var/lib/mysql
      - ./docker/mysql/init.sql:/docker-entrypoint-initdb.d/init.sql
      - ./docker/mysql/my.cnf:/etc/mysql/conf.d/my.cnf
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "linebot", "-pdevpassword"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - linebot-network

  # Redis
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --appendonly yes --requirepass devpassword
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "devpassword", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - linebot-network

  # phpMyAdmin (สำหรับ development)
  phpmyadmin:
    image: phpmyadmin:latest
    ports:
      - "8080:80"
    environment:
      PMA_HOST: db
      PMA_USER: linebot
      PMA_PASSWORD: devpassword
    depends_on:
      - db
    networks:
      - linebot-network

  # Redis Commander (สำหรับ development)
  redis-commander:
    image: rediscommander/redis-commander:latest
    ports:
      - "8081:8081"
    environment:
      REDIS_HOSTS: "local:redis:6379:0:devpassword"
    depends_on:
      - redis
    networks:
      - linebot-network

  # Mailhog (ทดสอบ email)
  mailhog:
    image: mailhog/mailhog:latest
    ports:
      - "1025:1025"
      - "8025:8025"
    networks:
      - linebot-network

networks:
  linebot-network:
    driver: bridge

volumes:
  mysql_data:
  redis_data:
```

### 3.2 Test docker-compose.yml

```yaml
# docker-compose.test.yml
version: '3.8'

services:
  test:
    build:
      context: .
      dockerfile: docker/Dockerfile.dev
    command: npm run test:integration
    environment:
      - NODE_ENV=test
      - DB_HOST=db-test
      - DB_NAME=linebot_test
      - DB_USER=linebot
      - DB_PASSWORD=testpassword
      - REDIS_HOST=redis-test
      - LINE_CHANNEL_SECRET=test-secret
      - LINE_CHANNEL_ACCESS_TOKEN=test-token
    depends_on:
      db-test:
        condition: service_healthy
      redis-test:
        condition: service_healthy
    networks:
      - test-network

  db-test:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: linebot_test
      MYSQL_USER: linebot
      MYSQL_PASSWORD: testpassword
    tmpfs:
      - /var/lib/mysql  # ใช้ RAM สำหรับ speed
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 5s
      retries: 10
    networks:
      - test-network

  redis-test:
    image: redis:7-alpine
    networks:
      - test-network

networks:
  test-network:
    driver: bridge
```

---

## 4. Kubernetes Basics

### 4.1 ทำความเข้าใจ Kubernetes Components

```
Kubernetes Components:
┌─────────────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                            │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                   Control Plane                           │  │
│  │  ┌────────────┐  ┌──────────────┐  ┌─────────────────┐  │  │
│  │  │ API Server │  │   Scheduler  │  │  Controller Mgr │  │  │
│  │  └────────────┘  └──────────────┘  └─────────────────┘  │  │
│  │  ┌────────────────────────────┐                           │  │
│  │  │         etcd               │                           │  │
│  │  └────────────────────────────┘                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                     Worker Node 1                         │  │
│  │  ┌──────────┐  ┌──────────┐  ┌────────────────────────┐ │  │
│  │  │  Pod A   │  │  Pod B   │  │       kubelet           │ │  │
│  │  │ linebot  │  │ linebot  │  │   (node agent)          │ │  │
│  │  └──────────┘  └──────────┘  └────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 5. Deployment YAML สำหรับ LINE Bot

### 5.1 Namespace

```yaml
# k8s/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: linebot-production
  labels:
    name: linebot-production
    environment: production
```

### 5.2 Deployment YAML

```yaml
# k8s/production/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: linebot
  namespace: linebot-production
  labels:
    app: linebot
    version: "1.0.0"
    environment: production
  annotations:
    kubernetes.io/change-cause: "Initial deployment"
spec:
  replicas: 3
  
  # Rolling Update
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  
  selector:
    matchLabels:
      app: linebot
  
  template:
    metadata:
      labels:
        app: linebot
        version: "1.0.0"
      annotations:
        # Force pod restart เมื่อ secret เปลี่ยน
        secret-hash: "{{ sha256sum .Values.secrets }}"
    
    spec:
      # Service Account
      serviceAccountName: linebot-sa
      
      # Security Context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        runAsGroup: 1001
        fsGroup: 1001
      
      # ป้องกัน scheduling บน same node (High Availability)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - linebot
                topologyKey: kubernetes.io/hostname
      
      containers:
        - name: linebot
          image: ghcr.io/your-org/linebot:latest
          imagePullPolicy: Always
          
          ports:
            - name: http
              containerPort: 3000
              protocol: TCP
          
          # Environment จาก ConfigMap
          envFrom:
            - configMapRef:
                name: linebot-config
          
          # Secrets
          env:
            - name: LINE_CHANNEL_SECRET
              valueFrom:
                secretKeyRef:
                  name: linebot-secrets
                  key: channel-secret
            
            - name: LINE_CHANNEL_ACCESS_TOKEN
              valueFrom:
                secretKeyRef:
                  name: linebot-secrets
                  key: channel-access-token
            
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: linebot-secrets
                  key: db-password
            
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: linebot-secrets
                  key: redis-password
            
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          
          # Readiness Probe
          readinessProbe:
            httpGet:
              path: /ready
              port: 3000
            initialDelaySeconds: 15
            periodSeconds: 5
            successThreshold: 1
            failureThreshold: 3
            timeoutSeconds: 3
          
          # Liveness Probe
          livenessProbe:
            httpGet:
              path: /health
              port: 3000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
            timeoutSeconds: 5
          
          # Startup Probe
          startupProbe:
            httpGet:
              path: /health
              port: 3000
            failureThreshold: 30
            periodSeconds: 5
          
          # Resources
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          
          # Graceful Shutdown
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]
          
          # Security Context
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
          
          # Volume Mounts
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: logs
              mountPath: /var/log/linebot
      
      # Graceful Shutdown Period
      terminationGracePeriodSeconds: 60
      
      # Volumes
      volumes:
        - name: tmp
          emptyDir: {}
        - name: logs
          emptyDir: {}
      
      # Image Pull Secret (ถ้า private registry)
      imagePullSecrets:
        - name: ghcr-secret
```

---

## 6. Service YAML

### 6.1 ClusterIP Service

```yaml
# k8s/production/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: linebot-service
  namespace: linebot-production
  labels:
    app: linebot
spec:
  type: ClusterIP
  selector:
    app: linebot
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 3000
  
  # Session Affinity (optional สำหรับ stateful operations)
  sessionAffinity: None
```

---

## 7. ConfigMap และ Secrets

### 7.1 ConfigMap

```yaml
# k8s/production/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: linebot-config
  namespace: linebot-production
data:
  # Application settings
  NODE_ENV: "production"
  PORT: "3000"
  LOG_LEVEL: "info"
  
  # Database
  DB_HOST: "mysql-service.linebot-production.svc.cluster.local"
  DB_PORT: "3306"
  DB_NAME: "linebot_production"
  DB_USER: "linebot"
  
  # Redis
  REDIS_HOST: "redis-service.linebot-production.svc.cluster.local"
  REDIS_PORT: "6379"
  
  # Features
  RATE_LIMITING_ENABLED: "true"
  MAX_REQUESTS_PER_MINUTE: "60"
  
  # Monitoring
  PROMETHEUS_ENABLED: "true"
  
  # Timezone
  TZ: "Asia/Bangkok"
```

### 7.2 Secrets

```yaml
# k8s/production/secrets.yaml
# ⚠️ ห้าม commit ไฟล์นี้! ใช้ External Secrets Operator แทน
apiVersion: v1
kind: Secret
metadata:
  name: linebot-secrets
  namespace: linebot-production
type: Opaque
# ค่าต้อง encode เป็น base64
# echo -n "your-value" | base64
stringData:
  channel-secret: "REPLACE_WITH_ACTUAL_SECRET"
  channel-access-token: "REPLACE_WITH_ACTUAL_TOKEN"
  db-password: "REPLACE_WITH_ACTUAL_PASSWORD"
  redis-password: "REPLACE_WITH_ACTUAL_PASSWORD"
  jwt-secret: "REPLACE_WITH_ACTUAL_JWT_SECRET"
```

### 7.3 External Secrets Operator (แนะนำสำหรับ Production)

```yaml
# k8s/production/external-secrets.yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: linebot-secrets
  namespace: linebot-production
spec:
  refreshInterval: 1h
  
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  
  target:
    name: linebot-secrets
    creationPolicy: Owner
  
  data:
    - secretKey: channel-secret
      remoteRef:
        key: secret/linebot/production
        property: line_channel_secret
    
    - secretKey: channel-access-token
      remoteRef:
        key: secret/linebot/production
        property: line_channel_access_token
    
    - secretKey: db-password
      remoteRef:
        key: secret/linebot/production
        property: db_password

---
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.your-company.com"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "linebot-production"
          serviceAccountRef:
            name: "linebot-sa"
            namespace: "linebot-production"
```

---

## 8. Horizontal Pod Autoscaler

### 8.1 HPA Configuration

```yaml
# k8s/production/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: linebot-hpa
  namespace: linebot-production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: linebot
  
  minReplicas: 3
  maxReplicas: 20
  
  metrics:
    # Scale ตาม CPU
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    
    # Scale ตาม Memory
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 75
    
    # Scale ตาม custom metric (requests per second)
    - type: Pods
      pods:
        metric:
          name: linebot_webhook_requests_per_second
        target:
          type: AverageValue
          averageValue: "50"
  
  behavior:
    # Scale up อย่างรวดเร็ว
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
        - type: Percent
          value: 100
          periodSeconds: 60
      selectPolicy: Max
    
    # Scale down อย่างช้า ๆ (ป้องกัน thrashing)
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
      selectPolicy: Min
```

---

## 9. Ingress สำหรับ Webhook

### 9.1 Nginx Ingress

```yaml
# k8s/production/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: linebot-ingress
  namespace: linebot-production
  annotations:
    # Nginx annotations
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    
    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "50"
    
    # Body size
    nginx.ingress.kubernetes.io/proxy-body-size: "1m"
    
    # Timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "10"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    
    # SSL
    cert-manager.io/cluster-issuer: "letsencrypt-production"
    
    # Security Headers
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Frame-Options "DENY" always;
      add_header X-Content-Type-Options "nosniff" always;
      add_header X-XSS-Protection "1; mode=block" always;
      add_header Referrer-Policy "strict-origin-when-cross-origin" always;
      add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;

spec:
  ingressClassName: nginx
  
  tls:
    - hosts:
        - bot.your-company.com
      secretName: linebot-tls
  
  rules:
    - host: bot.your-company.com
      http:
        paths:
          - path: /webhook
            pathType: Exact
            backend:
              service:
                name: linebot-service
                port:
                  number: 80
          
          - path: /health
            pathType: Exact
            backend:
              service:
                name: linebot-service
                port:
                  number: 80
          
          - path: /liff
            pathType: Prefix
            backend:
              service:
                name: linebot-service
                port:
                  number: 80
```

### 9.2 Certificate Management

```yaml
# k8s/production/certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: linebot-tls
  namespace: linebot-production
spec:
  secretName: linebot-tls
  issuerRef:
    name: letsencrypt-production
    kind: ClusterIssuer
  
  dnsNames:
    - bot.your-company.com
    - staging-bot.your-company.com
  
  # Auto-renewal
  renewBefore: 360h  # 15 days before expiry

---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@your-company.com
    privateKeySecretRef:
      name: letsencrypt-production
    solvers:
      - http01:
          ingress:
            class: nginx
```

---

## 10. Helm Chart สำหรับ LINE Bot

### 10.1 Chart Structure

```
helm/linebot/
├── Chart.yaml
├── values.yaml
├── values.staging.yaml
├── values.production.yaml
├── templates/
│   ├── _helpers.tpl
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   ├── pdb.yaml
│   └── NOTES.txt
└── charts/
```

### 10.2 Chart.yaml

```yaml
# helm/linebot/Chart.yaml
apiVersion: v2
name: linebot
description: LINE Official Account Bot
type: application
version: 1.0.0
appVersion: "1.0.0"

dependencies:
  - name: mysql
    version: "9.14.1"
    repository: "https://charts.bitnami.com/bitnami"
    condition: mysql.enabled
  
  - name: redis
    version: "18.4.0"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

### 10.3 values.yaml

```yaml
# helm/linebot/values.yaml
# Default values สำหรับ linebot chart

replicaCount: 3

image:
  repository: ghcr.io/your-org/linebot
  tag: latest
  pullPolicy: Always

# Service Account
serviceAccount:
  create: true
  name: linebot-sa
  annotations: {}

# Pod configuration
podAnnotations: {}
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1001
  fsGroup: 1001

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL

# Service
service:
  type: ClusterIP
  port: 80
  targetPort: 3000

# Ingress
ingress:
  enabled: true
  className: nginx
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-production
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
  hosts:
    - host: bot.your-company.com
      paths:
        - path: /webhook
          pathType: Exact
        - path: /health
          pathType: Exact
        - path: /liff
          pathType: Prefix
  tls:
    - secretName: linebot-tls
      hosts:
        - bot.your-company.com

# Resources
resources:
  requests:
    memory: "256Mi"
    cpu: "250m"
  limits:
    memory: "512Mi"
    cpu: "500m"

# Autoscaling
autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 75

# Pod Disruption Budget
podDisruptionBudget:
  enabled: true
  minAvailable: 2

# Health Checks
healthCheck:
  readiness:
    initialDelaySeconds: 15
    periodSeconds: 5
    failureThreshold: 3
  liveness:
    initialDelaySeconds: 30
    periodSeconds: 10
    failureThreshold: 3

# Application Config
config:
  nodeEnv: production
  port: "3000"
  logLevel: info
  rateLimitingEnabled: "true"
  timezone: "Asia/Bangkok"

# Secrets (จาก External Secrets หรือ Vault)
secrets:
  existingSecret: "linebot-secrets"

# MySQL (ใช้ Bitnami chart)
mysql:
  enabled: true
  auth:
    database: linebot_production
    username: linebot
    existingSecret: mysql-secrets

# Redis (ใช้ Bitnami chart)
redis:
  enabled: true
  auth:
    existingSecret: redis-secrets
  architecture: standalone
```

### 10.4 Deployment Template

```yaml
# helm/linebot/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "linebot.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "linebot.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  
  selector:
    matchLabels:
      {{- include "linebot.selectorLabels" . | nindent 6 }}
  
  template:
    metadata:
      labels:
        {{- include "linebot.selectorLabels" . | nindent 8 }}
      annotations:
        {{- toYaml .Values.podAnnotations | nindent 8 }}
    
    spec:
      serviceAccountName: {{ include "linebot.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app.kubernetes.io/name
                      operator: In
                      values:
                        - {{ include "linebot.name" . }}
                topologyKey: kubernetes.io/hostname
      
      containers:
        - name: {{ .Chart.Name }}
          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          
          ports:
            - name: http
              containerPort: {{ .Values.config.port }}
              protocol: TCP
          
          envFrom:
            - configMapRef:
                name: {{ include "linebot.fullname" . }}-config
          
          env:
            - name: LINE_CHANNEL_SECRET
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.secrets.existingSecret }}
                  key: channel-secret
            - name: LINE_CHANNEL_ACCESS_TOKEN
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.secrets.existingSecret }}
                  key: channel-access-token
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ .Values.secrets.existingSecret }}
                  key: db-password
          
          readinessProbe:
            httpGet:
              path: /ready
              port: http
            {{- toYaml .Values.healthCheck.readiness | nindent 12 }}
          
          livenessProbe:
            httpGet:
              path: /health
              port: http
            {{- toYaml .Values.healthCheck.liveness | nindent 12 }}
          
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          
          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 15"]
          
          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: logs
              mountPath: /var/log/linebot
      
      terminationGracePeriodSeconds: 60
      
      volumes:
        - name: tmp
          emptyDir: {}
        - name: logs
          emptyDir: {}
```

### 10.5 Pod Disruption Budget

```yaml
# k8s/production/pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: linebot-pdb
  namespace: linebot-production
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: linebot
```

---

## 11. Helm Commands สำหรับ Deployment

```bash
# ====================================
# Helm Installation & Deployment
# ====================================

# เพิ่ม dependencies
helm dependency update ./helm/linebot

# Dry run (ตรวจสอบก่อน deploy)
helm upgrade --install linebot ./helm/linebot \
  -n linebot-production \
  --create-namespace \
  --dry-run \
  -f helm/linebot/values.production.yaml

# Deploy to production
helm upgrade --install linebot ./helm/linebot \
  -n linebot-production \
  --create-namespace \
  -f helm/linebot/values.production.yaml \
  --set image.tag=$(git rev-parse --short HEAD) \
  --atomic \
  --cleanup-on-fail \
  --timeout 10m

# Deploy to staging
helm upgrade --install linebot ./helm/linebot \
  -n linebot-staging \
  --create-namespace \
  -f helm/linebot/values.staging.yaml \
  --set image.tag=$(git rev-parse --short HEAD)

# ดู release history
helm history linebot -n linebot-production

# Rollback ไปยัง revision ก่อนหน้า
helm rollback linebot -n linebot-production

# Rollback ไปยัง revision เฉพาะ
helm rollback linebot 3 -n linebot-production

# ดู values ที่ใช้งานอยู่
helm get values linebot -n linebot-production

# Uninstall
helm uninstall linebot -n linebot-production
```

---

## 12. Complete K8s Deployment Guide

### 12.1 Initial Setup Steps

```bash
#!/bin/bash
# scripts/k8s-initial-setup.sh

set -euo pipefail

NAMESPACE="linebot-production"
echo "Setting up LINE Bot on Kubernetes..."

# ====================================
# 1. สร้าง Namespace
# ====================================
kubectl apply -f k8s/namespace.yaml

# ====================================
# 2. สร้าง Secrets
# ====================================
echo "Creating secrets..."
kubectl create secret generic linebot-secrets \
  --namespace=$NAMESPACE \
  --from-literal=channel-secret="$LINE_CHANNEL_SECRET" \
  --from-literal=channel-access-token="$LINE_CHANNEL_ACCESS_TOKEN" \
  --from-literal=db-password="$DB_PASSWORD" \
  --from-literal=redis-password="$REDIS_PASSWORD" \
  --dry-run=client -o yaml | kubectl apply -f -

# สร้าง Image Pull Secret (สำหรับ private registry)
kubectl create secret docker-registry ghcr-secret \
  --namespace=$NAMESPACE \
  --docker-server=ghcr.io \
  --docker-username="$GITHUB_ACTOR" \
  --docker-password="$GITHUB_TOKEN" \
  --dry-run=client -o yaml | kubectl apply -f -

# ====================================
# 3. Deploy ConfigMap
# ====================================
kubectl apply -f k8s/production/configmap.yaml

# ====================================
# 4. Deploy Application
# ====================================
kubectl apply -f k8s/production/deployment.yaml

# ====================================
# 5. Deploy Service
# ====================================
kubectl apply -f k8s/production/service.yaml

# ====================================
# 6. Deploy Ingress
# ====================================
kubectl apply -f k8s/production/ingress.yaml

# ====================================
# 7. Deploy HPA
# ====================================
kubectl apply -f k8s/production/hpa.yaml

# ====================================
# 8. Deploy PDB
# ====================================
kubectl apply -f k8s/production/pdb.yaml

# ====================================
# 9. รอให้ Deployment เสร็จ
# ====================================
echo "Waiting for deployment to be ready..."
kubectl rollout status deployment/linebot \
  -n $NAMESPACE \
  --timeout=5m

# ====================================
# 10. ตรวจสอบสถานะ
# ====================================
echo "Deployment status:"
kubectl get pods -n $NAMESPACE
kubectl get services -n $NAMESPACE
kubectl get ingress -n $NAMESPACE

# ====================================
# 11. Health Check
# ====================================
sleep 30
curl -f https://bot.your-company.com/health && \
  echo "✅ Health check passed!" || \
  echo "❌ Health check failed!"
```

### 12.2 Troubleshooting Commands

```bash
# ====================================
# Kubernetes Troubleshooting
# ====================================

NAMESPACE="linebot-production"

# ดู pods ทั้งหมด
kubectl get pods -n $NAMESPACE -o wide

# ดู logs ของ pod
kubectl logs -n $NAMESPACE -l app=linebot --tail=100

# ดู logs แบบ follow
kubectl logs -n $NAMESPACE -l app=linebot -f

# ดู events
kubectl get events -n $NAMESPACE --sort-by='.lastTimestamp'

# Describe pod ที่มีปัญหา
kubectl describe pod <pod-name> -n $NAMESPACE

# เข้า pod
kubectl exec -it <pod-name> -n $NAMESPACE -- sh

# ดู resource usage
kubectl top pods -n $NAMESPACE
kubectl top nodes

# ดู deployment rollout history
kubectl rollout history deployment/linebot -n $NAMESPACE

# ดู HPA status
kubectl get hpa -n $NAMESPACE
kubectl describe hpa linebot-hpa -n $NAMESPACE

# Port forward สำหรับ debugging
kubectl port-forward -n $NAMESPACE service/linebot-service 3000:80

# ดู secret (ไม่แสดงค่า)
kubectl get secret linebot-secrets -n $NAMESPACE
kubectl describe secret linebot-secrets -n $NAMESPACE

# Decode secret (ระวัง!)
kubectl get secret linebot-secrets -n $NAMESPACE \
  -o jsonpath='{.data.channel-secret}' | base64 -d
```

### 12.3 Monitoring Commands

```bash
# ====================================
# Monitoring & Debugging
# ====================================

# ตรวจสอบ webhook connectivity
kubectl exec -it -n linebot-production \
  $(kubectl get pod -n linebot-production -l app=linebot -o name | head -1) \
  -- curl -f http://localhost:3000/health

# ตรวจสอบ database connectivity
kubectl exec -it -n linebot-production \
  $(kubectl get pod -n linebot-production -l app=linebot -o name | head -1) \
  -- node -e "
    const mysql = require('mysql2/promise');
    mysql.createConnection({
      host: process.env.DB_HOST,
      user: process.env.DB_USER,
      password: process.env.DB_PASSWORD,
      database: process.env.DB_NAME
    }).then(conn => {
      console.log('DB connected!');
      conn.end();
    }).catch(err => {
      console.error('DB error:', err.message);
      process.exit(1);
    });
  "

# ตรวจสอบ Redis connectivity
kubectl exec -it -n linebot-production \
  $(kubectl get pod -n linebot-production -l app=linebot -o name | head -1) \
  -- node -e "
    const redis = require('redis');
    const client = redis.createClient({
      host: process.env.REDIS_HOST,
      password: process.env.REDIS_PASSWORD
    });
    client.ping((err, pong) => {
      if (err) { console.error('Redis error:', err); process.exit(1); }
      console.log('Redis:', pong);
      client.quit();
    });
  "
```

---

## 13. Production Checklist

### 13.1 Pre-deployment Checklist

```
Docker & Kubernetes Pre-deployment Checklist:
┌──────────────────────────────────────────────────────────────────┐
│                                                                    │
│  Docker:                                                           │
│  ✅ Multi-stage build เพื่อลดขนาด image                          │
│  ✅ รันเป็น non-root user                                         │
│  ✅ ตั้งค่า HEALTHCHECK                                           │
│  ✅ ใช้ dumb-init สำหรับ signal handling                         │
│  ✅ ไม่มี secrets ใน image                                        │
│  ✅ Image scan สำหรับ vulnerabilities                             │
│                                                                    │
│  Kubernetes:                                                       │
│  ✅ กำหนด resource requests และ limits                            │
│  ✅ ตั้งค่า readiness และ liveness probes                        │
│  ✅ ใช้ Rolling Update strategy                                   │
│  ✅ HPA ตาม CPU/memory metrics                                    │
│  ✅ PodDisruptionBudget สำหรับ HA                                 │
│  ✅ ใช้ Ingress พร้อม TLS/HTTPS                                  │
│  ✅ Secrets จัดเก็บอย่างปลอดภัย                                  │
│  ✅ Network Policies (restrict traffic)                           │
│  ✅ Pod Anti-Affinity สำหรับ HA                                   │
│  ✅ Graceful shutdown (terminationGracePeriodSeconds)             │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

---

## สรุปบทที่ 80

### Key Concepts

```
Docker & Kubernetes Summary:
┌────────────────────────────────────────────────────────────┐
│                                                             │
│  Docker:                                                    │
│  • Multi-stage builds → ขนาด image เล็กลง                  │
│  • Non-root user → security                                 │
│  • dumb-init → proper signal handling                       │
│  • Health checks → container orchestration                  │
│                                                             │
│  Kubernetes:                                                │
│  • Deployment → manage pods                                 │
│  • Service → expose pods internally                         │
│  • Ingress → expose externally with TLS                     │
│  • ConfigMap → configuration                                │
│  • Secret → sensitive data                                  │
│  • HPA → auto-scaling                                       │
│  • PDB → high availability                                  │
│  • Helm → package management                                │
│                                                             │
└────────────────────────────────────────────────────────────┘
```

### ขั้นตอนสรุปการ Deploy

```bash
# 1. Build image
docker build -t ghcr.io/your-org/linebot:v1.0.0 .

# 2. Push image
docker push ghcr.io/your-org/linebot:v1.0.0

# 3. Deploy ด้วย Helm
helm upgrade --install linebot ./helm/linebot \
  -n linebot-production \
  --create-namespace \
  -f values.production.yaml \
  --set image.tag=v1.0.0 \
  --atomic

# 4. ตรวจสอบ
kubectl get pods -n linebot-production
kubectl rollout status deployment/linebot -n linebot-production

# 5. Test webhook
curl https://bot.your-company.com/health
```

---

## แหล่งข้อมูลเพิ่มเติม

- [Docker Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)
- [cert-manager](https://cert-manager.io/docs/)
- [External Secrets Operator](https://external-secrets.io/)

---

## จบ Series Part 76-80

ยินดีด้วย! คุณได้เรียนรู้เนื้อหาขั้นสูงสำหรับ LINE OA ครบแล้ว:

- **Part 76**: Security Hardening
- **Part 77**: PDPA Compliance  
- **Part 78**: Monitoring & Alerting
- **Part 79**: CI/CD Pipeline
- **Part 80**: Docker & Kubernetes

นำความรู้เหล่านี้ไป implement เพื่อสร้าง LINE Bot ที่ปลอดภัย เชื่อถือได้ และพร้อมรองรับ users ในระดับ production!
