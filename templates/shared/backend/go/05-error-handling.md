# Go エラーハンドリング

## カスタムエラー
```go
type AppError struct {
    Code    string `json:"code"`
    Message string `json:"message"`
    Details string `json:"details,omitempty"`
}

func (e *AppError) Error() string {
    return e.Message
}

// エラー定義
var (
    ErrUserNotFound = &AppError{
        Code:    "USER_NOT_FOUND",
        Message: "User not found",
    }
    ErrInvalidInput = &AppError{
        Code:    "INVALID_INPUT",
        Message: "Invalid input data",
    }
)
```

## エラーラッピング
```go
import "fmt"

func getUserFromDB(id int) (*User, error) {
    user, err := db.GetUser(id)
    if err != nil {
        return nil, fmt.Errorf("failed to get user %d: %w", id, err)
    }
    return user, nil
}

// エラーチェック
func handleError(err error) {
    if errors.Is(err, ErrUserNotFound) {
        // 特定エラー処理
    }
    
    var appErr *AppError
    if errors.As(err, &appErr) {
        // カスタムエラー処理
    }
}
```

## Ginでのエラーハンドリング
```go
func errorHandler() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Next()
        
        if len(c.Errors) > 0 {
            err := c.Errors.Last()
            
            if appErr, ok := err.Err.(*AppError); ok {
                c.JSON(http.StatusBadRequest, gin.H{
                    "success": false,
                    "error":   appErr,
                })
                return
            }
            
            c.JSON(http.StatusInternalServerError, gin.H{
                "success": false,
                "error":   "Internal server error",
            })
        }
    }
}
```