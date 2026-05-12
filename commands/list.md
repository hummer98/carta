---
allowed-tools: Bash
description: "carta プラグインにバンドルされているプロファイル一覧を表示する"
---

# /carta:list

`bin/carta-list` を実行し、その出力を **そのままユーザーに提示する** 読み込み専用コマンド。引数は受け取らない。要約・整形・解釈を加えない。

## 手順

```bash
"${CLAUDE_PLUGIN_ROOT:-$(pwd)}/bin/carta-list"
```

## 出力イメージ

```
[carta] available profiles (3)

  flutter      Flutter プロファイル（プレースホルダ）
  interactive  対話セッション運用（サマリ規約 / UI 実機確認）
  typescript   TypeScript プロファイル（プレースホルダ）

usage:
  /carta:show --profile <name>      単一プロファイル本文を表示
  /carta:apply --profile <names>    カンマ区切りで連結適用
```

プロファイル 0 件の場合は `[carta] no profiles installed` のみ表示される。

## やらないこと

- 引数解析（フラグ無し）
- 他コマンド呼び出し
- ファイル書き込み・編集
- スクリプト出力の翻訳・i18n・整形・要約

## 実装メモ

- ロジックの本体は `bin/carta-list`（POSIX 互換 bash + sed + awk）
- description 抽出の優先順: `<!-- carta-summary: ... -->` → `^## ` 見出し → 空白
- ファイル一覧はアルファベット順、name 列は最長プロファイル名 + 2 スペース幅で揃える
