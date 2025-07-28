# TypeScript ログ実装

## Winston

```typescript
import winston from 'winston'

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
    new winston.transports.Console({
      format: winston.format.simple(),
    }),
  ],
})

// 使用例
logger.info('User created', { userId: 123, email: 'user@example.com' })
logger.error('Database error', { error: error.message, stack: error.stack })
```

## Pino（高性能）

```typescript
import pino from 'pino'

const logger = pino({
  level: 'info',
  transport: {
    target: 'pino-pretty',
    options: {
      colorize: true,
    },
  },
})

// Express middleware
export const requestLogger = (req: Request, res: Response, next: NextFunction) => {
  req.log = logger.child({ requestId: generateId() })

  req.log.info(
    {
      method: req.method,
      url: req.url,
      userAgent: req.get('User-Agent'),
    },
    'Request started'
  )

  next()
}
```

## 構造化ログ

```typescript
interface LogContext {
  requestId?: string
  userId?: number
  action?: string
  duration?: number
}

class Logger {
  private winston: winston.Logger

  constructor() {
    this.winston = winston.createLogger({
      format: winston.format.combine(winston.format.timestamp(), winston.format.json()),
      transports: [new winston.transports.Console(), new winston.transports.File({ filename: 'app.log' })],
    })
  }

  info(message: string, context?: LogContext) {
    this.winston.info(message, context)
  }

  error(message: string, error?: Error, context?: LogContext) {
    this.winston.error(message, {
      ...context,
      error: error?.message,
      stack: error?.stack,
    })
  }
}
```
