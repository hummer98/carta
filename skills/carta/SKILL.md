---
name: carta
description: "全プロジェクト共通の開発憲法 (constitution) を CLAUDE.md に適用・同期するスキル。ユーザーが「carta」「憲法適用」「constitution 適用」「CLAUDE.md 同期」「開発原則を適用」「プロジェクトに principles を反映」「全プロジェクト共通ルール」「プロファイル適用」「typescript / flutter プロファイル」に言及したとき、または新規プロジェクトで CLAUDE.md を初めて作成しようとしているとき、または別リポジトリの CLAUDE.md にも同じ原則を反映したいとき、または既に carta 適用済みのリポジトリで「最新版に更新」と求められたときに使用する。"
---

# carta スキル

carta は **全プロジェクト共通で適用したい開発原則（憲法）** を 1 箇所で管理し、各リポジトリの `CLAUDE.md` に同期する Claude Code Plugin。v0.2.0 から **プロファイル機能**（言語/フレームワーク別の追加コンテンツを連結適用）に対応する。

## いつ使うか

- ユーザーが「憲法を適用」「CLAUDE.md を同期」「carta」「開発原則をプロジェクトに反映」等に言及したとき
- 新規プロジェクトで CLAUDE.md を初めて作成しようとしているとき
- プロジェクト間で原則の整合性を取りたいとき
- 既に carta が適用されたリポジトリで「最新版に更新」と求められたとき
- 言語/フレームワーク別のプロファイル（typescript / flutter 等）を連結適用したいとき
- carta 導入直後の CLAUDE.md からマーカー外に残った重複記述を整理したいとき（`/carta:migrate`）

## 提供するスラッシュコマンド

| コマンド | 役割 |
|---|---|
| `/carta:apply [<path>] [--profile <names>] [--force]` | 対象 CLAUDE.md に最新の憲法（+ 指定プロファイル）を適用 / 同期する |
| `/carta:show [--meta] [--profile <name>]` | 現在バンドルされている憲法本体（または指定プロファイル本文）を表示する |
| `/carta:diff [<path>] [--profile <names>]` | apply の dry-run。差分のみ表示し書き込まない |
| `/carta:migrate [<path>] [--profile <names>] [--include-medium] [--apply]` | マーカー外から憲法と意味的に重複する記述を検出し、削除候補を提示する（carta 導入直後の掃除用。デフォルト dry-run） |
| `/carta:help` | プラグインの全コマンド・使い方・マーカー仕様の要点を表示する |

## マーカー仕様

CLAUDE.md 内に以下のマーカーで囲まれた領域だけを carta が管理する:

```markdown
<!-- carta:begin v0.2.0 sha=abc1234 profiles=flutter,typescript -->
（憲法本体 + 指定プロファイル本文の連結結果がここに展開される）
<!-- carta:end -->
```

- `v0.2.0` — 適用された carta 本体のバージョン
- `sha=abc1234` — **連結後本文** の SHA-256 短縮 hash（先頭 7 文字）。マーカー内が手で改変されていないかを検証する目印
- `profiles=flutter,typescript` — 適用済みプロファイル名のアルファベット順カンマ区切り。**プロファイル指定なし時は属性自体を省略する**（v0.1.0 マーカーと完全同形）
- マーカー外のテキストは絶対に変更しない（プロジェクト固有の規約は外側に書く）

マーカー検出/解析の正規表現（apply / diff 共通）:

```
^<!-- carta:begin v[^ ]+ sha=[0-9a-f]+( profiles=[A-Za-z0-9_,-]+)? -->$
^<!-- carta:end -->$
```

`profiles=` 属性が無い v0.1.0 形式のマーカーは「プロファイルなし」として完全互換動作する。

## プロファイル

特定の言語/フレームワーク向けの追加ルールを「プロファイル」として連結適用できる。

- 利用可能なプロファイルは `assets/profiles/*.md`（v0.2.0 では typescript / flutter を最小プレースホルダで同梱）
- `/carta:apply --profile typescript,flutter` のようにカンマ区切りで複数指定可
- 連結順序はアルファベット順に正規化される。マーカーの `profiles=` 属性にも正規化済みの値が記録される
- `/carta:apply`（引数なし）は **既存マーカーの profiles を引き継いで** 再適用する
- `/carta:apply --profile=` でプロファイルなし（デフォルト憲法のみ）に明示的に戻す（推奨形式）
- 存在しないプロファイル名を指定するとエラーで停止する（自動作成しない）
- v0.1.0 マーカー（`profiles=` 属性なし）はプロファイルなしとして互換動作する
- **show は非対称**: `/carta:show --profile=` は name 必須エラー。デフォルト憲法を見たい場合は `/carta:show`（引数なし）を使う

### 連結フォーマット

```
{デフォルト本文}

---

{プロファイル本文 (アルファベット順)}
```

区切りは「空行 + 水平線 + 空行」。各 part は末尾改行 1 個に正規化された上で連結される。`sha=` は連結後本文（末尾改行 1 個含む）で計算する。profiles 空時は `assets/constitution.md` と byte-for-byte 完全一致するため、v0.1.0 と sha も一致する。

## 適用フロー（`/carta:apply` の安全要件）

1. 書き込み前に必ず diff を提示してユーザー承認を得る（一発適用しない）
2. 既存マーカーが見つかった場合:
   - version + sha + profiles + 本文がすべて一致 → 何もしない（idempotent）
   - 不一致 → 新版で置換し、マーカーの version / sha / profiles を更新
3. マーカーが破損している（begin だけ / end だけ / 順序逆 / 複数）→ エラー停止し、ユーザーに手動修復を促す（自動修復しない）
4. マーカー内の本文が手で編集された痕跡（sha 不一致）→ 警告を出す。`--force` 付きでのみ上書きを許可
5. マーカーが無い既存 CLAUDE.md → ファイル冒頭にマーカーブロックを挿入。既存内容は不変
6. CLAUDE.md が無い → 新規作成し、マーカーブロックの後ろに `## プロジェクト固有` の空セクションを置く

## してはいけないこと

- マーカー外のテキストを 1 byte でも書き換える
- diff レビュー無しで書き込む
- マーカーが破損していたら自動で修復する
- 適用済みバージョンが新しいのに古い版を強制的に上書きする（明示の `--force` 無しでは）
- profiles の順序を手で並べ替える（必ずアルファベット順で書き出す）
- 連結区切り（part 末尾 LF + `\n---\n\n`）を別の文字列に変える
- プロファイル本文の先頭マーカーコメント `<!-- carta-profile: NAME -->` の書式を変更する（**凍結された定数**）
- 「sha 不一致でも body 一致なら idempotent」フォールバックを入れる（手編集検出の安全性が壊れる）
- `/carta:migrate` がマーカー外への新規追記をする（migrate は **削除のみ**。言い換え保持はしない）
- `/carta:migrate` で LOW 確信度の候補を自動削除する（LOW はいかなる場合も削除候補にならない。参考表示のみ）
- `/carta:migrate` で MEDIUM 候補を `--include-medium` 無しに削除候補へ昇格させる（デフォルトは HIGH のみ）

## バージョン同期

- 適用済みバージョンはマーカーの `v0.2.0` 部分に記録される
- カレントの plugin version（`.claude-plugin/plugin.json` の `version`）と異なれば「更新提案」を出す
- ユーザーが受諾したら `/carta:apply` を実行して同期する

## 関連ドキュメント

- 設計の一次情報源: `docs/seed.md`
- 憲法本体: `assets/constitution.md`
- プロファイル: `assets/profiles/*.md`
- リリース手順: `.claude/commands/release.md`（メンテナ向け）
