# 開発コマンド

## セットアップ

### 初回セットアップ
```bash
# リポジトリクローン
git clone [repository-url]
cd [project-name]

# 依存関係インストール
npm install
# または
yarn install
# または
pnpm install
```

### 環境変数設定
```bash
# 環境変数ファイルをコピー
cp .env.example .env

# 必要な環境変数を設定
# DATABASE_URL=
# API_KEY=
# [その他の環境変数]
```

## 開発

### 開発サーバー起動
```bash
npm run dev
# または
yarn dev
```

### ホットリロード
```bash
npm run dev:hot
```

### 開発用データベース
```bash
# データベース起動
npm run db:start

# マイグレーション実行
npm run db:migrate

# シードデータ投入
npm run db:seed
```

## ビルド

### 本番ビルド
```bash
npm run build
```

### ステージングビルド
```bash
npm run build:staging
```

### ビルド確認
```bash
npm run preview
```

## テスト

### 全テスト実行
```bash
npm run test
```

### ユニットテスト
```bash
npm run test:unit
```

### 統合テスト
```bash
npm run test:integration
```

### E2Eテスト
```bash
npm run test:e2e
```

### テストカバレッジ
```bash
npm run test:coverage
```

### テストウォッチモード
```bash
npm run test:watch
```

## コード品質

### リント
```bash
npm run lint
```

### リント修正
```bash
npm run lint:fix
```

### フォーマット
```bash
npm run format
```

### 型チェック
```bash
npm run type-check
```

### 全品質チェック
```bash
npm run quality-check
```

## データベース

### マイグレーション
```bash
# マイグレーション作成
npm run db:migrate:create

# マイグレーション実行
npm run db:migrate

# マイグレーション取り消し
npm run db:migrate:rollback
```

### シード
```bash
# シードデータ作成
npm run db:seed:create

# シードデータ実行
npm run db:seed
```

## デプロイメント

### ステージング環境
```bash
npm run deploy:staging
```

### 本番環境
```bash
npm run deploy:production
```

### ロールバック
```bash
npm run rollback
```

## 便利なコマンド

### ログ確認
```bash
npm run logs
```

### 依存関係更新
```bash
npm run deps:update
```

### キャッシュクリア
```bash
npm run cache:clear
```

### ドキュメント生成
```bash
npm run docs:generate
```

## Docker（該当する場合）

### 開発環境起動
```bash
docker-compose up -d
```

### 本番イメージビルド
```bash
docker build -t [image-name] .
```

### コンテナシェル
```bash
docker-compose exec app sh
```

## トラブルシューティング用コマンド

### 依存関係再インストール
```bash
rm -rf node_modules package-lock.json
npm install
```

### キャッシュクリア
```bash
npm run cache:clear
```

### ヘルスチェック
```bash
npm run health-check
```