# Claude Code実践ガイド

## このドキュメント管理リポジトリの理解から始める

### システム全体の理解
1. **CLAUDE.mdを熟読** - システム全体の設計思想を理解
2. **[01-overview.md](01-overview.md)** - 詳細な設計原則を確認
3. **[02-directory-structure.md](02-directory-structure.md)** - 構造を理解

### 技術パターンの探索
技術パターンを探している場合は該当する templates/shared/ 配下を確認：
- **フロントエンド**: [templates/shared/frontend/](../../templates/shared/frontend/)
- **バックエンド**: [templates/shared/backend/](../../templates/shared/backend/)
- **DevOps**: [templates/shared/devops/](../../templates/shared/devops/)
- **汎用**: [templates/shared/general/](../../templates/shared/general/)

## Claude Codeでの新規プロジェクト開始

### 1. 基本テンプレートの配置

```bash
# Claude固有設定
cp claude-templates/claude-template.md ./CLAUDE.md
cp claude-templates/settings.json ./.claude/settings.json

# 非技術ドキュメント
cp -r templates/docs/ ./docs/

# プロジェクト固有情報（カスタマイズが必要）
cp -r templates/project/ ./.claude/project/
```

### 2. 必要な技術パターンの選択・配置

```bash
# フロントエンドプロジェクトの場合
cp -r templates/shared/frontend/ ./.claude/shared/frontend/
cp -r templates/shared/general/ ./.claude/shared/general/

# Go バックエンドプロジェクトの場合
cp templates/shared/backend/01-api-design-principles.md ./.claude/shared/backend/
cp templates/shared/backend/02-database-design.md ./.claude/shared/backend/
cp templates/shared/backend/03-security-principles.md ./.claude/shared/backend/
cp -r templates/shared/backend/go/ ./.claude/shared/backend/go/
cp -r templates/shared/devops/ ./.claude/shared/devops/
cp -r templates/shared/general/ ./.claude/shared/general/

# TypeScript バックエンドプロジェクトの場合
cp templates/shared/backend/01-api-design-principles.md ./.claude/shared/backend/
cp templates/shared/backend/02-database-design.md ./.claude/shared/backend/
cp templates/shared/backend/03-security-principles.md ./.claude/shared/backend/
cp -r templates/shared/backend/typescript/ ./.claude/shared/backend/typescript/
cp -r templates/shared/devops/ ./.claude/shared/devops/
cp -r templates/shared/general/ ./.claude/shared/general/
```

### 3. プロジェクト固有のカスタマイズ
- `.claude/project/` 配下をプロジェクトに合わせて更新
- `docs/requirements/` で要件管理開始
- `docs/CHANGELOG.md` で変更履歴管理開始

### 4. Claude Codeでの初期確認
```
".claude/project/ の内容を確認して、このプロジェクトの概要を教えて"
".claude/shared/backend/01-api-design-principles.md を参考に、基本的なAPI設計を始めたい"
```

### 5. 新規プロジェクト開始チェックリスト

#### 必須作業
- [ ] `CLAUDE.md` をプロジェクトルートに配置
- [ ] `.claude/settings.json` を配置
- [ ] `docs/` ディレクトリをコピー・配置
- [ ] `.claude/project/` を配置してプロジェクト固有情報にカスタマイズ
- [ ] 必要な技術パターンを `.claude/shared/` に選択・配置

#### プロジェクト固有カスタマイズ
- [ ] `.claude/project/01-readme.md` - プロジェクト概要を記述
- [ ] `.claude/project/03-architecture.md` - システム構成を記述
- [ ] `.claude/project/04-api.md` - API仕様を記述
- [ ] `docs/requirements/` - 最初の要件定義書を作成

#### Claude Code動作確認
- [ ] Claude Codeで「CLAUDE.mdの内容を確認」を実行
- [ ] Claude Codeで「.claude/project/の概要を教えて」を実行
- [ ] 選択した技術パターンが正しく参照できることを確認

## Claude Codeでの効果的な質問パターン

### 情報参照型
- **適切**: 「.claude/shared/backend/02-database-design.md を参考に...」
- **非効率**: 「データベースの設計方法を教えて」

### 実装指示型
- **適切**: 「.claude/project/03-architecture.md の構成で、新しい機能を追加」
- **非効率**: 「何かアーキテクチャはありますか？」

### 問題解決型
- **適切**: 「.claude/project/07-troubleshooting.md に類似事例がないか確認して」
- **非効率**: 「エラーが出ました。どうしたらいいですか？」

### 言語固有実装パターン
- **Go言語**: 「.claude/shared/backend/go/04-error-handling.md のパターンでエラーハンドリングを実装」
- **TypeScript**: 「.claude/shared/backend/typescript/02-database-patterns.md の Prisma パターンでデータアクセス層を作成」

## Claude Codeでのセッション管理

### セッション開始時
1. **コンテキスト確認**: 「CLAUDE.md の内容を確認」
2. **プロジェクト状況把握**: 「.claude/project/ の最新状況を確認」
3. **作業対象明確化**: 「今日は〇〇機能の実装を進めたい」

### セッション中
- **参照明示**: どの.claude/配下のドキュメントを参考にしているか明示
- **進捗共有**: 完了した作業と次のステップを明確化
- **問題発生時**: .claude/project/07-troubleshooting.md の活用

## Claude Codeでの構造化ワークフロー

### 新機能開発の体系的フロー

#### 1. 要件確認・分析
```
"docs/requirements/ で該当する要件書を確認"
"要件に関連する templates/shared/ のパターンを特定"
```

#### 2. 設計・実装計画
```
"templates/shared/backend/01-api-design-principles.md を参考に API 設計"
"templates/shared/frontend/01-react-patterns.md を参考に UI コンポーネント設計"
```

#### 3. 実装
```
"templates/shared/backend/go/01-api-patterns.md の Gin パターンで実装"
"templates/shared/frontend/03-state-management.md の状態管理パターンで実装"
```

#### 4. テスト・品質確認
```
"templates/shared/frontend/04-testing-patterns.md のテストパターンでテスト実装"
"templates/shared/general/01-coding-standards.md の規約でコードレビュー"
```

#### 5. リリース・ドキュメント更新
```
"docs/CHANGELOG.md に変更内容を記録"
"必要に応じて templates/project/ の情報を更新"
```

### 問題解決の体系的フロー

#### 1. 問題特定
```
"templates/project/07-troubleshooting.md で類似問題を確認"
"エラーログ・症状から原因を特定"
```

#### 2. 解決策検索
```
"templates/shared/ から関連するパターンを検索"
"過去の解決事例を確認"
```

#### 3. 解決実装
```
"特定したパターンを適用して問題を解決"
"解決策をテスト・検証"
```

#### 4. 知識蓄積
```
"解決策を templates/project/07-troubleshooting.md に追加"
"再発防止策を templates/ に反映"
```

## Claude Codeでの実践的活用シーン

### 要件ベース新機能開発
```
「docs/requirements/001_user-authentication_20250202.md の要件を確認して、
.claude/shared/backend/typescript/03-authentication.md のパターンでユーザー認証機能を実装」
```

### フロントエンド開発
```
# React コンポーネント設計
「.claude/shared/frontend/01-react-patterns.md のパターンでユーザー一覧コンポーネントを作成」

# 状態管理実装
「.claude/shared/frontend/03-state-management.md の Zustand パターンでグローバル状態を管理」

# パフォーマンス最適化
「.claude/shared/frontend/05-performance.md の最適化パターンでページ表示速度を改善」
```

### バックエンドAPI開発
```
# API設計
「.claude/shared/backend/01-api-design-principles.md を参考に RESTful API を設計」

# データベース設計
「.claude/shared/backend/02-database-design.md のパターンでユーザーテーブルを設計」

# セキュリティ実装
「.claude/shared/backend/03-security-principles.md を参考に認証・認可機能を実装」
```

### 言語固有の開発

#### Go言語プロジェクト
```
「.claude/shared/backend/go/01-api-patterns.md の Gin パターンで REST API を実装」
「.claude/shared/backend/go/04-error-handling.md のエラーハンドリングパターンを適用」
「.claude/shared/backend/go/05-logging.md の logrus パターンで構造化ログを実装」
```

#### TypeScript/Node.jsプロジェクト
```
「.claude/shared/backend/typescript/01-api-patterns.md の Express パターンで API を作成」
「.claude/shared/backend/typescript/02-database-patterns.md の Prisma パターンでデータアクセス層を実装」
「.claude/shared/backend/typescript/03-authentication.md の JWT パターンで認証機能を実装」
```

### DevOps・インフラ
```
# CI/CD構築
「.claude/shared/devops/01-cicd-patterns.md の GitHub Actions パターンでデプロイパイプラインを構築」

# コンテナ化
「.claude/shared/devops/02-docker-patterns.md のマルチステージビルドパターンでDockerfileを作成」

# 監視設定
「.claude/shared/devops/04-monitoring.md のパターンでアプリケーション監視を設定」
```

### トラブルシューティング
```
# 既知の問題確認
「.claude/project/07-troubleshooting.md の内容を確認して、この問題を解決して」

# パターン検索・解決実装
「.claude/shared/devops/04-monitoring.md を参考に、ログ出力を改善して」

# 知識蓄積
「このエラーを解決し、解決方法を .claude/project/07-troubleshooting.md に追加」
```

### プロジェクト管理・品質向上
```
# コーディング規約
「.claude/shared/general/01-coding-standards.md の規約に従ってコードレビューを実施」

# Git ワークフロー
「.claude/shared/general/02-git-workflow.md の Git Flow パターンでブランチ運用を設定」

# 変更履歴管理
「今回の変更内容を docs/CHANGELOG.md に追加し、適切なバージョニングを行う」
```

## Claude Code活用の成功指標

### 効率性指標
- Claude Code との対話効率向上（時間削減%）
- Claude Codeを活用した新機能開発速度向上（リードタイム短縮）
- パターン再利用率（.claude/shared/ 参照頻度）

### 品質指標
- Claude Codeが生成するコード品質向上（レビュー指摘事項削減）
- Claude Code支援によるバグ発生率低下
- Claude Codeを活用したセキュリティ脆弱性の早期発見・修正

### 学習指標
- Claude Codeを活用した新人オンボーディング時間短縮
- Claude Codeによる技術パターンの習得速度向上
- Claude Code経由でのプロジェクト間知識伝播速度

## よくあるつまづきポイント・対処法

### 新規プロジェクト開始時

**問題**: テンプレートコピー後にClaude Codeが参照できない
- **原因**: ファイルパスが正しくない、または必要なファイルが不足
- **対処法**: 
  ```bash
  # ファイル配置を確認
  ls -la .claude/project/
  ls -la .claude/shared/
  # CLAUDE.mdがプロジェクトルートにあることを確認
  ls -la CLAUDE.md
  ```

**問題**: Claude Codeが「ファイルが見つからない」と応答する
- **原因**: 相対パスが間違っている
- **対処法**: 「pwd」コマンドで現在位置を確認し、CLAUDE.mdからの相対パスを確認

### 質問パターンの問題

**問題**: Claude Codeが期待した技術パターンを参照してくれない
- **原因**: 質問が曖昧、または参照ファイルを明示していない
- **対処法**: 
  ```
  # 悪い例
  「API設計のベストプラクティスを教えて」
  
  # 良い例
  「.claude/shared/backend/01-api-design-principles.md を参考にユーザー認証APIを設計して」
  ```

**問題**: Claude Codeが古い情報を参照している
- **原因**: セッション開始時にコンテキストを更新していない
- **対処法**: セッション開始時に「CLAUDE.mdの内容を確認して最新の情報を把握して」を実行

### テンプレート活用の問題

**問題**: Go/TypeScriptのパターンが混在してしまう
- **原因**: 不要な言語パターンもコピーしている
- **対処法**: プロジェクトで使用する言語のパターンのみを選択的にコピー

**問題**: プロジェクト固有情報が更新されていない
- **原因**: templates/project/をそのまま使用している
- **対処法**: .claude/project/配下の全ファイルを実際のプロジェクトに合わせて更新

### 実装時の問題

**問題**: 生成されたコードが期待と異なる
- **原因**: プロジェクト固有の制約や設計が伝わっていない
- **対処法**: .claude/project/03-architecture.md や 05-development.md に設計制約を詳細に記述

**問題**: エラーが解決できない
- **原因**: トラブルシューティング情報が不足
- **対処法**: 
  1. .claude/project/07-troubleshooting.md を確認
  2. 解決策が見つからない場合は、解決後に情報を追加