# スタイリングガイド

## CSS設計手法

### BEM記法

```scss
// Block - Component
.user-card {
  padding: 1rem;
  border: 1px solid #ddd;
  border-radius: 8px;

  // Element - Component の一部
  &__header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 1rem;
  }

  &__title {
    font-size: 1.2rem;
    font-weight: bold;
    color: #333;
  }

  &__content {
    line-height: 1.6;
  }

  &__actions {
    display: flex;
    gap: 0.5rem;
    margin-top: 1rem;
  }

  // Modifier - Component のバリエーション
  &--highlighted {
    border-color: #007bff;
    box-shadow: 0 0 0 2px rgba(0, 123, 255, 0.25);
  }

  &--compact {
    padding: 0.5rem;

    .user-card__title {
      font-size: 1rem;
    }
  }

  &--disabled {
    opacity: 0.6;
    pointer-events: none;
  }
}
```

### CSS Custom Properties（CSS変数）

```scss
:root {
  // Color System
  --color-primary-50: #eff6ff;
  --color-primary-100: #dbeafe;
  --color-primary-500: #3b82f6;
  --color-primary-600: #2563eb;
  --color-primary-900: #1e3a8a;

  --color-gray-50: #f9fafb;
  --color-gray-100: #f3f4f6;
  --color-gray-500: #6b7280;
  --color-gray-900: #111827;

  --color-success: #10b981;
  --color-warning: #f59e0b;
  --color-error: #ef4444;

  // Typography
  --font-family-sans: 'Inter', system-ui, sans-serif;
  --font-family-mono: 'JetBrains Mono', 'Fira Code', monospace;

  --font-size-xs: 0.75rem;
  --font-size-sm: 0.875rem;
  --font-size-base: 1rem;
  --font-size-lg: 1.125rem;
  --font-size-xl: 1.25rem;
  --font-size-2xl: 1.5rem;
  --font-size-3xl: 1.875rem;

  --font-weight-normal: 400;
  --font-weight-medium: 500;
  --font-weight-semibold: 600;
  --font-weight-bold: 700;

  // Spacing
  --space-1: 0.25rem;
  --space-2: 0.5rem;
  --space-3: 0.75rem;
  --space-4: 1rem;
  --space-6: 1.5rem;
  --space-8: 2rem;
  --space-12: 3rem;
  --space-16: 4rem;

  // Border Radius
  --radius-sm: 0.125rem;
  --radius-md: 0.375rem;
  --radius-lg: 0.5rem;
  --radius-xl: 0.75rem;
  --radius-full: 9999px;

  // Shadows
  --shadow-sm: 0 1px 2px 0 rgb(0 0 0 / 0.05);
  --shadow-md: 0 4px 6px -1px rgb(0 0 0 / 0.1);
  --shadow-lg: 0 10px 15px -3px rgb(0 0 0 / 0.1);

  // Z-index
  --z-dropdown: 1000;
  --z-sticky: 1020;
  --z-fixed: 1030;
  --z-modal: 1040;
  --z-popover: 1050;
  --z-tooltip: 1060;
}

// ダークテーマ
[data-theme='dark'] {
  --color-primary-50: #1e3a8a;
  --color-primary-100: #2563eb;
  --color-primary-500: #60a5fa;
  --color-primary-600: #93c5fd;
  --color-primary-900: #dbeafe;

  --color-gray-50: #111827;
  --color-gray-100: #1f2937;
  --color-gray-500: #9ca3af;
  --color-gray-900: #f9fafb;
}
```

## CSS-in-JS パターン

### styled-components

```tsx
import styled, { css } from 'styled-components'

// 基本的なスタイルコンポーネント
const Button = styled.button<{
  variant?: 'primary' | 'secondary' | 'danger'
  size?: 'sm' | 'md' | 'lg'
  fullWidth?: boolean
}>`
  display: inline-flex;
  align-items: center;
  justify-content: center;
  gap: 0.5rem;

  border: none;
  border-radius: var(--radius-md);
  font-weight: var(--font-weight-medium);
  cursor: pointer;
  transition: all 0.2s ease-in-out;

  &:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  // サイズバリエーション
  ${props => {
    switch (props.size) {
      case 'sm':
        return css`
          padding: var(--space-2) var(--space-3);
          font-size: var(--font-size-sm);
        `
      case 'lg':
        return css`
          padding: var(--space-4) var(--space-6);
          font-size: var(--font-size-lg);
        `
      default:
        return css`
          padding: var(--space-3) var(--space-4);
          font-size: var(--font-size-base);
        `
    }
  }}

  // カラーバリエーション
  ${props => {
    switch (props.variant) {
      case 'primary':
        return css`
          background: var(--color-primary-500);
          color: white;

          &:hover:not(:disabled) {
            background: var(--color-primary-600);
          }

          &:focus {
            outline: 2px solid var(--color-primary-500);
            outline-offset: 2px;
          }
        `
      case 'danger':
        return css`
          background: var(--color-error);
          color: white;

          &:hover:not(:disabled) {
            background: #dc2626;
          }
        `
      default:
        return css`
          background: var(--color-gray-100);
          color: var(--color-gray-900);

          &:hover:not(:disabled) {
            background: var(--color-gray-200);
          }
        `
    }
  }}
  
  // 全幅スタイル
  ${props =>
    props.fullWidth &&
    css`
      width: 100%;
    `}
`

// 使用例
export const ActionButton: React.FC<{
  onClick: () => void
  loading?: boolean
  children: React.ReactNode
}> = ({ onClick, loading, children }) => (
  <Button variant="primary" size="md" onClick={onClick} disabled={loading}>
    {loading && <Spinner size="sm" />}
    {children}
  </Button>
)
```

### emotion

```tsx
import { css, jsx } from '@emotion/react'
import styled from '@emotion/styled'

// CSS関数を使用
const buttonStyles = css`
  padding: 12px 24px;
  border-radius: 6px;
  border: none;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;

  &:hover {
    transform: translateY(-1px);
  }
`

// 条件付きスタイル
const getButtonVariant = (variant: string) => {
  switch (variant) {
    case 'primary':
      return css`
        background: #3b82f6;
        color: white;
        &:hover {
          background: #2563eb;
        }
      `
    case 'outline':
      return css`
        background: transparent;
        color: #3b82f6;
        border: 1px solid #3b82f6;
        &:hover {
          background: #eff6ff;
        }
      `
    default:
      return css`
        background: #f3f4f6;
        color: #374151;
      `
  }
}

// styled コンポーネント
const StyledButton = styled.button<{ variant: string }>`
  ${buttonStyles}
  ${props => getButtonVariant(props.variant)}
`
```

## Tailwind CSS パターン

### 基本的なコンポーネント

```tsx
// ユーティリティクラスの組み合わせ
const Card: React.FC<{ children: React.ReactNode; className?: string }> = ({ children, className = '' }) => (
  <div
    className={`
    bg-white rounded-lg shadow-md border border-gray-200 
    p-6 hover:shadow-lg transition-shadow duration-200
    ${className}
  `}
  >
    {children}
  </div>
)

// レスポンシブデザイン
const ResponsiveGrid: React.FC<{ children: React.ReactNode }> = ({ children }) => (
  <div
    className="
    grid 
    grid-cols-1 
    md:grid-cols-2 
    lg:grid-cols-3 
    xl:grid-cols-4 
    gap-4 
    md:gap-6 
    lg:gap-8
  "
  >
    {children}
  </div>
)

// 状態に応じたスタイル
const StatusBadge: React.FC<{
  status: 'active' | 'inactive' | 'pending'
  children: React.ReactNode
}> = ({ status, children }) => {
  const baseClasses = 'px-2 py-1 rounded-full text-xs font-medium'
  const statusClasses = {
    active: 'bg-green-100 text-green-800',
    inactive: 'bg-gray-100 text-gray-800',
    pending: 'bg-yellow-100 text-yellow-800',
  }

  return <span className={`${baseClasses} ${statusClasses[status]}`}>{children}</span>
}
```

### カスタムクラスとコンポーネント

```scss
// tailwind.config.js でのカスタマイズ
module.exports = {
  theme: {
    extend: {
      colors: {
        brand: {
          50: '#eff6ff',
          500: '#3b82f6',
          900: '#1e3a8a'
        }
      },
      spacing: {
        '18': '4.5rem',
        '88': '22rem'
      }
    }
  },
  plugins: [
    require('@tailwindcss/forms'),
    require('@tailwindcss/typography')
  ]
};

// CSS での @apply ディレクティブ
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer components {
  .btn {
    @apply px-4 py-2 rounded font-medium transition-colors;
  }

  .btn-primary {
    @apply btn bg-blue-500 text-white hover:bg-blue-600;
  }

  .btn-secondary {
    @apply btn bg-gray-200 text-gray-900 hover:bg-gray-300;
  }

  .card {
    @apply bg-white rounded-lg shadow border p-6;
  }

  .form-input {
    @apply w-full px-3 py-2 border border-gray-300 rounded-md
           focus:outline-none focus:ring-2 focus:ring-blue-500 focus:border-transparent;
  }
}
```

## アニメーション・トランジション

### CSS アニメーション

```scss
// キーフレームアニメーション
@keyframes fadeIn {
  from {
    opacity: 0;
    transform: translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateY(0);
  }
}

@keyframes pulse {
  0%,
  100% {
    opacity: 1;
  }
  50% {
    opacity: 0.5;
  }
}

@keyframes spin {
  to {
    transform: rotate(360deg);
  }
}

// アニメーションクラス
.animate-fade-in {
  animation: fadeIn 0.3s ease-out;
}

.animate-pulse {
  animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}

.animate-spin {
  animation: spin 1s linear infinite;
}

// トランジション
.transition-all {
  transition: all 0.2s ease-in-out;
}

.transition-transform {
  transition: transform 0.2s ease-in-out;
}

// ホバーエフェクト
.hover-lift {
  transition: transform 0.2s ease-in-out;

  &:hover {
    transform: translateY(-2px);
  }
}

.hover-grow {
  transition: transform 0.2s ease-in-out;

  &:hover {
    transform: scale(1.05);
  }
}
```

### React Transition Group

```tsx
import { CSSTransition, TransitionGroup } from 'react-transition-group';

// リストアイテムのアニメーション
const AnimatedList: React.FC<{ items: Item[] }> = ({ items }) => (
  <TransitionGroup component="ul" className="animated-list">
    {items.map(item => (
      <CSSTransition
        key={item.id}
        timeout={300}
        classNames="list-item"
      >
        <li className="list-item">
          {item.name}
        </li>
      </CSSTransition>
    ))}
  </TransitionGroup>
);

// 対応するCSS
.list-item-enter {
  opacity: 0;
  transform: translateX(-100%);
}

.list-item-enter-active {
  opacity: 1;
  transform: translateX(0);
  transition: opacity 300ms, transform 300ms;
}

.list-item-exit {
  opacity: 1;
  transform: translateX(0);
}

.list-item-exit-active {
  opacity: 0;
  transform: translateX(100%);
  transition: opacity 300ms, transform 300ms;
}
```

### Framer Motion

```tsx
import { motion, AnimatePresence } from 'framer-motion'

// 基本的なアニメーション
const AnimatedCard: React.FC<{ children: React.ReactNode }> = ({ children }) => (
  <motion.div
    initial={{ opacity: 0, y: 20 }}
    animate={{ opacity: 1, y: 0 }}
    exit={{ opacity: 0, y: -20 }}
    transition={{ duration: 0.3 }}
    whileHover={{ scale: 1.02 }}
    whileTap={{ scale: 0.98 }}
    className="card"
  >
    {children}
  </motion.div>
)

// 条件付きアニメーション
const Modal: React.FC<{ isOpen: boolean; onClose: () => void; children: React.ReactNode }> = ({
  isOpen,
  onClose,
  children,
}) => (
  <AnimatePresence>
    {isOpen && (
      <>
        <motion.div
          initial={{ opacity: 0 }}
          animate={{ opacity: 1 }}
          exit={{ opacity: 0 }}
          className="modal-overlay"
          onClick={onClose}
        />
        <motion.div
          initial={{ opacity: 0, scale: 0.8 }}
          animate={{ opacity: 1, scale: 1 }}
          exit={{ opacity: 0, scale: 0.8 }}
          transition={{ type: 'spring', damping: 25, stiffness: 500 }}
          className="modal-content"
        >
          {children}
        </motion.div>
      </>
    )}
  </AnimatePresence>
)

// スタガー（時差）アニメーション
const container = {
  hidden: { opacity: 0 },
  show: {
    opacity: 1,
    transition: {
      staggerChildren: 0.1,
    },
  },
}

const item = {
  hidden: { opacity: 0, y: 20 },
  show: { opacity: 1, y: 0 },
}

const StaggeredList: React.FC<{ items: string[] }> = ({ items }) => (
  <motion.ul variants={container} initial="hidden" animate="show">
    {items.map((item, index) => (
      <motion.li key={index} variants={item}>
        {item}
      </motion.li>
    ))}
  </motion.ul>
)
```

## レスポンシブデザイン

### ブレイクポイント設計

```scss
// ブレイクポイント定義
$breakpoints: (
  xs: 0,
  sm: 576px,
  md: 768px,
  lg: 992px,
  xl: 1200px,
  xxl: 1400px,
);

// ミックスイン
@mixin media-up($size) {
  @media (min-width: map-get($breakpoints, $size)) {
    @content;
  }
}

@mixin media-down($size) {
  @media (max-width: map-get($breakpoints, $size) - 1px) {
    @content;
  }
}

@mixin media-between($lower, $upper) {
  @media (min-width: map-get($breakpoints, $lower)) and (max-width: map-get($breakpoints, $upper) - 1px) {
    @content;
  }
}

// 使用例
.container {
  padding: 1rem;

  @include media-up(md) {
    padding: 2rem;
  }

  @include media-up(lg) {
    padding: 3rem;
    max-width: 1200px;
    margin: 0 auto;
  }
}

.grid {
  display: grid;
  grid-template-columns: 1fr;
  gap: 1rem;

  @include media-up(md) {
    grid-template-columns: repeat(2, 1fr);
    gap: 2rem;
  }

  @include media-up(lg) {
    grid-template-columns: repeat(3, 1fr);
  }

  @include media-up(xl) {
    grid-template-columns: repeat(4, 1fr);
    gap: 3rem;
  }
}
```

### Container Queries

```scss
// モダンなContainer Queries
.card-container {
  container-type: inline-size;
}

.card {
  padding: 1rem;

  @container (min-width: 300px) {
    padding: 1.5rem;
    display: flex;
    align-items: center;
  }

  @container (min-width: 500px) {
    padding: 2rem;

    .card__image {
      width: 120px;
      height: 120px;
    }
  }
}
```

## パフォーマンス最適化

### CSS最適化

```scss
// Critical CSS の分離
// above-the-fold コンテンツに必要なスタイルのみを含める

// 効率的なセレクタの使用
// ❌ 非効率
div.container > ul.list > li.item > a.link {
  color: blue;
}

// ✅ 効率的
.nav-link {
  color: blue;
}

// レイアウトシフトを避ける
.image-container {
  aspect-ratio: 16 / 9;
  background-color: #f3f4f6;
}

.image-container img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

// GPU加速を活用
.animate-element {
  transform: translateZ(0); // GPUレイヤーを作成
  will-change: transform; // ブラウザに最適化のヒントを提供
}

// 不要なrepaintを避ける
.smooth-animation {
  // position, width, height の変更ではなく transform を使用
  transition: transform 0.3s ease;
}

.smooth-animation:hover {
  transform: scale(1.05);
}
```

### 動的インポート（CSS-in-JS）

```tsx
// 必要時にのみスタイルを読み込み
const LazyStyledComponent = React.lazy(() =>
  import('./HeavyStyledComponent').then(module => ({
    default: module.HeavyStyledComponent,
  }))
)

// テーマの動的切り替え
const ThemeProvider: React.FC<{ children: React.ReactNode }> = ({ children }) => {
  const [theme, setTheme] = useState<'light' | 'dark'>('light')

  useEffect(() => {
    // テーマに応じてCSS変数を動的に更新
    document.documentElement.setAttribute('data-theme', theme)
  }, [theme])

  return <ThemeContext.Provider value={{ theme, setTheme }}>{children}</ThemeContext.Provider>
}
```
