<!-- carta:begin v0.3.0 sha=c52ec08 profiles=interactive -->
<!--
carta-constitution
version: 0.3.0
last-updated: 2026-05-12
note: 対話セッション向けルール（サマリ規約 / UI 実機確認）は profile=interactive に分離
maintainer: hummer98
-->

# 開発憲法

このセクションは [carta](https://github.com/hummer98/carta) によって管理されている。
直接編集せず、リポジトリ固有のルールはマーカー外に記述すること。

## 1. コミュニケーション

- ドキュメント・コメント・ユーザー向けテキストは日本語で書く。
- コード・変数名・関数名・コマンド名・識別子は英語のままにする。日本語に翻訳しない。

## 2. テストと完了基準

- 既存のテストスイートが定義されていれば、変更後に必ず実行して通ることを確認する。
- 型チェック・lint の通過は機能の正しさを保証しない。挙動の検証は別途行う。
- 自分が触ったファイルのテスト失敗・型エラーが増えていない状態で完了とする。「既存エラー」「別タスクで」を理由に放置しない。
- 「完了」と報告するのは実際にやり遂げたことだけ。半分終わった実装・未確認の挙動・未解決のエラーを「完了」と書かない。

## 3. ツール運用

- 互いに依存しない tool call は 1 メッセージ内で並列に発行する。複数ファイルの Read、`git status` と `git diff` の併用などを逐次にしない。
- 直前の結果に依存する場合（取得した値を次の引数に使う等）のみ逐次にする。

## 4. 可観測性とロギング

- ログは「コードがどう動いたか」を後から AI / 人間が再現するためのもの。書くか書かないかは「障害発生時に追跡できるか」を基準に判断する。
- 例外を握りつぶさない。空の `catch` は書かない。最低でもログを残すか再 throw する。冪等な後処理など意図的な無視はその理由をコメントで明示する。
- 外部コマンド・サブプロセス・外部 API 呼び出しの失敗時は、`stderr` / `stdout` / レスポンス本文 / status code をエラー詳細に必ず含める。`e.message` だけでは追跡できない。
- 判断分岐・状態遷移・フォールバック発動など「なぜその経路に入ったか」が後から要る点はログに残す。逆に高頻度ループ（毎 tick / ポーリング）では状態変化があったときだけ記録し、ノイズを増やさない。
- API key・トークン・パスワード・個人情報をログ・エラーメッセージ・例外詳細に含めない。

---

<!-- carta-profile: interactive -->

## 対話セッション運用

このプロファイルは人間と Claude が対話形式で進める interactive Claude Code session を前提とする。elevens / OpenClaw / AutoClaude などの全自動オーケストレーター配下では、orchestrator が注入する role prompts が優先するため、本プロファイルは外してよい（`--profile=` でプロファイル全外し、または対象一覧から interactive を除外）。

- ターン末尾のサマリは「何を変えた / 次に何をするか」を 1–2 文に収める。差分の再記述・自己解説・冗長な前置きは加えない。
- UI・フロントエンド変更は dev server を立ち上げてブラウザで実機確認する。golden path とエッジケースを試す。確認できない場合はその旨を明示する（「テストは通ったが UI は未確認」）。
<!-- carta:end -->

# carta 開発ガイド

統一的な開発憲法 (constitution) を保持し、各プロジェクトの `CLAUDE.md` に適用・同期する Claude Code Plugin。

## まずこれを読む

**`docs/seed.md` がプロジェクト構想の一次情報源**。スキル・コマンド・憲法本体の設計はすべて seed に記述してある。実装作業に入る前に必ず seed を Read すること。

## ファイル構成

| ファイル / ディレクトリ | 役割 |
|------------------------|------|
| `docs/seed.md` | プロジェクト構想・設計の一次情報（ソースオブトゥルース） |
| `assets/constitution.md` | 憲法本体テキスト（全プロジェクト共通の開発原則） |
| `skills/carta/SKILL.md` | carta スキル定義（Claude が読む） |
| `commands/apply.md` | `/carta:apply` — CLAUDE.md に憲法を適用 |
| `commands/diff.md` | `/carta:diff` — 適用差分のプレビュー |
| `commands/show.md` | `/carta:show` — 現在の憲法を表示 |
| `.claude-plugin/plugin.json` | Plugin マニフェスト |
| `.claude-plugin/marketplace.json` | Marketplace カタログ |
| `.claude/commands/release.md` | このリポジトリ自身のリリース手順 |
| `README.md` | 人間向けガイド |
| `LICENSE` | MIT |

## 言語ルール

- **ドキュメント・コメント**: 日本語
- **コード（変数名・関数名・コマンド）**: 英語

## 設計原則

- **憲法はテキストファイル 1 枚** (`assets/constitution.md`)。複雑なテンプレートエンジンは入れない
- **CLAUDE.md への適用は idempotent**。マーカーで囲った領域だけを置換し、プロジェクト固有部分は保護する
- **バージョン管理**: 適用済みバージョンを CLAUDE.md 内のメタデータコメントに記録し、新バージョン検出時に更新提案する

## 配布

GitHub の tag + marketplace 経由で配布する。npm publish はしない。リリース手順は `.claude/commands/release.md` を参照。
