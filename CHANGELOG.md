# Changelog

All notable changes to this project will be documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.3.0] - 2026-05-12

### Added
- `/carta:help` — プラグインの全コマンド・典型ワークフロー・マーカー仕様の要点を 1 画面で表示する表示専用コマンド。
- `/carta:migrate` — マーカー外に残った憲法重複記述を検出して削除候補を提示するコマンド（carta 導入直後の掃除用、デフォルト dry-run、`--include-medium` で MEDIUM 候補も含む、`--apply` で実適用）。
- `/carta:list` — バンドル済みプロファイルの一覧（名前 + 1 行要約）を表示するコマンド。実体は `bin/carta-list` POSIX shell スクリプト（Claude を介さず直接実行されるため高速）。
- `assets/profiles/interactive.md` — 対話セッション専用ルール（ターン末尾サマリ規約 / UI dev server 実機確認）をプロファイルとして分離。elevens / OpenClaw / AutoClaude 等の全自動オーケストレーター配下では本プロファイルを外す。
- `<!-- carta-summary: ... -->` 任意フィールド — プロファイル本文 2 行目に置くと `/carta:list` の description として表示される（凍結対象外、§15.8）。
- `bin/` ディレクトリ — seed.md §7 で計画されていた shell スクリプト置き場が `bin/carta-list` で実体化。
- 憲法 §4「可観測性とロギング」— 言語非依存の観測原則（空 `catch` 禁止、外部コマンド失敗時に `stderr` / `stdout` を含める、機密情報をログに含めない、高頻度ループでのノイズ抑制）。

### Changed
- 憲法本体 `assets/constitution.md` を Opus 4.7 のシステムプロンプトと整合する形に整理:
  - 旧 §2 コード品質 / §4 Git ワークフロー / §5 AI 協業の心得 を削除（システムプロンプトと完全重複）
  - 旧 §1 のサマリ規約「サマリを書かない」→「1–2 文に収める」に変更（システムプロンプトの End-of-turn summary と整合）
  - 旧 §3 テスト・§5 から汎用部分を整理して §2「テストと完了基準」に集約
  - §3「ツール運用」を新設（独立 tool call の並列発行ルール）
- メタコメントに `note:` 行を追加し、対話前提ルールが `interactive` プロファイルに分離されたことを明示。
- `docs/seed.md` §15.7「凍結された定数」を「プロファイル 1 行目のみ凍結」に明確化、§15.8 を新設して `carta-summary` 仕様を文書化。

### Compatibility
- v0.2.0 のマーカー形式（`profiles=` 属性あり / なし）は引き続き互換動作する。
- 憲法本体 sha は変化する（profiles 空時の v0.2.0 値 `f0c10c8` → v0.3.0 値 `8a35bff`、interactive 連結時は `79ca94c`）。既存 CLAUDE.md は `/carta:apply` 再実行で同期される。
- v0.2.0 の不変条件「profiles 空時の連結結果は `cat assets/constitution.md` と byte-for-byte 一致」は維持される。
- プロファイル 1 行目 `<!-- carta-profile: NAME -->` は引き続き凍結。2 行目 `<!-- carta-summary: ... -->` は任意・凍結対象外。

### Notes
- `--list-profiles` は引き続き提供しない（`/carta:list` が独立コマンドとして担当）。
- `/carta:migrate` は **削除のみ**で、マーカー外への新規追記は行わない（責務分離）。`--force` は受け付けない（apply 側で先に同期する）。
- `/carta:list` の出力は `assets/profiles/*.md` をアルファベット順に走査し、description は `carta-summary` → 最初の H2 見出しの優先順で抽出する。

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
