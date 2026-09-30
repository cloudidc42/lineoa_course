# Part 93: API Gateway สำหรับระบบ LINE Bot

## บทนำ

API Gateway คือ entry point เดียวสำหรับทุก request ที่เข้าสู่ระบบ LINE Bot ทำหน้าที่เป็น reverse proxy, load balancer, และ security layer ในบทนี้จะครอบคลุมการตั้งค่า Kong Gateway และ Nginx เป็น alternative สำหรับระบบ LINE Bot

---

## 1. API Gateway Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        API GATEWAY LAYER                            │
│                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  LINE    │  │  LIFF    │  │  Admin   │  │  External APIs   │  │
│  │  Webhook │  │  Apps    │  │  Portal  │  │  (Partners)      │  │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────────┬─────────┘  │
│       │              │              │                  │             │
│       └──────────────┴──────────────┴──────────────────┘           │
│                                  │                                   │
│  ┌───────────────────────────────▼───────────────────────────────┐  │
│  │                    Kong API Gateway                             │  │
│  │                                                                 │  │
│  │  ┌─────────────┐  ┌───────────────┐  ┌──────────────────────┐│  │
│  │  │ Rate Limit  │  │ Auth (JWT/Key)│  │ Request Transform    ││  │
│  │  └─────────────┘  └───────────────┘  └──────────────────────┘│  │
│  │  ┌─────────────┐  ┌───────────────┐  ┌──────────────────────┐│  │
│  │  │ Load Balance│  │ Circuit Break │  │ Logging & Metrics    ││  │
│  │  └─────────────┘  └───────────────┘  └──────────────────────┘│  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                  │                                   │
│       ┌──────────────────────────▼──────────────────────────┐       │
│       │              Microservices                            │       │
│       │ ┌──────────┐ ┌──────────┐ ┌────────┐ ┌──────────┐  │       │
│       │ │ Webhook  │ │ Message  │ │  User  │ │Analytics │  │       │
│       │ │ Service  │ │ Service  │ │Service │ │ Service  │  │       │
│       │ └──────────┘ └──────────┘ └────────┘ └──────────┘  │       │
│       └──────────────────────────────────────────────────────┘       │
└────────────────────────────────────────────────────────────────────┘
```

---

## 2. Kong Gateway Setup

### 2.1 Docker Compose สำหรับ Kong

```yaml
# docker-compose.yml
version: '3.8'

services:
  kong-database:
    image: postgres:15-alpine
    container_name: kong-database
    environment:
      POSTGRES_USER: kong
      POSTGRES_DB: kong
      POSTGRES_PASSWORD: ${KONG_DB_PASSWORD}
    volumes:
      - kong_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "kong"]
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - kong-net

  kong-migrations:
    image: kong:3.5
    command: kong migrations bootstrap
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: ${KONG_DB_PASSWORD}
      KONG_PG_DATABASE: kong
    depends_on:
      kong-database:
        condition: service_healthy
    networks:
      - kong-net

  kong:
    image: kong:3.5
    container_name: kong
    environment:
      KONG_DATABASE: postgres
      KONG_PG_HOST: kong-database
      KONG_PG_USER: kong
      KONG_PG_PASSWORD: ${KONG_DB_PASSWORD}
      KONG_PG_DATABASE: kong
      KONG_PROXY_ACCESS_LOG: /dev/stdout
      KONG_ADMIN_ACCESS_LOG: /dev/stdout
      KONG_PROXY_ERROR_LOG: /dev/stderr
      KONG_ADMIN_ERROR_LOG: /dev/stderr
      KONG_PROXY_LISTEN: 0.0.0.0:8000, 0.0.0.0:8443 ssl
      KONG_ADMIN_LISTEN: 0.0.0.0:8001
      KONG_ADMIN_GUI_URL: http://localhost:8002
      KONG_SSL_CERT: /etc/kong/ssl/kong.crt
      KONG_SSL_CERT_KEY: /etc/kong/ssl/kong.key
      KONG_PLUGINS: bundled,jwt-auth-extended,rate-limiting-advanced,request-transformer-advanced
      KONG_LOG_LEVEL: notice
    ports:
      - "80:8000"
      - "443:8443"
      - "8001:8001"
      - "8002:8002"
    volumes:
      - ./kong/ssl:/etc/kong/ssl
      - ./kong/plugins:/usr/local/share/lua/5.1/kong/plugins
    depends_on:
      - kong-migrations
    healthcheck:
      test: ["CMD", "kong", "health"]
      interval: 30s
      timeout: 10s
      retries: 5
    networks:
      - kong-net

  konga:
    image: pantsel/konga:latest
    container_name: konga
    environment:
      NODE_ENV: production
      DB_ADAPTER: postgres
      DB_HOST: kong-database
      DB_USER: kong
      DB_PASSWORD: ${KONG_DB_PASSWORD}
      DB_DATABASE: konga
      KONGA_HOOK_TIMEOUT: 120000
    ports:
      - "1337:1337"
    depends_on:
      - kong-database
    networks:
      - kong-net

  prometheus:
    image: prom/prometheus:latest
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    networks:
      - kong-net

  grafana:
    image: grafana/grafana:latest
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/dashboards:/var/lib/grafana/dashboards
    ports:
      - "3000:3000"
    networks:
      - kong-net

volumes:
  kong_data:
  grafana_data:

networks:
  kong-net:
    driver: bridge
```

### 2.2 Kong Configuration Script

```bash
#!/bin/bash
# scripts/configure-kong.sh

KONG_ADMIN="http://localhost:8001"

echo "=== Configuring Kong for LINE Bot System ==="

# 1. Create Services

# Webhook Service
curl -s -X POST $KONG_ADMIN/services \
  --data "name=webhook-service" \
  --data "url=http://webhook-service:3001" \
  --data "connect_timeout=5000" \
  --data "write_timeout=60000" \
  --data "read_timeout=60000"

# Message Service
curl -s -X POST $KONG_ADMIN/services \
  --data "name=message-service" \
  --data "url=http://message-service:3002" \
  --data "connect_timeout=5000" \
  --data "write_timeout=30000" \
  --data "read_timeout=30000"

# User Service
curl -s -X POST $KONG_ADMIN/services \
  --data "name=user-service" \
  --data "url=http://user-service:3003"

# Analytics Service
curl -s -X POST $KONG_ADMIN/services \
  --data "name=analytics-service" \
  --data "url=http://analytics-service:3004"

# Admin Service
curl -s -X POST $KONG_ADMIN/services \
  --data "name=admin-service" \
  --data "url=http://admin-service:3005"

# 2. Create Routes

# Webhook Route (ไม่ต้องการ authentication)
curl -s -X POST $KONG_ADMIN/services/webhook-service/routes \
  --data "name=webhook-route" \
  --data "paths[]=/webhook" \
  --data "methods[]=POST" \
  --data "strip_path=false"

# API Routes
curl -s -X POST $KONG_ADMIN/services/message-service/routes \
  --data "name=message-route" \
  --data "paths[]=/api/v1/messages" \
  --data "methods[]=GET" \
  --data "methods[]=POST" \
  --data "strip_path=false"

curl -s -X POST $KONG_ADMIN/services/user-service/routes \
  --data "name=user-route" \
  --data "paths[]=/api/v1/users" \
  --data "methods[]=GET" \
  --data "methods[]=PUT" \
  --data "strip_path=false"

curl -s -X POST $KONG_ADMIN/services/analytics-service/routes \
  --data "name=analytics-route" \
  --data "paths[]=/api/v1/analytics" \
  --data "methods[]=GET" \
  --data "strip_path=false"

curl -s -X POST $KONG_ADMIN/services/admin-service/routes \
  --data "name=admin-route" \
  --data "paths[]=/admin" \
  --data "methods[]=GET" \
  --data "methods[]=POST" \
  --data "methods[]=PUT" \
  --data "methods[]=DELETE" \
  --data "strip_path=false"

echo "Services and Routes created successfully"
```

---

## 3. Rate Limiting Configuration

### 3.1 Kong Rate Limiting Plugin

```bash
#!/bin/bash
# scripts/configure-rate-limiting.sh

KONG_ADMIN="http://localhost:8001"

# Global Rate Limiting (สำหรับทุก endpoint)
curl -s -X POST $KONG_ADMIN/plugins \
  --data "name=rate-limiting" \
  --data "config.minute=1000" \
  --data "config.hour=50000" \
  --data "config.day=500000" \
  --data "config.policy=redis" \
  --data "config.redis_host=redis" \
  --data "config.redis_port=6379" \
  --data "config.redis_timeout=2000" \
  --data "config.hide_client_headers=false" \
  --data "config.fault_tolerant=true"

# Webhook-specific Rate Limiting (เข้มงวดกว่า)
WEBHOOK_ROUTE_ID=$(curl -s $KONG_ADMIN/routes/webhook-route | jq -r '.id')
curl -s -X POST $KONG_ADMIN/routes/$WEBHOOK_ROUTE_ID/plugins \
  --data "name=rate-limiting" \
  --data "config.minute=100" \
  --data "config.hour=5000" \
  --data "config.policy=redis" \
  --data "config.redis_host=redis" \
  --data "config.redis_port=6379"

# Analytics API - ขีดจำกัดต่ำกว่าเพราะ query หนัก
ANALYTICS_ROUTE_ID=$(curl -s $KONG_ADMIN/routes/analytics-route | jq -r '.id')
curl -s -X POST $KONG_ADMIN/routes/$ANALYTICS_ROUTE_ID/plugins \
  --data "name=rate-limiting" \
  --data "config.minute=30" \
  --data "config.hour=500"

echo "Rate limiting configured"
```

### 3.2 Advanced Rate Limiting ด้วย Custom Plugin

```lua
-- kong/plugins/line-rate-limiter/handler.lua
local BasePlugin = require "kong.plugins.base_plugin"
local redis = require "resty.redis"
local cjson = require "cjson"

local LinRateLimiter = BasePlugin:extend()

LinRateLimiter.PRIORITY = 920
LinRateLimiter.VERSION = "1.0.0"

function LinRateLimiter:new()
  LinRateLimiter.super.new(self, "line-rate-limiter")
end

function LinRateLimiter:access(conf)
  LinRateLimiter.super.access(self)
  
  local redis_client = redis:new()
  redis_client:set_timeout(1000)
  
  local ok, err = redis_client:connect(conf.redis_host, conf.redis_port)
  if not ok then
    kong.log.err("Failed to connect to Redis: ", err)
    return -- fail open
  end
  
  -- ดึง tenant ID จาก JWT token
  local tenant_id = self:extract_tenant_id()
  if not tenant_id then
    return kong.response.exit(401, { message = "Missing tenant ID" })
  end
  
  -- ดึง plan สำหรับ tenant นี้
  local plan = self:get_tenant_plan(tenant_id, redis_client)
  local limits = conf.plan_limits[plan] or conf.plan_limits["starter"]
  
  -- Sliding window rate limiting
  local key = string.format("rate:%s:%s:%s", tenant_id, conf.identifier, self:current_window())
  
  local count, err = redis_client:incr(key)
  if not count then
    kong.log.err("Redis INCR failed: ", err)
    return
  end
  
  if count == 1 then
    redis_client:expire(key, conf.window_size)
  end
  
  -- เพิ่ม response headers
  kong.response.set_header("X-RateLimit-Limit-" .. conf.identifier, limits.max)
  kong.response.set_header("X-RateLimit-Remaining-" .. conf.identifier, math.max(0, limits.max - count))
  kong.response.set_header("X-Tenant-Plan", plan)
  
  if count > limits.max then
    local retry_after = self:calculate_retry_after(key, redis_client)
    kong.response.set_header("Retry-After", retry_after)
    return kong.response.exit(429, {
      message = "API rate limit exceeded",
      plan = plan,
      limit = limits.max,
      retry_after = retry_after
    })
  end
  
  redis_client:set_keepalive(10000, 100)
end

function LinRateLimiter:extract_tenant_id()
  local auth_header = kong.request.get_header("Authorization")
  if not auth_header then
    return nil
  end
  
  local token = auth_header:match("Bearer (.+)")
  if not token then
    return nil
  end
  
  -- Decode JWT payload (ไม่ verify เพราะ JWT plugin ทำไว้แล้ว)
  local parts = {}
  for part in token:gmatch("[^%.]+") do
    parts[#parts + 1] = part
  end
  
  if #parts ~= 3 then
    return nil
  end
  
  local payload = cjson.decode(ngx.decode_base64(parts[2]))
  return payload and payload.tenant_id
end

function LinRateLimiter:get_tenant_plan(tenant_id, redis_client)
  local plan_key = string.format("tenant:plan:%s", tenant_id)
  local plan, err = redis_client:get(plan_key)
  
  if not plan or plan == ngx.null then
    return "starter" -- default plan
  end
  
  return plan
end

function LinRateLimiter:current_window()
  return math.floor(ngx.time() / 60) -- 1-minute windows
end

function LinRateLimiter:calculate_retry_after(key, redis_client)
  local ttl = redis_client:ttl(key)
  return ttl > 0 and ttl or 60
end

return LinRateLimiter
```

```lua
-- kong/plugins/line-rate-limiter/schema.lua
return {
  name = "line-rate-limiter",
  fields = {
    { config = {
      type = "record",
      fields = {
        { redis_host = { type = "string", required = true } },
        { redis_port = { type = "integer", default = 6379 } },
        { window_size = { type = "integer", default = 60 } },
        { identifier = { type = "string", default = "minute" } },
        { plan_limits = {
          type = "map",
          keys = { type = "string" },
          values = {
            type = "record",
            fields = {
              { max = { type = "integer", required = true } }
            }
          },
          default = {
            starter = { max = 100 },
            business = { max = 1000 },
            enterprise = { max = 10000 }
          }
        }}
      }
    }}
  }
}
```

---

## 4. Authentication Plugins

### 4.1 JWT Authentication

```bash
#!/bin/bash
# scripts/configure-auth.sh

KONG_ADMIN="http://localhost:8001"

# JWT Authentication สำหรับ API routes
curl -s -X POST $KONG_ADMIN/routes/message-route/plugins \
  --data "name=jwt" \
  --data "config.claims_to_verify[]=exp" \
  --data "config.claims_to_verify[]=nbf" \
  --data "config.key_claim_name=kid" \
  --data "config.secret_is_base64=false" \
  --data "config.run_on_preflight=false" \
  --data "config.maximum_expiration=86400"

# Key Authentication สำหรับ Service-to-Service
curl -s -X POST $KONG_ADMIN/routes/analytics-route/plugins \
  --data "name=key-auth" \
  --data "config.key_names[]=X-API-Key" \
  --data "config.key_names[]=api_key" \
  --data "config.key_in_header=true" \
  --data "config.key_in_query=false" \
  --data "config.key_in_body=false" \
  --data "config.hide_credentials=true"

# Admin route - Basic Auth + IP Restriction
curl -s -X POST $KONG_ADMIN/routes/admin-route/plugins \
  --data "name=basic-auth" \
  --data "config.hide_credentials=true"

curl -s -X POST $KONG_ADMIN/routes/admin-route/plugins \
  --data "name=ip-restriction" \
  --data "config.allow[]=10.0.0.0/8" \
  --data "config.allow[]=172.16.0.0/12" \
  --data "config.allow[]=192.168.0.0/16"

# OAuth 2.0 Plugin
curl -s -X POST $KONG_ADMIN/routes/user-route/plugins \
  --data "name=oauth2" \
  --data "config.scopes[]=read" \
  --data "config.scopes[]=write" \
  --data "config.mandatory_scope=true" \
  --data "config.token_expiration=7200" \
  --data "config.refresh_token_ttl=1209600" \
  --data "config.enable_authorization_code=true" \
  --data "config.enable_client_credentials=true"

echo "Authentication plugins configured"
```

### 4.2 Custom LINE Authentication Plugin

```lua
-- kong/plugins/line-auth/handler.lua
local http = require "resty.http"
local cjson = require "cjson"
local BasePlugin = require "kong.plugins.base_plugin"

local LineAuth = BasePlugin:extend()
LineAuth.PRIORITY = 1010
LineAuth.VERSION = "1.0.0"

function LineAuth:new()
  LineAuth.super.new(self, "line-auth")
end

function LineAuth:access(conf)
  LineAuth.super.access(self)
  
  local path = kong.request.get_path()
  
  -- Skip verification สำหรับ health check
  if path == "/health" then
    return
  end
  
  -- LINE Webhook Signature Verification
  if path:match("^/webhook") then
    return self:verify_line_signature(conf)
  end
  
  -- LINE Login Token Verification
  local auth_header = kong.request.get_header("Authorization")
  if auth_header and auth_header:match("^LINE ") then
    return self:verify_line_token(conf, auth_header)
  end
end

function LineAuth:verify_line_signature(conf)
  local signature = kong.request.get_header("X-Line-Signature")
  if not signature then
    return kong.response.exit(400, { message = "Missing X-Line-Signature header" })
  end
  
  local body, err = kong.request.get_raw_body()
  if not body then
    return kong.response.exit(400, { message = "Cannot read request body: " .. (err or "unknown") })
  end
  
  local computed = ngx.encode_base64(
    ngx.hmac_sha1(conf.channel_secret, body)
  )
  
  if computed ~= signature then
    kong.log.warn("Invalid LINE signature from IP: ", kong.client.get_forwarded_ip())
    return kong.response.exit(401, { message = "Invalid signature" })
  end
end

function LineAuth:verify_line_token(conf, auth_header)
  local token = auth_header:sub(6) -- Remove "LINE " prefix
  
  -- ตรวจสอบกับ LINE API
  local httpc = http.new()
  httpc:set_timeout(5000)
  
  local res, err = httpc:request_uri("https://api.line.me/oauth2/v2.1/verify", {
    method = "POST",
    body = string.format("access_token=%s&client_id=%s", token, conf.channel_id),
    headers = {
      ["Content-Type"] = "application/x-www-form-urlencoded"
    }
  })
  
  if not res then
    kong.log.err("LINE API verification failed: ", err)
    return kong.response.exit(401, { message = "Token verification failed" })
  end
  
  local data = cjson.decode(res.body)
  
  if res.status ~= 200 or data.error then
    return kong.response.exit(401, { 
      message = "Invalid LINE token",
      error = data.error_description
    })
  end
  
  -- ตรวจสอบว่า token เป็นของ channel นี้
  if tostring(data.client_id) ~= conf.channel_id then
    return kong.response.exit(401, { message = "Token not issued for this channel" })
  end
  
  -- ส่ง user info ไปยัง upstream service
  kong.service.request.set_header("X-Line-User-Id", data.sub or "")
  kong.service.request.set_header("X-Token-Expires-In", tostring(data.expires_in or 0))
end

return LineAuth
```

---

## 5. Load Balancing

### 5.1 Kong Upstream Configuration

```bash
#!/bin/bash
# scripts/configure-load-balancing.sh

KONG_ADMIN="http://localhost:8001"

# Webhook Service - Round Robin
curl -s -X POST $KONG_ADMIN/upstreams \
  --data "name=webhook-upstream" \
  --data "algorithm=round-robin" \
  --data "hash_on=none" \
  --data "slots=10000" \
  --data "healthchecks.active.healthy.interval=10" \
  --data "healthchecks.active.unhealthy.interval=5" \
  --data "healthchecks.active.http_path=/health" \
  --data "healthchecks.active.healthy.successes=2" \
  --data "healthchecks.active.unhealthy.http_failures=3"

# เพิ่ม Upstream Targets
for i in 1 2 3; do
  curl -s -X POST $KONG_ADMIN/upstreams/webhook-upstream/targets \
    --data "target=webhook-service-${i}:3001" \
    --data "weight=100"
done

# Message Service - Least Connections
curl -s -X POST $KONG_ADMIN/upstreams \
  --data "name=message-upstream" \
  --data "algorithm=least-connections" \
  --data "healthchecks.active.http_path=/health"

for i in 1 2; do
  curl -s -X POST $KONG_ADMIN/upstreams/message-upstream/targets \
    --data "target=message-service-${i}:3002" \
    --data "weight=100"
done

# User Service - Consistent Hashing by User ID
curl -s -X POST $KONG_ADMIN/upstreams \
  --data "name=user-upstream" \
  --data "algorithm=consistent-hashing" \
  --data "hash_on=header" \
  --data "hash_on_header=X-Line-User-Id"

curl -s -X POST $KONG_ADMIN/upstreams/user-upstream/targets \
  --data "target=user-service-1:3003" \
  --data "weight=100"
curl -s -X POST $KONG_ADMIN/upstreams/user-upstream/targets \
  --data "target=user-service-2:3003" \
  --data "weight=100"

# Update services to use upstreams
curl -s -X PATCH $KONG_ADMIN/services/webhook-service \
  --data "host=webhook-upstream"

curl -s -X PATCH $KONG_ADMIN/services/message-service \
  --data "host=message-upstream"

curl -s -X PATCH $KONG_ADMIN/services/user-service \
  --data "host=user-upstream"

echo "Load balancing configured"
```

---

## 6. Circuit Breaker

### 6.1 Kong Circuit Breaker Plugin

```bash
#!/bin/bash
KONG_ADMIN="http://localhost:8001"

# Circuit Breaker สำหรับ Message Service
curl -s -X POST $KONG_ADMIN/services/message-service/plugins \
  --data "name=request-termination" \
  --data "config.status_code=503" \
  --data "config.message=Service temporarily unavailable"

# ใช้ Kong's Health Checks เป็น Circuit Breaker
curl -s -X PATCH $KONG_ADMIN/upstreams/message-upstream \
  --data "healthchecks.passive.healthy.successes=5" \
  --data "healthchecks.passive.unhealthy.http_failures=5" \
  --data "healthchecks.passive.unhealthy.tcp_failures=3" \
  --data "healthchecks.passive.unhealthy.timeouts=3" \
  --data "healthchecks.active.healthy.interval=30" \
  --data "healthchecks.active.unhealthy.interval=5" \
  --data "healthchecks.active.http_path=/health" \
  --data "healthchecks.active.timeout=5"
```

### 6.2 Custom Circuit Breaker ด้วย Node.js

```typescript
// gateway/CircuitBreaker.ts
import { EventEmitter } from 'events';
import { Redis } from 'ioredis';

type CircuitState = 'CLOSED' | 'OPEN' | 'HALF_OPEN';

interface CircuitBreakerConfig {
  name: string;
  failureThreshold: number;      // จำนวน failure ก่อน open
  successThreshold: number;      // จำนวน success ก่อน close จาก half-open
  timeout: number;               // ms รอก่อน half-open
  volumeThreshold: number;       // จำนวน request ขั้นต่ำ
  errorPercentageThreshold: number; // % error ก่อน open
}

export class CircuitBreaker extends EventEmitter {
  private redis: Redis;
  private config: CircuitBreakerConfig;

  constructor(config: CircuitBreakerConfig) {
    super();
    this.config = config;
    this.redis = new Redis(process.env.REDIS_URL!);
  }

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    const state = await this.getState();

    if (state === 'OPEN') {
      // ตรวจสอบว่าถึงเวลา half-open หรือยัง
      const canAttempt = await this.shouldAttemptReset();
      if (!canAttempt) {
        throw new Error(`Circuit breaker ${this.config.name} is OPEN`);
      }
      await this.setState('HALF_OPEN');
    }

    try {
      const result = await fn();
      await this.recordSuccess();
      return result;
    } catch (error) {
      await this.recordFailure();
      throw error;
    }
  }

  private async getState(): Promise<CircuitState> {
    const state = await this.redis.get(`circuit:${this.config.name}:state`);
    return (state as CircuitState) || 'CLOSED';
  }

  private async setState(state: CircuitState): Promise<void> {
    await this.redis.set(`circuit:${this.config.name}:state`, state);
    this.emit('stateChange', { name: this.config.name, state });
  }

  private async recordSuccess(): Promise<void> {
    const state = await this.getState();
    
    if (state === 'HALF_OPEN') {
      const successKey = `circuit:${this.config.name}:half-open-success`;
      const count = await this.redis.incr(successKey);
      
      if (count >= this.config.successThreshold) {
        await this.reset();
      }
    }
    
    // บันทึก success
    const window = Math.floor(Date.now() / 1000);
    await this.redis.zadd(`circuit:${this.config.name}:successes`, window, `${window}-${Math.random()}`);
    await this.redis.zremrangebyscore(`circuit:${this.config.name}:successes`, 0, window - 60);
  }

  private async recordFailure(): Promise<void> {
    const window = Math.floor(Date.now() / 1000);
    const failureKey = `circuit:${this.config.name}:failures`;
    
    await this.redis.zadd(failureKey, window, `${window}-${Math.random()}`);
    await this.redis.zremrangebyscore(failureKey, 0, window - 60);
    
    // ตรวจสอบว่าควร open circuit หรือไม่
    await this.checkAndTrip();
  }

  private async checkAndTrip(): Promise<void> {
    const state = await this.getState();
    if (state === 'OPEN') return;

    const window = Math.floor(Date.now() / 1000) - 60;
    
    const [failures, successes] = await Promise.all([
      this.redis.zcount(`circuit:${this.config.name}:failures`, window, '+inf'),
      this.redis.zcount(`circuit:${this.config.name}:successes`, window, '+inf')
    ]);

    const total = failures + successes;
    
    if (total < this.config.volumeThreshold) return;
    
    const errorPercentage = (failures / total) * 100;
    
    if (failures >= this.config.failureThreshold || 
        errorPercentage >= this.config.errorPercentageThreshold) {
      await this.trip();
    }
  }

  private async trip(): Promise<void> {
    await this.setState('OPEN');
    const openedAt = Date.now();
    await this.redis.set(`circuit:${this.config.name}:opened-at`, openedAt);
    
    this.emit('tripped', { name: this.config.name, openedAt });
    console.warn(`Circuit breaker ${this.config.name} tripped to OPEN`);
  }

  private async reset(): Promise<void> {
    await this.setState('CLOSED');
    await this.redis.del(
      `circuit:${this.config.name}:failures`,
      `circuit:${this.config.name}:half-open-success`,
      `circuit:${this.config.name}:opened-at`
    );
    
    this.emit('reset', { name: this.config.name });
    console.info(`Circuit breaker ${this.config.name} reset to CLOSED`);
  }

  private async shouldAttemptReset(): Promise<boolean> {
    const openedAt = await this.redis.get(`circuit:${this.config.name}:opened-at`);
    if (!openedAt) return true;
    
    const elapsed = Date.now() - parseInt(openedAt);
    return elapsed >= this.config.timeout;
  }

  async getMetrics(): Promise<{
    state: CircuitState;
    failures: number;
    successes: number;
    errorRate: number;
  }> {
    const window = Math.floor(Date.now() / 1000) - 60;
    
    const [state, failures, successes] = await Promise.all([
      this.getState(),
      this.redis.zcount(`circuit:${this.config.name}:failures`, window, '+inf'),
      this.redis.zcount(`circuit:${this.config.name}:successes`, window, '+inf')
    ]);

    const total = failures + successes;
    const errorRate = total > 0 ? (failures / total) * 100 : 0;

    return { state, failures, successes, errorRate };
  }
}

// Usage
const messageServiceBreaker = new CircuitBreaker({
  name: 'message-service',
  failureThreshold: 5,
  successThreshold: 2,
  timeout: 30000,        // 30 วินาที
  volumeThreshold: 10,
  errorPercentageThreshold: 50
});
```

---

## 7. Request/Response Transformation

### 7.1 Kong Transformation Plugin

```bash
#!/bin/bash
KONG_ADMIN="http://localhost:8001"

# Request Transformer - เพิ่ม headers
curl -s -X POST $KONG_ADMIN/services/message-service/plugins \
  --data "name=request-transformer" \
  --data "config.add.headers[]=X-Gateway-Version:1.0" \
  --data "config.add.headers[]=X-Request-Source:kong" \
  --data "config.remove.headers[]=X-Forwarded-For" \
  --data "config.rename.headers[]=Authorization:X-Original-Auth"

# Response Transformer - เพิ่ม/ลบ headers
curl -s -X POST $KONG_ADMIN/services/message-service/plugins \
  --data "name=response-transformer" \
  --data "config.add.headers[]=X-Content-Type-Options:nosniff" \
  --data "config.add.headers[]=X-Frame-Options:DENY" \
  --data "config.remove.headers[]=Server" \
  --data "config.remove.headers[]=X-Powered-By"

echo "Transformation plugins configured"
```

### 7.2 Custom Request Transformer ด้วย Node.js

```typescript
// gateway/RequestTransformer.ts
import { Request, Response, NextFunction } from 'express';

interface TransformRule {
  path?: RegExp;
  method?: string;
  requestTransformers?: RequestTransformer[];
  responseTransformers?: ResponseTransformer[];
}

interface RequestTransformer {
  type: 'add_header' | 'remove_header' | 'rename_header' | 
        'add_query' | 'remove_query' | 'modify_body';
  key: string;
  value?: string;
  newKey?: string;
}

interface ResponseTransformer {
  type: 'add_header' | 'remove_header' | 'filter_body_fields';
  key: string;
  value?: string;
  allowedFields?: string[];
}

export class APIGatewayTransformer {
  private rules: TransformRule[] = [
    {
      // Webhook endpoint
      path: /^\/webhook/,
      method: 'POST',
      requestTransformers: [
        { type: 'add_header', key: 'X-Webhook-Source', value: 'line' },
        { type: 'add_header', key: 'X-Processing-Time', value: '{{timestamp}}' }
      ]
    },
    {
      // API endpoints - ลบ sensitive headers
      path: /^\/api\//,
      requestTransformers: [
        { type: 'add_header', key: 'X-Tenant-Id', value: '{{tenant_id}}' },
        { type: 'add_header', key: 'X-Request-Id', value: '{{request_id}}' },
        { type: 'remove_header', key: 'X-Forwarded-For' } // จัดการ IP เอง
      ],
      responseTransformers: [
        { type: 'remove_header', key: 'X-Powered-By' },
        { type: 'remove_header', key: 'Server' },
        { type: 'add_header', key: 'X-Response-Time', value: '{{elapsed_ms}}ms' }
      ]
    },
    {
      // User data - filter PII in response
      path: /^\/api\/v1\/users/,
      responseTransformers: [
        {
          type: 'filter_body_fields',
          key: 'data',
          allowedFields: ['id', 'displayName', 'pictureUrl', 'createdAt']
        }
      ]
    }
  ];

  middleware() {
    return async (req: Request, res: Response, next: NextFunction) => {
      const startTime = Date.now();
      const requestId = req.headers['x-request-id'] as string;
      const tenantId = (req as any).tenantId;

      // หา matching rule
      const rule = this.findMatchingRule(req);
      
      if (rule?.requestTransformers) {
        this.applyRequestTransformers(req, rule.requestTransformers, {
          timestamp: new Date().toISOString(),
          request_id: requestId,
          tenant_id: tenantId
        });
      }

      if (rule?.responseTransformers) {
        const originalJson = res.json.bind(res);
        res.json = (body: any) => {
          // Apply response transformers
          const context = {
            elapsed_ms: Date.now() - startTime
          };

          for (const transformer of rule.responseTransformers!) {
            switch (transformer.type) {
              case 'add_header':
                res.setHeader(transformer.key, this.resolveTemplate(transformer.value!, context));
                break;
              case 'remove_header':
                res.removeHeader(transformer.key);
                break;
              case 'filter_body_fields':
                if (transformer.allowedFields && body[transformer.key]) {
                  body[transformer.key] = this.filterFields(
                    body[transformer.key],
                    transformer.allowedFields
                  );
                }
                break;
            }
          }

          return originalJson(body);
        };
      }

      next();
    };
  }

  private applyRequestTransformers(
    req: Request,
    transformers: RequestTransformer[],
    context: Record<string, string>
  ): void {
    for (const transformer of transformers) {
      switch (transformer.type) {
        case 'add_header':
          req.headers[transformer.key.toLowerCase()] = 
            this.resolveTemplate(transformer.value!, context);
          break;
        case 'remove_header':
          delete req.headers[transformer.key.toLowerCase()];
          break;
        case 'rename_header':
          const value = req.headers[transformer.key.toLowerCase()];
          if (value && transformer.newKey) {
            req.headers[transformer.newKey.toLowerCase()] = value;
            delete req.headers[transformer.key.toLowerCase()];
          }
          break;
      }
    }
  }

  private filterFields(data: any, allowedFields: string[]): any {
    if (Array.isArray(data)) {
      return data.map(item => this.filterFields(item, allowedFields));
    }
    
    if (typeof data === 'object' && data !== null) {
      return Object.fromEntries(
        Object.entries(data).filter(([key]) => allowedFields.includes(key))
      );
    }
    
    return data;
  }

  private findMatchingRule(req: Request): TransformRule | undefined {
    return this.rules.find(rule => {
      const pathMatch = !rule.path || rule.path.test(req.path);
      const methodMatch = !rule.method || rule.method === req.method;
      return pathMatch && methodMatch;
    });
  }

  private resolveTemplate(template: string, context: Record<string, string>): string {
    return template.replace(/\{\{(\w+)\}\}/g, (_, key) => context[key] || '');
  }
}
```

---

## 8. Logging and Monitoring

### 8.1 Kong Logging Configuration

```bash
#!/bin/bash
KONG_ADMIN="http://localhost:8001"

# File Log
curl -s -X POST $KONG_ADMIN/plugins \
  --data "name=file-log" \
  --data "config.path=/var/log/kong/access.log" \
  --data "config.reopen=true"

# HTTP Log - ส่งไป Elasticsearch
curl -s -X POST $KONG_ADMIN/plugins \
  --data "name=http-log" \
  --data "config.http_endpoint=http://logstash:5000" \
  --data "config.method=POST" \
  --data "config.timeout=10000" \
  --data "config.keepalive=60000" \
  --data "config.flush_timeout=2" \
  --data "config.retry_count=10" \
  --data "config.queue_size=1000"

# Prometheus Metrics
curl -s -X POST $KONG_ADMIN/plugins \
  --data "name=prometheus" \
  --data "config.per_consumer=true" \
  --data "config.bandwidth_metrics=true" \
  --data "config.upstream_health_metrics=true"

# StatsD สำหรับ custom metrics
curl -s -X POST $KONG_ADMIN/plugins \
  --data "name=statsd" \
  --data "config.host=statsd" \
  --data "config.port=8125" \
  --data "config.prefix=kong" \
  --data "config.metrics[]=status_count" \
  --data "config.metrics[]=latency" \
  --data "config.metrics[]=bandwidth" \
  --data "config.metrics[]=upstream_latency" \
  --data "config.consumer_identifier=username"

echo "Logging configured"
```

### 8.2 Prometheus Configuration

```yaml
# prometheus/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'kong'
    static_configs:
      - targets: ['kong:8001']
    metrics_path: '/metrics'

  - job_name: 'webhook-service'
    static_configs:
      - targets: ['webhook-service:3001']
    metrics_path: '/metrics'

  - job_name: 'message-service'
    static_configs:
      - targets: ['message-service:3002']
    metrics_path: '/metrics'

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - 'alerts/*.yml'
```

```yaml
# prometheus/alerts/kong-alerts.yml
groups:
  - name: kong_alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(kong_http_requests_total{status=~"5.."}[5m])) 
          / sum(rate(kong_http_requests_total[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.95, rate(kong_latency_bucket[5m])) > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High API latency detected"
          description: "P95 latency is {{ $value }}ms"

      - alert: RateLimitHigh
        expr: |
          sum(rate(kong_http_requests_total{status="429"}[5m])) > 10
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High rate limit hits"
          description: "Rate limit hits: {{ $value }} per second"
```

---

## 9. API Versioning

### 9.1 API Versioning Strategy

```typescript
// versioning/ApiVersionRouter.ts
import express, { Router, Request, Response, NextFunction } from 'express';

interface VersionConfig {
  version: string;
  deprecated?: boolean;
  deprecationDate?: Date;
  sunsetDate?: Date;
  router: Router;
}

export class ApiVersionRouter {
  private versions: Map<string, VersionConfig> = new Map();
  private defaultVersion: string = 'v1';

  registerVersion(config: VersionConfig): void {
    this.versions.set(config.version, config);
  }

  createRouter(): Router {
    const router = Router();

    // Middleware สำหรับตรวจสอบ version
    router.use((req: Request, res: Response, next: NextFunction) => {
      const requestedVersion = this.extractVersion(req);
      
      if (!this.versions.has(requestedVersion)) {
        res.status(400).json({
          error: 'API_VERSION_NOT_FOUND',
          message: `API version ${requestedVersion} is not supported`,
          supportedVersions: Array.from(this.versions.keys()),
          currentVersion: this.defaultVersion
        });
        return;
      }

      const versionConfig = this.versions.get(requestedVersion)!;
      
      // เพิ่ม deprecation headers
      if (versionConfig.deprecated) {
        res.setHeader('Deprecation', versionConfig.deprecationDate?.toUTCString() || 'true');
        
        if (versionConfig.sunsetDate) {
          res.setHeader('Sunset', versionConfig.sunsetDate.toUTCString());
          res.setHeader('Link', `</api/${this.defaultVersion}>; rel="successor-version"`);
        }
      }

      res.setHeader('X-API-Version', requestedVersion);
      (req as any).apiVersion = requestedVersion;
      
      next();
    });

    // Mount แต่ละ version router
    for (const [version, config] of this.versions.entries()) {
      router.use(`/${version}`, config.router);
    }

    // Default version redirect
    router.use('/', (req, res) => {
      res.redirect(301, `/api/${this.defaultVersion}${req.path}`);
    });

    return router;
  }

  private extractVersion(req: Request): string {
    // Priority: URL path > Header > Query param
    const pathMatch = req.path.match(/^\/v(\d+)/);
    if (pathMatch) return `v${pathMatch[1]}`;
    
    const headerVersion = req.headers['api-version'] as string;
    if (headerVersion) return headerVersion;
    
    const queryVersion = req.query.api_version as string;
    if (queryVersion) return queryVersion;
    
    return this.defaultVersion;
  }
}

// Setup versioning
const versionRouter = new ApiVersionRouter();

// V1 Router
const v1Router = Router();
v1Router.get('/messages', (req, res) => {
  res.json({ version: 'v1', messages: [] });
});

// V2 Router (เพิ่ม cursor-based pagination)
const v2Router = Router();
v2Router.get('/messages', (req, res) => {
  res.json({ 
    version: 'v2', 
    messages: [],
    pagination: { cursor: null, hasNext: false }
  });
});

versionRouter.registerVersion({
  version: 'v1',
  deprecated: true,
  deprecationDate: new Date('2024-01-01'),
  sunsetDate: new Date('2024-06-01'),
  router: v1Router
});

versionRouter.registerVersion({
  version: 'v2',
  router: v2Router
});
```

---

## 10. Developer Portal

### 10.1 OpenAPI/Swagger Documentation

```typescript
// docs/SwaggerConfig.ts
import swaggerJsdoc from 'swagger-jsdoc';
import swaggerUi from 'swagger-ui-express';
import { Express } from 'express';

export function setupDeveloperPortal(app: Express): void {
  const options = {
    definition: {
      openapi: '3.0.0',
      info: {
        title: 'LINE Bot Platform API',
        version: '2.0.0',
        description: `
# LINE Bot Platform API Documentation

API สำหรับจัดการ LINE Bot ของคุณ

## Authentication
API รองรับ 3 วิธีการ authentication:
1. **JWT Bearer Token** - สำหรับ user authentication ผ่าน LINE Login
2. **API Key** - สำหรับ server-to-server calls
3. **LINE Webhook Signature** - สำหรับ webhook endpoints

## Rate Limiting
- Starter: 100 req/min
- Business: 1,000 req/min
- Enterprise: 10,000 req/min
        `,
        contact: {
          name: 'API Support',
          email: 'api@linebot-platform.com',
          url: 'https://docs.linebot-platform.com'
        },
        license: {
          name: 'Commercial License',
          url: 'https://linebot-platform.com/license'
        }
      },
      servers: [
        {
          url: 'https://api.linebot-platform.com',
          description: 'Production'
        },
        {
          url: 'https://staging-api.linebot-platform.com',
          description: 'Staging'
        }
      ],
      components: {
        securitySchemes: {
          BearerAuth: {
            type: 'http',
            scheme: 'bearer',
            bearerFormat: 'JWT'
          },
          ApiKeyAuth: {
            type: 'apiKey',
            in: 'header',
            name: 'X-API-Key'
          }
        },
        schemas: {
          Message: {
            type: 'object',
            properties: {
              id: { type: 'string', format: 'uuid' },
              type: { 
                type: 'string',
                enum: ['text', 'image', 'video', 'audio', 'flex', 'sticker']
              },
              content: { type: 'object' },
              userId: { type: 'string' },
              timestamp: { type: 'string', format: 'date-time' }
            }
          },
          User: {
            type: 'object',
            properties: {
              id: { type: 'string', format: 'uuid' },
              lineUserId: { type: 'string' },
              displayName: { type: 'string' },
              pictureUrl: { type: 'string', format: 'uri' },
              createdAt: { type: 'string', format: 'date-time' }
            }
          },
          Error: {
            type: 'object',
            properties: {
              error: { type: 'string' },
              message: { type: 'string' },
              requestId: { type: 'string' }
            }
          }
        }
      }
    },
    apis: ['./src/routes/*.ts']
  };

  const spec = swaggerJsdoc(options);

  app.use('/api/docs', swaggerUi.serve, swaggerUi.setup(spec, {
    customCss: '.swagger-ui .topbar { background-color: #00B900; }',
    customSiteTitle: 'LINE Bot Platform API Docs',
    customfavIcon: '/favicon.ico',
    swaggerOptions: {
      docExpansion: 'list',
      filter: true,
      showRequestDuration: true,
      tryItOutEnabled: true,
      persistAuthorization: true
    }
  }));

  // Serve raw spec
  app.get('/api/openapi.json', (req, res) => {
    res.json(spec);
  });
}
```

---

## 11. Nginx เป็น Alternative API Gateway

### 11.1 Nginx Configuration

```nginx
# nginx/nginx.conf
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    multi_accept on;
    use epoll;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging format
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for" '
                    'rt=$request_time uct="$upstream_connect_time" '
                    'uht="$upstream_header_time" urt="$upstream_response_time" '
                    'rid="$request_id" tenant="$tenant_id"';

    log_format json_combined escape=json
        '{'
            '"time_local":"$time_local",'
            '"remote_addr":"$remote_addr",'
            '"request":"$request",'
            '"status": "$status",'
            '"body_bytes_sent":"$body_bytes_sent",'
            '"request_time":"$request_time",'
            '"upstream_response_time":"$upstream_response_time",'
            '"request_id":"$request_id",'
            '"tenant_id":"$tenant_id"'
        '}';

    access_log /var/log/nginx/access.log json_combined;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 1000;
    types_hash_max_size 2048;
    server_tokens off;

    # Gzip
    gzip on;
    gzip_comp_level 6;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml application/xml+rss text/javascript;

    # Rate Limiting zones
    limit_req_zone $binary_remote_addr zone=webhook:10m rate=10r/s;
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/s;
    limit_req_zone $tenant_id zone=per_tenant:20m rate=50r/s;

    # Connection limiting
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    # Tenant ID extraction
    map $http_authorization $tenant_id {
        default "unknown";
        ~^Bearer\s+(.+)$ $1; # Simplified - ควรใช้ Lua module สำหรับ JWT parsing
    }

    # Upstream definitions
    upstream webhook_backend {
        least_conn;
        server webhook-service-1:3001 weight=1 max_fails=3 fail_timeout=30s;
        server webhook-service-2:3001 weight=1 max_fails=3 fail_timeout=30s;
        server webhook-service-3:3001 weight=1 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    upstream api_backend {
        least_conn;
        server message-service-1:3002 weight=2 max_fails=3 fail_timeout=30s;
        server message-service-2:3002 weight=2 max_fails=3 fail_timeout=30s;
        server user-service-1:3003 weight=1 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    upstream analytics_backend {
        server analytics-service-1:3004 max_fails=2 fail_timeout=20s;
        server analytics-service-2:3004 backup;
        keepalive 8;
    }

    # Cache configuration
    proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:10m 
                     max_size=1g inactive=60m use_temp_path=off;

    include /etc/nginx/conf.d/*.conf;
}
```

```nginx
# nginx/conf.d/linebot.conf

server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.linebot-platform.com;

    ssl_certificate /etc/nginx/ssl/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
    ssl_prefer_server_ciphers on;

    # Request ID
    set $request_id $http_x_request_id;
    if ($request_id = "") {
        set $request_id $request_id;
    }
    add_header X-Request-ID $request_id always;

    # Security headers
    add_header X-Frame-Options DENY always;
    add_header X-Content-Type-Options nosniff always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    # Webhook endpoint
    location /webhook {
        limit_req zone=webhook burst=20 nodelay;
        limit_conn conn_limit 100;

        # LINE Signature verification (lua script)
        # access_by_lua_file /etc/nginx/lua/verify_line_signature.lua;

        proxy_pass http://webhook_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Request-ID $request_id;

        proxy_read_timeout 60s;
        proxy_send_timeout 60s;

        # Buffer webhook body
        client_body_buffer_size 1m;
        client_max_body_size 5m;
    }

    # API endpoints
    location /api/ {
        limit_req zone=api burst=50 nodelay;
        limit_req zone=per_tenant burst=10 nodelay;

        # JWT verification
        # access_by_lua_file /etc/nginx/lua/verify_jwt.lua;

        proxy_pass http://api_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Request-ID $request_id;
        proxy_set_header X-Tenant-ID $tenant_id;

        # Cache GET requests
        proxy_cache api_cache;
        proxy_cache_methods GET;
        proxy_cache_valid 200 5m;
        proxy_cache_valid 404 1m;
        proxy_cache_key "$request_method:$host$uri$is_args$args:$tenant_id";
        proxy_cache_use_stale error timeout updating;
        add_header X-Cache-Status $upstream_cache_status;

        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }

    # Analytics endpoints (cache longer)
    location /api/v1/analytics {
        limit_req zone=api burst=5 nodelay;

        proxy_pass http://analytics_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Request-ID $request_id;
        proxy_set_header X-Tenant-ID $tenant_id;

        # Cache analytics data longer
        proxy_cache api_cache;
        proxy_cache_valid 200 15m;
        proxy_cache_key "$request_method:$uri:$args:$tenant_id";

        proxy_read_timeout 60s;
    }

    # Health check (no auth, no rate limit)
    location /health {
        access_log off;
        proxy_pass http://api_backend/health;
        proxy_connect_timeout 2s;
        proxy_read_timeout 5s;
    }

    # Error pages
    error_page 429 /rate_limited.json;
    error_page 502 503 504 /service_unavailable.json;

    location /rate_limited.json {
        internal;
        default_type application/json;
        return 429 '{"error":"TOO_MANY_REQUESTS","message":"Rate limit exceeded. Please try again later."}';
    }

    location /service_unavailable.json {
        internal;
        default_type application/json;
        return 503 '{"error":"SERVICE_UNAVAILABLE","message":"Service is temporarily unavailable."}';
    }
}
```

---

## สรุป

API Gateway สำหรับ LINE Bot system ต้องรองรับ:

1. **Rate Limiting** - ป้องกัน abuse และจัดการ load ตาม tenant plan
2. **Authentication** - JWT, API Key, LINE Signature Verification
3. **Load Balancing** - กระจาย traffic อย่างชาญฉลาด
4. **Circuit Breaker** - ป้องกัน cascade failures
5. **Request/Response Transform** - จัดการ headers และ body
6. **Logging & Monitoring** - track ทุก request สำหรับ debugging
7. **API Versioning** - backward compatibility
8. **Developer Portal** - documentation ที่สมบูรณ์

Kong Gateway เหมาะสำหรับ enterprise deployment ที่ต้องการ plugin ecosystem ที่ครบครัน ในขณะที่ Nginx เหมาะสำหรับ simple deployment ที่ต้องการ performance สูง
