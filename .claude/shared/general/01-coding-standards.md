# コーディング規約ガイド

## 一般原則

### Clean Code の原則
```typescript
// 悪い例：意味のない変数名
const d = new Date();
const u = users.filter(u => u.a > 18);

// 良い例：意図が明確な変数名
const currentDate = new Date();
const adultUsers = users.filter(user => user.age > 18);
```

### SOLID原則の適用
```typescript
// Single Responsibility Principle（単一責任の原則）
// 悪い例：複数の責任を持つクラス
class User {
  constructor(public name: string, public email: string) {}
  
  save() { /* データベースに保存 */ }
  sendEmail() { /* メール送信 */ }
  validateEmail() { /* メール検証 */ }
}

// 良い例：責任を分離
class User {
  constructor(public name: string, public email: string) {}
}

class UserRepository {
  save(user: User) { /* データベースに保存 */ }
}

class EmailService {
  send(to: string, message: string) { /* メール送信 */ }
}

class EmailValidator {
  isValid(email: string): boolean { /* メール検証 */ }
}
```

## 命名規則

### 変数・関数名
```typescript
// 変数名：camelCase
const userName = 'john_doe';
const isLoggedIn = true;
const userPreferences = {};

// 定数：SCREAMING_SNAKE_CASE
const MAX_RETRY_COUNT = 3;
const API_BASE_URL = 'https://api.example.com';

// 関数名：動詞 + 名詞の形式
function getUserById(id: string): User | null { }
function calculateTotalPrice(items: Item[]): number { }
function validateEmailFormat(email: string): boolean { }

// boolean値を返す関数：is/has/can で始める
function isValidEmail(email: string): boolean { }
function hasPermission(user: User, resource: string): boolean { }
function canEdit(user: User, document: Document): boolean { }
```

### クラス・インターface名
```typescript
// クラス名：PascalCase
class UserService { }
class PaymentProcessor { }
class DatabaseConnection { }

// Interface名：PascalCase（I プレフィックスは使わない）
interface User {
  id: string;
  name: string;
  email: string;
}

interface PaymentMethod {
  type: 'credit' | 'debit' | 'paypal';
  token: string;
}

// Type alias：PascalCase
type UserRole = 'admin' | 'user' | 'guest';
type APIResponse<T> = {
  data: T;
  status: number;
  message?: string;
};
```

## フォーマット規則

### インデントとスペース
```typescript
// 2スペースインデント
if (condition) {
  doSomething();
  if (nestedCondition) {
    doNestedThing();
  }
}

// オブジェクトのプロパティ
const config = {
  apiUrl: 'https://api.example.com',
  timeout: 5000,
  retries: 3,
};

// 配列の要素
const items = [
  'first',
  'second',
  'third',
];
```

### 行の長さと改行
```typescript
// 最大行長：100文字
// 長い関数呼び出しは適切に改行
const result = someVeryLongFunctionName(
  firstArgument,
  secondArgument,
  thirdArgument
);

// チェーンメソッドの改行
const processedData = rawData
  .filter(item => item.isActive)
  .map(item => ({
    id: item.id,
    name: item.name.trim(),
    createdAt: new Date(item.createdAt),
  }))
  .sort((a, b) => a.name.localeCompare(b.name));
```

## コメント規則

### JSDocコメント
```typescript
/**
 * ユーザー情報を取得する
 * @param userId - ユーザーID
 * @param includePreferences - 設定情報を含めるかどうか
 * @returns Promise<User | null> ユーザー情報、見つからない場合はnull
 * @throws {ValidationError} userIdが無効な場合
 * @example
 * ```typescript
 * const user = await getUserById('123', true);
 * if (user) {
 *   console.log(user.name);
 * }
 * ```
 */
async function getUserById(
  userId: string,
  includePreferences = false
): Promise<User | null> {
  // 実装
}
```

### インラインコメント
```typescript
// 良いコメント：なぜその処理が必要かを説明
// ブラウザのバグ回避のため、遅延を追加
await new Promise(resolve => setTimeout(resolve, 100));

// 複雑なビジネスロジックの説明
// 税率計算：消費税10% + 地方税2%（東京都の場合）
const taxRate = 0.12;

// 悪いコメント：コードを単に説明しているだけ
// iを1増やす
i++;

// 名前をコンソールに出力
console.log(user.name);
```

## エラーハンドリング

### エラークラスの定義
```typescript
// カスタムエラークラス
class ValidationError extends Error {
  constructor(
    message: string,
    public field: string,
    public value: any
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

class NotFoundError extends Error {
  constructor(resource: string, id: string) {
    super(`${resource} with id ${id} not found`);
    this.name = 'NotFoundError';
  }
}

// 使用例
if (!isValidEmail(email)) {
  throw new ValidationError('Invalid email format', 'email', email);
}
```

### try-catch の使用
```typescript
// 悪い例：エラーを隠蔽
try {
  await riskyOperation();
} catch (error) {
  // エラーを無視
}

// 良い例：適切なエラーハンドリング
try {
  await riskyOperation();
} catch (error) {
  logger.error('Failed to perform risky operation', {
    error: error.message,
    stack: error.stack,
    context: { userId, operationType: 'risky' },
  });
  
  // エラーを適切に処理または再スロー
  throw new Error('Operation failed. Please try again later.');
}
```

## 型定義規則

### TypeScript型の活用
```typescript
// Union Types
type Status = 'pending' | 'approved' | 'rejected';

// Intersection Types
type UserWithPreferences = User & {
  preferences: UserPreferences;
};

// Generic Types
interface Repository<T> {
  findById(id: string): Promise<T | null>;
  save(entity: T): Promise<T>;
  delete(id: string): Promise<void>;
}

// Utility Types
interface UpdateUserRequest extends Partial<Pick<User, 'name' | 'email'>> {
  id: string;
}

// Template Literal Types
type HTTPMethod = 'GET' | 'POST' | 'PUT' | 'DELETE';
type APIEndpoint<T extends string> = `/api/v1/${T}`;
```

### 型安全なAPI設計
```typescript
// 型安全なAPIクライアント
interface APIClient {
  get<T>(endpoint: string): Promise<APIResponse<T>>;
  post<TRequest, TResponse>(
    endpoint: string,
    data: TRequest
  ): Promise<APIResponse<TResponse>>;
}

// 型安全な設定オブジェクト
interface DatabaseConfig {
  readonly host: string;
  readonly port: number;
  readonly database: string;
  readonly ssl: boolean;
}

// 型ガード
function isUser(obj: any): obj is User {
  return obj && 
         typeof obj.id === 'string' &&
         typeof obj.name === 'string' &&
         typeof obj.email === 'string';
}
```

## テスト規則

### テスト命名規則
```typescript
// テスト関数名：should_期待される結果_when_条件
describe('UserService', () => {
  describe('getUserById', () => {
    it('should_return_user_when_valid_id_provided', async () => {
      // Arrange
      const userId = '123';
      const expectedUser = { id: userId, name: 'John', email: 'john@example.com' };
      mockRepository.findById.mockResolvedValue(expectedUser);
      
      // Act
      const result = await userService.getUserById(userId);
      
      // Assert
      expect(result).toEqual(expectedUser);
    });
    
    it('should_return_null_when_user_not_found', async () => {
      // Arrange
      const userId = 'nonexistent';
      mockRepository.findById.mockResolvedValue(null);
      
      // Act
      const result = await userService.getUserById(userId);
      
      // Assert
      expect(result).toBeNull();
    });
  });
});
```

### テスト構造
```typescript
// AAA パターン（Arrange, Act, Assert）
it('should calculate total price correctly', () => {
  // Arrange - テストデータの準備
  const items = [
    { price: 100, quantity: 2 },
    { price: 200, quantity: 1 },
  ];
  const expectedTotal = 400;
  
  // Act - テスト対象の実行
  const actualTotal = calculateTotal(items);
  
  // Assert - 結果の検証
  expect(actualTotal).toBe(expectedTotal);
});
```

## パフォーマンス規則

### メモリ効率的なコード
```typescript
// 悪い例：不必要なオブジェクト生成
const processItems = (items: Item[]) => {
  return items.map(item => {
    return {
      ...item,
      processed: true,
      timestamp: new Date(),
    };
  });
};

// 良い例：オブジェクトの再利用
const processItems = (items: Item[]) => {
  const timestamp = new Date();
  return items.map(item => ({
    ...item,
    processed: true,
    timestamp,
  }));
};
```

### 非同期処理の最適化
```typescript
// 悪い例：順次実行
const getUsers = async (userIds: string[]) => {
  const users = [];
  for (const id of userIds) {
    const user = await getUserById(id);
    users.push(user);
  }
  return users;
};

// 良い例：並列実行
const getUsers = async (userIds: string[]) => {
  const userPromises = userIds.map(id => getUserById(id));
  return Promise.all(userPromises);
};
```

## セキュリティ規則

### 入力検証
```typescript
// 常に入力値を検証
const createUser = (userData: CreateUserRequest) => {
  // バリデーション
  if (!userData.email || !isValidEmail(userData.email)) {
    throw new ValidationError('Invalid email', 'email', userData.email);
  }
  
  if (!userData.password || userData.password.length < 8) {
    throw new ValidationError('Password must be at least 8 characters', 'password', '');
  }
  
  // サニタイゼーション
  const sanitizedData = {
    email: userData.email.toLowerCase().trim(),
    name: userData.name.trim(),
    password: userData.password, // ハッシュ化は別処理で
  };
  
  return userRepository.create(sanitizedData);
};
```

### 機密情報の取り扱い
```typescript
// 悪い例：パスワードをログに出力
logger.info('User created', { user });

// 良い例：機密情報を除外
logger.info('User created', { 
  userId: user.id, 
  email: user.email,
  // パスワードは含めない
});

// 機密情報をマスク
const maskEmail = (email: string) => {
  const [username, domain] = email.split('@');
  return `${username.slice(0, 2)}***@${domain}`;
};
```

## コードレビュー規則

### レビューポイント
1. **機能性**: コードが要件を満たしているか
2. **可読性**: コードが理解しやすいか
3. **保守性**: 将来の変更に対応しやすいか
4. **パフォーマンス**: 効率的な実装になっているか
5. **セキュリティ**: セキュリティ上の問題がないか
6. **テスト**: 適切にテストされているか

### レビューコメントの例
```typescript
// 🔍 レビュー指摘例

// ❌ 問題のあるコメント
// これはダメです。

// ✅ 建設的なコメント
// この関数が大きくなっているので、複数の小さな関数に分割することを検討してください。
// 具体的には、バリデーション処理とビジネスロジックを分離できそうです。

// ✅ 提案を含むコメント
// パフォーマンスを向上させるため、ここでメモ化を使用することを検討してください：
// const memoizedCalculation = useMemo(() => expensiveCalculation(data), [data]);
```