# API仕様

## API概要

### ベースURL
- **開発環境**: `http://localhost:3000/api`
- **ステージング**: `https://staging-api.example.com/api`
- **本番環境**: `https://api.example.com/api`

### バージョニング
- **現在のバージョン**: v1
- **URL形式**: `/api/v1/`
- **バージョン管理**: ヘッダーベース（推奨）

### 認証方式
- **認証タイプ**: JWT Bearer Token
- **ヘッダー**: `Authorization: Bearer <token>`
- **トークン有効期限**: 24時間

## 共通仕様

### レスポンス形式
```json
{
  "success": true,
  "data": {
    // レスポンスデータ
  },
  "message": "操作が正常に完了しました",
  "timestamp": "2024-01-01T00:00:00Z"
}
```

### エラーレスポンス
```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "入力データが無効です",
    "details": [
      {
        "field": "email",
        "message": "有効なメールアドレスを入力してください"
      }
    ]
  },
  "timestamp": "2024-01-01T00:00:00Z"
}
```

### HTTPステータスコード
- `200 OK`: 成功
- `201 Created`: 作成成功
- `400 Bad Request`: 不正なリクエスト
- `401 Unauthorized`: 認証エラー
- `403 Forbidden`: 権限エラー
- `404 Not Found`: リソースが見つからない
- `422 Unprocessable Entity`: バリデーションエラー
- `500 Internal Server Error`: サーバーエラー

## 認証API

### ログイン
```http
POST /api/v1/auth/login
```

**リクエスト**
```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**レスポンス**
```json
{
  "success": true,
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "refresh-token-here",
    "user": {
      "id": 1,
      "email": "user@example.com",
      "name": "ユーザー名",
      "role": "user"
    }
  }
}
```

### ログアウト
```http
POST /api/v1/auth/logout
```

### トークン更新
```http
POST /api/v1/auth/refresh
```

### パスワードリセット
```http
POST /api/v1/auth/password-reset
```

## ユーザーAPI

### ユーザー一覧取得
```http
GET /api/v1/users?page=1&limit=10&search=keyword
```

**クエリパラメータ**
- `page`: ページ番号（デフォルト: 1）
- `limit`: 1ページあたりの件数（デフォルト: 10、最大: 100）
- `search`: 検索キーワード
- `sort`: ソート項目（name, email, created_at）
- `order`: ソート順（asc, desc）

**レスポンス**
```json
{
  "success": true,
  "data": {
    "users": [
      {
        "id": 1,
        "email": "user@example.com",
        "name": "ユーザー名",
        "role": "user",
        "createdAt": "2024-01-01T00:00:00Z"
      }
    ],
    "pagination": {
      "current": 1,
      "total": 100,
      "pages": 10,
      "limit": 10
    }
  }
}
```

### ユーザー詳細取得
```http
GET /api/v1/users/{id}
```

### ユーザー作成
```http
POST /api/v1/users
```

**リクエスト**
```json
{
  "email": "newuser@example.com",
  "name": "新規ユーザー",
  "password": "password123",
  "role": "user"
}
```

### ユーザー更新
```http
PUT /api/v1/users/{id}
```

### ユーザー削除
```http
DELETE /api/v1/users/{id}
```

## データAPI（例：投稿）

### 投稿一覧取得
```http
GET /api/v1/posts
```

### 投稿詳細取得
```http
GET /api/v1/posts/{id}
```

### 投稿作成
```http
POST /api/v1/posts
```

**リクエスト**
```json
{
  "title": "投稿タイトル",
  "content": "投稿内容",
  "tags": ["タグ1", "タグ2"],
  "publishedAt": "2024-01-01T00:00:00Z"
}
```

### 投稿更新
```http
PUT /api/v1/posts/{id}
```

### 投稿削除
```http
DELETE /api/v1/posts/{id}
```

## ファイルAPI

### ファイルアップロード
```http
POST /api/v1/files
Content-Type: multipart/form-data
```

**レスポンス**
```json
{
  "success": true,
  "data": {
    "id": "file-uuid",
    "url": "https://cdn.example.com/files/file-uuid.jpg",
    "filename": "example.jpg",
    "size": 1048576,
    "mimeType": "image/jpeg"
  }
}
```

### ファイル削除
```http
DELETE /api/v1/files/{id}
```

## 検索API

### 全文検索
```http
GET /api/v1/search?q=keyword&type=posts&page=1&limit=10
```

**クエリパラメータ**
- `q`: 検索キーワード（必須）
- `type`: 検索対象タイプ（posts, users, etc.）
- `page`: ページ番号
- `limit`: 件数制限

## WebSocket API

### 接続
```javascript
const ws = new WebSocket('wss://api.example.com/ws');
```

### メッセージ形式
```json
{
  "type": "message_type",
  "data": {
    // メッセージデータ
  },
  "timestamp": "2024-01-01T00:00:00Z"
}
```

## レート制限

### 制限内容
- **認証済みユーザー**: 1000リクエスト/時間
- **未認証ユーザー**: 100リクエスト/時間
- **ファイルアップロード**: 50リクエスト/時間

### ヘッダー
- `X-RateLimit-Limit`: 制限値
- `X-RateLimit-Remaining`: 残り回数
- `X-RateLimit-Reset`: リセット時刻（Unix timestamp）

## データ形式

### 日時形式
- **形式**: ISO 8601 (RFC 3339)
- **例**: `2024-01-01T00:00:00Z`
- **タイムゾーン**: UTC

### ID形式
- **形式**: UUID v4 または 連番
- **例**: `550e8400-e29b-41d4-a716-446655440000`

## バリデーションルール

### 共通ルール
- **email**: 有効なメールアドレス形式
- **password**: 8文字以上、英数字記号を含む
- **name**: 1-100文字
- **phone**: 国際電話番号形式

### カスタムルール
```json
{
  "title": {
    "required": true,
    "minLength": 1,
    "maxLength": 200
  },
  "content": {
    "required": true,
    "minLength": 10,
    "maxLength": 10000
  }
}
```

## エラーコード一覧

### 認証エラー
- `AUTH_TOKEN_MISSING`: トークンが存在しない
- `AUTH_TOKEN_INVALID`: 無効なトークン
- `AUTH_TOKEN_EXPIRED`: トークンの有効期限切れ
- `AUTH_INSUFFICIENT_PRIVILEGES`: 権限不足

### バリデーションエラー
- `VALIDATION_ERROR`: バリデーションエラー
- `REQUIRED_FIELD_MISSING`: 必須フィールド不足
- `INVALID_FORMAT`: 形式エラー
- `VALUE_OUT_OF_RANGE`: 値が範囲外

### リソースエラー
- `RESOURCE_NOT_FOUND`: リソースが見つからない
- `RESOURCE_ALREADY_EXISTS`: リソースが既に存在
- `RESOURCE_CONFLICT`: リソースの競合

### システムエラー
- `INTERNAL_SERVER_ERROR`: サーバー内部エラー
- `SERVICE_UNAVAILABLE`: サービス利用不可
- `DATABASE_ERROR`: データベースエラー