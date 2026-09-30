# Part 91: Advanced Security Architecture สำหรับระบบ LINE Bot

## บทนำ

ในยุคที่ภัยคุกคามทางไซเบอร์มีความซับซ้อนมากขึ้นทุกวัน การออกแบบระบบ LINE Bot ที่มีความปลอดภัยสูงจึงเป็นสิ่งสำคัญอย่างยิ่ง บทนี้จะครอบคลุมการออกแบบสถาปัตยกรรมความปลอดภัยแบบ Zero-Trust สำหรับระบบ LINE Official Account โดยใช้หลักการ Defense in Depth และ Principle of Least Privilege

---

## 1. Zero-Trust Architecture สำหรับ LINE Bot

### 1.1 หลักการ Zero-Trust

```
┌─────────────────────────────────────────────────────────────────┐
│                    ZERO-TRUST ARCHITECTURE                       │
│                                                                   │
│  ┌──────────┐    ┌──────────────┐    ┌─────────────────────┐   │
│  │  LINE    │───▶│   API        │───▶│   Identity &        │   │
│  │  Platform│    │   Gateway    │    │   Access Mgmt       │   │
│  └──────────┘    └──────────────┘    └─────────────────────┘   │
│                         │                        │               │
│                         ▼                        ▼               │
│                  ┌──────────────┐    ┌─────────────────────┐   │
│                  │  Policy      │    │   Service Mesh      │   │
│                  │  Engine      │    │   (mTLS)            │   │
│                  └──────────────┘    └─────────────────────┘   │
│                         │                        │               │
│                         ▼                        ▼               │
│                  ┌──────────────────────────────────────────┐   │
│                  │           Microservices                   │   │
│                  │  ┌────────┐ ┌────────┐ ┌────────────┐   │   │
│                  │  │Message │ │ User   │ │  Analytics │   │   │
│                  │  │Service │ │Service │ │  Service   │   │   │
│                  │  └────────┘ └────────┘ └────────────┘   │   │
│                  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 การ Implement Zero-Trust ด้วย Node.js

```typescript
// zero-trust/middleware/ZeroTrustMiddleware.ts
import { Request, Response, NextFunction } from 'express';
import { IdentityVerifier } from './IdentityVerifier';
import { PolicyEngine } from './PolicyEngine';
import { TrustScoreCalculator } from './TrustScoreCalculator';
import { AuditLogger } from '../logging/AuditLogger';

interface ZeroTrustContext {
  requestId: string;
  timestamp: Date;
  sourceIP: string;
  userAgent: string;
  identityToken?: string;
  trustScore: number;
  permissions: string[];
  riskLevel: 'LOW' | 'MEDIUM' | 'HIGH' | 'CRITICAL';
}

export class ZeroTrustMiddleware {
  private identityVerifier: IdentityVerifier;
  private policyEngine: PolicyEngine;
  private trustCalculator: TrustScoreCalculator;
  private auditLogger: AuditLogger;

  constructor() {
    this.identityVerifier = new IdentityVerifier();
    this.policyEngine = new PolicyEngine();
    this.trustCalculator = new TrustScoreCalculator();
    this.auditLogger = new AuditLogger();
  }

  async verify(req: Request, res: Response, next: NextFunction): Promise<void> {
    const context = await this.buildContext(req);
    
    try {
      // 1. Verify Identity
      const identity = await this.identityVerifier.verify(req.headers.authorization);
      if (!identity.isValid) {
        await this.auditLogger.logAccessDenied(context, 'INVALID_IDENTITY');
        res.status(401).json({
          error: 'Unauthorized',
          code: 'INVALID_IDENTITY',
          requestId: context.requestId
        });
        return;
      }

      // 2. Calculate Trust Score
      context.trustScore = await this.trustCalculator.calculate({
        identity,
        sourceIP: context.sourceIP,
        userAgent: context.userAgent,
        requestHistory: await this.getRequestHistory(identity.userId),
        deviceFingerprint: req.headers['x-device-fingerprint'] as string
      });

      // 3. Evaluate Policy
      const policyResult = await this.policyEngine.evaluate({
        identity,
        resource: req.path,
        action: req.method,
        context,
        trustScore: context.trustScore
      });

      if (!policyResult.allowed) {
        await this.auditLogger.logAccessDenied(context, policyResult.reason);
        res.status(403).json({
          error: 'Forbidden',
          code: policyResult.reason,
          requestId: context.requestId
        });
        return;
      }

      // 4. Attach context to request
      (req as any).zeroTrustContext = context;
      (req as any).identity = identity;
      
      await this.auditLogger.logAccessGranted(context);
      next();

    } catch (error) {
      await this.auditLogger.logError(context, error as Error);
      res.status(500).json({
        error: 'Internal Server Error',
        requestId: context.requestId
      });
    }
  }

  private async buildContext(req: Request): Promise<ZeroTrustContext> {
    return {
      requestId: this.generateRequestId(),
      timestamp: new Date(),
      sourceIP: this.extractRealIP(req),
      userAgent: req.headers['user-agent'] || 'unknown',
      identityToken: req.headers.authorization,
      trustScore: 0,
      permissions: [],
      riskLevel: 'LOW'
    };
  }

  private extractRealIP(req: Request): string {
    return (
      (req.headers['x-forwarded-for'] as string)?.split(',')[0].trim() ||
      (req.headers['x-real-ip'] as string) ||
      req.socket.remoteAddress ||
      'unknown'
    );
  }

  private generateRequestId(): string {
    return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  private async getRequestHistory(userId: string): Promise<any[]> {
    // ดึงประวัติ request จาก Redis
    return [];
  }
}
```

### 1.3 Trust Score Calculator

```typescript
// zero-trust/TrustScoreCalculator.ts
import { Redis } from 'ioredis';
import { GeoIPLookup } from './GeoIPLookup';
import { DeviceFingerprinter } from './DeviceFingerprinter';

interface TrustFactors {
  identity: any;
  sourceIP: string;
  userAgent: string;
  requestHistory: any[];
  deviceFingerprint?: string;
}

export class TrustScoreCalculator {
  private redis: Redis;
  private geoIP: GeoIPLookup;
  private deviceFingerprinter: DeviceFingerprinter;

  // น้ำหนักแต่ละปัจจัย (รวมกัน = 100)
  private readonly WEIGHTS = {
    identityStrength: 30,
    locationRisk: 20,
    deviceTrust: 25,
    behaviorPattern: 15,
    timeOfDay: 10
  };

  constructor() {
    this.redis = new Redis(process.env.REDIS_URL!);
    this.geoIP = new GeoIPLookup();
    this.deviceFingerprinter = new DeviceFingerprinter();
  }

  async calculate(factors: TrustFactors): Promise<number> {
    let totalScore = 0;

    // 1. Identity Strength Score
    const identityScore = this.calculateIdentityScore(factors.identity);
    totalScore += identityScore * (this.WEIGHTS.identityStrength / 100);

    // 2. Location Risk Score
    const locationScore = await this.calculateLocationScore(factors.sourceIP);
    totalScore += locationScore * (this.WEIGHTS.locationRisk / 100);

    // 3. Device Trust Score
    const deviceScore = await this.calculateDeviceScore(
      factors.deviceFingerprint,
      factors.identity.userId
    );
    totalScore += deviceScore * (this.WEIGHTS.deviceTrust / 100);

    // 4. Behavior Pattern Score
    const behaviorScore = this.calculateBehaviorScore(factors.requestHistory);
    totalScore += behaviorScore * (this.WEIGHTS.behaviorPattern / 100);

    // 5. Time-based Risk Score
    const timeScore = this.calculateTimeScore();
    totalScore += timeScore * (this.WEIGHTS.timeOfDay / 100);

    // Normalize to 0-100
    return Math.min(100, Math.max(0, totalScore));
  }

  private calculateIdentityScore(identity: any): number {
    let score = 0;
    
    if (identity.mfaVerified) score += 40;
    if (identity.emailVerified) score += 20;
    if (identity.phoneVerified) score += 20;
    if (identity.accountAge > 365) score += 10; // บัญชีอายุมากกว่า 1 ปี
    if (identity.previousSuccessfulLogins > 10) score += 10;
    
    return score;
  }

  private async calculateLocationScore(ip: string): Promise<number> {
    const geoData = await this.geoIP.lookup(ip);
    let score = 80; // Base score

    // ลด score สำหรับ IP ที่มีความเสี่ยง
    if (geoData.isVPN) score -= 30;
    if (geoData.isTor) score -= 50;
    if (geoData.isProxy) score -= 20;
    if (geoData.isDatacenter) score -= 15;
    
    // ตรวจสอบประเทศที่มีความเสี่ยงสูง
    const highRiskCountries = ['XX', 'YY']; // ตัวอย่าง
    if (highRiskCountries.includes(geoData.countryCode)) score -= 20;

    return Math.max(0, score);
  }

  private async calculateDeviceScore(fingerprint?: string, userId?: string): Promise<number> {
    if (!fingerprint) return 40; // Unknown device

    const key = `device:${userId}:${fingerprint}`;
    const deviceHistory = await this.redis.get(key);

    if (!deviceHistory) {
      // Device ใหม่ - เก็บ fingerprint
      await this.redis.setex(key, 86400 * 30, JSON.stringify({
        firstSeen: Date.now(),
        useCount: 1
      }));
      return 50; // New device, moderate trust
    }

    const history = JSON.parse(deviceHistory);
    await this.redis.setex(key, 86400 * 30, JSON.stringify({
      ...history,
      lastSeen: Date.now(),
      useCount: history.useCount + 1
    }));

    // Device ที่เคยใช้บ่อยจะได้ score สูงขึ้น
    if (history.useCount > 100) return 100;
    if (history.useCount > 10) return 85;
    if (history.useCount > 5) return 70;
    return 60;
  }

  private calculateBehaviorScore(requestHistory: any[]): number {
    if (requestHistory.length === 0) return 60;

    const recentErrors = requestHistory.filter(r => r.statusCode >= 400).length;
    const errorRate = recentErrors / requestHistory.length;

    if (errorRate > 0.5) return 20; // มี error เยอะมาก
    if (errorRate > 0.3) return 40;
    if (errorRate > 0.1) return 60;
    return 80;
  }

  private calculateTimeScore(): number {
    const hour = new Date().getHours();
    // ช่วงเวลาทำงาน (8-18) ได้ score สูงกว่า
    if (hour >= 8 && hour <= 18) return 90;
    if (hour >= 6 && hour <= 22) return 70;
    return 50; // กลางดึก risk สูงกว่า
  }
}
```

---

## 2. OAuth 2.0 และ OpenID Connect Implementation

### 2.1 OAuth 2.0 Flow สำหรับ LINE Bot

```
┌─────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────┐
│  User   │    │  LINE LIFF   │    │  Your Server │    │  LINE    │
│         │    │  Frontend    │    │              │    │  API     │
└────┬────┘    └──────┬───────┘    └──────┬───────┘    └────┬─────┘
     │                │                    │                  │
     │  Open LIFF     │                    │                  │
     │───────────────▶│                    │                  │
     │                │  liff.login()      │                  │
     │                │───────────────────────────────────────▶│
     │                │                    │                  │
     │                │  Auth Code         │                  │
     │                │◀───────────────────────────────────────│
     │                │                    │                  │
     │                │  POST /auth/token  │                  │
     │                │  {code, state}     │                  │
     │                │───────────────────▶│                  │
     │                │                    │  Exchange Code   │
     │                │                    │─────────────────▶│
     │                │                    │                  │
     │                │                    │  access_token    │
     │                │                    │◀─────────────────│
     │                │                    │                  │
     │                │  {jwt, refresh}    │                  │
     │                │◀───────────────────│                  │
     │                │                    │                  │
```

### 2.2 OAuth 2.0 Server Implementation

```typescript
// auth/OAuthServer.ts
import express from 'express';
import jwt from 'jsonwebtoken';
import { v4 as uuidv4 } from 'uuid';
import axios from 'axios';
import { Redis } from 'ioredis';
import { KeyRotationService } from './KeyRotationService';

interface TokenPayload {
  sub: string;        // LINE User ID
  displayName: string;
  pictureUrl?: string;
  email?: string;
  iat: number;
  exp: number;
  jti: string;        // JWT ID for revocation
  scope: string[];
}

interface RefreshTokenData {
  userId: string;
  tokenFamily: string;  // สำหรับ Refresh Token Rotation
  issuedAt: number;
  expiresAt: number;
}

export class OAuthServer {
  private redis: Redis;
  private keyRotation: KeyRotationService;
  
  private readonly ACCESS_TOKEN_TTL = 900;        // 15 นาที
  private readonly REFRESH_TOKEN_TTL = 2592000;   // 30 วัน
  private readonly REFRESH_TOKEN_REUSE_WINDOW = 30; // 30 วินาที

  constructor() {
    this.redis = new Redis(process.env.REDIS_URL!);
    this.keyRotation = new KeyRotationService();
  }

  async handleCallback(code: string, state: string): Promise<{
    accessToken: string;
    refreshToken: string;
    expiresIn: number;
    tokenType: string;
  }> {
    // 1. Verify state parameter (CSRF protection)
    const stateData = await this.verifyState(state);
    if (!stateData) {
      throw new Error('Invalid state parameter');
    }

    // 2. Exchange code for LINE tokens
    const lineTokens = await this.exchangeCodeForTokens(code);

    // 3. Get LINE user profile
    const userProfile = await this.getLINEProfile(lineTokens.access_token);

    // 4. Verify ID token
    const idTokenPayload = await this.verifyIDToken(lineTokens.id_token);

    // 5. Create or update user in database
    const user = await this.upsertUser({
      lineUserId: userProfile.userId,
      displayName: userProfile.displayName,
      pictureUrl: userProfile.pictureUrl,
      email: idTokenPayload.email
    });

    // 6. Issue JWT tokens
    const accessToken = await this.issueAccessToken(user);
    const refreshToken = await this.issueRefreshToken(user);

    // 7. Revoke LINE tokens (we have our own token system)
    await this.revokeLINEToken(lineTokens.access_token);

    return {
      accessToken,
      refreshToken,
      expiresIn: this.ACCESS_TOKEN_TTL,
      tokenType: 'Bearer'
    };
  }

  private async exchangeCodeForTokens(code: string): Promise<any> {
    const response = await axios.post('https://api.line.me/oauth2/v2.1/token', 
      new URLSearchParams({
        grant_type: 'authorization_code',
        code,
        redirect_uri: process.env.LINE_CALLBACK_URL!,
        client_id: process.env.LINE_CHANNEL_ID!,
        client_secret: process.env.LINE_CHANNEL_SECRET!
      }),
      {
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
      }
    );

    return response.data;
  }

  private async verifyIDToken(idToken: string): Promise<any> {
    const response = await axios.post('https://api.line.me/oauth2/v2.1/verify', 
      new URLSearchParams({
        id_token: idToken,
        client_id: process.env.LINE_CHANNEL_ID!
      }),
      {
        headers: { 'Content-Type': 'application/x-www-form-urlencoded' }
      }
    );

    return response.data;
  }

  async issueAccessToken(user: any): Promise<string> {
    const privateKey = await this.keyRotation.getCurrentPrivateKey();
    const jti = uuidv4();

    const payload: TokenPayload = {
      sub: user.lineUserId,
      displayName: user.displayName,
      pictureUrl: user.pictureUrl,
      email: user.email,
      iat: Math.floor(Date.now() / 1000),
      exp: Math.floor(Date.now() / 1000) + this.ACCESS_TOKEN_TTL,
      jti,
      scope: user.scopes || ['read', 'write']
    };

    const token = jwt.sign(payload, privateKey, {
      algorithm: 'RS256',
      keyid: await this.keyRotation.getCurrentKeyId()
    });

    // เก็บ JTI ใน Redis สำหรับ token revocation
    await this.redis.setex(
      `jti:${jti}`,
      this.ACCESS_TOKEN_TTL,
      user.lineUserId
    );

    return token;
  }

  async issueRefreshToken(user: any): Promise<string> {
    const tokenFamily = uuidv4();
    const refreshToken = this.generateSecureToken();

    const tokenData: RefreshTokenData = {
      userId: user.lineUserId,
      tokenFamily,
      issuedAt: Date.now(),
      expiresAt: Date.now() + (this.REFRESH_TOKEN_TTL * 1000)
    };

    await this.redis.setex(
      `refresh:${refreshToken}`,
      this.REFRESH_TOKEN_TTL,
      JSON.stringify(tokenData)
    );

    return refreshToken;
  }

  async refreshAccessToken(refreshToken: string): Promise<{
    accessToken: string;
    refreshToken: string;
  }> {
    const tokenDataStr = await this.redis.get(`refresh:${refreshToken}`);
    
    if (!tokenDataStr) {
      throw new Error('Invalid or expired refresh token');
    }

    const tokenData: RefreshTokenData = JSON.parse(tokenDataStr);

    // Refresh Token Rotation - invalidate old token
    await this.redis.del(`refresh:${refreshToken}`);

    // ตรวจสอบว่า token family ยังปลอดภัย
    const familyKey = `family:${tokenData.tokenFamily}`;
    const familyRevoked = await this.redis.get(familyKey);
    if (familyRevoked) {
      // มีการ reuse refresh token - อาจถูก compromise
      await this.revokeAllUserTokens(tokenData.userId);
      throw new Error('Refresh token reuse detected - all tokens revoked');
    }

    const user = await this.getUserById(tokenData.userId);
    
    const newAccessToken = await this.issueAccessToken(user);
    const newRefreshToken = await this.issueRefreshToken(user);

    return {
      accessToken: newAccessToken,
      refreshToken: newRefreshToken
    };
  }

  async revokeToken(token: string): Promise<void> {
    try {
      const decoded = jwt.decode(token) as TokenPayload;
      if (decoded?.jti) {
        await this.redis.del(`jti:${decoded.jti}`);
      }
    } catch {}
  }

  async verifyAccessToken(token: string): Promise<TokenPayload> {
    const decoded = jwt.decode(token, { complete: true }) as any;
    if (!decoded) throw new Error('Invalid token');

    // ดึง public key ตาม key ID
    const publicKey = await this.keyRotation.getPublicKey(decoded.header.kid);
    
    const payload = jwt.verify(token, publicKey, {
      algorithms: ['RS256']
    }) as TokenPayload;

    // ตรวจสอบว่า JTI ยังไม่ถูก revoke
    const jtiExists = await this.redis.get(`jti:${payload.jti}`);
    if (!jtiExists) {
      throw new Error('Token has been revoked');
    }

    return payload;
  }

  private generateSecureToken(): string {
    const bytes = require('crypto').randomBytes(32);
    return bytes.toString('base64url');
  }

  private async verifyState(state: string): Promise<any> {
    const stateData = await this.redis.get(`oauth:state:${state}`);
    if (!stateData) return null;
    await this.redis.del(`oauth:state:${state}`);
    return JSON.parse(stateData);
  }

  private async getLINEProfile(accessToken: string): Promise<any> {
    const response = await axios.get('https://api.line.me/v2/profile', {
      headers: { Authorization: `Bearer ${accessToken}` }
    });
    return response.data;
  }

  private async upsertUser(data: any): Promise<any> {
    // Implementation with your database
    return data;
  }

  private async getUserById(userId: string): Promise<any> {
    // Implementation with your database
    return { lineUserId: userId };
  }

  private async revokeAllUserTokens(userId: string): Promise<void> {
    // Implementation to revoke all tokens for a user
  }

  private async revokeLINEToken(accessToken: string): Promise<void> {
    await axios.post('https://api.line.me/oauth2/v2.1/revoke',
      new URLSearchParams({
        access_token: accessToken,
        client_id: process.env.LINE_CHANNEL_ID!,
        client_secret: process.env.LINE_CHANNEL_SECRET!
      })
    );
  }
}
```

---

## 3. LINE Signature Verification

### 3.1 Secure Webhook Signature Verification

```typescript
// security/SignatureVerifier.ts
import crypto from 'crypto';
import { Request, Response, NextFunction } from 'express';
import { TimingSafeCompare } from './TimingSafeCompare';

export class LineSignatureVerifier {
  private channelSecret: string;
  
  constructor(channelSecret: string) {
    this.channelSecret = channelSecret;
  }

  middleware() {
    return (req: Request, res: Response, next: NextFunction) => {
      const signature = req.headers['x-line-signature'] as string;
      
      if (!signature) {
        res.status(400).json({
          error: 'Missing X-Line-Signature header'
        });
        return;
      }

      if (!this.verify(req.body, signature)) {
        res.status(401).json({
          error: 'Invalid signature'
        });
        return;
      }

      next();
    };
  }

  verify(body: Buffer | string, signature: string): boolean {
    const bodyString = Buffer.isBuffer(body) ? body.toString('utf8') : body;
    
    const expectedSignature = crypto
      .createHmac('SHA256', this.channelSecret)
      .update(bodyString)
      .digest('base64');

    // ใช้ timing-safe comparison เพื่อป้องกัน timing attacks
    return TimingSafeCompare.compare(expectedSignature, signature);
  }
}

// security/TimingSafeCompare.ts
export class TimingSafeCompare {
  static compare(a: string, b: string): boolean {
    if (a.length !== b.length) {
      // ยังคง perform comparison เพื่อป้องกัน timing attack
      crypto.timingSafeEqual(
        Buffer.from(a),
        Buffer.from(a)
      );
      return false;
    }

    return crypto.timingSafeEqual(
      Buffer.from(a),
      Buffer.from(b)
    );
  }
}
```

### 3.2 Rate Limiting สำหรับ Webhook

```typescript
// security/RateLimiter.ts
import { Redis } from 'ioredis';
import { Request, Response, NextFunction } from 'express';

interface RateLimitConfig {
  windowMs: number;
  maxRequests: number;
  keyPrefix: string;
  skipSuccessfulRequests?: boolean;
}

export class AdvancedRateLimiter {
  private redis: Redis;

  constructor() {
    this.redis = new Redis(process.env.REDIS_URL!);
  }

  // Sliding Window Rate Limiting
  slidingWindow(config: RateLimitConfig) {
    return async (req: Request, res: Response, next: NextFunction) => {
      const key = this.buildKey(config.keyPrefix, req);
      const now = Date.now();
      const windowStart = now - config.windowMs;

      const pipeline = this.redis.pipeline();
      
      // Remove old entries
      pipeline.zremrangebyscore(key, 0, windowStart);
      
      // Count current entries
      pipeline.zcard(key);
      
      // Add current request
      pipeline.zadd(key, now, `${now}-${Math.random()}`);
      
      // Set expiry
      pipeline.pexpire(key, config.windowMs);

      const results = await pipeline.exec();
      const requestCount = (results?.[1]?.[1] as number) || 0;

      res.setHeader('X-RateLimit-Limit', config.maxRequests);
      res.setHeader('X-RateLimit-Remaining', Math.max(0, config.maxRequests - requestCount - 1));
      res.setHeader('X-RateLimit-Reset', Math.ceil((now + config.windowMs) / 1000));

      if (requestCount >= config.maxRequests) {
        const retryAfter = Math.ceil(config.windowMs / 1000);
        res.setHeader('Retry-After', retryAfter);
        res.status(429).json({
          error: 'Too Many Requests',
          retryAfter
        });
        return;
      }

      next();
    };
  }

  // Adaptive Rate Limiting - ปรับตาม trust score
  adaptive(baseConfig: RateLimitConfig) {
    return async (req: Request, res: Response, next: NextFunction) => {
      const trustScore = (req as any).zeroTrustContext?.trustScore || 50;
      
      // ปรับ limit ตาม trust score
      const multiplier = trustScore / 100;
      const adjustedConfig = {
        ...baseConfig,
        maxRequests: Math.floor(baseConfig.maxRequests * (0.5 + multiplier))
      };

      return this.slidingWindow(adjustedConfig)(req, res, next);
    };
  }

  private buildKey(prefix: string, req: Request): string {
    const ip = req.headers['x-forwarded-for']?.toString().split(',')[0] || 
               req.socket.remoteAddress || 
               'unknown';
    return `ratelimit:${prefix}:${ip}`;
  }
}
```

---

## 4. API Keys Management

### 4.1 API Key Service

```typescript
// apikeys/ApiKeyService.ts
import crypto from 'crypto';
import { Redis } from 'ioredis';
import { Database } from '../database/Database';
import { AuditLogger } from '../logging/AuditLogger';

interface ApiKeyConfig {
  tenantId: string;
  name: string;
  scopes: string[];
  expiresAt?: Date;
  ipWhitelist?: string[];
  rateLimit?: {
    requestsPerMinute: number;
    requestsPerDay: number;
  };
}

interface ApiKey {
  id: string;
  key: string;          // hashed key
  prefix: string;       // first 8 chars for identification
  tenantId: string;
  name: string;
  scopes: string[];
  expiresAt?: Date;
  createdAt: Date;
  lastUsedAt?: Date;
  isActive: boolean;
  ipWhitelist?: string[];
  rateLimit?: any;
}

export class ApiKeyService {
  private redis: Redis;
  private db: Database;
  private auditLogger: AuditLogger;

  constructor() {
    this.redis = new Redis(process.env.REDIS_URL!);
    this.db = new Database();
    this.auditLogger = new AuditLogger();
  }

  async createApiKey(config: ApiKeyConfig): Promise<{ apiKey: string; keyData: ApiKey }> {
    // สร้าง key ที่ปลอดภัย
    const rawKey = this.generateApiKey();
    const prefix = rawKey.substring(0, 8);
    
    // Hash key ก่อนเก็บ
    const hashedKey = await this.hashApiKey(rawKey);

    const keyData: ApiKey = {
      id: crypto.randomUUID(),
      key: hashedKey,
      prefix,
      tenantId: config.tenantId,
      name: config.name,
      scopes: config.scopes,
      expiresAt: config.expiresAt,
      createdAt: new Date(),
      isActive: true,
      ipWhitelist: config.ipWhitelist,
      rateLimit: config.rateLimit
    };

    await this.db.saveApiKey(keyData);

    // Cache ใน Redis สำหรับ fast lookup
    await this.cacheApiKey(prefix, hashedKey, keyData);

    await this.auditLogger.log({
      action: 'API_KEY_CREATED',
      tenantId: config.tenantId,
      keyPrefix: prefix,
      scopes: config.scopes
    });

    // คืน raw key ครั้งเดียว - จะไม่เห็นอีกแล้ว
    return {
      apiKey: rawKey,
      keyData
    };
  }

  async validateApiKey(rawKey: string, sourceIP: string): Promise<{
    isValid: boolean;
    keyData?: ApiKey;
    reason?: string;
  }> {
    const prefix = rawKey.substring(0, 8);
    const hashedKey = await this.hashApiKey(rawKey);

    // ลอง cache ก่อน
    let keyData = await this.getFromCache(prefix, hashedKey);
    
    if (!keyData) {
      // ถ้าไม่อยู่ใน cache ดูจาก database
      keyData = await this.db.getApiKeyByPrefixAndHash(prefix, hashedKey);
    }

    if (!keyData) {
      return { isValid: false, reason: 'KEY_NOT_FOUND' };
    }

    if (!keyData.isActive) {
      return { isValid: false, reason: 'KEY_INACTIVE' };
    }

    if (keyData.expiresAt && new Date() > keyData.expiresAt) {
      return { isValid: false, reason: 'KEY_EXPIRED' };
    }

    // ตรวจสอบ IP Whitelist
    if (keyData.ipWhitelist && keyData.ipWhitelist.length > 0) {
      if (!this.isIPAllowed(sourceIP, keyData.ipWhitelist)) {
        return { isValid: false, reason: 'IP_NOT_ALLOWED' };
      }
    }

    // อัพเดต last used
    await this.updateLastUsed(keyData.id);

    return { isValid: true, keyData };
  }

  async rotateApiKey(keyId: string): Promise<{ newApiKey: string }> {
    const keyData = await this.db.getApiKeyById(keyId);
    if (!keyData) throw new Error('Key not found');

    // สร้าง key ใหม่
    const newRawKey = this.generateApiKey();
    const newPrefix = newRawKey.substring(0, 8);
    const newHashedKey = await this.hashApiKey(newRawKey);

    // Update ใน database
    await this.db.updateApiKey(keyId, {
      key: newHashedKey,
      prefix: newPrefix,
      lastRotatedAt: new Date()
    });

    // Clear cache เดิม
    await this.clearCache(keyData.prefix);

    // Cache key ใหม่
    await this.cacheApiKey(newPrefix, newHashedKey, {
      ...keyData,
      key: newHashedKey,
      prefix: newPrefix
    });

    await this.auditLogger.log({
      action: 'API_KEY_ROTATED',
      keyId,
      oldPrefix: keyData.prefix,
      newPrefix
    });

    return { newApiKey: newRawKey };
  }

  async revokeApiKey(keyId: string, reason: string): Promise<void> {
    const keyData = await this.db.getApiKeyById(keyId);
    if (!keyData) throw new Error('Key not found');

    await this.db.updateApiKey(keyId, {
      isActive: false,
      revokedAt: new Date(),
      revocationReason: reason
    });

    await this.clearCache(keyData.prefix);

    await this.auditLogger.log({
      action: 'API_KEY_REVOKED',
      keyId,
      reason
    });
  }

  private generateApiKey(): string {
    const prefix = 'lk_'; // LINE Key prefix
    const randomBytes = crypto.randomBytes(32);
    return prefix + randomBytes.toString('base64url');
  }

  private async hashApiKey(rawKey: string): Promise<string> {
    // ใช้ scrypt สำหรับ key hashing (แทน bcrypt สำหรับ API keys)
    return new Promise((resolve, reject) => {
      crypto.scrypt(rawKey, process.env.API_KEY_SALT!, 64, (err, derivedKey) => {
        if (err) reject(err);
        else resolve(derivedKey.toString('hex'));
      });
    });
  }

  private async cacheApiKey(prefix: string, hashedKey: string, keyData: ApiKey): Promise<void> {
    const cacheKey = `apikey:${prefix}:${hashedKey.substring(0, 16)}`;
    await this.redis.setex(cacheKey, 300, JSON.stringify(keyData)); // Cache 5 นาที
  }

  private async getFromCache(prefix: string, hashedKey: string): Promise<ApiKey | null> {
    const cacheKey = `apikey:${prefix}:${hashedKey.substring(0, 16)}`;
    const cached = await this.redis.get(cacheKey);
    return cached ? JSON.parse(cached) : null;
  }

  private async clearCache(prefix: string): Promise<void> {
    const keys = await this.redis.keys(`apikey:${prefix}:*`);
    if (keys.length > 0) {
      await this.redis.del(...keys);
    }
  }

  private isIPAllowed(ip: string, whitelist: string[]): boolean {
    // รองรับ CIDR notation
    const { ipRangeCheck } = require('ip-range-check');
    return whitelist.some(range => ipRangeCheck(ip, range));
  }

  private async updateLastUsed(keyId: string): Promise<void> {
    // Async update - ไม่ block response
    this.db.updateApiKey(keyId, { lastUsedAt: new Date() }).catch(console.error);
  }
}
```

---

## 5. Secret Rotation Procedures

### 5.1 Automated Secret Rotation

```typescript
// secrets/SecretRotationService.ts
import { SecretsManagerClient, 
         GetSecretValueCommand,
         UpdateSecretCommand,
         RotateSecretCommand } from '@aws-sdk/client-secrets-manager';
import { EventEmitter } from 'events';
import { SlackNotifier } from '../notifications/SlackNotifier';

interface SecretMetadata {
  secretId: string;
  lastRotated: Date;
  rotationInterval: number; // วัน
  nextRotation: Date;
  type: 'LINE_CHANNEL_SECRET' | 'DATABASE_PASSWORD' | 'JWT_PRIVATE_KEY' | 'API_KEY';
  dependentServices: string[];
}

export class SecretRotationService extends EventEmitter {
  private client: SecretsManagerClient;
  private notifier: SlackNotifier;
  private secrets: Map<string, SecretMetadata> = new Map();

  constructor() {
    super();
    this.client = new SecretsManagerClient({ region: process.env.AWS_REGION });
    this.notifier = new SlackNotifier();
  }

  async scheduleRotations(): Promise<void> {
    // ตรวจสอบทุก 6 ชั่วโมง
    setInterval(async () => {
      await this.checkAndRotateExpiredSecrets();
    }, 6 * 60 * 60 * 1000);

    // รันครั้งแรกทันที
    await this.checkAndRotateExpiredSecrets();
  }

  private async checkAndRotateExpiredSecrets(): Promise<void> {
    const now = new Date();
    
    for (const [key, metadata] of this.secrets.entries()) {
      if (metadata.nextRotation <= now) {
        try {
          await this.rotateSecret(metadata);
        } catch (error) {
          await this.notifier.sendCriticalAlert({
            title: 'Secret Rotation Failed',
            message: `Failed to rotate secret: ${metadata.secretId}`,
            error: (error as Error).message
          });
        }
      }
      
      // แจ้งเตือน 7 วันก่อน rotation
      const daysUntilRotation = (metadata.nextRotation.getTime() - now.getTime()) / (1000 * 60 * 60 * 24);
      if (daysUntilRotation <= 7) {
        await this.notifier.sendWarning({
          title: 'Secret Rotation Due Soon',
          message: `Secret ${metadata.secretId} will rotate in ${Math.ceil(daysUntilRotation)} days`
        });
      }
    }
  }

  async rotateSecret(metadata: SecretMetadata): Promise<void> {
    console.log(`Starting rotation for secret: ${metadata.secretId}`);
    
    // 1. Generate new secret value
    const newSecretValue = await this.generateNewSecretValue(metadata.type);

    // 2. Test new secret before committing
    const isValid = await this.testNewSecret(metadata.type, newSecretValue);
    if (!isValid) {
      throw new Error(`New secret validation failed for ${metadata.secretId}`);
    }

    // 3. Update secret in AWS Secrets Manager
    await this.client.send(new UpdateSecretCommand({
      SecretId: metadata.secretId,
      SecretString: JSON.stringify({
        value: newSecretValue,
        rotatedAt: new Date().toISOString(),
        previousVersion: await this.getCurrentSecretVersion(metadata.secretId)
      })
    }));

    // 4. Notify dependent services
    await this.notifyDependentServices(metadata.dependentServices, metadata.secretId);

    // 5. Wait for services to reload (grace period)
    await this.waitForServiceReload(30000); // 30 วินาที

    // 6. Update metadata
    const nextRotation = new Date();
    nextRotation.setDate(nextRotation.getDate() + metadata.rotationInterval);
    
    this.secrets.set(metadata.secretId, {
      ...metadata,
      lastRotated: new Date(),
      nextRotation
    });

    // 7. Emit event
    this.emit('secretRotated', {
      secretId: metadata.secretId,
      type: metadata.type,
      timestamp: new Date()
    });

    await this.notifier.sendInfo({
      title: 'Secret Rotated Successfully',
      message: `Secret ${metadata.secretId} has been rotated. Next rotation: ${nextRotation.toISOString()}`
    });
  }

  private async generateNewSecretValue(type: SecretMetadata['type']): Promise<string> {
    switch (type) {
      case 'LINE_CHANNEL_SECRET':
        // LINE Channel Secret ต้องเปลี่ยนผ่าน LINE Developer Console API
        return await this.rotateLINEChannelSecret();
      
      case 'DATABASE_PASSWORD':
        return this.generateStrongPassword(32);
      
      case 'JWT_PRIVATE_KEY':
        return await this.generateRSAKeyPair();
      
      case 'API_KEY':
        const crypto = require('crypto');
        return 'lk_' + crypto.randomBytes(32).toString('base64url');
      
      default:
        throw new Error(`Unknown secret type: ${type}`);
    }
  }

  private generateStrongPassword(length: number): string {
    const charset = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*()_+-=[]{}|;:,.<>?';
    const crypto = require('crypto');
    let password = '';
    
    // Ensure all character types are included
    password += 'aA1!';
    
    for (let i = password.length; i < length; i++) {
      const randomIndex = crypto.randomInt(0, charset.length);
      password += charset[randomIndex];
    }
    
    // Shuffle
    return password.split('').sort(() => crypto.randomInt(-1, 2)).join('');
  }

  private async generateRSAKeyPair(): Promise<string> {
    const { generateKeyPairSync } = require('crypto');
    const { privateKey } = generateKeyPairSync('rsa', {
      modulusLength: 4096,
      publicKeyEncoding: { type: 'spki', format: 'pem' },
      privateKeyEncoding: { type: 'pkcs8', format: 'pem' }
    });
    return privateKey;
  }

  private async testNewSecret(type: SecretMetadata['type'], value: string): Promise<boolean> {
    // ทดสอบ secret ใหม่ก่อน commit
    switch (type) {
      case 'DATABASE_PASSWORD':
        return await this.testDatabaseConnection(value);
      case 'JWT_PRIVATE_KEY':
        return this.testJWTSigning(value);
      default:
        return true;
    }
  }

  private async testDatabaseConnection(password: string): Promise<boolean> {
    try {
      // ทดสอบ connect database ด้วย password ใหม่
      return true;
    } catch {
      return false;
    }
  }

  private testJWTSigning(privateKey: string): boolean {
    try {
      const jwt = require('jsonwebtoken');
      const token = jwt.sign({ test: true }, privateKey, { algorithm: 'RS256' });
      jwt.verify(token, this.extractPublicKey(privateKey));
      return true;
    } catch {
      return false;
    }
  }

  private extractPublicKey(privateKey: string): string {
    const { createPublicKey } = require('crypto');
    return createPublicKey(privateKey).export({ type: 'spki', format: 'pem' }) as string;
  }

  private async notifyDependentServices(services: string[], secretId: string): Promise<void> {
    // ส่งสัญญาณให้ services reload secret
    for (const service of services) {
      // ผ่าน message queue หรือ HTTP endpoint
      console.log(`Notifying service: ${service} about rotation of ${secretId}`);
    }
  }

  private async waitForServiceReload(ms: number): Promise<void> {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  private async getCurrentSecretVersion(secretId: string): Promise<string> {
    const response = await this.client.send(new GetSecretValueCommand({ SecretId: secretId }));
    return response.VersionId || '';
  }

  private async rotateLINEChannelSecret(): Promise<string> {
    // ต้องทำผ่าน LINE Developer Console
    // หรือใช้ LINE Management API ถ้ามี
    throw new Error('LINE Channel Secret must be rotated manually through LINE Developer Console');
  }
}
```

---

## 6. Encryption at Rest และ in Transit

### 6.1 Key Management ด้วย AWS KMS

```typescript
// encryption/KMSService.ts
import { KMSClient, 
         EncryptCommand, 
         DecryptCommand,
         GenerateDataKeyCommand,
         CreateKeyCommand,
         ScheduleKeyDeletionCommand } from '@aws-sdk/client-kms';

interface EncryptedData {
  ciphertext: string;
  encryptedDataKey: string;
  keyId: string;
  algorithm: string;
  iv: string;
}

export class KMSEncryptionService {
  private client: KMSClient;
  private masterKeyId: string;

  constructor() {
    this.client = new KMSClient({ region: process.env.AWS_REGION });
    this.masterKeyId = process.env.KMS_MASTER_KEY_ID!;
  }

  // Envelope Encryption Pattern
  async encryptData(plaintext: string, context: Record<string, string> = {}): Promise<EncryptedData> {
    // 1. สร้าง Data Encryption Key (DEK)
    const dekResponse = await this.client.send(new GenerateDataKeyCommand({
      KeyId: this.masterKeyId,
      KeySpec: 'AES_256',
      EncryptionContext: context
    }));

    const plaintextDEK = dekResponse.Plaintext!;
    const encryptedDEK = dekResponse.CiphertextBlob!;

    // 2. Encrypt data ด้วย DEK
    const crypto = require('crypto');
    const iv = crypto.randomBytes(16);
    const cipher = crypto.createCipheriv('aes-256-gcm', plaintextDEK, iv);
    
    let encrypted = cipher.update(plaintext, 'utf8', 'base64');
    encrypted += cipher.final('base64');
    const authTag = cipher.getAuthTag();

    // 3. Clear DEK จาก memory
    plaintextDEK.fill(0);

    return {
      ciphertext: encrypted + ':' + authTag.toString('base64'),
      encryptedDataKey: Buffer.from(encryptedDEK).toString('base64'),
      keyId: this.masterKeyId,
      algorithm: 'AES-256-GCM',
      iv: iv.toString('base64')
    };
  }

  async decryptData(encryptedData: EncryptedData, context: Record<string, string> = {}): Promise<string> {
    // 1. Decrypt DEK ด้วย KMS
    const dekResponse = await this.client.send(new DecryptCommand({
      CiphertextBlob: Buffer.from(encryptedData.encryptedDataKey, 'base64'),
      EncryptionContext: context
    }));

    const plaintextDEK = dekResponse.Plaintext!;

    // 2. Decrypt data ด้วย DEK
    const crypto = require('crypto');
    const [ciphertext, authTagBase64] = encryptedData.ciphertext.split(':');
    const iv = Buffer.from(encryptedData.iv, 'base64');
    const authTag = Buffer.from(authTagBase64, 'base64');
    
    const decipher = crypto.createDecipheriv('aes-256-gcm', plaintextDEK, iv);
    decipher.setAuthTag(authTag);
    
    let decrypted = decipher.update(ciphertext, 'base64', 'utf8');
    decrypted += decipher.final('utf8');

    // 3. Clear DEK จาก memory
    (plaintextDEK as any).fill(0);

    return decrypted;
  }

  // Field-Level Encryption สำหรับ PII data
  async encryptPII(data: Record<string, string>, piiFields: string[]): Promise<Record<string, any>> {
    const result = { ...data };
    
    for (const field of piiFields) {
      if (result[field]) {
        const encrypted = await this.encryptData(result[field], {
          fieldName: field,
          purpose: 'PII_STORAGE'
        });
        result[field] = JSON.stringify(encrypted);
      }
    }
    
    return result;
  }

  async decryptPII(data: Record<string, any>, piiFields: string[]): Promise<Record<string, string>> {
    const result = { ...data };
    
    for (const field of piiFields) {
      if (result[field] && typeof result[field] === 'string') {
        try {
          const encryptedData = JSON.parse(result[field]);
          result[field] = await this.decryptData(encryptedData, {
            fieldName: field,
            purpose: 'PII_STORAGE'
          });
        } catch {
          // Not encrypted, leave as is
        }
      }
    }
    
    return result;
  }
}
```

### 6.2 TLS Configuration

```nginx
# nginx/ssl.conf - Production TLS Configuration
server {
    listen 443 ssl http2;
    server_name your-linebot-domain.com;

    # Certificate
    ssl_certificate /etc/letsencrypt/live/your-domain/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/your-domain/privkey.pem;

    # TLS Protocols - เฉพาะ 1.2 และ 1.3
    ssl_protocols TLSv1.2 TLSv1.3;

    # Strong Cipher Suites
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # HSTS
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # OCSP Stapling
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/your-domain/chain.pem;

    # DH Parameters
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;

    # Session
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # Security Headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'nonce-${request_id}' https://cdn.linewebhook.me; style-src 'self' 'unsafe-inline'; img-src 'self' data: https://profile.line-scdn.net; connect-src 'self' https://api.line.me;" always;
    add_header Permissions-Policy "accelerometer=(), camera=(), geolocation=(), gyroscope=(), magnetometer=(), microphone=(), payment=(), usb=()" always;
}
```

---

## 7. Security Scanning ใน CI/CD

### 7.1 GitHub Actions Security Pipeline

```yaml
# .github/workflows/security-scan.yml
name: Security Scanning Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 1'  # ทุกวันจันทร์

jobs:
  # SAST - Static Application Security Testing
  sast-scan:
    name: SAST Scan
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Run ESLint Security Plugin
        run: |
          npm install --save-dev eslint-plugin-security eslint-plugin-no-secrets
          npx eslint . --ext .ts,.js \
            --plugin security \
            --plugin no-secrets \
            --rule 'security/detect-non-literal-regexp: error' \
            --rule 'security/detect-unsafe-regex: error' \
            --rule 'security/detect-eval-with-expression: error' \
            --rule 'no-secrets/no-secrets: [error, {tolerance: 4.5}]' \
            --format @microsoft/eslint-formatter-sarif \
            --output-file eslint-results.sarif || true

      - name: Upload ESLint SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: eslint-results.sarif

      - name: Run Semgrep
        uses: returntocorp/semgrep-action@v1
        with:
          config: >-
            p/typescript
            p/nodejs
            p/jwt
            p/owasp-top-ten
            p/security-audit
          generateSarif: '1'

      - name: Upload Semgrep SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: semgrep.sarif

      - name: CodeQL Analysis
        uses: github/codeql-action/init@v3
        with:
          languages: javascript, typescript
          queries: +security-and-quality

      - name: Perform CodeQL Analysis
        uses: github/codeql-action/analyze@v3

  # Dependency Scanning
  dependency-scan:
    name: Dependency Vulnerability Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run npm audit
        run: |
          npm audit --json > npm-audit.json || true
          # Fail on high/critical vulnerabilities
          npm audit --audit-level=high

      - name: Run Snyk
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high --sarif-file-output=snyk.sarif

      - name: Upload Snyk SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: snyk.sarif

      - name: Run OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'LINE Bot'
          path: '.'
          format: 'HTML,JSON,SARIF'
          out: 'reports'
          args: >
            --enableRetired
            --enableExperimental
            --failOnCVSS 7

  # Container Security Scanning
  container-scan:
    name: Container Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t linebot:${{ github.sha }} .

      - name: Run Trivy Container Scan
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'linebot:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-container.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy SARIF
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-container.sarif

      - name: Run Hadolint (Dockerfile linting)
        uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
          format: sarif
          output-file: hadolint.sarif

  # Secret Scanning
  secret-scan:
    name: Secret Detection
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Run TruffleHog
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --debug --only-verified

      - name: Run GitLeaks
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}

  # DAST - Dynamic Application Security Testing
  dast-scan:
    name: DAST Scan
    runs-on: ubuntu-latest
    needs: [sast-scan]
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4

      - name: Start application
        run: |
          docker-compose -f docker-compose.test.yml up -d
          sleep 30  # รอให้ app พร้อม

      - name: Run OWASP ZAP Baseline Scan
        uses: zaproxy/action-baseline@v0.10.0
        with:
          target: 'http://localhost:3000'
          rules_file_name: '.zap/rules.tsv'
          cmd_options: '-a -j'

      - name: Run OWASP ZAP API Scan
        uses: zaproxy/action-api-scan@v0.7.0
        with:
          target: 'http://localhost:3000/api/openapi.json'
          format: openapi
          cmd_options: '-a'

      - name: Stop application
        if: always()
        run: docker-compose -f docker-compose.test.yml down

  # Infrastructure Scanning
  infrastructure-scan:
    name: Infrastructure Security Scan
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Run Terraform Security Scan (tfsec)
        uses: aquasecurity/tfsec-action@v1.0.0
        with:
          soft_fail: false

      - name: Run Checkov
        id: checkov
        uses: bridgecrewio/checkov-action@master
        with:
          directory: terraform/
          framework: terraform
          output_format: sarif
          output_file_path: checkov.sarif
          soft_fail: false
          check: CKV_AWS_*,CKV_GCP_*,CKV_K8S_*

  # Security Report
  security-report:
    name: Generate Security Report
    runs-on: ubuntu-latest
    needs: [sast-scan, dependency-scan, container-scan, secret-scan]
    if: always()
    steps:
      - name: Download all artifacts
        uses: actions/download-artifact@v4

      - name: Generate combined security report
        run: |
          echo "## Security Scan Summary" > security-report.md
          echo "**Date:** $(date)" >> security-report.md
          echo "**Commit:** ${{ github.sha }}" >> security-report.md
          echo "" >> security-report.md
          # Add scan results summary

      - name: Comment PR with security report
        if: github.event_name == 'pull_request'
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const report = fs.readFileSync('security-report.md', 'utf8');
            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: report
            });
```

---

## 8. Vulnerability Management

### 8.1 Vulnerability Tracking System

```typescript
// vulnerability/VulnerabilityManager.ts
import { Octokit } from '@octokit/rest';
import { JiraClient } from '../integrations/JiraClient';

interface Vulnerability {
  id: string;
  cveId?: string;
  title: string;
  description: string;
  severity: 'CRITICAL' | 'HIGH' | 'MEDIUM' | 'LOW' | 'INFORMATIONAL';
  cvssScore?: number;
  affectedComponent: string;
  affectedVersion: string;
  fixedVersion?: string;
  status: 'OPEN' | 'IN_PROGRESS' | 'PATCHED' | 'WONT_FIX' | 'FALSE_POSITIVE';
  discoveredAt: Date;
  dueDate: Date;
  assignee?: string;
  references: string[];
  remediationSteps: string[];
}

export class VulnerabilityManager {
  private octokit: Octokit;
  private jira: JiraClient;

  // SLA ตาม severity
  private readonly SLA_DAYS = {
    CRITICAL: 1,
    HIGH: 7,
    MEDIUM: 30,
    LOW: 90,
    INFORMATIONAL: 180
  };

  constructor() {
    this.octokit = new Octokit({ auth: process.env.GITHUB_TOKEN });
    this.jira = new JiraClient();
  }

  async processVulnerabilityReport(report: any[]): Promise<void> {
    for (const finding of report) {
      const vuln = this.normalizeVulnerability(finding);
      await this.triageVulnerability(vuln);
    }
  }

  private async triageVulnerability(vuln: Vulnerability): Promise<void> {
    // 1. Calculate due date based on severity
    const dueDate = new Date();
    dueDate.setDate(dueDate.getDate() + this.SLA_DAYS[vuln.severity]);
    vuln.dueDate = dueDate;

    // 2. Create GitHub Issue
    if (vuln.severity === 'CRITICAL' || vuln.severity === 'HIGH') {
      await this.createGitHubSecurityIssue(vuln);
    }

    // 3. Create Jira ticket
    await this.createJiraTicket(vuln);

    // 4. Alert if CRITICAL
    if (vuln.severity === 'CRITICAL') {
      await this.sendImmediateAlert(vuln);
    }
  }

  private async createGitHubSecurityIssue(vuln: Vulnerability): Promise<void> {
    const body = this.formatVulnerabilityReport(vuln);
    
    await this.octokit.rest.issues.create({
      owner: process.env.GITHUB_OWNER!,
      repo: process.env.GITHUB_REPO!,
      title: `[SECURITY][${vuln.severity}] ${vuln.title}`,
      body,
      labels: ['security', vuln.severity.toLowerCase(), 'vulnerability'],
      assignees: vuln.assignee ? [vuln.assignee] : []
    });
  }

  private formatVulnerabilityReport(vuln: Vulnerability): string {
    return `
## Vulnerability Report

**Severity:** ${vuln.severity}
**CVSS Score:** ${vuln.cvssScore || 'N/A'}
**CVE ID:** ${vuln.cveId || 'N/A'}
**Due Date:** ${vuln.dueDate.toISOString().split('T')[0]}

### Description
${vuln.description}

### Affected Component
- Component: ${vuln.affectedComponent}
- Version: ${vuln.affectedVersion}
- Fixed Version: ${vuln.fixedVersion || 'Not yet patched'}

### Remediation Steps
${vuln.remediationSteps.map((step, i) => `${i + 1}. ${step}`).join('\n')}

### References
${vuln.references.map(ref => `- ${ref}`).join('\n')}
    `.trim();
  }

  private normalizeVulnerability(finding: any): Vulnerability {
    return {
      id: finding.id || require('crypto').randomUUID(),
      cveId: finding.cveId,
      title: finding.title || finding.name,
      description: finding.description,
      severity: this.mapSeverity(finding.severity),
      cvssScore: finding.cvssScore,
      affectedComponent: finding.component || finding.package,
      affectedVersion: finding.version,
      fixedVersion: finding.fixedVersion,
      status: 'OPEN',
      discoveredAt: new Date(),
      dueDate: new Date(), // Will be set in triage
      references: finding.references || [],
      remediationSteps: finding.remediationSteps || []
    };
  }

  private mapSeverity(severity: string): Vulnerability['severity'] {
    const mapping: Record<string, Vulnerability['severity']> = {
      'critical': 'CRITICAL',
      'high': 'HIGH',
      'medium': 'MEDIUM',
      'low': 'LOW',
      'info': 'INFORMATIONAL',
      'informational': 'INFORMATIONAL'
    };
    return mapping[severity.toLowerCase()] || 'MEDIUM';
  }

  private async createJiraTicket(vuln: Vulnerability): Promise<void> {
    // Implementation
  }

  private async sendImmediateAlert(vuln: Vulnerability): Promise<void> {
    // PagerDuty หรือ OpsGenie alert
  }
}
```

---

## 9. Penetration Testing Methodology

### 9.1 LINE Bot Pentest Framework

```typescript
// pentest/LineBotPentester.ts
// เครื่องมือสำหรับทดสอบความปลอดภัย LINE Bot (Internal Use Only)

interface PentestResult {
  testCase: string;
  status: 'PASS' | 'FAIL' | 'WARNING';
  severity?: string;
  description: string;
  evidence?: string;
  recommendation?: string;
}

export class LineBotSecurityTester {
  private targetUrl: string;
  private results: PentestResult[] = [];

  constructor(targetUrl: string) {
    this.targetUrl = targetUrl;
  }

  async runAllTests(): Promise<PentestResult[]> {
    console.log('Starting LINE Bot Security Assessment...\n');

    // 1. Webhook Security Tests
    await this.testWebhookSecurity();

    // 2. Authentication Tests
    await this.testAuthentication();

    // 3. Authorization Tests
    await this.testAuthorization();

    // 4. Input Validation Tests
    await this.testInputValidation();

    // 5. Rate Limiting Tests
    await this.testRateLimiting();

    // 6. Information Disclosure Tests
    await this.testInformationDisclosure();

    // 7. Business Logic Tests
    await this.testBusinessLogic();

    return this.results;
  }

  private async testWebhookSecurity(): Promise<void> {
    // Test 1: Webhook without signature
    await this.runTest({
      name: 'Webhook Without Signature',
      test: async () => {
        const response = await fetch(`${this.targetUrl}/webhook`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({ destination: 'test', events: [] })
        });
        return response.status === 401;
      },
      failMessage: 'Webhook accepts requests without X-Line-Signature',
      passMessage: 'Webhook correctly rejects unsigned requests'
    });

    // Test 2: Invalid signature
    await this.runTest({
      name: 'Invalid Webhook Signature',
      test: async () => {
        const response = await fetch(`${this.targetUrl}/webhook`, {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            'X-Line-Signature': 'invalid_signature'
          },
          body: JSON.stringify({ destination: 'test', events: [] })
        });
        return response.status === 401;
      },
      failMessage: 'Webhook accepts requests with invalid signature',
      passMessage: 'Webhook correctly rejects invalid signatures'
    });

    // Test 3: Replay attack
    await this.runTest({
      name: 'Webhook Replay Attack',
      test: async () => {
        // ทดสอบว่า webhook ป้องกัน replay attack หรือไม่
        // โดยส่ง request เดิมซ้ำๆ
        return true; // Placeholder
      },
      failMessage: 'Webhook is vulnerable to replay attacks',
      passMessage: 'Webhook has replay attack protection'
    });
  }

  private async testAuthentication(): Promise<void> {
    // Test JWT Security
    await this.runTest({
      name: 'JWT Algorithm Confusion Attack',
      test: async () => {
        // ทดสอบ none algorithm attack
        const fakeToken = this.createNoneAlgorithmJWT({ userId: 'attacker' });
        const response = await fetch(`${this.targetUrl}/api/profile`, {
          headers: { Authorization: `Bearer ${fakeToken}` }
        });
        return response.status === 401;
      },
      failMessage: 'API accepts JWT with none algorithm',
      passMessage: 'API correctly rejects JWT with none algorithm'
    });

    // Test expired tokens
    await this.runTest({
      name: 'Expired JWT Token',
      test: async () => {
        const expiredToken = this.createExpiredJWT();
        const response = await fetch(`${this.targetUrl}/api/profile`, {
          headers: { Authorization: `Bearer ${expiredToken}` }
        });
        return response.status === 401;
      },
      failMessage: 'API accepts expired JWT tokens',
      passMessage: 'API correctly rejects expired tokens'
    });
  }

  private async testInputValidation(): Promise<void> {
    const payloads = [
      // XSS payloads
      '<script>alert(1)</script>',
      '"><img src=x onerror=alert(1)>',
      
      // SQL Injection
      "' OR '1'='1",
      "1; DROP TABLE users--",
      
      // NoSQL Injection
      '{"$gt": ""}',
      '{"$where": "this.password.length > 0"}',
      
      // Command Injection
      '; ls -la',
      '`id`',
      '$(whoami)',
      
      // Path Traversal
      '../../../etc/passwd',
      '..\\..\\..\\windows\\win.ini',
      
      // SSRF
      'http://169.254.169.254/latest/meta-data/',
      'http://localhost:22/',
      
      // XXE
      '<?xml version="1.0"?><!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]><foo>&xxe;</foo>'
    ];

    for (const payload of payloads) {
      await this.runTest({
        name: `Input Validation: ${payload.substring(0, 30)}...`,
        test: async () => {
          const response = await fetch(`${this.targetUrl}/api/message`, {
            method: 'POST',
            headers: { 'Content-Type': 'application/json' },
            body: JSON.stringify({ message: payload })
          });
          // ควรได้ 400 Bad Request หรือ sanitize input ออก
          const body = await response.json();
          return !this.responseContainsPayload(body, payload);
        },
        failMessage: `Application vulnerable to: ${payload.substring(0, 30)}`,
        passMessage: `Application properly handles: ${payload.substring(0, 30)}`
      });
    }
  }

  private async testRateLimiting(): Promise<void> {
    await this.runTest({
      name: 'Rate Limiting Enforcement',
      test: async () => {
        const requests = Array(100).fill(null).map(() =>
          fetch(`${this.targetUrl}/api/search?q=test`)
        );
        
        const responses = await Promise.all(requests);
        const rateLimited = responses.some(r => r.status === 429);
        return rateLimited;
      },
      failMessage: 'No rate limiting detected after 100 rapid requests',
      passMessage: 'Rate limiting is properly enforced'
    });
  }

  private async testInformationDisclosure(): Promise<void> {
    // Test error messages
    await this.runTest({
      name: 'Error Message Information Disclosure',
      test: async () => {
        const response = await fetch(`${this.targetUrl}/api/user/99999`);
        const body = await response.text();
        
        const sensitivePatterns = [
          /stack trace/i,
          /at Object\./,
          /node_modules/,
          /\.ts:\d+:\d+/,
          /password/i,
          /secret/i
        ];
        
        return !sensitivePatterns.some(pattern => pattern.test(body));
      },
      failMessage: 'Error messages leak sensitive information',
      passMessage: 'Error messages do not disclose sensitive information'
    });

    // Test HTTP headers
    await this.runTest({
      name: 'Server Version Disclosure in Headers',
      test: async () => {
        const response = await fetch(this.targetUrl);
        const server = response.headers.get('server');
        const xPoweredBy = response.headers.get('x-powered-by');
        
        return !server?.includes('nginx/') && !server?.includes('Apache/') && !xPoweredBy;
      },
      failMessage: 'Server version disclosed in HTTP headers',
      passMessage: 'Server version not disclosed in headers'
    });
  }

  private async testBusinessLogic(): Promise<void> {
    // Test IDOR
    await this.runTest({
      name: 'IDOR - Insecure Direct Object Reference',
      test: async () => {
        // ทดสอบว่า user A ไม่สามารถเข้าถึงข้อมูลของ user B
        return true; // Placeholder
      },
      failMessage: 'IDOR vulnerability detected',
      passMessage: 'Authorization properly restricts object access'
    });
  }

  private async testAuthorization(): Promise<void> {
    // Test privilege escalation
    await this.runTest({
      name: 'Horizontal Privilege Escalation',
      test: async () => {
        return true; // Placeholder
      },
      failMessage: 'Horizontal privilege escalation possible',
      passMessage: 'Authorization properly enforced'
    });
  }

  private async runTest(config: {
    name: string;
    test: () => Promise<boolean>;
    failMessage: string;
    passMessage: string;
    severity?: string;
  }): Promise<void> {
    try {
      const passed = await config.test();
      this.results.push({
        testCase: config.name,
        status: passed ? 'PASS' : 'FAIL',
        severity: passed ? undefined : (config.severity || 'HIGH'),
        description: passed ? config.passMessage : config.failMessage,
        recommendation: passed ? undefined : `Fix: ${config.failMessage}`
      });
    } catch (error) {
      this.results.push({
        testCase: config.name,
        status: 'WARNING',
        description: `Test error: ${(error as Error).message}`
      });
    }
  }

  private createNoneAlgorithmJWT(payload: any): string {
    const header = Buffer.from(JSON.stringify({ alg: 'none', typ: 'JWT' })).toString('base64url');
    const body = Buffer.from(JSON.stringify(payload)).toString('base64url');
    return `${header}.${body}.`;
  }

  private createExpiredJWT(): string {
    const jwt = require('jsonwebtoken');
    return jwt.sign({ sub: 'test', exp: Math.floor(Date.now() / 1000) - 3600 }, 'test-secret');
  }

  private responseContainsPayload(body: any, payload: string): boolean {
    const bodyStr = JSON.stringify(body);
    return bodyStr.includes(payload);
  }

  generateReport(): string {
    const passed = this.results.filter(r => r.status === 'PASS').length;
    const failed = this.results.filter(r => r.status === 'FAIL').length;
    const warnings = this.results.filter(r => r.status === 'WARNING').length;

    return `
# LINE Bot Security Assessment Report

## Summary
- **Total Tests:** ${this.results.length}
- **Passed:** ${passed}
- **Failed:** ${failed}
- **Warnings:** ${warnings}
- **Security Score:** ${Math.round((passed / this.results.length) * 100)}%

## Findings

${this.results
  .filter(r => r.status === 'FAIL')
  .map(r => `### [${r.severity}] ${r.testCase}\n${r.description}\n\n**Recommendation:** ${r.recommendation}`)
  .join('\n\n')}

## All Test Results

| Test Case | Status | Severity |
|-----------|--------|----------|
${this.results.map(r => `| ${r.testCase} | ${r.status} | ${r.severity || '-'} |`).join('\n')}
    `.trim();
  }
}
```

---

## 10. Incident Response Playbook

### 10.1 Security Incident Response Plan

```typescript
// incident/IncidentResponseService.ts
import { SlackClient } from '../integrations/SlackClient';
import { PagerDutyClient } from '../integrations/PagerDutyClient';
import { Redis } from 'ioredis';

type IncidentSeverity = 'P1' | 'P2' | 'P3' | 'P4';
type IncidentStatus = 'DETECTED' | 'INVESTIGATING' | 'CONTAINED' | 'ERADICATED' | 'RECOVERED' | 'CLOSED';

interface SecurityIncident {
  id: string;
  severity: IncidentSeverity;
  type: string;
  title: string;
  description: string;
  status: IncidentStatus;
  detectedAt: Date;
  reportedAt?: Date;
  containedAt?: Date;
  resolvedAt?: Date;
  affectedSystems: string[];
  affectedUsers: number;
  timeline: IncidentEvent[];
  actions: IncidentAction[];
  assignee?: string;
}

interface IncidentEvent {
  timestamp: Date;
  description: string;
  author: string;
}

interface IncidentAction {
  id: string;
  action: string;
  status: 'PENDING' | 'IN_PROGRESS' | 'COMPLETED';
  assignee: string;
  dueDate: Date;
}

export class IncidentResponseService {
  private slack: SlackClient;
  private pagerDuty: PagerDutyClient;
  private redis: Redis;
  private incidents: Map<string, SecurityIncident> = new Map();

  // Response time SLAs
  private readonly RESPONSE_SLA = {
    P1: 15,   // 15 นาที
    P2: 60,   // 1 ชั่วโมง
    P3: 240,  // 4 ชั่วโมง
    P4: 1440  // 24 ชั่วโมง
  };

  constructor() {
    this.slack = new SlackClient();
    this.pagerDuty = new PagerDutyClient();
    this.redis = new Redis(process.env.REDIS_URL!);
  }

  async declareIncident(input: Partial<SecurityIncident>): Promise<SecurityIncident> {
    const incident: SecurityIncident = {
      id: `INC-${Date.now()}`,
      severity: input.severity || 'P3',
      type: input.type || 'UNKNOWN',
      title: input.title || 'Security Incident',
      description: input.description || '',
      status: 'DETECTED',
      detectedAt: new Date(),
      affectedSystems: input.affectedSystems || [],
      affectedUsers: input.affectedUsers || 0,
      timeline: [{
        timestamp: new Date(),
        description: 'Incident declared',
        author: 'System'
      }],
      actions: []
    };

    this.incidents.set(incident.id, incident);

    // ดำเนินการตาม playbook
    await this.executePlaybook(incident);

    return incident;
  }

  private async executePlaybook(incident: SecurityIncident): Promise<void> {
    // 1. Notify team
    await this.notifyTeam(incident);

    // 2. Execute immediate containment
    await this.executeImmediateContainment(incident);

    // 3. Start investigation
    await this.initiateInvestigation(incident);

    // 4. Set up war room
    if (incident.severity === 'P1' || incident.severity === 'P2') {
      await this.setupWarRoom(incident);
    }
  }

  private async notifyTeam(incident: SecurityIncident): Promise<void> {
    const message = this.formatIncidentAlert(incident);

    // Slack notification
    await this.slack.postMessage({
      channel: '#security-incidents',
      text: message,
      attachments: [{
        color: this.getSeverityColor(incident.severity),
        fields: [
          { title: 'Incident ID', value: incident.id, short: true },
          { title: 'Severity', value: incident.severity, short: true },
          { title: 'Type', value: incident.type, short: true },
          { title: 'Affected Systems', value: incident.affectedSystems.join(', '), short: false }
        ]
      }]
    });

    // PagerDuty for P1/P2
    if (incident.severity === 'P1' || incident.severity === 'P2') {
      await this.pagerDuty.createIncident({
        title: `[${incident.severity}] ${incident.title}`,
        severity: incident.severity === 'P1' ? 'critical' : 'error',
        body: incident.description,
        details: {
          incidentId: incident.id,
          affectedSystems: incident.affectedSystems
        }
      });
    }
  }

  private async executeImmediateContainment(incident: SecurityIncident): Promise<void> {
    const containmentActions: Record<string, () => Promise<void>> = {
      'DATA_BREACH': async () => {
        await this.isolateAffectedSystems(incident.affectedSystems);
        await this.revokeCompromisedCredentials();
        await this.enableEnhancedLogging();
      },
      'ACCOUNT_COMPROMISE': async () => {
        await this.lockCompromisedAccounts();
        await this.forcePasswordReset();
        await this.invalidateAllSessions();
      },
      'DDoS_ATTACK': async () => {
        await this.enableCloudflareUnderAttackMode();
        await this.increaseRateLimits();
        await this.blockMaliciousIPs();
      },
      'MALWARE': async () => {
        await this.quarantineAffectedSystems(incident.affectedSystems);
        await this.blockOutboundConnections();
      }
    };

    const action = containmentActions[incident.type];
    if (action) {
      await action();
      this.addToTimeline(incident.id, 'Immediate containment actions executed', 'System');
    }
  }

  async updateIncidentStatus(incidentId: string, status: IncidentStatus, note: string): Promise<void> {
    const incident = this.incidents.get(incidentId);
    if (!incident) throw new Error(`Incident ${incidentId} not found`);

    incident.status = status;

    if (status === 'CONTAINED') incident.containedAt = new Date();
    if (status === 'RESOLVED') incident.resolvedAt = new Date();

    this.addToTimeline(incidentId, `Status changed to ${status}: ${note}`, 'Analyst');

    await this.slack.postMessage({
      channel: '#security-incidents',
      text: `📊 Incident ${incidentId} status updated to: *${status}*\n${note}`
    });
  }

  async conductPostMortem(incidentId: string): Promise<string> {
    const incident = this.incidents.get(incidentId);
    if (!incident) throw new Error(`Incident ${incidentId} not found`);

    const duration = incident.resolvedAt 
      ? (incident.resolvedAt.getTime() - incident.detectedAt.getTime()) / (1000 * 60)
      : 0;

    return `
# Incident Post-Mortem: ${incident.id}

## Incident Overview
- **Incident ID:** ${incident.id}
- **Title:** ${incident.title}
- **Severity:** ${incident.severity}
- **Duration:** ${Math.round(duration)} minutes
- **Affected Systems:** ${incident.affectedSystems.join(', ')}
- **Affected Users:** ${incident.affectedUsers}

## Timeline
${incident.timeline.map(e => `- **${e.timestamp.toISOString()}** - ${e.description} (${e.author})`).join('\n')}

## Root Cause Analysis
[To be filled by security team]

## Impact Assessment
[Describe business and technical impact]

## What Went Well
[List things that worked during response]

## What Could Be Improved
[List areas for improvement]

## Action Items
| Action | Owner | Due Date | Status |
|--------|-------|----------|--------|
${incident.actions.map(a => `| ${a.action} | ${a.assignee} | ${a.dueDate.toISOString().split('T')[0]} | ${a.status} |`).join('\n')}

## Lessons Learned
[Key takeaways from this incident]
    `.trim();
  }

  private addToTimeline(incidentId: string, description: string, author: string): void {
    const incident = this.incidents.get(incidentId);
    if (incident) {
      incident.timeline.push({
        timestamp: new Date(),
        description,
        author
      });
    }
  }

  private getSeverityColor(severity: IncidentSeverity): string {
    const colors = { P1: '#FF0000', P2: '#FF6600', P3: '#FFAA00', P4: '#00AA00' };
    return colors[severity];
  }

  private formatIncidentAlert(incident: SecurityIncident): string {
    return `🚨 *Security Incident Declared* 🚨\n*ID:* ${incident.id}\n*Severity:* ${incident.severity}\n*Title:* ${incident.title}`;
  }

  private async initiateInvestigation(incident: SecurityIncident): Promise<void> {}
  private async setupWarRoom(incident: SecurityIncident): Promise<void> {}
  private async isolateAffectedSystems(systems: string[]): Promise<void> {}
  private async revokeCompromisedCredentials(): Promise<void> {}
  private async enableEnhancedLogging(): Promise<void> {}
  private async lockCompromisedAccounts(): Promise<void> {}
  private async forcePasswordReset(): Promise<void> {}
  private async invalidateAllSessions(): Promise<void> {}
  private async enableCloudflareUnderAttackMode(): Promise<void> {}
  private async increaseRateLimits(): Promise<void> {}
  private async blockMaliciousIPs(): Promise<void> {}
  private async quarantineAffectedSystems(systems: string[]): Promise<void> {}
  private async blockOutboundConnections(): Promise<void> {}
}
```

---

## 11. Bug Bounty Program Setup

### 11.1 Bug Bounty Program Policy

```markdown
# LINE Bot Bug Bounty Program

## Program Overview
เรายินดีรับรายงานช่องโหว่ด้านความปลอดภัยจากนักวิจัยความปลอดภัยทั่วโลก

## Scope

### In Scope
- LINE Bot Webhook endpoints
- LIFF Applications
- LINE Login implementation
- API endpoints (api.yourcompany.com/*)
- Mobile apps ที่เกี่ยวข้องกับ LINE integration
- Admin dashboard

### Out of Scope
- Social engineering attacks
- Physical attacks
- Volumetric DDoS
- Issues ใน third-party services
- Self-XSS
- CSRF ที่ไม่มี impact

## Rewards

| Severity | Reward Range |
|----------|-------------|
| Critical | $5,000 - $10,000 |
| High | $1,000 - $5,000 |
| Medium | $250 - $1,000 |
| Low | $50 - $250 |

## Severity Criteria

### Critical
- Remote Code Execution
- SQL/NoSQL Injection ที่ expose ข้อมูลทั้งหมด
- Authentication Bypass
- Unauthorized access to admin functions
- Mass data exposure (PII/LINE User IDs)

### High
- XSS ที่ steal session tokens
- CSRF ที่ affect sensitive operations
- IDOR ที่ expose user data
- Privilege Escalation
- LINE Signature Bypass

### Medium
- XSS แบบ Stored (non-critical pages)
- Information Disclosure (sensitive data)
- CSRF (non-sensitive operations)
- Open Redirect

### Low
- XSS แบบ Reflected
- Missing security headers
- Information disclosure (non-sensitive)

## Responsible Disclosure Process
1. Submit report ที่ security@yourcompany.com
2. เราจะ acknowledge ภายใน 24 ชั่วโมง
3. Initial assessment ภายใน 7 วัน
4. Fix timeline ขึ้นอยู่กับ severity
5. Reward จ่ายหลัง fix ได้รับการ verify

## Rules
- ห้าม access ข้อมูลของ user จริง
- ห้ามทดสอบกับ production environment โดยไม่ได้รับอนุญาต
- ห้าม disclose ก่อนได้รับการ fix
- ห้ามทดสอบโดยไม่ได้รับ token จาก test account ของเรา
```

---

## 12. Complete Security Middleware Stack

### 12.1 Security Middleware Integration

```typescript
// middleware/SecurityStack.ts
import express, { Express } from 'express';
import helmet from 'helmet';
import cors from 'cors';
import rateLimit from 'express-rate-limit';
import { ZeroTrustMiddleware } from './ZeroTrustMiddleware';
import { LineSignatureVerifier } from '../security/SignatureVerifier';
import { AdvancedRateLimiter } from '../security/RateLimiter';
import { RequestSanitizer } from '../security/RequestSanitizer';
import { AuditLogger } from '../logging/AuditLogger';

export function setupSecurityMiddleware(app: Express): void {
  // 1. Helmet - Security Headers
  app.use(helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "'nonce-{nonce}'", 'https://static.line-scdn.net'],
        styleSrc: ["'self'", "'unsafe-inline'"],
        imgSrc: ["'self'", 'data:', 'https://profile.line-scdn.net'],
        connectSrc: ["'self'", 'https://api.line.me'],
        fontSrc: ["'self'"],
        objectSrc: ["'none'"],
        mediaSrc: ["'self'"],
        frameSrc: ["'none'"],
        upgradeInsecureRequests: [],
      },
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true
    },
    noSniff: true,
    frameguard: { action: 'deny' },
    xssFilter: true,
    referrerPolicy: { policy: 'strict-origin-when-cross-origin' }
  }));

  // 2. CORS
  app.use(cors({
    origin: (origin, callback) => {
      const allowedOrigins = process.env.ALLOWED_ORIGINS?.split(',') || [];
      if (!origin || allowedOrigins.includes(origin)) {
        callback(null, true);
      } else {
        callback(new Error('Not allowed by CORS'));
      }
    },
    methods: ['GET', 'POST', 'PUT', 'DELETE', 'PATCH'],
    allowedHeaders: ['Content-Type', 'Authorization', 'X-API-Key'],
    credentials: true,
    maxAge: 86400
  }));

  // 3. Request Size Limiting
  app.use(express.json({ limit: '10kb' }));
  app.use(express.urlencoded({ extended: true, limit: '10kb' }));

  // 4. Global Rate Limiting
  const globalLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 นาที
    max: 100,
    standardHeaders: true,
    legacyHeaders: false,
    message: { error: 'Too many requests', retryAfter: 900 }
  });
  app.use('/api/', globalLimiter);

  // 5. Request Sanitization
  const sanitizer = new RequestSanitizer();
  app.use(sanitizer.middleware());

  // 6. Request ID
  app.use((req, res, next) => {
    req.headers['x-request-id'] = req.headers['x-request-id'] || 
      `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
    res.setHeader('X-Request-ID', req.headers['x-request-id']);
    next();
  });

  // 7. Audit Logging
  const auditLogger = new AuditLogger();
  app.use(auditLogger.requestLogger());

  // 8. Webhook-specific security (applied to /webhook route only)
  const signatureVerifier = new LineSignatureVerifier(process.env.LINE_CHANNEL_SECRET!);
  app.use('/webhook', signatureVerifier.middleware());

  // 9. API Authentication
  app.use('/api/', async (req, res, next) => {
    // Skip auth for public endpoints
    const publicPaths = ['/api/health', '/api/version'];
    if (publicPaths.includes(req.path)) return next();

    const authHeader = req.headers.authorization;
    if (!authHeader) {
      res.status(401).json({ error: 'Authentication required' });
      return;
    }
    next();
  });
}

// deployment/security-checklist.ts
export const SECURITY_CHECKLIST = {
  preDeployment: [
    '✅ All dependencies updated and vulnerability-free',
    '✅ SAST scan passed',
    '✅ DAST scan passed',
    '✅ Container scan passed',
    '✅ Secret scan passed',
    '✅ Security headers configured',
    '✅ TLS 1.3 enabled',
    '✅ Certificate valid',
    '✅ Rate limiting configured',
    '✅ Input validation implemented',
    '✅ Error handling does not expose sensitive data',
    '✅ Logging configured without PII',
    '✅ Backup encryption enabled',
    '✅ Access controls reviewed'
  ],
  postDeployment: [
    '✅ Penetration test conducted',
    '✅ Security monitoring enabled',
    '✅ Alerting configured',
    '✅ Incident response team notified',
    '✅ Rollback plan documented'
  ]
};
```

---

## สรุป

สถาปัตยกรรมความปลอดภัยสำหรับ LINE Bot ที่สมบูรณ์ต้องประกอบด้วย:

1. **Zero-Trust Architecture** - ไม่เชื่อใจใครโดยปริยาย ตรวจสอบทุก request
2. **OAuth 2.0 + OpenID Connect** - การ authenticate ที่มาตรฐาน
3. **Multi-layer Authentication** - Signature verification, JWT, API Keys
4. **Automated Secret Rotation** - หมุนเวียน credentials อัตโนมัติ
5. **Encryption at All Levels** - ทั้ง in-transit และ at-rest
6. **CI/CD Security Integration** - SAST, DAST, dependency scan
7. **Vulnerability Management** - ติดตามและจัดการ vulnerabilities
8. **Incident Response** - แผนรับมือเหตุการณ์ความปลอดภัย
9. **Bug Bounty Program** - Community-driven security testing

การรักษาความปลอดภัยเป็น **ongoing process** ไม่ใช่แค่การ setup ครั้งเดียว ต้องมีการ review, update และ test อย่างสม่ำเสมอ
