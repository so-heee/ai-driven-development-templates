# Gitガイドライン

このドキュメントは、ai-driven-development-templatesリポジトリでのGit運用ガイドラインを定義します。

## ブランチ戦略

### 基本原則

**ファイル修正前に必ずユーザーにブランチ作成を指示する**

- ファイル修正依頼を受けた際は、作業前にブランチ作成をユーザーに要求
- mainブランチでの直接ファイル修正は禁止
- ユーザーがブランチ作成を完了してから作業開始
- すべての変更作業は専用ブランチで実施

### ブランチ命名規則

[Conventional Branch](https://conventional-branch.github.io/)に従います：

- `feature/[FeatureName]-[実装した機能名]`
- `fix/[IssueDescription]-[修正内容]`
- `docs/[DocumentName]-[更新内容]`
- `refactor/[ComponentName]-[リファクタリング内容]`

### ブランチ作成手順

```bash
# 1. mainブランチに移動
git checkout main
git pull origin main

# 2. 作業ブランチ作成（ファイル修正前に必須）
git checkout -b feature/example-add-new-template

# 3. この時点でファイル編集開始
```

## コミットメッセージ

### Conventional Commits

すべてのコミットメッセージは[Conventional Commits](https://www.conventionalcommits.org/)に従います：

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

### コミットタイプ

- `feat`: 新機能
- `fix`: バグ修正
- `docs`: ドキュメントのみの変更
- `style`: コードの意味に影響しない変更（空白、フォーマットなど）
- `refactor`: バグ修正や機能追加以外のコード変更
- `test`: テストの追加・修正
- `chore`: ビルドプロセスや補助ツールの変更

## 品質管理システム

### lefthook自動化

このリポジトリではlefthookによるGitフック自動化を実装：

#### pre-commitフック

2段階処理でMarkdown品質を確保：

1. **prettier**: Markdownファイルの自動フォーマット
2. **markdownlint-cli2**: フォーマット後のリント検証

```yaml
# lefthook.yml
pre-commit:
  parallel: false
  commands:
    markdown-format:
      glob: '*.md'
      run: npx prettier --write {staged_files}
      stage_fixed: true
    markdown-lint:
      glob: '*.md'
      run: npx markdownlint-cli2 {staged_files}
      stage_fixed: true
```

#### commit-msgフック

Conventional Commits形式の自動検証

### markdownlint-cli2設定

技術ドキュメント向けに最適化された設定（`.markdownlint-cli2.jsonc`）：

```jsonc
{
  "config": {
    "default": true,
    "MD013": false, // 行長制限なし
    "MD031": { "list_items": false }, // リスト内コードブロック緩和
    "MD033": false, // インラインHTML許可
    "MD041": false, // H1必須要件緩和
  },
}
```

### prettier設定

Markdownフォーマット設定（`.prettierrc`）：

```json
{
  "proseWrap": "preserve",
  "tabWidth": 2,
  "useTabs": false
}
```

### npmスクリプト活用

```bash
# 個別実行
npm run format:md  # prettierフォーマット
npm run lint:md    # markdownlintチェック
npm run fix:md     # フォーマット→リント実行

# lefthook導入
npm run prepare    # lefthook install実行
```

## プルリクエスト

### 作成手順

```bash
# 1. ブランチをリモートにプッシュ
git push -u origin <branch_name>

# 2. Draft PRとして作成
gh pr create --draft --title "title" --body "body"
```

### 品質チェックワークフロー

1. **開発中**: `npm run fix:md`で手動品質管理
2. **コミット時**: lefthookによる自動品質チェック
3. **PR作成前**: 最終品質確認
4. **マージ前**: すべてのチェック通過確認

## 禁止事項

### 絶対禁止

- **ブランチ作成指示なしでのファイル修正**: ユーザーにブランチ作成を指示せずにファイル変更作業を開始すること
- **mainブランチでの直接ファイル修正**: ブランチ作成前にmainで変更作業を行うこと
- **指示なしでのGit操作**: ユーザーからの明確な指示なしにGit操作（commit、push、PR作成等）を実行すること
- **node_modules/のコミット**: 依存関係ファイルのバージョン管理対象化
- **個人設定ファイルのコミット**: `.claude/settings.local.json`等の個人設定

### 品質要件

- lefthookチェック未通過でのコミット
- markdownlint-cli2エラー未解決でのPR作成
- Conventional Commits形式違反

## ワークフロー例

### 標準開発フロー

#### ステップ1: ブランチ作成指示（ユーザーに要求）

ファイル修正依頼を受けた場合、まずユーザーに以下を指示：

```bash
# まず以下のコマンドでブランチを作成してください：
git checkout main
git pull origin main
git checkout -b feature/new-template
```

#### ステップ2: ブランチ作成確認後の作業開始

ユーザーがブランチ作成を完了したことを確認してから：

```bash
# ユーザーがブランチ作成完了後、以下の作業を実行：

# 1. ファイル編集作業
# ファイル修正開始

# 2. 品質チェック
npm run fix:md

# 3. コミット（lefthookによる自動チェック実行）
git add .
git commit -m "feat(templates): add new template"

# 4. プッシュ・PR作成（ユーザー指示があった場合のみ）
git push -u origin feature/new-template
gh pr create --draft --title "feat: 新テンプレート追加"
```

### 品質問題対応

```bash
# markdownlintエラー時
npm run lint:md     # エラー内容確認
npm run format:md   # prettier自動修正
npm run lint:md     # 再確認

# lefthookエラー時
lefthook run pre-commit  # 手動実行で詳細確認
npm run fix:md          # 品質修正
git add .               # 修正内容ステージング
git commit -m "..."     # 再コミット
```

---

このガイドラインにより、自動化された品質管理と適切なGit運用が実現されます。
