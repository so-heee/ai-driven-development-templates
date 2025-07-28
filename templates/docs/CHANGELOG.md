# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- 新機能の追加項目をここに記載

### Changed

- 既存機能の変更項目をここに記載

### Deprecated

- 廃止予定の機能をここに記載

### Removed

- 削除された機能をここに記載

### Fixed

- バグ修正項目をここに記載

### Security

- セキュリティ関連の変更をここに記載

## [1.0.0] - 2025-01-27

### Added

- Claude Code駆動開発ドキュメントシステムの初期リリース
- `.claude/shared/` - 再利用可能な技術パターン集
  - frontend/ - React・CSS・テスト等のパターン（5ファイル）
  - backend/ - API・DB・認証等のパターン（5ファイル）
  - devops/ - CI/CD・Docker・監視等のパターン（5ファイル）
  - general/ - Git・コーディング規約等の汎用パターン（5ファイル）
- `.claude/project/` - プロジェクト固有情報テンプレート（7ファイル）
- `.claude/claude-template.md` - 他プロジェクト用CLAUDE.mdテンプレート
- `docs/requirements/` - 要件定義書管理システム
- `docs/CHANGELOG.md` - 変更履歴管理

### Changed

- CLAUDE.mdを単一ファイルでシステム全体を説明する構成に変更

### Security

- 認証・認可パターンの実装ガイドライン追加
- セキュリティベストプラクティス文書化

---

## 使用方法

### バージョニング規則

- **Major (X.0.0)**: 破壊的変更、アーキテクチャの大幅変更
- **Minor (X.Y.0)**: 新機能追加、既存機能の拡張
- **Patch (X.Y.Z)**: バグ修正、軽微な改善

### 変更カテゴリ

- **Added**: 新機能
- **Changed**: 既存機能の変更
- **Deprecated**: 廃止予定機能（将来のバージョンで削除予定）
- **Removed**: 削除された機能
- **Fixed**: バグ修正
- **Security**: セキュリティ関連の変更

### 記載例

```markdown
## [1.1.0] - 2025-02-15

### Added

- ユーザー認証機能の実装
- JWT トークンベースの認証システム
- パスワードリセット機能

### Changed

- API レスポンス形式の統一
- データベーススキーマの最適化

### Fixed

- ログイン時のセッション管理バグ修正
- メール送信の非同期処理改善

### Security

- パスワードハッシュ化アルゴリズムの強化
- CSRF対策の実装
```

### リリース手順

1. 変更内容を該当バージョンに記載
2. `[Unreleased]` セクションから該当バージョンに移動
3. リリース日を追加
4. Git タグを作成: `git tag v1.1.0`
5. GitHub Releases で公開

### 注意事項

- 変更は発生時に即座にUnreleasedセクションに記載
- リリース時にバージョン番号と日付を確定
- 破壊的変更は必ずMajorバージョンアップ
- セキュリティ修正は優先的にPatchリリース
