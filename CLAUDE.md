# carta 開発ガイド

統一的な開発憲法 (constitution) を保持し、各プロジェクトの `CLAUDE.md` に適用・同期する Claude Code Plugin。

## まずこれを読む

**`docs/seed.md` がプロジェクト構想の一次情報源**。スキル・コマンド・憲法本体の設計はすべて seed に記述してある。実装作業に入る前に必ず seed を Read すること。

## ファイル構成

| ファイル / ディレクトリ | 役割 |
|------------------------|------|
| `docs/seed.md` | プロジェクト構想・設計の一次情報（ソースオブトゥルース） |
| `assets/constitution.md` | 憲法本体テキスト（全プロジェクト共通の開発原則） |
| `skills/carta/SKILL.md` | carta スキル定義（Claude が読む） |
| `commands/apply.md` | `/carta:apply` — CLAUDE.md に憲法を適用 |
| `commands/diff.md` | `/carta:diff` — 適用差分のプレビュー |
| `commands/show.md` | `/carta:show` — 現在の憲法を表示 |
| `.claude-plugin/plugin.json` | Plugin マニフェスト |
| `.claude-plugin/marketplace.json` | Marketplace カタログ |
| `.claude/commands/release.md` | このリポジトリ自身のリリース手順 |
| `README.md` | 人間向けガイド |
| `LICENSE` | MIT |

## 言語ルール

- **ドキュメント・コメント**: 日本語
- **コード（変数名・関数名・コマンド）**: 英語

## 設計原則

- **憲法はテキストファイル 1 枚** (`assets/constitution.md`)。複雑なテンプレートエンジンは入れない
- **CLAUDE.md への適用は idempotent**。マーカーで囲った領域だけを置換し、プロジェクト固有部分は保護する
- **バージョン管理**: 適用済みバージョンを CLAUDE.md 内のメタデータコメントに記録し、新バージョン検出時に更新提案する

## 配布

GitHub の tag + marketplace 経由で配布する。npm publish はしない。リリース手順は `.claude/commands/release.md` を参照。
