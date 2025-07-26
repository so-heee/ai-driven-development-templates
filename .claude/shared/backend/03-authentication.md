# 認証・認可パターンガイド

## JWT (JSON Web Token) 認証

### JWT実装の基本パターン
```typescript
import jwt from 'jsonwebtoken';
import bcrypt from 'bcrypt';
import { Request, Response, NextFunction } from 'express';

interface JWTPayload {
  userId: string;
  email: string;
  role: string;
  iat?: number;
  exp?: number;
}

interface AuthenticatedRequest extends Request {
  user?: JWTPayload;
}

class AuthService {
  private readonly JWT_SECRET = process.env.JWT_SECRET!;
  private readonly JWT_REFRESH_SECRET = process.env.JWT_REFRESH_SECRET!;
  private readonly ACCESS_TOKEN_EXPIRY = '15m';
  private readonly REFRESH_TOKEN_EXPIRY = '7d';
  
  // パスワードハッシュ化
  async hashPassword(password: string): Promise<string> {
    const saltRounds = 12;
    return bcrypt.hash(password, saltRounds);
  }
  
  // パスワード検証
  async verifyPassword(password: string, hash: string): Promise<boolean> {
    return bcrypt.compare(password, hash);
  }
  
  // アクセストークン生成
  generateAccessToken(payload: Omit<JWTPayload, 'iat' | 'exp'>): string {
    return jwt.sign(payload, this.JWT_SECRET, {
      expiresIn: this.ACCESS_TOKEN_EXPIRY,
      issuer: 'your-app',
      audience: 'your-app-users'
    });
  }
  
  // リフレッシュトークン生成
  generateRefreshToken(userId: string): string {
    return jwt.sign(
      { userId, type: 'refresh' },
      this.JWT_REFRESH_SECRET,
      {
        expiresIn: this.REFRESH_TOKEN_EXPIRY,
        issuer: 'your-app',
        audience: 'your-app-users'
      }
    );
  }
  
  // トークン検証
  verifyAccessToken(token: string): JWTPayload {
    try {
      return jwt.verify(token, this.JWT_SECRET, {
        issuer: 'your-app',
        audience: 'your-app-users'
      }) as JWTPayload;
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        throw new Error('ACCESS_TOKEN_EXPIRED');
      }
      if (error instanceof jwt.JsonWebTokenError) {
        throw new Error('INVALID_ACCESS_TOKEN');
      }
      throw error;
    }
  }
  
  // リフレッシュトークン検証
  verifyRefreshToken(token: string): { userId: string; type: string } {
    try {
      return jwt.verify(token, this.JWT_REFRESH_SECRET, {
        issuer: 'your-app',
        audience: 'your-app-users'
      }) as { userId: string; type: string };
    } catch (error) {
      if (error instanceof jwt.TokenExpiredError) {
        throw new Error('REFRESH_TOKEN_EXPIRED');
      }
      if (error instanceof jwt.JsonWebTokenError) {
        throw new Error('INVALID_REFRESH_TOKEN');
      }
      throw error;
    }
  }
  
  // ログイン処理
  async login(email: string, password: string): Promise<{
    user: User;
    accessToken: string;
    refreshToken: string;
  }> {
    // ユーザー存在確認
    const user = await userRepository.findByEmail(email);
    if (!user) {
      throw new Error('INVALID_CREDENTIALS');
    }
    
    // パスワード検証
    const isValidPassword = await this.verifyPassword(password, user.passwordHash);
    if (!isValidPassword) {
      throw new Error('INVALID_CREDENTIALS');
    }
    
    // アカウントロック確認
    if (user.lockedUntil && user.lockedUntil > new Date()) {
      throw new Error('ACCOUNT_LOCKED');
    }
    
    // ログイン試行回数をリセット
    await userRepository.resetLoginAttempts(user.id);
    
    // トークン生成
    const accessToken = this.generateAccessToken({
      userId: user.id,
      email: user.email,
      role: user.role
    });
    
    const refreshToken = this.generateRefreshToken(user.id);
    
    // リフレッシュトークンをデータベースに保存
    await refreshTokenRepository.create({
      userId: user.id,
      token: refreshToken,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) // 7日後
    });
    
    // 最終ログイン時刻を更新
    await userRepository.updateLastLogin(user.id);
    
    return {
      user: this.sanitizeUser(user),
      accessToken,
      refreshToken
    };
  }
  
  // トークンリフレッシュ
  async refreshTokens(refreshToken: string): Promise<{
    accessToken: string;
    refreshToken: string;
  }> {
    // リフレッシュトークン検証
    const payload = this.verifyRefreshToken(refreshToken);
    
    // データベースでトークン確認
    const storedToken = await refreshTokenRepository.findByToken(refreshToken);
    if (!storedToken || storedToken.userId !== payload.userId) {
      throw new Error('INVALID_REFRESH_TOKEN');
    }
    
    // ユーザー情報取得
    const user = await userRepository.findById(payload.userId);
    if (!user) {
      throw new Error('USER_NOT_FOUND');
    }
    
    // 新しいトークン生成
    const newAccessToken = this.generateAccessToken({
      userId: user.id,
      email: user.email,
      role: user.role
    });
    
    const newRefreshToken = this.generateRefreshToken(user.id);
    
    // 古いリフレッシュトークンを削除し、新しいものを保存
    await refreshTokenRepository.delete(storedToken.id);
    await refreshTokenRepository.create({
      userId: user.id,
      token: newRefreshToken,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000)
    });
    
    return {
      accessToken: newAccessToken,
      refreshToken: newRefreshToken
    };
  }
  
  // ログアウト
  async logout(refreshToken: string): Promise<void> {
    const storedToken = await refreshTokenRepository.findByToken(refreshToken);
    if (storedToken) {
      await refreshTokenRepository.delete(storedToken.id);
    }
  }
  
  // ユーザー情報サニタイズ
  private sanitizeUser(user: User): Omit<User, 'passwordHash'> {
    const { passwordHash, ...sanitizedUser } = user;
    return sanitizedUser;
  }
}
```

### JWT認証ミドルウェア
```typescript
class AuthMiddleware {
  // 必須認証ミドルウェア
  static requireAuth = (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return res.status(401).json({
        success: false,
        error: 'MISSING_AUTH_TOKEN',
        message: '認証トークンが必要です'
      });
    }
    
    const token = authHeader.substring(7);
    
    try {
      const authService = new AuthService();
      const payload = authService.verifyAccessToken(token);
      req.user = payload;
      next();
    } catch (error) {
      const errorMessages = {
        'ACCESS_TOKEN_EXPIRED': 'アクセストークンの有効期限が切れています',
        'INVALID_ACCESS_TOKEN': '無効なアクセストークンです'
      };
      
      return res.status(401).json({
        success: false,
        error: error.message,
        message: errorMessages[error.message as keyof typeof errorMessages] || '認証に失敗しました'
      });
    }
  };
  
  // オプション認証ミドルウェア
  static optionalAuth = (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
      return next();
    }
    
    const token = authHeader.substring(7);
    
    try {
      const authService = new AuthService();
      const payload = authService.verifyAccessToken(token);
      req.user = payload;
    } catch (error) {
      // オプション認証では認証失敗でもエラーにしない
    }
    
    next();
  };
  
  // ロール制限ミドルウェア
  static requireRole = (...allowedRoles: string[]) => {
    return (req: AuthenticatedRequest, res: Response, next: NextFunction) => {
      if (!req.user) {
        return res.status(401).json({
          success: false,
          error: 'AUTHENTICATION_REQUIRED',
          message: '認証が必要です'
        });
      }
      
      if (!allowedRoles.includes(req.user.role)) {
        return res.status(403).json({
          success: false,
          error: 'INSUFFICIENT_PERMISSIONS',
          message: '権限が不足しています'
        });
      }
      
      next();
    };
  };
  
  // レート制限による保護
  static protectSensitiveEndpoint = async (
    req: AuthenticatedRequest, 
    res: Response, 
    next: NextFunction
  ) => {
    const clientIp = req.ip;
    const userId = req.user?.userId;
    
    // IP ベースの制限
    const ipKey = `sensitive_endpoint:ip:${clientIp}`;
    const ipAttempts = await redis.incr(ipKey);
    if (ipAttempts === 1) {
      await redis.expire(ipKey, 3600); // 1時間
    }
    if (ipAttempts > 10) {
      return res.status(429).json({
        success: false,
        error: 'TOO_MANY_ATTEMPTS',
        message: '試行回数が上限に達しました'
      });
    }
    
    // ユーザー別の制限
    if (userId) {
      const userKey = `sensitive_endpoint:user:${userId}`;
      const userAttempts = await redis.incr(userKey);
      if (userAttempts === 1) {
        await redis.expire(userKey, 900); // 15分
      }
      if (userAttempts > 5) {
        return res.status(429).json({
          success: false,
          error: 'USER_RATE_LIMIT_EXCEEDED',
          message: 'ユーザー別の制限に達しました'
        });
      }
    }
    
    next();
  };
}
```

## OAuth 2.0 実装

### OAuth プロバイダー統合
```typescript
import passport from 'passport';
import { Strategy as GoogleStrategy } from 'passport-google-oauth20';
import { Strategy as GitHubStrategy } from 'passport-github2';
import { Strategy as FacebookStrategy } from 'passport-facebook';

class OAuthService {
  constructor() {
    this.configureStrategies();
  }
  
  private configureStrategies() {
    // Google OAuth設定
    passport.use(new GoogleStrategy({
      clientID: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
      callbackURL: '/auth/google/callback'
    }, async (accessToken, refreshToken, profile, done) => {
      try {
        const user = await this.handleOAuthLogin({
          provider: 'google',
          providerId: profile.id,
          email: profile.emails?.[0].value,
          name: profile.displayName,
          avatar: profile.photos?.[0].value,
          accessToken,
          refreshToken
        });
        
        return done(null, user);
      } catch (error) {
        return done(error, null);
      }
    }));
    
    // GitHub OAuth設定
    passport.use(new GitHubStrategy({
      clientID: process.env.GITHUB_CLIENT_ID!,
      clientSecret: process.env.GITHUB_CLIENT_SECRET!,
      callbackURL: '/auth/github/callback'
    }, async (accessToken, refreshToken, profile, done) => {
      try {
        const user = await this.handleOAuthLogin({
          provider: 'github',
          providerId: profile.id,
          email: profile.emails?.[0].value,
          name: profile.displayName || profile.username,
          avatar: profile.photos?.[0].value,
          accessToken,
          refreshToken
        });
        
        return done(null, user);
      } catch (error) {
        return done(error, null);
      }
    }));
  }
  
  private async handleOAuthLogin(oauthData: {
    provider: string;
    providerId: string;
    email?: string;
    name?: string;
    avatar?: string;
    accessToken: string;
    refreshToken?: string;
  }): Promise<User> {
    // 既存のOAuth接続を確認
    let oauthAccount = await oauthAccountRepository.findByProviderAndId(
      oauthData.provider,
      oauthData.providerId
    );
    
    if (oauthAccount) {
      // トークンを更新
      await oauthAccountRepository.updateTokens(oauthAccount.id, {
        accessToken: oauthData.accessToken,
        refreshToken: oauthData.refreshToken
      });
      
      return userRepository.findById(oauthAccount.userId);
    }
    
    // メールアドレスで既存ユーザーを確認
    let user: User | null = null;
    if (oauthData.email) {
      user = await userRepository.findByEmail(oauthData.email);
    }
    
    if (!user) {
      // 新規ユーザー作成
      user = await userRepository.create({
        email: oauthData.email || `${oauthData.provider}.${oauthData.providerId}@oauth.local`,
        name: oauthData.name || 'OAuth User',
        role: 'user',
        emailVerified: !!oauthData.email, // OAuth経由なら検証済みとみなす
        avatar: oauthData.avatar
      });
    }
    
    // OAuth接続情報を保存
    await oauthAccountRepository.create({
      userId: user.id,
      provider: oauthData.provider,
      providerId: oauthData.providerId,
      accessToken: oauthData.accessToken,
      refreshToken: oauthData.refreshToken,
      scope: this.getScope(oauthData.provider)
    });
    
    return user;
  }
  
  private getScope(provider: string): string {
    const scopes = {
      google: 'profile email',
      github: 'user:email',
      facebook: 'email'
    };
    
    return scopes[provider as keyof typeof scopes] || '';
  }
  
  // OAuth接続の解除
  async disconnectOAuth(userId: string, provider: string): Promise<void> {
    const user = await userRepository.findById(userId);
    if (!user) {
      throw new Error('USER_NOT_FOUND');
    }
    
    // パスワードが設定されていない場合、OAuth解除を禁止
    if (!user.passwordHash) {
      const otherConnections = await oauthAccountRepository.findByUserId(userId);
      if (otherConnections.length <= 1) {
        throw new Error('CANNOT_DISCONNECT_LAST_AUTH_METHOD');
      }
    }
    
    await oauthAccountRepository.deleteByUserAndProvider(userId, provider);
  }
  
  // OAuthトークンの更新
  async refreshOAuthToken(userId: string, provider: string): Promise<string> {
    const oauthAccount = await oauthAccountRepository.findByUserAndProvider(userId, provider);
    if (!oauthAccount || !oauthAccount.refreshToken) {
      throw new Error('OAUTH_REFRESH_TOKEN_NOT_FOUND');
    }
    
    // プロバイダー別のトークン更新処理
    switch (provider) {
      case 'google':
        return this.refreshGoogleToken(oauthAccount);
      case 'github':
        // GitHubは通常リフレッシュトークンを提供しない
        throw new Error('GITHUB_REFRESH_NOT_SUPPORTED');
      default:
        throw new Error('UNSUPPORTED_PROVIDER');
    }
  }
  
  private async refreshGoogleToken(oauthAccount: OAuthAccount): Promise<string> {
    const response = await fetch('https://oauth2.googleapis.com/token', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/x-www-form-urlencoded'
      },
      body: new URLSearchParams({
        client_id: process.env.GOOGLE_CLIENT_ID!,
        client_secret: process.env.GOOGLE_CLIENT_SECRET!,
        refresh_token: oauthAccount.refreshToken!,
        grant_type: 'refresh_token'
      })
    });
    
    if (!response.ok) {
      throw new Error('OAUTH_TOKEN_REFRESH_FAILED');
    }
    
    const data = await response.json();
    
    // 新しいトークンを保存
    await oauthAccountRepository.updateTokens(oauthAccount.id, {
      accessToken: data.access_token,
      refreshToken: data.refresh_token || oauthAccount.refreshToken
    });
    
    return data.access_token;
  }
}
```

## セッション管理

### Redis セッションストア
```typescript
import session from 'express-session';
import RedisStore from 'connect-redis';
import Redis from 'ioredis';

class SessionManager {
  private redis: Redis;
  private sessionStore: RedisStore;
  
  constructor() {
    this.redis = new Redis(process.env.REDIS_URL!, {
      keyPrefix: 'sess:',
      retryDelayOnFailover: 100,
      maxRetriesPerRequest: 3
    });
    
    this.sessionStore = new RedisStore({
      client: this.redis,
      prefix: 'sess:'
    });
  }
  
  getSessionMiddleware() {
    return session({
      store: this.sessionStore,
      secret: process.env.SESSION_SECRET!,
      name: 'sessionId',
      resave: false,
      saveUninitialized: false,
      rolling: true, // セッション有効期限をリセット
      cookie: {
        secure: process.env.NODE_ENV === 'production', // HTTPSでのみ
        httpOnly: true, // XSS攻撃を防ぐ
        maxAge: 24 * 60 * 60 * 1000, // 24時間
        sameSite: 'strict' // CSRF攻撃を防ぐ
      }
    });
  }
  
  // セッション情報の取得
  async getSession(sessionId: string): Promise<any> {
    return new Promise((resolve, reject) => {
      this.sessionStore.get(sessionId, (err, session) => {
        if (err) reject(err);
        else resolve(session);
      });
    });
  }
  
  // セッションの破棄
  async destroySession(sessionId: string): Promise<void> {
    return new Promise((resolve, reject) => {
      this.sessionStore.destroy(sessionId, (err) => {
        if (err) reject(err);
        else resolve();
      });
    });
  }
  
  // ユーザーの全セッションを破棄
  async destroyAllUserSessions(userId: string): Promise<void> {
    const pattern = `sess:*`;
    const keys = await this.redis.keys(pattern);
    
    for (const key of keys) {
      const sessionData = await this.redis.get(key);
      if (sessionData) {
        const session = JSON.parse(sessionData);
        if (session.userId === userId) {
          await this.redis.del(key);
        }
      }
    }
  }
  
  // アクティブセッション数の取得
  async getActiveSessionCount(userId: string): Promise<number> {
    const pattern = `sess:*`;
    const keys = await this.redis.keys(pattern);
    let count = 0;
    
    for (const key of keys) {
      const sessionData = await this.redis.get(key);
      if (sessionData) {
        const session = JSON.parse(sessionData);
        if (session.userId === userId) {
          count++;
        }
      }
    }
    
    return count;
  }
  
  // セッション制限の実装
  async enforceSingleSession(userId: string, currentSessionId: string): Promise<void> {
    const pattern = `sess:*`;
    const keys = await this.redis.keys(pattern);
    
    for (const key of keys) {
      const sessionId = key.replace('sess:', '');
      if (sessionId === currentSessionId) continue;
      
      const sessionData = await this.redis.get(key);
      if (sessionData) {
        const session = JSON.parse(sessionData);
        if (session.userId === userId) {
          await this.redis.del(key);
        }
      }
    }
  }
}

// セッション認証ミドルウェア
class SessionAuth {
  static requireAuth = (req: Request, res: Response, next: NextFunction) => {
    if (!req.session?.userId) {
      return res.status(401).json({
        success: false,
        error: 'AUTHENTICATION_REQUIRED',
        message: 'ログインが必要です'
      });
    }
    
    next();
  };
  
  static attachUserInfo = async (req: Request, res: Response, next: NextFunction) => {
    if (req.session?.userId) {
      try {
        const user = await userRepository.findById(req.session.userId);
        if (user) {
          req.user = user;
        } else {
          // ユーザーが見つからない場合はセッションを破棄
          req.session.destroy((err) => {
            if (err) console.error('Session destruction error:', err);
          });
        }
      } catch (error) {
        console.error('User fetch error:', error);
      }
    }
    
    next();
  };
  
  static requireRole = (...allowedRoles: string[]) => {
    return (req: Request, res: Response, next: NextFunction) => {
      if (!req.user) {
        return res.status(401).json({
          success: false,
          error: 'AUTHENTICATION_REQUIRED'
        });
      }
      
      if (!allowedRoles.includes(req.user.role)) {
        return res.status(403).json({
          success: false,
          error: 'INSUFFICIENT_PERMISSIONS'
        });
      }
      
      next();
    };
  };
}
```

## 多要素認証 (MFA)

### TOTP (Time-based One-Time Password) 実装
```typescript
import speakeasy from 'speakeasy';
import QRCode from 'qrcode';

class MFAService {
  // MFA設定の開始
  async setupMFA(userId: string): Promise<{
    secret: string;
    qrCodeUrl: string;
    backupCodes: string[];
  }> {
    const user = await userRepository.findById(userId);
    if (!user) {
      throw new Error('USER_NOT_FOUND');
    }
    
    // シークレットキーの生成
    const secret = speakeasy.generateSecret({
      name: `YourApp (${user.email})`,
      issuer: 'YourApp',
      length: 32
    });
    
    // バックアップコードの生成
    const backupCodes = Array.from({ length: 10 }, () => 
      Math.random().toString(36).substring(2, 8).toUpperCase()
    );
    
    // 一時的にシークレットを保存（未確認状態）
    await mfaRepository.createPendingSetup(userId, {
      secret: secret.base32,
      backupCodes: await this.hashBackupCodes(backupCodes)
    });
    
    // QRコードURL生成
    const qrCodeUrl = await QRCode.toDataURL(secret.otpauth_url!);
    
    return {
      secret: secret.base32,
      qrCodeUrl,
      backupCodes
    };
  }
  
  // MFA設定の確認
  async confirmMFASetup(userId: string, token: string): Promise<void> {
    const pendingSetup = await mfaRepository.getPendingSetup(userId);
    if (!pendingSetup) {
      throw new Error('MFA_SETUP_NOT_FOUND');
    }
    
    // TOTPトークンの検証
    const isValid = speakeasy.totp.verify({
      secret: pendingSetup.secret,
      encoding: 'base32',
      token,
      window: 2 // 1分前後の時間ずれを許容
    });
    
    if (!isValid) {
      throw new Error('INVALID_MFA_TOKEN');
    }
    
    // MFA設定を確定
    await mfaRepository.confirmSetup(userId, {
      secret: pendingSetup.secret,
      backupCodes: pendingSetup.backupCodes,
      isEnabled: true
    });
    
    // ペンディング設定を削除
    await mfaRepository.deletePendingSetup(userId);
    
    // ユーザーのMFA有効フラグを更新
    await userRepository.updateMFAStatus(userId, true);
  }
  
  // MFAトークンの検証
  async verifyMFAToken(userId: string, token: string): Promise<boolean> {
    const mfaConfig = await mfaRepository.findByUserId(userId);
    if (!mfaConfig || !mfaConfig.isEnabled) {
      throw new Error('MFA_NOT_CONFIGURED');
    }
    
    // TOTPトークンの検証
    const isValidTotp = speakeasy.totp.verify({
      secret: mfaConfig.secret,
      encoding: 'base32',
      token,
      window: 2
    });
    
    if (isValidTotp) {
      return true;
    }
    
    // バックアップコードの確認
    const isValidBackupCode = await this.verifyBackupCode(userId, token);
    if (isValidBackupCode) {
      // 使用されたバックアップコードを無効化
      await mfaRepository.markBackupCodeAsUsed(userId, token);
      return true;
    }
    
    return false;
  }
  
  // バックアップコードの検証
  private async verifyBackupCode(userId: string, code: string): Promise<boolean> {
    const mfaConfig = await mfaRepository.findByUserId(userId);
    if (!mfaConfig) return false;
    
    for (const backupCode of mfaConfig.backupCodes) {
      if (!backupCode.used && await bcrypt.compare(code, backupCode.hash)) {
        return true;
      }
    }
    
    return false;
  }
  
  // バックアップコードのハッシュ化
  private async hashBackupCodes(codes: string[]): Promise<Array<{ hash: string; used: boolean }>> {
    return Promise.all(
      codes.map(async code => ({
        hash: await bcrypt.hash(code, 10),
        used: false
      }))
    );
  }
  
  // MFA無効化
  async disableMFA(userId: string, password: string): Promise<void> {
    const user = await userRepository.findById(userId);
    if (!user) {
      throw new Error('USER_NOT_FOUND');
    }
    
    // パスワード確認
    const authService = new AuthService();
    const isValidPassword = await authService.verifyPassword(password, user.passwordHash);
    if (!isValidPassword) {
      throw new Error('INVALID_PASSWORD');
    }
    
    // MFA設定を削除
    await mfaRepository.deleteByUserId(userId);
    await userRepository.updateMFAStatus(userId, false);
  }
  
  // 新しいバックアップコードの生成
  async regenerateBackupCodes(userId: string): Promise<string[]> {
    const newBackupCodes = Array.from({ length: 10 }, () => 
      Math.random().toString(36).substring(2, 8).toUpperCase()
    );
    
    const hashedCodes = await this.hashBackupCodes(newBackupCodes);
    
    await mfaRepository.updateBackupCodes(userId, hashedCodes);
    
    return newBackupCodes;
  }
}

// MFA対応ログインフロー
class MFAAuthController {
  async login(req: Request, res: Response) {
    try {
      const { email, password, mfaToken } = req.body;
      
      // 基本認証
      const authService = new AuthService();
      const user = await userRepository.findByEmail(email);
      
      if (!user || !await authService.verifyPassword(password, user.passwordHash)) {
        return res.status(401).json({
          success: false,
          error: 'INVALID_CREDENTIALS'
        });
      }
      
      // MFAが有効かチェック
      if (user.mfaEnabled) {
        if (!mfaToken) {
          return res.status(200).json({
            success: true,
            requiresMFA: true,
            tempToken: this.generateTempToken(user.id)
          });
        }
        
        // MFAトークンの検証
        const mfaService = new MFAService();
        const isValidMFA = await mfaService.verifyMFAToken(user.id, mfaToken);
        
        if (!isValidMFA) {
          return res.status(401).json({
            success: false,
            error: 'INVALID_MFA_TOKEN'
          });
        }
      }
      
      // 正常ログイン処理
      const tokens = await authService.generateTokens(user);
      
      res.status(200).json({
        success: true,
        user: authService.sanitizeUser(user),
        ...tokens
      });
    } catch (error) {
      res.status(500).json({
        success: false,
        error: 'LOGIN_FAILED'
      });
    }
  }
  
  private generateTempToken(userId: string): string {
    return jwt.sign(
      { userId, type: 'mfa_pending' },
      process.env.JWT_SECRET!,
      { expiresIn: '5m' }
    );
  }
}
```