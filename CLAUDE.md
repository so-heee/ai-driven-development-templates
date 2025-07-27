# Claude Code駆動開発ドキュメントシステム

このリポジトリは、Claude Code (claude.ai/code) を活用した効率的な開発を支援する包括的なドキュメントシステムです。

## システム概要

Claude Codeとの対話を最適化し、プロジェクト間での知識共有を促進するために設計された3層構造のドキュメントシステムです：

- **docs/**: 要件定義・変更履歴等の非技術ドキュメント
- **.claude/shared/**: 再利用可能な技術パターン・ベストプラクティス集
- **.claude/project/**: プロジェクト固有の技術情報（設定・仕様・運用手順）

## 解決する課題

### 従来の問題点
- **コンテキスト喪失**: セッション間でプロジェクト知識が失われる
- **重複作業**: 類似パターンを毎回ゼロから説明する必要性
- **一貫性の欠如**: プロジェクト間でのアプローチの統一性不足
- **学習コストの増大**: 新しいプロジェクトごとにClaude Codeに説明が必要

### このシステムの解決策
- **永続的な知識**: CLAUDE.mdと.claudeディレクトリによる知識の永続化
- **再利用可能なパターン**: shared/ディレクトリでの汎用パターンの蓄積
- **プロジェクト固有性**: project/ディレクトリでの特化した情報管理
- **即座の理解**: Claude Codeが初回アクセス時から全体像を把握

## 設計原則

### 1. 分離と再利用性

**docs/ vs .claude/ 分離の理由**
- **docs/**: 「何を作るか」の非技術ドキュメント
  - 要件定義、ユーザーストーリー、変更履歴
  - ステークホルダーとの合意事項、ビジネス要件

- **.claude/shared/**: 「どう作るか」の技術パターン
  - React コンポーネントパターン、API設計原則、DevOps のベストプラクティス
  - 複数プロジェクトで共通利用可能な技術知識

- **.claude/project/**: プロジェクト固有の技術情報
  - 特定のAPI仕様、プロジェクト固有のアーキテクチャ、開発・運用手順
  - 単一プロジェクト内でのみ有効

### 2. 段階的詳細化

**ファイル番号付けの意図**
```
01-overview.md      # 全体像・概要
02-patterns.md      # 基本パターン
03-advanced.md      # 応用・詳細
04-integration.md   # 統合・連携
05-optimization.md  # 最適化・パフォーマンス
```

**学習曲線への配慮**
- 初心者: 01-02ファイルで基本理解
- 中級者: 03-04ファイルで実践的適用
- 上級者: 05ファイルで最適化技法

### 3. 実用性重視
- 抽象的説明より具体的コード例
- すぐにコピペ可能なテンプレート
- 実際のプロジェクトで検証済みパターン

## 📁 ディレクトリ構造

```
docs/                       # 非技術ドキュメント
├── CHANGELOG.md           # 変更履歴・リリースノート
└── requirements/          # 要件定義書管理
    ├── README.md         # 管理方法・命名規則ガイド
    └── 000_機能名_YYYYMMDD.md  # 要件定義書

.claude/                   # Claude Code技術情報
├── claude-template.md      # 他プロジェクト用CLAUDE.mdテンプレート
├── settings.json          # Claude Code設定
├── shared/               # 再利用可能なパターン集
│   ├── frontend/        # React・CSS・テスト等のパターン
│   │   ├── 01-react-patterns.md    # React コンポーネント設計
│   │   ├── 02-styling-guide.md     # CSS・スタイリング戦略
│   │   ├── 03-state-management.md  # 状態管理アプローチ
│   │   ├── 04-testing-patterns.md  # テスト戦略
│   │   └── 05-performance.md       # パフォーマンス最適化
│   ├── backend/         # API・DB・認証等のパターン
│   │   ├── 01-api-patterns.md      # API 設計パターン
│   │   ├── 02-database-patterns.md # データベース設計
│   │   ├── 03-authentication.md    # 認証・認可システム
│   │   ├── 04-error-handling.md    # エラーハンドリング戦略
│   │   └── 05-logging.md           # ログ管理・監視
│   ├── devops/          # CI/CD・Docker・監視等のパターン
│   │   ├── 01-cicd-patterns.md     # CI/CD パイプライン
│   │   ├── 02-docker-patterns.md   # コンテナ化戦略
│   │   ├── 03-infrastructure.md    # インフラ構成管理
│   │   ├── 04-monitoring.md        # 監視・可観測性
│   │   └── 05-security.md          # セキュリティ対策
│   └── general/         # Git・コーディング規約等の汎用パターン
│       ├── 01-coding-standards.md  # コーディング規約
│       ├── 02-git-workflow.md      # Git ワークフロー
│       ├── 03-project-management.md # プロジェクト管理
│       ├── 04-performance.md       # 汎用パフォーマンス
│       └── 05-security.md          # 汎用セキュリティ
└── project/             # プロジェクト固有情報のテンプレート
    ├── 01-readme.md     # プロジェクト概要
    ├── 02-commands.md   # 開発コマンド
    ├── 03-architecture.md # システム構成
    ├── 04-api.md        # API仕様
    ├── 05-development.md # 開発ガイドライン
    ├── 06-deployment.md # デプロイ・運用
    └── 07-troubleshooting.md # トラブルシューティング
```

## 💡 主な特徴

### DRY原則に基づく設計
- **shared/**: 複数プロジェクトで再利用可能な知識
- **project/**: プロジェクト固有の情報のみ
- 重複を排除し、保守性を向上

### Claude Code最適化
- 初回アクセス時から全体像を即座に把握
- 実践的なコードパターンとサンプル
- トラブルシューティング情報の体系化

### 段階的学習対応
- 01-05の番号付けによる学習順序
- 概要→詳細→応用の構造
- 必要に応じた深掘り可能

## 🚀 使用ガイド

### プロジェクト開始時

#### 1. 初期セットアップ
```bash
# 新規プロジェクトでの活用
cp -r /path/to/template/.claude/ ./
cp -r /path/to/template/docs/ ./
# CLAUDE.md をプロジェクトに合わせて更新
# project/ 配下をプロジェクト固有情報で更新
```

#### 2. Claude Code での確認
```
"project/ の内容を確認して、このプロジェクトの概要を教えて"
"shared/frontend/ の React パターンを参考に、新しいコンポーネントを作成して"
```

### 要件ベース開発

#### 1. 要件定義
```bash
# 新機能の要件書作成（詳細は docs/requirements/README.md を参照）
001_user-authentication_20250202.md
```

#### 2. 要件から実装へ
```
"docs/requirements/001_user-authentication_20250202.md の要件を満たす機能を、
.claude/shared/backend/03-authentication.md のパターンを参考に実装してください"
```

### 開発フェーズでの活用

#### 実装パターンの参照
```
"shared/backend/01-api-patterns.md を参考に、ユーザー認証 API を実装して"
"shared/frontend/03-state-management.md の Zustand パターンでストアを作成"
```

#### トラブルシューティング
```
"project/07-troubleshooting.md の内容を確認して、この問題を解決して"
"shared/devops/04-monitoring.md を参考に、ログ出力を改善して"
```

### 効果的な質問パターン

#### 情報参照型
- **適切**: 「shared/backend/02-database-patterns.md を参考に...」
- **非効率**: 「データベースの設計方法を教えて」

#### 実装指示型
- **適切**: 「project/03-architecture.md の構成で、新しい機能を追加」
- **非効率**: 「何かアーキテクチャはありますか？」

#### 問題解決型
- **適切**: 「project/07-troubleshooting.md に類似事例がないか確認して」
- **非効率**: 「エラーが出ました。どうしたらいいですか？」

## 📈 運用ベストプラクティス

### 更新頻度とタイミング

#### shared/ の更新
- **頻度**: 月次または四半期
- **トリガー**: 新技術パターンの確立、既存パターンの改善発見、チーム間での知見共有
- **レビュー**: 複数プロジェクトでの検証後

#### project/ の更新
- **頻度**: 週次または機能リリース時
- **トリガー**: API 仕様変更、アーキテクチャ変更、新機能追加・既存機能修正
- **責任者**: プロジェクトリード・アーキテクト

### セッション管理

#### セッション開始時
1. **コンテキスト確認**: 「CLAUDE.md の内容を確認」
2. **プロジェクト状況把握**: 「project/ の最新状況を確認」
3. **作業対象明確化**: 「今日は〇〇機能の実装を進めたい」

#### セッション中
- **参照明示**: どのドキュメントを参考にしているか明示
- **進捗共有**: 完了した作業と次のステップを明確化
- **問題発生時**: troubleshooting.md の活用

### 品質管理

#### 内容の精度確保
```bash
# 定期的な内容検証
# 1. リンク切れチェック
find .claude -name "*.md" -exec grep -l "http" {} \\; | xargs linkchecker

# 2. 古い情報の特定
git log --since="6 months ago" .claude/shared/
```

#### 一貫性の維持
- **スタイルガイド**: Markdown 記法の統一
- **用語集**: 技術用語・略語の統一定義
- **テンプレート**: 新規ファイル作成時の雛形使用

## 🎯 クイックスタート

### このシステムを理解したい場合
1. このCLAUDE.mdを熟読してシステム全体の設計思想を理解
2. 技術パターンを探している場合は該当する shared/ 配下を確認

### 新規プロジェクトで活用したい場合
1. [.claude/claude-template.md](.claude/claude-template.md) を新プロジェクトのルートに `CLAUDE.md` として配置
2. `.claude/project/` 配下をプロジェクトに合わせて更新
3. `docs/` ディレクトリをコピーして要件管理・変更履歴を開始

### 要件管理・開発フローを活用したい場合
1. [docs/requirements/README.md](docs/requirements/README.md) で要件書の管理方法を確認
2. 要件定義書を作成して開発をスタート
3. [docs/CHANGELOG.md](docs/CHANGELOG.md) で変更履歴を管理

### 技術パターンを探している場合
- **フロントエンド**: [.claude/shared/frontend/](.claude/shared/frontend/)
- **バックエンド**: [.claude/shared/backend/](.claude/shared/backend/)
- **DevOps**: [.claude/shared/devops/](.claude/shared/devops/)
- **汎用**: [.claude/shared/general/](.claude/shared/general/)

### よくある活用シーン

#### 要件ベース新機能開発
```
「docs/requirements/001_user-authentication_20250202.md の要件を確認して、
shared/backend/03-authentication.md のパターンでユーザー認証機能を実装」
```

#### 既存機能の拡張
```
「shared/frontend/01-react-patterns.md のフォームパターンで、
project/04-api.md の仕様に合わせたユーザー登録フォームを作成」
```

#### パフォーマンス改善
```
「shared/frontend/05-performance.md と shared/backend/の最適化パターンで、
この機能のパフォーマンスを改善」
```

#### エラー対応・トラブルシューティング
```
「project/07-troubleshooting.md の類似事例を確認して、
このエラーを解決し、解決方法をドキュメントに追加」
```

#### 変更履歴・リリース管理
```
「今回の変更内容を docs/CHANGELOG.md に追加し、適切なバージョニングを行う」
```

## 🔄 継続的改善

### フィードバック収集
- **定期レトロスペクティブ**: ドキュメント活用状況
- **改善提案**: 不足パターン・使いにくい点
- **成功事例**: 効果的だった活用方法

### 成功指標
- Claude Code との対話効率向上（時間削減%）
- 新機能開発速度向上（リードタイム短縮）
- コード品質向上（レビュー指摘事項削減）
- 新人オンボーディング時間短縮

---

このシステムにより、Claude Codeとの対話効率が大幅に向上し、一貫性のある高品質な開発が実現できます。shared/パターンを活用して効率的な開発を進めてください。