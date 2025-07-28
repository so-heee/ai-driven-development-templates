# フロントエンドテストパターン

## テスト戦略とピラミッド

### テストピラミッド
```
        E2E テスト (5-10%)
       ・シナリオテスト
       ・ユーザージャーニー
      ─────────────────────
     統合テスト (15-25%)
    ・コンポーネント間の連携
    ・API統合テスト
   ─────────────────────────
  単体テスト (70-80%)
 ・個別関数・コンポーネント
・ビジネスロジック
```

### テストの分類と目的
```typescript
// 単体テスト: 独立した機能の検証
describe('ユーティリティ関数', () => {
  test('formatCurrency: 数値を通貨形式でフォーマット', () => {
    expect(formatCurrency(1000)).toBe('¥1,000');
    expect(formatCurrency(0)).toBe('¥0');
    expect(formatCurrency(1234.56)).toBe('¥1,234');
  });
});

// 統合テスト: コンポーネント間の連携
describe('ユーザー管理機能', () => {
  test('ユーザー作成フローが正常に動作する', async () => {
    render(<UserManagement />);
    
    // フォーム入力
    await user.type(screen.getByLabelText('名前'), 'テストユーザー');
    await user.type(screen.getByLabelText('メール'), 'test@example.com');
    
    // 送信
    await user.click(screen.getByRole('button', { name: '作成' }));
    
    // 結果確認
    await waitFor(() => {
      expect(screen.getByText('ユーザーが作成されました')).toBeInTheDocument();
    });
  });
});

// E2Eテスト: エンドユーザーの体験を検証
test('ログインからダッシュボード表示まで', async ({ page }) => {
  await page.goto('/login');
  await page.fill('[data-testid=email]', 'user@example.com');
  await page.fill('[data-testid=password]', 'password');
  await page.click('[data-testid=login-button]');
  
  await expect(page).toHaveURL('/dashboard');
  await expect(page.getByText('ダッシュボード')).toBeVisible();
});
```

## 単体テスト（Jest + Testing Library）

### 基本的なコンポーネントテスト
```tsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { vi } from 'vitest';

// テスト対象コンポーネント
interface UserCardProps {
  user: User;
  onEdit: (user: User) => void;
  onDelete: (id: string) => void;
}

const UserCard: React.FC<UserCardProps> = ({ user, onEdit, onDelete }) => {
  const [isEditing, setIsEditing] = useState(false);
  
  return (
    <div data-testid={`user-card-${user.id}`}>
      <h3>{user.name}</h3>
      <p>{user.email}</p>
      {isEditing ? (
        <EditForm user={user} onSave={onEdit} onCancel={() => setIsEditing(false)} />
      ) : (
        <div>
          <button onClick={() => setIsEditing(true)}>編集</button>
          <button onClick={() => onDelete(user.id)}>削除</button>
        </div>
      )}
    </div>
  );
};

// テストケース
describe('UserCard', () => {
  const mockUser: User = {
    id: '1',
    name: 'テストユーザー',
    email: 'test@example.com'
  };
  
  const mockOnEdit = vi.fn();
  const mockOnDelete = vi.fn();
  
  beforeEach(() => {
    vi.clearAllMocks();
  });
  
  test('ユーザー情報が正しく表示される', () => {
    render(<UserCard user={mockUser} onEdit={mockOnEdit} onDelete={mockOnDelete} />);
    
    expect(screen.getByText('テストユーザー')).toBeInTheDocument();
    expect(screen.getByText('test@example.com')).toBeInTheDocument();
  });
  
  test('編集ボタンクリックで編集モードになる', async () => {
    const user = userEvent.setup();
    render(<UserCard user={mockUser} onEdit={mockOnEdit} onDelete={mockOnDelete} />);
    
    await user.click(screen.getByRole('button', { name: '編集' }));
    
    expect(screen.getByTestId('edit-form')).toBeInTheDocument();
    expect(screen.queryByRole('button', { name: '編集' })).not.toBeInTheDocument();
  });
  
  test('削除ボタンクリックでonDeleteが呼ばれる', async () => {
    const user = userEvent.setup();
    render(<UserCard user={mockUser} onEdit={mockOnEdit} onDelete={mockOnDelete} />);
    
    await user.click(screen.getByRole('button', { name: '削除' }));
    
    expect(mockOnDelete).toHaveBeenCalledTimes(1);
    expect(mockOnDelete).toHaveBeenCalledWith('1');
  });
});
```

### カスタムフックテスト
```tsx
import { renderHook, act } from '@testing-library/react';

// テスト対象フック
const useCounter = (initialValue = 0) => {
  const [count, setCount] = useState(initialValue);
  
  const increment = useCallback(() => {
    setCount(prev => prev + 1);
  }, []);
  
  const decrement = useCallback(() => {
    setCount(prev => prev - 1);
  }, []);
  
  const reset = useCallback(() => {
    setCount(initialValue);
  }, [initialValue]);
  
  return { count, increment, decrement, reset };
};

// テストケース
describe('useCounter', () => {
  test('初期値が正しく設定される', () => {
    const { result } = renderHook(() => useCounter(5));
    expect(result.current.count).toBe(5);
  });
  
  test('incrementで値が増加する', () => {
    const { result } = renderHook(() => useCounter());
    
    act(() => {
      result.current.increment();
    });
    
    expect(result.current.count).toBe(1);
  });
  
  test('decrementで値が減少する', () => {
    const { result } = renderHook(() => useCounter(5));
    
    act(() => {
      result.current.decrement();
    });
    
    expect(result.current.count).toBe(4);
  });
  
  test('resetで初期値に戻る', () => {
    const { result } = renderHook(() => useCounter(5));
    
    act(() => {
      result.current.increment();
      result.current.increment();
    });
    
    expect(result.current.count).toBe(7);
    
    act(() => {
      result.current.reset();
    });
    
    expect(result.current.count).toBe(5);
  });
});
```

### 非同期処理のテスト
```tsx
import { vi } from 'vitest';

// API呼び出しを含むコンポーネント
const UserList: React.FC = () => {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  const fetchUsers = useCallback(async () => {
    setLoading(true);
    setError(null);
    try {
      const response = await api.getUsers();
      setUsers(response.data);
    } catch (err) {
      setError('ユーザーの取得に失敗しました');
    } finally {
      setLoading(false);
    }
  }, []);
  
  useEffect(() => {
    fetchUsers();
  }, [fetchUsers]);
  
  if (loading) return <div>読み込み中...</div>;
  if (error) return <div role="alert">{error}</div>;
  
  return (
    <ul>
      {users.map(user => (
        <li key={user.id}>{user.name}</li>
      ))}
    </ul>
  );
};

// API のモック
vi.mock('../api', () => ({
  api: {
    getUsers: vi.fn()
  }
}));

const mockApi = vi.mocked(api);

describe('UserList', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });
  
  test('ユーザー一覧が正常に表示される', async () => {
    const mockUsers = [
      { id: '1', name: 'ユーザー1' },
      { id: '2', name: 'ユーザー2' }
    ];
    
    mockApi.getUsers.mockResolvedValueOnce({ data: mockUsers });
    
    render(<UserList />);
    
    // ローディング状態の確認
    expect(screen.getByText('読み込み中...')).toBeInTheDocument();
    
    // データ取得完了を待機
    await waitFor(() => {
      expect(screen.getByText('ユーザー1')).toBeInTheDocument();
      expect(screen.getByText('ユーザー2')).toBeInTheDocument();
    });
    
    expect(screen.queryByText('読み込み中...')).not.toBeInTheDocument();
  });
  
  test('API エラー時にエラーメッセージが表示される', async () => {
    mockApi.getUsers.mockRejectedValueOnce(new Error('Network Error'));
    
    render(<UserList />);
    
    await waitFor(() => {
      expect(screen.getByRole('alert')).toHaveTextContent('ユーザーの取得に失敗しました');
    });
  });
});
```

## モック・スタブパターン

### MSW（Mock Service Worker）
```tsx
// mocks/handlers.ts
import { rest } from 'msw';

export const handlers = [
  rest.get('/api/users', (req, res, ctx) => {
    return res(
      ctx.status(200),
      ctx.json({
        users: [
          { id: '1', name: 'ユーザー1', email: 'user1@example.com' },
          { id: '2', name: 'ユーザー2', email: 'user2@example.com' }
        ]
      })
    );
  }),
  
  rest.post('/api/users', async (req, res, ctx) => {
    const { name, email } = await req.json();
    
    if (!name || !email) {
      return res(
        ctx.status(400),
        ctx.json({ error: '名前とメールは必須です' })
      );
    }
    
    return res(
      ctx.status(201),
      ctx.json({
        id: Date.now().toString(),
        name,
        email
      })
    );
  }),
  
  rest.get('/api/users/:id', (req, res, ctx) => {
    const { id } = req.params;
    
    if (id === 'not-found') {
      return res(
        ctx.status(404),
        ctx.json({ error: 'ユーザーが見つかりません' })
      );
    }
    
    return res(
      ctx.status(200),
      ctx.json({
        id,
        name: `ユーザー${id}`,
        email: `user${id}@example.com`
      })
    );
  })
];

// mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);

// setup.ts
import { beforeAll, afterEach, afterAll } from 'vitest';
import { server } from './mocks/server';

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

// 動的なレスポンス変更
test('ネットワークエラーのハンドリング', async () => {
  server.use(
    rest.get('/api/users', (req, res, ctx) => {
      return res.networkError('ネットワークエラー');
    })
  );
  
  render(<UserList />);
  
  await waitFor(() => {
    expect(screen.getByText('ネットワークエラーが発生しました')).toBeInTheDocument();
  });
});
```

### Context・Provider のモック
```tsx
// テスト用プロバイダー
const createMockAuthProvider = (user: User | null = null) => {
  const MockAuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
    const value = {
      user,
      login: vi.fn(),
      logout: vi.fn(),
      loading: false
    };
    
    return (
      <AuthContext.Provider value={value}>
        {children}
      </AuthContext.Provider>
    );
  };
  
  return MockAuthProvider;
};

// カスタムレンダー関数
const renderWithAuth = (ui: React.ReactElement, user: User | null = null) => {
  const MockProvider = createMockAuthProvider(user);
  
  return render(
    <MockProvider>
      {ui}
    </MockProvider>
  );
};

// テスト例
test('ログイン済みユーザーにダッシュボードが表示される', () => {
  const mockUser = { id: '1', name: 'テストユーザー' };
  
  renderWithAuth(<Dashboard />, mockUser);
  
  expect(screen.getByText('ダッシュボード')).toBeInTheDocument();
  expect(screen.getByText('テストユーザー さん、こんにちは')).toBeInTheDocument();
});

test('未ログインユーザーにログインページが表示される', () => {
  renderWithAuth(<Dashboard />, null);
  
  expect(screen.getByText('ログインしてください')).toBeInTheDocument();
});
```

## E2Eテスト（Playwright）

### 基本的なE2Eテスト
```typescript
// e2e/login.spec.ts
import { test, expect } from '@playwright/test';

test.describe('ログイン機能', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/login');
  });
  
  test('正常なログインフロー', async ({ page }) => {
    // ログインフォームに入力
    await page.fill('[data-testid=email]', 'user@example.com');
    await page.fill('[data-testid=password]', 'password123');
    
    // ログインボタンをクリック
    await page.click('[data-testid=login-button]');
    
    // ダッシュボードにリダイレクトされることを確認
    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText('ダッシュボード')).toBeVisible();
  });
  
  test('不正な認証情報でのログイン失敗', async ({ page }) => {
    await page.fill('[data-testid=email]', 'invalid@example.com');
    await page.fill('[data-testid=password]', 'wrongpassword');
    await page.click('[data-testid=login-button]');
    
    // エラーメッセージが表示されることを確認
    await expect(page.getByText('メールアドレスまたはパスワードが正しくありません')).toBeVisible();
    
    // ログインページに留まることを確認
    await expect(page).toHaveURL('/login');
  });
  
  test('バリデーションエラーの表示', async ({ page }) => {
    // 空の状態でログインボタンをクリック
    await page.click('[data-testid=login-button]');
    
    // バリデーションエラーが表示されることを確認
    await expect(page.getByText('メールアドレスは必須です')).toBeVisible();
    await expect(page.getByText('パスワードは必須です')).toBeVisible();
  });
});

// ユーザージャーニーテスト
test.describe('ユーザー管理フロー', () => {
  test('新規ユーザー作成から編集まで', async ({ page }) => {
    // ログイン
    await page.goto('/login');
    await page.fill('[data-testid=email]', 'admin@example.com');
    await page.fill('[data-testid=password]', 'password');
    await page.click('[data-testid=login-button]');
    
    // ユーザー管理ページに移動
    await page.click('text=ユーザー管理');
    await expect(page).toHaveURL('/users');
    
    // 新規ユーザー作成
    await page.click('[data-testid=add-user-button]');
    await page.fill('[data-testid=user-name]', 'テスト太郎');
    await page.fill('[data-testid=user-email]', 'test@example.com');
    await page.selectOption('[data-testid=user-role]', 'user');
    await page.click('[data-testid=save-button]');
    
    // 作成されたユーザーがリストに表示されることを確認
    await expect(page.getByText('テスト太郎')).toBeVisible();
    await expect(page.getByText('test@example.com')).toBeVisible();
    
    // ユーザー編集
    await page.click('[data-testid=edit-user-test@example.com]');
    await page.fill('[data-testid=user-name]', 'テスト次郎');
    await page.click('[data-testid=save-button]');
    
    // 更新されたことを確認
    await expect(page.getByText('テスト次郎')).toBeVisible();
    await expect(page.getByText('テスト太郎')).not.toBeVisible();
  });
});
```

### ページオブジェクトモデル
```typescript
// e2e/pages/LoginPage.ts
export class LoginPage {
  constructor(private page: Page) {}
  
  async goto() {
    await this.page.goto('/login');
  }
  
  async fillEmail(email: string) {
    await this.page.fill('[data-testid=email]', email);
  }
  
  async fillPassword(password: string) {
    await this.page.fill('[data-testid=password]', password);
  }
  
  async clickLoginButton() {
    await this.page.click('[data-testid=login-button]');
  }
  
  async login(email: string, password: string) {
    await this.fillEmail(email);
    await this.fillPassword(password);
    await this.clickLoginButton();
  }
  
  async expectValidationError(message: string) {
    await expect(this.page.getByText(message)).toBeVisible();
  }
}

// テストでの使用
test('ページオブジェクトモデルの使用例', async ({ page }) => {
  const loginPage = new LoginPage(page);
  
  await loginPage.goto();
  await loginPage.login('user@example.com', 'password');
  
  await expect(page).toHaveURL('/dashboard');
});
```

## ビジュアルリグレッションテスト

### Storybook でのスナップショットテスト
```tsx
// Button.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Button } from './Button';

const meta: Meta<typeof Button> = {
  title: 'Components/Button',
  component: Button,
  parameters: {
    layout: 'centered',
  },
  argTypes: {
    backgroundColor: { control: 'color' },
  },
};

export default meta;
type Story = StoryObj<typeof meta>;

export const Primary: Story = {
  args: {
    primary: true,
    label: 'Button',
  },
};

export const Secondary: Story = {
  args: {
    label: 'Button',
  },
};

export const Large: Story = {
  args: {
    size: 'large',
    label: 'Button',
  },
};

export const Small: Story = {
  args: {
    size: 'small',
    label: 'Button',
  },
};

// ビジュアルリグレッションテスト
import { test, expect } from '@playwright/test';

test.describe('Button Visual Tests', () => {
  test('Button variants', async ({ page }) => {
    await page.goto('/iframe.html?id=components-button--primary');
    await expect(page).toHaveScreenshot('button-primary.png');
    
    await page.goto('/iframe.html?id=components-button--secondary');
    await expect(page).toHaveScreenshot('button-secondary.png');
  });
});
```

## テストユーティリティとヘルパー

### カスタムマッチャー
```typescript
// test-utils/custom-matchers.ts
import { expect } from '@playwright/test';

expect.extend({
  async toHaveLoadingState(locator: Locator) {
    const isLoading = await locator.getAttribute('aria-busy') === 'true' ||
                     await locator.getByRole('status').isVisible();
    
    return {
      message: () => isLoading 
        ? 'Expected element not to have loading state'
        : 'Expected element to have loading state',
      pass: isLoading,
    };
  },
  
  async toBeAccessible(page: Page) {
    const violations = await page.evaluate(() => {
      return new Promise((resolve) => {
        // axe-core を使用したアクセシビリティ検証
        axe.run().then(results => resolve(results.violations));
      });
    });
    
    return {
      message: () => violations.length > 0 
        ? `Accessibility violations found: ${JSON.stringify(violations, null, 2)}`
        : 'No accessibility violations found',
      pass: violations.length === 0,
    };
  },
});

// 使用例
test('ローディング状態のテスト', async ({ page }) => {
  await page.click('[data-testid=load-data]');
  await expect(page.locator('[data-testid=data-container]')).toHaveLoadingState();
});

test('アクセシビリティテスト', async ({ page }) => {
  await page.goto('/form');
  await expect(page).toBeAccessible();
});
```

### テストデータファクトリ
```typescript
// test-utils/factories.ts
interface UserFactory {
  id?: string;
  name?: string;
  email?: string;
  role?: 'admin' | 'user';
  createdAt?: Date;
}

export const createUser = (overrides: UserFactory = {}): User => ({
  id: faker.string.uuid(),
  name: faker.person.fullName(),
  email: faker.internet.email(),
  role: 'user',
  createdAt: new Date(),
  ...overrides,
});

export const createUsers = (count: number, overrides: UserFactory = {}): User[] => {
  return Array.from({ length: count }, () => createUser(overrides));
};

// テストでの使用
test('ユーザー一覧表示', () => {
  const mockUsers = createUsers(5, { role: 'admin' });
  
  render(<UserList users={mockUsers} />);
  
  expect(screen.getAllByRole('listitem')).toHaveLength(5);
  mockUsers.forEach(user => {
    expect(screen.getByText(user.name)).toBeInTheDocument();
  });
});
```