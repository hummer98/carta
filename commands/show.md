---
allowed-tools: Read
description: "carta プラグインにバンドルされている開発憲法本体を表示する"
argument-hint: "[--meta]"
---

# /carta:show

`assets/constitution.md` の内容を表示する。読み込み専用で、ファイルの編集・書き込みは一切しない。

## 引数

`$ARGUMENTS`:
- 指定なし: 憲法本文をそのまま出力
- `--meta`: 本文に加えて、ファイル冒頭の `<!-- carta-constitution ... -->` メタ情報（version / last-updated / maintainer）を読みやすい形に整形して先頭に表示

## 手順

1. プラグインのルートにある `assets/constitution.md` を Read する。

   - 環境変数 `${CLAUDE_PLUGIN_ROOT}` が設定されていればそれを使う
   - 未設定の場合はカレントリポジトリ（`git rev-parse --show-toplevel` の結果、無ければ `pwd`）の `assets/constitution.md` を読む

2. `$ARGUMENTS` に `--meta` が含まれているかを判定する。

3. 出力:

   - 引数なし: ファイル内容をそのまま 1:1 で出力する。冒頭の `<!-- carta-constitution ... -->` コメントもそのまま含まれる
   - `--meta`: ファイル冒頭の HTML コメントブロック（`<!--` から最初の `-->` まで）を抽出し、以下のような 1 行ヘッダに整形して本文の前に出す:

     ```
     [carta] version: 0.1.0  last-updated: 2026-05-10  maintainer: hummer98
     ```

     その後にファイル本文をそのまま続ける。

## 出力する範囲

本コマンドは表示のみ。ファイル変更・新規作成・他コマンドの呼び出しはしない。
