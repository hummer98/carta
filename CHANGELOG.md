# Changelog

All notable changes to this project will be documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.2.0] - 2026-05-11

### Added
- プロファイル機能: `assets/profiles/<name>.md` をデフォルト憲法に連結適用できる。
- `assets/profiles/typescript.md` / `assets/profiles/flutter.md` の最小プレースホルダを同梱。
- `/carta:apply --profile <names>` で複数プロファイルをカンマ区切り指定可能（例: `--profile typescript,flutter`）。アルファベット順に正規化されてマーカーに記録される。
- `/carta:show --profile <name>` で単一プロファイル本文の表示に対応。
- `/carta:diff` も `--profile` 引数を受け取り、連結後本文の差分を表示。

### Changed
- マーカー形式に `profiles=` 属性を追加: `<!-- carta:begin v0.2.0 sha=abc1234 profiles=flutter,typescript -->`。
- `sha` は **連結後本文** から計算するように変更（プロファイル組み合わせ変更が更新として検出される）。連結ロジックは「profiles 空時は constitution.md と完全に同じバイト列」になるよう設計され、v0.1.0 と完全互換の sha を生成する。
- `/carta:apply` を引数なしで実行した場合、既存マーカーに記録された profiles を引き継いで再適用するようになった。

### Compatibility
- v0.1.0 マーカー（`profiles=` 属性なし）は「プロファイルなし」として完全互換動作する。
- v0.1.0 で適用済みの CLAUDE.md に v0.2.0 を再 apply しても、profiles が空のままなら sha・本文ともに変化しない（version 文字列のみ `v0.1.0` → `v0.2.0` に更新）。実機検証用の sha 期待値: `f0c10c8`。
- プロファイル本文の先頭マーカーコメント `<!-- carta-profile: NAME -->` は **凍結された定数**。書式を変更すると既存 CLAUDE.md の sha が破壊されるので、リリース後は変えない。
- `assets/constitution.md` および `assets/profiles/*.md` は **末尾改行 1 個** で保存する運用ルール。

### Notes
- `--list-profiles` は提供しない（プラグインの `assets/profiles/` を直接見れば足りる）。
- プロファイル間の依存関係・自動検出・プロファイル別 sha 検証はサポートしない。
- `<release-date>` プレースホルダは `release` コマンド運用（`.claude/commands/release.md` 参照）で書き換える。Implementer は `<YYYY-MM-DD>` のままコミットしてよい。
- 空指定の推奨形式は `--profile=`（イコール接続）。`--profile ""` は環境依存のため非推奨。
- `/carta:show --profile=` は name 必須エラー（apply の `--profile=` 全外しと意図的に非対称）。

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
