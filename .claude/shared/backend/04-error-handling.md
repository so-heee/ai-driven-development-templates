# エラーハンドリングパターン

## カスタムエラークラス

### 基本エラークラス設計
```typescript
// ベースエラークラス
abstract class AppError extends Error {
  abstract readonly statusCode: number;
  abstract readonly isOperational: boolean;
  abstract readonly errorCode: string;
  
  readonly timestamp: Date;
  readonly stack?: string;
  
  constructor(
    message: string,
    public readonly context?: Record<string, any>
  ) {
    super(message);
    this.name = this.constructor.name;
    this.timestamp = new Date();
    
    // スタックトレースの取得
    Error.captureStackTrace(this, this.constructor);
  }
  
  toJSON() {
    return {
      name: this.name,
      message: this.message,
      statusCode: this.statusCode,
      errorCode: this.errorCode,
      timestamp: this.timestamp,
      context: this.context,
      isOperational: this.isOperational
    };
  }
}

// バリデーションエラー
class ValidationError extends AppError {
  readonly statusCode = 400;
  readonly isOperational = true;
  readonly errorCode = 'VALIDATION_ERROR';
  
  constructor(
    message: string,
    public readonly field?: string,
    public readonly validationRules?: string[],
    context?: Record<string, any>
  ) {
    super(message, {
      ...context,
      field,
      validationRules
    });
  }
}

// 認証エラー
class AuthenticationError extends AppError {
  readonly statusCode = 401;
  readonly isOperational = true;
  readonly errorCode = 'AUTHENTICATION_ERROR';
  
  constructor(message: string = '認証が必要です', context?: Record<string, any>) {
    super(message, context);
  }
}

// 認可エラー
class AuthorizationError extends AppError {
  readonly statusCode = 403;
  readonly isOperational = true;
  readonly errorCode = 'AUTHORIZATION_ERROR';
  
  constructor(
    message: string = '権限が不足しています',
    public readonly requiredRole?: string,
    public readonly userRole?: string,
    context?: Record<string, any>
  ) {
    super(message, {
      ...context,
      requiredRole,
      userRole
    });
  }
}

// リソース未検出エラー
class NotFoundError extends AppError {
  readonly statusCode = 404;
  readonly isOperational = true;
  readonly errorCode = 'NOT_FOUND_ERROR';
  
  constructor(
    message: string = 'リソースが見つかりません',
    public readonly resourceType?: string,
    public readonly resourceId?: string,
    context?: Record<string, any>
  ) {
    super(message, {
      ...context,
      resourceType,
      resourceId
    });
  }
}

// 競合エラー
class ConflictError extends AppError {
  readonly statusCode = 409;
  readonly isOperational = true;
  readonly errorCode = 'CONFLICT_ERROR';
  
  constructor(
    message: string = 'リソースが競合しています',
    public readonly conflictType?: string,
    context?: Record<string, any>
  ) {
    super(message, {
      ...context,
      conflictType
    });
  }
}

// レート制限エラー
class RateLimitError extends AppError {
  readonly statusCode = 429;
  readonly isOperational = true;
  readonly errorCode = 'RATE_LIMIT_ERROR';
  
  constructor(
    message: string = 'リクエスト制限に達しました',
    public readonly retryAfter?: number,
    context?: Record<string, any>
  ) {
    super(message, {
      ...context,
      retryAfter
    });
  }
}

// ビジネスロジックエラー
class BusinessLogicError extends AppError {
  readonly statusCode = 422;
  readonly isOperational = true;
  readonly errorCode = 'BUSINESS_LOGIC_ERROR';
  
  constructor(
    message: string,
    public readonly businessRule?: string,
    context?: Record<string, any>
  ) {
    super(message, {
      ...context,
      businessRule
    });
  }
}

// 外部サービスエラー
class ExternalServiceError extends AppError {
  readonly statusCode = 502;
  readonly isOperational = true;
  readonly errorCode = 'EXTERNAL_SERVICE_ERROR';
  
  constructor(
    message: string,
    public readonly service?: string,
    public readonly originalError?: Error,
    context?: Record<string, any>
  ) {
    super(message, {
      ...context,
      service,
      originalError: originalError?.message
    });
  }
}

// システムエラー
class SystemError extends AppError {
  readonly statusCode = 500;
  readonly isOperational = false;
  readonly errorCode = 'SYSTEM_ERROR';
  
  constructor(
    message: string,
    public readonly originalError?: Error,
    context?: Record<string, any>
  ) {
    super(message, {
      ...context,
      originalError: originalError?.message
    });
  }
}
```

### エラーファクトリ
```typescript
class ErrorFactory {
  // バリデーションエラーの生成
  static validation(field: string, value: any, rules: string[]): ValidationError {
    return new ValidationError(
      `${field}の値が無効です`,
      field,
      rules,
      { invalidValue: value }
    );
  }
  
  // 必須フィールドエラー
  static required(field: string): ValidationError {
    return new ValidationError(
      `${field}は必須です`,
      field,
      ['required']
    );
  }
  
  // 重複エラー
  static duplicate(resource: string, field: string, value: any): ConflictError {
    return new ConflictError(
      `この${field}は既に使用されています`,
      'duplicate',
      { resource, field, value }
    );
  }
  
  // 期限切れエラー
  static expired(resource: string, expiryDate?: Date): AuthenticationError {
    return new AuthenticationError(
      `${resource}の有効期限が切れています`,
      { resource, expiryDate }
    );
  }
  
  // 不十分な権限エラー
  static insufficientPermissions(action: string, resource: string): AuthorizationError {
    return new AuthorizationError(
      `${resource}に対する${action}権限がありません`,
      undefined,
      undefined,
      { action, resource }
    );
  }
  
  // 外部API エラー
  static externalAPI(service: string, operation: string, error: Error): ExternalServiceError {
    return new ExternalServiceError(
      `${service}の${operation}でエラーが発生しました`,
      service,
      error,
      { operation }
    );
  }
  
  // データベースエラー
  static database(operation: string, error: Error): SystemError {
    return new SystemError(
      `データベース操作でエラーが発生しました: ${operation}`,
      error,
      { operation, dbError: true }
    );
  }
}
```

## グローバルエラーハンドラー

### Express エラーハンドリングミドルウェア
```typescript
import { Request, Response, NextFunction } from 'express';
import { ZodError } from 'zod';

class ErrorHandler {
  // メインエラーハンドラー
  static handle = (error: Error, req: Request, res: Response, next: NextFunction) => {
    // リクエストID を追加
    const requestId = req.id || 'unknown';
    
    // エラーログ記録
    logger.error('Error caught by global handler:', {
      error: error.message,
      stack: error.stack,
      requestId,
      method: req.method,
      url: req.originalUrl,
      userAgent: req.get('User-Agent'),
      ip: req.ip,
      userId: req.user?.id
    });
    
    // 既知のエラータイプの処理
    if (error instanceof AppError) {
      return this.handleAppError(error, req, res);
    }
    
    // Zodバリデーションエラー
    if (error instanceof ZodError) {
      return this.handleZodError(error, req, res);
    }
    
    // Prisma エラー
    if (error.name === 'PrismaClientKnownRequestError') {
      return this.handlePrismaError(error as any, req, res);
    }
    
    // JWT エラー
    if (error.name === 'JsonWebTokenError' || error.name === 'TokenExpiredError') {
      return this.handleJWTError(error, req, res);
    }
    
    // Multer エラー（ファイルアップロード）
    if (error.name === 'MulterError') {
      return this.handleMulterError(error as any, req, res);
    }
    
    // 予期しないエラー
    return this.handleUnknownError(error, req, res);
  };
  
  // AppError の処理
  private static handleAppError(error: AppError, req: Request, res: Response) {
    const response = {
      success: false,
      error: {
        code: error.errorCode,
        message: error.message,
        ...(process.env.NODE_ENV === 'development' && {
          context: error.context
        })
      },
      requestId: req.id,
      timestamp: error.timestamp
    };
    
    return res.status(error.statusCode).json(response);
  }
  
  // Zod バリデーションエラーの処理
  private static handleZodError(error: ZodError, req: Request, res: Response) {
    const validationErrors = error.errors.map(err => ({
      field: err.path.join('.'),
      message: err.message,
      code: err.code,
      received: err.received
    }));
    
    return res.status(400).json({
      success: false,
      error: {
        code: 'VALIDATION_ERROR',
        message: 'バリデーションエラーが発生しました',
        details: validationErrors
      },
      requestId: req.id,
      timestamp: new Date()
    });
  }
  
  // Prisma エラーの処理
  private static handlePrismaError(error: any, req: Request, res: Response) {
    switch (error.code) {
      case 'P2002': // Unique constraint violation
        return res.status(409).json({
          success: false,
          error: {
            code: 'DUPLICATE_RESOURCE',
            message: '重複したリソースです',
            field: error.meta?.target?.[0]
          },
          requestId: req.id,
          timestamp: new Date()
        });
        
      case 'P2025': // Record not found
        return res.status(404).json({
          success: false,
          error: {
            code: 'RESOURCE_NOT_FOUND',
            message: 'リソースが見つかりません'
          },
          requestId: req.id,
          timestamp: new Date()
        });
        
      case 'P2003': // Foreign key constraint violation
        return res.status(400).json({
          success: false,
          error: {
            code: 'INVALID_REFERENCE',
            message: '参照先のリソースが存在しません'
          },
          requestId: req.id,
          timestamp: new Date()
        });
        
      default:
        logger.error('Unhandled Prisma error:', error);
        return res.status(500).json({
          success: false,
          error: {
            code: 'DATABASE_ERROR',
            message: 'データベースエラーが発生しました'
          },
          requestId: req.id,
          timestamp: new Date()
        });
    }
  }
  
  // JWT エラーの処理
  private static handleJWTError(error: Error, req: Request, res: Response) {
    if (error.name === 'TokenExpiredError') {
      return res.status(401).json({
        success: false,
        error: {
          code: 'TOKEN_EXPIRED',
          message: 'トークンの有効期限が切れています'
        },
        requestId: req.id,
        timestamp: new Date()
      });
    }
    
    return res.status(401).json({
      success: false,
      error: {
        code: 'INVALID_TOKEN',
        message: '無効なトークンです'
      },
      requestId: req.id,
      timestamp: new Date()
    });
  }
  
  // Multer エラーの処理
  private static handleMulterError(error: any, req: Request, res: Response) {
    switch (error.code) {
      case 'LIMIT_FILE_SIZE':
        return res.status(413).json({
          success: false,
          error: {
            code: 'FILE_TOO_LARGE',
            message: 'ファイルサイズが上限を超えています'
          },
          requestId: req.id,
          timestamp: new Date()
        });
        
      case 'LIMIT_FILE_COUNT':
        return res.status(400).json({
          success: false,
          error: {
            code: 'TOO_MANY_FILES',
            message: 'ファイル数が上限を超えています'
          },
          requestId: req.id,
          timestamp: new Date()
        });
        
      default:
        return res.status(400).json({
          success: false,
          error: {
            code: 'FILE_UPLOAD_ERROR',
            message: 'ファイルアップロードでエラーが発生しました'
          },
          requestId: req.id,
          timestamp: new Date()
        });
    }
  }
  
  // 予期しないエラーの処理
  private static handleUnknownError(error: Error, req: Request, res: Response) {
    // 本番環境では詳細なエラー情報を隠す
    const isDevelopment = process.env.NODE_ENV === 'development';
    
    // セキュリティ上重要なエラー情報を記録
    logger.error('Unhandled error:', {
      error: error.message,
      stack: error.stack,
      requestId: req.id,
      method: req.method,
      url: req.originalUrl,
      headers: req.headers,
      body: req.body,
      userId: req.user?.id
    });
    
    return res.status(500).json({
      success: false,
      error: {
        code: 'INTERNAL_SERVER_ERROR',
        message: 'サーバー内部エラーが発生しました',
        ...(isDevelopment && {
          details: error.message,
          stack: error.stack
        })
      },
      requestId: req.id,
      timestamp: new Date()
    });
  }
  
  // 404 ハンドラー
  static notFound = (req: Request, res: Response) => {
    return res.status(404).json({
      success: false,
      error: {
        code: 'ROUTE_NOT_FOUND',
        message: 'リクエストされたリソースが見つかりません',
        path: req.originalUrl,
        method: req.method
      },
      requestId: req.id,
      timestamp: new Date()
    });
  };
}
```

## 非同期エラーハンドリング

### Promise ラッパー
```typescript
// 非同期関数用のエラーハンドラー
export const asyncHandler = (fn: Function) => {
  return (req: Request, res: Response, next: NextFunction) => {
    Promise.resolve(fn(req, res, next)).catch(next);
  };
};

// 使用例
const getUserById = asyncHandler(async (req: Request, res: Response) => {
  const { id } = req.params;
  
  const user = await userService.findById(id);
  if (!user) {
    throw new NotFoundError('ユーザーが見つかりません', 'user', id);
  }
  
  res.json({
    success: true,
    data: user
  });
});

// 複数の非同期操作
const createUserWithProfile = asyncHandler(async (req: Request, res: Response) => {
  const { userData, profileData } = req.body;
  
  // バリデーション
  const validation = await validateUserCreation(userData);
  if (!validation.isValid) {
    throw new ValidationError('ユーザーデータが無効です', null, validation.errors);
  }
  
  // トランザクション内での処理
  const result = await db.transaction(async (tx) => {
    try {
      const user = await userService.create(userData, tx);
      const profile = await profileService.create({
        ...profileData,
        userId: user.id
      }, tx);
      
      return { user, profile };
    } catch (error) {
      // ロールバックは自動で行われる
      if (error.code === 'P2002') {
        throw ErrorFactory.duplicate('user', 'email', userData.email);
      }
      throw error;
    }
  });
  
  res.status(201).json({
    success: true,
    data: result
  });
});
```

### エラー境界パターン
```typescript
// サービス層でのエラー境界
class UserService {
  async findById(id: string): Promise<User> {
    try {
      const user = await userRepository.findById(id);
      if (!user) {
        throw new NotFoundError('ユーザーが見つかりません', 'user', id);
      }
      return user;
    } catch (error) {
      if (error instanceof AppError) {
        throw error;
      }
      
      logger.error('User lookup failed:', { id, error: error.message });
      throw new SystemError('ユーザー検索でエラーが発生しました', error);
    }
  }
  
  async create(userData: CreateUserData): Promise<User> {
    try {
      // 重複チェック
      const existingUser = await userRepository.findByEmail(userData.email);
      if (existingUser) {
        throw ErrorFactory.duplicate('user', 'email', userData.email);
      }
      
      // パスワードハッシュ化
      const hashedPassword = await bcrypt.hash(userData.password, 12);
      
      const user = await userRepository.create({
        ...userData,
        passwordHash: hashedPassword
      });
      
      // ウェルカムメール送信（エラーがあっても作成は成功とする）
      this.sendWelcomeEmail(user).catch(error => {
        logger.warn('Welcome email failed:', { userId: user.id, error: error.message });
      });
      
      return user;
    } catch (error) {
      if (error instanceof AppError) {
        throw error;
      }
      
      logger.error('User creation failed:', { userData: { email: userData.email }, error: error.message });
      throw new SystemError('ユーザー作成でエラーが発生しました', error);
    }
  }
  
  private async sendWelcomeEmail(user: User): Promise<void> {
    try {
      await emailService.sendWelcomeEmail(user.email, user.name);
    } catch (error) {
      // メール送信失敗は例外を再スローしない
      logger.error('Welcome email sending failed:', {
        userId: user.id,
        email: user.email,
        error: error.message
      });
    }
  }
}

// レポジトリ層でのエラー境界
class UserRepository {
  async findById(id: string): Promise<User | null> {
    try {
      return await prisma.user.findUnique({
        where: { id },
        include: { profile: true }
      });
    } catch (error) {
      logger.error('Database query failed:', { operation: 'findById', id, error: error.message });
      throw ErrorFactory.database('ユーザー検索', error);
    }
  }
  
  async create(userData: CreateUserData): Promise<User> {
    try {
      return await prisma.user.create({
        data: userData,
        include: { profile: true }
      });
    } catch (error) {
      if (error.code === 'P2002') {
        const field = error.meta?.target?.[0] || 'unknown';
        throw ErrorFactory.duplicate('user', field, userData[field as keyof CreateUserData]);
      }
      
      logger.error('Database insertion failed:', { operation: 'create', error: error.message });
      throw ErrorFactory.database('ユーザー作成', error);
    }
  }
}
```

## サーキットブレーカーパターン

### 外部サービス呼び出しの保護
```typescript
enum CircuitState {
  CLOSED = 'CLOSED',
  OPEN = 'OPEN',
  HALF_OPEN = 'HALF_OPEN'
}

class CircuitBreaker {
  private state: CircuitState = CircuitState.CLOSED;
  private failureCount: number = 0;
  private lastFailureTime: number = 0;
  private successCount: number = 0;
  
  constructor(
    private readonly threshold: number = 5,
    private readonly timeout: number = 60000, // 1分
    private readonly monitoringPeriod: number = 30000 // 30秒
  ) {}
  
  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === CircuitState.OPEN) {
      if (Date.now() - this.lastFailureTime > this.timeout) {
        this.state = CircuitState.HALF_OPEN;
        this.successCount = 0;
        logger.info('Circuit breaker moved to HALF_OPEN state');
      } else {
        throw new ExternalServiceError(
          'サービスが一時的に利用できません（サーキットブレーカーがオープン）',
          'circuit-breaker'
        );
      }
    }
    
    try {
      const result = await operation();
      return this.onSuccess(result);
    } catch (error) {
      return this.onFailure(error);
    }
  }
  
  private onSuccess<T>(result: T): T {
    if (this.state === CircuitState.HALF_OPEN) {
      this.successCount++;
      if (this.successCount >= 3) {
        this.reset();
        logger.info('Circuit breaker reset to CLOSED state');
      }
    } else {
      this.failureCount = 0;
    }
    
    return result;
  }
  
  private onFailure(error: Error): never {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    
    if (this.state === CircuitState.HALF_OPEN || this.failureCount >= this.threshold) {
      this.state = CircuitState.OPEN;
      logger.warn('Circuit breaker opened', {
        failureCount: this.failureCount,
        threshold: this.threshold,
        error: error.message
      });
    }
    
    throw error;
  }
  
  private reset(): void {
    this.state = CircuitState.CLOSED;
    this.failureCount = 0;
    this.successCount = 0;
    this.lastFailureTime = 0;
  }
  
  getState(): { state: CircuitState; failureCount: number; lastFailureTime: number } {
    return {
      state: this.state,
      failureCount: this.failureCount,
      lastFailureTime: this.lastFailureTime
    };
  }
}

// 外部サービス呼び出しでの使用
class ExternalAPIService {
  private circuitBreaker = new CircuitBreaker(5, 60000);
  
  async fetchUserData(userId: string): Promise<ExternalUserData> {
    return this.circuitBreaker.execute(async () => {
      const response = await fetch(`https://api.external.com/users/${userId}`, {
        headers: {
          'Authorization': `Bearer ${process.env.EXTERNAL_API_TOKEN}`,
          'Content-Type': 'application/json'
        },
        timeout: 10000
      });
      
      if (!response.ok) {
        throw new Error(`External API error: ${response.status} ${response.statusText}`);
      }
      
      return response.json();
    });
  }
  
  async syncUserData(userId: string): Promise<void> {
    try {
      const externalData = await this.fetchUserData(userId);
      await userService.updateFromExternalSource(userId, externalData);
    } catch (error) {
      if (error instanceof ExternalServiceError && error.service === 'circuit-breaker') {
        // サーキットブレーカーが開いている場合は警告ログのみ
        logger.warn('User sync skipped due to circuit breaker', { userId });
        return;
      }
      
      // その他のエラーは再スロー
      throw ErrorFactory.externalAPI('user-sync', 'fetchUserData', error);
    }
  }
}
```

## エラー監視・通知

### エラー集計と分析
```typescript
class ErrorMonitor {
  private errorCounts = new Map<string, number>();
  private errorPatterns = new Map<string, { count: number; lastSeen: Date; samples: any[] }>();
  
  recordError(error: Error, context: Record<string, any>): void {
    const errorKey = this.generateErrorKey(error);
    const timestamp = new Date();
    
    // エラー回数をカウント
    this.errorCounts.set(errorKey, (this.errorCounts.get(errorKey) || 0) + 1);
    
    // エラーパターンを記録
    const pattern = this.errorPatterns.get(errorKey) || {
      count: 0,
      lastSeen: timestamp,
      samples: []
    };
    
    pattern.count++;
    pattern.lastSeen = timestamp;
    
    // サンプルを保存（最大10件）
    if (pattern.samples.length < 10) {
      pattern.samples.push({
        timestamp,
        context,
        stack: error.stack
      });
    }
    
    this.errorPatterns.set(errorKey, pattern);
    
    // 閾値チェック
    this.checkThresholds(errorKey, pattern);
  }
  
  private generateErrorKey(error: Error): string {
    if (error instanceof AppError) {
      return `${error.errorCode}:${error.message}`;
    }
    
    return `${error.name}:${error.message}`;
  }
  
  private checkThresholds(errorKey: string, pattern: any): void {
    const now = Date.now();
    const fiveMinutesAgo = now - 5 * 60 * 1000;
    const oneHourAgo = now - 60 * 60 * 1000;
    
    // 5分間で10回以上の同じエラー
    const recentSamples = pattern.samples.filter(
      (sample: any) => sample.timestamp.getTime() > fiveMinutesAgo
    );
    
    if (recentSamples.length >= 10) {
      this.sendAlert({
        type: 'HIGH_FREQUENCY_ERROR',
        errorKey,
        count: recentSamples.length,
        timeframe: '5 minutes',
        severity: 'high'
      });
    }
    
    // 1時間で100回以上の同じエラー
    const hourlyCount = pattern.samples.filter(
      (sample: any) => sample.timestamp.getTime() > oneHourAgo
    ).length;
    
    if (hourlyCount >= 100) {
      this.sendAlert({
        type: 'SUSTAINED_ERROR_PATTERN',
        errorKey,
        count: hourlyCount,
        timeframe: '1 hour',
        severity: 'critical'
      });
    }
  }
  
  private async sendAlert(alert: any): Promise<void> {
    try {
      // Slack通知
      await this.sendSlackAlert(alert);
      
      // メール通知（重要度が高い場合）
      if (alert.severity === 'critical') {
        await this.sendEmailAlert(alert);
      }
      
      // 外部監視サービス通知
      await this.sendToMonitoringService(alert);
    } catch (error) {
      logger.error('Alert sending failed:', error);
    }
  }
  
  private async sendSlackAlert(alert: any): Promise<void> {
    const webhook = process.env.SLACK_WEBHOOK_URL;
    if (!webhook) return;
    
    const message = {
      text: `🚨 Error Alert: ${alert.type}`,
      attachments: [{
        color: alert.severity === 'critical' ? 'danger' : 'warning',
        fields: [
          { title: 'Error', value: alert.errorKey, short: false },
          { title: 'Count', value: alert.count.toString(), short: true },
          { title: 'Timeframe', value: alert.timeframe, short: true },
          { title: 'Severity', value: alert.severity, short: true }
        ],
        footer: 'Error Monitoring System',
        ts: Math.floor(Date.now() / 1000)
      }]
    };
    
    await fetch(webhook, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(message)
    });
  }
  
  generateReport(): {
    topErrors: Array<{ error: string; count: number; lastSeen: Date }>;
    totalErrors: number;
    errorRate: number;
  } {
    const sortedErrors = Array.from(this.errorPatterns.entries())
      .sort(([, a], [, b]) => b.count - a.count)
      .slice(0, 10)
      .map(([error, data]) => ({
        error,
        count: data.count,
        lastSeen: data.lastSeen
      }));
    
    const totalErrors = Array.from(this.errorPatterns.values())
      .reduce((sum, pattern) => sum + pattern.count, 0);
    
    return {
      topErrors: sortedErrors,
      totalErrors,
      errorRate: totalErrors / (60 * 60) // 1時間あたりのエラー率
    };
  }
}

// グローバルエラーモニターの初期化
const errorMonitor = new ErrorMonitor();

// エラーハンドラーでの使用
ErrorHandler.handle = (error: Error, req: Request, res: Response, next: NextFunction) => {
  // エラーを監視システムに記録
  errorMonitor.recordError(error, {
    requestId: req.id,
    method: req.method,
    url: req.originalUrl,
    userAgent: req.get('User-Agent'),
    ip: req.ip,
    userId: req.user?.id
  });
  
  // 既存のエラーハンドリング処理...
};
```