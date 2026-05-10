# carta

全プロジェクト共通で適用したい **開発原則（憲法）** を 1 箇所で管理し、各リポジトリの `CLAUDE.md` に同期する Claude Code Plugin。

> "carta" はラテン語・イタリア語で「憲章」を意味する（*Magna Carta* の "carta"）。

[English README](./README.md)

## なぜ必要か

複数のリポジトリを並行で扱っていると、各 `CLAUDE.md` に同じルール（ドキュメントは日本語、不要なコメントは書かない、テストを必ず実行、`--no-verify` 禁止、セッション末尾にサマリ書かない、等）を毎回ペーストすることになる。これには 3 つの問題がある:

1. **drift（差分の発生）** — 1 つのリポジトリでルールを改善しても他に反映されない
2. **冗長** — 新規プロジェクトを始めるたびに同じ内容を貼り付ける手間
3. **検証困難** — どのリポジトリが最新版で、どこが古いままなのかを目視確認する手段がない

carta は憲法本体を 1 枚の Markdown（`assets/constitution.md`）として管理し、各リポジトリの `CLAUDE.md` の **マーカーブロック内だけ** に同期する。プロジェクト固有のルールはマーカー外に残り、絶対に書き換えられない。

## しくみ

```
assets/constitution.md  ── /carta:apply ──>  <project>/CLAUDE.md
                                              ┌────────────────────┐
                                              │ <!-- carta:begin --> │
                                              │ （憲法本体）          │
                                              │ <!-- carta:end -->   │
                                              ├────────────────────┤
                                              │ プロジェクト固有     │
                                              │ （触らない）         │
                                              └────────────────────┘
```

- `<!-- carta:begin v0.2.0 sha=abc1234 -->` と `<!-- carta:end -->` で囲まれた領域だけを carta が管理する
- マーカー外は **絶対に変更しない**
- 同じバージョンを再適用しても変化なし（idempotent）
- begin マーカーの `sha` で本文の改変を検知できる

## プロファイル

デフォルト憲法に追加で連結適用できる、用途別のオプションコンテンツ。

- `assets/profiles/*.md` に配置（v0.2.0 では `typescript` と `flutter` を最小プレースホルダとして同梱）
- `--profile` で 1 個以上を選択:

  ```
  /carta:apply --profile typescript
  /carta:apply --profile typescript,flutter
  /carta:apply --profile=                # 明示的にプロファイルなしへ戻す（推奨形式）
  ```

- `/carta:apply` を引数なしで実行すると、既存マーカーに記録されている profiles をそのまま引き継いで再適用する
- 選択されたプロファイル一覧はマーカーに記録される: `<!-- carta:begin v0.2.0 sha=abc1234 profiles=flutter,typescript -->`
- `sha` は **連結後の本文** から計算するので、プロファイル組み合わせの変更は更新として検出される
- `profiles=` 属性のない v0.1.0 マーカーは「プロファイルなし」として完全互換動作する（再 apply しても sha は同一）
- 注: `/carta:show --profile=` は **デフォルト憲法を見るためではなく** name 必須エラーになる。デフォルトを見たい場合は `/carta:show`（引数なし）を使う

## インストール

```
/plugin marketplace add hummer98/carta
/plugin install carta@hummer98-carta
```

## コマンド

| コマンド | 役割 |
|---|---|
| `/carta:apply [<path>] [--profile <names>] [--force]` | 対象の `CLAUDE.md` に最新の憲法（+ 指定プロファイル）を適用する。書き込み前に diff を提示して承認を取る |
| `/carta:show [--meta] [--profile <name>]` | このプラグインにバンドルされている憲法本体（または指定プロファイル本文）を表示する。`--meta` で 1 行ヘッダも併記 |
| `/carta:diff [<path>] [--profile <names>]` | `/carta:apply` の dry-run。差分のみ表示し、書き込まない |

引数なしの場合は `git rev-parse --show-toplevel` で解決したリポジトリ root の `CLAUDE.md` が対象になる。

## 例

Before（`./CLAUDE.md`）:

```markdown
# my-project

このプロジェクトは Bun を使い、Cloudflare Workers にデプロイする。push 前に `bun test` を実行する。
```

After `/carta:apply`:

```markdown
<!-- carta:begin v0.2.0 sha=abc1234 -->
# 開発憲法

このセクションは [carta](https://github.com/hummer98/carta) によって管理されている。
直接編集せず、リポジトリ固有のルールはマーカー外に記述すること。

## 1. コミュニケーション
- ドキュメント・コメント・ユーザー向けテキストは日本語で書く。
- ...
<!-- carta:end -->

# my-project

このプロジェクトは Bun を使い、Cloudflare Workers にデプロイする。push 前に `bun test` を実行する。
```

既存の内容はマーカーブロックの下にそのまま残る。

## 憲法本体のカスタマイズ

憲法本体はこのリポジトリの `assets/constitution.md` 1 枚。自分用にカスタマイズするには:

1. carta を fork するか、自分の marketplace を作る
2. `assets/constitution.md` を編集
3. version をバンプ（`.claude-plugin/plugin.json` と `marketplace.json`）
4. tag を打って release（`/release`）
5. 各プロジェクトで `/carta:apply` を実行して新バージョンを取り込む

新しいプロファイルを追加するには、`assets/profiles/` に Markdown ファイル（末尾改行 1 個必須）を置いて plugin version を bump するだけ。先頭行は `<!-- carta-profile: <name> -->` 固定（リリース後に書式を変更すると既存 sha が壊れるので **凍結された定数** として扱う）。

## バージョニング

- [SemVer](https://semver.org/): major = 憲法構造・マーカー形式の破壊的変更、minor = 新セクション追加、patch = 文言修正
- 適用済みバージョンは begin マーカーの `v0.1.0` 部分に記録される。各プロジェクトが最新かどうかを判別できる

## ステータス

v0.2.0 — Profiles。デフォルト憲法に加えて、言語/フレームワーク別のプロファイル（`typescript`、`flutter` を最小プレースホルダとして同梱）を連結適用できる。v0.1.0 マーカーとは完全互換で、profiles 空のまま再 apply しても sha は同一。

v0.x のスコープ外: 複数 `CLAUDE.md`（mono-repo）対応、プロジェクト別 override レイヤ、CI 検出など。詳細は `docs/seed.md` §10 を参照。

## License

[MIT](LICENSE)
