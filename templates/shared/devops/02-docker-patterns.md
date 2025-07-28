# Dockerパターンガイド

## 基本的なDockerfile

### Node.js アプリケーション

```dockerfile
# マルチステージビルド
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS runtime

# セキュリティ: 非rootユーザーの作成
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001

WORKDIR /app

# 依存関係のコピー
COPY --from=builder /app/node_modules ./node_modules
COPY --chown=nextjs:nodejs . .

# ポート公開
EXPOSE 3000

# ユーザー切り替え
USER nextjs

# ヘルスチェック
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

CMD ["npm", "start"]
```

### Python アプリケーション

```dockerfile
FROM python:3.11-slim

# システムの更新とセキュリティパッチ
RUN apt-get update && apt-get install -y \
    --no-install-recommends \
    curl \
    && rm -rf /var/lib/apt/lists/*

# 非rootユーザー作成
RUN useradd --create-home --shell /bin/bash app

WORKDIR /app

# 依存関係のインストール
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# アプリケーションコードのコピー
COPY --chown=app:app . .

USER app

EXPOSE 8000

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8000/health || exit 1

CMD ["gunicorn", "--bind", "0.0.0.0:8000", "app:app"]
```

## マルチステージビルドパターン

### フロントエンド最適化

```dockerfile
# ビルドステージ
FROM node:18-alpine AS builder

WORKDIR /app
COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

# 本番ステージ
FROM nginx:alpine AS production

# カスタムnginx設定
COPY nginx.conf /etc/nginx/nginx.conf

# ビルド成果物のコピー
COPY --from=builder /app/dist /usr/share/nginx/html

# セキュリティ設定
RUN chown -R nginx:nginx /usr/share/nginx/html && \
    chmod -R 755 /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### Go アプリケーション

```dockerfile
# ビルドステージ
FROM golang:1.20-alpine AS builder

RUN apk add --no-cache git ca-certificates

WORKDIR /src
COPY go.mod go.sum ./
RUN go mod download

COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app .

# 最小本番イメージ
FROM scratch

# CA証明書のコピー
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

# アプリケーションのコピー
COPY --from=builder /src/app /app

EXPOSE 8080

ENTRYPOINT ["/app"]
```

## Docker Compose パターン

### 開発環境

```yaml
version: '3.8'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - '3000:3000'
    environment:
      - NODE_ENV=development
      - DATABASE_URL=postgresql://user:password@db:5432/myapp
    volumes:
      - .:/app
      - /app/node_modules
    depends_on:
      - db
      - redis
    restart: unless-stopped

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: user
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    ports:
      - '5432:5432'
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'
    volumes:
      - redis_data:/data
    restart: unless-stopped

  nginx:
    image: nginx:alpine
    ports:
      - '80:80'
    volumes:
      - ./nginx.dev.conf:/etc/nginx/nginx.conf
    depends_on:
      - app
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:

networks:
  default:
    driver: bridge
```

### 本番環境

```yaml
version: '3.8'

services:
  app:
    image: myapp:latest
    ports:
      - '3000:3000'
    environment:
      - NODE_ENV=production
      - DATABASE_URL=${DATABASE_URL}
      - REDIS_URL=${REDIS_URL}
    secrets:
      - db_password
      - app_secret
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    healthcheck:
      test: ['CMD', 'curl', '-f', 'http://localhost:3000/health']
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

secrets:
  db_password:
    external: true
  app_secret:
    external: true
```

## セキュリティベストプラクティス

### 最小権限の原則

```dockerfile
# 非rootユーザーでの実行
FROM node:18-alpine

# 専用ユーザー作成
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nextjs -u 1001 -G nodejs

# 必要最小限のファイルのみコピー
COPY --chown=nextjs:nodejs package*.json ./
RUN npm ci --only=production

COPY --chown=nextjs:nodejs . .

USER nextjs

# 読み取り専用ファイルシステム
CMD ["node", "server.js"]
```

### シークレット管理

```yaml
# Docker Compose with secrets
services:
  app:
    image: myapp:latest
    secrets:
      - db_password
      - api_key
    environment:
      - DB_PASSWORD_FILE=/run/secrets/db_password
      - API_KEY_FILE=/run/secrets/api_key

secrets:
  db_password:
    file: ./secrets/db_password.txt
  api_key:
    external: true
    name: myapp_api_key
```

### イメージスキャン

```bash
# Trivyでの脆弱性スキャン
trivy image myapp:latest

# Dockerベンチマークセキュリティ
docker run --rm --net host --pid host --userns host --cap-add audit_control \
  -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
  -v /etc:/etc:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  --label docker_bench_security \
  docker/docker-bench-security
```

## パフォーマンス最適化

### レイヤーキャッシュ最適化

```dockerfile
# 悪い例：依存関係の変更でレイヤーが無効化される
COPY . .
RUN npm install

# 良い例：依存関係のレイヤーを分離
COPY package*.json ./
RUN npm ci
COPY . .
```

### .dockerignore 設定

```
# .dockerignore
node_modules
npm-debug.log
.git
.gitignore
README.md
.env
.nyc_output
coverage
.coverage
.pytest_cache
__pycache__
.DS_Store
.vscode
```

### イメージサイズ最適化

```dockerfile
# Alpine Linuxの使用
FROM node:18-alpine

# 不要なパッケージの削除
RUN apk add --no-cache git && \
    npm install && \
    apk del git

# 多段階ビルドで開発依存関係を除外
FROM node:18-alpine AS deps
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-alpine AS runtime
COPY --from=deps /app/node_modules ./node_modules
```

## 監視とログ

### ヘルスチェック

```dockerfile
# アプリケーションレベルのヘルスチェック
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# データベース接続チェック
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
  CMD pg_isready -U $POSTGRES_USER -d $POSTGRES_DB || exit 1
```

### ログ設定

```yaml
services:
  app:
    image: myapp:latest
    logging:
      driver: 'json-file'
      options:
        max-size: '10m'
        max-file: '3'
    labels:
      - 'log.service=myapp'
      - 'log.env=production'
```

### メトリクス収集

```yaml
services:
  app:
    image: myapp:latest
    labels:
      - 'prometheus.io/scrape=true'
      - 'prometheus.io/port=3000'
      - 'prometheus.io/path=/metrics'

  prometheus:
    image: prom/prometheus:latest
    ports:
      - '9090:9090'
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
```

## トラブルシューティング

### デバッグコマンド

```bash
# コンテナ内でのシェル実行
docker exec -it container_name sh

# ログの確認
docker logs container_name
docker logs -f container_name  # リアルタイム

# リソース使用量の確認
docker stats container_name

# プロセス確認
docker exec container_name ps aux

# ネットワーク確認
docker network ls
docker network inspect network_name
```

### よくある問題と解決策

#### ポートバインドエラー

```bash
# ポート使用状況確認
netstat -tulpn | grep :3000
lsof -i :3000

# 強制停止
docker stop $(docker ps -q)
docker rm $(docker ps -aq)
```

#### ディスク容量不足

```bash
# 未使用リソースの削除
docker system prune -a

# ボリューム削除
docker volume prune

# イメージ削除
docker image prune -a
```

#### メモリ不足

```bash
# メモリ制限の設定
docker run --memory="512m" myapp:latest

# Docker Compose での制限
services:
  app:
    deploy:
      resources:
        limits:
          memory: 512M
```

## コンテナオーケストレーション準備

### Kubernetes準備

```yaml
# Deployment用のラベル設定
labels:
  app: myapp
  version: v1.0.0
  tier: frontend
```

### Docker Swarm対応

```yaml
version: '3.8'
services:
  app:
    image: myapp:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
```
