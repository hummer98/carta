---
allowed-tools: Read, Bash
description: "/carta:apply の dry-run。CLAUDE.md がどう変わるかを diff で表示する（書き込まない）"
argument-hint: "[<path>] [--profile <names>]"
---

# /carta:diff

`/carta:apply` を実行したらどう変わるかを **書き込まずに** 表示する dry-run コマンド。

## 引数

`$ARGUMENTS`:
- 指定なし / `<path>`: `/carta:apply` と同じ規則で対象 CLAUDE.md を特定する
- `--profile <names>` / `--profile=<names>`: 連結適用するプロファイルをカンマ区切りで指定（apply と同形）
- `--profile=`（空指定）: プロファイル全外しの dry-run
- `--profile` 未指定時: **既存マーカーから profiles を引き継いで** dry-run 計算する（apply と同じ挙動）

`<path>` と `--profile` の出現順序は問わない（先頭以外でも可）。`--force` は受け取らない（書き込まないため不要）。

## 手順

`/carta:apply` の **Step 1 〜 Step 5（diff 提示）まで** を実行する。Step 6 以降の書き込みは行わない。

具体的には:

1. プラグインの `assets/constitution.md` + 指定プロファイルを連結し、SHA-256 短縮 hash と version を取得
2. 対象 CLAUDE.md を特定（git root / 引数 path / カレント）
3. 既存マーカーの有無・破損・同期状態を判定（`profiles=` 属性の引き継ぎを含む）
4. 結果に応じて以下を表示:

   | ケース | 表示内容 |
   |---|---|
   | CLAUDE.md が無い | 「新規作成予定」+ 想定ファイル内容 |
   | マーカー無し | 「冒頭に X 行のマーカーブロックを挿入予定」+ 挿入される内容 |
   | マーカーあり、本文一致 | 「最新（v{version} sha={hash} profiles={...}）。変更なし」 |
   | マーカーあり、本文相違 | 「マーカー内が更新される予定」+ unified diff |
   | マーカーあり、sha 不一致（手編集の疑い） | 「マーカー内が手で編集された可能性。`/carta:apply --force` で上書き可能」+ diff |
   | マーカー破損 | 「マーカー破損（手動修復が必要）」 |

5. profiles の差分も 1 行で表示する:

   ```
   profiles: (none) → typescript,flutter
   ```

   両側が一致するときは省略してもよい。

6. 書き込みは一切行わない。Read / Bash のみで完結する。

## 出力形式の例

```
[carta:diff] /Users/foo/myrepo/CLAUDE.md
  current: v0.1.0 sha=f0c10c8 profiles=(none)  (57 行)
  next:    v0.2.0 sha=def5678 profiles=typescript  (62 行)
  status:  update available

  profiles: (none) → typescript

  --- マーカー内 unified diff ---
  @@ -3,3 +3,4 @@
  -古い項目
  +新しい項目
  +追加項目
```

## 注意

- このコマンドは Read 専用。承認プロンプトを出さない（書き込まないため）
- `/carta:apply` を呼ぶ前段の確認用途として使う
- 連結ロジック・sha 計算・マーカー判定の正規表現は apply.md と完全に同じ仕様を使う
