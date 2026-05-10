# carta — プロジェクト構想 (seed)

> このドキュメントは carta プロジェクトの **一次情報源 (source of truth)** である。
> 実装に着手する Claude Code セッションは、まずこのファイルを通読してから設計判断を行うこと。
> 現状はリポジトリ scaffold のみが存在する。skill / commands / assets は未実装。

---

## 1. ミッション

**全プロジェクト共通で適用したい開発原則 (= 憲法 / constitution) を 1 箇所で管理し、各リポジトリの `CLAUDE.md` に適用・同期する。**

### 解決したい課題

ユーザー (yamamoto) は複数のプロジェクト (cmux-team, mado, Dear, using-cmux, ...) を並行して扱う。各プロジェクトの `CLAUDE.md` には共通して書きたい原則 (例: 「日本語でドキュメントを書く」「不要なコメントを書かない」「テストを必ず実行する」「セッション最後にサマリを書かない」など) があるが、現状は手で各リポジトリに同じ内容を書き写しており、以下の問題がある:

1. **drift**: あるリポジトリで原則を改訂しても、他リポジトリには反映されない
2. **冗長**: 原則を新規プロジェクトに導入するたびに同じ内容をペーストする手間
3. **検証困難**: 原則が常に最新版に揃っているかを目視確認する手段がない

### 解決アプローチ

- 憲法本体を carta plugin の `assets/constitution.md` として 1 箇所で管理する
- `/carta:apply` コマンドで対象プロジェクトの `CLAUDE.md` 内の **マーカー領域だけ** を最新の憲法で置換する
- マーカー外のプロジェクト固有記述 (使用フレームワーク・テストコマンド・特殊な運用) は保護する
- 適用済みバージョンを `CLAUDE.md` 内のメタコメントに記録し、新バージョン検出時に同期を促す

---

## 2. ネーミングと配布

| 項目 | 値 |
|------|---|
| プラグイン名 | `carta` |
| 由来 | 羅: carta (憲章)。Magna Carta の "carta"。短く打ちやすく `/carta:apply` のように commandが映える |
| 公開先 | `github.com/hummer98/carta` |
| author | `hummer98` (ユーザーの GitHub identity) |
| ライセンス | MIT |
| 配布方法 | Claude Code Plugin (marketplace 経由) — npm publish はしない |
| バージョニング | SemVer |

> プラグイン名と GitHub repo 名は同じ `carta` で統一する。marketplace の登録 ID は `hummer98-carta`。

---

## 3. アーキテクチャ

### 3.1 概念モデル

```
┌─────────────────────────────────────────┐
│ carta plugin                            │
│ ├─ assets/constitution.md (← 憲法本体)  │
│ ├─ skills/carta/SKILL.md                │
│ └─ commands/{apply,diff,show}.md        │
└─────────────────┬───────────────────────┘
                  │  /carta:apply
                  ▼
┌─────────────────────────────────────────┐
│ 対象プロジェクト                         │
│ └─ CLAUDE.md                            │
│    ├─ <!-- carta:begin v0.1.0 -->       │
│    │  (憲法本体がここに同期される)       │
│    ├─ <!-- carta:end -->                │
│    │                                    │
│    └─ プロジェクト固有部分               │
│       (フレームワーク, テスト, 運用ルール)│
└─────────────────────────────────────────┘
```

### 3.2 マーカー仕様

CLAUDE.md 内に以下のマーカーで囲まれた領域を carta が管理する:

```markdown
<!-- carta:begin v0.2.0 sha=abc1234 profiles=flutter,typescript -->
（憲法本体テキスト + 指定プロファイル本文の連結結果がここに展開される）
<!-- carta:end -->
```

- `v0.2.0` — 適用された carta のバージョン
- `sha=abc1234` — **連結後本文** の SHA-256 short hash (先頭 7 文字)。ローカル編集が無いことを検証する。**profiles 空時は `assets/constitution.md` を `cat` したバイト列と完全一致するため v0.1.0 と sha も一致する**
- `profiles=flutter,typescript` — 適用済みプロファイル名のアルファベット順カンマ区切り。**プロファイル指定なし時は属性自体を省略する**（v0.1.0 マーカーと同形）
- マーカー外の内容は **絶対に変更しない**
- マーカーの位置 (CLAUDE.md の冒頭か末尾か) は適用時に決まる: 既存 CLAUDE.md があればその冒頭に挿入、無ければ新規作成して冒頭に置く

マーカー検出/解析の正規表現（apply Step 3 の grep と解析の両方で同じパターンを使う）:

```
^<!-- carta:begin v[^ ]+ sha=[0-9a-f]+( profiles=[A-Za-z0-9_,-]+)? -->$
^<!-- carta:end -->$
```

`profiles=` 属性はオプショナル（`(...)? ` 全体が optional）。`profiles=` 値の文字種は `[A-Za-z0-9_,-]+`（空白・`/`・`.` などは含めない）。`[^ ]+` は次の空白文字までで version トークンを終え、version 文字列に空白が混入することは禁止する。

解析用（capture 群を使う版、§5.2 / commands/apply.md Step 4c）:

```
<!-- carta:begin v(\S+) sha=([0-9a-f]+)( profiles=([A-Za-z0-9_,-]+))? -->
```

### 3.3 idempotency 保証

`/carta:apply` を同じ憲法バージョンで複数回実行しても、CLAUDE.md は変化しないこと:

- 既存の `<!-- carta:begin ... -->` ブロックを検出し、内容＋バージョン＋hash がすべて一致するならファイル書き込みを skip
- 一致しない場合のみ、マーカー内部を新しい憲法本体で置換する

---

## 4. コマンド仕様

### 4.1 `/carta:apply`

**目的**: 現在のプロジェクトの CLAUDE.md に最新の憲法を適用 / 同期する。

**実行内容**:

1. プラグインの `assets/constitution.md` と `assets/constitution.md` の hash, plugin.json の version を取得
2. カレントディレクトリ (もしくは git root) の `CLAUDE.md` を読み取る
   - 存在しない場合 → `<!-- carta:begin -->...<!-- carta:end -->` で囲んだ新規 CLAUDE.md を作成
   - 存在する場合 → 既存マーカーの有無を確認
3. マーカーが見つかれば内容を置換、見つからなければファイル冒頭に挿入
4. 変更があった場合は diff を要約して報告する

**引数**:

- 引数なし: カレントディレクトリの CLAUDE.md を更新（既存マーカーの profiles を引き継ぐ）
- `/carta:apply <path>`: 指定パスの CLAUDE.md を更新
- `/carta:apply --profile <names>` / `--profile=<names>`: カンマ区切りで連結適用するプロファイルを指定（v0.2.0〜、§15 参照）
- `/carta:apply --profile=`（空指定の推奨形式）: プロファイル全外し
- `/carta:apply --force`: マーカー内 sha 不一致でも上書きを許可

**安全性**:

- 書き込み前に必ずユーザーに diff を見せて確認を取る (一発適用しない)
- マーカー外のテキストは絶対に編集しない (assertion で検証)

### 4.2 `/carta:diff`

**目的**: `/carta:apply` を実行したらどう変わるかを dry-run で見せる。

**実行内容**: 上記 `apply` の手順 1-3 までを実行し、書き込みは行わずに diff だけ表示する。

### 4.3 `/carta:show`

**目的**: 現在 plugin にバンドルされている憲法本体テキスト（または指定プロファイル本文）を表示する。

**実行内容**: `assets/constitution.md`（または `assets/profiles/<name>.md`）の内容を読み取って表示する。`--meta` オプションでバージョン・hash 等のメタ情報も併記。

**引数**:

- 引数なし: 憲法本文をそのまま出力
- `--meta`: ファイル冒頭のメタコメントから 1 行ヘッダ（`[carta] version: ...` または `[carta-profile] name: ...`）を整形して先頭に置く
- `--profile <name>` / `--profile=<name>`: 単一プロファイル本文を表示（v0.2.0〜）。`--meta` と併用可能
- `--profile=`（空指定）は **show では name 必須エラー**（`/carta:apply` の `--profile=` 全外しと意図的に非対称）。デフォルト憲法を見たい場合は引数なしの `/carta:show`

---

## 5. スキル仕様 (`skills/carta/SKILL.md`)

### 5.1 トリガー条件

`description` には以下のニュアンスを含める:

- ユーザーが「憲法を適用」「CLAUDE.md を同期」「carta」「開発原則をプロジェクトに反映」などに言及したとき
- 新規プロジェクトで CLAUDE.md を初めて作成しようとしているとき
- プロジェクト間で原則の整合性を取りたいとき

### 5.2 スキルの責務

- マーカー仕様の解説
- 適用前の安全確認 (diff レビュー必須) の徹底
- マーカー外領域を保護するルールの強調
- 既存 CLAUDE.md がある場合の merge 戦略
- バージョン不一致時のユーザー誘導 (古い適用版が見つかったら更新提案)

### 5.3 スキル本文に書くべきこと

```markdown
---
name: carta
description: "全プロジェクト共通の開発憲法 (constitution) を CLAUDE.md に適用・同期するスキル。
ユーザーが「carta」「憲法適用」「CLAUDE.md 同期」「開発原則」「principles 適用」に言及したとき、
または新規プロジェクトで CLAUDE.md を初めて作成しようとしているときに使用する。"
---

# carta スキル

(中略)

## マーカー仕様
(セクション 3.2 を要約)

## 適用フロー
1. apply 前に diff を提示
2. ユーザー承認後に書き込み
3. マーカー外を絶対に変更しない
```

---

## 6. 憲法本体 (`assets/constitution.md`) のスキーマ

### 6.1 セクション構成 (推奨案)

```markdown
<!-- カレント版: v0.1.0 -->

# 開発憲法

このセクションは [carta](https://github.com/hummer98/carta) によって管理されている。
直接編集せず、リポジトリ固有のルールはマーカー外に記述すること。

## 1. コミュニケーション
- ドキュメント・コメントは日本語
- コードと識別子は英語
- セッション末尾にサマリを書かない

## 2. コード品質
- コメントは原則書かない (WHY が非自明な場合のみ)
- 過剰な抽象化・先回りの汎化を避ける
- 後方互換シムや使われない引数を残さない

## 3. テスト
- (...)

## 4. Git ワークフロー
- (...)

## 5. AI 協業の心得
- (...)
```

### 6.2 初期コンテンツの取得元

ユーザー側で別途用意した「憲法ドラフト」が存在する。実装着手時にユーザーから受領するか、現時点で読みやすい近似として **本リポジトリ scaffold 時の親プロジェクト (cmux-team) の `CLAUDE.md` 上部「設計原則」「コーディング規約」セクション + Claude Code 一般原則** をたたき台にすること。最終確定はユーザー承認を得る。

### 6.3 憲法のメタ情報

ファイル冒頭にコメントで以下を記述:

```markdown
<!--
carta-constitution
version: 0.1.0
last-updated: YYYY-MM-DD
maintainer: hummer98
-->
```

---

## 7. 実装言語と依存

- **shell + Markdown 中心**。複雑なロジックが必要になった時のみ Bun/Node スクリプトを `bin/` に置く
- 依存ツール: `git`, `sha256sum` (or `shasum -a 256`), `awk`/`sed`, 標準的な POSIX shell
- バイナリの bundling・compile step は持たない。インストール後即動作する状態を維持

---

## 8. ファイルレイアウト (実装後の最終形)

```
carta/
├── .claude-plugin/
│   ├── plugin.json
│   └── marketplace.json
├── .claude/
│   └── commands/
│       └── release.md            # この repo 自身のリリース手順
├── skills/
│   └── carta/
│       └── SKILL.md              # AI が読むスキル定義
├── commands/
│   ├── apply.md                  # /carta:apply
│   ├── diff.md                   # /carta:diff
│   └── show.md                   # /carta:show
├── assets/
│   └── constitution.md           # 憲法本体テキスト (ソースオブトゥルース)
├── bin/                          # (将来: shell スクリプトを追加する場合)
├── docs/
│   └── seed.md                   # ← このファイル
├── README.md                     # 人間向けガイド (英)
├── README.ja.md                  # 人間向けガイド (日)
├── CLAUDE.md                     # 開発ガイド (リポジトリ自身用)
├── LICENSE
├── CHANGELOG.md                  # リリース時に追記
└── .gitignore
```

---

## 9. エッジケースと方針

| 状況 | 方針 |
|------|------|
| CLAUDE.md が存在しない | 新規作成。マーカー領域 + 「## プロジェクト固有」見出しの空セクションを置く |
| 既存 CLAUDE.md にマーカーが無い | ファイル冒頭にマーカーブロックを挿入。既存内容はそのまま下に残す |
| マーカーが破損 (begin あるが end が無い等) | エラーで停止。ユーザーに手動修復を求める (自動修復しない) |
| マーカー内が手で編集されている | hash 不一致を検出 → ユーザーに通知、`--force` で上書きを許可 |
| 複数の CLAUDE.md がリポジトリ内にある (mono-repo) | v0.1.0 では root の 1 つだけ対象。引数 `<path>` で個別指定可 |
| 憲法が更新された後の追従 | 適用済みバージョン < plugin version の検出は **本人が `/carta:apply` を叩いたとき** に行う。push 通知は出さない |

---

## 10. v0.1.0 のスコープ (MVP)

**含める**:

- `assets/constitution.md` のドラフト (たたき台)
- `/carta:apply` コマンド
- `/carta:show` コマンド
- マーカー方式の同期ロジック (idempotent)
- README.md / README.ja.md

**含めない (v0.2 以降)**:

- `/carta:diff` (apply 内で diff 確認するので一旦不要かもしれない。実装の優先度は中) → **v0.1.0 で実装済み**
- 複数 CLAUDE.md の一括同期
- CI で「憲法が古いリポジトリ」を検出する機能
- ~~憲法のリポジトリ別カスタマイズレイヤ (override)~~ → **v0.2.0 で profiles 機能として実現** (§15 参照)
- ~~憲法を分割して必要なセクションだけ適用するモード~~ → **v0.2.0 で profiles 機能として実現** (§15 参照)

---

## 11. 実装着手時の進め方 (推奨)

1. **`assets/constitution.md` の初稿を書く**: ユーザーから既存の「憲法素案」を受け取り、本リポジトリに転記。受け取れない場合は親プロジェクトの CLAUDE.md からたたき台を抽出してユーザーに提示
2. **`commands/apply.md` を実装**: スラッシュコマンドの本体。`allowed-tools: Read, Edit, Write, Bash` を frontmatter に持たせ、手順を Markdown で記述する (Claude が手順に沿って動作する形)
3. **`commands/show.md` を実装**: `assets/constitution.md` を Read して表示するだけ
4. **`skills/carta/SKILL.md` を実装**: 上記コマンド群と矛盾しないトリガー定義 + マーカー仕様の説明
5. **README.md と README.ja.md** を整備
6. **テスト**: 手動で `/carta:apply` を別プロジェクト (例: 一時的に作った dummy ディレクトリ) に対して実行し、idempotency と保護動作を確認
7. **v0.1.0 を release**: `.claude/commands/release.md` の手順に沿ってタグ・GitHub Release・marketplace 登録

---

## 12. 設計判断のメモ

### 「ユーザーメモリではなくスキル化」の理由

- 憲法本文は長文になりうるため、ユーザーメモリに置くと毎会話 context を圧迫する
- 「適用したいときだけ読む」性質なので、トリガー条件の明示できるスキルの方が自然
- 別プロジェクトの CLAUDE.md を更新するという **明確な操作** が中心なので、スラッシュコマンドが UI として適切

### 「cmux-team の中ではなく独立 plugin」の理由

- cmux-team は「マルチエージェント基盤」が責務で、汎用的な開発憲法は別軸の関心事
- 同居させると plugin install 時に意図しない押し付けになる
- ライフサイクル (リリース頻度) も別

### 「マーカー方式」の理由

- 既存 CLAUDE.md にプロジェクト固有のセクションがあるのが一般的
- 全置換するとプロジェクト固有部分が消える / 全マージ戦略は競合解決が必要 → どちらも危険
- マーカーで「carta が触る領域」を明示的に分離するのが安全かつ予測可能

### 「`assets/constitution.md` を Markdown 1 枚で持つ」の理由

- 編集と diff レビューが容易
- テンプレートエンジン無しで CLAUDE.md に貼り付けられる
- 将来カスタマイズレイヤを足す際も、追加ファイルとして並置できる

---

## 13. オープンクエスチョン (実装時に決める)

1. 憲法本文の章立て構成は最終的にどうするか? (ユーザーから素案受領後に確定)
2. CLAUDE.md 冒頭にマーカーを置くか、末尾に置くか? → **冒頭推奨** (Claude が読み始めた時点で原則が目に入る)。ただし既存 CLAUDE.md に大きな見出しがある場合は読みづらい可能性あり。要検討
3. **[解決]** プロジェクト別カスタマイズレイヤ → v0.2.0 で `assets/profiles/*.md` の連結方式として実現。`.carta/local.md` 案ではなく、プラグイン内同梱・引数選択方式を採用（§15 参照）。
4. 既存プロジェクト CLAUDE.md の「言語ルール」「コーディング規約」セクションを憲法側に巻き取るリファクタガイドが欲しいか? (`/carta:migrate` のような移行コマンド)

---

## 14. 参考プロジェクト

- **using-cmux** (`~/git/using-cmux`): 同じ author の Claude Code Plugin。レイアウト・配布方法・release.md のテンプレート元
- **cmux-team** (`~/git/cmux-team`): 親プロジェクト。本構想が生まれた文脈

---

## 15. プロファイル仕様 (v0.2.0 で追加)

### 15.1 概要

デフォルト憲法に追加で連結適用できる、用途別オプションコンテンツ。`assets/profiles/<name>.md` として同梱し、`/carta:apply --profile <names>` で連結する。

### 15.2 連結アルゴリズム

```
function build_combined_body(default_body, profile_names):
  # 1. プロファイル名を正規化（trim → 空除去 → アルファベット順 sort -u）
  normalized = sort_unique_alphabetical(strip_whitespace(profile_names))
  # 2. 各 part 末尾 LF を「ちょうど 1 個」に正規化（剥がさず保つ）
  parts = [normalize_trailing_lf_to_one(default_body)]
  for name in normalized:
    path = "assets/profiles/" + name + ".md"
    if not file_exists(path): fail("profile not found: " + path)
    parts.append(normalize_trailing_lf_to_one(read(path)))
  # 3. 各 part 末尾 LF を保持したまま part 間に "\n---\n\n"（6 byte）を挿入
  combined = parts[0]
  for p in parts[1:]:
    combined = combined + "\n---\n\n" + p
  return (combined, normalized)
```

**不変条件**: profiles が空のとき `combined` は `cat assets/constitution.md` と byte-for-byte 完全一致する → v0.1.0 sha と一致（実機検証値: `f0c10c8`）。

### 15.3 引数仕様

apply / diff / show の引数仕様は §4 / commands/{apply,diff,show}.md を参照。要点のみ:

- `--profile <names>` / `--profile=<names>`: カンマ区切り
- 空指定の推奨形式は `--profile=`（イコール接続）。`--profile ""` は環境依存で非推奨
- apply / diff: 引数未指定なら既存マーカーから profiles を引き継ぐ。`--profile=` で全外し
- show: `--profile=`（空）は **name 必須エラー**（apply との意図的な非対称）

### 15.4 マーカー拡張

§3.2 参照。`profiles=` 属性は **空時は省略**（v0.1.0 マーカーと完全同形）し、非空時のみアルファベット順正規化値を出力する。

### 15.5 不存在エラーの方針

- 指定された `assets/profiles/<name>.md` が無ければ即座に停止（自動作成しない）
- 名前の許容文字種は `[A-Za-z0-9_-]+`（空白・`/`・`.` は禁止）
- 既存マーカーの `profiles=` 値が指す `.md` が無いケースも同じエラーで停止し、誘導メッセージで「`--profile=正しい値` で上書きしてください」と案内
- profiles 間の依存関係・自動検出・プロファイル別 sha 検証はサポートしない

### 15.6 v0.1.0 互換性ルール

- v0.1.0 マーカーは「プロファイルなし」として完全互換動作
- v0.1.0 で適用済み CLAUDE.md に v0.2.0 を再 apply（profiles 空のまま）すると、本文と sha は変わらず、version 文字列のみ更新される
- sha 計算は v0.1.0 の `(sha256sum||shasum -a 256) "$SRC" | awk '{print $1}'` 方式と **同値の sha** を生成する。「sha 不一致でも body 一致なら idempotent」フォールバックは採用しない（手編集検出の安全性を維持）

### 15.7 凍結された定数

以下はリリース後に **絶対に変更しない**。書式が変わると既存 CLAUDE.md の sha が破壊される:

- プロファイル本文先頭の `<!-- carta-profile: NAME -->` 書式
- 連結区切り文字列: 各 part 末尾 LF + `"\n---\n\n"`（6 byte リテラル）
- 末尾改行ポリシー: `assets/constitution.md` および `assets/profiles/*.md` は常に末尾 LF 1 個で保存する運用ルール（`.editorconfig` は新規作成しない方針）

---

> 実装着手前にこの seed をユーザーと一緒にレビューし、§13 のオープンクエスチョンを確定させてから動くこと。
