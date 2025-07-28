# 監視・ログパターンガイド

## アプリケーション監視

### メトリクス設計
```typescript
// Prometheus メトリクス例
import { register, Counter, Histogram, Gauge } from 'prom-client';

// リクエストカウンター
const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
});

// レスポンス時間
const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route'],
  buckets: [0.1, 0.5, 1, 2, 5],
});

// アクティブ接続数
const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
});

// ミドルウェアでの計測
export const metricsMiddleware = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    
    httpRequestsTotal.inc({
      method: req.method,
      route: req.route?.path || req.path,
      status_code: res.statusCode.toString(),
    });
    
    httpRequestDuration.observe(
      {
        method: req.method,
        route: req.route?.path || req.path,
      },
      duration
    );
  });
  
  next();
};
```

### ヘルスチェックエンドポイント
```typescript
// 基本的なヘルスチェック
app.get('/health', (req, res) => {
  res.status(200).json({
    status: 'OK',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
  });
});

// 詳細なヘルスチェック
app.get('/health/detailed', async (req, res) => {
  const checks = {
    database: await checkDatabase(),
    redis: await checkRedis(),
    external_api: await checkExternalAPI(),
  };
  
  const allHealthy = Object.values(checks).every(check => check.status === 'healthy');
  
  res.status(allHealthy ? 200 : 503).json({
    status: allHealthy ? 'healthy' : 'unhealthy',
    timestamp: new Date().toISOString(),
    checks,
  });
});

async function checkDatabase(): Promise<HealthCheck> {
  try {
    await db.raw('SELECT 1');
    return { status: 'healthy', message: 'Database connection OK' };
  } catch (error) {
    return { status: 'unhealthy', message: `Database error: ${error.message}` };
  }
}
```

### カスタムメトリクス
```typescript
// ビジネスメトリクス
const userRegistrations = new Counter({
  name: 'user_registrations_total',
  help: 'Total number of user registrations',
  labelNames: ['source'],
});

const orderValue = new Histogram({
  name: 'order_value_dollars',
  help: 'Value of orders in dollars',
  buckets: [10, 50, 100, 500, 1000, 5000],
});

// 使用例
userRegistrations.inc({ source: 'web' });
orderValue.observe(123.45);
```

## ログ管理

### 構造化ログ
```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: {
    service: 'myapp',
    version: process.env.APP_VERSION,
  },
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
  ],
});

// リクエストログミドルウェア
export const requestLogger = (req: Request, res: Response, next: NextFunction) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    
    logger.info('HTTP Request', {
      method: req.method,
      url: req.originalUrl,
      status: res.statusCode,
      duration,
      userAgent: req.get('User-Agent'),
      ip: req.ip,
      userId: req.user?.id,
    });
  });
  
  next();
};

// エラーログ
export const errorLogger = (error: Error, req: Request, res: Response, next: NextFunction) => {
  logger.error('Unhandled Error', {
    error: {
      name: error.name,
      message: error.message,
      stack: error.stack,
    },
    request: {
      method: req.method,
      url: req.originalUrl,
      headers: req.headers,
      body: req.body,
      userId: req.user?.id,
    },
  });
  
  next(error);
};
```

### ログレベル戦略
```typescript
// 環境別ログレベル
const getLogLevel = () => {
  switch (process.env.NODE_ENV) {
    case 'development':
      return 'debug';
    case 'staging':
      return 'info';
    case 'production':
      return 'warn';
    default:
      return 'info';
  }
};

// コンテキスト付きロガー
class ContextualLogger {
  private context: Record<string, any>;
  
  constructor(context: Record<string, any> = {}) {
    this.context = context;
  }
  
  child(additionalContext: Record<string, any>): ContextualLogger {
    return new ContextualLogger({ ...this.context, ...additionalContext });
  }
  
  info(message: string, meta: Record<string, any> = {}) {
    logger.info(message, { ...this.context, ...meta });
  }
  
  error(message: string, error?: Error, meta: Record<string, any> = {}) {
    logger.error(message, {
      ...this.context,
      ...meta,
      error: error ? {
        name: error.name,
        message: error.message,
        stack: error.stack,
      } : undefined,
    });
  }
}

// 使用例
const userLogger = new ContextualLogger({ userId: '12345', feature: 'user-management' });
userLogger.info('User profile updated');
```

## アラート設定

### Prometheus アラートルール
```yaml
# alert-rules.yml
groups:
  - name: application.rules
    rules:
      - alert: HighErrorRate
        expr: |
          (
            rate(http_requests_total{status_code=~"5.."}[5m]) /
            rate(http_requests_total[5m])
          ) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value | humanizePercentage }} for the last 5 minutes"

      - alert: HighResponseTime
        expr: |
          histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High response time detected"
          description: "95th percentile response time is {{ $value }}s"

      - alert: DatabaseConnectionsHigh
        expr: pg_stat_database_numbackends > 80
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High number of database connections"
          description: "Database has {{ $value }} active connections"

      - alert: ServiceDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service is down"
          description: "{{ $labels.instance }} is down"
```

### Alertmanager設定
```yaml
# alertmanager.yml
global:
  smtp_smarthost: 'localhost:587'
  smtp_from: 'alerts@mycompany.com'

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'web.hook'
  routes:
    - match:
        severity: critical
      receiver: 'critical-alerts'
    - match:
        severity: warning
      receiver: 'warning-alerts'

receivers:
  - name: 'web.hook'
    webhook_configs:
      - url: 'http://127.0.0.1:5001/'

  - name: 'critical-alerts'
    email_configs:
      - to: 'oncall@mycompany.com'
        subject: 'CRITICAL: {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Description: {{ .Annotations.description }}
          {{ end }}
    slack_configs:
      - api_url: 'YOUR_SLACK_WEBHOOK_URL'
        channel: '#alerts'
        title: 'Critical Alert'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'

  - name: 'warning-alerts'
    email_configs:
      - to: 'team@mycompany.com'
        subject: 'WARNING: {{ .GroupLabels.alertname }}'
```

## ダッシュボード設計

### Grafana ダッシュボード
```json
{
  "dashboard": {
    "title": "Application Overview",
    "panels": [
      {
        "title": "Request Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[1m]))",
            "legendFormat": "Requests/sec"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "reqps"
          }
        }
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status_code=~\"5..\"}[5m])) / sum(rate(http_requests_total[5m]))",
            "legendFormat": "Error Rate"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percentunit",
            "max": 1,
            "thresholds": {
              "steps": [
                {"color": "green", "value": 0},
                {"color": "yellow", "value": 0.01},
                {"color": "red", "value": 0.05}
              ]
            }
          }
        }
      },
      {
        "title": "Response Time Percentiles",
        "type": "timeseries",
        "targets": [
          {
            "expr": "histogram_quantile(0.50, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "50th percentile"
          },
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "95th percentile"
          },
          {
            "expr": "histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))",
            "legendFormat": "99th percentile"
          }
        ]
      }
    ]
  }
}
```

### SLI/SLO ダッシュボード
```json
{
  "dashboard": {
    "title": "SLI/SLO Dashboard",
    "panels": [
      {
        "title": "API Availability SLI",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{status_code!~\"5..\"}[30d])) / sum(rate(http_requests_total[30d]))",
            "legendFormat": "30-day availability"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "percentunit",
            "thresholds": {
              "steps": [
                {"color": "red", "value": 0},
                {"color": "yellow", "value": 0.995},
                {"color": "green", "value": 0.999}
              ]
            }
          }
        }
      },
      {
        "title": "Latency SLI",
        "type": "stat",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[30d]))",
            "legendFormat": "95th percentile latency"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s",
            "thresholds": {
              "steps": [
                {"color": "green", "value": 0},
                {"color": "yellow", "value": 1},
                {"color": "red", "value": 2}
              ]
            }
          }
        }
      }
    ]
  }
}
```

## 分散トレーシング

### OpenTelemetry 設定
```typescript
import { NodeSDK } from '@opentelemetry/sdk-node';
import { getNodeAutoInstrumentations } from '@opentelemetry/auto-instrumentations-node';
import { JaegerExporter } from '@opentelemetry/exporter-jaeger';

const jaegerExporter = new JaegerExporter({
  endpoint: process.env.JAEGER_ENDPOINT,
});

const sdk = new NodeSDK({
  traceExporter: jaegerExporter,
  instrumentations: [getNodeAutoInstrumentations()],
});

sdk.start();

// カスタムスパン
import { trace } from '@opentelemetry/api';

const tracer = trace.getTracer('myapp');

export async function processOrder(orderId: string) {
  const span = tracer.startSpan('process-order');
  
  try {
    span.setAttributes({
      'order.id': orderId,
      'operation.name': 'process-order',
    });
    
    // ビジネスロジック
    const order = await fetchOrder(orderId);
    span.addEvent('order-fetched', { 'order.status': order.status });
    
    await validateOrder(order);
    span.addEvent('order-validated');
    
    await updateInventory(order);
    span.addEvent('inventory-updated');
    
    span.setStatus({ code: trace.SpanStatusCode.OK });
    return order;
  } catch (error) {
    span.setStatus({
      code: trace.SpanStatusCode.ERROR,
      message: error.message,
    });
    throw error;
  } finally {
    span.end();
  }
}
```

## パフォーマンス監視

### アプリケーションパフォーマンス監視
```typescript
// メモリ使用量監視
const memoryUsage = new Gauge({
  name: 'nodejs_memory_usage_bytes',
  help: 'Node.js memory usage',
  labelNames: ['type'],
  collect() {
    const usage = process.memoryUsage();
    this.set({ type: 'rss' }, usage.rss);
    this.set({ type: 'heapUsed' }, usage.heapUsed);
    this.set({ type: 'heapTotal' }, usage.heapTotal);
    this.set({ type: 'external' }, usage.external);
  },
});

// イベントループ遅延監視
import { monitorEventLoopDelay } from 'perf_hooks';

const histogram = monitorEventLoopDelay({ resolution: 20 });
histogram.enable();

const eventLoopDelay = new Gauge({
  name: 'nodejs_eventloop_delay_seconds',
  help: 'Event loop delay',
  collect() {
    this.set(histogram.mean / 1000000000);
  },
});

// GC メトリクス
import v8 from 'v8';

const gcMetrics = new Counter({
  name: 'nodejs_gc_runs_total',
  help: 'Total number of garbage collection runs',
  labelNames: ['type'],
});

// GC 統計の定期収集
setInterval(() => {
  const stats = v8.getHeapStatistics();
  // メトリクスを更新
}, 10000);
```

## ログ集約

### ELK Stack 設定
```yaml
# logstash.conf
input {
  beats {
    port => 5044
  }
}

filter {
  if [fields][log_type] == "application" {
    json {
      source => "message"
    }
    
    date {
      match => [ "timestamp", "ISO8601" ]
    }
    
    if [level] == "error" {
      mutate {
        add_tag => ["error"]
      }
    }
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "application-logs-%{+YYYY.MM.dd}"
  }
}
```

### Filebeat 設定
```yaml
# filebeat.yml
filebeat.inputs:
- type: log
  enabled: true
  paths:
    - /var/log/myapp/*.log
  fields:
    log_type: application
  multiline.pattern: '^\d{4}-\d{2}-\d{2}'
  multiline.negate: true
  multiline.match: after

output.logstash:
  hosts: ["logstash:5044"]

processors:
  - add_host_metadata: ~
  - add_docker_metadata: ~
```

## セキュリティ監視

### セキュリティメトリクス
```typescript
// 認証失敗監視
const authFailures = new Counter({
  name: 'auth_failures_total',
  help: 'Total authentication failures',
  labelNames: ['reason', 'ip'],
});

// 異常なアクセスパターン検出
const suspiciousActivity = new Counter({
  name: 'suspicious_activity_total',
  help: 'Suspicious activity detected',
  labelNames: ['type', 'severity'],
});

// レート制限違反
const rateLimitViolations = new Counter({
  name: 'rate_limit_violations_total',
  help: 'Rate limit violations',
  labelNames: ['endpoint', 'ip'],
});

// セキュリティミドルウェア
export const securityMonitoring = (req: Request, res: Response, next: NextFunction) => {
  // 異常なリクエストパターンの検出
  if (req.path.includes('../') || req.path.includes('..\\')) {
    suspiciousActivity.inc({
      type: 'path_traversal_attempt',
      severity: 'high',
    });
  }
  
  // SQLインジェクション試行の検出
  const sqlPatterns = /(union|select|insert|update|delete|drop|create|alter)/i;
  if (sqlPatterns.test(JSON.stringify(req.query) + JSON.stringify(req.body))) {
    suspiciousActivity.inc({
      type: 'sql_injection_attempt',
      severity: 'critical',
    });
  }
  
  next();
};
```

## トラブルシューティング

### 監視システムの健全性確認
```bash
# Prometheus の健全性確認
curl http://prometheus:9090/-/healthy

# メトリクスエンドポイントの確認
curl http://app:3000/metrics

# Grafana ダッシュボードのインポート
curl -X POST \
  http://admin:admin@grafana:3000/api/dashboards/db \
  -H 'Content-Type: application/json' \
  -d @dashboard.json
```

### よくある問題と解決策

#### メトリクスが表示されない
```bash
# Prometheus のターゲット確認
curl http://prometheus:9090/api/v1/targets

# サービスディスカバリの確認
kubectl get pods -l app=myapp -o wide

# ネットワーク接続確認
kubectl exec -it prometheus-pod -- wget -qO- http://myapp:3000/metrics
```

#### ログが収集されない
```bash
# Filebeat の状態確認
kubectl logs -l app=filebeat

# Logstash のパイプライン確認
curl http://logstash:9600/_node/stats/pipelines

# Elasticsearch のインデックス確認
curl http://elasticsearch:9200/_cat/indices
```