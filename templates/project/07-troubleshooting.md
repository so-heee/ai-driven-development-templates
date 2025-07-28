# トラブルシューティング

## よくある問題と解決策

### 開発環境の問題

#### Node.jsバージョン不一致

**症状**: `npm install` でエラーが発生

```bash
Error: The engine "node" is incompatible with this module.
```

**解決策**:

```bash
# Node.jsバージョン確認
node --version

# nvmでバージョン切り替え
nvm use 18
# または
nvm install 18
nvm use 18

# package.jsonのenginesフィールドを確認
cat package.json | grep -A3 "engines"
```

#### 依存関係の競合

**症状**: パッケージインストール時の依存関係エラー

```bash
npm ERR! peer dep missing: react@^18.0.0
```

**解決策**:

```bash
# 依存関係を強制解決
npm install --force

# または clean install
rm -rf node_modules package-lock.json
npm install

# yarn の場合
rm -rf node_modules yarn.lock
yarn install
```

#### ポート衝突

**症状**: サーバー起動時のポートエラー

```bash
Error: listen EADDRINUSE: address already in use :::3000
```

**解決策**:

```bash
# ポート使用状況確認
lsof -i :3000
netstat -tulpn | grep :3000

# プロセス終了
kill -9 [PID]

# 別ポートで起動
PORT=3001 npm run dev
```

### ビルド・デプロイの問題

#### TypeScript型エラー

**症状**: ビルド時の型エラー

```bash
src/components/UserCard.tsx(15,7): error TS2322: Type 'string' is not assignable to type 'number'.
```

**解決策**:

```typescript
// 型定義を確認・修正
interface User {
  id: string // number から string に修正
  name: string
}

// 型アサーションを使用（最後の手段）
const userId = response.data.id as string
```

#### 環境変数未設定

**症状**: アプリケーション起動時のエラー

```bash
Error: Environment variable DATABASE_URL is required
```

**解決策**:

```bash
# .env ファイル作成
cp .env.example .env

# 必要な環境変数を設定
echo "DATABASE_URL=postgresql://user:pass@localhost:5432/db" >> .env

# 環境変数確認
echo $DATABASE_URL
```

#### メモリ不足

**症状**: ビルド時のメモリエラー

```bash
FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory
```

**解決策**:

```bash
# Node.jsヒープサイズを増加
export NODE_OPTIONS="--max-old-space-size=4096"
npm run build

# package.jsonでスクリプト修正
"build": "NODE_OPTIONS='--max-old-space-size=4096' next build"
```

### データベースの問題

#### 接続エラー

**症状**: データベース接続失敗

```bash
Error: connect ECONNREFUSED 127.0.0.1:5432
```

**解決策**:

```bash
# PostgreSQL サービス確認
sudo systemctl status postgresql
sudo systemctl start postgresql

# Docker での起動
docker-compose up -d db

# 接続テスト
psql -h localhost -U username -d database_name
```

#### マイグレーションエラー

**症状**: マイグレーション実行時のエラー

```bash
Error: relation "users" already exists
```

**解決策**:

```bash
# マイグレーション状態確認
npm run db:migrate:status

# 特定のマイグレーションまでロールバック
npm run db:migrate:rollback --to=20240101000000

# 強制的にマイグレーション再実行
npm run db:migrate:reset
npm run db:migrate
```

#### データ整合性エラー

**症状**: 外部キー制約違反

```sql
ERROR: insert or update on table "posts" violates foreign key constraint "posts_user_id_fkey"
```

**解決策**:

```sql
-- 参照されているデータが存在するか確認
SELECT id FROM users WHERE id = 'target_user_id';

-- 孤立したレコードを削除
DELETE FROM posts WHERE user_id NOT IN (SELECT id FROM users);

-- 制約を一時的に無効化（緊急時のみ）
ALTER TABLE posts DISABLE TRIGGER ALL;
-- データ修正後
ALTER TABLE posts ENABLE TRIGGER ALL;
```

### パフォーマンスの問題

#### ページ読み込み遅延

**症状**: 初回読み込みが遅い

**診断方法**:

```bash
# バンドルサイズ分析
npm run build -- --analyze

# Lighthouse でパフォーマンス測定
npx lighthouse http://localhost:3000

# ネットワーク分析
curl -w "@curl-format.txt" -o /dev/null -s http://localhost:3000
```

**解決策**:

```typescript
// コード分割
const LazyComponent = React.lazy(() => import('./LazyComponent'));

// 画像最適化
import Image from 'next/image';
<Image src="/image.jpg" alt="説明" width={500} height={300} />

// 不要なライブラリの削除
npm uninstall unused-library
```

#### メモリリーク

**症状**: 長時間実行後のメモリ使用量増加

**診断方法**:

```typescript
// メモリ使用量監視
setInterval(() => {
  const usage = process.memoryUsage()
  console.log('Memory usage:', {
    rss: Math.round(usage.rss / 1024 / 1024) + 'MB',
    heapUsed: Math.round(usage.heapUsed / 1024 / 1024) + 'MB',
  })
}, 30000)
```

**解決策**:

```typescript
// イベントリスナーの適切な削除
useEffect(() => {
  const handleResize = () => {
    /* ... */
  }
  window.addEventListener('resize', handleResize)

  return () => {
    window.removeEventListener('resize', handleResize)
  }
}, [])

// タイマーのクリアアップ
useEffect(() => {
  const timer = setInterval(() => {
    /* ... */
  }, 1000)
  return () => clearInterval(timer)
}, [])
```

### API・ネットワークの問題

#### CORS エラー

**症状**: ブラウザでCORSエラー

```bash
Access to fetch at 'http://api.example.com' from origin 'http://localhost:3000' has been blocked by CORS policy
```

**解決策**:

```typescript
// Express.js でCORS設定
import cors from 'cors'

app.use(
  cors({
    origin: process.env.NODE_ENV === 'production' ? 'https://example.com' : 'http://localhost:3000',
    credentials: true,
  })
)
```

#### API タイムアウト

**症状**: API呼び出しがタイムアウト

**解決策**:

```typescript
// タイムアウト設定
const api = axios.create({
  timeout: 10000, // 10秒
  retry: 3,
})

// 指数バックオフでリトライ
const retryRequest = async (fn: () => Promise<any>, retries = 3) => {
  try {
    return await fn()
  } catch (error) {
    if (retries > 0) {
      await new Promise(resolve => setTimeout(resolve, 1000 * (4 - retries)))
      return retryRequest(fn, retries - 1)
    }
    throw error
  }
}
```

#### 認証エラー

**症状**: API呼び出しで401エラー

**診断・解決策**:

```typescript
// JWTトークンの有効性確認
const verifyToken = (token: string) => {
  try {
    const decoded = jwt.verify(token, process.env.JWT_SECRET!)
    console.log('Token valid:', decoded)
    return decoded
  } catch (error) {
    console.error('Token invalid:', error.message)
    // トークン更新または再ログイン
  }
}

// Axios インターセプターでトークン自動更新
api.interceptors.response.use(
  response => response,
  async error => {
    if (error.response?.status === 401) {
      await refreshToken()
      return api.request(error.config)
    }
    return Promise.reject(error)
  }
)
```

## デバッグツールと手法

### ログ分析

```bash
# アプリケーションログ確認
tail -f logs/application.log

# エラーログのみ抽出
grep "ERROR" logs/application.log

# 特定の時間範囲のログ
awk '/2024-01-01 10:00/,/2024-01-01 11:00/' logs/application.log
```

### データベースデバッグ

```sql
-- 実行中のクエリ確認
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes';

-- ロック確認
SELECT * FROM pg_locks
WHERE NOT granted;

-- インデックス使用率確認
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC;
```

### パフォーマンス分析

```typescript
// 実行時間測定
console.time('operation')
await expensiveOperation()
console.timeEnd('operation')

// プロファイリング
const profiler = require('clinic/profiler')
profiler.start()
// アプリケーション実行
profiler.stop()
```

## 緊急時対応

### サービス停止時の対応

1. **即座に対応チームに連絡**
2. **サービス状態確認**
3. **ログ確認でエラー原因特定**
4. **一時的な回避策実施**
5. **根本原因の調査・修正**
6. **事後報告書作成**

### ロールバック手順

```bash
# アプリケーションのロールバック
git revert [commit-hash]
npm run deploy

# データベースのロールバック
npm run db:migrate:rollback

# 設定ファイルの復元
cp backup/config.json config/production.json
```

### 連絡先・エスカレーション

| 問題レベル | 対応者       | 連絡先             | 対応時間   |
| ---------- | ------------ | ------------------ | ---------- |
| Level 1    | 開発者       | Slack #dev-support | 営業時間内 |
| Level 2    | チームリード | 電話・Slack        | 24時間以内 |
| Level 3    | SRE チーム   | オンコール         | 即座       |
| Level 4    | 経営陣       | 緊急連絡網         | 即座       |

## 予防策

### 監視・アラート設定

```yaml
# 監視項目
- CPU使用率 > 80%
- メモリ使用率 > 85%
- ディスク使用率 > 90%
- レスポンス時間 > 2秒
- エラー率 > 5%
- データベース接続数 > 80%
```

### 定期ヘルスチェック

```bash
#!/bin/bash
# daily-health-check.sh

echo "=== Daily Health Check ==="
echo "Date: $(date)"

# ディスク容量確認
echo "Disk usage:"
df -h

# メモリ使用量確認
echo "Memory usage:"
free -h

# ログエラー件数確認
echo "Error count in last 24h:"
grep "ERROR" /var/log/app.log | grep "$(date -d '1 day ago' '+%Y-%m-%d')" | wc -l

# データベース接続確認
echo "Database connection:"
psql -c "SELECT 1;" >/dev/null 2>&1 && echo "OK" || echo "FAIL"
```

### バックアップ戦略

```bash
# 自動バックアップスクリプト
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)

# データベースバックアップ
pg_dump $DATABASE_URL > "db_backup_${DATE}.sql"

# ファイルバックアップ
tar -czf "files_backup_${DATE}.tar.gz" uploads/

# クラウドストレージに保存
aws s3 cp "db_backup_${DATE}.sql" s3://backups/
aws s3 cp "files_backup_${DATE}.tar.gz" s3://backups/

# 古いバックアップを削除（30日以上）
find . -name "*backup_*.sql" -mtime +30 -delete
```
