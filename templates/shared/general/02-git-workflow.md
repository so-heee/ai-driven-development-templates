# Git ワークフローガイド

## ブランチ戦略

### Git Flow
```bash
# メインブランチ
main          # 本番環境にデプロイされるコード
develop       # 開発の統合ブランチ

# サポートブランチ
feature/*     # 新機能開発
release/*     # リリース準備
hotfix/*      # 緊急修正

# ブランチ作成例
git checkout -b feature/user-authentication develop
git checkout -b release/v1.2.0 develop
git checkout -b hotfix/critical-bug main
```

### GitHub Flow（推奨）
```bash
# シンプルなワークフロー
main          # 常にデプロイ可能な状態
feature/*     # 機能開発ブランチ

# 基本的な流れ
git checkout main
git pull origin main
git checkout -b feature/new-feature
# 開発作業
git push origin feature/new-feature
# Pull Request 作成
# レビュー → マージ
```

### ブランチ命名規則
```bash
# 機能開発
feature/user-profile-page
feature/payment-integration
feature/api-user-management

# バグ修正
bugfix/login-validation-error
bugfix/memory-leak-fix

# ホットフィックス
hotfix/security-vulnerability
hotfix/critical-performance-issue

# ドキュメント
docs/api-documentation
docs/deployment-guide

# リファクタリング
refactor/user-service-cleanup
refactor/database-optimization
```

## コミット規則

### Conventional Commits
```bash
# 基本形式
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]

# 例
feat(auth): add OAuth2 login functionality
fix(api): resolve user data validation error
docs(readme): update installation instructions
style(header): fix navigation menu alignment
refactor(utils): extract common validation functions
test(user): add unit tests for user service
chore(deps): update dependencies to latest versions
```

### コミットタイプ
```bash
feat:     新機能の追加
fix:      バグ修正
docs:     ドキュメントの変更
style:    コードフォーマットの変更（機能に影響しない）
refactor: リファクタリング（機能変更なし）
test:     テストの追加・修正
chore:    ビルドプロセスやツールの変更
perf:     パフォーマンス改善
ci:       CI設定の変更
build:    ビルドシステムの変更
revert:   コミットの取り消し
```

### コミットメッセージの例
```bash
# 良い例
feat(user): add user profile editing functionality

- Add form validation for email and phone
- Implement real-time preview
- Add success/error notifications

Closes #123

# 悪い例
fix stuff
update code
```

## Pull Request ガイド

### PR テンプレート
```markdown
## 概要
このPRの目的と変更内容を簡潔に説明してください。

## 変更内容
- [ ] 新機能の追加
- [ ] バグ修正
- [ ] ドキュメント更新
- [ ] リファクタリング
- [ ] テスト追加

## 詳細な変更点
- 機能A: 〇〇を追加
- 機能B: 〇〇を修正
- テスト: 〇〇のテストを追加

## テスト
- [ ] 既存のテストが通ることを確認
- [ ] 新しいテストを追加
- [ ] 手動テストを実施

## スクリーンショット（該当する場合）
変更前：
変更後：

## チェックリスト
- [ ] コードレビューを受けた
- [ ] テストが通ることを確認
- [ ] ドキュメントを更新
- [ ] CHANGELOG.mdを更新（必要に応じて）

## 関連Issue
Closes #123
Related to #456
```

### レビュー観点
```markdown
## コードレビューチェックリスト

### 機能性
- [ ] 要件を満たしているか
- [ ] エッジケースが考慮されているか
- [ ] エラーハンドリングが適切か

### コード品質
- [ ] 可読性が高いか
- [ ] 適切な設計パターンが使用されているか
- [ ] DRY原則が守られているか
- [ ] SOLID原則が守られているか

### パフォーマンス
- [ ] 不要な処理がないか
- [ ] メモリリークの可能性はないか
- [ ] N+1問題などの問題はないか

### セキュリティ
- [ ] 入力値の検証が適切か
- [ ] 認証・認可が適切か
- [ ] 機密情報の漏洩はないか

### テスト
- [ ] 適切なテストが書かれているか
- [ ] テストカバレッジが十分か
- [ ] テストの命名が適切か
```

## マージ戦略

### マージ方法の選択
```bash
# 1. Merge Commit（履歴を保持）
git checkout main
git merge --no-ff feature/new-feature

# 2. Squash and Merge（コミット履歴を整理）
git checkout main
git merge --squash feature/new-feature
git commit -m "feat: add new feature"

# 3. Rebase and Merge（線形履歴）
git checkout feature/new-feature
git rebase main
git checkout main
git merge feature/new-feature
```

### 各手法の使い分け
```markdown
## Merge Commit
- 使用場面: 機能の開発履歴を保持したい場合
- メリット: 完全な履歴が残る
- デメリット: 履歴が複雑になる

## Squash and Merge
- 使用場面: 小さな機能や修正（推奨）
- メリット: 履歴がクリーン
- デメリット: 詳細な履歴が失われる

## Rebase and Merge
- 使用場面: 線形履歴を保ちたい場合
- メリット: クリーンな線形履歴
- デメリット: コンフリクト解決が複雑
```

## コンフリクト解決

### コンフリクト発生時の対応
```bash
# 1. リモートの最新状態を取得
git fetch origin

# 2. メインブランチを最新に更新
git checkout main
git pull origin main

# 3. フィーチャーブランチにリベース
git checkout feature/my-feature
git rebase main

# 4. コンフリクト解決
# エディタでコンフリクトを解決

# 5. 解決後のコミット
git add .
git rebase --continue

# 6. プッシュ（強制プッシュが必要）
git push --force-with-lease origin feature/my-feature
```

### コンフリクト解決のベストプラクティス
```bash
# コンフリクトマーカーの理解
<<<<<<< HEAD
現在のブランチの内容
=======
マージしようとしているブランチの内容
>>>>>>> feature/my-feature

# 解決後の確認
git status
git diff --cached

# テストの実行
npm test
```

## リリース管理

### セマンティックバージョニング
```bash
# バージョン形式: MAJOR.MINOR.PATCH
# 例: 1.2.3

MAJOR: 破壊的変更があるとき（1.0.0 → 2.0.0）
MINOR: 後方互換性のある機能追加（1.0.0 → 1.1.0）
PATCH: 後方互換性のあるバグ修正（1.0.0 → 1.0.1）

# プレリリース版
1.0.0-alpha.1    # アルファ版
1.0.0-beta.1     # ベータ版
1.0.0-rc.1       # リリース候補版
```

### タグとリリース
```bash
# タグの作成
git tag v1.2.0
git tag -a v1.2.0 -m "Release version 1.2.0"

# タグのプッシュ
git push origin v1.2.0
git push origin --tags

# GitHub Release の自動化
# .github/workflows/release.yml
name: Release
on:
  push:
    tags:
      - 'v*'
jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Create Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ github.ref }}
          release_name: Release ${{ github.ref }}
```

## Git Hooks

### Pre-commit フック
```bash
#!/bin/sh
# .git/hooks/pre-commit

# リント実行
npm run lint
if [ $? -ne 0 ]; then
  echo "Lint failed. Please fix the issues before committing."
  exit 1
fi

# テスト実行
npm test
if [ $? -ne 0 ]; then
  echo "Tests failed. Please fix the issues before committing."
  exit 1
fi

# フォーマット確認
npm run format:check
if [ $? -ne 0 ]; then
  echo "Code formatting issues found. Run 'npm run format' to fix."
  exit 1
fi
```

### Husky設定
```json
// package.json
{
  "husky": {
    "hooks": {
      "pre-commit": "lint-staged",
      "commit-msg": "commitlint -E HUSKY_GIT_PARAMS",
      "pre-push": "npm test"
    }
  },
  "lint-staged": {
    "*.{js,ts,tsx}": [
      "eslint --fix",
      "prettier --write",
      "git add"
    ],
    "*.{json,md}": [
      "prettier --write",
      "git add"
    ]
  }
}
```

## GitLab/GitHub 設定

### ブランチ保護規則
```yaml
# GitHub Branch Protection Rules
protected_branches:
  main:
    required_status_checks:
      strict: true
      contexts:
        - continuous-integration
        - security-scan
    enforce_admins: true
    required_pull_request_reviews:
      required_approving_review_count: 2
      dismiss_stale_reviews: true
      require_code_owner_reviews: true
    restrictions:
      users: []
      teams: ["core-team"]
```

### CODEOWNERS ファイル
```bash
# .github/CODEOWNERS

# Global owners
* @core-team

# Frontend specific
/src/components/ @frontend-team
/src/styles/ @ui-team

# Backend specific
/src/api/ @backend-team
/src/database/ @backend-team @dba-team

# DevOps specific
/docker/ @devops-team
/.github/workflows/ @devops-team
/terraform/ @devops-team

# Documentation
/docs/ @tech-writers @core-team
README.md @core-team
```

## トラブルシューティング

### よくある問題と解決策

#### コミット履歴の修正
```bash
# 直前のコミットメッセージを修正
git commit --amend -m "fix: correct commit message"

# 複数のコミットを修正
git rebase -i HEAD~3

# 特定のファイルを前のコミットに追加
git add forgotten-file.js
git commit --amend --no-edit
```

#### 間違ったコミットの取り消し
```bash
# 直前のコミットを取り消し（変更は保持）
git reset --soft HEAD~1

# 直前のコミットを完全に取り消し
git reset --hard HEAD~1

# 特定のコミットを取り消し
git revert <commit-hash>
```

#### ブランチの復旧
```bash
# 削除したブランチの復旧
git reflog
git checkout -b recovered-branch <commit-hash>

# 失われたコミットの確認
git fsck --lost-found
```

#### 大きなファイルの処理
```bash
# Git LFS の使用
git lfs track "*.pdf"
git lfs track "*.zip"
git add .gitattributes

# 履歴から大きなファイルを削除
git filter-branch --tree-filter 'rm -f large-file.zip' HEAD
```

## ベストプラクティス

### 日常的な Git 操作
```bash
# 毎日の開始時
git checkout main
git pull origin main

# フィーチャーブランチの作成
git checkout -b feature/new-feature

# 定期的な同期
git fetch origin
git rebase origin/main

# 作業終了時
git add .
git commit -m "feat: implement user authentication"
git push origin feature/new-feature
```

### チーム協力のための規則
1. **小さなコミット**: 論理的に関連する変更をまとめる
2. **頻繁なプッシュ**: 作業の可視性を保つ
3. **明確なメッセージ**: 他の開発者が理解できるメッセージ
4. **コードレビュー**: 必ず他のメンバーのレビューを受ける
5. **テスト**: 変更前後でテストを実行する