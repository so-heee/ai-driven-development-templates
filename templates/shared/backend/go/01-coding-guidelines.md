# Go コーディングガイドライン

## 公式リソース

このガイドラインは以下の公式ドキュメントに基づいています：

- [Effective Go](https://go.dev/doc/effective_go) - Go公式の基本ガイドライン
- [Go Code Review Comments](https://go.dev/wiki/CodeReviewComments) - 公式レビューコメント集
- [Google Go Style Guide](https://google.github.io/styleguide/go/) - 業界標準スタイルガイド

## フォーマット

### 基本フォーマット

- **`gofmt`を使用** - 全てのGoコードは`gofmt`で自動フォーマット
- **タブを使用** - インデントはタブ、スペースは避ける
- **行の長さ制限なし** - 必要に応じて改行し、追加タブでインデント

```go
// 良い例
func longFunctionName(parameterOne string,
    parameterTwo int, parameterThree bool) error {
    // 実装
}
```

## 命名規則

### パッケージ名

- **小文字、短縮形** - `io`、`fmt`、`http`
- **単数形** - `user`（`users`ではない）
- **意味のある名前** - パッケージの機能を表現

```go
// 良い例
package user
package httputil

// 悪い例
package users
package http_util
```

### 関数・変数名

- **CamelCase** - エクスポートする場合は大文字開始
- **短縮名を適切に使用** - スコープが狭い場合は短縮可

```go
// 良い例
func GetUserByID(id int) *User    // エクスポート
func getUserByID(id int) *User    // 非エクスポート

// ループ内での短縮名
for i, user := range users {
    // i, userは適切
}

// 長いスコープでは明確な名前
func ProcessUserData(userData *UserData) error {
    // userDataは明確
}
```

### 定数

- **CamelCase** - エクスポートする場合は大文字開始
- **グループ化** - 関連する定数は`const`ブロックで

```go
// 良い例
const (
    StatusActive   = "active"
    StatusInactive = "inactive"
    StatusPending  = "pending"
)

const DefaultTimeout = 30 * time.Second
```

## 型定義

### インターフェース

- **"-er"で終わる** - 単一メソッドインターフェース
- **小さく保つ** - 必要最小限のメソッド数

```go
// 良い例
type Reader interface {
    Read([]byte) (int, error)
}

type UserRepository interface {
    GetUser(id int) (*User, error)
    SaveUser(*User) error
}
```

### 構造体

- **フィールドは論理的順序** - 関連フィールドをグループ化
- **埋め込み** - 継承の代わりに構造体埋め込みを使用

```go
// 良い例
type User struct {
    // 識別情報
    ID       int    `json:"id"`
    Username string `json:"username"`

    // プロファイル情報
    Email     string    `json:"email"`
    CreatedAt time.Time `json:"created_at"`

    // 埋め込み構造体
    Profile UserProfile `json:"profile"`
}
```

## エラーハンドリング

### エラーの扱い

- **エラーは値** - エラーを戻り値として返す
- **すぐに処理** - エラーを受け取ったらすぐに処理
- **コンテキストを追加** - `fmt.Errorf`でラップ

```go
// 良い例
func GetUser(id int) (*User, error) {
    user, err := repository.FindUser(id)
    if err != nil {
        return nil, fmt.Errorf("failed to get user %d: %w", id, err)
    }

    if user == nil {
        return nil, fmt.Errorf("user %d not found", id)
    }

    return user, nil
}

// 呼び出し側
user, err := GetUser(123)
if err != nil {
    log.Printf("error: %v", err)
    return err
}
```

### カスタムエラー

- **エラー型を定義** - 特定のエラー処理が必要な場合

```go
// 良い例
type ValidationError struct {
    Field string
    Value interface{}
    Msg   string
}

func (e *ValidationError) Error() string {
    return fmt.Sprintf("validation failed for field %s: %s", e.Field, e.Msg)
}

// 使用例
if user.Email == "" {
    return &ValidationError{
        Field: "email",
        Value: user.Email,
        Msg:   "email is required",
    }
}
```

## 並行性

### Goroutineの使用

- **チャネルで通信** - 共有メモリは避ける
- **適切な同期** - `sync`パッケージを活用

```go
// 良い例
func ProcessUsers(users []User) <-chan Result {
    results := make(chan Result, len(users))

    go func() {
        defer close(results)
        for _, user := range users {
            result := processUser(user)
            results <- result
        }
    }()

    return results
}

// WaitGroupの使用
func ProcessConcurrently(items []Item) {
    var wg sync.WaitGroup

    for _, item := range items {
        wg.Add(1)
        go func(item Item) {
            defer wg.Done()
            processItem(item)
        }(item)
    }

    wg.Wait()
}
```

## ベストプラクティス

### コメント

- **パッケージコメント** - package文の直前に記述
- **エクスポート要素** - 公開関数・型にはコメント必須
- **日本語可** - チーム内で統一

```go
// Package user provides user management functionality.
// このパッケージはユーザー管理機能を提供します。
package user

// User represents a system user.
// Userはシステムのユーザーを表現します。
type User struct {
    ID   int
    Name string
}

// GetUser retrieves a user by ID.
// GetUserはIDによってユーザーを取得します。
func GetUser(id int) (*User, error) {
    // 実装
}
```

### 初期化

- **ゼロ値を活用** - 明示的な初期化は最小限に
- **コンストラクタ関数** - 複雑な初期化が必要な場合

```go
// 良い例：ゼロ値が有効
type Counter struct {
    value int
}

// カウンターは初期化なしで使用可能
var c Counter
c.Increment()

// 複雑な初期化が必要な場合
func NewDatabase(config Config) (*Database, error) {
    db := &Database{
        config: config,
        pool:   make(chan *Connection, config.MaxConnections),
    }

    if err := db.connect(); err != nil {
        return nil, fmt.Errorf("failed to connect: %w", err)
    }

    return db, nil
}
```

### テスト

- **テーブルドリブンテスト** - 複数のケースを効率的に

```go
// 良い例
func TestValidateEmail(t *testing.T) {
    tests := []struct {
        name    string
        email   string
        wantErr bool
    }{
        {"valid email", "user@example.com", false},
        {"empty email", "", true},
        {"invalid format", "invalid-email", true},
    }

    for _, tt := range tests {
        t.Run(tt.name, func(t *testing.T) {
            err := ValidateEmail(tt.email)
            if (err != nil) != tt.wantErr {
                t.Errorf("ValidateEmail() error = %v, wantErr %v", err, tt.wantErr)
            }
        })
    }
}
```

## 避けるべきパターン

### アンチパターン

```go
// 悪い例：エラー無視
user, _ := GetUser(id)

// 悪い例：panic使用（回復不可能な場合以外）
if err != nil {
    panic(err)
}

// 悪い例：不要な型変換
var i interface{} = 42
num := i.(int) // 直接int型を使用すべき

// 悪い例：長い関数
func ProcessEverything() {
    // 200行の処理...
    // 小さな関数に分割すべき
}
```

### パフォーマンス

```go
// 悪い例：文字列連結
result := ""
for _, item := range items {
    result += item.String()
}

// 良い例：strings.Builderを使用
var builder strings.Builder
for _, item := range items {
    builder.WriteString(item.String())
}
result := builder.String()
```

## ツール活用

### 推奨ツール

- **`gofmt`** - コードフォーマット
- **`goimports`** - import文の整理
- **`go vet`** - 静的解析
- **`golint`** - コーディング規約チェック
- **`go mod tidy`** - 依存関係の整理

### エディタ設定例

```json
// VS Code settings.json
{
  "go.formatTool": "goimports",
  "go.lintOnSave": "package",
  "go.vetOnSave": "package",
  "[go]": {
    "editor.formatOnSave": true,
    "editor.codeActionsOnSave": {
      "source.organizeImports": true
    }
  }
}
```

---

このガイドラインに従うことで、読みやすく保守性の高いGoコードが書けます。詳細は各公式ドキュメントを参照してください。
