# ディレクトリ構造

## ドキュメント管理リポジトリの全体像

```
templates/                 # プロジェクト用テンプレート
├── docs/                  # 非技術ドキュメントテンプレート
│   ├── CHANGELOG.md      # 変更履歴・リリースノートテンプレート
│   └── requirements/     # 要件定義書管理テンプレート
│       ├── README.md     # 管理方法・命名規則ガイド
│       └── 000_機能名_YYYYMMDD.md  # 要件定義書テンプレート
├── shared/               # 再利用可能なパターン集
│   ├── frontend/        # React・CSS・テスト等のパターン
│   │   ├── 01-react-patterns.md    # React コンポーネント設計
│   │   ├── 02-styling-guide.md     # CSS・スタイリング戦略
│   │   ├── 03-state-management.md  # 状態管理アプローチ
│   │   ├── 04-testing-patterns.md  # テスト戦略
│   │   └── 05-performance.md       # パフォーマンス最適化
│   ├── backend/         # API・DB・認証等のパターン
│   │   ├── 01-api-design-principles.md    # API設計原則
│   │   ├── 02-database-design.md          # データベース設計
│   │   ├── 03-security-principles.md      # セキュリティ原則
│   │   ├── go/          # Go言語固有実装
│   │   │   ├── 01-coding-guidelines.md  # Goコーディング規約
│   │   │   ├── 02-api-patterns.md       # API実装パターン
│   │   │   ├── 03-database-patterns.md  # データベースパターン
│   │   │   ├── 04-authentication.md     # 認証実装
│   │   │   ├── 05-error-handling.md     # エラーハンドリング
│   │   │   └── 06-logging.md            # ログ実装
│   │   └── typescript/  # TypeScript固有実装
│   │       ├── 01-coding-guidelines.md  # TypeScriptコーディング規約
│   │       ├── 02-api-patterns.md       # API実装パターン
│   │       ├── 03-database-patterns.md  # データベースパターン
│   │       ├── 04-authentication.md     # 認証実装
│   │       ├── 05-error-handling.md     # エラーハンドリング
│   │       └── 06-logging.md            # ログ実装
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
└── project/             # プロジェクトテンプレート
    ├── 01-readme.md     # プロジェクト概要
    ├── 02-commands.md   # 開発コマンド
    ├── 03-architecture.md # システム構成
    ├── 04-api.md        # API仕様
    ├── 05-development.md # 開発ガイドライン
    ├── 06-deployment.md # デプロイ・運用
    └── 07-troubleshooting.md # トラブルシューティング

claude-templates/           # Claude固有テンプレート
├── claude-template.md     # 他プロジェクト用CLAUDE.mdテンプレート
└── settings.json          # Claude Code設定テンプレート

.claude/                   # このドキュメント管理リポジトリ自体のコンテキスト
├── context/              # ドキュメント管理リポジトリの説明・ガイド
│   ├── 01-overview.md                  # ドキュメント管理リポジトリ概要・設計原則
│   ├── 02-directory-structure.md      # ディレクトリ構造説明
│   └── 03-claudecode-getting-started.md # Claude Code実践ガイド
├── workflows/            # リポジトリの運用・管理手順
│   ├── 01-template-maintenance.md  # テンプレートメンテナンス運用
│   └── 02-improvement-management.md # 改善計画・プロジェクト管理
└── settings.json         # このドキュメント管理リポジトリ用設定
```

## 各ディレクトリの役割

### templates/docs/ - 非技術ドキュメントテンプレート
- **目的**: 「何を作るか」のテンプレート提供
- **対象**: 要件定義、変更履歴、ビジネス仕様のテンプレート
- **使用方法**: 各プロジェクトのdocs/ディレクトリにコピー

### templates/shared/ - 技術パターンライブラリ
- **目的**: 技術パターンの再利用促進
- **対象**: フロントエンド・バックエンド・DevOps・汎用パターン
- **使用方法**: 必要なパターンを選択して各プロジェクトの.claude/shared/にコピー

### templates/project/ - プロジェクト固有情報テンプレート
- **目的**: プロジェクト固有情報のテンプレート提供
- **対象**: アーキテクチャ、API仕様、運用手順のテンプレート
- **使用方法**: 各プロジェクトの.claude/project/にコピーしてカスタマイズ

### claude-templates/ - Claude固有テンプレート
- **目的**: Claude Code専用テンプレートの提供
- **対象**: CLAUDE.md、settings.json等のClaude固有設定
- **使用方法**: 各プロジェクトにコピーして使用

### .claude/ - このドキュメント管理リポジトリのコンテキスト

#### context/ - ドキュメント管理リポジトリの理解・活用ガイド
- **目的**: ドキュメント管理リポジトリの概念的理解・Claude Code活用方法
- **対象**: システム設計、ディレクトリ構造、Claude Code実践ガイド
- **注意**: 他プロジェクトでは使用しない

#### workflows/ - リポジトリの運用・管理手順
- **目的**: 実際の運用手順・プロジェクト管理
- **対象**: テンプレートメンテナンス、改善計画管理
- **注意**: 他プロジェクトでは使用しない

## Claude Codeでのコンテキスト最適化

**ドキュメント管理リポジトリの利点：**
- **templates/**: Claude Codeで必要な技術パターンのみを選択・配置可能
- **claude-templates/**: 各プロジェクトでClaude Code用に適切にカスタマイズ可能
- **各プロジェクトの.claude/**: プロジェクト固有情報のみでClaude Codeのコンテキスト最適化

**Claude Codeでの活用プロセス：**
1. templates/から必要なパターンを選択
2. 各プロジェクトの.claude/にコピー・配置
3. プロジェクト固有にカスタマイズ
4. Claude Codeがプロジェクト最適化されたコンテキストで動作