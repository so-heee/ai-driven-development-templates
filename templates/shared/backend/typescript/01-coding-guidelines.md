# TypeScript コーディングガイドライン

## 公式リソース

このガイドラインは以下の権威のあるリソースに基づいています：
- [Microsoft TypeScript Coding Guidelines](https://github.com/Microsoft/TypeScript/wiki/Coding-guidelines) - TypeScript開発チームの内部ガイドライン
- [TypeScript Handbook](https://www.typescriptlang.org/docs/) - 公式ドキュメント
- [ESLint TypeScript Rules](https://typescript-eslint.io/rules/) - TypeScript用静的解析ルール
- [Prettier](https://prettier.io/) - 業界標準コードフォーマッター

## フォーマット

### 基本フォーマット
- **Prettierを使用** - 自動フォーマット推奨
- **インデント**: 2スペース（4スペースも可、プロジェクトで統一）
- **セミコロン**: 必須
- **文字列**: ダブルクォート推奨

```typescript
// 良い例
interface User {
  id: number;
  name: string;
  email: string;
}

const getUser = (id: number): User | null => {
  // 実装
  return null;
};
```

### import/export
- **名前付きimport推奨** - デフォルトimportは最小限
- **絶対パス推奨** - 可能な場合は相対パスより絶対パス

```typescript
// 良い例
import { User, UserRepository } from "@/types/user";
import { logger } from "@/utils/logger";
import { validateEmail } from "@/utils/validation";

// 悪い例
import User from "../../../types/user";
import * as utils from "../utils";
```

## 命名規則

### 基本命名
- **変数・関数**: camelCase
- **型・インターフェース・クラス**: PascalCase
- **定数**: UPPER_SNAKE_CASE
- **ファイル名**: kebab-case

```typescript
// 良い例
const userName = "john";
const MAX_RETRY_COUNT = 3;

interface UserProfile {
  id: number;
  displayName: string;
}

class UserService {
  private readonly userRepository: UserRepository;
}

type ApiResponse<T> = {
  data: T;
  status: number;
};
```

### インターフェース
- **「I」プレフィックスは使用しない** - 単にPascalCaseで命名

```typescript
// 良い例
interface User {
  id: number;
  name: string;
}

interface UserRepository {
  findById(id: number): Promise<User | null>;
  save(user: User): Promise<void>;
}

// 悪い例
interface IUser {
  id: number;
  name: string;
}
```

## 型定義

### 基本型の使用
- **プリミティブ型**: `string`、`number`、`boolean`（`String`、`Number`、`Boolean`は避ける）
- **配列**: `T[]`または`Array<T>`（一貫性を保つ）
- **undefined**: `null`より`undefined`を優先

```typescript
// 良い例
const users: User[] = [];
const activeUser: User | undefined = getActiveUser();

function processUser(user: User): void {
  // 実装
}

// 悪い例
const users: Array<User> = []; // T[]と混在させない
const activeUser: User | null = getActiveUser(); // undefinedを優先
function processUser(user: User): Void { // voidは小文字
```

### ユニオン型・交差型
- **ユニオン型**: 複数の型のいずれか
- **交差型**: 複数の型を組み合わせ

```typescript
// 良い例
type Status = "pending" | "approved" | "rejected";

type UserWithProfile = User & {
  profile: UserProfile;
};

// 判別可能なユニオン型
type ApiResult<T> = 
  | { success: true; data: T }
  | { success: false; error: string };
```

### ジェネリクス
- **型パラメータ**: 意味のある名前を使用
- **制約**: `extends`で型を制限

```typescript
// 良い例
interface Repository<Entity> {
  findById(id: number): Promise<Entity | undefined>;
  save(entity: Entity): Promise<void>;
}

type ApiEndpoint<Request, Response> = {
  method: "GET" | "POST" | "PUT" | "DELETE";
  path: string;
  handler: (req: Request) => Promise<Response>;
};

// 制約付きジェネリクス
function serialize<T extends Record<string, unknown>>(
  obj: T
): string {
  return JSON.stringify(obj);
}
```

## エラーハンドリング

### 例外処理
- **カスタムエラークラス**: 特定のエラータイプを定義
- **Result型パターン**: 関数型プログラミングアプローチ

```typescript
// カスタムエラークラス
class ValidationError extends Error {
  constructor(
    public field: string,
    public value: unknown,
    message: string
  ) {
    super(message);
    this.name = "ValidationError";
  }
}

// Result型パターン
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

async function getUser(id: number): Promise<Result<User, ValidationError>> {
  if (id <= 0) {
    return {
      success: false,
      error: new ValidationError("id", id, "ID must be positive"),
    };
  }

  try {
    const user = await userRepository.findById(id);
    return { success: true, data: user };
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error : new Error("Unknown error"),
    };
  }
}
```

### 型ガード
- **カスタム型ガード**: 型の絞り込み
- **`is`キーワード**: 型述語の使用

```typescript
// 型ガード関数
function isUser(obj: unknown): obj is User {
  return (
    typeof obj === "object" &&
    obj !== null &&
    "id" in obj &&
    "name" in obj &&
    typeof (obj as User).id === "number" &&
    typeof (obj as User).name === "string"
  );
}

// 使用例
function processUserData(data: unknown): void {
  if (isUser(data)) {
    // ここでdataはUser型として扱われる
    console.log(`Processing user: ${data.name}`);
  }
}
```

## 非同期処理

### Promise・async/await
- **async/await優先** - Promiseチェーンより読みやすい
- **エラーハンドリング**: try/catchで適切に処理

```typescript
// 良い例
async function createUser(userData: CreateUserRequest): Promise<User> {
  try {
    // バリデーション
    const validatedData = await validateUserData(userData);
    
    // ユーザー作成
    const user = await userRepository.create(validatedData);
    
    // 後処理
    await notificationService.sendWelcomeEmail(user.email);
    
    return user;
  } catch (error) {
    logger.error("Failed to create user", { userData, error });
    throw error;
  }
}

// 並列処理
async function getUserWithProfile(id: number): Promise<UserWithProfile> {
  const [user, profile] = await Promise.all([
    userRepository.findById(id),
    profileRepository.findByUserId(id),
  ]);

  if (!user) {
    throw new Error(`User not found: ${id}`);
  }

  return { ...user, profile };
}
```

## 関数型プログラミング

### 純粋関数・イミュータブル
- **副作用を最小限に** - 純粋関数を優先
- **イミュータブルなデータ** - オブジェクトの変更は避ける

```typescript
// 良い例：純粋関数
function calculateTax(price: number, taxRate: number): number {
  return price * (1 + taxRate);
}

function updateUser(user: User, updates: Partial<User>): User {
  return { ...user, ...updates };
}

// 配列の操作
function getActiveUsers(users: User[]): User[] {
  return users.filter(user => user.status === "active");
}

function getUserNames(users: User[]): string[] {
  return users.map(user => user.name);
}
```

### 高階関数・ユーティリティ型
```typescript
// 高階関数
function withLogging<T extends (...args: any[]) => any>(
  fn: T
): T {
  return ((...args: Parameters<T>) => {
    console.log(`Calling ${fn.name} with args:`, args);
    const result = fn(...args);
    console.log(`Result:`, result);
    return result;
  }) as T;
}

// ユーティリティ型の活用
type CreateUserRequest = Pick<User, "name" | "email">;
type UpdateUserRequest = Partial<Pick<User, "name" | "email">>;
type UserResponse = Omit<User, "password">;
```

## クラス設計

### クラスの使用
- **データ構造**: インターフェースまたは型エイリアス優先
- **動作を持つオブジェクト**: クラスを使用

```typescript
// 良い例：サービスクラス
class UserService {
  constructor(
    private readonly userRepository: UserRepository,
    private readonly logger: Logger
  ) {}

  async createUser(request: CreateUserRequest): Promise<User> {
    this.logger.info("Creating user", { request });
    
    const existingUser = await this.userRepository.findByEmail(request.email);
    if (existingUser) {
      throw new ValidationError("email", request.email, "Email already exists");
    }

    return this.userRepository.create(request);
  }

  async updateUser(id: number, updates: UpdateUserRequest): Promise<User> {
    const user = await this.userRepository.findById(id);
    if (!user) {
      throw new Error(`User not found: ${id}`);
    }

    const updatedUser = { ...user, ...updates };
    return this.userRepository.update(updatedUser);
  }
}
```

## ベストプラクティス

### 型注釈
- **明示的な型注釈**: 型推論が不明確な場合のみ
- **戻り値の型**: 公開関数では明示

```typescript
// 良い例：型推論が効く場合は省略
const users = await getUserList(); // User[]と推論される
const userName = user.name; // stringと推論される

// 明示的な型注釈が必要な場合
const config: DatabaseConfig = JSON.parse(configString);
const handlers: Record<string, RequestHandler> = {};

// 公開関数の戻り値は明示
export async function createUser(request: CreateUserRequest): Promise<User> {
  // 実装
}
```

### コメント・ドキュメント
- **JSDoc**: 公開API用
- **TypeDoc**: ドキュメント生成

```typescript
/**
 * ユーザーを作成します
 * @param request - ユーザー作成リクエスト
 * @returns 作成されたユーザー
 * @throws {ValidationError} バリデーションエラー時
 * @example
 * ```typescript
 * const user = await createUser({
 *   name: "John Doe",
 *   email: "john@example.com"
 * });
 * ```
 */
export async function createUser(request: CreateUserRequest): Promise<User> {
  // 実装
}
```

## 避けるべきパターン

### アンチパターン
```typescript
// 悪い例：any型の多用
function processData(data: any): any {
  return data.someProperty;
}

// 悪い例：型アサーション
const user = data as User; // 型ガードを使用すべき

// 悪い例：非nullアサーション演算子の乱用
const userName = user!.name!; // 適切な型チェックを行うべき

// 悪い例：functionキーワード（アロー関数を優先）
function oldStyle() {
  return this.value; // thisの扱いが複雑
}
```

## ツール設定

### ESLint設定例
```json
{
  "extends": [
    "@typescript-eslint/recommended",
    "@typescript-eslint/recommended-requiring-type-checking"
  ],
  "rules": {
    "@typescript-eslint/no-unused-vars": "error",
    "@typescript-eslint/explicit-function-return-type": "warn",
    "@typescript-eslint/no-explicit-any": "error",
    "@typescript-eslint/prefer-nullish-coalescing": "error",
    "@typescript-eslint/prefer-optional-chain": "error"
  }
}
```

### Prettier設定例
```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": false,
  "printWidth": 80,
  "tabWidth": 2,
  "useTabs": false
}
```

### tsconfig.json設定例
```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "noImplicitReturns": true,
    "noImplicitThis": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true
  }
}
```

---

このガイドラインに従うことで、型安全で保守性の高いTypeScriptコードが書けます。詳細は各公式ドキュメントを参照してください。