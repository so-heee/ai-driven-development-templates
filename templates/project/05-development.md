# 開発ガイドライン

## 開発フロー

### ブランチ戦略

```
main
├── develop
│   ├── feature/user-authentication
│   ├── feature/payment-integration
│   └── bugfix/login-error
├── release/v1.2.0
└── hotfix/critical-security-fix
```

**ブランチタイプ**

- `main`: 本番環境デプロイ用
- `develop`: 開発統合ブランチ
- `feature/`: 新機能開発
- `bugfix/`: バグ修正
- `release/`: リリース準備
- `hotfix/`: 緊急修正

### 開発ワークフロー

1. **Issue作成**: GitHub/Jira でタスク管理
2. **ブランチ作成**: develop から feature ブランチを作成
3. **開発**: コード実装・テスト作成
4. **品質チェック**: リント・テスト・型チェック
5. **Pull Request**: レビュー依頼
6. **コードレビュー**: 品質・設計確認
7. **マージ**: develop ブランチにマージ
8. **デプロイ**: ステージング環境で動作確認

## コーディング規約

### ファイル・ディレクトリ命名

- **ファイル名**: kebab-case
- **コンポーネント**: PascalCase
- **関数・変数**: camelCase
- **定数**: SCREAMING_SNAKE_CASE
- **ディレクトリ**: kebab-case

### JavaScript/TypeScript

```typescript
// ✅ 良い例
const getUserData = async (userId: string): Promise<User> => {
  try {
    const response = await api.get(`/users/${userId}`)
    return response.data
  } catch (error) {
    logger.error('Failed to fetch user data', { userId, error })
    throw new Error('ユーザー情報の取得に失敗しました')
  }
}

// ❌ 悪い例
function get_user(id) {
  return fetch('/users/' + id).then(r => r.json())
}
```

### React/Vue コンポーネント

```tsx
// ✅ React コンポーネントの例
interface UserCardProps {
  user: User
  onEdit: (user: User) => void
}

export const UserCard: React.FC<UserCardProps> = ({ user, onEdit }) => {
  const handleEditClick = useCallback(() => {
    onEdit(user)
  }, [user, onEdit])

  return (
    <div className="user-card">
      <h3>{user.name}</h3>
      <button onClick={handleEditClick}>編集</button>
    </div>
  )
}
```

### CSS/SCSS

```scss
// ✅ BEM記法
.user-card {
  &__header {
    font-size: 1.2rem;
    font-weight: bold;
  }

  &__content {
    padding: 1rem;
  }

  &--highlighted {
    border: 2px solid blue;
  }
}
```

## テスト戦略

### テストピラミッド

```
    E2E テスト (10%)
   ─────────────────
  統合テスト (20%)
 ─────────────────────
単体テスト (70%)
```

### 単体テスト

```typescript
// Jest + Testing Library の例
describe('UserCard', () => {
  it('ユーザー名が正しく表示される', () => {
    const user = { id: '1', name: 'テストユーザー' };
    render(<UserCard user={user} onEdit={jest.fn()} />);

    expect(screen.getByText('テストユーザー')).toBeInTheDocument();
  });

  it('編集ボタンクリックでonEditが呼ばれる', () => {
    const user = { id: '1', name: 'テストユーザー' };
    const onEdit = jest.fn();
    render(<UserCard user={user} onEdit={onEdit} />);

    fireEvent.click(screen.getByText('編集'));
    expect(onEdit).toHaveBeenCalledWith(user);
  });
});
```

### API テスト

```typescript
// Supertest の例
describe('POST /api/users', () => {
  it('有効なデータで新規ユーザーを作成', async () => {
    const userData = {
      email: 'test@example.com',
      name: 'テストユーザー',
    }

    const response = await request(app).post('/api/users').send(userData).expect(201)

    expect(response.body.data.email).toBe(userData.email)
  })
})
```

### E2E テスト

```typescript
// Playwright の例
test('ユーザーログインフロー', async ({ page }) => {
  await page.goto('/login')

  await page.fill('[data-testid=email]', 'test@example.com')
  await page.fill('[data-testid=password]', 'password123')
  await page.click('[data-testid=login-button]')

  await expect(page).toHaveURL('/dashboard')
  await expect(page.getByText('ダッシュボード')).toBeVisible()
})
```

## コードレビュー

### レビュー観点

1. **機能性**: 要件を満たしているか
2. **可読性**: 理解しやすいコードか
3. **保守性**: 修正・拡張しやすいか
4. **パフォーマンス**: 性能上の問題はないか
5. **セキュリティ**: セキュリティホールはないか
6. **テスト**: 適切なテストが書かれているか

### レビューガイドライン

- **建設的なフィードバック**: 問題点と改善案を提示
- **具体的な指摘**: 行番号とコード例を含める
- **優先度の明示**: Must fix / Should fix / Suggestion
- **ポジティブなコメント**: 良い部分も評価する

### Pull Request テンプレート

```markdown
## 概要

<!-- 変更内容の概要 -->

## 変更内容

- [ ] 新機能追加
- [ ] バグ修正
- [ ] リファクタリング
- [ ] ドキュメント更新

## テスト

- [ ] 単体テスト追加
- [ ] 統合テスト確認
- [ ] 手動テスト実施

## チェックリスト

- [ ] リント・フォーマット確認
- [ ] 型チェック通過
- [ ] 全テスト通過
- [ ] セキュリティチェック
```

## パフォーマンス

### フロントエンド最適化

```typescript
// ✅ React最適化の例
const ExpensiveComponent = React.memo(({ data }) => {
  const memoizedValue = useMemo(() => {
    return data.reduce((sum, item) => sum + item.value, 0);
  }, [data]);

  const handleClick = useCallback((id) => {
    // イベントハンドラー
  }, []);

  return <div>{memoizedValue}</div>;
});

// ✅ 遅延読み込み
const LazyComponent = React.lazy(() => import('./LazyComponent'));
```

### バックエンド最適化

```typescript
// ✅ データベースクエリ最適化
const getUsers = async (filters: UserFilters) => {
  return await db.user.findMany({
    where: filters,
    include: {
      profile: true, // 必要な関連データのみ
    },
    take: 20, // ページネーション
    skip: (page - 1) * 20,
  })
}

// ✅ キャッシュ活用
const getCachedUserData = async (userId: string) => {
  const cached = await redis.get(`user:${userId}`)
  if (cached) return JSON.parse(cached)

  const user = await db.user.findUnique({ where: { id: userId } })
  await redis.setex(`user:${userId}`, 300, JSON.stringify(user))
  return user
}
```

## セキュリティ

### 入力検証

```typescript
// ✅ バリデーション例
import { z } from 'zod'

const userSchema = z.object({
  email: z.string().email('有効なメールアドレスを入力してください'),
  password: z.string().min(8, 'パスワードは8文字以上で入力してください'),
  name: z.string().min(1).max(100),
})

const createUser = async (input: unknown) => {
  const validatedInput = userSchema.parse(input)
  // 処理続行
}
```

### 認証・認可

```typescript
// ✅ JWT検証ミドルウェア
const authenticateToken = (req: Request, res: Response, next: NextFunction) => {
  const authHeader = req.headers['authorization']
  const token = authHeader && authHeader.split(' ')[1]

  if (!token) {
    return res.status(401).json({ error: 'アクセストークンが必要です' })
  }

  jwt.verify(token, process.env.JWT_SECRET!, (err, user) => {
    if (err) return res.status(403).json({ error: '無効なトークンです' })
    req.user = user
    next()
  })
}
```

## ログ・監視

### ログ出力

```typescript
// ✅ 構造化ログ
import { logger } from './logger'

const processPayment = async (paymentData: PaymentData) => {
  logger.info('Payment processing started', {
    userId: paymentData.userId,
    amount: paymentData.amount,
    traceId: req.traceId,
  })

  try {
    const result = await paymentService.process(paymentData)
    logger.info('Payment processing completed', {
      userId: paymentData.userId,
      paymentId: result.id,
      traceId: req.traceId,
    })
    return result
  } catch (error) {
    logger.error('Payment processing failed', {
      userId: paymentData.userId,
      error: error.message,
      traceId: req.traceId,
    })
    throw error
  }
}
```

### エラー処理

```typescript
// ✅ エラーハンドリング
class AppError extends Error {
  constructor(
    public message: string,
    public statusCode: number,
    public code: string
  ) {
    super(message)
  }
}

const errorHandler = (error: Error, req: Request, res: Response, next: NextFunction) => {
  if (error instanceof AppError) {
    return res.status(error.statusCode).json({
      success: false,
      error: {
        code: error.code,
        message: error.message,
      },
    })
  }

  logger.error('Unexpected error', { error: error.stack })
  res.status(500).json({
    success: false,
    error: {
      code: 'INTERNAL_SERVER_ERROR',
      message: 'サーバー内部エラーが発生しました',
    },
  })
}
```

## 依存関係管理

### パッケージ選定基準

- **アクティブなメンテナンス**: 定期的な更新
- **セキュリティ**: 既知の脆弱性がない
- **パフォーマンス**: バンドルサイズが適切
- **型サポート**: TypeScript対応

### 更新戦略

```bash
# 定期的な依存関係チェック
npm audit
npm outdated

# セキュリティ修正
npm audit fix

# 依存関係更新
npm update
```
