# 状態管理ガイド

## React標準の状態管理

### useState パターン
```tsx
// 基本的な状態管理
const Counter: React.FC = () => {
  const [count, setCount] = useState(0);
  const [step, setStep] = useState(1);

  const increment = useCallback(() => {
    setCount(prev => prev + step);
  }, [step]);

  const decrement = useCallback(() => {
    setCount(prev => prev - step);
  }, [step]);

  return (
    <div>
      <p>Count: {count}</p>
      <input 
        type="number" 
        value={step} 
        onChange={(e) => setStep(Number(e.target.value))} 
      />
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
    </div>
  );
};

// 複雑なオブジェクト状態の管理
interface UserProfile {
  name: string;
  email: string;
  preferences: {
    theme: 'light' | 'dark';
    notifications: boolean;
  };
}

const ProfileForm: React.FC = () => {
  const [profile, setProfile] = useState<UserProfile>({
    name: '',
    email: '',
    preferences: {
      theme: 'light',
      notifications: true
    }
  });

  const updateField = useCallback((field: keyof UserProfile, value: any) => {
    setProfile(prev => ({
      ...prev,
      [field]: value
    }));
  }, []);

  const updatePreference = useCallback((key: keyof UserProfile['preferences'], value: any) => {
    setProfile(prev => ({
      ...prev,
      preferences: {
        ...prev.preferences,
        [key]: value
      }
    }));
  }, []);

  return (
    <form>
      <input
        value={profile.name}
        onChange={(e) => updateField('name', e.target.value)}
        placeholder="名前"
      />
      <input
        value={profile.email}
        onChange={(e) => updateField('email', e.target.value)}
        placeholder="メール"
      />
      <select
        value={profile.preferences.theme}
        onChange={(e) => updatePreference('theme', e.target.value)}
      >
        <option value="light">ライト</option>
        <option value="dark">ダーク</option>
      </select>
    </form>
  );
};
```

### useReducer パターン
```tsx
// アクション定義
type CounterAction = 
  | { type: 'INCREMENT'; payload?: number }
  | { type: 'DECREMENT'; payload?: number }
  | { type: 'RESET' }
  | { type: 'SET_VALUE'; payload: number };

// 状態定義
interface CounterState {
  value: number;
  step: number;
}

// リデューサー関数
const counterReducer = (state: CounterState, action: CounterAction): CounterState => {
  switch (action.type) {
    case 'INCREMENT':
      return {
        ...state,
        value: state.value + (action.payload ?? state.step)
      };
    case 'DECREMENT':
      return {
        ...state,
        value: state.value - (action.payload ?? state.step)
      };
    case 'RESET':
      return {
        ...state,
        value: 0
      };
    case 'SET_VALUE':
      return {
        ...state,
        value: action.payload
      };
    default:
      return state;
  }
};

// コンポーネントでの使用
const CounterWithReducer: React.FC = () => {
  const [state, dispatch] = useReducer(counterReducer, {
    value: 0,
    step: 1
  });

  return (
    <div>
      <p>Value: {state.value}</p>
      <button onClick={() => dispatch({ type: 'INCREMENT' })}>
        +{state.step}
      </button>
      <button onClick={() => dispatch({ type: 'DECREMENT' })}>
        -{state.step}
      </button>
      <button onClick={() => dispatch({ type: 'RESET' })}>
        Reset
      </button>
    </div>
  );
};

// 複雑なフォーム状態管理
type FormAction = 
  | { type: 'SET_FIELD'; field: string; value: any }
  | { type: 'SET_ERROR'; field: string; error: string }
  | { type: 'CLEAR_ERRORS' }
  | { type: 'RESET_FORM' };

interface FormState {
  values: Record<string, any>;
  errors: Record<string, string>;
  touched: Record<string, boolean>;
  isSubmitting: boolean;
}

const formReducer = (state: FormState, action: FormAction): FormState => {
  switch (action.type) {
    case 'SET_FIELD':
      return {
        ...state,
        values: {
          ...state.values,
          [action.field]: action.value
        },
        touched: {
          ...state.touched,
          [action.field]: true
        }
      };
    case 'SET_ERROR':
      return {
        ...state,
        errors: {
          ...state.errors,
          [action.field]: action.error
        }
      };
    case 'CLEAR_ERRORS':
      return {
        ...state,
        errors: {}
      };
    case 'RESET_FORM':
      return {
        values: {},
        errors: {},
        touched: {},
        isSubmitting: false
      };
    default:
      return state;
  }
};
```

## React Context API

### 基本的なContext設定
```tsx
// コンテキスト定義
interface AuthContextType {
  user: User | null;
  login: (email: string, password: string) => Promise<void>;
  logout: () => void;
  loading: boolean;
}

const AuthContext = React.createContext<AuthContextType | null>(null);

// カスタムフック
export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within an AuthProvider');
  }
  return context;
};

// プロバイダーコンポーネント
export const AuthProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  const login = useCallback(async (email: string, password: string) => {
    setLoading(true);
    try {
      const response = await authAPI.login(email, password);
      setUser(response.user);
      localStorage.setItem('token', response.token);
    } catch (error) {
      throw error;
    } finally {
      setLoading(false);
    }
  }, []);

  const logout = useCallback(() => {
    setUser(null);
    localStorage.removeItem('token');
  }, []);

  useEffect(() => {
    const token = localStorage.getItem('token');
    if (token) {
      authAPI.verifyToken(token)
        .then(user => setUser(user))
        .catch(() => localStorage.removeItem('token'))
        .finally(() => setLoading(false));
    } else {
      setLoading(false);
    }
  }, []);

  const value = useMemo(() => ({
    user,
    login,
    logout,
    loading
  }), [user, login, logout, loading]);

  return (
    <AuthContext.Provider value={value}>
      {children}
    </AuthContext.Provider>
  );
};

// 使用例
const LoginButton: React.FC = () => {
  const { user, login, logout } = useAuth();
  
  if (user) {
    return <button onClick={logout}>ログアウト</button>;
  }
  
  return <button onClick={() => login('test@example.com', 'password')}>ログイン</button>;
};
```

### Context の最適化
```tsx
// 状態を分離してレンダリングを最適化
interface ThemeContextType {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

interface SettingsContextType {
  language: string;
  notifications: boolean;
  updateSettings: (settings: Partial<Settings>) => void;
}

// テーマ専用Context
const ThemeContext = React.createContext<ThemeContextType | null>(null);

// 設定専用Context
const SettingsContext = React.createContext<SettingsContextType | null>(null);

// 組み合わせプロバイダー
export const AppProviders: React.FC<{ children: React.ReactNode }> = ({ children }) => (
  <ThemeProvider>
    <SettingsProvider>
      <AuthProvider>
        {children}
      </AuthProvider>
    </SettingsProvider>
  </ThemeProvider>
);
```

## Zustand（軽量状態管理）

### 基本的なストア
```tsx
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

// 状態とアクションの定義
interface CounterStore {
  count: number;
  increment: () => void;
  decrement: () => void;
  reset: () => void;
  setCount: (count: number) => void;
}

// ストア作成
export const useCounterStore = create<CounterStore>()(
  devtools(
    persist(
      (set) => ({
        count: 0,
        increment: () => set((state) => ({ count: state.count + 1 })),
        decrement: () => set((state) => ({ count: state.count - 1 })),
        reset: () => set({ count: 0 }),
        setCount: (count) => set({ count }),
      }),
      {
        name: 'counter-storage'
      }
    )
  )
);

// コンポーネントでの使用
const Counter: React.FC = () => {
  const { count, increment, decrement, reset } = useCounterStore();
  
  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={increment}>+</button>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>Reset</button>
    </div>
  );
};

// 選択的サブスクリプション
const CountDisplay: React.FC = () => {
  // countの変更時のみ再レンダリング
  const count = useCounterStore((state) => state.count);
  return <span>{count}</span>;
};
```

### 複雑なストア例
```tsx
interface Todo {
  id: string;
  text: string;
  completed: boolean;
  createdAt: Date;
}

interface TodoStore {
  todos: Todo[];
  filter: 'all' | 'active' | 'completed';
  
  // アクション
  addTodo: (text: string) => void;
  toggleTodo: (id: string) => void;
  deleteTodo: (id: string) => void;
  setFilter: (filter: 'all' | 'active' | 'completed') => void;
  clearCompleted: () => void;
  
  // セレクタ
  filteredTodos: () => Todo[];
  completedCount: () => number;
  activeCount: () => number;
}

export const useTodoStore = create<TodoStore>()(
  devtools(
    (set, get) => ({
      todos: [],
      filter: 'all',
      
      addTodo: (text) => {
        const newTodo: Todo = {
          id: crypto.randomUUID(),
          text,
          completed: false,
          createdAt: new Date()
        };
        set((state) => ({
          todos: [...state.todos, newTodo]
        }));
      },
      
      toggleTodo: (id) => {
        set((state) => ({
          todos: state.todos.map(todo =>
            todo.id === id ? { ...todo, completed: !todo.completed } : todo
          )
        }));
      },
      
      deleteTodo: (id) => {
        set((state) => ({
          todos: state.todos.filter(todo => todo.id !== id)
        }));
      },
      
      setFilter: (filter) => set({ filter }),
      
      clearCompleted: () => {
        set((state) => ({
          todos: state.todos.filter(todo => !todo.completed)
        }));
      },
      
      filteredTodos: () => {
        const { todos, filter } = get();
        switch (filter) {
          case 'active':
            return todos.filter(todo => !todo.completed);
          case 'completed':
            return todos.filter(todo => todo.completed);
          default:
            return todos;
        }
      },
      
      completedCount: () => get().todos.filter(todo => todo.completed).length,
      activeCount: () => get().todos.filter(todo => !todo.completed).length,
    })
  )
);
```

## React Query / TanStack Query（サーバー状態管理）

### 基本的なデータフェッチング
```tsx
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// ユーザー情報取得
const useUser = (userId: string) => {
  return useQuery({
    queryKey: ['user', userId],
    queryFn: () => api.getUser(userId),
    enabled: !!userId,
    staleTime: 5 * 60 * 1000, // 5分間はフレッシュとみなす
    cacheTime: 10 * 60 * 1000, // 10分間キャッシュを保持
  });
};

// ユーザー一覧取得（ページネーション付き）
const useUsers = (page: number, limit: number) => {
  return useQuery({
    queryKey: ['users', page, limit],
    queryFn: () => api.getUsers({ page, limit }),
    keepPreviousData: true, // ページ切り替え時に前のデータを保持
  });
};

// 無限スクロール
const useInfiniteUsers = () => {
  return useInfiniteQuery({
    queryKey: ['users', 'infinite'],
    queryFn: ({ pageParam = 1 }) => api.getUsers({ page: pageParam, limit: 20 }),
    getNextPageParam: (lastPage, pages) => {
      return lastPage.hasMore ? pages.length + 1 : undefined;
    },
  });
};

// ミューテーション（データ更新）
const useCreateUser = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: (userData: CreateUserRequest) => api.createUser(userData),
    onSuccess: (newUser) => {
      // ユーザー一覧のキャッシュを無効化
      queryClient.invalidateQueries({ queryKey: ['users'] });
      
      // 楽観的更新
      queryClient.setQueryData(['user', newUser.id], newUser);
    },
    onError: (error) => {
      // エラー処理
      toast.error('ユーザーの作成に失敗しました');
    },
  });
};

// 楽観的更新
const useUpdateUser = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: ({ id, data }: { id: string; data: UpdateUserRequest }) => 
      api.updateUser(id, data),
    onMutate: async ({ id, data }) => {
      // 進行中のクエリをキャンセル
      await queryClient.cancelQueries({ queryKey: ['user', id] });
      
      // 現在のデータを保存
      const previousUser = queryClient.getQueryData(['user', id]);
      
      // 楽観的更新
      queryClient.setQueryData(['user', id], (old: User) => ({
        ...old,
        ...data
      }));
      
      return { previousUser };
    },
    onError: (err, variables, context) => {
      // エラー時に前のデータに戻す
      if (context?.previousUser) {
        queryClient.setQueryData(['user', variables.id], context.previousUser);
      }
    },
    onSettled: (data, error, variables) => {
      // 完了時にクエリを再フェッチ
      queryClient.invalidateQueries({ queryKey: ['user', variables.id] });
    },
  });
};
```

### カスタムフック例
```tsx
// リアルタイム更新
const useRealtimeData = (endpoint: string) => {
  const queryClient = useQueryClient();
  
  useEffect(() => {
    const eventSource = new EventSource(endpoint);
    
    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data);
      queryClient.setQueryData(['realtime', endpoint], data);
    };
    
    return () => eventSource.close();
  }, [endpoint, queryClient]);
  
  return useQuery({
    queryKey: ['realtime', endpoint],
    queryFn: () => api.get(endpoint),
    refetchInterval: 30000, // フォールバック用の定期更新
  });
};

// 依存関係のあるクエリ
const useUserWithPosts = (userId: string) => {
  const userQuery = useUser(userId);
  
  const postsQuery = useQuery({
    queryKey: ['posts', 'user', userId],
    queryFn: () => api.getUserPosts(userId),
    enabled: !!userQuery.data,
  });
  
  return {
    user: userQuery.data,
    posts: postsQuery.data,
    isLoading: userQuery.isLoading || postsQuery.isLoading,
    error: userQuery.error || postsQuery.error,
  };
};
```

## Redux Toolkit（複雑なアプリケーション状態）

### スライス定義
```tsx
import { createSlice, createAsyncThunk, PayloadAction } from '@reduxjs/toolkit';

// 非同期アクション
export const fetchUsers = createAsyncThunk(
  'users/fetchUsers',
  async ({ page, limit }: { page: number; limit: number }) => {
    const response = await api.getUsers(page, limit);
    return response;
  }
);

export const createUser = createAsyncThunk(
  'users/createUser',
  async (userData: CreateUserRequest, { rejectWithValue }) => {
    try {
      const response = await api.createUser(userData);
      return response;
    } catch (error) {
      return rejectWithValue(error.response.data);
    }
  }
);

// 状態の型定義
interface UsersState {
  users: User[];
  loading: boolean;
  error: string | null;
  currentPage: number;
  totalPages: number;
  filters: {
    search: string;
    role: string | null;
  };
}

const initialState: UsersState = {
  users: [],
  loading: false,
  error: null,
  currentPage: 1,
  totalPages: 1,
  filters: {
    search: '',
    role: null,
  },
};

// スライス作成
const usersSlice = createSlice({
  name: 'users',
  initialState,
  reducers: {
    setFilters: (state, action: PayloadAction<Partial<UsersState['filters']>>) => {
      state.filters = { ...state.filters, ...action.payload };
    },
    clearFilters: (state) => {
      state.filters = initialState.filters;
    },
    updateUser: (state, action: PayloadAction<{ id: string; data: Partial<User> }>) => {
      const index = state.users.findIndex(user => user.id === action.payload.id);
      if (index !== -1) {
        state.users[index] = { ...state.users[index], ...action.payload.data };
      }
    },
    removeUser: (state, action: PayloadAction<string>) => {
      state.users = state.users.filter(user => user.id !== action.payload);
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending, (state) => {
        state.loading = true;
        state.error = null;
      })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.loading = false;
        state.users = action.payload.users;
        state.currentPage = action.payload.page;
        state.totalPages = action.payload.totalPages;
      })
      .addCase(fetchUsers.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message || 'ユーザーの取得に失敗しました';
      })
      .addCase(createUser.fulfilled, (state, action) => {
        state.users.push(action.payload);
      });
  },
});

export const { setFilters, clearFilters, updateUser, removeUser } = usersSlice.actions;
export default usersSlice.reducer;

// セレクタ
export const selectUsers = (state: RootState) => state.users.users;
export const selectUsersLoading = (state: RootState) => state.users.loading;
export const selectUsersError = (state: RootState) => state.users.error;
export const selectFilteredUsers = createSelector(
  [selectUsers, (state: RootState) => state.users.filters],
  (users, filters) => {
    return users.filter(user => {
      const matchesSearch = filters.search === '' || 
        user.name.toLowerCase().includes(filters.search.toLowerCase());
      const matchesRole = filters.role === null || user.role === filters.role;
      return matchesSearch && matchesRole;
    });
  }
);
```

### ストア設定
```tsx
import { configureStore } from '@reduxjs/toolkit';
import { useDispatch, useSelector, TypedUseSelectorHook } from 'react-redux';

import usersReducer from './slices/usersSlice';
import authReducer from './slices/authSlice';

export const store = configureStore({
  reducer: {
    users: usersReducer,
    auth: authReducer,
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware({
      serializableCheck: {
        ignoredActions: ['persist/PERSIST'],
      },
    }),
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;

// 型付きフック
export const useAppDispatch = () => useDispatch<AppDispatch>();
export const useAppSelector: TypedUseSelectorHook<RootState> = useSelector;
```

## 状態管理の選択指針

### 使い分けの基準
```
ローカル状態（コンポーネント内）
├─ useState: シンプルな値の管理
└─ useReducer: 複雑な状態遷移の管理

グローバル状態（アプリケーション全体）
├─ React Context: 
│  ├─ テーマ、言語、認証状態など
│  └─ 頻繁に変更されない設定値
├─ Zustand:
│  ├─ 軽量なグローバル状態
│  └─ 中規模アプリケーション
└─ Redux Toolkit:
   ├─ 複雑な状態管理が必要
   └─ 大規模アプリケーション

サーバー状態
├─ React Query/SWR:
│  ├─ API データのキャッシュ・同期
│  ├─ バックグラウンド更新
│  └─ 楽観的更新
└─ Apollo Client:
   └─ GraphQL API との統合
```

### パフォーマンス最適化
```tsx
// 状態の分割
const useOptimizedState = () => {
  // 頻繁に変更される状態を分離
  const [fastChangingState, setFastChangingState] = useState(0);
  
  // 安定した状態は別管理
  const [stableState, setStableState] = useState({
    config: {},
    userPreferences: {}
  });
  
  return {
    fastChangingState,
    setFastChangingState,
    stableState,
    setStableState
  };
};

// メモ化によるレンダリング最適化
const OptimizedComponent = React.memo<{
  data: ComplexData;
  onUpdate: (data: ComplexData) => void;
}>(({ data, onUpdate }) => {
  const handleUpdate = useCallback((newData: Partial<ComplexData>) => {
    onUpdate({ ...data, ...newData });
  }, [data, onUpdate]);
  
  return (
    <div>
      {/* レンダリング内容 */}
    </div>
  );
});
```