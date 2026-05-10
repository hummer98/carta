---
allowed-tools: Read
description: "carta プラグインにバンドルされている開発憲法本体（およびプロファイル）を表示する"
argument-hint: "[--meta] [--profile <name>]"
---

# /carta:show

`assets/constitution.md`（または指定プロファイル本文）の内容を表示する。読み込み専用で、ファイルの編集・書き込みは一切しない。

## 引数

`$ARGUMENTS`:
- 指定なし: 憲法本文（`assets/constitution.md`）をそのまま出力
- `--meta`: 本文に加えて、ファイル冒頭のメタ情報を読みやすい形に整形して先頭に表示
- `--profile <name>` / `--profile=<name>`: 単一プロファイル本文（`assets/profiles/<name>.md`）を表示する。`--meta` と併用可能

`--profile=`（空指定）は **show では name 必須エラー**（`--profile requires a name`）になる。デフォルト憲法を見たい場合は引数なしの `/carta:show` を使う。これは `/carta:apply` の `--profile=`（プロファイル全外し）とは挙動が異なる **意図的な非対称**。`--list-profiles` は提供しない（`assets/profiles/` を直接見れば足りる）。

## 手順

1. プラグインのルートを特定する。

   - 環境変数 `${CLAUDE_PLUGIN_ROOT}` が設定されていればそれを使う
   - 未設定の場合はカレントリポジトリ（`git rev-parse --show-toplevel` の結果、無ければ `pwd`）

2. `$ARGUMENTS` を解析:

   - `--meta` の有無
   - `--profile <name>` / `--profile=<name>` の値（無ければ NONE、空文字なら name 必須エラー）
   - プロファイル名は `[A-Za-z0-9_-]+` のみ許容。それ以外なら `invalid profile name: <name>` エラー

3. 対象ファイルを Read:

   - profile 指定なし: `${ROOT}/assets/constitution.md`
   - profile 指定あり: `${ROOT}/assets/profiles/<name>.md`（不存在なら `assets/profiles/<name>.md not found` エラー、exit 1）

4. 出力:

   - `--meta` なし: ファイル内容をそのまま 1:1 で出力する。冒頭の `<!-- carta-constitution ... -->` または `<!-- carta-profile: <name> -->` コメントもそのまま含まれる
   - `--meta` あり（profile 指定なし）: ファイル冒頭の HTML コメントブロック（`<!--` から最初の `-->` まで）を抽出し、以下のような 1 行ヘッダに整形して本文の前に出す:

     ```
     [carta] version: 0.2.0  last-updated: 2026-05-10  maintainer: hummer98
     ```

   - `--meta` あり（profile 指定あり）: プロファイル本文先頭の `<!-- carta-profile: <name> -->` コメントから 1 行ヘッダを整形して先頭に置く（角括弧プレフィックス・コロン区切りで既存ヘッダと並びを揃える）:

     ```
     [carta-profile] name: typescript
     ```

     その後にファイル本文をそのまま続ける。

## 出力する範囲

本コマンドは表示のみ。ファイル変更・新規作成・他コマンドの呼び出しはしない。
