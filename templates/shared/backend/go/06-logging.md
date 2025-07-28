# Go ログ実装

## 構造化ログ（logrus）

```go
import "github.com/sirupsen/logrus"

func init() {
    logrus.SetFormatter(&logrus.JSONFormatter{})
    logrus.SetLevel(logrus.InfoLevel)
}

func logWithFields() {
    logger := logrus.WithFields(logrus.Fields{
        "user_id":    123,
        "request_id": "req-456",
        "action":     "create_user",
    })

    logger.Info("User created successfully")
    logger.Error("Failed to create user")
}
```

## Zap（高性能ログ）

```go
import "go.uber.org/zap"

func setupZap() *zap.Logger {
    config := zap.NewProductionConfig()
    config.OutputPaths = []string{"stdout", "/var/log/app.log"}

    logger, _ := config.Build()
    defer logger.Sync()

    return logger
}

func logWithZap(logger *zap.Logger) {
    logger.Info("User operation",
        zap.Int("user_id", 123),
        zap.String("action", "create"),
        zap.Duration("latency", time.Millisecond*100),
    )
}
```

## コンテキストログ

```go
type Logger struct {
    *zap.Logger
    ctx context.Context
}

func (l *Logger) WithContext(ctx context.Context) *Logger {
    return &Logger{
        Logger: l.Logger,
        ctx:    ctx,
    }
}

func (l *Logger) Info(msg string, fields ...zap.Field) {
    if requestID := l.ctx.Value("request_id"); requestID != nil {
        fields = append(fields, zap.String("request_id", requestID.(string)))
    }
    l.Logger.Info(msg, fields...)
}
```
