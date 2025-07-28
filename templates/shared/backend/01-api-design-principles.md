# API設計原則

## RESTful API設計の基本原則

### リソース設計
- **名詞を使用**: URL はリソースを表現（動詞ではなく）
- **複数形を使用**: `/users`、`/posts`（単数形 `/user` ではなく）
- **階層構造**: ネストしたリソースは適切な階層で表現
- **一貫性**: 命名規則・構造を全体で統一

```
✅ 良い例
GET    /api/v1/users              # ユーザー一覧取得
POST   /api/v1/users              # ユーザー作成
GET    /api/v1/users/{id}         # 特定ユーザー取得
PUT    /api/v1/users/{id}         # ユーザー全体更新
PATCH  /api/v1/users/{id}         # ユーザー部分更新
DELETE /api/v1/users/{id}         # ユーザー削除

# ネストしたリソース
GET    /api/v1/users/{id}/posts   # ユーザーの投稿一覧
POST   /api/v1/users/{id}/posts   # ユーザーの投稿作成

❌ 悪い例
GET    /api/v1/getUsers
POST   /api/v1/createUser
GET    /api/v1/user/1/post/all
```

### HTTPメソッドの適切な使用
- **GET**: リソース取得（べき等・安全）
- **POST**: リソース作成・非べき等操作
- **PUT**: リソース全体更新（べき等）
- **PATCH**: リソース部分更新
- **DELETE**: リソース削除（べき等）

### ステータスコードの標準使用
- **2xx 成功**: 200 OK、201 Created、204 No Content
- **4xx クライアントエラー**: 400 Bad Request、401 Unauthorized、403 Forbidden、404 Not Found
- **5xx サーバーエラー**: 500 Internal Server Error、502 Bad Gateway、503 Service Unavailable

### バージョニング戦略
- **URL パス**: `/api/v1/users`、`/api/v2/users`
- **ヘッダー**: `Accept: application/vnd.api+json;version=1`
- **クエリパラメータ**: `/api/users?version=1`

### レスポンス設計
#### 一貫したレスポンス形式
```json
{
  "success": true,
  "data": {
    "id": 1,
    "name": "John Doe"
  },
  "meta": {
    "timestamp": "2025-01-27T10:00:00Z",
    "version": "1.0"
  }
}

// エラー時
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid input data",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format"
      }
    ]
  },
  "meta": {
    "timestamp": "2025-01-27T10:00:00Z"
  }
}
```

### ページネーション
```json
{
  "data": [...],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  }
}
```

### フィルタリング・ソート・検索
- **フィルタリング**: `/api/v1/users?status=active&role=admin`
- **ソート**: `/api/v1/users?sort=created_at&order=desc`
- **検索**: `/api/v1/users?search=john&fields=name,email`

## GraphQL設計原則

### スキーマ設計
- **単一責任**: 各フィールドは明確な責任を持つ
- **null許可**: 必要な場合のみnon-nullにする
- **階層構造**: 関連するデータは適切にネスト

### クエリ最適化
- **N+1問題対策**: DataLoaderパターンの活用
- **深い階層制限**: 無限再帰防止
- **コスト分析**: 複雑なクエリの制限

### エラーハンドリング
```json
{
  "errors": [
    {
      "message": "User not found",
      "locations": [{"line": 2, "column": 3}],
      "path": ["user"],
      "extensions": {
        "code": "USER_NOT_FOUND",
        "exception": {
          "stacktrace": [...]
        }
      }
    }
  ],
  "data": {
    "user": null
  }
}
```

## セキュリティ原則

### 認証・認可
- **最小権限の原則**: 必要最小限のアクセス権のみ付与
- **トークンベース認証**: JWT・OAuth 2.0の活用
- **RBAC**: ロールベースアクセス制御の実装

### 入力検証
- **ホワイトリスト検証**: 許可された値のみ受け入れ
- **型安全性**: 厳密な型チェック
- **サニタイゼーション**: 危険な文字列の無害化

### レート制限
- **API制限**: リクエスト数・頻度制限
- **IP制限**: 送信元IPベースの制限
- **ユーザー制限**: ユーザー単位での制限

## パフォーマンス原則

### キャッシュ戦略
- **HTTP キャッシュ**: ETag、Cache-Control ヘッダー
- **アプリケーションキャッシュ**: Redis・Memcached
- **CDN**: 静的コンテンツ配信の最適化

### データベース最適化
- **インデックス**: 適切なインデックス設計
- **クエリ最適化**: N+1問題、過度なJOINの回避
- **コネクションプール**: データベース接続の効率化

### 非同期処理
- **バックグラウンドジョブ**: 重い処理の非同期化
- **イベント駆動**: リアルタイム更新の実装
- **ストリーミング**: 大容量データの段階的処理

## 監視・ログ設計

### メトリクス収集
- **レスポンス時間**: API パフォーマンス監視
- **エラー率**: 可用性監視
- **リクエスト数**: 使用量監視

### ログ構造化
- **構造化ログ**: JSON形式での出力
- **相関ID**: リクエスト追跡のためのID
- **ログレベル**: ERROR、WARN、INFO、DEBUGの適切な使い分け

### ヘルスチェック
```json
{
  "status": "healthy",
  "timestamp": "2025-01-27T10:00:00Z",
  "version": "1.2.3",
  "services": {
    "database": "healthy",
    "redis": "healthy",
    "external_api": "degraded"
  }
}
```

## ドキュメント化

### OpenAPI/Swagger
- **完全な仕様**: 全エンドポイントの詳細記述
- **例の提供**: リクエスト・レスポンス例
- **自動生成**: コードからの自動更新

### API ガイドライン
- **使用例**: 一般的なユースケース
- **エラー対応**: エラーハンドリング方法
- **制限事項**: レート制限・制約の明記

---

**言語固有の実装パターンは以下を参照：**
- [Go実装パターン](./go/)
- [TypeScript実装パターン](./typescript/)