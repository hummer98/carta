---
allowed-tools: Read, Bash
description: "/carta:apply の dry-run。CLAUDE.md がどう変わるかを diff で表示する（書き込まない）"
argument-hint: "[<path>]"
---

# /carta:diff

`/carta:apply` を実行したらどう変わるかを **書き込まずに** 表示する dry-run コマンド。

## 引数

`$ARGUMENTS`:
- 指定なし / `<path>`: `/carta:apply` と同じ規則で対象 CLAUDE.md を特定する

## 手順

`/carta:apply` の **Step 1 〜 Step 5（diff 提示）まで** を実行する。Step 6 の書き込みは行わない。

具体的には:

1. プラグインの `assets/constitution.md` を読み、SHA-256 短縮 hash と version を取得
2. 対象 CLAUDE.md を特定（git root / 引数 path / カレント）
3. 既存マーカーの有無・破損・同期状態を判定
4. 結果に応じて以下を表示:

   | ケース | 表示内容 |
   |---|---|
   | CLAUDE.md が無い | 「新規作成予定」+ 想定ファイル内容 |
   | マーカー無し | 「冒頭に X 行のマーカーブロックを挿入予定」+ 挿入される内容 |
   | マーカーあり、本文一致 | 「最新（v{version} sha={hash}）。変更なし」 |
   | マーカーあり、本文相違 | 「マーカー内が更新される予定」+ unified diff |
   | マーカーあり、sha 不一致（手編集の疑い） | 「マーカー内が手で編集された可能性。`/carta:apply --force` で上書き可能」+ diff |
   | マーカー破損 | 「マーカー破損（手動修復が必要）」 |

5. 書き込みは一切行わない。Read / Bash のみで完結する。

## 出力形式の例

```
[carta:diff] /Users/foo/myrepo/CLAUDE.md
  current: v0.0.9 sha=abc1234  (57 行)
  next:    v0.1.0 sha=def5678  (60 行)
  status:  update available

  --- マーカー内 unified diff ---
  @@ -3,3 +3,4 @@
  -古い項目
  +新しい項目
  +追加項目
```

## 注意

- このコマンドは Read 専用。承認プロンプトを出さない（書き込まないため）
- `/carta:apply` を呼ぶ前段の確認用途として使う
