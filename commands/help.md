---
allowed-tools: Read
description: "carta プラグインの全コマンド・使い方・マーカー仕様の要点を表示する"
---

# /carta:help

carta プラグインの全コマンド・使い方・マーカー仕様の要点を表示する表示専用コマンド。引数を取らない。ファイル書き込み・他コマンド呼び出しはしない。

## 表示内容（順序固定）

以下のテキストをそのままユーザーに提示する。`${CLAUDE_PLUGIN_ROOT:-<このリポジトリ>}` はリテラル文字列としてそのまま表示し、変数展開しない。

```
# carta — 開発憲法を CLAUDE.md に適用・同期するプラグイン

## コマンド
| コマンド | 役割 |
|---|---|
| /carta:apply [<path>] [--profile <names>] [--force] | 対象 CLAUDE.md に憲法を適用 |
| /carta:diff [<path>] [--profile <names>] | apply の dry-run（書き込まない） |
| /carta:show [--meta] [--profile <name>] | 憲法本体 / プロファイル本文を表示 |
| /carta:help | このヘルプを表示 |

## よく使う例
- 新規プロジェクトに適用:           /carta:apply
- 既存に TypeScript プロファイル追加: /carta:apply --profile typescript
- 複数プロファイル:                  /carta:apply --profile typescript,flutter
- 適用前に差分確認:                  /carta:diff
- 現在の憲法を確認:                  /carta:show --meta

## マーカー仕様
CLAUDE.md 内の `<!-- carta:begin v<VER> sha=<HASH> profiles=<NAMES> -->`
～ `<!-- carta:end -->` の領域だけを carta が管理する。
マーカー外は 1 byte も変更しない。プロジェクト固有の規約はマーカー外に書く。

## 詳細
- README: ${CLAUDE_PLUGIN_ROOT:-<このリポジトリ>}/README.ja.md
- スキル定義: ${CLAUDE_PLUGIN_ROOT:-<このリポジトリ>}/skills/carta/SKILL.md
- プラグインルート: ${CLAUDE_PLUGIN_ROOT:-<このリポジトリ>}
```

## やらないこと

- 引数解析（フラグなし、`--verbose` / `--examples` 等は v0.3 では作らない）
- 他コマンドの呼び出し
- ファイル書き込み・編集
- 出力の i18n / 動的生成（`${CLAUDE_PLUGIN_ROOT:-<このリポジトリ>}` もリテラル表示するため、変数展開も行わない）
