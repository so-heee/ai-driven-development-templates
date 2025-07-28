# セキュリティガイド

## 認証・認可

### JWT実装
```typescript
// JWT トークン管理
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';

interface TokenPayload {
  userId: string;
  email: string;
  role: string;
  iat?: number;
  exp?: number;
}

class AuthService {
  private readonly JWT_SECRET = process.env.JWT_SECRET!;
  private readonly JWT_EXPIRES_IN = '15m';
  private readonly REFRESH_TOKEN_EXPIRES_IN = '7d';

  async generateTokens(user: User): Promise<{ accessToken: string; refreshToken: string }> {
    const payload: TokenPayload = {
      userId: user.id,
      email: user.email,
      role: user.role,
    };

    const accessToken = jwt.sign(payload, this.JWT_SECRET, {
      expiresIn: this.JWT_EXPIRES_IN,
      issuer: 'myapp',
      audience: 'myapp-users',
    });

    const refreshToken = jwt.sign(
      { userId: user.id, type: 'refresh' },
      this.JWT_SECRET,
      { expiresIn: this.REFRESH_TOKEN_EXPIRES_IN }
    );

    // リフレッシュトークンをデータベースに保存
    await this.saveRefreshToken(user.id, refreshToken);

    return { accessToken, refreshToken };
  }

  async verifyToken(token: string): Promise<TokenPayload> {
    try {
      const decoded = jwt.verify(token, this.JWT_SECRET, {
        issuer: 'myapp',
        audience: 'myapp-users',
      }) as TokenPayload;
      
      return decoded;
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        throw new Error('Token expired');
      }
      throw new Error('Invalid token');
    }
  }

  async hashPassword(password: string): Promise<string> {
    const saltRounds = 12;
    return bcrypt.hash(password, saltRounds);
  }

  async comparePassword(password: string, hash: string): Promise<boolean> {
    return bcrypt.compare(password, hash);
  }

  private async saveRefreshToken(userId: string, token: string): Promise<void> {
    const hashedToken = await this.hashPassword(token);
    await db.refreshTokens.create({
      userId,
      token: hashedToken,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 7日後
    });
  }
}
```

### OAuth 2.0 実装
```typescript
// OAuth プロバイダー設定
class OAuthService {
  private readonly providers = {
    google: {
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      redirectUri: process.env.GOOGLE_REDIRECT_URI!,
      scope: 'openid email profile',
    },
    github: {
      clientId: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
      redirectUri: process.env.GITHUB_REDIRECT_URI!,
      scope: 'user:email',
    },
  };

  getAuthUrl(provider: keyof typeof this.providers, state: string): string {
    const config = this.providers[provider];
    const params = new URLSearchParams({
      client_id: config.clientId,
      redirect_uri: config.redirectUri,
      scope: config.scope,
      state,
      response_type: 'code',
    });

    const baseUrls = {
      google: 'https://accounts.google.com/o/oauth2/auth',
      github: 'https://github.com/login/oauth/authorize',
    };

    return `${baseUrls[provider]}?${params.toString()}`;
  }

  async exchangeCodeForToken(
    provider: keyof typeof this.providers,
    code: string
  ): Promise<{ accessToken: string; user: any }> {
    const config = this.providers[provider];
    
    // トークン交換
    const tokenResponse = await fetch(this.getTokenUrl(provider), {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded',
        Accept: 'application/json',
      },
      body: new URLSearchParams({
        client_id: config.clientId,
        client_secret: config.clientSecret,
        code,
        redirect_uri: config.redirectUri,
        grant_type: 'authorization_code',
      }),
    });

    const tokenData = await tokenResponse.json();

    // ユーザー情報取得
    const user = await this.fetchUserInfo(provider, tokenData.access_token);

    return { accessToken: tokenData.access_token, user };
  }

  private getTokenUrl(provider: keyof typeof this.providers): string {
    const urls = {
      google: 'https://oauth2.googleapis.com/token',
      github: 'https://github.com/login/oauth/access_token',
    };
    return urls[provider];
  }

  private async fetchUserInfo(provider: keyof typeof this.providers, accessToken: string): Promise<any> {
    const urls = {
      google: 'https://www.googleapis.com/oauth2/v2/userinfo',
      github: 'https://api.github.com/user',
    };

    const response = await fetch(urls[provider], {
      headers: {
        Authorization: `Bearer ${accessToken}`,
      },
    });

    return response.json();
  }
}
```

## 入力検証・サニタイゼーション

### バリデーション実装
```typescript
import Joi from 'joi';
import DOMPurify from 'dompurify';
import validator from 'validator';

// Joi スキーマ定義
const schemas = {
  userRegistration: Joi.object({
    email: Joi.string()
      .email({ tlds: { allow: false } })
      .required()
      .messages({
        'string.email': 'メールアドレスの形式が正しくありません',
        'any.required': 'メールアドレスは必須です',
      }),
    
    password: Joi.string()
      .min(8)
      .max(128)
      .pattern(new RegExp('^(?=.*[a-z])(?=.*[A-Z])(?=.*\\d)(?=.*[@$!%*?&])[A-Za-z\\d@$!%*?&]'))
      .required()
      .messages({
        'string.min': 'パスワードは8文字以上である必要があります',
        'string.pattern.base': 'パスワードは大文字、小文字、数字、特殊文字を含む必要があります',
      }),
    
    name: Joi.string()
      .min(1)
      .max(100)
      .trim()
      .required(),
    
    age: Joi.number()
      .integer()
      .min(13)
      .max(120)
      .optional(),
  }),

  userUpdate: Joi.object({
    name: Joi.string().min(1).max(100).trim().optional(),
    bio: Joi.string().max(500).optional(),
    website: Joi.string().uri().optional(),
  }),
};

// バリデーションミドルウェア
const validateRequest = (schema: Joi.ObjectSchema) => {
  return (req: Request, res: Response, next: NextFunction) => {
    const { error, value } = schema.validate(req.body, {
      abortEarly: false, // すべてのエラーを収集
      stripUnknown: true, // 不明なフィールドを削除
    });

    if (error) {
      const errors = error.details.map(detail => ({
        field: detail.path.join('.'),
        message: detail.message,
        value: detail.context?.value,
      }));

      return res.status(400).json({
        error: 'Validation failed',
        details: errors,
      });
    }

    req.body = value; // 検証済みの値を使用
    next();
  };
};

// サニタイゼーション
class SanitizationService {
  static sanitizeHtml(input: string): string {
    return DOMPurify.sanitize(input, {
      ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'p', 'br'],
      ALLOWED_ATTR: [],
    });
  }

  static sanitizeString(input: string): string {
    return validator.escape(input.trim());
  }

  static sanitizeEmail(input: string): string {
    return validator.normalizeEmail(input, {
      gmail_lowercase: true,
      gmail_remove_dots: false,
      outlookdotcom_lowercase: true,
    }) || '';
  }

  static sanitizeUrl(input: string): string | null {
    if (!validator.isURL(input, { protocols: ['http', 'https'] })) {
      return null;
    }
    return input;
  }

  static sanitizeUser(input: any): any {
    return {
      email: this.sanitizeEmail(input.email),
      name: this.sanitizeString(input.name),
      bio: input.bio ? this.sanitizeHtml(input.bio) : undefined,
      website: input.website ? this.sanitizeUrl(input.website) : undefined,
    };
  }
}
```

### CSRF対策
```typescript
import csrf from 'csurf';
import cookieParser from 'cookie-parser';

// CSRF トークン設定
const csrfProtection = csrf({
  cookie: {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'strict',
  },
});

app.use(cookieParser());
app.use(csrfProtection);

// CSRF トークンをクライアントに提供
app.get('/api/csrf-token', (req, res) => {
  res.json({ csrfToken: req.csrfToken() });
});

// フロントエンド側でのCSRFトークン管理
class CSRFService {
  private token: string | null = null;

  async getToken(): Promise<string> {
    if (!this.token) {
      const response = await fetch('/api/csrf-token', {
        credentials: 'include',
      });
      const data = await response.json();
      this.token = data.csrfToken;
    }
    return this.token;
  }

  async makeRequest(url: string, options: RequestInit = {}): Promise<Response> {
    const token = await this.getToken();
    
    return fetch(url, {
      ...options,
      credentials: 'include',
      headers: {
        ...options.headers,
        'X-CSRF-Token': token,
      },
    });
  }

  clearToken(): void {
    this.token = null;
  }
}
```

## セキュアなAPI設計

### レート制限
```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import Redis from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

// 基本的なレート制限
const basicLimiter = rateLimit({
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
  windowMs: 15 * 60 * 1000, // 15分
  max: 100, // リクエスト数
  message: {
    error: 'Too many requests',
    retryAfter: 15 * 60, // 秒
  },
  standardHeaders: true,
  legacyHeaders: false,
});

// 認証エンドポイント用の厳しい制限
const authLimiter = rateLimit({
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
  }),
  windowMs: 15 * 60 * 1000,
  max: 5, // 15分間で5回まで
  skipSuccessfulRequests: true, // 成功したリクエストはカウントしない
  keyGenerator: (req) => {
    return req.ip + ':' + (req.body?.email || 'unknown');
  },
});

// アダプティブレート制限
class AdaptiveRateLimit {
  private baseLimits = new Map<string, number>();

  constructor() {
    this.baseLimits.set('/', 100);
    this.baseLimits.set('/api/search', 50);
    this.baseLimits.set('/api/upload', 10);
  }

  createLimiter(endpoint: string) {
    const baseLimit = this.baseLimits.get(endpoint) || 60;
    
    return rateLimit({
      windowMs: 15 * 60 * 1000,
      max: (req) => {
        // 認証ユーザーには高い制限を適用
        if (req.user?.isPremium) {
          return baseLimit * 5;
        }
        if (req.user) {
          return baseLimit * 2;
        }
        return baseLimit;
      },
      keyGenerator: (req) => {
        // 認証ユーザーはユーザーIDベース、未認証はIPベース
        return req.user?.id || req.ip;
      },
    });
  }
}
```

### セキュリティヘッダー
```typescript
import helmet from 'helmet';

// セキュリティヘッダーの設定
app.use(helmet({
  // XSS Protection
  xssFilter: true,
  
  // Content Type Options
  noSniff: true,
  
  // Frame Options
  frameguard: { action: 'deny' },
  
  // HSTS (HTTPS Strict Transport Security)
  hsts: {
    maxAge: 31536000, // 1年
    includeSubDomains: true,
    preload: true,
  },
  
  // Content Security Policy
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      styleSrc: [
        "'self'",
        "'unsafe-inline'", // インラインスタイル許可（最小限に）
        'https://fonts.googleapis.com',
      ],
      scriptSrc: [
        "'self'",
        "'unsafe-inline'", // 本番では避けるべき
        'https://cdn.jsdelivr.net',
      ],
      imgSrc: [
        "'self'",
        'data:',
        'https://images.unsplash.com',
      ],
      connectSrc: [
        "'self'",
        'https://api.example.com',
      ],
      fontSrc: [
        "'self'",
        'https://fonts.gstatic.com',
      ],
      upgradeInsecureRequests: [],
    },
  },
  
  // Referrer Policy
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },
}));

// カスタムセキュリティミドルウェア
const securityMiddleware = (req: Request, res: Response, next: NextFunction) => {
  // Remove sensitive headers
  res.removeHeader('X-Powered-By');
  res.removeHeader('Server');
  
  // Add custom security headers
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  
  // Prevent caching of sensitive endpoints
  if (req.path.startsWith('/api/admin') || req.path.startsWith('/api/user')) {
    res.setHeader('Cache-Control', 'no-store, no-cache, must-revalidate, private');
    res.setHeader('Pragma', 'no-cache');
    res.setHeader('Expires', '0');
  }
  
  next();
};
```

## データ暗号化

### 暗号化サービス
```typescript
import crypto from 'crypto';

class EncryptionService {
  private readonly algorithm = 'aes-256-gcm';
  private readonly keyLength = 32; // 256 bits
  private readonly ivLength = 16; // 128 bits
  private readonly tagLength = 16; // 128 bits
  private readonly saltLength = 64; // 512 bits

  private getKey(password: string, salt: Buffer): Buffer {
    return crypto.pbkdf2Sync(password, salt, 100000, this.keyLength, 'sha256');
  }

  encrypt(plaintext: string, password: string): string {
    const salt = crypto.randomBytes(this.saltLength);
    const key = this.getKey(password, salt);
    const iv = crypto.randomBytes(this.ivLength);
    
    const cipher = crypto.createCipher(this.algorithm, key);
    cipher.setAAD(salt); // Additional Authenticated Data
    
    let encrypted = cipher.update(plaintext, 'utf8', 'hex');
    encrypted += cipher.final('hex');
    
    const tag = cipher.getAuthTag();
    
    // salt + iv + tag + encrypted data
    return salt.toString('hex') + 
           iv.toString('hex') + 
           tag.toString('hex') + 
           encrypted;
  }

  decrypt(encryptedData: string, password: string): string {
    const saltHex = encryptedData.slice(0, this.saltLength * 2);
    const ivHex = encryptedData.slice(this.saltLength * 2, (this.saltLength + this.ivLength) * 2);
    const tagHex = encryptedData.slice((this.saltLength + this.ivLength) * 2, (this.saltLength + this.ivLength + this.tagLength) * 2);
    const encryptedHex = encryptedData.slice((this.saltLength + this.ivLength + this.tagLength) * 2);
    
    const salt = Buffer.from(saltHex, 'hex');
    const iv = Buffer.from(ivHex, 'hex');
    const tag = Buffer.from(tagHex, 'hex');
    const key = this.getKey(password, salt);
    
    const decipher = crypto.createDecipher(this.algorithm, key);
    decipher.setAAD(salt);
    decipher.setAuthTag(tag);
    
    let decrypted = decipher.update(encryptedHex, 'hex', 'utf8');
    decrypted += decipher.final('utf8');
    
    return decrypted;
  }

  generateKeyPair(): { publicKey: string; privateKey: string } {
    const { publicKey, privateKey } = crypto.generateKeyPairSync('rsa', {
      modulusLength: 2048,
      publicKeyEncoding: {
        type: 'spki',
        format: 'pem',
      },
      privateKeyEncoding: {
        type: 'pkcs8',
        format: 'pem',
      },
    });

    return { publicKey, privateKey };
  }

  encryptWithPublicKey(data: string, publicKey: string): string {
    const buffer = Buffer.from(data, 'utf8');
    const encrypted = crypto.publicEncrypt(publicKey, buffer);
    return encrypted.toString('base64');
  }

  decryptWithPrivateKey(encryptedData: string, privateKey: string): string {
    const buffer = Buffer.from(encryptedData, 'base64');
    const decrypted = crypto.privateDecrypt(privateKey, buffer);
    return decrypted.toString('utf8');
  }
}
```

### 機密データの管理
```typescript
// 環境変数の暗号化管理
class SecretManager {
  private encryptionService = new EncryptionService();
  private masterKey: string;

  constructor() {
    this.masterKey = process.env.MASTER_KEY || this.generateMasterKey();
  }

  private generateMasterKey(): string {
    const key = crypto.randomBytes(32).toString('hex');
    console.warn('Generated new master key. Store securely:', key);
    return key;
  }

  encryptSecret(value: string): string {
    return this.encryptionService.encrypt(value, this.masterKey);
  }

  decryptSecret(encryptedValue: string): string {
    return this.encryptionService.decrypt(encryptedValue, this.masterKey);
  }

  // データベース内の機密データ暗号化
  async storeUserSensitiveData(userId: string, data: any): Promise<void> {
    const encryptedData = {
      ssn: data.ssn ? this.encryptSecret(data.ssn) : null,
      creditCard: data.creditCard ? this.encryptSecret(data.creditCard) : null,
      bankAccount: data.bankAccount ? this.encryptSecret(data.bankAccount) : null,
    };

    await db.userSensitiveData.upsert({
      where: { userId },
      update: encryptedData,
      create: { userId, ...encryptedData },
    });
  }

  async getUserSensitiveData(userId: string): Promise<any> {
    const data = await db.userSensitiveData.findUnique({
      where: { userId },
    });

    if (!data) return null;

    return {
      ssn: data.ssn ? this.decryptSecret(data.ssn) : null,
      creditCard: data.creditCard ? this.decryptSecret(data.creditCard) : null,
      bankAccount: data.bankAccount ? this.decryptSecret(data.bankAccount) : null,
    };
  }
}
```

## セキュリティテスト

### 脆弱性テストの自動化
```typescript
// セキュリティテストスイート
describe('Security Tests', () => {
  describe('Authentication', () => {
    it('should reject invalid JWT tokens', async () => {
      const invalidToken = 'invalid.jwt.token';
      
      const response = await request(app)
        .get('/api/protected')
        .set('Authorization', `Bearer ${invalidToken}`)
        .expect(401);
      
      expect(response.body.error).toBe('Invalid token');
    });

    it('should prevent brute force attacks', async () => {
      const email = 'test@example.com';
      
      // 5回失敗した後はブロックされるはず
      for (let i = 0; i < 6; i++) {
        const response = await request(app)
          .post('/api/auth/login')
          .send({ email, password: 'wrong-password' });
        
        if (i < 5) {
          expect(response.status).toBe(401);
        } else {
          expect(response.status).toBe(429); // Too Many Requests
        }
      }
    });
  });

  describe('Input Validation', () => {
    it('should prevent XSS attacks', async () => {
      const maliciousScript = '<script>alert("XSS")</script>';
      
      const response = await request(app)
        .post('/api/users')
        .send({ name: maliciousScript })
        .set('Authorization', `Bearer ${validToken}`)
        .expect(400);
      
      expect(response.body.error).toContain('Validation failed');
    });

    it('should prevent SQL injection', async () => {
      const sqlInjection = "'; DROP TABLE users; --";
      
      const response = await request(app)
        .get(`/api/users/search?q=${encodeURIComponent(sqlInjection)}`)
        .set('Authorization', `Bearer ${validToken}`)
        .expect(400);
    });
  });

  describe('CSRF Protection', () => {
    it('should require CSRF token for state-changing operations', async () => {
      const response = await request(app)
        .post('/api/users')
        .send({ name: 'Test User' })
        .set('Authorization', `Bearer ${validToken}`)
        .expect(403);
      
      expect(response.body.error).toContain('CSRF');
    });
  });
});

// ペネトレーションテストツールとの連携
class SecurityScanRunner {
  async runOWASPZAP(baseUrl: string): Promise<void> {
    const zapClient = new ZapClient({
      proxy: 'http://localhost:8080',
    });

    try {
      // スパイダー実行
      await zapClient.spider.scan(baseUrl);
      
      // アクティブスキャン実行
      await zapClient.ascan.scan(baseUrl);
      
      // レポート生成
      const report = await zapClient.core.htmlreport();
      
      // 重要度の高い脆弱性があれば失敗
      const highRiskAlerts = await zapClient.core.alerts('High');
      if (highRiskAlerts.length > 0) {
        throw new Error(`High risk vulnerabilities found: ${highRiskAlerts.length}`);
      }
      
    } catch (error) {
      console.error('Security scan failed:', error);
      throw error;
    }
  }
}
```

## セキュリティ監視

### セキュリティイベントログ
```typescript
// セキュリティイベント管理
enum SecurityEventType {
  LOGIN_SUCCESS = 'login_success',
  LOGIN_FAILURE = 'login_failure',
  PASSWORD_RESET = 'password_reset',
  SUSPICIOUS_ACTIVITY = 'suspicious_activity',
  DATA_ACCESS = 'data_access',
  PRIVILEGE_ESCALATION = 'privilege_escalation',
}

interface SecurityEvent {
  type: SecurityEventType;
  userId?: string;
  ip: string;
  userAgent: string;
  timestamp: Date;
  details: Record<string, any>;
  riskLevel: 'low' | 'medium' | 'high' | 'critical';
}

class SecurityLogger {
  async logEvent(event: SecurityEvent): Promise<void> {
    // データベースへの保存
    await db.securityEvents.create({ data: event });
    
    // 高リスクイベントの即座の通知
    if (event.riskLevel === 'critical' || event.riskLevel === 'high') {
      await this.sendAlert(event);
    }
    
    // メトリクス更新
    securityMetrics.eventCounter.inc({
      type: event.type,
      risk_level: event.riskLevel,
    });
  }

  private async sendAlert(event: SecurityEvent): Promise<void> {
    const alertMessage = {
      title: `Security Alert: ${event.type}`,
      description: `High risk security event detected from IP ${event.ip}`,
      details: event.details,
      timestamp: event.timestamp,
    };

    // Slack通知
    await this.notificationService.sendSlack(alertMessage);
    
    // メール通知
    await this.notificationService.sendEmail(
      'security@company.com',
      'Security Alert',
      JSON.stringify(alertMessage, null, 2)
    );
  }

  async analyzeAnomalies(): Promise<void> {
    // 異常なログインパターンの検出
    const suspiciousLogins = await db.securityEvents.findMany({
      where: {
        type: SecurityEventType.LOGIN_FAILURE,
        timestamp: {
          gte: new Date(Date.now() - 60 * 60 * 1000), // 1時間以内
        },
      },
      group: ['ip'],
      having: db.raw('COUNT(*) > 10'), // 1時間に10回以上の失敗
    });

    for (const login of suspiciousLogins) {
      await this.logEvent({
        type: SecurityEventType.SUSPICIOUS_ACTIVITY,
        ip: login.ip,
        userAgent: login.userAgent,
        timestamp: new Date(),
        details: { 
          reason: 'Multiple login failures',
          failureCount: login.count,
        },
        riskLevel: 'high',
      });
    }
  }
}
```

## インシデント対応

### セキュリティインシデント対応計画
```typescript
// インシデント対応自動化
class SecurityIncidentResponse {
  async handleSecurityIncident(incident: SecurityIncident): Promise<void> {
    // インシデントの分類
    const severity = this.classifyIncident(incident);
    
    // 即座の対応
    await this.immediateResponse(incident, severity);
    
    // 詳細調査の開始
    await this.startInvestigation(incident);
    
    // ステークホルダーへの通知
    await this.notifyStakeholders(incident, severity);
  }

  private classifyIncident(incident: SecurityIncident): 'low' | 'medium' | 'high' | 'critical' {
    const indicators = {
      dataExfiltration: 'critical',
      unauthorizedAccess: 'high',
      bruteForceAttack: 'medium',
      suspiciousActivity: 'low',
    };

    return indicators[incident.type] || 'low';
  }

  private async immediateResponse(incident: SecurityIncident, severity: string): Promise<void> {
    switch (severity) {
      case 'critical':
        // システムの部分的停止
        await this.isolateAffectedSystems(incident);
        await this.revokeUserSessions(incident.affectedUsers);
        break;
        
      case 'high':
        // 影響ユーザーのセッション無効化
        await this.revokeUserSessions(incident.affectedUsers);
        // 追加監視の開始
        await this.enhanceMonitoring(incident);
        break;
        
      case 'medium':
        // レート制限の強化
        await this.increaseRateLimit(incident.sourceIPs);
        break;
    }
  }

  private async isolateAffectedSystems(incident: SecurityIncident): Promise<void> {
    // 影響を受けたサービスを隔離
    for (const service of incident.affectedServices) {
      await this.k8sClient.patchDeployment(service, {
        spec: {
          replicas: 0, // スケールダウン
        },
      });
    }
    
    // WAFルールの追加
    await this.addWAFRule({
      action: 'block',
      conditions: {
        ip: incident.sourceIPs,
      },
    });
  }

  private async generateIncidentReport(incident: SecurityIncident): Promise<string> {
    const report = {
      incidentId: incident.id,
      detectedAt: incident.timestamp,
      severity: incident.severity,
      affectedSystems: incident.affectedSystems,
      timeline: incident.timeline,
      rootCause: incident.rootCause,
      impact: incident.impact,
      remediation: incident.remediation,
      lessonsLearned: incident.lessonsLearned,
    };

    return JSON.stringify(report, null, 2);
  }
}
```

## コンプライアンス

### GDPR対応
```typescript
// GDPR データ管理
class GDPRCompliance {
  async handleDataSubjectRequest(request: DataSubjectRequest): Promise<void> {
    switch (request.type) {
      case 'access':
        await this.handleAccessRequest(request);
        break;
      case 'portability':
        await this.handlePortabilityRequest(request);
        break;
      case 'erasure':
        await this.handleErasureRequest(request);
        break;
      case 'rectification':
        await this.handleRectificationRequest(request);
        break;
    }
  }

  private async handleAccessRequest(request: DataSubjectRequest): Promise<void> {
    const userData = await this.collectUserData(request.userId);
    const report = this.generateDataReport(userData);
    
    await this.sendDataReport(request.email, report);
  }

  private async collectUserData(userId: string): Promise<any> {
    const [
      profile,
      orders,
      sessions,
      communications,
    ] = await Promise.all([
      db.users.findUnique({ where: { id: userId } }),
      db.orders.findMany({ where: { userId } }),
      db.sessions.findMany({ where: { userId } }),
      db.communications.findMany({ where: { userId } }),
    ]);

    return {
      profile,
      orders,
      sessions: sessions.map(s => ({ 
        id: s.id, 
        createdAt: s.createdAt,
        ip: this.anonymizeIP(s.ip),
      })),
      communications,
    };
  }

  private async handleErasureRequest(request: DataSubjectRequest): Promise<void> {
    // データの完全削除
    await db.$transaction(async (tx) => {
      await tx.sessions.deleteMany({ where: { userId: request.userId } });
      await tx.communications.deleteMany({ where: { userId: request.userId } });
      await tx.orders.updateMany({
        where: { userId: request.userId },
        data: { 
          userDeleted: true,
          personalDataErased: true,
        },
      });
      await tx.users.delete({ where: { id: request.userId } });
    });

    // 外部システムからのデータ削除要求
    await this.requestExternalDataDeletion(request.userId);
  }

  private anonymizeIP(ip: string): string {
    const parts = ip.split('.');
    return `${parts[0]}.${parts[1]}.${parts[2]}.xxx`;
  }
}
```