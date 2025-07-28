# フロントエンドパフォーマンス最適化

## パフォーマンス測定・監視

### Core Web Vitals
```typescript
// Web Vitals の測定
import { getCLS, getFID, getFCP, getLCP, getTTFB } from 'web-vitals';

const sendToAnalytics = (metric: Metric) => {
  // Google Analytics 4 に送信
  gtag('event', metric.name, {
    event_category: 'Web Vitals',
    value: Math.round(metric.name === 'CLS' ? metric.value * 1000 : metric.value),
    event_label: metric.id,
    non_interaction: true,
  });
  
  // カスタム分析サービスに送信
  analytics.track('performance', {
    metric: metric.name,
    value: metric.value,
    rating: metric.rating,
    delta: metric.delta,
    id: metric.id,
    url: window.location.href,
    userAgent: navigator.userAgent,
  });
};

// メトリクス収集の開始
getCLS(sendToAnalytics);
getFID(sendToAnalytics);
getFCP(sendToAnalytics);
getLCP(sendToAnalytics);
getTTFB(sendToAnalytics);

// カスタムパフォーマンス測定
const measureCustomMetric = (name: string, fn: () => void) => {
  const start = performance.now();
  fn();
  const end = performance.now();
  
  // Performance Timeline API を使用
  performance.mark(`${name}-start`);
  performance.mark(`${name}-end`);
  performance.measure(name, `${name}-start`, `${name}-end`);
  
  console.log(`${name}: ${end - start}ms`);
};

// React の Profiler API
const ProfilerWrapper: React.FC<{ id: string; children: React.ReactNode }> = ({ 
  id, 
  children 
}) => (
  <Profiler
    id={id}
    onRender={(id, phase, actualDuration, baseDuration, startTime, commitTime) => {
      // パフォーマンス情報をログ
      console.log('Profiler:', {
        id,
        phase,
        actualDuration,
        baseDuration,
        startTime,
        commitTime,
      });
      
      // 分析サービスに送信
      if (actualDuration > 50) { // 50ms以上のレンダリングをトラッキング
        analytics.track('slow-render', {
          componentId: id,
          phase,
          duration: actualDuration,
          url: window.location.href,
        });
      }
    }}
  >
    {children}
  </Profiler>
);
```

### パフォーマンス監視ダッシュボード
```typescript
// リアルタイム パフォーマンス監視
class PerformanceMonitor {
  private metrics: Map<string, number[]> = new Map();
  private observer: PerformanceObserver;
  
  constructor() {
    this.observer = new PerformanceObserver((list) => {
      for (const entry of list.getEntries()) {
        this.recordMetric(entry.name, entry.duration);
      }
    });
    
    this.observer.observe({ entryTypes: ['measure', 'navigation', 'paint'] });
  }
  
  recordMetric(name: string, value: number) {
    if (!this.metrics.has(name)) {
      this.metrics.set(name, []);
    }
    this.metrics.get(name)!.push(value);
    
    // 異常値の検出
    if (this.isAnomalous(name, value)) {
      this.reportAnomaly(name, value);
    }
  }
  
  private isAnomalous(name: string, value: number): boolean {
    const values = this.metrics.get(name) || [];
    if (values.length < 10) return false;
    
    const avg = values.reduce((a, b) => a + b, 0) / values.length;
    const threshold = avg * 2; // 平均の2倍を異常値とする
    
    return value > threshold;
  }
  
  private reportAnomaly(name: string, value: number) {
    console.warn(`Performance anomaly detected: ${name} = ${value}ms`);
    
    // アラート送信
    fetch('/api/performance/alert', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        metric: name,
        value,
        timestamp: Date.now(),
        url: window.location.href,
        userAgent: navigator.userAgent,
      }),
    });
  }
  
  getMetrics() {
    const summary: Record<string, { avg: number; max: number; min: number }> = {};
    
    for (const [name, values] of this.metrics.entries()) {
      if (values.length > 0) {
        summary[name] = {
          avg: values.reduce((a, b) => a + b, 0) / values.length,
          max: Math.max(...values),
          min: Math.min(...values),
        };
      }
    }
    
    return summary;
  }
}

const performanceMonitor = new PerformanceMonitor();

// 使用例
const App: React.FC = () => {
  useEffect(() => {
    // 定期的にメトリクスを送信
    const interval = setInterval(() => {
      const metrics = performanceMonitor.getMetrics();
      analytics.track('performance-summary', metrics);
    }, 60000); // 1分ごと
    
    return () => clearInterval(interval);
  }, []);
  
  return (
    <ProfilerWrapper id="App">
      <Router>
        <Routes>
          {/* ルート定義 */}
        </Routes>
      </Router>
    </ProfilerWrapper>
  );
};
```

## バンドル最適化

### コード分割（Code Splitting）
```typescript
// ルートレベルでの分割
const Dashboard = React.lazy(() => import('./pages/Dashboard'));
const UserManagement = React.lazy(() => import('./pages/UserManagement'));
const Settings = React.lazy(() => import('./pages/Settings'));

const App: React.FC = () => (
  <Suspense fallback={<div>読み込み中...</div>}>
    <Routes>
      <Route path="/dashboard" element={<Dashboard />} />
      <Route path="/users" element={<UserManagement />} />
      <Route path="/settings" element={<Settings />} />
    </Routes>
  </Suspense>
);

// コンポーネントレベルでの分割
const HeavyChart = React.lazy(() => 
  import('./components/HeavyChart').then(module => ({
    default: module.HeavyChart
  }))
);

const DataVisualization: React.FC = () => {
  const [showChart, setShowChart] = useState(false);
  
  return (
    <div>
      <button onClick={() => setShowChart(true)}>
        チャートを表示
      </button>
      {showChart && (
        <Suspense fallback={<ChartSkeleton />}>
          <HeavyChart />
        </Suspense>
      )}
    </div>
  );
};

// 動的インポート（条件付き読み込み）
const ConditionalImport: React.FC = () => {
  const handleFeatureActivation = async () => {
    if (userHasFeatureAccess) {
      const { AdvancedFeature } = await import('./AdvancedFeature');
      // 機能を使用
    }
  };
  
  return <button onClick={handleFeatureActivation}>高度な機能</button>;
};

// ライブラリの動的インポート
const DatePicker: React.FC = () => {
  const [DatePickerLib, setDatePickerLib] = useState<any>(null);
  
  useEffect(() => {
    const loadDatePicker = async () => {
      const { default: ReactDatePicker } = await import('react-datepicker');
      await import('react-datepicker/dist/react-datepicker.css');
      setDatePickerLib(() => ReactDatePicker);
    };
    
    loadDatePicker();
  }, []);
  
  if (!DatePickerLib) return <div>読み込み中...</div>;
  
  return <DatePickerLib selected={new Date()} onChange={() => {}} />;
};
```

### Tree Shaking最適化
```typescript
// ❌ 悪い例：ライブラリ全体をインポート
import * as _ from 'lodash';
import { Button, Input, Select, Table } from 'antd';

// ✅ 良い例：必要な関数のみインポート
import debounce from 'lodash/debounce';
import isEqual from 'lodash/isEqual';

// ES6 modules の適切な使用
export { Button } from './Button';
export { Input } from './Input';
export { Select } from './Select';

// Instead of
export * from './components'; // すべてバンドルに含まれる

// webpack での最適化設定
// webpack.config.js
module.exports = {
  optimization: {
    usedExports: true,
    sideEffects: false, // package.json で指定することも可能
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

// Vite での最適化
// vite.config.ts
export default defineConfig({
  build: {
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          ui: ['@mui/material', '@mui/icons-material'],
          utils: ['lodash', 'date-fns'],
        },
      },
    },
  },
});
```

### バンドルサイズ分析
```typescript
// webpack-bundle-analyzer の使用
// package.json
{
  "scripts": {
    "analyze": "npm run build && npx webpack-bundle-analyzer build/static/js/*.js"
  }
}

// バンドルサイズの監視
const BUNDLE_SIZE_LIMITS = {
  main: 250 * 1024, // 250KB
  vendor: 500 * 1024, // 500KB
  chunk: 100 * 1024, // 100KB
};

// CI/CD でのサイズチェック
const checkBundleSize = () => {
  const buildDir = './dist';
  const files = fs.readdirSync(buildDir);
  
  for (const file of files) {
    if (file.endsWith('.js')) {
      const filePath = path.join(buildDir, file);
      const stats = fs.statSync(filePath);
      const sizeInBytes = stats.size;
      
      let limit;
      if (file.includes('vendor')) {
        limit = BUNDLE_SIZE_LIMITS.vendor;
      } else if (file.includes('main')) {
        limit = BUNDLE_SIZE_LIMITS.main;
      } else {
        limit = BUNDLE_SIZE_LIMITS.chunk;
      }
      
      if (sizeInBytes > limit) {
        console.error(`Bundle size exceeded: ${file} (${sizeInBytes} > ${limit})`);
        process.exit(1);
      }
    }
  }
};
```

## レンダリング最適化

### React最適化パターン
```typescript
// React.memo での不要な再レンダリング防止
const ExpensiveComponent = React.memo<{
  data: ComplexData;
  onUpdate: (data: ComplexData) => void;
}>(({ data, onUpdate }) => {
  console.log('ExpensiveComponent rendering');
  
  return (
    <div>
      {/* 複雑なレンダリング処理 */}
      <ComplexVisualization data={data} />
      <button onClick={() => onUpdate(data)}>更新</button>
    </div>
  );
}, (prevProps, nextProps) => {
  // カスタム比較関数
  return isEqual(prevProps.data, nextProps.data) &&
         prevProps.onUpdate === nextProps.onUpdate;
});

// useMemo での計算結果のメモ化
const DataProcessor: React.FC<{ rawData: RawData[] }> = ({ rawData }) => {
  // 重い計算処理をメモ化
  const processedData = useMemo(() => {
    console.log('Processing data...');
    return rawData
      .filter(item => item.isValid)
      .map(item => ({
        ...item,
        computed: heavyComputation(item),
        formatted: formatData(item),
      }))
      .sort((a, b) => b.priority - a.priority);
  }, [rawData]);
  
  // 派生データもメモ化
  const statistics = useMemo(() => ({
    total: processedData.length,
    average: processedData.reduce((sum, item) => sum + item.value, 0) / processedData.length,
    max: Math.max(...processedData.map(item => item.value)),
    min: Math.min(...processedData.map(item => item.value)),
  }), [processedData]);
  
  return (
    <div>
      <DataStatistics stats={statistics} />
      <DataTable data={processedData} />
    </div>
  );
};

// useCallback でのイベントハンドラーメモ化
const OptimizedList: React.FC<{ items: Item[] }> = ({ items }) => {
  const [selectedItems, setSelectedItems] = useState<Set<string>>(new Set());
  
  const handleItemSelect = useCallback((itemId: string) => {
    setSelectedItems(prev => {
      const newSet = new Set(prev);
      if (newSet.has(itemId)) {
        newSet.delete(itemId);
      } else {
        newSet.add(itemId);
      }
      return newSet;
    });
  }, []);
  
  const handleSelectAll = useCallback(() => {
    setSelectedItems(new Set(items.map(item => item.id)));
  }, [items]);
  
  const handleClearSelection = useCallback(() => {
    setSelectedItems(new Set());
  }, []);
  
  return (
    <div>
      <div>
        <button onClick={handleSelectAll}>すべて選択</button>
        <button onClick={handleClearSelection}>選択解除</button>
      </div>
      {items.map(item => (
        <ListItem
          key={item.id}
          item={item}
          isSelected={selectedItems.has(item.id)}
          onSelect={handleItemSelect}
        />
      ))}
    </div>
  );
};
```

### 仮想化（Virtualization）
```typescript
// react-window を使用した大量データの効率的な表示
import { FixedSizeList as List } from 'react-window';

interface VirtualizedListProps {
  items: Item[];
  height: number;
  itemHeight: number;
}

const VirtualizedList: React.FC<VirtualizedListProps> = ({ 
  items, 
  height, 
  itemHeight 
}) => {
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => (
    <div style={style}>
      <ItemComponent item={items[index]} />
    </div>
  );
  
  return (
    <List
      height={height}
      itemCount={items.length}
      itemSize={itemHeight}
      width="100%"
    >
      {Row}
    </List>
  );
};

// 動的サイズの仮想化
import { VariableSizeList as List } from 'react-window';

const DynamicVirtualizedList: React.FC<{ items: Item[] }> = ({ items }) => {
  const listRef = useRef<List>(null);
  const rowHeights = useRef<Record<number, number>>({});
  
  const getItemSize = (index: number) => {
    return rowHeights.current[index] || 50; // デフォルト高さ
  };
  
  const Row = ({ index, style }: { index: number; style: React.CSSProperties }) => {
    const rowRef = useRef<HTMLDivElement>(null);
    
    useEffect(() => {
      if (rowRef.current) {
        const height = rowRef.current.getBoundingClientRect().height;
        if (rowHeights.current[index] !== height) {
          rowHeights.current[index] = height;
          listRef.current?.resetAfterIndex(index);
        }
      }
    });
    
    return (
      <div style={style}>
        <div ref={rowRef}>
          <ItemComponent item={items[index]} />
        </div>
      </div>
    );
  };
  
  return (
    <List
      ref={listRef}
      height={600}
      itemCount={items.length}
      itemSize={getItemSize}
      width="100%"
    >
      {Row}
    </List>
  );
};

// 無限スクロール + 仮想化
import InfiniteLoader from 'react-window-infinite-loader';

const InfiniteVirtualizedList: React.FC = () => {
  const [items, setItems] = useState<Item[]>([]);
  const [hasNextPage, setHasNextPage] = useState(true);
  const [isNextPageLoading, setIsNextPageLoading] = useState(false);
  
  const loadNextPage = useCallback(async () => {
    setIsNextPageLoading(true);
    try {
      const newItems = await api.getItems(items.length, 50);
      setItems(prev => [...prev, ...newItems]);
      setHasNextPage(newItems.length === 50);
    } finally {
      setIsNextPageLoading(false);
    }
  }, [items.length]);
  
  const isItemLoaded = (index: number) => !!items[index];
  
  const Item = ({ index, style }: { index: number; style: React.CSSProperties }) => {
    const item = items[index];
    
    return (
      <div style={style}>
        {item ? (
          <ItemComponent item={item} />
        ) : (
          <div>読み込み中...</div>
        )}
      </div>
    );
  };
  
  return (
    <InfiniteLoader
      isItemLoaded={isItemLoaded}
      itemCount={hasNextPage ? items.length + 1 : items.length}
      loadMoreItems={loadNextPage}
    >
      {({ onItemsRendered, ref }) => (
        <List
          ref={ref}
          height={600}
          itemCount={hasNextPage ? items.length + 1 : items.length}
          itemSize={60}
          onItemsRendered={onItemsRendered}
          width="100%"
        >
          {Item}
        </List>
      )}
    </InfiniteLoader>
  );
};
```

## 画像・メディア最適化

### 次世代フォーマット対応
```typescript
// 複数フォーマット対応の画像コンポーネント
interface OptimizedImageProps {
  src: string;
  alt: string;
  width?: number;
  height?: number;
  className?: string;
  loading?: 'lazy' | 'eager';
}

const OptimizedImage: React.FC<OptimizedImageProps> = ({ 
  src, 
  alt, 
  width, 
  height, 
  className,
  loading = 'lazy'
}) => {
  const [imageFormat, setImageFormat] = useState<'webp' | 'avif' | 'jpg'>('jpg');
  
  useEffect(() => {
    // ブラウザサポート確認
    const checkWebPSupport = () => {
      const canvas = document.createElement('canvas');
      canvas.width = 1;
      canvas.height = 1;
      return canvas.toDataURL('image/webp').indexOf('data:image/webp') === 0;
    };
    
    const checkAVIFSupport = () => {
      const canvas = document.createElement('canvas');
      canvas.width = 1;
      canvas.height = 1;
      return canvas.toDataURL('image/avif').indexOf('data:image/avif') === 0;
    };
    
    if (checkAVIFSupport()) {
      setImageFormat('avif');
    } else if (checkWebPSupport()) {
      setImageFormat('webp');
    }
  }, []);
  
  const getOptimizedSrc = (originalSrc: string, format: string) => {
    const ext = originalSrc.split('.').pop();
    return originalSrc.replace(`.${ext}`, `.${format}`);
  };
  
  return (
    <picture>
      <source srcSet={getOptimizedSrc(src, 'avif')} type="image/avif" />
      <source srcSet={getOptimizedSrc(src, 'webp')} type="image/webp" />
      <img
        src={src}
        alt={alt}
        width={width}
        height={height}
        className={className}
        loading={loading}
        decoding="async"
      />
    </picture>
  );
};

// レスポンシブ画像
const ResponsiveImage: React.FC<{
  src: string;
  alt: string;
  sizes?: string;
}> = ({ src, alt, sizes = '100vw' }) => {
  const generateSrcSet = (baseSrc: string) => {
    const sizes = [320, 640, 768, 1024, 1280, 1920];
    return sizes
      .map(size => `${baseSrc}?w=${size} ${size}w`)
      .join(', ');
  };
  
  return (
    <img
      src={src}
      srcSet={generateSrcSet(src)}
      sizes={sizes}
      alt={alt}
      loading="lazy"
      decoding="async"
    />
  );
};
```

### 遅延読み込み（Lazy Loading）
```typescript
// Intersection Observer を使用した遅延読み込み
const useLazyLoad = (threshold = 0.1) => {
  const [isVisible, setIsVisible] = useState(false);
  const ref = useRef<HTMLElement>(null);
  
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
  
  return { ref, isVisible };
};

const LazyImage: React.FC<{
  src: string;
  alt: string;
  placeholder?: string;
}> = ({ src, alt, placeholder = '/placeholder.jpg' }) => {
  const { ref, isVisible } = useLazyLoad();
  const [loaded, setLoaded] = useState(false);
  
  return (
    <div ref={ref} className="lazy-image-container">
      {isVisible && (
        <>
          <img
            src={placeholder}
            alt={alt}
            className={`placeholder ${loaded ? 'hidden' : 'visible'}`}
          />
          <img
            src={src}
            alt={alt}
            className={`main-image ${loaded ? 'visible' : 'hidden'}`}
            onLoad={() => setLoaded(true)}
          />
        </>
      )}
    </div>
  );
};

// プログレッシブ画像読み込み
const ProgressiveImage: React.FC<{
  lowQualitySrc: string;
  highQualitySrc: string;
  alt: string;
}> = ({ lowQualitySrc, highQualitySrc, alt }) => {
  const [isHighQualityLoaded, setIsHighQualityLoaded] = useState(false);
  const { ref, isVisible } = useLazyLoad();
  
  useEffect(() => {
    if (isVisible) {
      const img = new Image();
      img.onload = () => setIsHighQualityLoaded(true);
      img.src = highQualitySrc;
    }
  }, [isVisible, highQualitySrc]);
  
  return (
    <div ref={ref} className="progressive-image">
      <img
        src={lowQualitySrc}
        alt={alt}
        className={`low-quality ${isHighQualityLoaded ? 'hidden' : 'visible'}`}
      />
      {isHighQualityLoaded && (
        <img
          src={highQualitySrc}
          alt={alt}
          className="high-quality"
        />
      )}
    </div>
  );
};
```

## ネットワーク最適化

### Service Worker でのキャッシュ戦略
```typescript
// sw.ts (Service Worker)
const CACHE_NAME = 'app-cache-v1';
const STATIC_CACHE = 'static-cache-v1';
const DYNAMIC_CACHE = 'dynamic-cache-v1';

const STATIC_ASSETS = [
  '/',
  '/static/css/main.css',
  '/static/js/main.js',
  '/manifest.json',
  '/offline.html'
];

// インストール時
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(STATIC_CACHE)
      .then(cache => cache.addAll(STATIC_ASSETS))
      .then(() => self.skipWaiting())
  );
});

// フェッチ時のキャッシュ戦略
self.addEventListener('fetch', (event) => {
  const { request } = event;
  const url = new URL(request.url);
  
  // HTML ファイル: Network First
  if (request.headers.get('accept')?.includes('text/html')) {
    event.respondWith(
      fetch(request)
        .then(response => {
          const responseClone = response.clone();
          caches.open(DYNAMIC_CACHE)
            .then(cache => cache.put(request, responseClone));
          return response;
        })
        .catch(() => {
          return caches.match(request)
            .then(response => response || caches.match('/offline.html'));
        })
    );
  }
  
  // 静的アセット: Cache First
  else if (url.pathname.startsWith('/static/')) {
    event.respondWith(
      caches.match(request)
        .then(response => {
          return response || fetch(request)
            .then(fetchResponse => {
              const responseClone = fetchResponse.clone();
              caches.open(STATIC_CACHE)
                .then(cache => cache.put(request, responseClone));
              return fetchResponse;
            });
        })
    );
  }
  
  // API: Network First with timeout
  else if (url.pathname.startsWith('/api/')) {
    event.respondWith(
      Promise.race([
        fetch(request),
        new Promise((_, reject) => 
          setTimeout(() => reject(new Error('timeout')), 3000)
        )
      ])
      .then(response => {
        if (response.ok) {
          const responseClone = response.clone();
          caches.open(DYNAMIC_CACHE)
            .then(cache => cache.put(request, responseClone));
        }
        return response;
      })
      .catch(() => caches.match(request))
    );
  }
});

// React での Service Worker 登録
const registerServiceWorker = () => {
  if ('serviceWorker' in navigator) {
    window.addEventListener('load', () => {
      navigator.serviceWorker.register('/sw.js')
        .then(registration => {
          console.log('SW registered: ', registration);
          
          // 更新チェック
          registration.addEventListener('updatefound', () => {
            const newWorker = registration.installing;
            newWorker?.addEventListener('statechange', () => {
              if (newWorker.state === 'installed') {
                if (navigator.serviceWorker.controller) {
                  // 新しいバージョンが利用可能
                  showUpdateNotification();
                }
              }
            });
          });
        })
        .catch(registrationError => {
          console.log('SW registration failed: ', registrationError);
        });
    });
  }
};
```

### リソースの優先度付け
```typescript
// Critical Resources の優先読み込み
const CriticalResourceLoader: React.FC = () => {
  useEffect(() => {
    // 重要なフォントのプリロード
    const fontLink = document.createElement('link');
    fontLink.rel = 'preload';
    fontLink.href = '/fonts/inter.woff2';
    fontLink.as = 'font';
    fontLink.type = 'font/woff2';
    fontLink.crossOrigin = 'anonymous';
    document.head.appendChild(fontLink);
    
    // 重要な画像のプリロード
    const heroImageLink = document.createElement('link');
    heroImageLink.rel = 'preload';
    heroImageLink.href = '/images/hero.webp';
    heroImageLink.as = 'image';
    document.head.appendChild(heroImageLink);
    
    // 次のページのプリフェッチ
    const nextPageLink = document.createElement('link');
    nextPageLink.rel = 'prefetch';
    nextPageLink.href = '/dashboard';
    document.head.appendChild(nextPageLink);
    
    return () => {
      document.head.removeChild(fontLink);
      document.head.removeChild(heroImageLink);
      document.head.removeChild(nextPageLink);
    };
  }, []);
  
  return null;
};

// 動的な優先度設定
const usePriorityHints = () => {
  const preloadResource = useCallback((url: string, as: string, priority: 'high' | 'low' = 'high') => {
    const link = document.createElement('link');
    link.rel = 'preload';
    link.href = url;
    link.as = as;
    if (priority === 'high') {
      link.setAttribute('importance', 'high');
    }
    document.head.appendChild(link);
    
    return () => document.head.removeChild(link);
  }, []);
  
  const prefetchResource = useCallback((url: string) => {
    const link = document.createElement('link');
    link.rel = 'prefetch';
    link.href = url;
    document.head.appendChild(link);
    
    return () => document.head.removeChild(link);
  }, []);
  
  return { preloadResource, prefetchResource };
};
```

## Runtime パフォーマンス最適化

### Web Workers の活用
```typescript
// worker.ts
self.onmessage = function(e) {
  const { type, data } = e.data;
  
  switch (type) {
    case 'HEAVY_COMPUTATION':
      const result = performHeavyComputation(data);
      self.postMessage({ type: 'COMPUTATION_RESULT', result });
      break;
      
    case 'PROCESS_CSV':
      const processedData = processCsvData(data);
      self.postMessage({ type: 'CSV_PROCESSED', data: processedData });
      break;
      
    case 'IMAGE_PROCESSING':
      const processedImage = processImage(data);
      self.postMessage({ type: 'IMAGE_PROCESSED', image: processedImage });
      break;
  }
};

function performHeavyComputation(data: number[]): number {
  // CPU集約的な処理
  let result = 0;
  for (let i = 0; i < data.length; i++) {
    for (let j = 0; j < 1000000; j++) {
      result += Math.sqrt(data[i] * j);
    }
  }
  return result;
}

// React での Web Worker 使用
const useWebWorker = (workerScript: string) => {
  const worker = useRef<Worker>();
  const [result, setResult] = useState<any>(null);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  useEffect(() => {
    worker.current = new Worker(workerScript);
    
    worker.current.onmessage = (e) => {
      setResult(e.data);
      setLoading(false);
    };
    
    worker.current.onerror = (err) => {
      setError(err.message);
      setLoading(false);
    };
    
    return () => worker.current?.terminate();
  }, [workerScript]);
  
  const postMessage = useCallback((message: any) => {
    setLoading(true);
    setError(null);
    worker.current?.postMessage(message);
  }, []);
  
  return { postMessage, result, loading, error };
};

// 使用例
const HeavyComputationComponent: React.FC = () => {
  const { postMessage, result, loading } = useWebWorker('/worker.js');
  
  const handleComputation = () => {
    const data = Array.from({ length: 10000 }, () => Math.random() * 100);
    postMessage({ type: 'HEAVY_COMPUTATION', data });
  };
  
  return (
    <div>
      <button onClick={handleComputation} disabled={loading}>
        {loading ? '計算中...' : '重い計算を実行'}
      </button>
      {result && <p>結果: {result.result}</p>}
    </div>
  );
};
```

### メモリ使用量の最適化
```typescript
// メモリリークの防止
const useMemoryOptimizedEffect = (effect: React.EffectCallback, deps?: React.DependencyList) => {
  useEffect(() => {
    const cleanup = effect();
    
    return () => {
      // 明示的なクリーンアップ
      if (cleanup && typeof cleanup === 'function') {
        cleanup();
      }
      
      // ガベージコレクションのヒント（非標準）
      if ('gc' in window && typeof window.gc === 'function') {
        window.gc();
      }
    };
  }, deps);
};

// 大きなオブジェクトの効率的な管理
const useLargeDataset = (data: LargeDataItem[]) => {
  const [processedData, setProcessedData] = useState<ProcessedData[]>([]);
  const processingRef = useRef<boolean>(false);
  
  useEffect(() => {
    if (processingRef.current) return;
    
    processingRef.current = true;
    
    // 分割処理でメインスレッドをブロックしない
    const processInChunks = async () => {
      const chunkSize = 1000;
      const result: ProcessedData[] = [];
      
      for (let i = 0; i < data.length; i += chunkSize) {
        const chunk = data.slice(i, i + chunkSize);
        const processedChunk = await new Promise<ProcessedData[]>(resolve => {
          setTimeout(() => {
            resolve(chunk.map(item => processItem(item)));
          }, 0);
        });
        
        result.push(...processedChunk);
        
        // 進捗更新（UIをブロックしない）
        await new Promise(resolve => setTimeout(resolve, 0));
      }
      
      setProcessedData(result);
      processingRef.current = false;
    };
    
    processInChunks();
    
    return () => {
      processingRef.current = false;
      setProcessedData([]);
    };
  }, [data]);
  
  return processedData;
};

// WeakMap を使用したメモリ効率的なキャッシュ
class MemoryEfficientCache<K extends object, V> {
  private cache = new WeakMap<K, V>();
  
  get(key: K): V | undefined {
    return this.cache.get(key);
  }
  
  set(key: K, value: V): void {
    this.cache.set(key, value);
  }
  
  has(key: K): boolean {
    return this.cache.has(key);
  }
  
  delete(key: K): boolean {
    return this.cache.delete(key);
  }
}

// 使用例
const useComponentCache = () => {
  const cache = useMemo(() => new MemoryEfficientCache<ComponentProps, JSX.Element>(), []);
  
  const getCachedComponent = useCallback((props: ComponentProps) => {
    if (cache.has(props)) {
      return cache.get(props)!;
    }
    
    const component = <ExpensiveComponent {...props} />;
    cache.set(props, component);
    return component;
  }, [cache]);
  
  return getCachedComponent;
};
```