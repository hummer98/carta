# `/carta:migrate` 手動テスト手順

`/carta:migrate` の挙動を確認するためのシナリオ集。**自動テストランナーは無い**（plan §7 / seed §7 「複雑なロジックが必要になった時のみ Bun/Node を `bin/` に置く」方針）。代わりに各 fixture について「コマンド例 / 期待出力 / exit code」を本ドキュメントに明記し、目視 / `diff` で比較する。

将来 CI 化する場合は v0.5.0 以降の別タスクで apply / diff / show と横断的に harness 化する想定。

## 必須シナリオ（4 件）

| # | fixture | 目的 |
|---|---|---|
| T1 | `T1-no-markers/` | マーカー無し → 「先に `/carta:apply`」誘導 + exit 1 |
| T2 | `T2-no-duplicates/` | candidates 0 件 → 重複なし no-op + exit 0 |
| T3 | `T3-high-confidence-bullet/` | HIGH 一致 1 件 → 削除候補 + unified diff（書き込みなし） |
| T4 | `T4-apply-marker-unchanged/` | `--apply` 後にマーカー領域 byte 不変 |

T2/T3/T4 のマーカー内本文は **重複検出には影響しないプレースホルダ** で、対応する `sha=4a06f48` を marker に記録してある。`/carta:migrate` は REF として **実 `assets/constitution.md`** を読むため、検出結果はマーカー内ではなくマーカー外の bullet を実 constitution と比較した結果になる。

## 共通の準備

各シナリオは「dummy リポジトリに fixture を置いて `/carta:migrate` を回す」形で確認する。

```bash
# 1. 一時ディレクトリを作って git init
TMP=$(mktemp -d)
cd "$TMP" && git init -q

# 2. 対象 fixture の input.md を CLAUDE.md として置く
cp /Users/yamamoto/git/carta/.worktrees/task-009-1778483690/tests/migrate/fixtures/T1-no-markers/input.md ./CLAUDE.md
git add CLAUDE.md && git commit -qm "fixture"

# 3. CLAUDE_PLUGIN_ROOT を carta worktree に向け、Claude Code セッションで /carta:migrate を実行
#    （セッション側で対象 path を引数指定するか、cd "$TMP" の状態で /carta:migrate を呼ぶ）
export CLAUDE_PLUGIN_ROOT=/Users/yamamoto/git/carta/.worktrees/task-009-1778483690
```

## T1 — マーカー無し

**目的**: マーカーが見つからない場合、書き込みも diff も出さず誘導メッセージで停止することを確認。

```bash
# fixture を CLAUDE.md として配置した dummy リポで:
/carta:migrate
```

**期待**:
- stderr に `expected.stderr` の内容が出る
- exit code = 1
- ファイル書き込みは一切発生しない

確認:
```bash
diff <(/carta:migrate 2>&1 1>/dev/null) tests/migrate/fixtures/T1-no-markers/expected.stderr
```

## T2 — 重複なし

**目的**: マーカー外がプロジェクト固有の内容のみで、実 constitution との重複が無い場合に no-op で正常終了することを確認。

```bash
/carta:migrate
```

**期待**:
- stdout の本体が `expected.stdout` と同じ（`<TARGET>` は実 path に展開されている前提で目視確認）
- exit code = 0
- ファイル書き込みは一切発生しない（`git status` で no-change）

検証ポイント:
- `[carta:migrate]` ヘッダの `version=` / `sha=` / `profiles=(none) (from marker)` がプレースホルダ内容に対応した表示になる
- "no duplicates — マーカー外に重複は検出されませんでした" を含む

## T3 — HIGH 一致（dry-run）

**目的**: マーカー外の bullet が実 constitution の bullet と完全一致した場合に、HIGH 候補として表示され unified diff が出ることを確認。`--apply` 無しなので書き込みは発生しない。

```bash
/carta:migrate
```

**期待**:
- stdout に `expected.stdout` の候補テーブルが出る（行番号 / ref_line / ref_quote が一致）
- 続いて `expected.diff` の unified diff（hunk header `@@ -15,1 +15,0 @@`）が表示される
- exit code = 0
- ファイル書き込みは一切発生しない（dry-run のため最終承認プロンプトは出るが、`n` で抜けても良い。むしろ dry-run はそもそも write しないので承認プロンプト無しで終わる）

検証ポイント:
- 該当行 `- ドキュメント・コメント・ユーザー向けテキストは日本語で書く。` が candidate に上がっている
- 同じ見出し配下の `- ライブラリ名・サードパーティ API 名はカナ表記しない。` は **candidate ではない**（プロジェクト固有）
- 見出し `## 言語ルール` は **集約されない**（配下に非候補 bullet が残っているため）

## T4 — `--apply` 後にマーカー byte 不変

**目的**: `--apply` で書き込みを行った後、マーカー領域（begin / end / 内側本文）が **byte-for-byte** 同一であることを確認。

```bash
/carta:migrate --apply
# 最終承認プロンプトに y で答える
```

**期待**:
- 書き込み後の CLAUDE.md が `expected.output.md` と一致（マーカー外の該当 bullet 1 行のみ消えている）
- マーカー領域（line 1 〜 line 7）は input.md と完全一致
- exit code = 0
- assertion (`inside_after == inside_before` のバイト比較 + 再 sha) が pass

確認手順（マーカー領域の byte-for-byte 検証）:

```bash
# マーカー領域だけを抜き出して sha を比較する
sed -n '1,7p' input.md | (sha256sum 2>/dev/null || shasum -a 256)
sed -n '1,7p' CLAUDE.md | (sha256sum 2>/dev/null || shasum -a 256)
# → sha が一致すれば byte-for-byte 不変が確認できる
```

加えて全体 diff の確認:

```bash
diff CLAUDE.md tests/migrate/fixtures/T4-apply-marker-unchanged/expected.output.md
# → 差分なし
```

## 本リポジトリ自身の CLAUDE.md に対する dry-run（plan §7.5 サニティチェック）

```bash
cd /Users/yamamoto/git/carta/.worktrees/task-009-1778483690
/carta:migrate
```

**期待**: 候補 0 件 / `no duplicates` / exit 0。

本リポの CLAUDE.md マーカー外は「carta 開発ガイド」（開発者向けメタ情報）であり、憲法本文とは重複しないはずなので、誤検出を出さないサニティチェック相当。HIGH 検出のベンチマークではない。

## CRLF / `.gitattributes`

`tests/migrate/fixtures/.gitattributes` で `*.md binary` を指定し、git の autocrlf 等で改行コードが意図せず変換されることを防ぐ（plan §7.4）。CRLF fixture は本リリースには含めない（必要なら次リリースで T5 として追加）。

## しきい値の実測（plan §4.4）

`MIGRATE_OVERLAP_MEDIUM` (default `0.85`) / `MIGRATE_OVERLAP_LLM` (default `0.50`) は `[要調整]`。fixture を増やしてグレーゾーン（言い換え 1 件で overlap ≒ 0.7）を実測したい場合は、本ディレクトリ配下に T5 などとして任意で追加可能。本リリースのスコープでは必須 4 件のみ。

## なぜランナーが無いのか

- 既存の apply / diff / show / help も「Markdown コマンドファイル + Claude が手順を実行する」方式で、自動テスト harness は持たない
- migrate だけに大規模な harness を新設すると一貫性が崩れる
- まず手動シナリオで挙動を確定させ、後続リリースで横断 harness を整備する（plan §7 / §9）
