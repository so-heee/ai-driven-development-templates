# TypeScript エラーハンドリング

## カスタムエラークラス
```typescript
export class AppError extends Error {
  constructor(
    public message: string,
    public statusCode: number = 500,
    public code?: string,
    public isOperational: boolean = true
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

export class ValidationError extends AppError {
  constructor(message: string, public field?: string) {
    super(message, 400, 'VALIDATION_ERROR');
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string) {
    super(`${resource} not found`, 404, 'NOT_FOUND');
  }
}
```

## Express エラーハンドラー
```typescript
export const errorHandler = (
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  if (err instanceof AppError) {
    return res.status(err.statusCode).json({
      success: false,
      error: {
        code: err.code,
        message: err.message
      }
    });
  }

  // 予期しないエラー
  console.error(err);
  res.status(500).json({
    success: false,
    error: {
      code: 'INTERNAL_ERROR',
      message: 'Internal server error'
    }
  });
};

// 使用例
app.use(errorHandler);
```

## Result型パターン
```typescript
type Result<T, E = Error> = {
  success: true;
  data: T;
} | {
  success: false;
  error: E;
};

class UserService {
  async createUser(userData: CreateUserDto): Promise<Result<User>> {
    try {
      const user = await this.userRepo.create(userData);
      return { success: true, data: user };
    } catch (error) {
      return { 
        success: false, 
        error: new ValidationError('Failed to create user') 
      };
    }
  }
}
```