# パフォーマンス最適化ガイド

## フロントエンドパフォーマンス

### Core Web Vitals 最適化

```typescript
// Largest Contentful Paint (LCP) 最適化
// 画像最適化
const OptimizedImage: React.FC<{ src: string; alt: string }> = ({ src, alt }) => {
  return (
    <picture>
      <source srcSet={`${src}.webp`} type="image/webp" />
      <source srcSet={`${src}.avif`} type="image/avif" />
      <img
        src={src}
        alt={alt}
        loading="lazy"
        decoding="async"
        style={{ aspectRatio: '16/9' }}
      />
    </picture>
  );
};

// リソースプリロード
const PreloadCriticalResources = () => {
  useEffect(() => {
    // 重要なフォントのプリロード
    const fontLink = document.createElement('link');
    fontLink.rel = 'preload';
    fontLink.href = '/fonts/main-font.woff2';
    fontLink.as = 'font';
    fontLink.type = 'font/woff2';
    fontLink.crossOrigin = 'anonymous';
    document.head.appendChild(fontLink);

    // 重要な画像のプリロード
    const imageLink = document.createElement('link');
    imageLink.rel = 'preload';
    imageLink.href = '/images/hero-image.webp';
    imageLink.as = 'image';
    document.head.appendChild(imageLink);
  }, []);

  return null;
};
```

### バンドル最適化

```typescript
// 動的インポート（Code Splitting）
const LazyComponent = lazy(() => import('./HeavyComponent'));
const LazyRoute = lazy(() => import('../pages/AdminPage'));

// React Router での実装
const App = () => (
  <Router>
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/" element={<HomePage />} />
        <Route path="/admin" element={<LazyRoute />} />
      </Routes>
    </Suspense>
  </Router>
);

// Webpack Bundle Analyzer での分析
// webpack.config.js
const BundleAnalyzerPlugin = require('webpack-bundle-analyzer').BundleAnalyzerPlugin;

module.exports = {
  plugins: [
    new BundleAnalyzerPlugin({
      analyzerMode: 'static',
      openAnalyzer: false,
    }),
  ],
  optimization: {
    splitChunks: {
      chunks: 'all',
      cacheGroups: {
        vendor: {
          test: /[\\/]node_modules[\\/]/,
          name: 'vendors',
          chunks: 'all',
        },
        common: {
          name: 'common',
          minChunks: 2,
          chunks: 'all',
          enforce: true,
        },
      },
    },
  },
};
```

### レンダリング最適化

```typescript
// メモ化の活用
const ExpensiveComponent = React.memo<{
  data: ComplexData[];
  onItemClick: (id: string) => void;
}>(({ data, onItemClick }) => {
  const sortedData = useMemo(() => {
    return data.sort((a, b) => a.priority - b.priority);
  }, [data]);

  const handleItemClick = useCallback((id: string) => {
    onItemClick(id);
  }, [onItemClick]);

  return (
    <div>
      {sortedData.map(item => (
        <ItemComponent
          key={item.id}
          item={item}
          onClick={handleItemClick}
        />
      ))}
    </div>
  );
});

// 仮想スクロール
import { FixedSizeList as List } from 'react-window';

const VirtualizedList: React.FC<{ items: Item[] }> = ({ items }) => {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      <ItemComponent item={items[index]} />
    </div>
  );

  return (
    <List
      height={600}
      itemCount={items.length}
      itemSize={80}
      width="100%"
    >
      {Row}
    </List>
  );
};

// Intersection Observer による遅延読み込み
const useLazyLoad = (threshold = 0.1) => {
  const [isVisible, setIsVisible] = useState(false);
  const ref = useRef<HTMLDivElement>(null);

  useEffect(() => {
    const observer = new IntersectionObserver(
      ([entry]) => {
        if (entry.isIntersecting) {
          setIsVisible(true);
          observer.disconnect();
        }
      },
      { threshold }
    );

    if (ref.current) {
      observer.observe(ref.current);
    }

    return () => observer.disconnect();
  }, [threshold]);

  return [ref, isVisible] as const;
};
```

## バックエンドパフォーマンス

### データベース最適化

```sql
-- インデックス戦略
-- 単一カラムインデックス
CREATE INDEX idx_users_email ON users(email);

-- 複合インデックス（順序が重要）
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);

-- 部分インデックス
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- カバリングインデックス
CREATE INDEX idx_user_profile_covering ON users(id) INCLUDE (name, email, created_at);

-- クエリ最適化
-- 悪い例：N+1問題
SELECT * FROM users;
-- 各ユーザーに対して
SELECT * FROM orders WHERE user_id = ?;

-- 良い例：JOIN使用
SELECT u.*, o.*
FROM users u
LEFT JOIN orders o ON u.id = o.user_id;

-- 良い例：サブクエリでの制限
SELECT u.*,
       (SELECT COUNT(*) FROM orders WHERE user_id = u.id) as order_count
FROM users u
WHERE u.is_active = true;
```

### キャッシュ戦略

```typescript
// Redis でのキャッシュ実装
import Redis from 'ioredis'

class CacheService {
  private redis: Redis

  constructor() {
    this.redis = new Redis(process.env.REDIS_URL)
  }

  async get<T>(key: string): Promise<T | null> {
    const cached = await this.redis.get(key)
    return cached ? JSON.parse(cached) : null
  }

  async set<T>(key: string, value: T, ttl = 3600): Promise<void> {
    await this.redis.setex(key, ttl, JSON.stringify(value))
  }

  async invalidate(pattern: string): Promise<void> {
    const keys = await this.redis.keys(pattern)
    if (keys.length > 0) {
      await this.redis.del(...keys)
    }
  }
}

// キャッシュ装飾子
function Cache(ttl = 3600) {
  return function (target: any, propertyName: string, descriptor: PropertyDescriptor) {
    const method = descriptor.value

    descriptor.value = async function (...args: any[]) {
      const cacheKey = `${target.constructor.name}:${propertyName}:${JSON.stringify(args)}`

      const cached = await cacheService.get(cacheKey)
      if (cached) {
        return cached
      }

      const result = await method.apply(this, args)
      await cacheService.set(cacheKey, result, ttl)

      return result
    }
  }
}

// 使用例
class UserService {
  @Cache(1800) // 30分間キャッシュ
  async getUserProfile(userId: string): Promise<UserProfile> {
    return await this.userRepository.findById(userId)
  }
}
```

### API最適化

```typescript
// ページネーション実装
interface PaginationOptions {
  page: number
  limit: number
  sortBy?: string
  sortOrder?: 'asc' | 'desc'
}

interface PaginatedResponse<T> {
  data: T[]
  pagination: {
    page: number
    limit: number
    total: number
    totalPages: number
    hasNext: boolean
    hasPrev: boolean
  }
}

class PaginationService {
  static async paginate<T>(query: any, options: PaginationOptions): Promise<PaginatedResponse<T>> {
    const { page, limit, sortBy = 'id', sortOrder = 'asc' } = options
    const offset = (page - 1) * limit

    const [data, total] = await Promise.all([
      query.orderBy(sortBy, sortOrder).limit(limit).offset(offset),
      query.clone().count('* as count').first(),
    ])

    const totalPages = Math.ceil(total.count / limit)

    return {
      data,
      pagination: {
        page,
        limit,
        total: total.count,
        totalPages,
        hasNext: page < totalPages,
        hasPrev: page > 1,
      },
    }
  }
}

// GraphQL DataLoader でのN+1問題解決
import DataLoader from 'dataloader'

class UserLoader {
  private userLoader = new DataLoader(async (userIds: readonly string[]) => {
    const users = await User.findByIds([...userIds])
    const userMap = new Map(users.map(user => [user.id, user]))
    return userIds.map(id => userMap.get(id))
  })

  async loadUser(id: string): Promise<User | undefined> {
    return this.userLoader.load(id)
  }

  async loadUsers(ids: string[]): Promise<(User | undefined)[]> {
    return this.userLoader.loadMany(ids)
  }
}
```

## メモリ管理

### メモリリーク対策

```typescript
// イベントリスナーのクリーンアップ
useEffect(() => {
  const handleScroll = () => {
    // スクロール処理
  }

  const handleResize = debounce(() => {
    // リサイズ処理
  }, 100)

  window.addEventListener('scroll', handleScroll)
  window.addEventListener('resize', handleResize)

  return () => {
    window.removeEventListener('scroll', handleScroll)
    window.removeEventListener('resize', handleResize)
  }
}, [])

// WeakMap を使った弱参照
class ComponentRegistry {
  private registry = new WeakMap<HTMLElement, ComponentInstance>()

  register(element: HTMLElement, component: ComponentInstance) {
    this.registry.set(element, component)
  }

  get(element: HTMLElement): ComponentInstance | undefined {
    return this.registry.get(element)
  }

  // elementが削除されると自動的にエントリも削除される
}

// メモリプロファイリング
const profileMemory = () => {
  if ('memory' in performance) {
    const memory = (performance as any).memory
    console.log({
      usedJSMemory: `${Math.round(memory.usedJSMemory / 1048576)} MB`,
      totalJSMemory: `${Math.round(memory.totalJSMemory / 1048576)} MB`,
      jsMemoryLimit: `${Math.round(memory.jsMemoryLimit / 1048576)} MB`,
    })
  }
}
```

### オブジェクトプール

```typescript
// オブジェクトプールパターン
class ObjectPool<T> {
  private pool: T[] = []
  private factory: () => T
  private reset: (obj: T) => void

  constructor(factory: () => T, reset: (obj: T) => void, initialSize = 10) {
    this.factory = factory
    this.reset = reset

    // 初期オブジェクトの作成
    for (let i = 0; i < initialSize; i++) {
      this.pool.push(factory())
    }
  }

  acquire(): T {
    if (this.pool.length > 0) {
      return this.pool.pop()!
    }
    return this.factory()
  }

  release(obj: T): void {
    this.reset(obj)
    this.pool.push(obj)
  }
}

// 使用例：DOM要素プール
const divPool = new ObjectPool(
  () => document.createElement('div'),
  div => {
    div.innerHTML = ''
    div.className = ''
    div.removeAttribute('style')
  }
)

const createListItem = (text: string) => {
  const div = divPool.acquire()
  div.textContent = text
  div.className = 'list-item'
  return div
}

const removeListItem = (div: HTMLElement) => {
  div.remove()
  divPool.release(div)
}
```

## ネットワーク最適化

### HTTP/2 と HTTP/3 最適化

```typescript
// Server Push の実装（HTTP/2）
app.get('/', (req, res) => {
  // 重要なリソースをプッシュ
  res.push('/css/critical.css', {
    response: {
      'content-type': 'text/css',
    },
  })

  res.push('/js/main.js', {
    response: {
      'content-type': 'application/javascript',
    },
  })

  res.render('index')
})

// リソースヒント
const ResourceHints: React.FC = () => {
  useEffect(() => {
    // DNS prefetch
    const dnsPrefetch = document.createElement('link')
    dnsPrefetch.rel = 'dns-prefetch'
    dnsPrefetch.href = '//api.example.com'
    document.head.appendChild(dnsPrefetch)

    // Preconnect
    const preconnect = document.createElement('link')
    preconnect.rel = 'preconnect'
    preconnect.href = 'https://fonts.googleapis.com'
    preconnect.crossOrigin = 'anonymous'
    document.head.appendChild(preconnect)
  }, [])

  return null
}
```

### 圧縮とminification

```javascript
// Webpack 圧縮設定
const CompressionPlugin = require('compression-webpack-plugin')
const TerserPlugin = require('terser-webpack-plugin')

module.exports = {
  optimization: {
    minimize: true,
    minimizer: [
      new TerserPlugin({
        terserOptions: {
          compress: {
            drop_console: process.env.NODE_ENV === 'production',
            drop_debugger: true,
          },
        },
      }),
    ],
  },
  plugins: [
    new CompressionPlugin({
      algorithm: 'brotliCompress',
      test: /\.(js|css|html|svg)$/,
      compressionOptions: { level: 11 },
      threshold: 8192,
      minRatio: 0.8,
    }),
  ],
}

// Express での圧縮
const compression = require('compression')

app.use(
  compression({
    filter: (req, res) => {
      if (req.headers['x-no-compression']) {
        return false
      }
      return compression.filter(req, res)
    },
    level: 6,
    threshold: 1024,
  })
)
```

## パフォーマンス監視

### Real User Monitoring (RUM)

```typescript
// Web Vitals 計測
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals'

const vitalsThresholds = {
  CLS: { good: 0.1, needs_improvement: 0.25 },
  FID: { good: 100, needs_improvement: 300 },
  FCP: { good: 1800, needs_improvement: 3000 },
  LCP: { good: 2500, needs_improvement: 4000 },
  TTFB: { good: 800, needs_improvement: 1800 },
}

const sendToAnalytics = (metric: any) => {
  const threshold = vitalsThresholds[metric.name as keyof typeof vitalsThresholds]
  const rating =
    metric.value <= threshold.good ? 'good' : metric.value <= threshold.needs_improvement ? 'needs_improvement' : 'poor'

  // Google Analytics への送信
  gtag('event', metric.name, {
    event_category: 'Web Vitals',
    event_label: metric.id,
    value: Math.round(metric.name === 'CLS' ? metric.value * 1000 : metric.value),
    custom_map: { rating },
  })

  // カスタム分析サービスへの送信
  fetch('/api/analytics/vitals', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      metric: metric.name,
      value: metric.value,
      rating,
      url: window.location.href,
      timestamp: Date.now(),
    }),
  })
}

// メトリクス収集の開始
getCLS(sendToAnalytics)
getFID(sendToAnalytics)
getFCP(sendToAnalytics)
getLCP(sendToAnalytics)
getTTFB(sendToAnalytics)
```

### パフォーマンスプロファイリング

```typescript
// カスタムメトリクス計測
class PerformanceTracker {
  private marks = new Map<string, number>()

  startTiming(name: string): void {
    this.marks.set(`${name}-start`, performance.now())
    performance.mark(`${name}-start`)
  }

  endTiming(name: string): number {
    const endTime = performance.now()
    const startTime = this.marks.get(`${name}-start`)

    if (!startTime) {
      console.warn(`No start time found for ${name}`)
      return 0
    }

    const duration = endTime - startTime
    performance.mark(`${name}-end`)
    performance.measure(name, `${name}-start`, `${name}-end`)

    this.marks.delete(`${name}-start`)
    return duration
  }

  getPerformanceEntries(): PerformanceEntry[] {
    return performance.getEntriesByType('measure')
  }
}

// 使用例
const tracker = new PerformanceTracker()

tracker.startTiming('api-call')
const data = await fetchUserData()
const apiDuration = tracker.endTiming('api-call')

tracker.startTiming('rendering')
renderComponent(data)
const renderDuration = tracker.endTiming('rendering')

console.log(`API call: ${apiDuration}ms, Rendering: ${renderDuration}ms`)
```

## 負荷テスト

### Artillery.js での負荷テスト

```yaml
# load-test.yml
config:
  target: 'https://api.example.com'
  phases:
    - duration: 60
      arrivalRate: 10
      name: 'Warm up'
    - duration: 120
      arrivalRate: 50
      name: 'Sustained load'
    - duration: 60
      arrivalRate: 100
      name: 'Peak load'
  variables:
    userIds:
      - '123'
      - '456'
      - '789'

scenarios:
  - name: 'User authentication flow'
    weight: 30
    flow:
      - post:
          url: '/auth/login'
          json:
            email: 'test@example.com'
            password: 'password123'
          capture:
            - json: '$.token'
              as: 'authToken'
      - get:
          url: '/user/profile'
          headers:
            Authorization: 'Bearer {{ authToken }}'

  - name: 'API endpoints'
    weight: 70
    flow:
      - get:
          url: '/api/users/{{ $randomPick(userIds) }}'
      - post:
          url: '/api/data'
          json:
            type: 'test'
            value: '{{ $randomInt(1, 100) }}'
```

### K6 での負荷テスト

```javascript
// load-test.js
import http from 'k6/http'
import { check, sleep } from 'k6'
import { Rate } from 'k6/metrics'

const errorRate = new Rate('errors')

export const options = {
  stages: [
    { duration: '2m', target: 10 },
    { duration: '5m', target: 10 },
    { duration: '2m', target: 20 },
    { duration: '5m', target: 20 },
    { duration: '2m', target: 0 },
  ],
  thresholds: {
    http_req_duration: ['p(95)<2000'],
    errors: ['rate<0.1'],
  },
}

export default function () {
  const response = http.get('https://api.example.com/users')

  const result = check(response, {
    'status is 200': r => r.status === 200,
    'response time < 500ms': r => r.timings.duration < 500,
  })

  errorRate.add(!result)
  sleep(1)
}
```

## 最適化のベストプラクティス

### パフォーマンス予算

```typescript
// パフォーマンス予算の定義
const performanceBudget = {
  // ネットワーク
  totalSize: 2000, // KB
  imageSize: 1000, // KB
  jsSize: 400, // KB
  cssSize: 200, // KB

  // メトリクス
  LCP: 2500, // ms
  FID: 100, // ms
  CLS: 0.1, // 単位なし

  // リソース数
  totalRequests: 50,
  domElements: 1500,
}

// 予算チェック
const checkPerformanceBudget = async () => {
  const metrics = await getPerformanceMetrics()
  const budget = performanceBudget

  const violations = []

  if (metrics.totalSize > budget.totalSize) {
    violations.push(`Total size: ${metrics.totalSize}KB > ${budget.totalSize}KB`)
  }

  if (metrics.LCP > budget.LCP) {
    violations.push(`LCP: ${metrics.LCP}ms > ${budget.LCP}ms`)
  }

  if (violations.length > 0) {
    console.error('Performance budget violations:', violations)
    throw new Error('Performance budget exceeded')
  }
}
```

### 継続的な最適化

```typescript
// パフォーマンス回帰検出
const performanceRegression = {
  baseline: {
    LCP: 2000,
    FID: 80,
    CLS: 0.05,
  },
  threshold: 0.1, // 10% の悪化で警告

  check(current: any) {
    const regressions = []

    Object.entries(this.baseline).forEach(([metric, baseline]) => {
      const currentValue = current[metric]
      const regression = (currentValue - baseline) / baseline

      if (regression > this.threshold) {
        regressions.push({
          metric,
          baseline,
          current: currentValue,
          regression: Math.round(regression * 100),
        })
      }
    })

    return regressions
  },
}
```
