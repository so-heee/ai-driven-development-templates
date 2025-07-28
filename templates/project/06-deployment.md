# デプロイメント・運用

## 環境構成

### 環境一覧

| 環境        | 用途         | URL                         | ブランチ   | 自動デプロイ |
| ----------- | ------------ | --------------------------- | ---------- | ------------ |
| Local       | 開発         | http://localhost:3000       | -          | ❌           |
| Development | 開発統合     | https://dev.example.com     | develop    | ✅           |
| Staging     | 本番前テスト | https://staging.example.com | release/\* | ✅           |
| Production  | 本番         | https://example.com         | main       | ✅           |

### 環境別設定

```bash
# Development
NODE_ENV=development
DATABASE_URL=postgresql://dev_user:password@db-dev:5432/app_dev
REDIS_URL=redis://redis-dev:6379
LOG_LEVEL=debug

# Staging
NODE_ENV=staging
DATABASE_URL=postgresql://staging_user:password@db-staging:5432/app_staging
REDIS_URL=redis://redis-staging:6379
LOG_LEVEL=info

# Production
NODE_ENV=production
DATABASE_URL=postgresql://prod_user:password@db-prod:5432/app_prod
REDIS_URL=redis://redis-prod:6379
LOG_LEVEL=warn
```

## CI/CD パイプライン

### GitHub Actions ワークフロー

```yaml
# .github/workflows/deploy.yml
name: Deploy
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'

      - run: npm ci
      - run: npm run lint
      - run: npm run type-check
      - run: npm run test
      - run: npm run build

  deploy-staging:
    needs: test
    if: github.ref == 'refs/heads/develop'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: |
          # Staging deployment script

  deploy-production:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Production
        run: |
          # Production deployment script
```

### パイプライン段階

1. **コード品質チェック**
   - ESLint による静的解析
   - Prettier によるフォーマットチェック
   - TypeScript 型チェック

2. **テスト実行**
   - 単体テスト (Jest)
   - 統合テスト
   - E2E テスト (Playwright)

3. **ビルド**
   - 本番ビルド実行
   - アセット最適化
   - Docker イメージ作成

4. **デプロイ前チェック**
   - セキュリティスキャン
   - 依存関係チェック
   - パフォーマンステスト

5. **デプロイ実行**
   - Blue-Green デプロイ
   - ヘルスチェック
   - ロールバック準備

## Docker 設定

### Dockerfile

```dockerfile
# Multi-stage build
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS runtime
WORKDIR /app

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nextjs -u 1001

# Copy built application
COPY --from=builder /app/node_modules ./node_modules
COPY --chown=nextjs:nodejs . .

USER nextjs
EXPOSE 3000
CMD ["npm", "start"]
```

### docker-compose.yml

```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - '3000:3000'
    environment:
      - NODE_ENV=production
      - DATABASE_URL=postgresql://user:password@db:5432/app
    depends_on:
      - db
      - redis

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes:
      - redis_data:/data

volumes:
  postgres_data:
  redis_data:
```

## クラウドデプロイ

### AWS 構成例

```yaml
# serverless.yml (Serverless Framework)
service: myapp
frameworkVersion: '3'

provider:
  name: aws
  runtime: nodejs18.x
  region: ap-northeast-1
  stage: ${opt:stage, 'dev'}

  environment:
    NODE_ENV: ${self:provider.stage}
    DATABASE_URL: ${env:DATABASE_URL}

  iam:
    role:
      statements:
        - Effect: Allow
          Action:
            - dynamodb:Query
            - dynamodb:Scan
            - dynamodb:GetItem
            - dynamodb:PutItem
          Resource: 'arn:aws:dynamodb:*:*:table/*'

functions:
  api:
    handler: dist/lambda.handler
    events:
      - http:
          path: /{proxy+}
          method: ANY
          cors: true

resources:
  Resources:
    UserTable:
      Type: AWS::DynamoDB::Table
      Properties:
        TableName: ${self:provider.stage}-users
        BillingMode: PAY_PER_REQUEST
```

### Vercel 設定例

```json
{
  "version": 2,
  "builds": [
    {
      "src": "package.json",
      "use": "@vercel/next"
    }
  ],
  "env": {
    "DATABASE_URL": "@database-url",
    "JWT_SECRET": "@jwt-secret"
  },
  "regions": ["nrt1"]
}
```

## データベース管理

### マイグレーション

```bash
# 新しいマイグレーション作成
npm run db:migrate:create add_user_profile

# マイグレーション実行
npm run db:migrate

# 本番環境マイグレーション
NODE_ENV=production npm run db:migrate
```

### バックアップ戦略

```bash
# 定期バックアップ (cron)
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
pg_dump $DATABASE_URL > "backup_${DATE}.sql"
aws s3 cp "backup_${DATE}.sql" s3://my-backups/
rm "backup_${DATE}.sql"
```

### シードデータ

```bash
# 開発環境用シードデータ
npm run db:seed

# 本番環境用初期データ
NODE_ENV=production npm run db:seed:production
```

## 監視・ログ

### アプリケーション監視

```typescript
// ヘルスチェックエンドポイント
app.get('/health', async (req, res) => {
  try {
    // データベース接続チェック
    await db.raw('SELECT 1')

    // Redis接続チェック
    await redis.ping()

    res.json({
      status: 'ok',
      timestamp: new Date().toISOString(),
      uptime: process.uptime(),
      memory: process.memoryUsage(),
    })
  } catch (error) {
    res.status(503).json({
      status: 'error',
      error: error.message,
    })
  }
})
```

### ログ設定

```typescript
// Winston ログ設定
import winston from 'winston'

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  defaultMeta: { service: 'myapp' },
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
    ...(process.env.NODE_ENV !== 'production' ? [new winston.transports.Console()] : []),
  ],
})
```

### メトリクス収集

```typescript
// Prometheus メトリクス
import promClient from 'prom-client'

const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
})

// ミドルウェア
app.use((req, res, next) => {
  const start = Date.now()

  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000
    httpRequestDuration.labels(req.method, req.route?.path || req.path, res.statusCode).observe(duration)
  })

  next()
})
```

## セキュリティ

### SSL/TLS 設定

```nginx
# Nginx 設定例
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/private.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### 環境変数管理

```bash
# .env.example
NODE_ENV=production
PORT=3000
DATABASE_URL=postgresql://user:password@localhost:5432/db
JWT_SECRET=your-super-secret-key
API_KEY=your-api-key
```

### セキュリティヘッダー

```typescript
// セキュリティミドルウェア
app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
        scriptSrc: ["'self'"],
        imgSrc: ["'self'", 'data:', 'https:'],
      },
    },
    hsts: {
      maxAge: 31536000,
      includeSubDomains: true,
      preload: true,
    },
  })
)
```

## 運用手順

### デプロイ手順

1. **デプロイ前チェック**
   - 機能テスト完了確認
   - パフォーマンステスト実施
   - セキュリティチェック

2. **デプロイ実行**
   - メンテナンスモード有効化
   - データベースマイグレーション
   - アプリケーションデプロイ
   - ヘルスチェック確認

3. **デプロイ後確認**
   - 主要機能動作確認
   - ログエラーチェック
   - メトリクス監視

### ロールバック手順

```bash
# アプリケーションロールバック
kubectl rollout undo deployment/myapp

# データベースロールバック
npm run db:migrate:rollback

# 設定確認
kubectl get pods
kubectl logs -f deployment/myapp
```

### 緊急対応

1. **アラート受信**
2. **現状確認**
3. **原因調査**
4. **一時対処** (サービス復旧優先)
5. **根本対策**
6. **事後分析**

### 定期メンテナンス

- **日次**: ログローテーション、バックアップ確認
- **週次**: セキュリティアップデート確認
- **月次**: パフォーマンス分析、依存関係更新
- **四半期**: セキュリティ監査、災害復旧テスト
