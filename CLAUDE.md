# AI駆動開発ドキュメント管理リポジトリ

このリポジトリは、AI駆動開発を支援するドキュメント管理リポジトリです。主にClaude Codeでの運用を想定し、Claude Codeのコンテキストとして取り込みやすい形式でテンプレートを提供します。

## リポジトリの指針

### 設計原則

- **Claude Code最適化**: Claude Codeのコンテキストとして効率的に動作する形式
- **再利用性重視**: プロジェクト間での知識共有と技術パターンの再利用促進
- **識別性重視**: 番号付けにより開発者がファイルを識別しやすい構造
- **実用性重視**: 実際のプロジェクトで検証済みのパターンのみ提供

### 運用制約

- **言語固有パターン**: Go・TypeScript の実装パターンに特化
- **Claude Code特化**: 他のAI開発支援ツールでの活用は現在対象外
- **コンテキスト最適化**: 各プロジェクトで必要なパターンのみを選択・配置
- **アイコン使用禁止**: ドキュメント全体でアイコン・絵文字の使用を禁止
- **ドキュメント整合性維持**: ファイル変更時は関連するドキュメント（CLAUDE.md、.claude/、claude-templates/、templates/）の整合性を必ず保つ

## ドキュメントガイド

このリポジトリの詳細な使い方は、以下のドキュメントを参照してください：

### システム理解

- **[概要](.claude/context/01-overview.md)** - 設計思想・解決課題・主な特徴
- **[ディレクトリ構造](.claude/context/02-directory-structure.md)** - 構造の全体像と各ディレクトリの役割

### 実践ガイド

- **[Claude Code実践ガイド](.claude/context/03-claudecode-getting-started.md)** - 新規プロジェクト開始から実践的活用まで包括的ガイド

### 運用管理

- **[テンプレートメンテナンス](.claude/workflows/01-template-maintenance.md)** - Claude Codeによる機械的メンテナンス運用
- **[改善計画・管理](.claude/workflows/02-improvement-management.md)** - リポジトリの継続的改善・プロジェクト管理

## 提供テンプレート

- **[templates/](templates/)** - プロジェクト用テンプレート（shared/project/docs）
- **[claude-templates/](claude-templates/)** - Claude固有テンプレート（CLAUDE.md・settings.json）

## リポジトリ管理

詳細な改善計画・運用手順は[.claude/workflows/](.claude/workflows/)を参照してください。

---

**このドキュメント管理リポジトリにより、Claude Code との対話効率が大幅に向上し、一貫性のある高品質な開発が実現できます。**
