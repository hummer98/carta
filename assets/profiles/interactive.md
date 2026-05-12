<!-- carta-profile: interactive -->
<!-- carta-summary: 対話セッション運用（サマリ規約 / UI 実機確認） -->

## 対話セッション運用

このプロファイルは人間と Claude が対話形式で進める interactive Claude Code session を前提とする。elevens / OpenClaw / AutoClaude などの全自動オーケストレーター配下では、orchestrator が注入する role prompts が優先するため、本プロファイルは外してよい（`--profile=` でプロファイル全外し、または対象一覧から interactive を除外）。

- ターン末尾のサマリは「何を変えた / 次に何をするか」を 1–2 文に収める。差分の再記述・自己解説・冗長な前置きは加えない。
- UI・フロントエンド変更は dev server を立ち上げてブラウザで実機確認する。golden path とエッジケースを試す。確認できない場合はその旨を明示する（「テストは通ったが UI は未確認」）。
