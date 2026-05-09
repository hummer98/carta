# Changelog

All notable changes to this project will be documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.0] - 2026-05-10

### Added
- 開発憲法本体 `assets/constitution.md`（5 章 31 項目: コミュニケーション / コード品質 / テスト / Git ワークフロー / AI 協業の心得）
- `/carta:apply` — マーカー方式で対象 `CLAUDE.md` に憲法を適用・同期するスラッシュコマンド。idempotent 保証、マーカー外保護、書き込み前 diff 承認、sha 不一致検出、`--force` 上書きをサポート。
- `/carta:show` — バンドルされている憲法本体を表示する。`--meta` オプションで version / last-updated / maintainer ヘッダ併記。
- `/carta:diff` — `/carta:apply` の dry-run。差分のみ表示し書き込まない。
- `skills/carta/SKILL.md` — Claude が読むスキル定義。トリガー条件、マーカー仕様、適用フロー、エッジケースを記載。
- `README.md` / `README.ja.md` — 英日両対応のユーザーガイド。
- `.claude/commands/release.md` — リポジトリ自身のリリース手順（version bump → tag → GitHub Release → ローカル plugin cache 追従）。

### Notes
- v0.1.0 のスコープ外: 複数 `CLAUDE.md` mono-repo 対応、プロジェクト別 override レイヤ、CI 検出、章選択適用。詳細は `docs/seed.md` §10。
- マーカーフォーマット: `<!-- carta:begin v0.1.0 sha=abc1234 -->` ～ `<!-- carta:end -->`（sha は SHA-256 短縮 7 文字）。
