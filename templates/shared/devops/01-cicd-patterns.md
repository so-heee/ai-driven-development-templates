# CI/CDパターンガイド

## 継続的インテグレーション（CI）

### 基本的なCIパイプライン
```yaml
# GitHub Actions例
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'
    
    - name: Install dependencies
      run: npm ci
    
    - name: Run linter
      run: npm run lint
    
    - name: Run tests
      run: npm run test:coverage
    
    - name: Build
      run: npm run build
```

### テスト戦略
```yaml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - name: Unit Tests
        run: npm run test:unit
  
  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - name: Integration Tests
        run: npm run test:integration
  
  e2e-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests, integration-tests]
    steps:
      - name: E2E Tests
        run: npm run test:e2e
```

### 品質ゲート
```yaml
jobs:
  quality-gate:
    runs-on: ubuntu-latest
    steps:
      - name: Code Coverage Check
        run: |
          npm run test:coverage
          npm run coverage:check
      
      - name: Security Scan
        run: npm audit
      
      - name: Dependency Check
        run: npm run deps:check
```

## 継続的デプロイメント（CD）

### ブランチング戦略とデプロイ
```yaml
# Feature Branch → Development
deploy-dev:
  if: github.ref == 'refs/heads/develop'
  environment: development
  steps:
    - name: Deploy to Development
      run: npm run deploy:dev

# Main Branch → Staging
deploy-staging:
  if: github.ref == 'refs/heads/main'
  environment: staging
  steps:
    - name: Deploy to Staging
      run: npm run deploy:staging

# Release Tag → Production
deploy-production:
  if: startsWith(github.ref, 'refs/tags/v')
  environment: production
  steps:
    - name: Deploy to Production
      run: npm run deploy:prod
```

### Blue-Greenデプロイメント
```yaml
deploy-blue-green:
  runs-on: ubuntu-latest
  steps:
    - name: Deploy to Green Environment
      run: |
        kubectl apply -f k8s/green-deployment.yaml
        kubectl wait --for=condition=ready pod -l app=myapp,slot=green
    
    - name: Run Health Checks
      run: |
        npm run health-check:green
    
    - name: Switch Traffic
      run: |
        kubectl patch service myapp-service -p '{"spec":{"selector":{"slot":"green"}}}'
    
    - name: Cleanup Blue Environment
      run: |
        kubectl delete deployment myapp-blue
```

### カナリアデプロイメント
```yaml
canary-deploy:
  runs-on: ubuntu-latest
  steps:
    - name: Deploy Canary (10%)
      run: |
        kubectl apply -f k8s/canary-deployment.yaml
        kubectl scale deployment myapp-canary --replicas=1
        kubectl scale deployment myapp-stable --replicas=9
    
    - name: Monitor Metrics
      run: |
        sleep 300
        npm run monitor:canary
    
    - name: Increase Canary Traffic (50%)
      if: success()
      run: |
        kubectl scale deployment myapp-canary --replicas=5
        kubectl scale deployment myapp-stable --replicas=5
    
    - name: Full Rollout
      if: success()
      run: |
        kubectl scale deployment myapp-canary --replicas=10
        kubectl scale deployment myapp-stable --replicas=0
```

## パイプライン設計パターン

### 並列実行パターン
```yaml
jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - name: Lint
        run: npm run lint
  
  unit-test:
    runs-on: ubuntu-latest
    steps:
      - name: Unit Test
        run: npm run test:unit
  
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - name: Security Scan
        run: npm run security:scan
  
  build:
    runs-on: ubuntu-latest
    needs: [lint, unit-test, security-scan]
    steps:
      - name: Build
        run: npm run build
```

### マトリックス戦略
```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node-version: [16, 18, 20]
    
    steps:
      - name: Test on ${{ matrix.os }} with Node ${{ matrix.node-version }}
        run: npm test
```

### 条件付き実行
```yaml
jobs:
  deploy:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        run: npm run deploy
  
  notify-slack:
    if: failure()
    runs-on: ubuntu-latest
    steps:
      - name: Notify Failure
        run: |
          curl -X POST -H 'Content-type: application/json' \
            --data '{"text":"Build failed for ${{ github.repository }}"}' \
            ${{ secrets.SLACK_WEBHOOK_URL }}
```

## 環境管理

### 環境変数管理
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production
    env:
      NODE_ENV: production
      DATABASE_URL: ${{ secrets.DATABASE_URL }}
      API_KEY: ${{ secrets.API_KEY }}
    
    steps:
      - name: Deploy with Environment Variables
        run: |
          echo "Deploying to $NODE_ENV environment"
          npm run deploy
```

### シークレット管理
```yaml
# GitHub Secrets使用例
steps:
  - name: Configure AWS Credentials
    uses: aws-actions/configure-aws-credentials@v2
    with:
      aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
      aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      aws-region: ap-northeast-1
```

## ロールバック戦略

### 自動ロールバック
```yaml
jobs:
  deploy-with-rollback:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy
        id: deploy
        run: npm run deploy
      
      - name: Health Check
        id: health
        run: npm run health-check
      
      - name: Rollback on Failure
        if: failure()
        run: |
          echo "Deployment failed, rolling back..."
          npm run rollback
          exit 1
```

### 手動承認フロー
```yaml
jobs:
  deploy-approval:
    runs-on: ubuntu-latest
    environment: production
    steps:
      - name: Wait for Approval
        uses: trstringer/manual-approval@v1
        with:
          secret: ${{ github.TOKEN }}
          approvers: team-leads
          minimum-approvals: 2
      
      - name: Deploy to Production
        run: npm run deploy:prod
```

## 監視とアラート

### デプロイメント後の監視
```yaml
jobs:
  post-deploy-monitoring:
    runs-on: ubuntu-latest
    needs: deploy
    steps:
      - name: Wait for Deployment
        run: sleep 60
      
      - name: Check Application Health
        run: |
          curl -f https://api.example.com/health
      
      - name: Check Response Time
        run: |
          npm run performance:check
      
      - name: Alert on Issues
        if: failure()
        run: |
          npm run alert:send
```

## 最適化パターン

### キャッシュ戦略
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Cache Dependencies
        uses: actions/cache@v3
        with:
          path: ~/.npm
          key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
      
      - name: Cache Build Output
        uses: actions/cache@v3
        with:
          path: dist
          key: ${{ runner.os }}-build-${{ github.sha }}
```

### アーティファクト管理
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build
        run: npm run build
      
      - name: Upload Build Artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-files
          path: dist/
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download Build Artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-files
          path: dist/
```

## トラブルシューティング

### よくある問題と解決策

#### ビルド失敗
```bash
# 依存関係の問題
npm ci --prefer-offline

# キャッシュクリア
npm run cache:clear
rm -rf node_modules package-lock.json
npm install
```

#### デプロイメント失敗
```bash
# ヘルスチェック失敗時の診断
kubectl logs deployment/myapp
kubectl describe pod -l app=myapp

# ロールバック
kubectl rollout undo deployment/myapp
```

#### パフォーマンス問題
```bash
# リソース使用量チェック
kubectl top pods
kubectl top nodes

# パイプライン最適化
# 並列実行の増加
# キャッシュ戦略の見直し
# 不要なステップの削除
```