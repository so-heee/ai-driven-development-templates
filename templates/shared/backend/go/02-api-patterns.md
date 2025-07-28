# Go API実装パターン

## HTTP フレームワーク

### net/http（標準ライブラリ）

```go
package main

import (
    "encoding/json"
    "net/http"
    "github.com/gorilla/mux"
)

type User struct {
    ID   int    `json:"id"`
    Name string `json:"name"`
}

// RESTful ルーティング
func setupRoutes() *mux.Router {
    r := mux.NewRouter()

    // API prefix
    api := r.PathPrefix("/api/v1").Subrouter()

    // Users endpoints
    api.HandleFunc("/users", getUsers).Methods("GET")
    api.HandleFunc("/users", createUser).Methods("POST")
    api.HandleFunc("/users/{id:[0-9]+}", getUser).Methods("GET")
    api.HandleFunc("/users/{id:[0-9]+}", updateUser).Methods("PUT")
    api.HandleFunc("/users/{id:[0-9]+}", deleteUser).Methods("DELETE")

    return r
}

// ハンドラーの実装
func getUsers(w http.ResponseWriter, r *http.Request) {
    users := []User{
        {ID: 1, Name: "John"},
        {ID: 2, Name: "Jane"},
    }

    w.Header().Set("Content-Type", "application/json")
    json.NewEncoder(w).Encode(users)
}

func createUser(w http.ResponseWriter, r *http.Request) {
    var user User
    if err := json.NewDecoder(r.Body).Decode(&user); err != nil {
        http.Error(w, "Invalid JSON", http.StatusBadRequest)
        return
    }

    // バリデーション
    if user.Name == "" {
        http.Error(w, "Name is required", http.StatusBadRequest)
        return
    }

    // 作成処理
    user.ID = generateID()

    w.Header().Set("Content-Type", "application/json")
    w.WriteHeader(http.StatusCreated)
    json.NewEncoder(w).Encode(user)
}
```

### Gin フレームワーク

```go
package main

import (
    "net/http"
    "strconv"

    "github.com/gin-gonic/gin"
)

type User struct {
    ID   int    `json:"id" binding:"required"`
    Name string `json:"name" binding:"required"`
}

func main() {
    r := gin.Default()

    // ミドルウェア
    r.Use(gin.Logger())
    r.Use(gin.Recovery())

    // API group
    api := r.Group("/api/v1")
    {
        users := api.Group("/users")
        {
            users.GET("", getUsers)
            users.POST("", createUser)
            users.GET("/:id", getUser)
            users.PUT("/:id", updateUser)
            users.DELETE("/:id", deleteUser)
        }
    }

    r.Run(":8080")
}

// Gin ハンドラー
func getUsers(c *gin.Context) {
    users := []User{
        {ID: 1, Name: "John"},
        {ID: 2, Name: "Jane"},
    }

    c.JSON(http.StatusOK, gin.H{
        "success": true,
        "data":    users,
    })
}

func createUser(c *gin.Context) {
    var user User
    if err := c.ShouldBindJSON(&user); err != nil {
        c.JSON(http.StatusBadRequest, gin.H{
            "success": false,
            "error":   err.Error(),
        })
        return
    }

    // 作成処理
    user.ID = generateID()

    c.JSON(http.StatusCreated, gin.H{
        "success": true,
        "data":    user,
    })
}

func getUser(c *gin.Context) {
    idStr := c.Param("id")
    id, err := strconv.Atoi(idStr)
    if err != nil {
        c.JSON(http.StatusBadRequest, gin.H{
            "success": false,
            "error":   "Invalid user ID",
        })
        return
    }

    // 検索処理
    user, err := findUserByID(id)
    if err != nil {
        c.JSON(http.StatusNotFound, gin.H{
            "success": false,
            "error":   "User not found",
        })
        return
    }

    c.JSON(http.StatusOK, gin.H{
        "success": true,
        "data":    user,
    })
}
```

### Echo フレームワーク

```go
package main

import (
    "net/http"
    "strconv"

    "github.com/labstack/echo/v4"
    "github.com/labstack/echo/v4/middleware"
)

func main() {
    e := echo.New()

    // ミドルウェア
    e.Use(middleware.Logger())
    e.Use(middleware.Recover())
    e.Use(middleware.CORS())

    // ルーティング
    api := e.Group("/api/v1")
    api.GET("/users", getUsers)
    api.POST("/users", createUser)
    api.GET("/users/:id", getUser)
    api.PUT("/users/:id", updateUser)
    api.DELETE("/users/:id", deleteUser)

    e.Logger.Fatal(e.Start(":8080"))
}

// Echo ハンドラー
func getUsers(c echo.Context) error {
    users := []User{
        {ID: 1, Name: "John"},
        {ID: 2, Name: "Jane"},
    }

    return c.JSON(http.StatusOK, map[string]interface{}{
        "success": true,
        "data":    users,
    })
}

func createUser(c echo.Context) error {
    var user User
    if err := c.Bind(&user); err != nil {
        return c.JSON(http.StatusBadRequest, map[string]interface{}{
            "success": false,
            "error":   "Invalid input",
        })
    }

    // バリデーション
    if err := c.Validate(&user); err != nil {
        return c.JSON(http.StatusBadRequest, map[string]interface{}{
            "success": false,
            "error":   err.Error(),
        })
    }

    user.ID = generateID()

    return c.JSON(http.StatusCreated, map[string]interface{}{
        "success": true,
        "data":    user,
    })
}
```

## ミドルウェアパターン

### 認証ミドルウェア

```go
package middleware

import (
    "net/http"
    "strings"

    "github.com/gin-gonic/gin"
    "github.com/golang-jwt/jwt/v4"
)

func AuthMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        token := extractToken(c.Request)
        if token == "" {
            c.JSON(http.StatusUnauthorized, gin.H{
                "success": false,
                "error":   "Authorization token required",
            })
            c.Abort()
            return
        }

        claims, err := validateToken(token)
        if err != nil {
            c.JSON(http.StatusUnauthorized, gin.H{
                "success": false,
                "error":   "Invalid token",
            })
            c.Abort()
            return
        }

        // ユーザー情報をコンテキストに設定
        c.Set("user_id", claims["user_id"])
        c.Set("user_role", claims["role"])

        c.Next()
    }
}

func extractToken(r *http.Request) string {
    bearerToken := r.Header.Get("Authorization")
    if len(strings.Split(bearerToken, " ")) == 2 {
        return strings.Split(bearerToken, " ")[1]
    }
    return ""
}

func validateToken(tokenString string) (jwt.MapClaims, error) {
    token, err := jwt.Parse(tokenString, func(token *jwt.Token) (interface{}, error) {
        return []byte("secret"), nil
    })

    if err != nil {
        return nil, err
    }

    if claims, ok := token.Claims.(jwt.MapClaims); ok && token.Valid {
        return claims, nil
    }

    return nil, err
}
```

### ログミドルウェア

```go
package middleware

import (
    "log"
    "time"

    "github.com/gin-gonic/gin"
)

func LoggerMiddleware() gin.HandlerFunc {
    return gin.LoggerWithFormatter(func(param gin.LogFormatterParams) string {
        return fmt.Sprintf("%s - [%s] \"%s %s %s %d %s \"%s\" %s\"\n",
            param.ClientIP,
            param.TimeStamp.Format(time.RFC1123),
            param.Method,
            param.Path,
            param.Request.Proto,
            param.StatusCode,
            param.Latency,
            param.Request.UserAgent(),
            param.ErrorMessage,
        )
    })
}

// カスタムログ構造
type LogEntry struct {
    Timestamp  time.Time `json:"timestamp"`
    Method     string    `json:"method"`
    Path       string    `json:"path"`
    StatusCode int       `json:"status_code"`
    Latency    string    `json:"latency"`
    ClientIP   string    `json:"client_ip"`
    UserAgent  string    `json:"user_agent"`
    Error      string    `json:"error,omitempty"`
}

func StructuredLoggerMiddleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        start := time.Now()
        path := c.Request.URL.Path

        c.Next()

        entry := LogEntry{
            Timestamp:  time.Now(),
            Method:     c.Request.Method,
            Path:       path,
            StatusCode: c.Writer.Status(),
            Latency:    time.Since(start).String(),
            ClientIP:   c.ClientIP(),
            UserAgent:  c.Request.UserAgent(),
        }

        if len(c.Errors) > 0 {
            entry.Error = c.Errors.String()
        }

        logJSON, _ := json.Marshal(entry)
        log.Println(string(logJSON))
    }
}
```

### レート制限ミドルウェア

```go
package middleware

import (
    "net/http"
    "sync"
    "time"

    "github.com/gin-gonic/gin"
    "golang.org/x/time/rate"
)

type RateLimiter struct {
    visitors map[string]*rate.Limiter
    mu       sync.RWMutex
    rate     rate.Limit
    burst    int
}

func NewRateLimiter(r rate.Limit, b int) *RateLimiter {
    return &RateLimiter{
        visitors: make(map[string]*rate.Limiter),
        rate:     r,
        burst:    b,
    }
}

func (rl *RateLimiter) getLimiter(ip string) *rate.Limiter {
    rl.mu.RLock()
    limiter, exists := rl.visitors[ip]
    rl.mu.RUnlock()

    if !exists {
        rl.mu.Lock()
        limiter = rate.NewLimiter(rl.rate, rl.burst)
        rl.visitors[ip] = limiter
        rl.mu.Unlock()
    }

    return limiter
}

func (rl *RateLimiter) Middleware() gin.HandlerFunc {
    return func(c *gin.Context) {
        ip := c.ClientIP()
        limiter := rl.getLimiter(ip)

        if !limiter.Allow() {
            c.JSON(http.StatusTooManyRequests, gin.H{
                "success": false,
                "error":   "Rate limit exceeded",
            })
            c.Abort()
            return
        }

        c.Next()
    }
}

// 使用例
func main() {
    r := gin.Default()

    // 1秒間に10リクエスト、バースト20まで許可
    limiter := NewRateLimiter(rate.Every(100*time.Millisecond), 20)
    r.Use(limiter.Middleware())

    r.GET("/api/users", getUsers)
    r.Run(":8080")
}
```

## バリデーション

### struct タグを使用したバリデーション

```go
package main

import (
    "github.com/gin-gonic/gin"
    "github.com/go-playground/validator/v10"
)

type User struct {
    ID    int    `json:"id"`
    Name  string `json:"name" binding:"required,min=2,max=50"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age" binding:"required,min=0,max=120"`
}

type CreateUserRequest struct {
    Name  string `json:"name" binding:"required,min=2,max=50"`
    Email string `json:"email" binding:"required,email"`
    Age   int    `json:"age" binding:"required,min=0,max=120"`
}

// カスタムバリデーション
func customValidation(fl validator.FieldLevel) bool {
    value := fl.Field().String()
    // カスタムロジック
    return len(value) > 0 && value != "admin"
}

func init() {
    // カスタムバリデーション登録
    if v, ok := binding.Validator.Engine().(*validator.Validate); ok {
        v.RegisterValidation("notadmin", customValidation)
    }
}

func createUser(c *gin.Context) {
    var req CreateUserRequest
    if err := c.ShouldBindJSON(&req); err != nil {
        // バリデーションエラーの詳細化
        var validationErrors []string
        if errs, ok := err.(validator.ValidationErrors); ok {
            for _, e := range errs {
                validationErrors = append(validationErrors,
                    fmt.Sprintf("Field '%s' failed validation '%s'",
                        e.Field(), e.Tag()))
            }
        }

        c.JSON(http.StatusBadRequest, gin.H{
            "success": false,
            "error":   "Validation failed",
            "details": validationErrors,
        })
        return
    }

    // 作成処理
    user := User{
        ID:    generateID(),
        Name:  req.Name,
        Email: req.Email,
        Age:   req.Age,
    }

    c.JSON(http.StatusCreated, gin.H{
        "success": true,
        "data":    user,
    })
}
```

## GraphQL 実装

### gqlgen を使用した GraphQL サーバー

```go
// schema.graphql
type User {
  id: ID!
  name: String!
  email: String!
  posts: [Post!]!
}

type Post {
  id: ID!
  title: String!
  content: String!
  author: User!
}

type Query {
  users: [User!]!
  user(id: ID!): User
  posts: [Post!]!
}

type Mutation {
  createUser(input: CreateUserInput!): User!
  updateUser(id: ID!, input: UpdateUserInput!): User!
  deleteUser(id: ID!): Boolean!
}

input CreateUserInput {
  name: String!
  email: String!
}

input UpdateUserInput {
  name: String
  email: String
}
```

```go
// resolver.go
package main

import (
    "context"
    "fmt"

    "github.com/99designs/gqlgen/graphql/handler"
    "github.com/gin-gonic/gin"
)

type Resolver struct {
    users []User
    posts []Post
}

// Query リゾルバー
func (r *queryResolver) Users(ctx context.Context) ([]*User, error) {
    return r.users, nil
}

func (r *queryResolver) User(ctx context.Context, id string) (*User, error) {
    for _, user := range r.users {
        if user.ID == id {
            return &user, nil
        }
    }
    return nil, fmt.Errorf("user not found")
}

// Mutation リゾルバー
func (r *mutationResolver) CreateUser(ctx context.Context, input CreateUserInput) (*User, error) {
    user := User{
        ID:    generateID(),
        Name:  input.Name,
        Email: input.Email,
    }

    r.users = append(r.users, user)
    return &user, nil
}

// フィールドリゾルバー（N+1問題対策）
func (r *userResolver) Posts(ctx context.Context, obj *User) ([]*Post, error) {
    var userPosts []*Post
    for _, post := range r.posts {
        if post.AuthorID == obj.ID {
            userPosts = append(userPosts, &post)
        }
    }
    return userPosts, nil
}

// サーバー設定
func main() {
    r := gin.Default()

    resolver := &Resolver{
        users: []User{},
        posts: []Post{},
    }

    srv := handler.NewDefaultServer(generated.NewExecutableSchema(generated.Config{
        Resolvers: resolver,
    }))

    r.POST("/graphql", func(c *gin.Context) {
        srv.ServeHTTP(c.Writer, c.Request)
    })

    r.GET("/playground", func(c *gin.Context) {
        playground.Handler("GraphQL Playground", "/graphql").ServeHTTP(c.Writer, c.Request)
    })

    r.Run(":8080")
}
```

## コンテキスト管理

### リクエストコンテキストの活用

```go
package main

import (
    "context"
    "time"

    "github.com/gin-gonic/gin"
)

// コンテキストキー
type contextKey string

const (
    UserIDKey    contextKey = "user_id"
    RequestIDKey contextKey = "request_id"
    TraceIDKey   contextKey = "trace_id"
)

// コンテキスト設定
func setRequestContext() gin.HandlerFunc {
    return func(c *gin.Context) {
        ctx := c.Request.Context()

        // リクエストID生成
        requestID := generateRequestID()
        ctx = context.WithValue(ctx, RequestIDKey, requestID)

        // タイムアウト設定
        ctx, cancel := context.WithTimeout(ctx, 30*time.Second)
        defer cancel()

        // 更新されたコンテキストを設定
        c.Request = c.Request.WithContext(ctx)
        c.Next()
    }
}

// サービス層でのコンテキスト使用
func getUserService(ctx context.Context, id string) (*User, error) {
    // タイムアウトチェック
    select {
    case <-ctx.Done():
        return nil, ctx.Err()
    default:
    }

    // リクエストIDの取得
    requestID := ctx.Value(RequestIDKey).(string)
    log.Printf("Processing request %s for user %s", requestID, id)

    // データベース操作（コンテキスト渡し）
    return db.GetUser(ctx, id)
}
```

---

**関連ドキュメント:**

- [データベースパターン](./02-database-patterns.md)
- [認証実装](./03-authentication.md)
- [エラーハンドリング](./04-error-handling.md)
- [ログ実装](./05-logging.md)
