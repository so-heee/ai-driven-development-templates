# CLAUDE.md

このファイルは、このリポジトリでコードを扱う際にClaude Code (claude.ai/code) へのガイダンスを提供します。

## プロジェクト情報

このプロジェクトの詳細情報は以下を参照してください：

- **プロジェクト概要**: `.claude/project/01-readme.md`
- **開発コマンド**: `.claude/project/02-commands.md`
- **アーキテクチャ**: `.claude/project/03-architecture.md`

## 開発ガイド

技術領域別の開発ガイドラインは以下を参照してください：

- **フロントエンド**: `.claude/shared/frontend/`
- **バックエンド**: `.claude/shared/backend/`
- **DevOps**: `.claude/shared/devops/`
- **汎用ガイド**: `.claude/shared/general/`

## Claude Code設定

Claude Codeの動作設定は `.claude/settings.json` で管理されています。

## ドキュメント構造

全てのプロジェクト情報は `.claude/` ディレクトリ配下に統合されています：

```
.claude/
├── settings.json         # Claude Code設定
├── shared/              # プロジェクト横断で再利用可能なガイド
│   ├── frontend/        # フロントエンド開発パターン
│   ├── backend/         # バックエンド開発パターン
│   ├── devops/          # DevOps関連パターン
│   └── general/         # 汎用開発ガイド
└── project/             # このプロジェクト固有の情報
    ├── 01-readme.md     # プロジェクト概要
    ├── 02-commands.md   # 開発コマンド
    ├── 03-architecture.md # アーキテクチャ
    ├── 04-api.md        # API仕様
    ├── 05-development.md # 開発ガイドライン
    ├── 06-deployment.md # デプロイメント
    └── 07-troubleshooting.md # トラブルシューティング
```
