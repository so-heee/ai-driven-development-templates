# ログ・監視パターンガイド

## 構造化ログ

### Winston設定パターン
```typescript
import winston from 'winston';
import 'winston-daily-rotate-file';

// ログレベル定義
const LOG_LEVELS = {
  error: 0,
  warn: 1,
  info: 2,
  http: 3,
  debug: 4,
};

const LOG_COLORS = {
  error: 'red',
  warn: 'yellow',
  info: 'green',
  http: 'magenta',
  debug: 'white',
};

winston.addColors(LOG_COLORS);

// カスタムフォーマット
const customFormat = winston.format.combine(
  winston.format.timestamp({ format: 'YYYY-MM-DD HH:mm:ss' }),
  winston.format.errors({ stack: true }),
  winston.format.json(),
  winston.format.printf(({ timestamp, level, message, ...meta }) => {
    return JSON.stringify({
      timestamp,
      level,
      message,
      ...meta,
    });
  })
);

// 開発環境用フォーマット
const developmentFormat = winston.format.combine(
  winston.format.timestamp({ format: 'HH:mm:ss' }),
  winston.format.colorize({ all: true }),
  winston.format.printf(({ timestamp, level, message, ...meta }) => {
    const metaStr = Object.keys(meta).length ? JSON.stringify(meta, null, 2) : '';
    return `${timestamp} [${level}]: ${message} ${metaStr}`;
  })
);

// ロガー設定
class Logger {
  private logger: winston.Logger;
  
  constructor() {
    const isProduction = process.env.NODE_ENV === 'production';
    
    this.logger = winston.createLogger({
      levels: LOG_LEVELS,
      level: process.env.LOG_LEVEL || (isProduction ? 'info' : 'debug'),
      format: isProduction ? customFormat : developmentFormat,
      defaultMeta: {
        service: process.env.SERVICE_NAME || 'backend-api',
        version: process.env.SERVICE_VERSION || '1.0.0',
        environment: process.env.NODE_ENV || 'development',
      },
      transports: this.createTransports(isProduction),
      exceptionHandlers: [
        new winston.transports.File({ filename: 'logs/exceptions.log' })
      ],
      rejectionHandlers: [
        new winston.transports.File({ filename: 'logs/rejections.log' })
      ],
    });
  }
  
  private createTransports(isProduction: boolean): winston.transport[] {
    const transports: winston.transport[] = [];
    
    // コンソール出力
    if (!isProduction) {
      transports.push(
        new winston.transports.Console({
          format: developmentFormat,
        })
      );
    }
    
    // ファイル出力（本番環境）
    if (isProduction) {
      // エラーログ（ローテーション）
      transports.push(
        new winston.transports.DailyRotateFile({
          filename: 'logs/error-%DATE%.log',
          level: 'error',
          datePattern: 'YYYY-MM-DD',
          maxSize: '20m',
          maxFiles: '14d',
          format: customFormat,
        })
      );
      
      // 一般ログ（ローテーション）
      transports.push(
        new winston.transports.DailyRotateFile({
          filename: 'logs/app-%DATE%.log',
          datePattern: 'YYYY-MM-DD',
          maxSize: '20m',
          maxFiles: '7d',
          format: customFormat,
        })
      );
      
      // HTTPアクセスログ
      transports.push(
        new winston.transports.DailyRotateFile({
          filename: 'logs/access-%DATE%.log',
          level: 'http',
          datePattern: 'YYYY-MM-DD',
          maxSize: '50m',
          maxFiles: '30d',
          format: customFormat,
        })
      );
    }
    
    return transports;
  }
  
  // ログメソッド
  error(message: string, meta?: Record<string, any>) {
    this.logger.error(message, meta);
  }
  
  warn(message: string, meta?: Record<string, any>) {
    this.logger.warn(message, meta);
  }
  
  info(message: string, meta?: Record<string, any>) {
    this.logger.info(message, meta);
  }
  
  http(message: string, meta?: Record<string, any>) {
    this.logger.http(message, meta);
  }
  
  debug(message: string, meta?: Record<string, any>) {
    this.logger.debug(message, meta);
  }
  
  // 構造化ログ
  logUserAction(action: string, userId: string, meta?: Record<string, any>) {
    this.info('User action performed', {
      action,
      userId,
      timestamp: new Date().toISOString(),
      ...meta,
    });
  }
  
  logAPIRequest(req: any, res: any, duration: number) {
    this.http('API request', {
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
      userAgent: req.get('User-Agent'),
      ip: req.ip,
      userId: req.user?.id,
      requestId: req.id,
    });
  }
  
  logDatabaseQuery(query: string, duration: number, meta?: Record<string, any>) {
    this.debug('Database query executed', {
      query: query.substring(0, 200), // クエリを200文字に制限
      duration: `${duration}ms`,
      ...meta,
    });
  }
  
  logBusinessEvent(event: string, data: Record<string, any>) {
    this.info('Business event', {
      event,
      data,
      timestamp: new Date().toISOString(),
    });
  }
}

export const logger = new Logger();
```

### Express.js ログミドルウェア
```typescript
import { Request, Response, NextFunction } from 'express';
import { performance } from 'perf_hooks';
import { v4 as uuidv4 } from 'uuid';

// リクエストログミドルウェア
export const requestLogger = (req: Request, res: Response, next: NextFunction) => {
  req.id = uuidv4();
  req.startTime = performance.now();
  
  // リクエスト開始ログ
  logger.info('Request started', {
    requestId: req.id,
    method: req.method,
    url: req.originalUrl,
    userAgent: req.get('User-Agent'),
    ip: req.ip,
    userId: req.user?.id,
  });
  
  // レスポンス完了時のログ
  res.on('finish', () => {
    const endTime = performance.now();
    const duration = endTime - req.startTime;
    
    const logData = {
      requestId: req.id,
      method: req.method,
      url: req.originalUrl,
      statusCode: res.statusCode,
      duration: Math.round(duration),
      userAgent: req.get('User-Agent'),
      ip: req.ip,
      userId: req.user?.id,
      contentLength: res.get('Content-Length'),
    };
    
    // ステータスコードに応じてログレベルを変更
    if (res.statusCode >= 500) {
      logger.error('Request completed with server error', logData);
    } else if (res.statusCode >= 400) {
      logger.warn('Request completed with client error', logData);
    } else {
      logger.http('Request completed', logData);
    }
    
    // パフォーマンス警告
    if (duration > 1000) {
      logger.warn('Slow request detected', {
        ...logData,
        performance: 'slow',
      });
    }
  });
  
  next();
};

// エラーログミドルウェア
export const errorLogger = (error: Error, req: Request, res: Response, next: NextFunction) => {
  logger.error('Unhandled error occurred', {
    error: error.message,
    stack: error.stack,
    requestId: req.id,
    method: req.method,
    url: req.originalUrl,
    userId: req.user?.id,
    body: req.body,
    params: req.params,
    query: req.query,
  });
  
  next(error);
};

// セキュリティログミドルウェア
export const securityLogger = (req: Request, res: Response, next: NextFunction) => {
  // 疑わしいアクティビティの検出
  const suspiciousPatterns = [
    /script.*>/i,
    /javascript:/i,
    /vbscript:/i,
    /onload=/i,
    /onerror=/i,
    /<.*>/,
    /\.\.\//,
    /etc\/passwd/,
    /bin\/bash/,
  ];
  
  const checkSuspicious = (value: string): boolean => {
    return suspiciousPatterns.some(pattern => pattern.test(value));
  };
  
  // URLパラメータのチェック
  const urlParams = new URL(req.url, `http://${req.get('host')}`).searchParams;
  for (const [key, value] of urlParams) {
    if (checkSuspicious(value)) {
      logger.warn('Suspicious URL parameter detected', {
        requestId: req.id,
        parameter: key,
        value,
        ip: req.ip,
        userAgent: req.get('User-Agent'),
      });
    }
  }
  
  // リクエストボディのチェック
  if (req.body && typeof req.body === 'object') {
    const bodyStr = JSON.stringify(req.body);
    if (checkSuspicious(bodyStr)) {
      logger.warn('Suspicious request body detected', {
        requestId: req.id,
        body: req.body,
        ip: req.ip,
        userAgent: req.get('User-Agent'),
      });
    }
  }
  
  // 異常な数のリクエスト
  const rateLimitKey = `rate_limit:${req.ip}`;
  // Redis等でレート制限をチェックし、異常な場合はログ
  
  next();
};
```

## APMツール統合

### New Relic統合
```typescript
import newrelic from 'newrelic';

class NewRelicLogger {
  // カスタムメトリクスの記録
  recordCustomMetric(name: string, value: number) {
    newrelic.recordMetric(`Custom/${name}`, value);
  }
  
  // カスタムイベントの記録
  recordCustomEvent(eventType: string, attributes: Record<string, any>) {
    newrelic.recordCustomEvent(eventType, attributes);
  }
  
  // エラーの記録
  noticeError(error: Error, attributes?: Record<string, any>) {
    newrelic.noticeError(error, attributes);
  }
  
  // トランザクション属性の追加
  addCustomAttribute(key: string, value: string | number | boolean) {
    newrelic.addCustomAttribute(key, value);
  }
  
  // ビジネスメトリクス
  recordBusinessMetrics(metrics: {
    userRegistrations: number;
    activeUsers: number;
    revenue: number;
    errorRate: number;
  }) {
    this.recordCustomMetric('Business/UserRegistrations', metrics.userRegistrations);
    this.recordCustomMetric('Business/ActiveUsers', metrics.activeUsers);
    this.recordCustomMetric('Business/Revenue', metrics.revenue);
    this.recordCustomMetric('Business/ErrorRate', metrics.errorRate);
  }
  
  // パフォーマンスメトリクス
  recordPerformanceMetrics(metrics: {
    databaseQueryTime: number;
    externalApiTime: number;
    cacheHitRate: number;
  }) {
    this.recordCustomMetric('Performance/DatabaseQueryTime', metrics.databaseQueryTime);
    this.recordCustomMetric('Performance/ExternalApiTime', metrics.externalApiTime);
    this.recordCustomMetric('Performance/CacheHitRate', metrics.cacheHitRate);
  }
}

export const newRelicLogger = new NewRelicLogger();

// ミドルウェアでの使用
export const newRelicMiddleware = (req: Request, res: Response, next: NextFunction) => {
  // リクエスト情報を属性として追加
  newrelic.addCustomAttribute('userId', req.user?.id);
  newrelic.addCustomAttribute('userRole', req.user?.role);
  newrelic.addCustomAttribute('requestId', req.id);
  
  // カスタムトランザクション名
  newrelic.setTransactionName('Web', `${req.method} ${req.route?.path || req.path}`);
  
  next();
};
```

### DataDog統合
```typescript
import { StatsD } from 'node-statsd';
import tracer from 'dd-trace';

// DataDog APM初期化
tracer.init({
  service: 'backend-api',
  version: process.env.SERVICE_VERSION,
  env: process.env.NODE_ENV,
});

class DataDogLogger {
  private statsD: StatsD;
  
  constructor() {
    this.statsD = new StatsD({
      host: process.env.DATADOG_AGENT_HOST || 'localhost',
      port: parseInt(process.env.DATADOG_AGENT_PORT || '8125'),
      prefix: 'backend_api.',
    });
  }
  
  // カウンター
  increment(metric: string, tags?: string[]) {
    this.statsD.increment(metric, tags);
  }
  
  // ゲージ
  gauge(metric: string, value: number, tags?: string[]) {
    this.statsD.gauge(metric, value, tags);
  }
  
  // ヒストグラム
  histogram(metric: string, value: number, tags?: string[]) {
    this.statsD.histogram(metric, value, tags);
  }
  
  // タイミング
  timing(metric: string, value: number, tags?: string[]) {
    this.statsD.timing(metric, value, tags);
  }
  
  // ビジネスメトリクス
  recordUserAction(action: string, userId: string) {
    this.increment('user.action', [`action:${action}`, `user:${userId}`]);
  }
  
  recordApiRequest(method: string, endpoint: string, statusCode: number, duration: number) {
    this.increment('api.request', [`method:${method}`, `endpoint:${endpoint}`, `status:${statusCode}`]);
    this.timing('api.response_time', duration, [`method:${method}`, `endpoint:${endpoint}`]);
  }
  
  recordDatabaseQuery(operation: string, duration: number) {
    this.timing('database.query_time', duration, [`operation:${operation}`]);
    this.increment('database.query', [`operation:${operation}`]);
  }
  
  recordCacheOperation(operation: 'hit' | 'miss', key: string) {
    this.increment('cache.operation', [`operation:${operation}`, `key:${key}`]);
  }
  
  recordError(errorType: string, severity: 'low' | 'medium' | 'high') {
    this.increment('error.count', [`type:${errorType}`, `severity:${severity}`]);
  }
}

export const dataDogLogger = new DataDogLogger();

// Express.js統合
export const dataDogMiddleware = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    dataDogLogger.recordApiRequest(
      req.method,
      req.route?.path || req.path,
      res.statusCode,
      duration
    );
  });
  
  next();
};
```

## ログ分析・アラート

### ELK Stack統合
```typescript
// Elasticsearch クライアント
import { Client } from '@elastic/elasticsearch';

class ElasticsearchLogger {
  private client: Client;
  
  constructor() {
    this.client = new Client({
      node: process.env.ELASTICSEARCH_URL || 'http://localhost:9200',
      auth: {
        username: process.env.ELASTICSEARCH_USERNAME || '',
        password: process.env.ELASTICSEARCH_PASSWORD || '',
      },
    });
  }
  
  // ログの送信
  async indexLog(index: string, document: Record<string, any>) {
    try {
      await this.client.index({
        index: `${index}-${new Date().toISOString().split('T')[0]}`, // 日付別インデックス
        body: {
          ...document,
          '@timestamp': new Date().toISOString(),
          service: process.env.SERVICE_NAME,
          environment: process.env.NODE_ENV,
        },
      });
    } catch (error) {
      console.error('Failed to index log to Elasticsearch:', error);
    }
  }
  
  // アプリケーションログ
  async logApplication(level: string, message: string, meta?: Record<string, any>) {
    await this.indexLog('application-logs', {
      level,
      message,
      ...meta,
    });
  }
  
  // アクセスログ
  async logAccess(req: Request, res: Response, duration: number) {
    await this.indexLog('access-logs', {
      method: req.method,
      url: req.originalUrl,
      status_code: res.statusCode,
      duration,
      user_agent: req.get('User-Agent'),
      ip_address: req.ip,
      user_id: req.user?.id,
      request_id: req.id,
    });
  }
  
  // エラーログ
  async logError(error: Error, context?: Record<string, any>) {
    await this.indexLog('error-logs', {
      error_message: error.message,
      error_stack: error.stack,
      error_name: error.name,
      ...context,
    });
  }
  
  // ビジネスイベント
  async logBusinessEvent(event: string, data: Record<string, any>) {
    await this.indexLog('business-events', {
      event_type: event,
      event_data: data,
    });
  }
}

export const elasticsearchLogger = new ElasticsearchLogger();
```

### アラートシステム
```typescript
// Slack通知
class SlackAlerter {
  private webhookUrl: string;
  
  constructor(webhookUrl: string) {
    this.webhookUrl = webhookUrl;
  }
  
  async sendAlert(level: 'info' | 'warning' | 'error', message: string, details?: Record<string, any>) {
    const colors = {
      info: '#36a64f',
      warning: '#ffb347',
      error: '#ff6b6b',
    };
    
    const payload = {
      attachments: [
        {
          color: colors[level],
          title: `🚨 ${level.toUpperCase()} Alert`,
          text: message,
          fields: details ? Object.entries(details).map(([key, value]) => ({
            title: key,
            value: String(value),
            short: true,
          })) : [],
          footer: process.env.SERVICE_NAME,
          ts: Math.floor(Date.now() / 1000),
        },
      ],
    };
    
    try {
      await fetch(this.webhookUrl, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
      });
    } catch (error) {
      console.error('Failed to send Slack alert:', error);
    }
  }
}

// Email通知
class EmailAlerter {
  private nodemailer = require('nodemailer');
  private transporter: any;
  
  constructor() {
    this.transporter = this.nodemailer.createTransporter({
      host: process.env.SMTP_HOST,
      port: process.env.SMTP_PORT,
      secure: process.env.SMTP_SECURE === 'true',
      auth: {
        user: process.env.SMTP_USER,
        pass: process.env.SMTP_PASS,
      },
    });
  }
  
  async sendAlert(
    level: 'info' | 'warning' | 'error',
    subject: string,
    message: string,
    recipients: string[]
  ) {
    const mailOptions = {
      from: process.env.ALERT_FROM_EMAIL,
      to: recipients.join(', '),
      subject: `[${level.toUpperCase()}] ${subject}`,
      html: `
        <h2>Alert Notification</h2>
        <p><strong>Level:</strong> ${level}</p>
        <p><strong>Service:</strong> ${process.env.SERVICE_NAME}</p>
        <p><strong>Environment:</strong> ${process.env.NODE_ENV}</p>
        <p><strong>Message:</strong> ${message}</p>
        <p><strong>Timestamp:</strong> ${new Date().toISOString()}</p>
      `,
    };
    
    try {
      await this.transporter.sendMail(mailOptions);
    } catch (error) {
      console.error('Failed to send email alert:', error);
    }
  }
}

// 統合アラートマネージャー
class AlertManager {
  private slackAlerter: SlackAlerter;
  private emailAlerter: EmailAlerter;
  
  constructor() {
    this.slackAlerter = new SlackAlerter(process.env.SLACK_WEBHOOK_URL!);
    this.emailAlerter = new EmailAlerter();
  }
  
  async sendAlert(
    level: 'info' | 'warning' | 'error',
    title: string,
    message: string,
    details?: Record<string, any>
  ) {
    // Slack通知（すべてのレベル）
    await this.slackAlerter.sendAlert(level, `${title}: ${message}`, details);
    
    // Email通知（warningとerrorのみ）
    if (level === 'warning' || level === 'error') {
      const recipients = process.env.ALERT_EMAIL_RECIPIENTS?.split(',') || [];
      await this.emailAlerter.sendAlert(level, title, message, recipients);
    }
  }
  
  // エラー率のアラート
  async checkErrorRate() {
    // 過去5分間のエラー率を計算
    const errorRate = await this.calculateErrorRate();
    
    if (errorRate > 5) { // 5%以上
      await this.sendAlert(
        'error',
        'High Error Rate Detected',
        `Error rate is ${errorRate}% over the last 5 minutes`,
        { errorRate, threshold: 5 }
      );
    } else if (errorRate > 2) { // 2%以上
      await this.sendAlert(
        'warning',
        'Elevated Error Rate',
        `Error rate is ${errorRate}% over the last 5 minutes`,
        { errorRate, threshold: 2 }
      );
    }
  }
  
  // レスポンス時間のアラート
  async checkResponseTime() {
    const avgResponseTime = await this.calculateAverageResponseTime();
    
    if (avgResponseTime > 1000) { // 1秒以上
      await this.sendAlert(
        'warning',
        'Slow Response Time',
        `Average response time is ${avgResponseTime}ms over the last 5 minutes`,
        { responseTime: avgResponseTime, threshold: 1000 }
      );
    }
  }
  
  private async calculateErrorRate(): Promise<number> {
    // 実装: 過去5分間のエラー率を計算
    return 0;
  }
  
  private async calculateAverageResponseTime(): Promise<number> {
    // 実装: 過去5分間の平均レスポンス時間を計算
    return 0;
  }
}

export const alertManager = new AlertManager();

// 定期的なヘルスチェック
setInterval(async () => {
  await alertManager.checkErrorRate();
  await alertManager.checkResponseTime();
}, 60000); // 1分ごと
```

## ログ保持・アーカイブ

### ログローテーション戦略
```typescript
// ログアーカイブ管理
class LogArchiveManager {
  private archivePath: string;
  private compressionLevel: number;
  
  constructor(archivePath: string = './archives', compressionLevel: number = 6) {
    this.archivePath = archivePath;
    this.compressionLevel = compressionLevel;
  }
  
  // 古いログファイルの圧縮
  async compressOldLogs(daysOld: number = 7) {
    const fs = require('fs').promises;
    const path = require('path');
    const zlib = require('zlib');
    
    try {
      const logDir = './logs';
      const files = await fs.readdir(logDir);
      const cutoffDate = new Date();
      cutoffDate.setDate(cutoffDate.getDate() - daysOld);
      
      for (const file of files) {
        if (file.endsWith('.log')) {
          const filePath = path.join(logDir, file);
          const stats = await fs.stat(filePath);
          
          if (stats.mtime < cutoffDate) {
            await this.compressFile(filePath);
            await fs.unlink(filePath); // 元ファイルを削除
          }
        }
      }
    } catch (error) {
      logger.error('Failed to compress old logs', { error: error.message });
    }
  }
  
  private async compressFile(filePath: string) {
    const fs = require('fs').promises;
    const path = require('path');
    const zlib = require('zlib');
    
    const data = await fs.readFile(filePath);
    const compressed = zlib.gzipSync(data, { level: this.compressionLevel });
    
    const archiveFileName = `${path.basename(filePath)}-${new Date().toISOString().split('T')[0]}.gz`;
    const archiveFilePath = path.join(this.archivePath, archiveFileName);
    
    await fs.writeFile(archiveFilePath, compressed);
  }
  
  // 古いアーカイブの削除
  async cleanupOldArchives(retentionDays: number = 90) {
    const fs = require('fs').promises;
    const path = require('path');
    
    try {
      const files = await fs.readdir(this.archivePath);
      const cutoffDate = new Date();
      cutoffDate.setDate(cutoffDate.getDate() - retentionDays);
      
      for (const file of files) {
        const filePath = path.join(this.archivePath, file);
        const stats = await fs.stat(filePath);
        
        if (stats.mtime < cutoffDate) {
          await fs.unlink(filePath);
          logger.info('Deleted old archive', { file });
        }
      }
    } catch (error) {
      logger.error('Failed to cleanup old archives', { error: error.message });
    }
  }
  
  // S3へのアップロード（オプション）
  async uploadToS3(localPath: string, s3Key: string) {
    // AWS SDK S3実装
    // 長期保存のためのクラウドストレージ統合
  }
}

// 日次実行のクリーンアップジョブ
const archiveManager = new LogArchiveManager();

// cron的な処理（実際はcronジョブで実行）
const runDailyCleanup = async () => {
  await archiveManager.compressOldLogs(7);  // 7日より古いログを圧縮
  await archiveManager.cleanupOldArchives(90); // 90日より古いアーカイブを削除
};

// 毎日午前2時に実行（実装例）
if (process.env.ENABLE_LOG_CLEANUP === 'true') {
  const cron = require('node-cron');
  cron.schedule('0 2 * * *', runDailyCleanup);
}
```

### GDPR対応ログ管理
```typescript
// GDPR準拠のログ管理
class GDPRLogManager {
  // 個人情報の匿名化
  anonymizePersonalData(logData: Record<string, any>): Record<string, any> {
    const sensitiveFields = ['email', 'name', 'phone', 'address', 'ip'];
    const anonymized = { ...logData };
    
    sensitiveFields.forEach(field => {
      if (anonymized[field]) {
        anonymized[field] = this.hash(anonymized[field]);
      }
    });
    
    return anonymized;
  }
  
  private hash(value: string): string {
    const crypto = require('crypto');
    return crypto.createHash('sha256').update(value).digest('hex').substring(0, 10);
  }
  
  // ユーザーデータの削除
  async deleteUserLogs(userId: string) {
    // Elasticsearch からの削除
    try {
      await elasticsearchLogger.client.deleteByQuery({
        index: 'application-logs-*',
        body: {
          query: {
            term: { user_id: userId }
          }
        }
      });
      
      logger.info('User logs deleted for GDPR compliance', { userId });
    } catch (error) {
      logger.error('Failed to delete user logs', { userId, error: error.message });
    }
  }
  
  // ログ保持期間のチェック
  async enforceRetentionPolicy() {
    const retentionDays = parseInt(process.env.LOG_RETENTION_DAYS || '365');
    const cutoffDate = new Date();
    cutoffDate.setDate(cutoffDate.getDate() - retentionDays);
    
    try {
      await elasticsearchLogger.client.deleteByQuery({
        index: 'application-logs-*',
        body: {
          query: {
            range: {
              '@timestamp': {
                lt: cutoffDate.toISOString()
              }
            }
          }
        }
      });
      
      logger.info('Old logs deleted per retention policy', { 
        cutoffDate: cutoffDate.toISOString(),
        retentionDays 
      });
    } catch (error) {
      logger.error('Failed to enforce retention policy', { error: error.message });
    }
  }
}

export const gdprLogManager = new GDPRLogManager();
```