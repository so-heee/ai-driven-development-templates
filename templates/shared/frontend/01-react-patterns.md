# Reactパターンガイド

## 基本的なコンポーネントパターン

### 関数コンポーネントの基本形

```tsx
interface UserCardProps {
  user: User
  onEdit?: (user: User) => void
  className?: string
}

export const UserCard: React.FC<UserCardProps> = ({ user, onEdit, className = '' }) => {
  const handleEdit = useCallback(() => {
    onEdit?.(user)
  }, [user, onEdit])

  return (
    <div className={`user-card ${className}`}>
      <h3>{user.name}</h3>
      {onEdit && <button onClick={handleEdit}>編集</button>}
    </div>
  )
}
```

### Propsの型定義パターン

```tsx
// 基本的なProps
interface BaseProps {
  children?: React.ReactNode
  className?: string
  'data-testid'?: string
}

// 条件付きProps
interface ButtonProps extends BaseProps {
  variant: 'primary' | 'secondary' | 'danger'
  size?: 'small' | 'medium' | 'large'
  disabled?: boolean
  loading?: boolean
  onClick: () => void
}

// HTMLProps継承
interface InputProps extends React.InputHTMLAttributes<HTMLInputElement> {
  label: string
  error?: string
  helperText?: string
}
```

## Hooksパターン

### カスタムフックの基本形

```tsx
// データフェッチング用フック
export const useUser = (userId: string) => {
  const [user, setUser] = useState<User | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const fetchUser = async () => {
      try {
        setLoading(true)
        setError(null)
        const userData = await api.getUser(userId)
        setUser(userData)
      } catch (err) {
        setError(err instanceof Error ? err.message : 'エラーが発生しました')
      } finally {
        setLoading(false)
      }
    }

    if (userId) {
      fetchUser()
    }
  }, [userId])

  return { user, loading, error }
}
```

### フォーム管理フック

```tsx
export const useForm = <T extends Record<string, any>>(initialValues: T, validationSchema?: z.ZodSchema<T>) => {
  const [values, setValues] = useState<T>(initialValues)
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({})
  const [touched, setTouched] = useState<Partial<Record<keyof T, boolean>>>({})

  const setValue = useCallback(
    (name: keyof T, value: T[keyof T]) => {
      setValues(prev => ({ ...prev, [name]: value }))

      // バリデーション実行
      if (validationSchema && touched[name]) {
        try {
          validationSchema.parse({ ...values, [name]: value })
          setErrors(prev => ({ ...prev, [name]: undefined }))
        } catch (error) {
          if (error instanceof z.ZodError) {
            const fieldError = error.errors.find(e => e.path[0] === name)
            setErrors(prev => ({ ...prev, [name]: fieldError?.message }))
          }
        }
      }
    },
    [values, validationSchema, touched]
  )

  const setTouchedField = useCallback((name: keyof T) => {
    setTouched(prev => ({ ...prev, [name]: true }))
  }, [])

  const isValid = Object.keys(errors).length === 0

  return {
    values,
    errors,
    touched,
    setValue,
    setTouchedField,
    isValid,
  }
}
```

### ローカルストレージフック

```tsx
export const useLocalStorage = <T>(
  key: string,
  initialValue: T
): [T, (value: T | ((val: T) => T)) => void] => {
  const [storedValue, setStoredValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error(`Error reading localStorage key "${key}":`, error);
      return initialValue;
    }
  });

  const setValue = useCallback((value: T | ((val: T) => T)) => {
    try {
      const valueToStore = value instanceof Function ? value(storedValue) : value;
      setStoredValue(valueToStore);
      window.localStorage.setItem(key, JSON.stringify(valueToStore));
    } catch (error) {
      console.error(`Error setting localStorage key "${key}":`, error);
    }
  }, [key, storedValue]);

  return [storedValue, setValue];
};
```

## レンダーパターン

### 条件付きレンダリング

```tsx
// 早期リターンパターン
const UserProfile: React.FC<{ userId: string }> = ({ userId }) => {
  const { user, loading, error } = useUser(userId)

  if (loading) return <LoadingSpinner />
  if (error) return <ErrorMessage message={error} />
  if (!user) return <NotFound message="ユーザーが見つかりません" />

  return (
    <div className="user-profile">
      <UserCard user={user} />
    </div>
  )
}

// 三項演算子パターン
const ToggleButton: React.FC<{ isOn: boolean; onToggle: () => void }> = ({ isOn, onToggle }) => (
  <button className={`toggle-button ${isOn ? 'on' : 'off'}`} onClick={onToggle}>
    {isOn ? 'ON' : 'OFF'}
  </button>
)

// &&演算子パターン
const NotificationBadge: React.FC<{ count: number }> = ({ count }) => (
  <div className="notification-container">{count > 0 && <span className="badge">{count}</span>}</div>
)
```

### リストレンダリング

```tsx
const UserList: React.FC<{ users: User[] }> = ({ users }) => {
  if (users.length === 0) {
    return <EmptyState message="ユーザーがいません" />
  }

  return (
    <div className="user-list">
      {users.map(user => (
        <UserCard key={user.id} user={user} onEdit={handleEditUser} />
      ))}
    </div>
  )
}

// 仮想化が必要な大量データの場合
import { FixedSizeList as List } from 'react-window'

const VirtualizedUserList: React.FC<{ users: User[] }> = ({ users }) => {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      <UserCard user={users[index]} />
    </div>
  )

  return (
    <List height={600} itemCount={users.length} itemSize={100}>
      {Row}
    </List>
  )
}
```

## 高階コンポーネント（HOC）パターン

### 認証HOC

```tsx
export const withAuth = <P extends object>(WrappedComponent: React.ComponentType<P>) => {
  const AuthenticatedComponent: React.FC<P> = props => {
    const { user, loading } = useAuth()

    if (loading) return <LoadingSpinner />
    if (!user) return <Navigate to="/login" />

    return <WrappedComponent {...props} />
  }

  AuthenticatedComponent.displayName = `withAuth(${WrappedComponent.displayName || WrappedComponent.name})`

  return AuthenticatedComponent
}

// 使用例
const ProtectedDashboard = withAuth(Dashboard)
```

### エラー境界HOC

```tsx
interface ErrorBoundaryState {
  hasError: boolean
  error?: Error
}

export const withErrorBoundary = <P extends object>(WrappedComponent: React.ComponentType<P>) => {
  class ErrorBoundaryWrapper extends React.Component<P, ErrorBoundaryState> {
    constructor(props: P) {
      super(props)
      this.state = { hasError: false }
    }

    static getDerivedStateFromError(error: Error): ErrorBoundaryState {
      return { hasError: true, error }
    }

    componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
      console.error('Error caught by boundary:', error, errorInfo)
      // エラー報告サービスに送信
    }

    render() {
      if (this.state.hasError) {
        return <ErrorFallback error={this.state.error} />
      }

      return <WrappedComponent {...this.props} />
    }
  }

  return ErrorBoundaryWrapper
}
```

## Render Propsパターン

### データプロバイダー

```tsx
interface RenderProps<T> {
  data: T | null
  loading: boolean
  error: string | null
  refetch: () => void
}

interface DataProviderProps<T> {
  url: string
  children: (props: RenderProps<T>) => React.ReactNode
}

export const DataProvider = <T,>({ url, children }: DataProviderProps<T>) => {
  const [data, setData] = useState<T | null>(null)
  const [loading, setLoading] = useState(true)
  const [error, setError] = useState<string | null>(null)

  const fetchData = useCallback(async () => {
    try {
      setLoading(true)
      setError(null)
      const response = await fetch(url)
      const result = await response.json()
      setData(result)
    } catch (err) {
      setError(err instanceof Error ? err.message : 'エラーが発生しました')
    } finally {
      setLoading(false)
    }
  }, [url])

  useEffect(() => {
    fetchData()
  }, [fetchData])

  return <>{children({ data, loading, error, refetch: fetchData })}</>
}

// 使用例
const UserProfile = () => (
  <DataProvider<User> url="/api/user/1">
    {({ data: user, loading, error }) => {
      if (loading) return <LoadingSpinner />
      if (error) return <ErrorMessage message={error} />
      if (!user) return <NotFound />
      return <UserCard user={user} />
    }}
  </DataProvider>
)
```

## Compound Componentパターン

### ドロップダウンメニュー

```tsx
interface DropdownContextType {
  isOpen: boolean
  toggle: () => void
  close: () => void
}

const DropdownContext = React.createContext<DropdownContextType | null>(null)

const useDropdown = () => {
  const context = useContext(DropdownContext)
  if (!context) {
    throw new Error('useDropdown must be used within a Dropdown')
  }
  return context
}

export const Dropdown: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [isOpen, setIsOpen] = useState(false)

  const toggle = useCallback(() => setIsOpen(prev => !prev), [])
  const close = useCallback(() => setIsOpen(false), [])

  useEffect(() => {
    const handleClickOutside = () => close()
    if (isOpen) {
      document.addEventListener('click', handleClickOutside)
      return () => document.removeEventListener('click', handleClickOutside)
    }
  }, [isOpen, close])

  return (
    <DropdownContext.Provider value={{ isOpen, toggle, close }}>
      <div className="dropdown">{children}</div>
    </DropdownContext.Provider>
  )
}

Dropdown.Trigger = ({ children }: { children: React.ReactNode }) => {
  const { toggle } = useDropdown()
  return (
    <button onClick={toggle} className="dropdown-trigger">
      {children}
    </button>
  )
}

Dropdown.Menu = ({ children }: { children: React.ReactNode }) => {
  const { isOpen } = useDropdown()
  if (!isOpen) return null

  return <div className="dropdown-menu">{children}</div>
}

Dropdown.Item = ({ children, onClick }: { children: React.ReactNode; onClick: () => void }) => {
  const { close } = useDropdown()

  const handleClick = () => {
    onClick()
    close()
  }

  return (
    <button className="dropdown-item" onClick={handleClick}>
      {children}
    </button>
  )
}
```

## パフォーマンス最適化パターン

### メモ化

```tsx
// React.memo での props メモ化
export const ExpensiveComponent = React.memo<{
  data: ComplexData[]
  onItemClick: (id: string) => void
}>(
  ({ data, onItemClick }) => {
    console.log('ExpensiveComponent rendering')

    return (
      <div>
        {data.map(item => (
          <div key={item.id} onClick={() => onItemClick(item.id)}>
            {item.name}
          </div>
        ))}
      </div>
    )
  },
  (prevProps, nextProps) => {
    // カスタム比較関数
    return prevProps.data.length === nextProps.data.length && prevProps.onItemClick === nextProps.onItemClick
  }
)

// useMemo での計算結果メモ化
const ProcessedData: React.FC<{ rawData: RawData[] }> = ({ rawData }) => {
  const processedData = useMemo(() => {
    console.log('Processing data...')
    return rawData
      .filter(item => item.isActive)
      .map(item => ({ ...item, displayName: `${item.name} (${item.category})` }))
      .sort((a, b) => a.displayName.localeCompare(b.displayName))
  }, [rawData])

  return (
    <div>
      {processedData.map(item => (
        <div key={item.id}>{item.displayName}</div>
      ))}
    </div>
  )
}

// useCallback でのイベントハンドラメモ化
const OptimizedList: React.FC<{ items: Item[] }> = ({ items }) => {
  const [selectedId, setSelectedId] = useState<string | null>(null)

  const handleItemClick = useCallback((id: string) => {
    setSelectedId(id)
  }, [])

  return (
    <div>
      {items.map(item => (
        <OptimizedItem key={item.id} item={item} isSelected={selectedId === item.id} onClick={handleItemClick} />
      ))}
    </div>
  )
}
```
